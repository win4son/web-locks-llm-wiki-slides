---
id: WIKI-03
title: Web Locks API 核心原理（Web Locks API Fundamentals）
tags: [web-locks, navigator.locks, exclusive-lock, shared-lock, lifecycle]
---

## 核心定義 (Core Definition)

**Web Locks API**（透過 `navigator.locks` 介面存取）是 W3C 制定的瀏覽器原生併發控制標準。它允許同源（Same-Origin）下的多個分頁、iframe 或 Web Worker，針對特定名稱的資源申請**非同步鎖（Asynchronous Locks）**，並由瀏覽器核心底層保證鎖的**原子性分配**與**生命週期自動清理**。

## 為什麼重要 / 解決什麼問題 (Problem & Motivation)

Web Locks API 從根本上消滅了 `WIKI-02` 中傳統方案的兩大致命傷：
1. **真正的原子性排隊（Atomic Queueing）**：瀏覽器內部的 Lock Manager 統一管理所有同源分頁的請求佇列，絕不會發生兩個分頁同時取得同一把獨占鎖的情況。
2. **分頁崩潰自動釋放（Crash-Safe Auto-Release）**：鎖的持有狀態與分頁的執行環境（Execution Context）緊密綁定。一旦持有鎖的分頁被關閉、重新整理或當機，瀏覽器會**立即自動釋放該鎖**並將控制權交給佇列中的下一個分頁，徹底杜絕殭屍鎖。

## 運作機制與生活類比 (Mechanism & Analogy)

- **生活類比**：瀏覽器內建了一位絕對公正且 24 小時在崗的**智慧門禁管理員（Lock Manager）**。助理想要進入資料室，必須先向管理員領取磁扣；資料室裡有人時，其他人必須在門口排隊。一旦裡面的人心臟病發被抬出去（分頁崩潰），門禁系統感應到人員離開，會自動解鎖並讓下一位排隊者進入。
- **核心技術機制**：
  1. **呼叫方式**：`navigator.locks.request(lockName, options, callback)`。
  2. **鎖的存活週期（Promise-Driven Lifecycle）**：鎖並不是靠手動呼叫 `unlock()` 釋放，而是**綁定 `callback` 回傳的 `Promise`**。只要該 Promise 處於 `Pending` 狀態，鎖就持續被持有；一旦 Promise 變為 `Fulfilled` 或 `Rejected`，鎖便自動釋放。
  3. **兩種鎖定模式（Lock Modes）**：
     - **`exclusive`（獨占鎖，預設值）**：同一時間只能有 1 個分頁持有該名稱的鎖，適用於寫入資料或發起唯一網路請求。
     - **`shared`（共享鎖）**：允許同時有多個分頁一起持有，但會阻擋 `exclusive` 鎖的進入，適用於「多方同時讀取快取」的場景。

## 關鍵限制與注意事項 (Caveats & Constraints)

- **安全上下文限制（Secure Context Required）**：`navigator.locks` 僅能在 **HTTPS** 環境或本機開發環境（`localhost` / `127.0.0.1`）下使用，普通 HTTP 網頁無法存取。
- **嚴格同源隔離（Origin-Scoped）**：不同網域（例如 `app.example.com` 與 `admin.example.com`）各自擁有獨立的 Lock Manager，彼此的鎖互不干擾。
- **跨分頁無直接資料通道**：Web Locks API 本身**只負責排隊與互斥控制，不負責傳遞回傳資料**；若要讓排隊的分頁直接拿到第一個分頁抓取的結果，必須搭配共享儲存或通訊機制（見 `WIKI-04`）。