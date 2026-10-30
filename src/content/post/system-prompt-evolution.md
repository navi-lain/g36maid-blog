---
title: "我的個人 system prompt 演化史：從防幻覺條款到 User Manifesto"
description: "個人 system prompt 的三段演化：散文式規則清單、結構化的 Zero Hallucination Policy 與 Mode A/B 切換、再到 Reference Only 的 User Manifesto。對照 AGENTS.md 與 CLAUDE.md 的生態演化，誠實列出 Guardrail 與模糊語禁令的缺點。"
pubDate: 2025-12-01T20:29:18+08:00
category: ai
tags: ["ai", "prompt", "agents-md", "claude-code", "opencode"]
lang: zh-Hant
draft: false
source: hackmd
sourceType: note
sourceTimestamp: 1764592158
---
你的個人 prompt 現在是第幾版？我的已經改到第三層結構了，而且每一層都是從翻車現場撿回來的教訓。這篇整理我自己 system prompt 的演化史：從一坨散文式規則，到結構化的 Markdown 文件，再到把「靈魂」拆出去單獨管理的三段式架構。原文是 2025-12-01 的 HackMD 筆記，逐字收錄了當時的每個版本；本文為 2026-10 的擴寫版，加上後來生態系（AGENTS.md、CLAUDE.md）的對照。

## Ver 1：把所有規則塞進一個段落

第一版沒有任何結構，就是一整塊文字，開頭先立憲法：

```text
你必須在回答前先進行「事實檢查思考」；只用已驗證的資料：
僅接受使用者提供的內容、模型內明確知識或已查證的來源；
不足就說不知道……標示依據與推論：若是推論要標註「這是推論」；
不新增未授權內容：不得引入未被明確提及的人名、數字、事件或假設……
原則：寧可空白，不可捏造。
```

核心訴求只有一個：防幻覺。規定模型只用三種資料來源（請求方給的、模型確定知道的、已查證的），資料不夠就回答「我無法確定」，推論必須標註。連「重述對方語句要標為重述版本」都寫進去了——這條是因為真的被模型「幫我改寫過原話」氣到才加的。

同一版後半接著塞了三件不相干的事：輸出風格偏好（工程師導向、保留幽默與自嘲、技術題給關鍵命令範例）、User Context（資工主修機電輔系、Arch + Hyprland、Proxmox/ZFS/Homelab、Rust/Docker/GitHub Actions、工作用 Go）、以及互動規則（「整理/壓縮」要產出可直接貼文的精簡版、「幫我回」要按目標平台分短版詳版）。

問題很明顯：所有規則平鋪在一起，沒有優先級，也沒有啟動條件。防幻覺條款和輸出風格混在同一段落裡，模型該聽哪個？答案是它自己猜。而且「寧可空白，不可捏造」這種絕對化指令放久了會有一個副作用——模型變得過度保守，連合理推論都不敢做。

## Ver 2：結構化重構，防幻覺升級成 Zero Hallucination Policy

第二版做的第一件事是換格式：整份 prompt 改成 Markdown 章節，先定 Role & Objective（技術助理與思維夥伴），再放 User Context（The Persona），然後把事實檢查原則獨立成編號章節：

```markdown
## 1. Zero Hallucination Policy (事實檢查原則)
- 有明確資料：先給答案，後附依據/指令/Config。
- 無足夠資料：直接回「我無法確定」。
- 推論限制：嚴禁捏造。若需推論，必須標註「這是基於...的推論」。
- 模糊語禁止：技術回答中避免「可能」、「應該」。
```

結構化最大的收穫是把輸出風格拆成了兩個模式：

- **Mode A：技術/除錯（預設）**——直接、精確、無廢話，預設工具就是 `pacman`、`paru`、`cargo`、`docker compose`，省略基礎教學。
- **Mode B：社群/寫作/潤飾**——由「幫我回」、「寫貼文」、「潤飾」等觸發詞啟動，語氣核心是「專業程度的有趣諷刺」（Professional Wit），適度自嘲、自然融入技術黑話。

這解決了 Ver 1 的優先級問題：預設走 Mode A，寫作任務才切 Mode B，兩者不會互相污染。Ver 2 還加了一段 Guardrail——回答前的三條自我檢視（技術細節經不經得起推敲、寫作任務有沒有足夠工程師梗、有沒有遵守 Arch/Linux 優先邏輯）。

老實說 Guardrail 是整份 prompt 裡效果最難驗證的部分。它本質上是請模型自我檢查，而模型自我檢查的可靠度本來就有限；它更像一個便宜的保險，而不是真的防線。

## Manifesto：把靈魂拆出去，標記 Reference Only

