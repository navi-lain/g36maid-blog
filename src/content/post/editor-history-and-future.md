---
title: "編輯器的歷史，現況與未來：從 Electron 到 Rust 與 GPU 驅動的 Zed"
description: "回顧 Atom 如何把 Web 技術帶進桌面、催生 Electron 並奠定 VS Code 的統治地位，再到 Atom 核心成員用 Rust 與自研 GPUI 框架從零打造的 Zed：120 FPS 的 GPU 渲染、MCP/ACP 原生 AI 整合，以及 1.0 之後的路線圖。"
pubDate: 2026-06-11T07:18:30+08:00
category: programming
tags: ["editor", "zed", "rust", "electron", "gpu", "ai", "agent", "效能", "編輯器"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1781133510
---

> 本文由 2026-06-11 的演講講稿〈編輯器的歷史，現況與未來〉（HackMD note）於 2026-10-27 擴寫而成。講稿中的現場互動（開場調查、迷因投影片）改寫為敘述；Atom／Zed 影片的章節時間戳與統計引用為演講當下資料，出處列於文末。原稿附圖為外連圖床，本文一概不引用，純文字呈現。

你主力用的編輯器是哪一款？這是任何一場編輯器主題演講最好的開場白——VS Code／VSCodium、vi/vim/neovim、JetBrains IntelliJ、Helix、Emacs（Doom、Spacemacs 一脈）、Sublime Text、Notepad++、Visual Studio，以及近兩年竄起的 Cursor／Windsurf／Antigravity，幾乎每一次現場統計都是這張名單的重新洗牌。

編輯器從來不只是工具，它是工程師身份認同的一部分。[Wikipedia 的 Editor war 條目](https://en.wikipedia.org/wiki/Editor_war) 記錄了 Vim vs Emacs 那場打了數十年的經典戰爭：雙方陣營的論戰、迷因與宗教式的忠誠，至今仍未真正停火。而另一條光譜上的 Notepad／Notepad++／Sublime Text 則代表了「輕量派」——開啟速度快、資源佔用低，但擴充性與生態系有明顯的天花板。

## 2025 Rust 開發者調查：版圖正在移動

[2025 State of Rust Survey](https://blog.rust-lang.org/2026/03/02/2025-State-Of-Rust-Survey-results/)（2026-03 發佈）裡有一段很有趣的統計，官方原文是這麼說的：

> 「作為一個有趣的小統計，我們詢問了大家在寫 Rust 時使用的工具。**Zed** 在受訪者的偏好中出現了驚人的躍升（**Helix** 緊隨其後）。如文字雲所示，具備 **Agentic（AI 代理）支援的編輯器也正在崛起**，從直方圖來看，它們似乎正在侵蝕 VS Code 和 IntelliJ 的用戶基礎。同時，我們也很高興能再次見到那 11 位還在使用 Atom 的開發者（嘿 👋！），並向那些堅守 Emacs 和 Vim（或其衍生版本）等經典編輯器的人致敬。」

這段話有三個訊號值得放大：其一，原生（非 Web 技術）編輯器 Zed 與 Helix 在 Rust 社群的支持度大幅躍升；其二，AI 代理編輯器正在實質地瓜分 VS Code 與 IntelliJ 的用戶；其三，Atom 在死後四年居然還有 11 個使用者——這 11 個數字背後，是一段值得完整回顧的歷史。

## Web 技術的黃金十年：Atom 與 Electron 的誕生

故事要從 GitHub 講起。參考 *Atom: The Editor That Accidentally Built Its Own Killer*（[YouTube](https://www.youtube.com/watch?v=QsTYFsmCL5E)）的梳理：GitHub 創辦人 Chris Wanstrath 最初的願景，是打造一個現代化且高度可駭客化（hackable）的開源編輯器。有別於學習曲線陡峭、依賴專有設定語言的 Emacs，他希望能用當時前端工程師已經熟悉的技術——HTML / CSS / JavaScript——把「自訂介面與擴充功能」的門檻降到地板上。

技術上，團隊經過多次迭代，把 Chromium 渲染引擎與 Node.js 結合，做出了名為 **Atom Shell** 的底層框架。這項創舉讓原本受限於瀏覽器沙盒的前端技術得以存取系統底層 API，轉化為真正的跨平台桌面應用程式——Atom Shell 後來更名為 **Electron**，成為獨立專案。

這套架構的 Trade-off 清單，十年後回頭看依然準確：

* **優勢**：跨平台（Linux、Windows、macOS）開發成本極低，迭代週期快，能大量重用網頁前端的程式碼與人才。
* **代價**：每一次啟動都要載入一整個 Chromium 引擎與 Node.js runtime。影片給的數字是：Atom 開啟一個 2MB 的純文字檔，可能耗時超過 3 秒並佔據 400MB 記憶體。

「Everything is hackable」的理念確實兌現了——代價是把一整顆瀏覽器塞進每個使用者的 RAM 裡。

## 微軟的降維打擊：VS Code 如何殺死 Atom

微軟當時正在開發基於瀏覽器的 Monaco 專案，隨後直接採用了開源的 Atom Shell（Electron），僅僅六個月就推出了 Visual Studio Code。同樣臃腫的 Electron 架構，VS Code 卻做到比 Atom 快上約四倍。關鍵在幾個工程決策：

1. **擴充套件隔離執行**：Extensions 運行於獨立進程、以非同步方式載入，不阻擋主 UI 執行緒。開啟 VS Code 的瞬間就能開始編輯，不必等套件載完——Atom 的啟動時間則被同步載入的套件拖垮。
2. **Monaco 引擎底層調校**：VS Code 的核心編輯器引擎在整合進 Electron 之前就已經過嚴格最佳化。
3. **二進位編碼傳輸**：內部狀態傳輸與資料儲存優先採用 binary 格式而非純文字，降低記憶體與儲存開銷。

終局發生得很快。2018 年微軟以 75 億美元收購 GitHub，初期官方承諾同時維護 Atom 與 VS Code，但隨著 VS Code 高頻迭代，開源社群的核心貢獻者持續靠攏，Atom 的 commit 頻率在半年內銳減 76%，最終於 2022 年被正式宣告終止維護（sunsetting）。

## Atom 的遺產：殺死它的東西都活在它蓋的地基上

Atom 的本體消亡了，但它催生的 Electron 架構如今支撐著 Slack、Discord，以及擊敗它的 VS Code。社群流傳的兩句話概括得最好：

> *"Every cursor blinking in VScode, carries Atom's ghost."*（VS Code 裡閃爍的每一個游標，都承載著 Atom 的靈魂。）
> *"So atom is the reason why every app uses 400mb of ram now. love that."*

第一句是致敬，第二句是控訴，兩句都是真的。另外那張經典迷因——「喜報：這台電腦上總共有 XX 個 Chromium 內核的應用」——每個人打開工作管理員都能自己複現一次。而 *"Microsoft acquiring Github was the beginning of the controversial timeline"* 則代表了另一派對收購後開源治理的長期質疑。

在開源文化裡，程式碼與技術理念總會透過競爭與 fork 在繼承者身上延續。Atom 死後的兩條血脈特別清楚：

* **Zed**：三位 Atom 核心貢獻者記取了 Electron 記憶體浩劫的教訓，改用 Rust 這種具備 low-level 控制力與極致多執行緒效能的語言，從零打造專注於效能的開源編輯器。
* **Cursor**：另一個團隊直接 fork VS Code 的開源版本，深度整合 AI 輔助，成為目前最受矚目的 AI 編輯器之一。

## Why Zed：回到 GPU 的路線

Zed 官方對自己的描述非常直白：

> 「大多數編輯器都是基於網頁瀏覽器建構的。**Zed 不是。** 它 100% 由 Rust 編寫，基於自研的 UI 框架 **GPUI**。每一幀都使用 **GPU 加速**。打下這個基礎花了數年時間，但一切都是值得的。因為我們擁有堆疊的每一層，這款編輯器的響應速度是大多數工具在物理上無法比擬的。你必須親身嘗試才能真正體會。」

核心技術選擇可以拆成三件事：

* **放棄 Web 框架，回歸底層渲染**：不用 HTML/CSS 排版 UI，改用 Rust 從頭編寫的 immediate mode GUI 框架 GPUI。
* **UI 元件直接交由 GPU 渲染**：Zed 的[官方部落格](https://zed.dev/blog/videogame)形容它是「用 Rust 和 GPU 以 120 FPS 渲染使用者介面」——把打字跟手感拉到電競螢幕的更新率等級。他們明確說過：這是打造電玩的方式，不是打造編輯器的方式。
* **AI 與協作原生設計**：深度結合 MCP（Model Context Protocol）與 ACP（Agent Client Protocol），AI 代理不是外掛，而是一等公民。

### From 0 to Zed 1.0

Zed 走到 1.0 的關鍵里程碑（依官方年終回顧與部落格）：

| 年份 | 里程碑 | 參考 |
| --- | --- | --- |
| 2024 | 開源化與生態拓展（年度總結） | [zed.dev/2024](https://zed.dev/2024) |
| 2024 | Open Source Release | [blog](https://zed.dev/blog/zed-is-now-open-source) |
| 2024 | Zed on Linux | [blog](https://zed.dev/blog/zed-on-linux) |
| 2024 | Zed AI Beta | [blog](https://zed.dev/blog/zed-ai) |
| 2024 | Remote Development | [blog](https://zed.dev/blog/remote-development) |
| 2025 | Edit Predictions（自研模型 Zeta） | [blog](https://zed.dev/blog/edit-prediction) |
| 2025 | Native Git Support | [blog](https://zed.dev/blog/git) |
| 2025 | Agentic／「最快的 AI 編輯器」 | [blog](https://zed.dev/blog/fastest-ai-code-editor) |
| 2025 | Debugger 登場 | [blog](https://zed.dev/blog/debugger) |
| 2025 | Agent Client Protocol（ACP） | [docs](https://zed.dev/acp) |
| 2025 | Zed on Windows（全平台制霸） | [blog](https://zed.dev/blog/zed-for-windows-is-here) |
| 2025 | 1.0 發佈與年度總結 | [zed.dev/2025](https://zed.dev/2025) |

過去一年新增的能力包括：內建 git 工具（各種大小的 diff 檢視、stage、不離開編輯器就能 commit）、強大的多緩衝區編輯（單一視圖處理多個檔案），以及一流的遠端開發支援。

## 挑戰與對手

Zed 不是沒有弱點，誠實列出來：

* **Extension 生態豐富度不足**：擴充套件需要以 Rust（或 WASM）開發，相較 VS Code 的 JS 生態，第三方貢獻者門檻更高、存量更少。
* **原生 GUI 的佈局彈性上限**：GPUI 的佈局與樣式能力相較 HTML/CSS 仍有天花板。
* **GPUI 與 Zed 耦合度過高**：目前無法作為獨立 crate 供其他專案使用——社群一直有人敲碗，但官方還沒把它抽出去。

競爭對手兩面夾擊：一邊是「老大哥」VS Code／VSCodium／Code - OSS 及其龐大生態，另一邊是 AI 新貴 Cursor／Windsurf／Antigravity。2025 Rust 調查顯示的「agentic 編輯器侵蝕 VS Code 用戶」這件事，對 Zed 同樣是機會也是警訊——AI 整合已經是所有編輯器的標配戰場，差別只在整合的深度與架構。

## Roadmap After 1.0

官方目前進行中的項目：**Instant Sharing**（透過連結讓任何人查看/編輯你的程式碼）、**Code History as Context**（檢視程式碼演進歷史及影響它的對話脈絡）、**Async Collaboration**（非同步在同一個 branch 上協作）、**Skills**（讓 Zed Agent 運用特定技能完成任務）、**Edit Tool Improvements**（更輕量、具備精準取代與刪除能力的編輯工具）、**UI for Terminal Agents**（在 sidebar 運行平行的 terminal agents）。

下一步計畫：**ACP Improvements**（讓 Claude Agent 與 Codex 的互動更流暢）、**Zed on the Web**（跨裝置隨時開啟專案）、**Plan Mode**（讓 AI agent 在修改程式碼前先擬定計畫）、**Notebooks**（互動式資料視覺化與操作介面）、**Hands-Free Coding**（語音指令與內建通話，打造無障礙工作流）。

可以看到路線圖的重心明顯壓在「協作」與「AI 代理」兩條線上——這正是它與前輩們競爭的差異化戰場。

## 結論

把這三十年的歷史攤開來看，編輯器的演化是同一組張力的反覆循環：**可駭客性 vs 效能**，**生態 vs 純粹**。Emacs 用 ELisp 贏得可駭客性，輸給了學習曲線；Atom 用 Web 技術贏得生態，輸給了記憶體；VS Code 用工程紀律證明 Electron 也能快，卻在 AI 時代面對架構級的競爭者；而 Zed 押注的是「擁有堆疊的每一層」——Rust、GPUI、GPU，一路到底。

歷史的弔詭在於：Atom 建立的框架殺死了 Atom 自己，而殺死 Atom 的團隊成員又回頭用另一套技術棧挑戰 VS Code。在開源世界裡，死亡從來不是終點，只是 fork 的另一種形式。至於未來，AI 代理協議（MCP/ACP）正在成為新的「編輯器之戰」戰場——這次比的不是渲染引擎或擴充套件數量，而是誰能讓 agent 與人類在同一個緩衝區裡合作得最順。

最後是講者自己的實務心得：參與 Zed 這類解決底層瓶頸的開源專案，是推動現代工具鏈演進最直接的方式。社群的成長靠的不是大公司的路線圖，而是一個個願意讀 GPUI source code、回報 issue、送 PR 的人。如果你對編輯器的下一個十年有興趣，這正是最好的進場時機。

## 參考資料

* Wikipedia：[Editor war](https://en.wikipedia.org/wiki/Editor_war)
* Rust Blog：[2025 State of Rust Survey results](https://blog.rust-lang.org/2026/03/02/2025-State-Of-Rust-Survey-results/)
* YouTube：[Atom: The Editor That Accidentally Built Its Own Killer](https://www.youtube.com/watch?v=QsTYFsmCL5E)
* Zed Blog：[Video games (GPU 渲染 UI)](https://zed.dev/blog/videogame)、[Zed is now open source](https://zed.dev/blog/zed-is-now-open-source)、[Zed on Linux](https://zed.dev/blog/zed-on-linux)、[Zed AI](https://zed.dev/blog/zed-ai)、[Remote Development](https://zed.dev/blog/remote-development)、[Edit Prediction](https://zed.dev/blog/edit-prediction)、[Git](https://zed.dev/blog/git)、[Fastest AI code editor](https://zed.dev/blog/fastest-ai-code-editor)、[Debugger](https://zed.dev/blog/debugger)、[Zed for Windows](https://zed.dev/blog/zed-for-windows-is-here)
* Zed 年度回顧：[2024](https://zed.dev/2024)、[2025](https://zed.dev/2025)；[ACP 文件](https://zed.dev/acp)
