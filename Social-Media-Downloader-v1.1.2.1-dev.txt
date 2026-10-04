// ==UserScript==
// @name         Social Media Downloader - Hover Download
// @namespace    nono-social-media-downloader
// @version      1.1.2.1-dev
// @description  Download Control — Threads / Instagram / X / Facebook - X Hybrid Engine (API MP4 + HLS/FFmpeg fallback)
//
// @match        https://www.threads.com/*
// @match        https://threads.com/*
// @match        https://www.threads.net/*
// @match        https://threads.net/*
//
// @match        https://www.instagram.com/*
// @match        https://instagram.com/*
//
// @match        https://x.com/*
// @match        https://www.x.com/*
// @match        https://twitter.com/*
// @match        https://www.twitter.com/*
//
// @match        https://www.facebook.com/*
// @match        https://facebook.com/*
// @match        https://m.facebook.com/*
//
// @grant        GM_download
// @grant        GM_xmlhttpRequest
// @grant        unsafeWindow
// @connect      *
// @connect      cdn.jsdelivr.net
// @run-at       document-start
// ==/UserScript==

(() => {
'use strict';

// v1.0.6b：X 的下載核心沿用 v1.0.6a。
// 唯一主要變更：移除 @ffmpeg/ffmpeg ESM blob import，
// 改用 ffmpeg.js 單一 Worker 進行 VIDEO + AUDIO stream-copy 合併。

const VERSION = '1.1.2.1-dev';

// ------------------------------------------------------------
// FFmpeg
// ------------------------------------------------------------
//
// 重要：
// 不使用 @require。
// 只有 X 的 VIDEO + AUDIO 都下載完成後，才載入。
//
// 這樣即使 FFmpeg CDN / Worker / WASM 發生問題，
// 主下載器仍然會正常啟動。
//

const FFMPEG_WORKER_URL =
    'https://cdn.jsdelivr.net/npm/ffmpeg.js@4.2.9003/ffmpeg-worker-mp4.js';

let ffmpegWorkerSource = null;
let ffmpegWorkerLoadingPromise = null;

let activeMedia = null;
let activeType = null;
let button = null;
let hideTimer = null;
let resolving = false;


// ============================================================
// X API MEDIA CACHE (v1.1.0-dev)
// ============================================================
//
// X 的 GraphQL / API JSON 有時會直接帶 video_info.variants。
// 若找到 video/mp4，就優先使用最高 bitrate 的 MP4；
// 找不到時完全不影響舊版流程，會回到 HLS + FFmpeg。
//

const xApiMediaCache = new Map();

function normalizeXVariant(variant) {
    if (!variant || typeof variant !== 'object') return null;

    const url = variant.url || variant.src || '';
    const contentType =
        variant.content_type ||
        variant.contentType ||
        variant.type ||
        '';

    if (!/^https?:\/\//i.test(url)) return null;

    return {
        url,
        contentType,
        bitrate: Number(variant.bitrate) || 0
    };
}

function cacheXMediaRecord(media, tweetId = '') {
    if (!media || typeof media !== 'object') return;

    const mediaId = String(
        media.id_str ||
        media.id ||
        media.media_key?.match?.(/(\d+)$/)?.[1] ||
        ''
    );

    const variants =
        media.video_info?.variants ||
        media.videoInfo?.variants ||
        [];

    if (!mediaId || !Array.isArray(variants) || !variants.length) return;

    const mp4Variants = variants
        .map(normalizeXVariant)
        .filter(Boolean)
        .filter(item =>
            /video\/mp4/i.test(item.contentType) ||
            /\.mp4(?:\?|$)/i.test(item.url)
        )
        .sort((a, b) => b.bitrate - a.bitrate);

    if (!mp4Variants.length) return;

    const previous = xApiMediaCache.get(mediaId);
    const best = mp4Variants[0];

    if (!previous || best.bitrate >= (previous.best?.bitrate || 0)) {
        xApiMediaCache.set(mediaId, {
            mediaId,
            tweetId: String(tweetId || previous?.tweetId || ''),
            best,
            variants: mp4Variants,
            updatedAt: Date.now()
        });

        console.log(
            '[SMD:X API] cached MP4',
            mediaId,
            `${Math.round(best.bitrate / 1000)} kbps`,
            best.url
        );
    }
}

function scanXApiJSON(root) {
    if (!root || typeof root !== 'object') return;

    const seen = new WeakSet();

    function walk(node, tweetId = '') {
        if (!node || typeof node !== 'object') return;
        if (seen.has(node)) return;
        seen.add(node);

        let localTweetId = tweetId;

        // GraphQL Tweet result 常見結構：rest_id + legacy。
        if (node.rest_id && node.legacy && typeof node.legacy === 'object') {
            localTweetId = String(node.rest_id);
        } else if (node.id_str && (node.full_text || node.extended_entities)) {
            localTweetId = String(node.id_str);
        }

        // 常見 legacy.extended_entities.media / entities.media。
        const mediaLists = [
            node.extended_entities?.media,
            node.entities?.media,
            node.legacy?.extended_entities?.media,
            node.legacy?.entities?.media
        ];

        for (const list of mediaLists) {
            if (!Array.isArray(list)) continue;
            for (const media of list) {
                cacheXMediaRecord(media, localTweetId);
            }
        }

        // 防止 API 結構改版：任何帶 video_info.variants 的物件也嘗試收錄。
        if (node.video_info?.variants || node.videoInfo?.variants) {
            cacheXMediaRecord(node, localTweetId);
        }

        if (Array.isArray(node)) {
            for (const item of node) walk(item, localTweetId);
            return;
        }

        for (const value of Object.values(node)) {
            if (value && typeof value === 'object') {
                walk(value, localTweetId);
            }
        }
    }

    walk(root, '');
}

function inspectXApiPayload(payload) {
    try {
        if (!payload) return;
        if (typeof payload === 'string') {
            if (!payload.includes('video_info') && !payload.includes('videoInfo')) return;
            scanXApiJSON(JSON.parse(payload));
        } else if (typeof payload === 'object') {
            scanXApiJSON(payload);
        }
    } catch (_) {
        // API 回應很多，不讓單一非 JSON / 非預期結構影響頁面。
    }
}

function installXApiInterceptor() {
    const host = location.hostname.toLowerCase();
    if (!(host === 'x.com' || host.endsWith('.x.com') || host.includes('twitter.com'))) {
        return;
    }

    try {
        const page = typeof unsafeWindow !== 'undefined' ? unsafeWindow : window;

        if (!page.__SMD_X_API_FETCH_PATCHED__ && typeof page.fetch === 'function') {
            page.__SMD_X_API_FETCH_PATCHED__ = true;
            const originalFetch = page.fetch;

            page.fetch = function(...args) {
                const promise = originalFetch.apply(this, args);

                promise.then(response => {
                    try {
                        const type = response?.headers?.get?.('content-type') || '';
                        const url = response?.url || '';
                        if (
                            /json/i.test(type) &&
                            (/\/i\/api\//i.test(url) || /graphql/i.test(url) || /api\.x\.com/i.test(url))
                        ) {
                            response.clone().text().then(inspectXApiPayload).catch(() => {});
                        }
                    } catch (_) {}
                }).catch(() => {});

                return promise;
            };

            console.log('[SMD:X API] fetch interceptor installed');
        }

        const XHR = page.XMLHttpRequest;
        if (XHR?.prototype && !XHR.prototype.__SMD_X_API_PATCHED__) {
            XHR.prototype.__SMD_X_API_PATCHED__ = true;
            const originalSend = XHR.prototype.send;

            XHR.prototype.send = function(...args) {
                this.addEventListener('load', () => {
                    try {
                        const url = this.responseURL || '';
                        if (!(/\/i\/api\//i.test(url) || /graphql/i.test(url) || /api\.x\.com/i.test(url))) {
                            return;
                        }

                        if (this.responseType === 'json') {
                            inspectXApiPayload(this.response);
                        } else if (!this.responseType || this.responseType === 'text') {
                            inspectXApiPayload(this.responseText);
                        }
                    } catch (_) {}
                }, { once: true });

                return originalSend.apply(this, args);
            };

            console.log('[SMD:X API] XHR interceptor installed');
        }
    } catch (error) {
        console.warn('[SMD:X API] interceptor unavailable; HLS fallback remains active', error);
    }
}

installXApiInterceptor();


// ============================================================
// RESOURCE CAPTURE
// ============================================================

const capturedResources = [];
const capturedSet = new Set();

function rememberResource(url, type = '') {
    if (!url || capturedSet.has(url)) return;

    capturedSet.add(url);

    capturedResources.push({
        url,
        type,
        time: Date.now()
    });

    if (capturedResources.length > 5000) {
        const removed = capturedResources.shift();

        if (removed) {
            capturedSet.delete(removed.url);
        }
    }
}

try {
    const observer = new PerformanceObserver(list => {
        for (const entry of list.getEntries()) {
            if (entry?.name) {
                rememberResource(
                    entry.name,
                    entry.initiatorType || ''
                );
            }
        }
    });

    observer.observe({
        type: 'resource',
        buffered: true
    });

} catch (error) {
    console.warn('[SMD] PerformanceObserver error', error);
}


// ============================================================
// START
// ============================================================

function start() {

    // ========================================================
    // PLATFORM
    // ========================================================

    function getPlatform() {
        const host = location.hostname.toLowerCase();

        if (
            host.includes('threads.com') ||
            host.includes('threads.net')
        ) {
            return 'threads';
        }

        if (host.includes('instagram.com')) {
            return 'instagram';
        }

        if (
            host === 'x.com' ||
            host.endsWith('.x.com') ||
            host.includes('twitter.com')
        ) {
            return 'x';
        }

        if (host.includes('facebook.com')) {
            return 'facebook';
        }

        return 'unknown';
    }


    function platformLabel() {
        switch (getPlatform()) {
            case 'threads':
                return 'Threads';

            case 'instagram':
                return 'Instagram';

            case 'x':
                return 'X';

            case 'facebook':
                return 'Facebook';

            default:
                return 'Social';
        }
    }


    // ========================================================
    // CSS
    // ========================================================

    const style = document.createElement('style');

    style.textContent = `
        #smd106a-button {
            position: fixed !important;
            z-index: 2147483647 !important;

            display: none;
            align-items: center !important;
            justify-content: center !important;

            height: 36px !important;
            padding: 0 14px !important;

            border: 1px solid rgba(255,255,255,.22) !important;
            border-radius: 999px !important;

            background: rgba(20,20,20,.92) !important;
            color: white !important;

            font: 600 14px/1 system-ui,-apple-system,"Segoe UI",sans-serif !important;

            white-space: nowrap !important;
            cursor: pointer !important;

            box-shadow: 0 3px 14px rgba(0,0,0,.35) !important;

            backdrop-filter: blur(8px);
        }

        #smd106a-button:hover {
            background: rgba(0,0,0,.99) !important;
        }

        #smd106a-button[disabled] {
            opacity: .65 !important;
            cursor: wait !important;
        }

        #smd106a-toast {
            position: fixed !important;

            left: 50% !important;
            bottom: 28px !important;

            transform: translateX(-50%) !important;

            z-index: 2147483647 !important;

            padding: 11px 17px !important;

            border-radius: 12px !important;

            background: rgba(20,20,20,.96) !important;
            color: white !important;

            font: 500 14px/1.4 system-ui,-apple-system,"Segoe UI",sans-serif !important;

            box-shadow: 0 4px 20px rgba(0,0,0,.30) !important;

            pointer-events: none !important;
            white-space: nowrap !important;
        }

        #smd106a-progress {
            position: fixed !important;

            left: 50% !important;
            top: 24px !important;

            transform: translateX(-50%) !important;

            z-index: 2147483647 !important;

            min-width: 320px !important;
            max-width: 620px !important;

            padding: 13px 18px !important;

            border-radius: 14px !important;

            background: rgba(15,15,15,.96) !important;
            color: white !important;

            font: 500 14px/1.5 system-ui,-apple-system,"Segoe UI",sans-serif !important;

            box-shadow: 0 5px 24px rgba(0,0,0,.38) !important;

            text-align: center !important;
            pointer-events: none !important;
        }
    `;

    (document.head || document.documentElement)
        .appendChild(style);


    document.querySelectorAll(`
        #smd100-button,
        #smd101-button,
        #smd102-button,
        #smd103-button,
        #smd104-button,
        #smd104a-button,
        #smd105-button,
        #smd106-button,
        #smd106a-button,

        #smd100-toast,
        #smd101-toast,
        #smd102-toast,
        #smd103-toast,
        #smd104-toast,
        #smd104a-toast,
        #smd105-toast,
        #smd106-toast,
        #smd106a-toast,

        #smd104-stream-overlay,
        #smd104a-overlay,
        #smd105-progress,
        #smd106-progress,
        #smd106a-progress
    `).forEach(el => el.remove());


    // ========================================================
    // UI
    // ========================================================

    function toast(message, duration = 3500) {
        document
            .getElementById('smd106a-toast')
            ?.remove();

        const el = document.createElement('div');

        el.id = 'smd106a-toast';
        el.textContent = message;

        document.body.appendChild(el);

        setTimeout(
            () => el.remove(),
            duration
        );
    }


    let xControl = null;

    function controlError(code = 'CANCELLED') {
        const error = new Error(code === 'SWITCH_HLS' ? '改用 HLS' : '已取消下載');
        error.code = code;
        return error;
    }

    function checkCancelled() {
        if (xControl?.cancelled) throw controlError();
    }

    function stopXDownload(switchHLS = false) {
        const control = xControl;
        if (!control || control.stopping) return;
        if (switchHLS && (control.phase !== 'mp4' || !control.abortCurrent)) return;
        control.stopping = true;
        if (!switchHLS) control.cancelled = true;
        const error = controlError(switchHLS ? 'SWITCH_HLS' : 'CANCELLED');
        try {
            if (control.abortCurrent) control.abortCurrent(error);
        } catch (abortError) {
            // Never start HLS when abort fails.
            control.cancelled = false;
            toast('無法中止目前請求：' + abortError.message);
        } finally {
            control.stopping = false;
        }
    }

    function progress(message) {
        let el =
            document.getElementById(
                'smd106a-progress'
            );

        if (!el) {
            el = document.createElement('div');

            el.id = 'smd106a-progress';

            document.body.appendChild(el);
        }

        let label = el.querySelector('[data-smd-status]');
        if (!label) {
            label = document.createElement('div');
            label.dataset.smdStatus = 'true';
            label.setAttribute('role', 'status');
            el.appendChild(label);
            const actions = document.createElement('div');
            actions.style.cssText = 'display:flex;gap:10px;justify-content:center;margin-top:10px';
            for (const [text, switchHLS] of [['✕ 取消', false], ['⇄ 改用 HLS', true]]) {
                const action = document.createElement('button');
                action.type = 'button';
                action.textContent = text;
                action.dataset.smdAction = switchHLS ? 'hls' : 'cancel';
                action.style.cssText = 'pointer-events:auto;cursor:pointer;padding:7px 12px;border:1px solid #666;border-radius:8px;background:#333;color:white';
                action.addEventListener('click', event => {
                    event.preventDefault();
                    event.stopPropagation();
                    stopXDownload(switchHLS);
                });
                actions.appendChild(action);
            }
            el.appendChild(actions);
        }
        label.textContent = message;
        const cancel = el.querySelector('[data-smd-action="cancel"]');
        const hls = el.querySelector('[data-smd-action="hls"]');
        cancel.hidden = !xControl;
        cancel.disabled = !!xControl?.cancelled;
        hls.hidden = xControl?.phase !== 'mp4';
        hls.disabled = !xControl?.abortCurrent;
    }


    function closeProgress() {
        document
            .getElementById('smd106a-progress')
            ?.remove();
    }


    function formatBytes(bytes) {
        const n = Number(bytes) || 0;
        if (n <= 0) return '0 B';
        const units = ['B', 'KB', 'MB', 'GB'];
        const i = Math.min(Math.floor(Math.log(n) / Math.log(1024)), units.length - 1);
        const value = n / Math.pow(1024, i);
        return `${value >= 100 || i === 0 ? value.toFixed(0) : value.toFixed(1)} ${units[i]}`;
    }


    // ========================================================
    // HELPERS
    // ========================================================

    function cleanFilename(text) {
        return (text || 'SocialMedia')
            .replace(/[\\/:*?"<>|]+/g, '_')
            .replace(/\s+/g, ' ')
            .trim()
            .slice(0, 150);
    }


    function absoluteURL(url, base = location.href) {
        if (!url) return '';

        try {
            return new URL(url, base).href;
        } catch (_) {
            return '';
        }
    }


    function decodeURL(text) {
        if (!text) return '';

        let result = text;

        for (let i = 0; i < 4; i++) {
            result = result
                .replace(/\\u0026/gi, '&')
                .replace(/\\u003d/gi, '=')
                .replace(/\\u002f/gi, '/')
                .replace(/\\u0025/gi, '%')
                .replace(/\\u003a/gi, ':')
                .replace(/\\\//g, '/')
                .replace(/&amp;/gi, '&');
        }

        return result;
    }


    function unique(array) {
        return [...new Set(array.filter(Boolean))];
    }


    function sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }


    // ========================================================
    // MEDIA DETECTION
    // ========================================================

    function isUsableMedia(el) {
        if (!el || !el.isConnected) {
            return false;
        }

        if (
            el.tagName !== 'IMG' &&
            el.tagName !== 'VIDEO'
        ) {
            return false;
        }

        const rect = el.getBoundingClientRect();

        if (
            rect.width < 170 ||
            rect.height < 130
        ) {
            return false;
        }

        const css = getComputedStyle(el);

        if (
            css.display === 'none' ||
            css.visibility === 'hidden' ||
            Number(css.opacity) === 0
        ) {
            return false;
        }

        return true;
    }


    function findMediaFromPoint(x, y) {
        const elements =
            document.elementsFromPoint(x, y);

        for (const el of elements) {
            if (
                (
                    el.tagName === 'IMG' ||
                    el.tagName === 'VIDEO'
                ) &&
                isUsableMedia(el)
            ) {
                return el;
            }
        }


        for (const top of elements.slice(0, 8)) {
            let node = top;

            for (
                let depth = 0;
                depth < 6 && node;
                depth++
            ) {
                if (
                    (
                        node.tagName === 'IMG' ||
                        node.tagName === 'VIDEO'
                    ) &&
                    isUsableMedia(node)
                ) {
                    return node;
                }


                for (
                    const video
                    of node.querySelectorAll?.('video') || []
                ) {
                    if (isUsableMedia(video)) {
                        return video;
                    }
                }


                for (
                    const img
                    of node.querySelectorAll?.('img') || []
                ) {
                    if (isUsableMedia(img)) {
                        return img;
                    }
                }


                node = node.parentElement;
            }
        }

        return null;
    }


    // ========================================================
    // BUTTON
    // ========================================================

    button = document.createElement('button');

    button.id = 'smd106a-button';
    button.type = 'button';

    document.body.appendChild(button);


    function showButton(media) {
        activeMedia = media;

        activeType =
            media.tagName === 'VIDEO'
                ? 'video'
                : 'image';

        button.textContent =
            activeType === 'video'
                ? '↓ 下載影片'
                : '↓ 下載照片';

        button.style.display = 'inline-flex';

        const rect = media.getBoundingClientRect();

        const width =
            button.offsetWidth || 105;

        let left =
            rect.left +
            rect.width * .68 -
            width / 2;

        let top =
            rect.top + 12;

        left =
            Math.max(
                8,
                Math.min(
                    left,
                    innerWidth - width - 8
                )
            );

        top = Math.max(8, top);

        button.style.left =
            `${Math.round(left)}px`;

        button.style.top =
            `${Math.round(top)}px`;
    }


    function hideButton() {
        if (
            resolving ||
            button.disabled
        ) {
            return;
        }

        button.style.display = 'none';

        activeMedia = null;
        activeType = null;
    }


    function scheduleHide() {
        clearTimeout(hideTimer);

        hideTimer =
            setTimeout(
                hideButton,
                350
            );
    }


    button.addEventListener(
        'mouseenter',
        () => clearTimeout(hideTimer)
    );


    button.addEventListener(
        'mouseleave',
        scheduleHide
    );


    [
        'pointerdown',
        'pointerup',
        'mousedown',
        'mouseup',
        'dblclick'
    ].forEach(name => {
        button.addEventListener(
            name,
            event => {
                event.preventDefault();
                event.stopPropagation();
                event.stopImmediatePropagation();
            },
            true
        );
    });


    document.addEventListener(
        'mousemove',
        event => {
            if (
                event.target === button ||
                button.contains(event.target)
            ) {
                return;
            }

            const media =
                findMediaFromPoint(
                    event.clientX,
                    event.clientY
                );

            if (!media) {
                if (
                    activeMedia &&
                    !resolving
                ) {
                    scheduleHide();
                }

                return;
            }

            clearTimeout(hideTimer);

            if (
                media !== activeMedia ||
                button.style.display === 'none'
            ) {
                showButton(media);
            }
        },
        true
    );


    window.addEventListener(
        'scroll',
        () => {
            if (
                activeMedia &&
                activeMedia.isConnected &&
                button.style.display !== 'none'
            ) {
                showButton(activeMedia);
            }
        },
        true
    );


    window.addEventListener(
        'resize',
        () => {
            if (
                activeMedia &&
                activeMedia.isConnected &&
                button.style.display !== 'none'
            ) {
                showButton(activeMedia);
            }
        }
    );


    // ========================================================
    // NETWORK
    // ========================================================

    function controlledXRequest(options) {
        const control = xControl;
        if (getPlatform() !== 'x' || !control) return GM_xmlhttpRequest(options);
        checkCancelled();
        let finished = false;
        let handle;
        const clear = () => {
            if (finished) return false;
            finished = true;
            if (control.abortCurrent === abort) control.abortCurrent = null;
            return true;
        };
        const abort = error => {
            if (finished) return;
            if (typeof handle?.abort !== 'function') throw new Error('下載管理器未提供 abort');
            // Suppress late callbacks before abort, then reject the awaiting operation.
            finished = true;
            try { handle.abort(); } catch (failure) { finished = false; throw failure; }
            if (control.abortCurrent === abort) control.abortCurrent = null;
            options.onerror(error);
        };
        const wrapped = { ...options };
        for (const name of ['onload', 'onerror', 'ontimeout', 'onabort']) {
            wrapped[name] = value => {
                if (!clear()) return;
                if (name === 'onabort') options.onerror(controlError());
                else options[name]?.(value);
            };
        }
        handle = GM_xmlhttpRequest(wrapped);
        if (!finished) control.abortCurrent = abort;
        return handle;
    }

    function requestText(url) {
        return new Promise((resolve, reject) => {
            controlledXRequest({
                method: 'GET',
                url,
                timeout: 30000,

                onload(response) {
                    if (
                        response.status >= 200 &&
                        response.status < 400
                    ) {
                        resolve(response.responseText);
                    } else {
                        reject(
                            new Error(
                                `HTTP ${response.status}`
                            )
                        );
                    }
                },

                onerror: reject,

                ontimeout() {
                    reject(
                        new Error('網路逾時')
                    );
                }
            });
        });
    }


    function requestBuffer(url) {
        return new Promise((resolve, reject) => {
            controlledXRequest({
                method: 'GET',
                url,
                responseType: 'arraybuffer',
                timeout: 45000,

                onload(response) {
                    if (
                        response.status >= 200 &&
                        response.status < 400
                    ) {
                        resolve(response.response);
                    } else {
                        reject(
                            new Error(
                                `HTTP ${response.status}`
                            )
                        );
                    }
                },

                onerror: reject,

                ontimeout() {
                    reject(
                        new Error(
                            '下載片段逾時'
                        )
                    );
                }
            });
        });
    }


    function requestBlobURL(url, mimeType) {
        return new Promise((resolve, reject) => {
            GM_xmlhttpRequest({
                method: 'GET',
                url,
                responseType: 'arraybuffer',
                timeout: 60000,

                onload(response) {
                    if (
                        response.status < 200 ||
                        response.status >= 400
                    ) {
                        reject(
                            new Error(
                                `HTTP ${response.status}`
                            )
                        );

                        return;
                    }

                    const blob =
                        new Blob(
                            [response.response],
                            {
                                type: mimeType
                            }
                        );

                    resolve(
                        URL.createObjectURL(blob)
                    );
                },

                onerror: reject,

                ontimeout() {
                    reject(
                        new Error(
                            '載入 FFmpeg 元件逾時'
                        )
                    );
                }
            });
        });
    }


    // ========================================================
    // SAVE BLOB
    // ========================================================

    function saveBlob(blob, filename) {
        const url =
            URL.createObjectURL(blob);

        const a =
            document.createElement('a');

        a.href = url;
        a.download = cleanFilename(filename);
        a.style.display = 'none';

        document.body.appendChild(a);

        a.click();
        a.remove();

        setTimeout(
            () => URL.revokeObjectURL(url),
            60000
        );
    }


    // ========================================================
    // IMAGE
    // ========================================================

    function getBestImageURL(img) {
        const candidates = [];

        const srcset =
            img.getAttribute('srcset');

        if (srcset) {
            const entries =
                srcset
                    .split(',')
                    .map(item => {
                        const parts =
                            item.trim().split(/\s+/);

                        let score =
                            parseFloat(parts[1]) || 0;

                        if (
                            parts[1]?.endsWith('x')
                        ) {
                            score *= 1000;
                        }

                        return {
                            url:
                                absoluteURL(parts[0]),

                            score
                        };
                    })
                    .sort(
                        (a, b) =>
                            b.score - a.score
                    );

            candidates.push(
                ...entries.map(
                    item => item.url
                )
            );
        }


        candidates.push(
            absoluteURL(img.currentSrc),
            absoluteURL(img.src)
        );


        if (getPlatform() === 'x') {
            const originals = [];

            for (const candidate of candidates) {
                try {
                    const u =
                        new URL(candidate);

                    if (
                        u.hostname.includes(
                            'pbs.twimg.com'
                        )
                    ) {
                        u.searchParams.set(
                            'name',
                            'orig'
                        );

                        originals.push(
                            u.href
                        );
                    }

                } catch (_) {}
            }

            candidates.unshift(
                ...originals
            );
        }


        return (
            unique(candidates)
                .find(
                    url =>
                        /^https?:\/\//i
                            .test(url)
                ) ||
            ''
        );
    }


    function imageExtension(url) {
        try {
            const u =
                new URL(url);

            const format =
                u.searchParams
                    .get('format')
                    ?.toLowerCase();

            if (
                [
                    'jpg',
                    'jpeg',
                    'png',
                    'webp',
                    'gif'
                ].includes(format)
            ) {
                return (
                    '.' +
                    format.replace(
                        'jpeg',
                        'jpg'
                    )
                );
            }

            const match =
                u.pathname.match(
                    /\.(jpg|jpeg|png|webp|gif)/i
                );

            if (match) {
                return (
                    '.' +
                    match[1]
                        .toLowerCase()
                        .replace(
                            'jpeg',
                            'jpg'
                        )
                );
            }

        } catch (_) {}

        return '.jpg';
    }


    function downloadImage(img) {
        const url =
            getBestImageURL(img);

        if (!url) {
            toast(
                '找不到照片來源'
            );

            return;
        }

        GM_download({
            url,

            name:
                cleanFilename(
                    `${platformLabel()}_${Date.now()}`
                ) +
                imageExtension(url),

            saveAs: true,

            onerror() {
                toast(
                    '照片下載失敗'
                );
            }
        });
    }


    // ========================================================
    // META HELPERS
    // ========================================================

    function getDirectVideoSrc(video) {
        const values = [
            video.currentSrc,
            video.src,
            video.getAttribute('src')
        ];

        for (
            const source
            of video.querySelectorAll('source')
        ) {
            values.push(
                source.src,
                source.getAttribute('src')
            );
        }

        return (
            unique(values)
                .find(
                    url =>
                        /^https?:\/\//i
                            .test(url)
                ) ||
            ''
        );
    }


    function extractMetaVideoURLs(text) {
        const found = new Set();

        const patterns = [
            /"video_url"\s*:\s*"([^"]+)"/gi,
            /"playable_url"\s*:\s*"([^"]+)"/gi,
            /"playable_url_quality_hd"\s*:\s*"([^"]+)"/gi,
            /"contentUrl"\s*:\s*"([^"]+\.mp4[^"]*)"/gi,
            /"content_url"\s*:\s*"([^"]+\.mp4[^"]*)"/gi
        ];

        for (const pattern of patterns) {
            let match;

            while (
                (
                    match =
                        pattern.exec(text)
                ) !== null
            ) {
                const url =
                    decodeURL(match[1]);

                if (
                    /^https?:\/\//i
                        .test(url)
                ) {
                    found.add(url);
                }
            }
        }


        const blockPattern =
            /"video_versions"\s*:\s*\[([\s\S]*?)\]/gi;

        let block;

        while (
            (
                block =
                    blockPattern.exec(text)
            ) !== null
        ) {
            const urlPattern =
                /"url"\s*:\s*"([^"]+)"/gi;

            let match;

            while (
                (
                    match =
                        urlPattern.exec(block[1])
                ) !== null
            ) {
                const url =
                    decodeURL(match[1]);

                if (
                    /^https?:\/\//i
                        .test(url)
                ) {
                    found.add(url);
                }
            }
        }

        return [...found];
    }


    // Read JSON already delivered to this page, matched by exact post shortcode.
    // Never choose an unrelated recommendation or reply from the full document.
    function getInlineMetaVideoURL(postURL, video) {
        const code = postURL.match(/\/(?:post|p|reel|tv)\/([^/?#]+)/)?.[1];
        if (!code) return '';
        const records = [];
        const seen = new WeakSet();
        const walk = node => {
            if (!node || typeof node !== 'object' || seen.has(node)) return;
            seen.add(node);
            if (String(node.code || node.shortcode || '') === code) {
                const media = Array.isArray(node.carousel_media) ? node.carousel_media : [node];
                for (const item of media) {
                    const urls = (item.video_versions || []).map(v => v.url)
                        .concat(item.video_url || [], item.playable_url || [])
                        .filter(url => typeof url === 'string' && /^https?:\/\//i.test(url));
                    if (urls.length) records.push({ urls, posters: (item.image_versions2?.candidates || []).map(p => p.url) });
                }
            }
            for (const value of Object.values(node)) walk(value);
        };
        for (const script of document.querySelectorAll('script[type="application/json"]')) {
            const text = script.textContent || '';
            if (!text.includes(code)) continue;
            try { walk(JSON.parse(text)); } catch (_) {}
        }
        const key = url => {
            try { return new URL(url).pathname; } catch (_) { return ''; }
        };
        const poster = key(video.poster || video.getAttribute('poster') || '');
        const matched = poster ? records.filter(r => r.posters.some(p => key(p) === poster)) : [];
        const urls = unique((matched.length ? matched : records).flatMap(r => r.urls));
        // Multiple carousel videos require a poster match; otherwise use the existing resolver.
        if (!matched.length && records.length > 1 && new Set(records.map(r => r.urls[0])).size > 1) return '';
        return urls.sort((a,b) => scoreMeta(b) - scoreMeta(a))[0] || '';
    }

    function scoreMeta(url) {
        let score = 0;

        if (/\.mp4/i.test(url)) {
            score += 1000;
        }

        if (/hd/i.test(url)) {
            score += 100;
        }

        return score;
    }


    // ========================================================
    // THREADS
    // ========================================================

    function findThreadsPostURL(media) {
        const current =
            location.pathname.match(
                /\/@([^/]+)\/post\/([^/]+)/
            );

        if (current) {
            return (
                `${location.origin}/@${current[1]}/post/${current[2]}`
            );
        }


        let node = media;

        for (
            let i = 0;
            i < 22 && node;
            i++
        ) {
            for (
                const link
                of node.querySelectorAll?.(
                    'a[href*="/post/"]'
                ) || []
            ) {
                const match =
                    link
                        .getAttribute('href')
                        ?.match(
                            /\/@([^/]+)\/post\/([^/?#]+)/
                        );

                if (match) {
                    return (
                        `${location.origin}/@${match[1]}/post/${match[2]}`
                    );
                }
            }

            node =
                node.parentElement;
        }

        return '';
    }


    async function resolveThreadsVideo(video) {
        const directSource = getDirectVideoSrc(video);
        if (directSource) {
            return { url: directSource, filename: 'Threads_' + Date.now() + '.mp4' };
        }
        const postURL =
            findThreadsPostURL(video);

        const match =
            postURL.match(
                /\/@([^/]+)\/post\/([^/?#]+)/
            );

        if (!match) {
            throw new Error(
                '找不到 Threads 貼文'
            );
        }

        const username =
            match[1];

        const code =
            match[2];

        const inlineSource = getInlineMetaVideoURL(postURL, video);
        if (inlineSource) {
            return { url: inlineSource, filename: 'Threads_' + username + '_' + code + '.mp4' };
        }
        const html =
            await requestText(postURL);

        const found =
            new Set();

        let pos = 0;

        while (true) {
            const index =
                html.indexOf(
                    code,
                    pos
                );

            if (index < 0) {
                break;
            }

            const block =
                html.slice(
                    Math.max(
                        0,
                        index - 50000
                    ),

                    Math.min(
                        html.length,
                        index + 120000
                    )
                );

            if (
                /video_versions|video_url|playable_url/
                    .test(block)
            ) {
                extractMetaVideoURLs(block)
                    .forEach(
                        url =>
                            found.add(url)
                    );
            }

            pos =
                index +
                code.length;
        }


        const urls =
            [...found]
                .filter(
                    url =>
                        /\.mp4|fbcdn|cdninstagram/i
                            .test(url)
                )
                .sort(
                    (a, b) =>
                        scoreMeta(b) -
                        scoreMeta(a)
                );

        if (urls.length) {
            return {
                url:
                    urls[0],

                filename:
                    `Threads_${username}_${code}.mp4`
            };
        }


        const direct =
            getDirectVideoSrc(video);

        if (direct) {
            return {
                url:
                    direct,

                filename:
                    `Threads_${username}_${code}.mp4`
            };
        }


        throw new Error(
            '找不到 Threads 影片來源'
        );
    }


    // ========================================================
    // INSTAGRAM
    // ========================================================

    function findInstagramPostURL(media) {
        const current =
            location.pathname.match(
                /^\/(p|reel|tv)\/([^/?#]+)/
            );

        if (current) {
            return (
                `${location.origin}/${current[1]}/${current[2]}/`
            );
        }


        let node = media;

        for (
            let i = 0;
            i < 20 && node;
            i++
        ) {
            for (
                const link
                of node.querySelectorAll?.(
                    'a[href*="/p/"],a[href*="/reel/"],a[href*="/tv/"]'
                ) || []
            ) {
                const match =
                    link
                        .getAttribute('href')
                        ?.match(
                            /\/(p|reel|tv)\/([^/?#]+)/
                        );

                if (match) {
                    return (
                        `${location.origin}/${match[1]}/${match[2]}/`
                    );
                }
            }

            node =
                node.parentElement;
        }

        return '';
    }


    async function resolveInstagramVideo(video) {
        const direct =
            getDirectVideoSrc(video);

        if (direct) {
            return {
                url:
                    direct,

                filename:
                    `Instagram_${Date.now()}.mp4`
            };
        }


        const postURL =
            findInstagramPostURL(video);

        if (!postURL) {
            throw new Error(
                '找不到 Instagram 貼文'
            );
        }


        const inlineSource = getInlineMetaVideoURL(postURL, video);
        if (inlineSource) {
            return { url: inlineSource, filename: 'Instagram_' + Date.now() + '.mp4' };
        }
        const html =
            await requestText(postURL);

        const urls =
            extractMetaVideoURLs(html)
                .filter(
                    url =>
                        /\.mp4|fbcdn|cdninstagram/i
                            .test(url)
                )
                .sort(
                    (a, b) =>
                        scoreMeta(b) -
                        scoreMeta(a)
                );

        if (!urls.length) {
            throw new Error(
                '找不到 Instagram 影片來源'
            );
        }


        return {
            url:
                urls[0],

            filename:
                `Instagram_${Date.now()}.mp4`
        };
    }


    // ========================================================
    // X TARGET
    // ========================================================

    function findXTweetInfo(video) {
        const article =
            video.closest('article');

        if (article) {
            for (
                const link
                of article.querySelectorAll(
                    'a[href*="/status/"]'
                )
            ) {
                const match =
                    link
                        .getAttribute('href')
                        ?.match(
                            /^\/([^/]+)\/status\/(\d+)/
                        );

                if (match) {
                    return {
                        article,

                        username:
                            match[1],

                        tweetId:
                            match[2],

                        url:
                            `${location.origin}/${match[1]}/status/${match[2]}`
                    };
                }
            }
        }


        const current =
            location.pathname.match(
                /^\/([^/]+)\/status\/(\d+)/
            );

        if (current) {
            return {
                article,

                username:
                    current[1],

                tweetId:
                    current[2],

                url:
                    `${location.origin}/${current[1]}/status/${current[2]}`
            };
        }


        return null;
    }


    function getXMediaId(video) {
        const values = [
            video.poster,
            video.getAttribute('poster'),
            video.currentSrc,
            video.src
        ];

        const patterns = [
            /amplify_video_thumb\/(\d+)/i,
            /ext_tw_video_thumb\/(\d+)/i,
            /tweet_video_thumb\/(\d+)/i,
            /amplify_video\/(\d+)/i,
            /ext_tw_video\/(\d+)/i
        ];

        for (const value of values) {
            if (!value) continue;

            for (const pattern of patterns) {
                const match =
                    value.match(pattern);

                if (match) {
                    return match[1];
                }
            }
        }

        return '';
    }


    function belongsToXMedia(
        url,
        mediaId
    ) {
        return (
            url.includes(
                `/amplify_video/${mediaId}/`
            ) ||
            url.includes(
                `/ext_tw_video/${mediaId}/`
            )
        );
    }


    // ========================================================
    // X RESOURCES
    // ========================================================

    function getAllKnownResources() {
        return unique([
            ...capturedResources.map(
                item => item.url
            ),

            ...performance
                .getEntriesByType('resource')
                .map(
                    entry => entry.name
                )
        ]);
    }


    function findXMasterPlaylist(mediaId) {
        const urls =
            getAllKnownResources()
                .filter(
                    url =>
                        belongsToXMedia(
                            url,
                            mediaId
                        ) &&
                        /\.m3u8(?:\?|$)/i
                            .test(url)
                );


        const master =
            urls.find(url => {
                try {
                    return (
                        /\/pl\/[^/]+\.m3u8$/i
                            .test(
                                new URL(url)
                                    .pathname
                            )
                    );
                } catch (_) {
                    return false;
                }
            });


        return (
            master ||
            urls[0] ||
            ''
        );
    }


    // ========================================================
    // M3U8
    // ========================================================

    function parseAttributeList(text) {
        const result = {};

        const regex =
            /([A-Z0-9-]+)=("[^"]*"|[^,]*)/gi;

        let match;

        while (
            (
                match =
                    regex.exec(text)
            ) !== null
        ) {
            let value =
                match[2];

            if (
                value.startsWith('"') &&
                value.endsWith('"')
            ) {
                value =
                    value.slice(1, -1);
            }

            result[
                match[1].toUpperCase()
            ] = value;
        }

        return result;
    }


    function parseMasterPlaylist(
        text,
        masterURL
    ) {
        const lines =
            text
                .split(/\r?\n/)
                .map(
                    line => line.trim()
                )
                .filter(Boolean);

        const audio = [];
        const video = [];


        for (
            let i = 0;
            i < lines.length;
            i++
        ) {
            const line =
                lines[i];


            if (
                line.startsWith(
                    '#EXT-X-MEDIA:'
                )
            ) {
                const attrs =
                    parseAttributeList(
                        line.slice(
                            '#EXT-X-MEDIA:'
                                .length
                        )
                    );

                if (
                    attrs.TYPE ===
                    'AUDIO' &&
                    attrs.URI
                ) {
                    audio.push({
                        ...attrs,

                        url:
                            absoluteURL(
                                attrs.URI,
                                masterURL
                            )
                    });
                }
            }


            if (
                line.startsWith(
                    '#EXT-X-STREAM-INF:'
                )
            ) {
                const attrs =
                    parseAttributeList(
                        line.slice(
                            '#EXT-X-STREAM-INF:'
                                .length
                        )
                    );

                let j =
                    i + 1;

                while (
                    j < lines.length &&
                    lines[j]
                        .startsWith('#')
                ) {
                    j++;
                }

                if (
                    j <
                    lines.length
                ) {
                    video.push({
                        ...attrs,

                        url:
                            absoluteURL(
                                lines[j],
                                masterURL
                            )
                    });
                }
            }
        }


        return {
            audio,
            video
        };
    }


    function findKnownXChildPlaylists(
        mediaId
    ) {
        const videos = [];
        const audios = [];

        for (
            const url
            of getAllKnownResources()
        ) {
            if (
                !belongsToXMedia(
                    url,
                    mediaId
                )
            ) {
                continue;
            }

            if (
                !/\.m3u8(?:\?|$)/i
                    .test(url)
            ) {
                continue;
            }

            if (
                /\/pl\/avc1\//i
                    .test(url)
            ) {
                videos.push(url);
            }

            if (
                /\/pl\/mp4a\//i
                    .test(url)
            ) {
                audios.push(url);
            }
        }


        return {
            videos:
                unique(videos),

            audios:
                unique(audios)
        };
    }


    // ========================================================
    // QUALITY
    // ========================================================

    function resolutionScore(url) {
        const match =
            url.match(
                /(\d+)x(\d+)/
            );

        if (!match) {
            return 0;
        }

        return (
            Number(match[1]) *
            Number(match[2])
        );
    }


    function audioScore(url) {
        const match =
            url.match(
                /\/mp4a\/(\d+)/
            );

        return (
            Number(
                match?.[1]
            ) ||
            0
        );
    }


    function selectBestURL(
        urls,
        scorer
    ) {
        return (
            [...urls]
                .sort(
                    (a, b) =>
                        scorer(b) -
                        scorer(a)
                )[0] ||
            ''
        );
    }


    // ========================================================
    // MEDIA PLAYLIST
    // ========================================================

    function parseMediaPlaylist(
        text,
        playlistURL
    ) {
        const lines =
            text
                .split(/\r?\n/)
                .map(
                    line => line.trim()
                )
                .filter(Boolean);

        let initURL = '';

        const segments = [];


        for (const line of lines) {
            if (
                line.startsWith(
                    '#EXT-X-MAP:'
                )
            ) {
                const attrs =
                    parseAttributeList(
                        line.slice(
                            '#EXT-X-MAP:'
                                .length
                        )
                    );

                if (
                    attrs.URI
                ) {
                    initURL =
                        absoluteURL(
                            attrs.URI,
                            playlistURL
                        );
                }

                continue;
            }


            if (
                line.startsWith('#')
            ) {
                continue;
            }


            segments.push(
                absoluteURL(
                    line,
                    playlistURL
                )
            );
        }


        return {
            initURL,

            segments:
                unique(segments)
        };
    }


    // ========================================================
    // DOWNLOAD TRACK
    // ========================================================

    async function downloadTrack(
        playlistURL,
        label
    ) {
        progress(
            `X：讀取${label}播放清單…`
        );

        const playlistText =
            await requestText(
                playlistURL
            );

        const playlist =
            parseMediaPlaylist(
                playlistText,
                playlistURL
            );

        if (
            !playlist.initURL
        ) {
            throw new Error(
                `${label}播放清單沒有 EXT-X-MAP`
            );
        }

        if (
            !playlist.segments.length
        ) {
            throw new Error(
                `${label}播放清單沒有媒體片段`
            );
        }


        const buffers = [];

        progress(
            `X：下載${label}初始化資料…`
        );

        buffers.push(
            await requestBuffer(
                playlist.initURL
            )
        );


        for (
            let i = 0;
            i < playlist.segments.length;
            i++
        ) {
            progress(
                `X：下載${label} ${Math.round(i / playlist.segments.length * 100)}%｜${i}/${playlist.segments.length}`
            );

            buffers.push(
                await requestBuffer(
                    playlist.segments[i]
                )
            );
        }


        let total = 0;

        for (
            const buffer
            of buffers
        ) {
            total +=
                buffer.byteLength;
        }


        const output =
            new Uint8Array(total);

        let offset = 0;

        for (
            const buffer
            of buffers
        ) {
            const bytes =
                new Uint8Array(buffer);

            output.set(
                bytes,
                offset
            );

            offset +=
                bytes.byteLength;
        }


        checkCancelled();
        progress(`X：${label} 100%｜${formatBytes(output.byteLength)}`);
        return output;
    }


    // ========================================================
    // FALLBACK SAVE
    // ========================================================

    async function saveXSeparateTracks(
        videoBytes,
        audioBytes,
        base
    ) {
        progress(
            'X：改用雙檔模式輸出…'
        );

        saveBlob(
            new Blob(
                [videoBytes],
                {
                    type: 'video/mp4'
                }
            ),

            `${base}_VIDEO.mp4`
        );


        await sleep(600);
        checkCancelled();


        saveBlob(
            new Blob(
                [audioBytes],
                {
                    type: 'audio/mp4'
                }
            ),

            `${base}_AUDIO.m4a`
        );


        closeProgress();


        toast(
            '⚠ 自動合併失敗，已改存 VIDEO + AUDIO 兩個完整檔案',
            7000
        );
    }


    // ========================================================
    // LAZY FFMPEG WORKER LOADER - v1.0.6b
    // ========================================================

    async function ensureFFmpegWorkerSource() {
        if (ffmpegWorkerSource) {
            return ffmpegWorkerSource;
        }

        if (ffmpegWorkerLoadingPromise) {
            return ffmpegWorkerLoadingPromise;
        }

        ffmpegWorkerLoadingPromise = (async () => {
            progress('X：載入 MP4 合併器（約 10 MB）…');

            const source = await requestText(FFMPEG_WORKER_URL);

            if (!source || source.length < 1000000) {
                throw new Error('FFmpeg Worker 下載內容異常');
            }

            ffmpegWorkerSource = source;

            console.log(
                '[SMD:X] FFmpeg Worker loaded:',
                (source.length / 1024 / 1024).toFixed(2),
                'MB'
            );

            return source;
        })();

        try {
            return await ffmpegWorkerLoadingPromise;
        } catch (error) {
            ffmpegWorkerLoadingPromise = null;
            throw error;
        }
    }


    function runFFmpegWorker(videoBytes, audioBytes) {
        return new Promise(async (resolve, reject) => {
            let worker = null;
            let workerURL = '';
            let settled = false;
            let timeout;
            const control = xControl;
            const abortWorker = error => fail(error);

            const cleanup = () => {
                clearTimeout(timeout);
                if (control?.abortCurrent === abortWorker) control.abortCurrent = null;
                try {
                    worker?.terminate();
                } catch (_) {}

                if (workerURL) {
                    URL.revokeObjectURL(workerURL);
                }
            };

            const fail = error => {
                if (settled) return;
                settled = true;
                cleanup();
                reject(error instanceof Error ? error : new Error(String(error)));
            };

            try {
                const source = await ensureFFmpegWorkerSource();
                checkCancelled();
                if (control) control.abortCurrent = abortWorker;

                progress('X：啟動 MP4 合併器…');

                const workerBlob = new Blob(
                    [source],
                    { type: 'text/javascript' }
                );

                workerURL = URL.createObjectURL(workerBlob);
                worker = new Worker(workerURL);

                timeout = setTimeout(() => {
                    fail(new Error('FFmpeg Worker 合併逾時'));
                }, 180000);

                worker.onerror = event => {
                    clearTimeout(timeout);
                    fail(
                        new Error(
                            `FFmpeg Worker 啟動失敗：${event.message || 'unknown error'}`
                        )
                    );
                };

                worker.onmessage = event => {
                    const msg = event.data || {};

                    if (msg.type === 'ready') {
                        progress('X：FFmpeg 已就緒，正在合併影片＋聲音…');

                        // ffmpeg.js worker 的 MEMFS 介面。
                        // 兩個輸入都是我們前面已經組好的完整 fragmented MP4 track。
                        worker.postMessage({
                            type: 'run',
                            MEMFS: [
                                {
                                    name: 'video.mp4',
                                    data: videoBytes
                                },
                                {
                                    name: 'audio.m4a',
                                    data: audioBytes
                                }
                            ],
                            arguments: [
                                '-i', 'video.mp4',
                                '-i', 'audio.m4a',
                                '-map', '0:v:0',
                                '-map', '1:a:0',
                                '-c:v', 'copy',
                                '-c:a', 'copy',
                                '-shortest',
                                '-movflags', '+faststart',
                                'output.mp4'
                            ]
                        });

                        return;
                    }

                    if (msg.type === 'stdout' || msg.type === 'stderr') {
                        console.log(`[SMD:FFmpeg:${msg.type}]`, msg.data);
                        return;
                    }

                    if (msg.type === 'exit') {
                        console.log('[SMD:FFmpeg] exit:', msg.data);
                        return;
                    }

                    if (msg.type === 'done') {
                        clearTimeout(timeout);

                        try {
                            const files = msg.data?.MEMFS || [];
                            const output = files.find(file => file.name === 'output.mp4');

                            if (!output?.data?.length) {
                                throw new Error('FFmpeg 完成但找不到 output.mp4');
                            }

                            const result = new Uint8Array(output.data.length);
                            result.set(output.data);

                            if (settled) return;
                            settled = true;
                            cleanup();
                            resolve(result);
                        } catch (error) {
                            fail(error);
                        }
                    }
                };

            } catch (error) {
                fail(error);
            }
        });
    }


    // ========================================================
    // MUX
    // ========================================================

    async function muxXTracks(
        videoBytes,
        audioBytes,
        mediaId
    ) {
        console.log(
            '[SMD:X] Start mux:',
            mediaId,
            'video=',
            (videoBytes.byteLength / 1024 / 1024).toFixed(2),
            'MB, audio=',
            (audioBytes.byteLength / 1024 / 1024).toFixed(2),
            'MB'
        );

        const result = await runFFmpegWorker(videoBytes, audioBytes);

        console.log(
            '[SMD:X] Mux complete:',
            (result.byteLength / 1024 / 1024).toFixed(2),
            'MB'
        );

        return result;
    }


    // ========================================================
    // X DOWNLOAD
    // ========================================================

    async function downloadXVideo(
        video
    ) {
        xControl = { phase: 'hls', cancelled: false, stopping: false, abortCurrent: null };
        const info =
            findXTweetInfo(video);

        if (!info) {
            throw new Error(
                'X：找不到這支影片所屬貼文'
            );
        }


        const mediaId =
            getXMediaId(video);

        if (!mediaId) {
            throw new Error(
                'X：無法辨識 Media ID'
            );
        }


        progress(
            `X：已鎖定影片 ${mediaId}`
        );


        // ----------------------------------------------------
        // ENGINE A：X API MP4 DIRECT
        // ----------------------------------------------------

        const apiRecord = xApiMediaCache.get(String(mediaId));
        const directMP4 = apiRecord?.best?.url || '';

        if (directMP4) {
            const bitrate = apiRecord.best.bitrate || 0;
            const base = cleanFilename(
                `X_${info.username}_${info.tweetId}`
            );

            progress(
                `X：⚡ API MP4 直載｜${bitrate ? Math.round(bitrate / 1000) + ' kbps' : '最高可用畫質'}`
            );

            console.log(
                '[SMD:X] Engine A = API MP4 direct',
                { mediaId, tweetId: info.tweetId, bitrate, url: directMP4 }
            );

            xControl.phase = 'mp4';
            try {
                await new Promise((resolve, reject) => {
                    const control = xControl;
                    let handle;
                    let finished = false;
                    let loaded = 0, total = 0;
                    let sampleLoaded = 0, sampleAt = performance.now();
                    let speed = 0, lastChangeAt = Date.now();
                    const render = () => {
                        if (finished) return;
                        const idle = Date.now() - lastChangeAt;
                        const pct = total > 0 ? Math.min(100, Math.round(loaded / total * 100)) + '%' : '總大小未知';
                        const rate = idle >= 10000 ? 0 : speed;
                        const speedText = rate >= 1024 * 1024
                            ? (rate / 1024 / 1024).toFixed(2) + ' MB/s'
                            : (rate / 1024).toFixed(1) + ' KB/s';
                        progress(`X：⚡ MP4 直載｜${pct}｜${formatBytes(loaded)}${total > 0 ? ' / ' + formatBytes(total) : ''}｜${speedText}${idle >= 10000 ? '｜等待資料中…' : ''}`);
                    };
                    const watchdog = setInterval(render, 1000);
                    const finish = (error) => {
                        if (finished) return;
                        finished = true;
                        clearInterval(watchdog);
                        if (control.abortCurrent === abort) control.abortCurrent = null;
                        error ? reject(error) : resolve();
                    };
                    const abort = error => {
                        if (finished) return;
                        if (typeof handle?.abort !== 'function') throw new Error('GM_download 未提供 abort，無法安全切換 HLS');
                        // abort is called before this promise rejects and Engine B can start.
                        finished = true;
                        try { handle.abort(); } catch (failure) { finished = false; throw failure; }
                        finished = false;
                        finish(error);
                    };
                    try {
                        handle = GM_download({
                            url: directMP4, name: base + '.mp4', saveAs: true,
                            onprogress: event => {
                                if (finished) return;
                                loaded = Number(event?.loaded) || 0;
                                total = Number(event?.total) || 0;
                                const now = performance.now();
                                if (loaded !== sampleLoaded) lastChangeAt = Date.now();
                                if (now - sampleAt >= 500) {
                                    speed = Math.max(0, loaded - sampleLoaded) * 1000 / (now - sampleAt);
                                    sampleLoaded = loaded;
                                    sampleAt = now;
                                }
                                render();
                            },
                            onload: () => finish(),
                            onerror: error => finish(new Error('X：API MP4 直載失敗：' + (error?.error || 'download error'))),
                            ontimeout: () => finish(new Error('X：API MP4 直載逾時')),
                            onabort: () => finish(controlError())
                        });
                        if (!finished) control.abortCurrent = abort;
                        render();
                    } catch (error) { finish(error); }
                });
                checkCancelled();
                closeProgress();
                toast('✓ X 影片下載完成｜API MP4 直載｜免 FFmpeg', 6000);
                return;
            } catch (error) {
                if (error.code !== 'SWITCH_HLS') throw error;
                checkCancelled();
                xControl.phase = 'hls';
                progress('X：MP4 已中止，切換 HLS 引擎…');
            }
        }

        console.log(
            '[SMD:X] Engine B = HLS/FFmpeg fallback',
            { mediaId, tweetId: info.tweetId, cacheSize: xApiMediaCache.size }
        );

        progress(
            directMP4 ? 'X：MP4 已中止，啟動 HLS 引擎…' : 'X：API 沒有命中 MP4，切換 HLS 引擎…'
        );


        const masterURL =
            findXMasterPlaylist(
                mediaId
            );


        let videoPlaylists = [];
        let audioPlaylists = [];


        if (masterURL) {
            try {
                progress(
                    'X：解析 Master Playlist…'
                );

                const masterText =
                    await requestText(
                        masterURL
                    );

                const master =
                    parseMasterPlaylist(
                        masterText,
                        masterURL
                    );

                videoPlaylists =
                    master.video
                        .map(
                            item =>
                                item.url
                        );

                audioPlaylists =
                    master.audio
                        .map(
                            item =>
                                item.url
                        );

            } catch (error) {
                checkCancelled();
                console.warn(
                    '[SMD:X] Master parse failed',
                    error
                );
            }
        }


        const known =
            findKnownXChildPlaylists(
                mediaId
            );


        videoPlaylists =
            unique([
                ...videoPlaylists,
                ...known.videos
            ]);


        audioPlaylists =
            unique([
                ...audioPlaylists,
                ...known.audios
            ]);


        if (
            !videoPlaylists.length
        ) {
            throw new Error(
                'X：找不到影片播放清單'
            );
        }


        if (
            !audioPlaylists.length
        ) {
            throw new Error(
                'X：找不到音訊播放清單'
            );
        }


        const bestVideo =
            selectBestURL(
                videoPlaylists,
                resolutionScore
            );


        const bestAudio =
            selectBestURL(
                audioPlaylists,
                audioScore
            );


        console.log(
            '[SMD:X] Video playlist:',
            bestVideo
        );


        console.log(
            '[SMD:X] Audio playlist:',
            bestAudio
        );


        // ----------------------------------------------------
        // DOWNLOAD VIDEO
        // ----------------------------------------------------

        const videoBytes =
            await downloadTrack(
                bestVideo,
                '影片'
            );


        console.log(
            '[SMD:X] Video:',
            (
                videoBytes.byteLength /
                1024 /
                1024
            ).toFixed(2),
            'MB'
        );


        // ----------------------------------------------------
        // DOWNLOAD AUDIO
        // ----------------------------------------------------

        const audioBytes =
            await downloadTrack(
                bestAudio,
                '音訊'
            );


        console.log(
            '[SMD:X] Audio:',
            (
                audioBytes.byteLength /
                1024 /
                1024
            ).toFixed(2),
            'MB'
        );


        const base =
            cleanFilename(
                `X_${info.username}_${info.tweetId}`
            );


        // ----------------------------------------------------
        // NOW TRY FFMPEG
        // ----------------------------------------------------

        try {
            progress(
                'X：影音下載完成，準備自動合併…'
            );


            const outputBytes =
                await muxXTracks(
                    videoBytes,
                    audioBytes,
                    mediaId
                );


            checkCancelled();
            progress(
                'X：建立最終 MP4…'
            );


            const finalBlob =
                new Blob(
                    [outputBytes],
                    {
                        type:
                            'video/mp4'
                    }
                );


            saveBlob(
                finalBlob,
                `${base}.mp4`
            );


            closeProgress();


            toast(
                '✓ X 影片下載完成｜畫面＋聲音已自動合併',
                6000
            );


            console.log(
                '[SMD:X] Final MP4:',
                (
                    finalBlob.size /
                    1024 /
                    1024
                ).toFixed(2),
                'MB'
            );

        } catch (muxError) {
            checkCancelled();

            /*
             * 非常重要：
             *
             * FFmpeg 失敗 ≠ X 下載失敗
             *
             * VIDEO + AUDIO 已經抓到了，
             * 所以直接 fallback。
             */

            console.error(
                '[SMD:X] Auto mux failed:',
                muxError
            );


            progress(
                'X：自動合併器無法使用，改存兩個完整檔案…'
            );


            await saveXSeparateTracks(
                videoBytes,
                audioBytes,
                base
            );
        }
    }


    // ========================================================
    // FACEBOOK
    // ========================================================

    async function resolveFacebookVideo(
        video
    ) {
        const direct =
            getDirectVideoSrc(video);

        if (direct) {
            return {
                url:
                    direct,

                filename:
                    `Facebook_${Date.now()}.mp4`
            };
        }


        throw new Error(
            'FB Beta：目前無法解析這支影片'
        );
    }


    // ========================================================
    // NORMAL VIDEO
    // ========================================================

    async function downloadNormalVideo(
        video
    ) {
        let result;


        switch (
            getPlatform()
        ) {
            case 'threads':

                result =
                    await resolveThreadsVideo(
                        video
                    );

                break;


            case 'instagram':

                result =
                    await resolveInstagramVideo(
                        video
                    );

                break;


            case 'facebook':

                result =
                    await resolveFacebookVideo(
                        video
                    );

                break;


            default:

                throw new Error(
                    '不支援這個網站'
                );
        }


        if (
            !result?.url
        ) {
            throw new Error(
                '找不到影片來源'
            );
        }


        GM_download({
            url:
                result.url,

            name:
                cleanFilename(
                    result.filename
                ),

            saveAs:
                true,

            onload() {
                toast(
                    `✓ ${platformLabel()} 影片下載完成`
                );
            },

            onerror(error) {
                console.error(
                    '[SMD] download error',
                    error
                );

                toast(
                    '影片下載失敗'
                );
            }
        });
    }


    // ========================================================
    // CLICK
    // ========================================================

    button.addEventListener(
        'click',
        async event => {
            event.preventDefault();
            event.stopPropagation();
            event.stopImmediatePropagation();


            if (
                !activeMedia ||
                resolving
            ) {
                return;
            }


            const media =
                activeMedia;

            const type =
                activeType;


            resolving = true;
            button.disabled = true;


            try {
                if (
                    type === 'image'
                ) {
                    downloadImage(
                        media
                    );

                    return;
                }


                button.textContent =
                    '… 解析影片';


                if (
                    getPlatform() === 'x'
                ) {
                    await downloadXVideo(
                        media
                    );
                }

                else {
                    await downloadNormalVideo(
                        media
                    );
                }

            } catch (error) {

                console.error(
                    '[SMD]',
                    error
                );


                closeProgress();


                toast(
                    error?.message ||
                    '影片下載失敗',
                    7000
                );

            } finally {

                xControl = null;
                closeProgress();
                resolving = false;

                button.disabled = false;

                button.textContent =
                    type === 'video'
                        ? '↓ 下載影片'
                        : '↓ 下載照片';
            }
        },
        true
    );


    // ========================================================
    // SPA
    // ========================================================

    let lastURL =
        location.href;


    setInterval(
        () => {
            if (
                lastURL !==
                location.href
            ) {
                lastURL =
                    location.href;

                activeMedia = null;
                activeType = null;

                if (
                    !button.disabled
                ) {
                    button.style.display =
                        'none';
                }
            }
        },
        700
    );


    console.log(
        `%c Social Media Downloader v${VERSION} | ${platformLabel()} `,
        'background:#111;color:#fff;padding:6px 10px;border-radius:6px;font-weight:bold'
    );
}


// ============================================================
// DOM READY
// ============================================================

if (
    document.readyState ===
    'loading'
) {
    document.addEventListener(
        'DOMContentLoaded',
        start,
        {
            once: true
        }
    );

} else {
    start();
}

})();