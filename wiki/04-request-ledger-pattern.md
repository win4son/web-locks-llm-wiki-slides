---
id: WIKI-04
title: 請求帳本與協調模式（Request Ledger & Coordination Pattern）
tags: [request-ledger, single-flight, coordination, zero-dependency]
---

## 核心定義 (Core Definition)

**請求帳本模式（Request Ledger Pattern）** 是一種結合 **Web Locks API（互斥控制）** 與 **共享狀態帳本（Shared Ledger State）** 的無依賴架構模式。它透過記錄每筆請求的執行狀態、時間戳與結果快取，讓多個同時發起相同請求的分頁自動分工：由第一位取得鎖的分頁擔任執行者（Leader），其餘跟隨者分頁（Followers）則在鎖釋放後直接讀取帳本中的新鮮結果，實現跨分頁的 **Single-Flight（單一飛行請求）**。

## 為什麼重要 / 解決什麼問題 (Problem & Motivation)

單獨使用 Web Locks API 雖然能讓 5 個分頁「排隊」發送請求，但若沒有狀態帳本記錄「任務已經被完成了」，排在後面的 4 個分頁在取得鎖之後，還是會傻傻地各自重新發送一次網路請求（只是從「同時發 5 次」變成「依序發 5 次」）。請求帳本模式解決了這個「排隊後仍重複執行」的關鍵缺口。

## 運作機制與生活類比 (Mechanism & Analogy)

- **生活類比**：門禁管理員（Web Locks）現在在資料室門口放了一本**公用登記帳本（Request Ledger）**。
  - **第一位助理（Leader）** 刷卡進門，翻開帳本發現「今日匯率」欄位是空白的，於是他打電話查詢，將結果「32.5」與查詢時間寫進帳本，然後刷卡離開。
  - **第二、三位助理（Followers）** 依序進門，先看帳本，發現「今日匯率」在 1 秒前剛更新過且仍在有效期限（TTL）內，便直接抄走「32.5」離開，完全不用再打電話。
- **標準執行四步驟（Double-Checked Locking Flow）**：
  1. **快速路徑檢查（Fast-Path Check）**：分頁先讀取本地帳本（如 `localStorage` 或 `IndexedDB`），若已有未過期的有效結果，直接回傳，連鎖都不用申請。
  2. **申請獨占鎖（Acquire Exclusive Lock）**：若無有效快取，呼叫 `navigator.locks.request(resourceKey, ...)` 進入排隊。
  3. **取得鎖後二次檢查（Double-Check Inside Lock）**：進入回呼函式後，**必須再次檢查帳本**！因為在排隊等待期間，前一個持有鎖的分頁可能已經把資料抓回來寫進帳本了。
  4. **執行並寫入帳本（Fetch & Commit）**：若二次檢查仍無資料，才真正發送網路請求，將結果與時間戳寫入帳本後釋放鎖。

## 關鍵限制與注意事項 (Caveats & Constraints)

- **必須執行二次檢查（Double-Checked Locking）**：缺少步驟 3 會導致所有排隊分頁重複執行請求，失去協調意義。
- **資料序列化成本**：帳本若存放在 `localStorage` 中，資料必須經過 `JSON.stringify` / `JSON.parse`，不適合存放數 MB 以上的龐大二進位檔案（大型資料應改用 `IndexedDB` 作為帳本載體）。

## 參考來源 (Sources)

- [S1] MDN — Web Locks API：https://developer.mozilla.org/en-US/docs/Web/API/Web_Locks_API
- [S2] MDN — LockManager: request() method：https://developer.mozilla.org/en-US/docs/Web/API/LockManager/request
- [S4] MDN — Web Storage API：https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API
