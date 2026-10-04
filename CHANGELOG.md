# 更新紀錄

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
