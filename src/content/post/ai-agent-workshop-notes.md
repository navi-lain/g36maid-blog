---
title: "AI Agent Workshop 講座筆記：從 web chatbot 到 agentic workflow"
slug: ai-agent-workshop-notes
description: "2026-07-01 workshop 講稿的提煉筆記：LLM 是 I/O 黑盒子、agent 是 loop 的心智模型遷移，mini agent 現場實作與 AGENTS.md 有無對照實驗、MCP 的 Fat/Thin Tool 設計哲學、skill 與多模型編排，以及 subagent context 節省 demo 與 alphaXiv/autoarxiv URL 小把戲。"
pubDate: 2026-07-01T00:44:20+08:00
tags: ["agent", "opencode", "mcp", "llm", "glm", "agentic-workflow", "skill", "workshop"]
category: ai
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1782837860
draft: false
---
本文由 2026-07-01 的 workshop 講稿於 2026-11-01 整理擴寫。原稿是當天開著念、也給台下看的同一份講稿，這裡把它提煉成一篇可獨立閱讀的筆記：議程怎麼排、核心觀念是什麼、mini agent 怎麼刻、MCP 與 skill 各自的角色，以及現場 demo 與互動的亮點。

## 議程總覽

整場 workshop 分成三條線：

- **上半場（pure OpenCode）**：AI/Agent 是什麼＋mini agent 現場實作 → 環境配置與 first run → MCP 配置實戰 → skill 手搓 → 多模型編排 → 自建 context 語料庫＋AGENTS.md。
- **下半場（條件性）**：subagent 概念與 context 節省 demo，時間允許再加 omo（OpenCode 之上的 orchestrator harness）。
- **研究工具段**：教授特別要求不只講開發——web chatbot 的 deep-research 模式、alphaXiv 與 autoresearch。

