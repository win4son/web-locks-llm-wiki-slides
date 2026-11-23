---
id: WIKI-01
title: 多分頁併發痛點（The Multi-Tab Concurrency Problem）
tags: [concurrency, race-condition, duplicate-requests, lost-update]
---

## 核心定義 (Core Definition)

**多分頁併發痛點（Multi-Tab Concurrency Problem）** 是指當使用者在同一個瀏覽器中同時開啟同一網站的多個分頁（Tabs）時，各分頁擁有獨立的 JavaScript 執行環境（Execution Context），卻共享相同的網路配額、認證憑證（Auth Tokens）與本地儲存空間（如 `localStorage`、`IndexedDB`），進而在缺乏協調下產生**競態條件（Race Condition）**與資源浪費的現象。

## 為什麼重要 / 解決什麼問題 (Problem & Motivation)

在單頁應用程式（SPA）與雲端資料庫（如 Firestore）普及後，每個分頁都會在載入時主動同步資料或刷新權杖（Token）。若沒有協調機制，會引發三大嚴重後果：
1. **重複 API 請求與費用暴增**：使用者開啟 5 個分頁，前端就同時向後端或雲端資料庫發送 5 次一模一樣的查詢，造成 5 倍的讀取計費與頻寬浪費。
2. **資料覆寫遺失（Lost Update）**：分頁 A 與分頁 B 同時讀取同一份設定檔，各自修改不同欄位後寫回本地快取，後寫入的分頁會直接覆蓋先寫入的分頁結果。
3. **認證權杖刷新衝突（Token Refresh Collision）**：當 Access Token 過期時，多個分頁同時拿舊的 Refresh Token 去換發新憑證，導致後端判定重放攻擊而強制登出所有分頁。

## 運作機制與生活類比 (Mechanism & Analogy)

- **生活類比**：想像一間辦公室裡有 **5 位助理（5 個分頁）**，他們共用同一份**紙本客戶名單（本地儲存）**與同一支**外線電話（網路請求）**。如果沒有任何溝通規則，當主管說「去查一下客戶最新地址」時，5 位助理會同時撥打同一支電話詢問同一件事，甚至同時拿筆在紙本名單上塗改，把彼此寫的字蓋掉。
- **技術機制**：每個分頁運行在各自獨立的 Main Thread（或獨立渲染進程）中，記憶體彼此隔離。分頁 A 宣告的變數 `isFetching = true`，分頁 B 完全看不到，因此無法透過單純的記憶體變數阻止其他分頁重複執行。

## 關鍵限制與注意事項 (Caveats & Constraints)

- **記憶體隔離特性**：JavaScript 的閉包（Closure）、全域變數或單例模式（Singleton Pattern）僅在「單一分頁內部」有效，跨分頁完全無效。
- **觸發時機不可預測**：使用者隨時可能按 `Ctrl+T` 複製分頁、從書籤開啟多個分頁，或同時喚醒休眠中的背景分頁（Background Tabs），問題往往在無預警的瞬間同時爆發。