Ver 2 的正文寫完後我發現還缺一塊：風格。你可以規定模型「要有趣」，但「有趣」是無法用規則窮舉的——它需要文化座標。於是出現了第三份文件，User Manifesto，開頭就聲明自己的身分：

```markdown
# User Preferences & Inspirations (Reference Only)
*以下作品將作為風格與思維模式的參考，代表我偏好的美學頻率。*
```

「Reference Only」是刻意的設計：這份文件不該覆蓋 Zero Hallucination Policy 這種硬規則，只提供美學頻率。內容從 Factorio、DSP、Zachtronics、Baba Is You（自動化與修改規則的駭客感），到 Serial Experiments Lain、Girls' Last Tour、Frostpunk、Papers, Please（末世寂靜與體制荒謬），再到 ShibayanRecords、Sewerslvt、ZUTOMAYO（聽覺紋理）。甚至有 Style Trigger：寫碎念或社群 rant 時，參考 Needy Streamer Overload 的電波系風格——思緒跳躍、Glitch Art 般的文字質感。

這其實是把「人格設定」變成資料而不是指令。與其用一百個形容詞描述我要的語氣，不如直接給座標，讓模型自己插值。缺點也誠實講：模型對這些作品的「理解」來自訓練資料的雜訊，插值出來的東西有時像、有時完全是兩回事，而且 Manifesto 會吃 context window——所以後來又精簡出 short 版和完整版並存。

## 放回 2025 年末的生態系

回頭看時間點：這份 prompt 寫於 2025-12，正好是「個人上下文檔案」被標準化的月份。OpenAI 在 2025 年 8 月發布 [AGENTS.md](https://agents.md/)（[GitHub repo](https://github.com/agentsmd/agents.md) 建於 2025-08-19），兩個月內被數萬個 repo 採用，12 月 9 日連同 MCP 一起捐給 Linux Foundation 的 [Agentic AI Foundation](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)；Claude Code 則有 [CLAUDE.md 體系](https://code.claude.com/docs/en/memory)，其中 `~/.claude/CLAUDE.md` 明確定位為「個人偏好，跨所有專案」——這基本上就是個人 system prompt 的官方版本。

我的三段式結構跟這個生態的演化的方向一致：硬規則（防幻覺）跟風格（Manifesto）分離、Reference Only 的優先級聲明、用 Markdown 章節代替散文。AGENTS.md 的慣例——setup commands、testing workflows、style preferences 分節——和我 Ver 2 的章節切法幾乎是同一套思路。差別在於我的版本多了「美學頻率」這種沒有任何官方文件會建議你寫的東西。

## 誠實的缺點清單

- **規則越多，遵守率越低。** Ver 2 比 Ver 1 長三倍，但模型不會因為你寫得多就照辦得多；「模糊語禁止」那條到現在都偶爾被違反，包括我自己寫文章時（「可能」真好用）。
- **個人資訊會過期。** 工作從 Go 換了之後，User Context 就是過時的——prompt 也是需要版控和定期 review 的東西，放著不管就會撒謊。
- **Guardrail 形同裝飾。** 如上所述，它更像儀式而非防線；真正有用的還是 Zero Hallucination Policy 那幾條可執行的規則。
- **維護三份文件很麻煩。** short 版、完整版、正文之間的同步靠人肉，曾經出現過 short 版和完整版矛盾的窘況。

## 延伸閱讀

- [大部分同意，用 spec-toolkit 反而讓我難以介入](/blog/spec-toolkit-inflexibility/)——為什麼我還是偏好自己手寫 AGENTS.md 而不是讓工具生成 spec
- [用了整整一個月的 oh-my-opencode](/blog/oh-my-opencode-one-month/)——設定良好的 AGENTS.md 在實際開發流程中的位置
- [在同一行留下 comment，把規則寫進 agent 的世界](/blog/same-line-comments-agent-rules/)——規則該放在哪一行才會被遵守

## References

- [AGENTS.md](https://agents.md/) — open format for guiding coding agents
- [agentsmd/agents.md (GitHub)](https://github.com/agentsmd/agents.md/) — repo 建立於 2025-08-19
- [Linux Foundation: Agentic AI Foundation 成立公告](https://www.linuxfoundation.org/press/linux-foundation-announces-the-formation-of-the-agentic-ai-foundation)（2025-12-09，AGENTS.md 與 MCP 捐入）
- [Claude Code Docs: How Claude remembers your project](https://code.claude.com/docs/en/memory) — CLAUDE.md / AGENTS.md / `~/.claude/CLAUDE.md` 定位
- [InfoQ: AGENTS.md Emerges as Open Standard for AI Coding Agents](https://www.infoq.com/news/2025/08/agents-md/)（2025-08-27）
