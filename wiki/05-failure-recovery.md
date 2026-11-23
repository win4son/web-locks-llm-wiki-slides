---
id: WIKI-05
title: 異常處理與死鎖預防（Failure Handling & Deadlock Prevention）
tags: [deadlock, AbortController, ifAvailable, timeout, rollback]
---

## 核心定義 (Core Definition)

**異常處理與死鎖預防（Failure Handling & Deadlock Prevention）** 是指在多分頁協調系統中，針對「網路請求永遠不回應」、「分頁在寫入半途崩潰」以及「循環等待鎖資源」等極端異常情境，設計超時中斷、非阻塞探測與狀態回滾（Rollback）的防禦性工程機制。

## 為什麼重要 / 解決什麼問題 (Problem & Motivation)

雖然 Web Locks API 能防止「分頁關閉」造成的殭屍鎖，但它無法阻止「分頁還活著、但內部邏輯永遠卡死」的狀況。例如：持有鎖的分頁發送了一個沒有設定 Timeout 的網路請求，且伺服器永遠不回應，該 Promise 就會永遠處於 `Pending` 狀態，導致其他 4 個排隊分頁全部陷入無限期凍結（Deadlock / Starvation）。

## 運作機制與生活類比 (Mechanism & Analogy)

- **生活類比**：為了防止第一位進資料室的助理在電話裡講個沒完（網路卡死），門禁系統加裝了**自動計時警報器（`AbortController`）**——超過 10 秒沒出來就強制斷線請他離場；同時也提供**「門縫偷看」選項（`ifAvailable`）**——如果助理只是想順便打掃（非緊急背景同步），發現裡面有人就直接放棄離開，不站在門口乾等。
- **三大原生防禦機制**：
  1. **超時強制釋放（`AbortController` + `signal`）**：
     將 `AbortSignal` 傳入 `navigator.locks.request(name, { signal }, callback)`。若等待排隊的時間超過門檻，瀏覽器會直接取消該分頁的排隊請求並拋出 `AbortError`，避免使用者介面長時間無回應。同時在 `callback` 內部對 `fetch` 設定超時，確保 Promise 必定會結束。
  2. **非阻塞試探模式（`ifAvailable: true`）**：
     設定 `{ ifAvailable: true }` 時，若鎖目前已被其他分頁持有，瀏覽器不會讓目前分頁排隊，而是立即將 `null` 傳入 `callback(lock)`。這非常適合背景定期清理或遙測日誌上傳（誰搶到誰做，沒搶到直接略過）。
  3. **錯誤隔離與帳本回滾（Error Isolation & Rollback）**：
     當 Leader 分頁網路請求失敗時，應在 `catch` 區塊中確保不將錯誤的空值寫入帳本，並直接讓 Promise 結束以釋放鎖。這樣排隊中的下一個 Follower 分頁接手鎖後，會自動接力重試請求，實現無縫容錯（Failover）。

## 關鍵限制與注意事項 (Caveats & Constraints)

- **避免巢狀鎖定（Avoid Nested Locks）**：切勿在持有 `Lock A` 的回呼函式內部再去申請 `Lock B`，否則當另一個分頁以相反順序（先 `B` 後 `A`）申請時，即使瀏覽器原生管理員也無法解開經典的循環等待死鎖（Circular Wait Deadlock）。
- **區分「排隊超時」與「執行超時」**：`navigator.locks.request` 的 `signal` 僅能中止「還在排隊等待取得鎖」的請求；一旦成功取得鎖並進入 `callback`，就必須依靠內部邏輯（如 `fetch` 的 `AbortSignal.timeout()`）來控制執行時間。