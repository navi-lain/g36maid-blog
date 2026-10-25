---
title: "用 OpenCode 打造資安 Agent Harness：從單人解題到多代理協作"
description: "單一模型對話做資安的極限在哪？這篇以 OpenCode 與 oh-my-opencode 為基底，按搭建順序記錄資安 agent harness 的實作：agents、MCP、skills 設定全可複製，附踩雷與個人評價。"
pubDate: 2026-04-21 05:43:28 +0800
category: security
tags: ["opencode", "agent", "mcp", "ctf", "cybersecurity", "llm", "ai"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1776721408
---

> 本文擴寫自 2026-04-21 的筆記（原為 HackMD 草稿），內容以當時的版本狀態為準，文末另有擴寫註記。

把 CTF 題目丟進單一模型的對話視窗，大概人人都遇過同一種崩壞曲線：前兩輪還在好好分析 binary，第三輪開始上下文塞滿了 recon 輸出，模型開始忘記自己剛剛找到的 offset；想邊逆向邊寫 exploit，同一顆腦又被規劃、執行、驗證三種角色輪流占據；更別說想同時開三題時，對話是線性的，你只能排隊。單一模型對話做資安的三個極限——上下文、角色混雜、無法並行——不是換更強的模型就能解，而是架構問題。

這篇記錄我怎麼用 [OpenCode](https://opencode.ai/) 加 [oh-my-opencode](https://github.com/code-yeongyu/oh-my-openagent)（現更名 oh-my-openagent）搭出一套資安 agent harness。先定義名詞：**Agent** 是「模型＋工具＋記憶」的綜合體，不是模型本身；**Harness** 則是 agent 與目標環境互動的整套工具鏈與提示工程。

## 架構總覽

OpenCode 原生分兩層：primary agent（Tab 鍵切換的主對話，如 Build/Plan）與 subagent（由主代理依描述自動呼叫，或用 `@` 手動指定）。oh-my-opencode 在這之上加了一個編排團隊：Sisyphus（主指揮）、Prometheus（規劃）、Oracle（架構與除錯顧問）、Librarian（文件檢索）、Explore（快速 grep）、sisyphus-junior（被委派的子任務，會依 deep/quick 等類別路由到不同模型），並支援背景平行代理與完整的 hook 系統。資安工作正好吃這套：recon 交給便宜快的、exploit 交給深的、規劃交給推理強的。

## 1. 裝 OpenCode，先拒絕供應商綁定

OpenCode 是 TypeScript 寫的終端原生開源代理，核心賣點是不綁定特定供應商，可以對接 75 種以上的模型，包含本地部署。這點對資安特別重要：閉源模型的安全護欄經常在攻防場景「降智」，拒絕被單一生態綁住，才能自由混搭。基本設定：

```jsonc
// ~/.config/opencode/opencode.json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "zhipu/glm-5"
}
```

## 2. 定義資安 subagents

在 `.opencode/agents/` 放 Markdown 檔，檔名就是 agent 名。每個角色限定工具與模型：

```markdown
---
description: 撰寫漏洞利用腳本與 payload
mode: subagent
model: zhipu/glm-5
temperature: 0.1
permission:
  edit: allow
  bash: allow
---
你是漏洞開發工程師。動手前先讀完 recon 結果，
寫完必須實際跑過並貼出輸出，不許宣稱未驗證的成果。
```

再配一個 `edit: deny` 的逆向分析 agent、一個只讀的 recon agent。**為什麼要限權限？** subagent 是會被主代理批量派發的，權限寫在 agent 定義裡，比靠提示詞求它別亂來可靠得多——這也是 OpenCode [官方文件](https://opencode.ai/docs/agents/)建議的 security auditor 做法。

## 3. 裝 oh-my-opencode，把單工變團隊

裝完 oh-my-opencode 後，Sisyphus 會接手編排：它把意圖分發給各 subagent、平行開背景代理、用 Todo Continuator 強迫中斷後接續。它也內建三個 MCP——Exa（網搜）、Context7（官方文件）、Grep.app（全域程式碼搜尋）——並能在安裝時依你手上的訂閱自動配模型與 fallback。模型配置寫在 `~/.config/opencode/oh-my-openagent.json`，我的配置放在[這個 gist](https://gist.github.com/G36maid/070db9005971a5e0a3a56c201c91458a)。

## 4. 接資安專用 MCP

[MCP](https://modelcontextprotocol.io/) 是標準化 LLM 與外部工具通訊的協定，在 `opencode.json` 的 `mcp` 區塊掛上即可：

```jsonc
"mcp": {
  "ghidra": {
    "type": "local",
    "command": ["python", "-m", "ghidra_mcp"],
    "enabled": true
  }
}
```

實用組合：ghidra-mcp 回傳反編譯、呼叫圖與符號表，逆向題的基本盤；ida-pro-mcp 更精準但要授權；動態分析則靠整合 GDB/Frida 的 MCP。Context7 注入 API 文件、Exa 補外部情資。**trade-off 在這**：每個 MCP 的工具描述都吃 context，OpenCode 官方[明文警告](https://opencode.ai/docs/mcp-servers/)挂太多會爆掉，我的做法是全域關閉、只在一兩個 agent 上啟用。

## 5. 把解題流程寫成 Skill

Skill 是把領域知識綁成模組的 `SKILL.md`，agent 需要時才載入，不吃常駐 context：

```markdown
---
name: ctf-pwn
description: 解 pwn 題的固定流程
---
## What I do
- checksec → 找 offset → 搜 ROP gadget → 寫 exploit
## When to use me
題目附 binary 且無原始碼時。
```

我把這類技能整理成 [ctf-arsenal](https://github.com/G36maid/ctf-arsenal) 專案，按 reverse/pwn/web 分類。

## 6. 實戰：金盾獎 CSC2025

這套架構在 2025 金盾獎實測：設計專屬 skill 與 MCP 後，單人編排四個平行 agent 同時解題，解出四題，並在 "llm-this" 拿到大會首殺。成本控制靠混合模型策略——推理頂尖的 Claude Opus 當 Prometheus 規劃（Plan），具攻擊性的腳本撰寫交給開源 GLM（Execute）。當時的 [模型排名](https://artificialanalysis.ai/evaluations/artificial-analysis-intelligence-index)已顯示開源權重模型逼近閉源旗艦，護欄又鬆得多，這個策略後來就成了我的預設。

## 踩雷注意

- **LLM 內建 revert 不可靠**：大檔案的字串替換常出錯，把 Git 當聖旨，狀態回退一律 `git reset`。
- **上下文要主動壓縮**：輪次一多記憶衰退，果斷 `/compact`；思路偏移就 fork 分支重給精確 prompt。
- **subagent 沒配好 provider 就是 token burner**：背景代理平行的代價是帳單也平行。
- **MCP 寧缺勿濫**：見上，glob pattern 關掉用不到的工具。

## 個人評價

跑過一輪實戰後我的結論：資安自動化卡的不是模型強弱，而是 harness。OpenCode 的價值在拒絕供應商綁定＋開放的 agent/MCP/skill 擴充面，oh-my-opencode 的價值在把「一個助理」變成「一支團隊」。缺點也老實講：OmO 啟動 overhead 不小、設定面很寬要自己摸索，閉源模型的護欄在攻防場景依舊煩人。但單人靠編排解四題這件事，已經證明開源 harness 的實戰價值。

> **擴寫註記**：此為 2026-04 的判斷。原作者後來（2026 下半年）已停用 oh-my-opencode、改回純 OpenCode——理由與取捨記錄在[用 oh-my-opencode 一個月後](/blog/oh-my-opencode-one-month/)，讀者應把兩篇對照著看。

## 延伸閱讀

- [HW12 CTF writeup](/blog/hw12-ctf-writeup/)——同一條路更早的單兵作戰版
- [用 oh-my-opencode 一個月後](/blog/oh-my-opencode-one-month/)——token 帳單的真實代價
- The Mythos Situation（[YouTube](https://www.youtube.com/watch?v=6szQxMGX5JI)）——資安代理的極限測試
