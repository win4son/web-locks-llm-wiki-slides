---
id: WIKI-06
title: 架構權衡與適用邊界（Architecture Trade-offs & Scope）
tags: [architecture, trade-offs, redis, distributed-lock, scope]
---

## 核心定義 (Core Definition)

**架構權衡與適用邊界（Architecture Trade-offs & Scope）** 是指在系統設計階段，明確區分「單一使用者在單一瀏覽器內的多分頁協調（Client-Side Multi-Tab Coordination）」與「跨裝置、跨使用者的全域分散式協調（Server-Side Distributed Coordination）」，並依據一致性要求與基礎設施成本選擇合適方案的決策準則。

## 為什麼重要 / 解決什麼問題 (Problem & Motivation)

初學者在學會 Web Locks API 後，容易產生「把所有並發問題都交給瀏覽器鎖解決」的誤解；反之，部分架構師則會過度設計（Over-Engineering），為了解決前端多分頁重複打 API 的問題，卻在後端架設昂貴的 Redis 分散式鎖叢集。釐清邊界能幫助開發者用最低成本解決正確的問題。

## 運作機制與生活類比 (Mechanism & Analogy)

- **生活類比**：**Web Locks API** 就像是**家裡的浴室門鎖**，只協調「同一棟房子裡的家人（同一台電腦同一瀏覽器的多個分頁）」不要同時衝進去；而 **後端分散式鎖（如 Redis Redlock）** 則是**全國演唱會售票系統的中央資料庫鎖**，用來協調「全台灣不同城市、不同手機的使用者」搶同一張票。
- **適用場景對照矩陣**：
  - **適合使用 Web Locks + Request Ledger 的場景（同源瀏覽器內）**：
    - 避免同一使用者開 5 個分頁重複讀取相同的個人資料或設定檔。
    - 協調多分頁共用的 OAuth Access Token 自動刷新。
    - 協調多分頁對本機 `IndexedDB` 離線佇列的批次寫入與同步。
  - **必須使用後端分散式鎖 / 資料庫交易（Transaction）的場景（跨裝置／跨使用者）**：
    - 電商庫存扣減、搶票系統（不同使用者的瀏覽器彼此完全隔離）。
    - 同一使用者同時用「手機 App」與「電腦瀏覽器」修改同一筆金融帳務。

## 關鍵限制與注意事項 (Caveats & Constraints)

- **無痕模式隔離（Incognito / Private Browsing Isolation）**：瀏覽器的無痕視窗擁有獨立的儲存空間與 Lock Manager，與一般視窗之間不會共享鎖定狀態。
- **不能取代後端安全驗證**：前端協調機制屬於「效能與體驗最佳化（Client-Side Optimization）」，惡意使用者仍可透過開發者工具繞過前端鎖直接發送請求，因此關鍵資料的一致性與權限檢查仍須由後端把關。