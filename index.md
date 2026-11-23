---
id: WIKI-00
title: 瀏覽器多分頁狀態同步與分散式鎖知識庫（Multi-Tab State Coordination & Web Locks Wiki）
audience: 具備基礎 JavaScript 知識、但尚未處理過多分頁併發問題的開發者與學生
principle: Zero-Runtime-Dependency（僅使用瀏覽器原生 Web API，不依賴外部套件）
---

# 知識庫總覽與閱讀路徑（Index & Reading Path）

本知識庫旨在解釋現代網頁應用程式在**多分頁環境（Multi-Tab Environment）**下所面臨的併發衝突，並說明如何利用瀏覽器原生的 **Web Locks API** 與 **請求帳本模式（Request Ledger Pattern）** 建立零外部依賴的協調機制。

## 條目目錄（Directory）

1. **[WIKI-01: 多分頁併發痛點（The Multi-Tab Concurrency Problem）](./01-multi-tab-problem.md)**
   - 探討多個分頁同時存取雲端資料庫或本地儲存時，引發的重複請求與資料覆寫問題。
2. **[WIKI-02: 傳統協調機制的極限（Limitations of Legacy Approaches）](./02-legacy-approaches.md)**
   - 分析 `localStorage` 輪詢與 `BroadcastChannel` 為何無法解決「原子性（Atomicity）」與「分頁崩潰殘留鎖」的缺陷。
3. **[WIKI-03: Web Locks API 核心原理（Web Locks API Fundamentals）](./03-web-locks-api.md)**
   - 介紹 `navigator.locks.request()` 的獨占鎖與共享鎖機制，以及瀏覽器層級的自動釋放保證。
4. **[WIKI-04: 請求帳本與協調模式（Request Ledger & Coordination Pattern）](./04-request-ledger-pattern.md)**
   - 結合 Web Locks 與共享狀態帳本，實現「單一分頁執行（Single-Flight）、其餘分頁共享結果」的完整架構。
5. **[WIKI-05: 異常處理與死鎖預防（Failure Handling & Deadlock Prevention）](./05-failure-recovery.md)**
   - 說明如何透過 `AbortController` 超時機制、`ifAvailable` 參數與錯誤狀態回滾（Rollback）防止系統卡死。
6. **[WIKI-06: 架構權衡與適用邊界（Architecture Trade-offs & Scope）](./06-architecture-tradeoffs.md)**
   - 釐清同源瀏覽器分頁鎖（Web Locks）與跨裝置分散式鎖（如 Redis）的邊界與選型標準。

## 核心術語表（Glossary）

- **同源（Same-Origin）**：指協定（Protocol）、網域（Host）與連接埠（Port）完全相同的網頁視窗或分頁，它們共享同一組瀏覽器儲存空間與鎖定管理器（Lock Manager）。
- **競態條件（Race Condition）**：多個分頁在缺乏協調的情況下同時讀寫同一份資源，導致最終結果取決於不可預測的執行順序。
- **原子性（Atomicity）**：一個操作要嘛全部完成，要嘛完全不執行，中間不會被其他分頁插隊打斷。
- **單一飛行請求（Single-Flight Request）**：當多個分頁同時需要同一份遠端資料時，僅允許一個領導分頁（Leader Tab）實際發送網路請求，其餘分頁等待並複用該結果的模式。