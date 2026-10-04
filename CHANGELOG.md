# 更新紀錄

## v1.1.6.3-dev — Meta Video Fixes

- 修正 IG／Threads 使用 blob 播放來源且無封面時，無法配對目前影片的情況。
- 擷取 GraphQL／API 動態回應，補上無尾斜線及带查詢參數的 GraphQL 路徑。
- 從影片附近的 React 元件、媒體 ID、封面及貼文連結辨識目標；有歧義時不任選影片。
- IG 增加登入狀態的媒體資訊 API 備援，支援 reel／reels 及使用者名稱前綴網址。
- 按影片尺寸選擇較高解析度的完整 MP4。
- FB 加入動態回應解析、目標影片辨識及 HD 來源優先，補讀新版 progressive 來源的畫質與尺寸。
- 加入 IG／Threads／FB 失敗診斷 TXT。
- 保留 X 的進度、速度、取消與 HLS 切換核心。

### 驗證

- JavaScript syntax check 通過。
- blob 無封面、元件上層 ID、A 切換 B、嵌套媒體、歧義拒絕及 GraphQL 路徑的模擬測試通過。
- 使用者確認本版 IG／Threads 已可下載；先前確認 FB 畫質改善。

開發版，沿用 Pre-release 標記。FB 尚未支援 DASH 分離影音合併。


## v1.1.2.1-dev

- 修正 Threads 先重新抓取貼文、漏掉目前頁面影片來源的流程。
- 新增同篇貼文 JSON 影片來源備援，適用 Threads／Instagram。
- 依貼文 shortcode 比對，避免選到推薦貼文；輪播來源有歧義時不任意選片。
- 將 X 取消控制明確限定在 X 網路請求。

## v1.1.2-dev — Download Control

- X MP4 直載顯示下載進度、大小及 KB/s／MB/s。
- 新增取消與改用 HLS；切換前先呼叫 MP4 abort。
- HLS 請求及 FFmpeg 合併可取消，取消後不繼續輸出檔案。
- 忽略中止後的遲到回呼。
