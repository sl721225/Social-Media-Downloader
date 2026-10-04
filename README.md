# Social Media Downloader

繁體中文滑鼠懸停下載腳本，支援 Threads、Instagram、X；Facebook 為 Beta。

目前版本：**v1.1.2.1-dev — Download Control**。

## 安裝

1. 安裝 Tampermonkey 腳本管理器。
2. 開啟本儲存庫的 [Social-Media-Downloader.user.js](https://github.com/sl721225/Social-Media-Downloader/raw/refs/heads/main/Social-Media-Downloader.user.js)，確認安裝。
3. 已安裝舊版者，更新同一份腳本；請避免同時啟用多份版本。
4. 重新整理 Threads、Instagram 或 X 頁面。

也可從 [Releases](https://github.com/sl721225/Social-Media-Downloader/releases) 下載完整 TXT，將內容貼入原有腳本後儲存。

## 使用方式

滑鼠移到照片或影片上，按「下載照片」或「下載影片」。

X 優先使用 API 提供的 MP4；可查看進度、已下載大小與 KB/s／MB/s，按「取消」中止，或按「改用 HLS」先中止 MP4 再啟動 HLS。HLS 影片與音訊下載後使用 FFmpeg 合併；合併不可用時保留原有雙檔輸出備援。HLS 下載及合併期間也能取消。

Threads 優先使用目前播放器來源，再讀同篇貼文的頁面 JSON，最後使用原有貼文解析。Instagram 保留直接影片下載，加入同篇頁面資料備援。

## 已驗證範圍

- JavaScript 語法檢查通過。
- 控制邏輯與解析測試通過。
- 使用者實測確認 Threads、Instagram 影片可下載，X 取消與切換 HLS 可操作。
- 一篇 Threads 範例的 MP4 已實際下載並核對檔案格式。

不同貼文、輪播及網站改版仍可能影響解析；Facebook Beta 未在本次驗證。

## 外部元件

X 的 HLS 合併會按需從 jsDelivr 載入 ffmpeg.js 4.2.9003 Worker。腳本需要跨網域下載權限；本專案不附帶登入資料。

只下載你有權保存的內容。
