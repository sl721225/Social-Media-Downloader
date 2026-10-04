# Social Media Downloader

繁體中文滑鼠懸停下載腳本，支援 Threads、Instagram、X；Facebook 為 Beta。

目前版本：**v1.1.6.3-dev — Meta Video Fixes**。

## 安裝與更新

1. 安裝 Tampermonkey。
2. 開啟 [Social-Media-Downloader.user.js](https://github.com/sl721225/Social-Media-Downloader/raw/refs/heads/main/Social-Media-Downloader.user.js)，安裝或更新同一份腳本。
3. 避免同時啟用多份版本，更新後重新整理網站。

也可從 [最新開發版 Release](https://github.com/sl721225/Social-Media-Downloader/releases/tag/v1.1.6.3-dev) 下載完整 TXT，覆蓋原有腳本後儲存。

## 使用方式

滑鼠移到照片或影片上，按「下載照片」或「下載影片」。

- X：API MP4 顯示進度、大小與 KB/s／MB/s；可取消或先中止 MP4 再改用 HLS。HLS 影片與音訊由 FFmpeg 合併，也可取消。
- IG／Threads：擷取頁面與動態 API 的影片資料，配對目前影片，依提供的尺寸優先選較高解析度的完整 MP4；保留播放器與貼文解析備援。
- Facebook Beta：擷取影片資料，優先選完整 HD MP4，再使用播放來源備援。尚未加入 DASH 分離影音合併。
- IG／Threads／FB 解析失敗時可下載診斷 TXT，回報問題。

## 驗證與限制

JavaScript 語法檢查與模擬配對測試通過。使用者已確認本版 IG／Threads 可下載，先前 FB 畫質修正有效，X 取消與 HLS 切換可操作。這些確認不代表所有貼文、輪播、帳號與網站改版都已測試。

X 的 HLS 合併按需從 jsDelivr 載入 ffmpeg.js 4.2.9003 Worker。腳本需要跨網域下載權限；本專案不附帶登入資料。只下載你有權保存的內容。
