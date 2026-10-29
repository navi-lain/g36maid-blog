---
title: "基於多 Agent 協作的 AI 資安分析系統：把四個開源專案接成一條分析流水線"
description: "以 OpenCode 為引擎、oh-my-openagent 為編排層，新增 Hades 資安 agent，掛上 Ghidra-MCP 的 193 個逆向工具與 VulnClaw 的 21 個資安 skill——零代碼整合的多 Agent 資安分析系統實記。"
pubDate: 2026-06-12 16:00:54 +0800
category: ai
tags: ["agent", "ai", "security", "opencode", "mcp", "llm", "ghidra", "vulnclaw"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1781251254
---

> 本文擴寫自 2026-06-12 的專案報告（原為 HackMD 筆記，完整版約兩萬字）。原文是一份系統設計報告，本篇取其骨幹；所有版本號與數據以 2026 年 6 月當時狀態為準，文末另有擴寫註記。

做資安分析的人大概都遇過同一種疲勞：逆向開 Ghidra、查漏洞翻 CVE 資料庫、打滲透測試開另一套框架，資料在工具之間靠複製貼上搬家。工具碎片化、知識孤島、單一模型什麼都想會但什麼都不專——這不是換更強的模型能解的，是架構問題。這篇記錄我怎麼把四個開源專案接成一條「自然語言進、資安報告出」的分析流水線。

先立術語，避免全文概念漂移：**Agent** 指「模型＋工具＋記憶」的綜合體，不是模型本身；**skill** 是 Markdown 格式的領域指令集，可附帶自己的 MCP server；**MCP**（Model Context Protocol）是讓 agent 透過統一協定呼叫外部工具的開放標準，支援 stdio、SSE、Streamable HTTP 三種傳輸。

## 四個專案，四個角色

整條流水線是：

- **OpenCode**：基礎引擎。提供 LLM 對話迴圈、20+ 內建工具、Plugin API（11+ hook points）與 MCP 客戶端，技術棧是 Bun / TypeScript / Effect-TS / Vercel AI SDK。
- **oh-my-openagent**：編排層 plugin（非 fork）。關係像「Ubuntu 之於 Debian」——在 OpenCode 上加上多模型多 agent 協作、54+ lifecycle hooks、skill 系統與 category 模型路由。
- **Ghidra-MCP**：逆向工程能力。一個 MCP server，把 Ghidra 的 193 個分析工具（反編譯、xrefs、P-code 模擬、IOC 提取等）暴露給 agent，三層架構是 agent → Python bridge → Ghidra plugin。
- **VulnClaw**：資安知識庫。21 個 skill（7 核心＋14 特化，共 180 份參考文件），從 recon、漏洞利用到 CTF 解題，這裡不直接執行它，只取其知識。

最想強調的一點：**整套整合零代碼**。沒改任何一個專案的原始碼，Hades 這個資安 agent 完全靠 `opencode.json` 定義出來。

## Hades：一個用 config 長出來的資安 agent

Hades 是我透過 oh-my-openagent 新增的 primary agent，設計原則有四條：模型可配、權限可配（`task: allow`、`call_omo_agent: deny`，強制委派走統一路徑）、VulnClaw 的 21 個 skill 透過 SKILL.md 的 `agent` 欄位鎖定為 Hades 專用、Ghidra-MCP 用 skill_mcp 機制嵌入——skill 被載入時 MCP server 自動啟動，session 結束自動清理，不常駐吃 context window。每個 session 的 MCP client 以 `sessionID:skillName:serverName` 為 key 隔離，並行安全。

oh-my-openagent 的 agent 團隊分四層權限：第一層是頂層 orchestrator（Sisyphus、Prometheus 等，Hades 就加在這層）、第二層是被委派自動路由的 Sisyphus-Junior、第三層是 Oracle／Metis 這類唯讀諮詢型 subagent、第四層是 Explore／Librarian 等終端執行者。每往下一層，能動的工具就受限一分，Hades 的權限設定也是順著這套分級走。

真正的核心創新是**知識傳遞**：不是為每個分析面向蓋專門 agent，而是 Hades 委派任務時用 `task(load_skills=[...])` 把資安 skill 注入 subagent 的 system prompt。於是原本只會 grep codebase 的 Explore、只會查文件的 Librarian，接任務的瞬間就「懂資安」了。oh-my-openagent 的 skill-resolver 會先檢查 `isSkillAllowedForTargetAgent()`，所以知識不會漏到沒權限的 agent 手上。

## 為什麼是 GLM-5.1

Hades 的主模型選了 GLM-5.1——智譜 2026 年 4 月發布的 754B MoE 開源模型，MIT 授權。理由有三：SWE-Bench Pro 58.4% 排第一（勝過 Claude Opus 4.6 的 57.3%），而程式碼理解正是漏洞追蹤的基礎；CyberGym（1,507 個真實漏洞、188 個開源專案的資安 benchmark，ICLR 2026）上拿到 68.7%、開源第一；再加上 API 定價遠低於閉源旗艦。CyberGym 整體領先者仍是閉源的 Claude Mythos Preview（83.1%），開源要追的距離誠實存在，但已進入能競爭的區間。其餘 subagent 各配各的模型：Oracle 用高推理的 GPT-5.5 當架構顧問，Explore/Librarian 用 mini 級跑快速檢索；`task(category="deep")` 委派時，category 路由系統會自動幫 Sisyphus-Junior 挑模型。

## 實測起來長怎樣

驗證過幾個場景。Binary 分析：Hades 載入含 Ghidra-MCP 的 skill，193 個工具上線，反編譯＋call graph 跑靜態分析，同時平行委派 Oracle 做架構安全評估、Explore 搜惡意模式，最後匯整成報告。Web 滲透：`task(category="deep")` 把 pentest-flow 和 exploitation skill 丟給 Sisyphus-Junior，它在單一委派內自主跑完偵察→發現→利用→後滲透，Hades 只負責最後載入 reporting skill 產結構化報告。CTF 場景則是先讓 Librarian 撈 writeup 學思路，再讓 Junior 帶 ctf-web＋web-security-advanced（34 份 playbook）雙 skill 自主解題。四個場景跑下來，最深刻的體感是「平行」二字：以前要排隊做的靜態分析、架構評估、外部檢索，現在是同時發生、由 Hades 收斂。

## 誠實的缺點清單

這套架構不是沒代價。skill 注入靠的是 prompt 空間，資安 skill 一多，context 預算就得精算；Ghidra-MCP 需要本地 Ghidra（我用 Docker headless 跑），bridge 掛了整條逆向鏈就斷；skill 格式轉換（VulnClaw → SKILL.md）是一次性苦工；多模型協作的帳單也需要盯——深度自主循環的 token 消耗和單輪對話完全不同量級。另外這是一份設計與驗證報告，測試場景是自建靶場與 CTF 題，不是大規模實戰評測，準確率數字我沒有、也不敢宣稱。

## References

- [OpenCode](https://opencode.ai/) — 引擎本體；agent/MCP 機制見[官方文件](https://opencode.ai/docs/agents/)、[MCP servers](https://opencode.ai/docs/mcp-servers/)（[GitHub](https://github.com/sst/opencode)）
- [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) — 編排層，agent 分層權限與 `task()` 委派協議
- [GhidraMCP](https://github.com/LaurieWired/GhidraMCP) — Ghidra 的 MCP bridge，本系統逆向工具的來源
- [VulnClaw](https://github.com/Netw0rkNoob/VulnClaw) — AI 滲透測試 agent，本系統資安 skill 的改編來源
- [GLM-5.1 發布公告](https://z.ai/blog/glm-5.1)、[權重](https://huggingface.co/zai-org/GLM-5.1)（MIT）— 754B MoE，SWE-Bench Pro 58.4%
- [CyberGym: Evaluating AI Agents' Real-World Cybersecurity Capabilities at Scale](https://openreview.net/forum?id=2YvbLQEdYt)（ICLR 2026）— 1,507 個真實漏洞、188 個專案

## 延伸閱讀

- [/blog/opencode-security-harness/](/blog/opencode-security-harness/)：更早（4 月）用 OpenCode＋oh-my-opencode 搭資安 harness 的實作記錄，本篇算是它的架構升級版。
- [/blog/oh-my-opencode-one-month/](/blog/oh-my-opencode-one-month/)：用一個月之後的後見與 token 帳單。

**擴寫註記（2026-10-29）**：本篇依 6 月原稿精煉，benchmark 數字（SWE-Bench Pro、CyberGym）引自原稿引用的當時測試，未重新實測；GLM-5.1 規格與 CyberGym 規模已對照官方 blog 與論文核實。後見：原作者下半年已調整這套編排的使用方式（見延伸閱讀第二篇），閱讀時請把本篇當作 6 月時點的架構快照。