選的工具是 [OpenCode](https://opencode.ai/)（[官方文件](https://opencode.ai/docs/)）：開源的 coding agent，可想像成開源版 Claude Code。支援 LSP、多工作階段、session 分享連結，透過 Models.dev 介接超過 75 家供應商，也吃本地模型。

## 主軸：web chatbot → agentic workflow

整場講座的知識核心是三個 insight，串成一條「心智模型遷移」的階梯：

**一、LLM 本質上就是 I/O 黑盒子。** 不管包裝成 chatbot 還是 agent，底層都是根據 input 吐 output 的機器，跟最陽春的 web chatbot 沒有本質差別。要改善 output 只有兩根槓桿：改善 input（context）、換更強的模型。Prompt、RAG、MCP、skill、loop、hook，全都是「改善 I/O」的不同實作。由此可以戳破幾個迷思：沒有魔法；「vibe coding」等於把品質交給隨機性；「換工具就變強」往往只是換了包裝 context 的方法。

**二、把 LLM 當系統來 debug，而不是當神諭求神問卜。** 對應的工程手法你都會：重新 roll 一次＝重跑測試；改 input 再 roll＝調參重跑；fork 對話 A/B 測試＝受控實驗；偷看 thinking process＝查 stack trace。模型單次輸出只是機率分佈裡的一個樣本，多抽幾次才找得到答案。

**三、Agent 說穿了就是一個 loop。** 出處是 Adam 在 ThePrimeagen podcast 的定義：Agent = LLM + Tools + Loops。流程五行講完：給 system prompt 與工具定義 → LLM 吐 output → 若是工具呼叫就執行並把結果回灌 context → 更新後的 context 再送 LLM → 重複直到模型自己決定停。Claude Code、Cursor、Gemini CLI、Zed、OpenCode 底層全是這個 loop 的不同實作。從 web chatbot 到 agentic workflow 的演進，不是換了新科技，而是**在黑盒子外圍補上工具與迴圈**。

## Mini agent 實作要點

重頭戲是現場從零刻一個 mini agent。最精華的一段 pseudo code 完整保留如下（工具層）：

```python
import os
import subprocess

SYSTEM = "You are a coding agent. Use your tools to explore and modify the codebase. Be concise."

def tool(name, desc, props, req=None):
    return {"type": "function", "function": {"name": name, "description": desc,
            "parameters": {"type": "object", "properties": props, "required": req or list(props)}}}

TOOLS = [
    tool("read_file", "Read a file's contents", {"path": {"type": "string"}}),
    tool("list_files", "List directory contents", {"path": {"type": "string", "default": "."}}),
    tool("write_file", "Create or overwrite a file (creates parent dirs)",
         {"path": {"type": "string"}, "content": {"type": "string"}}),
    tool("bash", "Run a shell command and return output", {"command": {"type": "string"}}),
]


def run_tool(name, args):
    try:
        if name == "read_file":
            return open(args["path"]).read()
        if name == "list_files":
            return "\n".join(sorted(os.listdir(args.get("path", "."))))
        if name == "write_file":
            os.makedirs(os.path.dirname(args["path"]) or ".", exist_ok=True)
            open(args["path"], "w").write(args["content"])
            return f"Wrote {args['path']}"
        if name == "bash":
            r = subprocess.run(args["command"], shell=True, capture_output=True, text=True, timeout=30)
            return (r.stdout + r.stderr).strip() or "(no output)"
    except Exception as e:
        return f"Error: {e}"
```

搭配的 loop 就是把 `TOOLS` 塞進 API 呼叫、接到 `run_tool`、把結果回灌 context 再重來。這段 code 同時預告了後面的所有主題：`config["model"]` 可抽換（多模型編排）、`SYSTEM` 對應 AGENTS.md、`read/write/bash` 之外的工具來源就是 MCP。想深入可讀 [AMP 的 how-to-build-an-agent](https://ampcode.com/notes/how-to-build-an-agent) 與 [Mihail Eric 的 200 行 Claude Code](https://www.mihaileric.com/The-Emperor-Has-No-Clothes/)。

寫完不是重點，重點是拿它當實驗台：同一任務先後加、不加 AGENTS.md 跑一次，行為天差地別；同一份 AGENTS.md 抽換不同模型，品質落差一目了然——「只改一個維度、比較 output」的受控實驗法直接內化。

## MCP：動態擴充工具，但不是越多越好

MCP（Model Context Protocol）就是一個通訊協定：把 local/remote 的 RPC 服務包裝成「工具＋工具描述」掛到 agent 清單上，呼叫起來跟內建 bash 一樣自然。常被比喻成「AI 的 USB 介面」，裡面沒有魔法。現場掛了 Context7、Exa Web Search、Grep_app、MarkItDown、DeepWiki，加碼還有 Ghidra（binary 逆向）與 Playwright。

但接著戳破迷思：**工具絕對不是越多越好**。講稿引用了幾組數據——GPT-4o 控制實驗中 tool-selection 準確率從 4 個工具的 43% 暴跌到 51 個工具的 2%；有實測掛 4 個 server 共 167 個工具，光讀工具描述就吃掉 60K tokens；Anthropic 也警告塞 30–50 個工具後能力明顯衰退（見[官方工程指引](https://www.anthropic.com/engineering/writing-tools-for-agents)）。設計上分兩派：**Fat Tool**（1–3 個、內部深度實作，把複雜度消化在工具裡，如 Context7）與 **Thin Tool**（數十個薄 API，把編排負擔推給 LLM，如 GitHub MCP）——後者用弱模型會很慘，緩解手法是先讓強模型跑出「示範軌跡」再讓弱模型模仿。

## Skill：任務級的 context 注入

有了 repo 級的 system prompt（AGENTS.md）與動態工具（MCP），第三層是 skill：一個 markdown 檔，定義特定任務的指令與先備知識，本質是「模組化、可隨選呼叫的 prompt」，用 slash command 在需要時才觸發。Hands-on 環節每人手搓一個自己關心主題的 `<anything>.md`，掛進 skill 目錄後用 `/` 呼叫；想不到題目就去 GitHub 找現成的 clone 下來玩。

## 多模型編排與自建語料庫

改善 I/O 的兩根槓桿中，「模型」不必只能選一個：貴模型規劃＋便宜模型實作、長 context 模型當壓縮機、多模態模型當 OCR——共同邏輯是**純文字是最通用的匯流排**。講者也分享了被 Google ban 帳號的血淚史，以及 omo 把 Claude 訂閱 OAuth token 接進 OpenCode 之後引發 Anthropic 封鎖的始終（[omo repo](https://github.com/code-yeongyu/oh-my-openagent)）。再進一步是 INSIGHT-7：與其讓 AI 盲猜，不如為每門課建一個 git 管理的 markdown 語料庫，作業 repo 用 submodule 引用——徒手搓出比 NotebookLM 更可控、可重組的系統。AGENTS.md 則已是業界事實標準（[agents.md](https://agents.md/)），README 寫給人看、AGENTS.md 寫給 agent 看；排雷重點是各家對巢狀目錄的探索機制不一致。

## 現場 demo 與互動亮點

- 同一模型不同 input，output 天差地別——講者從 Zed text-thread 時代就靠這招搞懂 context。
- 全場一起 re-roll 同一道題，親眼看見機率分佈。
- mini agent 的 AGENTS.md 有無對照實驗與模型大亂鬥。
- subagent 受控實驗：不 delegate 時 context 被灌爆，delegate 給 explore subagent 只回傳一句精煉總結，主 context 保持乾淨。
- URL 小把戲：把 arXiv 網址改成 `alphaxiv` 解鎖論文問答與行級討論，改成 `autoarxiv` 雲端部署 agent 修環境、跑最小重現。
- 收尾警示：Meta AI 安全總監的「信箱事件」——context compaction 吃掉「確認前不要動作」的指令，agent 進入 speedrun 模式差點刪光信箱。

## 兩條 meta insight

**資訊飲食決定 insight。** LLM 是 I/O 黑盒子，人腦也是：餵搬運層內容就長不出判斷力。直接讀初級來源——ThePrimeagen、Theo、Fireship、GitHub issues——三個月就會有感。**AI 不會轉移工程責任，只改變施力點。** TDD 讓你定義「Done」的客觀標準、human-in-loop 依風險彈性調整、過濾 LSP 噪音、Git 才是唯一 source of truth。學會駕馭 AI，反而要求你站在更高維度把控全局。

---

延伸閱讀：本地跑開源模型的部署筆記見 [EXL3 GLM 本地部署](/blog/exl3-glm-local-deploy/)與[MiMo EXL3 進度](/blog/miaai-exl3-progress/)；Agent 逃逸與安全面的實例見 [OpenAI Agent 的 HF escape](/blog/openai-agent-hf-escape/)；講者作為 Zed contributor 的移植紀錄見 [Zed FreeBSD port](/blog/zed-freebsd-port/)。
