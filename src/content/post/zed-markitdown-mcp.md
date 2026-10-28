---
title: "自製 Zed 的 MarkItDown MCP Server：讓文字 Agent 也讀得懂 PPTX 和 PDF"
description: "為 Zed Editor 寫了一個包裝 Microsoft MarkItDown 的 MCP server 擴充：設計思路、Zed 擴充機制、安裝設定、Python 3.14 踩雷與誠實評價。"
pubDate: 2025-09-17T17:58:04+08:00
category: ai
tags: ["zed", "mcp", "markitdown", "rust", "ai", "文件轉換"]
draft: false
source: "facebook"
sourceType: "連結分享"
sourceTimestamp: 1758103084
---

> 本文擴寫自 2025-09-17 的 FB 隨筆。

手上常見的痛點：Zed 的 Agent Panel 很好用，但它是個**純文字** agent——你叫它分析一份簡報、讀一份 PDF 報告，它只會攤手。除非改用 multimodal model（貴、而且不一定是你想用的那顆），否則 PPTX、PDF、圖片這類資料根本進不了 context。

解法其實很樸素：與其讓模型「看懂」多媒體，不如**在進 context 之前先把東西轉成 Markdown**。文字 agent 就能處理了。所以我在 2025-09-16 一個晚上寫了 [zed-mcp-server-markitdown](https://github.com/G36maid/zed-mcp-server-markitdown)（MIT），把 Microsoft 的 [MarkItDown](https://github.com/microsoft/markitdown) 包成 Zed 的 MCP server 擴充，目前還在 Beta。

## MarkItDown：文件轉 Markdown 的瑞士刀

[MarkItDown](https://github.com/microsoft/markitdown) 是 Microsoft AutoGen 團隊 2024 年底開源的 Python 工具，設計立場很明確：輸出**不是給人看的高保真轉換**，而是給 LLM 消費的文字管線——保留標題、列表、表格、連結這些結構，其他裝飾通通丟掉。這個取捨對我的用途剛剛好。

它支援的格式相當貪心：PDF、Word、Excel、PowerPoint、圖片（EXIF + OCR）、音訊（語音轉錄）、HTML、CSV/JSON/XML、ZIP、YouTube 連結……官方還提供 [markitdown-mcp](https://github.com/microsoft/markitdown/tree/main/packages/markitdown-mcp) server，只暴露**一個 tool**：

```text
convert_to_markdown(uri: str) -> str
```

`uri` 可以是 `http:`、`https:`、`file:`、`data:` 任何一種。一個 tool 吃所有格式，這種 API 表面積我非常欣賞——agent 不用學一堆參數，丟 URI 進去拿 Markdown 出來就對了。

## 1. 擴充本身其實很薄：Zed 的 context_servers 機制

Zed 從 [2025 年初宣布支援 MCP](https://zed.dev/blog/mcp) 以來，MCP server 有兩種掛法：直接寫在 `settings.json` 的 `context_servers`，或包成[擴充](https://zed.dev/docs/extensions/mcp-extensions)。後者很貼心——裝完擴充就自動幫你把 server 跑起來。

做法是在 `extension.toml` 註冊 `[context_servers.mcp-server-markitdown]`，然後在 Rust 的 `context_server_command()` 裡回傳啟動指令。我的實作整個流程是：

1. 讀設定裡的 `package_version`（預設 `latest`）；
2. 用 Zed 擴充 API 的 `npm_package_installed_version()` / `npm_install_package()` 裝好 npm 套件；
3. 回傳 `{node_binary_path} node_modules/.bin/markitdown-mcp-npx` 這條指令。

也就是說：真正的 server 是 [xkiranj 的 markitdown-mcp-npx](https://github.com/xkiranj/markitdown-mcp-npx)（把官方 Python server 包成免 Docker 的 npm wrapper），轉換邏輯全是 Microsoft 上游的。我的 Rust 程式碼只做「把這條命令鏈正確接起來」這件事——寫擴充不等於寫功能，這是件好事。

## 2. 安裝（目前是 dev extension）

```sh
git clone https://github.com/G36maid/zed-mcp-server-markitdown.git
cd zed-mcp-server-markitdown
cargo build --release
```

然後在 Zed：`Extensions → Install Dev Extension → 選這個目錄`。想固定版本可以在 `settings.json` 加：

```json
{
  "context_servers": {
    "mcp-server-markitdown": {
      "settings": { "package_version": "latest" }
    }
  }
}
```

前置需求只有 Node.js（Zed 會自己處理 npm 安裝）和網路。裝好後重啟 Agent Panel，`convert_to_markdown` 就會出現在工具列表，記得到 Agent Profiles 裡確認它沒被關掉。

## 3. 實際用法：直接在對話裡丟東西給它

裝完之後的使用體驗就是自然語言，不用任何特殊指令：

```text
請分析這份 PDF 報告，摘出三個重點。
```

```text
把這個 Excel 轉成 Markdown，列出前五筆資料。
```

```text
總結這篇文章的重點：<URL>
```

Agent 會自己呼叫 `convert_to_markdown`，拿到轉好的 Markdown 再回答。實際用下來最順的是**網頁和 PDF**：抓網頁做摘要幾乎無腦；PDF 看文件結構，純文字型 PDF 效果很好，重度排版的（多欄、表格嵌圖）就會有損傷——這是所有「轉 Markdown」路線的宿命。

## 4. 踩雷筆記

後來陸續有人回報（我自己在 Arch 上也撞到）的坑，都整理進 [README 的 Quick Fix](https://github.com/G36maid/zed-mcp-server-markitdown#%EF%B8%8F-python-314-quick-fix) 了：

- **Python 3.14 / Arch 系環境**：上游的 `onnxruntime` 依賴在 Python 3.14 上會卡住，安裝逾時。解法是繞過 npm wrapper，改用 `uvx` 直接指定 Python 3.13：

  ```json
  {
    "context_servers": {
      "markitdown": {
        "command": "uvx",
        "args": ["--python", "3.13", "markitdown-mcp"]
      }
    }
  }
  ```

- **首次啟動會下載模型**：MarkItDown 用 [Magika](https://github.com/google/magika) 做檔案類型偵測，第一次啟動下載 Python 環境 + 模型可能超過 Zed 的 60 秒啟動 timeout。先自己在終端跑一次 `uvx --python 3.13 markitdown-mcp`，等它連線成功再 Ctrl+C，重啟 Zed 就正常。
- **安全邊界要想清楚**：官方文件明說這個 server 不做驗證、以你當前帳號的權限跑，`file:` URI 可以讀你帳號能讀的**任何檔案**。拿來餵本地可信的 agent 沒問題，但別把它綁到 localhost 以外的介面。

## 個人評價：好用，但要知道它不是魔術

優點說完了，誠實列缺點：

- **轉換品質天花板在上游**。OCR 和語音轉錄是「能用」等級，不是「精準」等級；複雜排版的 PDF 出來的 Markdown 需要人工檢查。
- **Token 消耗很兇**。一份 50 頁的簡報轉成 Markdown 照樣是幾萬 token，agent 拿到的是整份文件，不是摘要。大檔案要先自己想辦法切。
- **還在 Beta**。目前以 dev extension 安裝，還沒走 Zed 官方 extension store 的發佈流程；Zed 官方也[計畫改用官方 MCP registry](https://zed.dev/docs/extensions/mcp-extensions) 來分發 MCP server，之後會再調整。

但整體而言我很滿意這個架構：轉換交給 MarkItDown、包裝交給 npx wrapper、Zed 擴充只負責接線，每一層都站在別人的肩膀上，自己只寫了最薄的一層 glue code。對「文字 agent 吃多媒體」這件事，轉 Markdown 目前還是最省錢省事的解。

repo 在這裡：[G36maid/zed-mcp-server-markitdown](https://github.com/G36maid/zed-mcp-server-markitdown)，有興趣試用或開 issue 都歡迎，順手的話給個星星 (◕‿◕✿)

## 延伸閱讀

- [Zed blog: The Context Outside the Code](https://zed.dev/blog/mcp) — Zed 官方對 MCP 擴充的定位說明
- [Zed docs: MCP Servers](https://zed.dev/docs/ai/mcp) — `context_servers` 設定與自訂 server
- [microsoft/markitdown](https://github.com/microsoft/markitdown) — 轉換引擎本體（MIT）
- [Microsoft AutoGen Team 的 markitdown-mcp](https://github.com/microsoft/markitdown/tree/main/packages/markitdown-mcp) — 官方 MCP server
- 站內相關：[用 AUR 思維替 Zed Editor 做非官方 FreeBSD port](/blog/zed-freebsd-port/)、[cargo-zigbuild 交叉編譯 zed-remote-server](/blog/cargo-zigbuild-freebsd-cross/) —— 兩篇同樣是替 Zed 補齊它沒官方支援的環境
