---
title: "單機 DGX Spark 跑 DeepSeek-V4-Flash-0731，讓 pi agent 自己打完 AoC 2025"
description: "一台 DGX Spark、一份 UD-Q2_K_XL、一個極簡 coding agent：讓 DeepSeek-V4-Flash-0731 自己寫 Rust 解 Advent of Code 2025 的實錄——貪婪策略、131k context、與那個過期的 session cookie。"
pubDate: 2026-08-07T16:33:26+08:00
updated: 2026-10-30T00:00:00+08:00
category: ai
tags: ["local-llm", "dgx-spark", "deepseek", "pi", "advent-of-code", "rust", "quantization"]
draft: false
source: "facebook"
sourceType: "相片"
sourceTimestamp: 1786091606
---

> 本文擴寫自 2026-08-07 的 Facebook 相片貼文（原稿只有兩行＋一張 agent 截圖）。擴寫於 2026-10-30，硬體與模型版本以發文當時為準。

今年八月我給自己出了一個組合題：**用單一台 DGX Spark 部署 DeepSeek-V4-Flash-0731（UD-Q2_K_XL），再讓 [pi](https://pi.dev/) agent 自己把 [Advent of Code 2025](https://adventofcode.com/2025) 打完**。結果如封面截圖：agent 在 `~/Code/playground/advent-of-code-2025` 裡自己寫 Rust、跑 `cargo test`、檢查 clippy、commit——圖中那天它剛解完 Day 3。

![pi agent TUI 截圖：DeepSeek-V4-Flash-0731 解完 AoC 2025 Day 3，feat(day03) commit、Part 1 答案 17031、Part 2 答案 168575096286051，並附貪婪策略解說；底部狀態列顯示 deepseek-v4-flash-0731 · medium、context 131k](/media/2026-08/122171359904958395.jpg)

*圖說：pi 的 TUI（標題列那個 π）。畫面裡 agent 剛跑完 `cargo clippy`、確認 warning 都是模板既有問題後，下了 `git commit -m "feat(day03): solve part 1 and 2"`（`3f49443`，27 insertions / 4 deletions）。*

## 1. 為什麼是這三樣東西

**[DGX Spark](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)** 的賣點早已講爛：GB10 Grace Blackwell、128GB LPDDR5x 統一記憶體、273 GB/s 頻寬。對跑大 MoE 來說，重點是那個「統一」——CPU 和 GPU 共享同一塊 128GB，模型能塞多大只看記憶體總量，不像獨顯卡那樣被 24GB VRAM 卡死。

**[DeepSeek-V4-Flash-0731](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash-0731)** 是 2026-07-30 發佈的官方版本（MIT 授權、附 DSpark 草稿層），304B 總參數、每 token 只啟用 13B 的稀疏 MoE。原生權重（專家 FP4、其餘 FP8）在 HF 上實測約 167GB——單機 128GB 明確塞不下。所以要靠第三塊拼圖：

**[Unsloth 的 UD-Q2_K_XL](https://unsloth.ai/docs/basics/dynamic-3.0-ggufs)** 動態量化 GGUF。它的思路是「不平均施暴」：對品質敏感的層（attention、shared expert 之類）保留較高位元，對佔了參數大宗的 MoE 專家權重壓到 2-bit 附近，整體體積才壓得進 128GB。對 13B 活躍參數的 MoE 來說，decode 每步要讀的權重本來就少，Q2 的代價沒有稠密模型那麼致命——這也是我一直敢在 MoE 上用低位元量化的原因。

## 2. 串起來：llama.cpp + pi

部署本身沒有太多戲：下載 `*UD-Q2_K_XL*` 的 GGUF shards，用 llama.cpp 起一個 OpenAI 相容的 server，就結束了。真正讓這件事變成「agent 自己解 AoC」的是 [pi](https://pi.dev/)——Mario Zechner 寫的極簡 coding agent（開發心路歷程見[他的長文](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)，repo 在 [earendil-works/pi](https://github.com/earendil-works/pi)）。它不像 OmO 那樣幫你把工作流全打包好（我前一陣子寫過[用一個月的心得](/blog/oh-my-opencode-one-month/)），pi 反過來：一個乾淨的 agent loop、你自己的模型、你自己的工具。指向本地 llama.cpp endpoint，狀態列右下角就是 `deepseek-v4-flash-0731 · medium`（medium 推理檔位）。

這種組合的好處是**沒有中間商賺 token**：AoC 一天天的題目要來回讀題、寫碼、跑測試，丟給 API 商一個月下來不便宜；本地跑只有電費，而且可以無腦開長 session。

## 3. 它是怎麼解 Day 3 的

截圖那天是 [Day 3「Lobby」](https://adventofcode.com/2025/day/3)：每行輸入是一串電池 joltage 數字，要從中選出固定 k 個數字（Part 1 k=2、Part 2 k=12）、保持順序、組成最大的數，再把每行結果加總。

agent 的解法是教科書級的貪婪：**對每個輸出位置，從「還留得出剩餘 k−1 個名額」的窗口裡挑最大數字，然後把指標移到它後面**。它自己寫的解說是：

> Solution approach: For each bank (line of digits), to maximize the number formed by selecting exactly k digits in order, I use a greedy strategy. For each output position, pick the maximum digit from the window that still leaves room for the remaining digits after it, then move past it.

結果：Part 1（k=2）總和 **17031**，Part 2（k=12）總和 **168575096286051**，兩者都對。這個貪婪為什麼是對的值得說一句：本質上是在求「字典序最大的 k 位子序列」，任意位置 i 的選擇若被更晚的較大數字替換，交換論證立刻給出反例不成立，所以不需要 DP、也不需要回溯——k=2 和 k=12 是同一個函式換參數而已，agent 的實作也確實是這樣共用了一份程式碼。

幾個我在旁邊看的細節比答案本身有意思：

- **它會自己驗收**：跑 `cargo test`、再跑 `cargo clippy --bin 03`，發現有 warning 時先判斷「這些是模板與 macro 展開的既有 warning，不是我的 code」，才 commit。模型有沒有把 lint 當自己的事，是 agent 體感品質的一大半。
- **context 用量**：狀態列顯示 context 累積到 131k（auto）。一整天的題目讀題、原始碼、測試輸出反覆進出，Q2 之下 1M context 的長尾能力就是這種用法的安全網。
- **GPU 大部分時間是閒的**：桌面系統監視器裡 GPU 佔用只有個位數百分比——13B 活躍參數的 decode 是記憶體頻寬綁定，273 GB/s 的 Spark 這種用法其實餵得動，瓶頸不在算力。
- **最誠實的一行**：`Note: auto-submission failed because the AoC session cookie is invalid/expired, not a code issue.`——agent 想自動提交答案，發現我的 session cookie 過期了，然後正確地把「這是 cookie 的問題、不是 code 的問題」寫進總結。我到現在還沒換 cookie。

## 4. Q2 到底行不行

誠實說：**在這個工作負載上，行**。AoC 的題目自帶明確規格與可驗證的測試（答案對不對當場知道），錯了就重跑，模型品質的抖動被回饋迴圈吸收掉大半。13B 活躍的 MoE 加上 unsloth 動態量化把容量塞進 128GB，這筆交易在 Spark 上是成立的。

但缺點要照實列：

- **Q2 不是萬能**：規格模糊、需要長鏈推理的題目（AoC 後段那種），低位元模型「想歪」的機率確實比較高；有測試當護欄時無所謂，沒有護欄的任務就得三思。
- **prefill 依舊是 Spark 的痛**：這點跟我[之前部署 GLM-5.3-Flash 的結論](/blog/exl3-glm-local-deploy/)完全一致——273 GB/s 餵 decode 很香，但灌入長 context 時的計算綁定階段就是慢。AoC 題目文字量小所以沒炸，換成要餵大文件的場景就沒這麼好過。
- **自動提交要自己顧**：session cookie 過期這種小事，agent 會正確診斷但不會替你修（也不該替你修）。

## 個人評價

單機 DGX Spark + Q2 級 MoE + pi 這個組合，我願意稱之為「窮人的 agentic 工作站」：一次投資之後，agent 任務的邊際成本趨近於零，而 DeepSeek-V4-Flash-0731 的 agentic 能力（它本來就是主打這個的官方版）撐得住讓它自己跑完一整個倉庫的工作流。AoC 只是個驗證場——題目有標準答案，成敗一目了然，非常適合當本地模型的體檢。接下來想讓它去打沒有標準答案的仗，那才是 Q2 的真正考場。

## 延伸閱讀

- [Advent of Code 2025 Day 3: Lobby](https://adventofcode.com/2025/day/3)
- [deepseek-ai/DeepSeek-V4-Flash-0731（Hugging Face）](https://huggingface.co/deepseek-ai/DeepSeek-V4-Flash-0731)、[NVIDIA NGC 模型頁](https://catalog.ngc.nvidia.com/orgs/nim/deepseek-ai/models/deepseek-v4-flash-0731)
- [Unsloth Dynamic 3.0 GGUFs 文件](https://unsloth.ai/docs/basics/dynamic-3.0-ggufs)
- [pi.dev](https://pi.dev/)、[earendil-works/pi](https://github.com/earendil-works/pi)、[Mario Zechner：What I learned building pi](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- [NVIDIA DGX Spark 硬體文件](https://docs.nvidia.com/dgx/dgx-spark/hardware.html)
- 站內：[在 2× DGX Spark 上部署 GLM-5.3-Flash-EXL3](/blog/exl3-glm-local-deploy/)、[用了整整一個月的 oh-my-opencode](/blog/oh-my-opencode-one-month/)
