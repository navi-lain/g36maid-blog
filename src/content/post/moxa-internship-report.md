---
title: "實習總結：把公司 fork 的 Headscale 從 v0.22.3 追上上游"
slug: moxa-internship-report
description: "Moxa 暑期實習結業戰記：公司基於 Headscale v0.22.3 的 fork 落後上游三個版本，我負責評估可移植性並把自有功能搬上新架構——Notifier 推送、cluster 事件驅動同步、ACL gRPC API 重實作，加上 Gemini/Claude/GPT 分工的 LLM 工作流，與誠實列出的未竟事項。"
pubDate: 2025-08-27T08:00:08+08:00
tags: ["headscale", "tailscale", "internship", "golang", "vpn", "llm", "docker-compose", "grpc"]
category: work
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1756252808
draft: false
---
> 本文由 2025-08 的 HackMD 結業報告於 2026-11-01 擴寫。原稿是 slide 格式的暑期結業報告，本文重寫成敘事版本；內部資料與客戶細節已去識別化，技術內容保留。

暑期實習的結業報告我準備了十分鐘，工作內容佔八分鐘。但如果濃縮成一句話，大概是這樣：**把公司 fork 的 [Headscale](https://github.com/juanfont/headscale) 從 v0.22.3 追上上游**。這篇是那場報告的戰記版。

## 背景：一個落後三個版本的 fork

實習在 Moxa 的網通軟體部門。公司的遠端連線產品底層建立在 Headscale 之上——也就是[自架版的 Tailscale 控制伺服器](https://github.com/juanfont/headscale)。一個是頭（Head），一個是尾（Tail）：[Tailscale 客戶端](https://tailscale.com/blog/how-nat-traversal-works/)自帶內網穿透，Headscale 負責發號施令。

問題很經典：公司基於 [v0.22.3](https://github.com/juanfont/headscale/releases/tag/v0.22.3) 開發了許多自有功能，但上游在我實習期間已經發布到 [v0.26.0](https://github.com/juanfont/headscale/releases/tag/v0.26.0)（2025-05-14）。差距逐漸擴大，錯失新功能、效能優化和安全性修復，而且自有功能與上游分離得越久，未來同步的成本越高。

我的任務：確認版本變化、在新版上重新實作現有功能、保持**相容性**與**行為一致**。

## 第一個震撼：上游已經整個重寫

打開新版原始碼，我愣了一下。舊版的 `hscontrol/` 是一大包扁平的 `.go` 檔；新版切成了 `db/`、`types/`、`policy/`、`mapper/`、`notifier/`、`util/` 等模組。[v0.23.0-beta.5 的 release notes](https://github.com/juanfont/headscale/releases/tag/v0.23.0-beta.5) 明說「rewrote over 10000 lines of code」——這不是升級，是換了一個骨架。

對移植影響最大的三個改變：

1. **[Notifier 統一通知機制](https://github.com/juanfont/headscale/releases/tag/v0.23.0)**。舊版每個客戶端各自開 goroutine 輪詢 server、各自管理資料狀態，頻繁讀寫 DB。新版改用 Go channel 集中管理事件，支援指定節點、排除節點、廣播三種模式，還能送 delta 更新——從輪詢變推送。
2. **認證流程重寫**。[v0.25.0](https://github.com/juanfont/headscale/releases/tag/v0.25.0)（2025-02-11）的 breaking change 就是 authentication flow 整個 rewrite，我必須把舊版登入行為移植到新架構上，並確保 API 回應格式一致。
3. **路由機制砍掉重練**。[v0.26.0](https://github.com/juanfont/headscale/releases/tag/v0.26.0) 把 route 資料表整個移除，狀態改放記憶體。凡是有動到 routes 的自有功能，都得跟著改。

## 移植的四件硬仗

**Cluster 同步**：舊版完全靠共享 DB，各節點定期檢查時間戳、主動輪詢。我保留時間戳機制確保一致性，但結合 Notifier 改成事件驅動——收到通知才讀寫 DB。DB 操作量直接掉一個量級。

**Login process**：純移植。把舊版邏輯搬進新架構，另外實作了靈活的節點查找機制，支援不同 Key 與認證方式的組合。

**ACL Config API**：新版沒有涵蓋公司需要的所有 ACL Policy 操作，我重新實作了完整的 gRPC API，適配新版內部結構、保持介面向下相容（[官方 API 文件](https://headscale.net/stable/ref/api/)裡沒有的部分全靠自己補）。

**其他零碎但麻煩的**：PreAuth Key 綁定節點、依 tag 配發 VPN IP、routes/peers 快取減少 DB 查詢、設定檔支援環境變數格式……這種功能一個接一個，平均兩天能移植完一個，靠的正是下面這套工作流。

**驗證**靠 Docker Compose：公司的產品是多服務架構，我用 Compose 建出一套 cluster server 加多個 client 的完整環境，模擬生產互動，再用 Postman 對比新舊版本的 API 行為差異、LazyDocker 盯容器狀態。控制平面行為一致才算過關。

## LLM 是隊友，不是代打

這場戰役我的分工是：Gemini 有最大上下文窗口，負責分析 code base 和 diff、產出 markdown spec；Claude 最會寫程式，照 spec 實作、配 LSP 修 bug、跑診斷與整合測試；GPT 最便宜，改文件、格式化報告、生 commit message。

但誠實說：LLM 無法維持清晰的 mental model。程式碼它寫得飛快，業務邏輯卻藏在 mentor 的腦袋和幾個早已關閉的 ticket 裡——測試場景不足時，它會很自信地寫出「符合 spec 但不符合現實」的程式。整個移植過程中，真正記得全貌的只有我。如果公司能把 feature 背後的業務邏輯文件化，或提供 MCP 讓模型查內部脈絡，這個問題就能大幅緩解。

## 收穫，以及沒做完的

最大的成長是理解了「開發 ≠ 完成功能」：功能寫完只是開始，能對齊實際業務場景才算數。這是大規模專案教我的第一課，也是自架 side project 永遠學不到的。

誠實的缺點也要列：整合測試還要在類生產環境跑過、效能基準測試與調優都還沒做，這些是實習結束後的後續工作。這份報告寫的是「移植完成」，不是「上線完成」。

延伸閱讀：實習期間每天在用的 [Zed 是我自己 port 到 FreeBSD 的](/blog/zed-freebsd-port/)；實習結束後我把 [GLM 用 EXL3 跑在本地的過程也記錄成文](/blog/exl3-glm-local-deploy/)。
