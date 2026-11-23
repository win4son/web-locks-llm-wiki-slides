---
id: WIKI-02
title: 傳統協調機制的極限（Limitations of Legacy Approaches）
tags: [localStorage, BroadcastChannel, atomicity, zombie-lock]
---

## 核心定義 (Core Definition)

**傳統協調機制（Legacy Coordination Approaches）** 是指在原生鎖定標準普及前，開發者利用瀏覽器既有的跨分頁通訊或儲存介面——主要是 **`localStorage` 搭配 `storage` 事件**，以及 **`BroadcastChannel` API**——手動拼湊出的簡易鎖定與狀態通知方案。

## 為什麼重要 / 解決什麼問題 (Problem & Motivation)

為了解決 `WIKI-01` 提到的重複請求問題，早期工程師嘗試在 `localStorage` 中寫入一個標記（例如 `localStorage.setItem('fetch_lock', 'true')`），或透過 `BroadcastChannel` 廣播「我正在抓資料，你們先等一下」。理解這些做法的缺陷，是理解為何現代應用必須轉向原生鎖定機制的關鍵前提。

## 運作機制與生活類比 (Mechanism & Analogy)

- **生活類比**：這就像把一張**便條紙貼在白板上**代表「鎖」。每位助理進門前先看白板上有沒有便條紙；如果沒有，就自己寫一張貼上去再開始工作，做完後撕掉。
  - **缺陷一（非原子性）**：兩位助理可能在同一毫秒看到白板是空的，同時伸手貼上便條紙，結果兩個人都以為自己搶到了工作權。
  - **缺陷二（分頁崩潰殘留）**：貼上便條紙的助理突然暈倒離開辦公室（分頁當機或被強制關閉），那張便條紙會永遠留在白板上，導致其他助理無限期乾等。
- **技術機制與缺陷分析**：
  1. **`localStorage` 非原子性（Non-Atomic Check-Then-Act）**：讀取 `getItem()` 與寫入 `setItem()` 是兩個分離的步驟，在多執行緒／多進程環境下存在時間差（TOCTOU 漏洞），無法保證互斥。
  2. **殭屍鎖（Zombie Lock）**：若持有鎖的分頁因使用者直接關閉視窗、系統記憶體不足（OOM）或程式碼拋出未捕獲例外而終止，`beforeunload` 事件不一定會觸發，導致 `localStorage` 中的鎖永遠無法被清除。
  3. **`BroadcastChannel` 無狀態性**：`BroadcastChannel` 僅負責即時訊息傳遞（Pub/Sub），不保存狀態。新開啟的分頁無法得知「目前是否已有分頁正在執行任務」，也無法自動感知發送者是否已經當機。

## 關鍵限制與注意事項 (Caveats & Constraints)

- **輪詢與退避演算法過於複雜**：若強行用 `localStorage` 實作心跳檢測（Heartbeat）與過期時間戳（TTL）來清除殭屍鎖，會引入大量難以維護的計時器與邊界條件 Bug，違背輕量化設計初衷。
- **效能瓶頸**：`localStorage` 為同步阻塞 I/O（Synchronous I/O），高頻率讀寫會直接卡頓主執行緒的畫面渲染。