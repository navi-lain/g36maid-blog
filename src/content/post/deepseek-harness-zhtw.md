---
title: "一天完成 DeepSeek-Harness 繁體中文在地化：zhtw-mcp 與「把軟體殺掉」考據"
slug: deepseek-harness-zhtw
description: "DeepSeek 官方開源 agent harness 的繁中在地化一天到位：文件英中繁、Web UI 台灣用語、/zh-TW/ 路由與翻譯 pipeline，全程靠 jserv 的 zhtw-mcp 把關教育部標準。文末併入台灣軟體用語考據——為什麼我們說「把軟體殺掉」。"
pubDate: 2026-08-14T22:33:11+08:00
tags: ["ai", "github", "i18n", "mcp", "opensource", "traditional-chinese"]
category: ai
source: "facebook"
sourceType: "連結分享"
sourceTimestamp: 1786717991
draft: false
---
> 本文由 2026-08-14 FB 貼文於 2026-11-01 擴寫，並併入 2026-06-07 的軟體用語草稿。

八月中的一天，我把 [DeepSeek](https://www.deepseek.com/en/harness) 官方開源的 DeepSeek-Harness 做完了繁體中文在地化，全程使用 [Jim Huang](https://github.com/jserv)（jserv）維護的 zhtw-mcp。文件英中繁三語、Web UI 換上台灣用語、`/zh-TW/` 路由與翻譯 pipeline 都到位，變更放在[我的 fork](https://github.com/G36maid/deepseek-harness)，歡迎官方與 i18n 貢獻者接手。這篇記錄做了什麼、用什麼工具，以及過程中反覆出現的一個問題：台灣的軟體用語，到底是怎麼長成現在這個樣子的？

## DeepSeek-Harness 是什麼

[DeepSeek-Harness](https://github.com/deepseek-ai/deepseek-harness) 是 DeepSeek 於 2026 年 8 月中開源的 agent harness，採 MIT 授權，主打「Everything is a Plugin」——以 Cordis 的外掛架構為基底，桌面版與 Web UI 都能用，`npx @deepseek-ai/dsh web` 一行就能把 Web UI 開在本機。換句話說，它和 Hermes、OpenCode 同屬「模型之外的整套工具鏈」這個賽道，而我先前在[本地部署 GLM](/blog/exl3-glm-local-deploy/) 與[ agent 越獄實測](/blog/openai-agent-hf-escape/)的文章裡談過：harness 決定了模型實際上能做什麼、看到什麼。對繁體中文社群來說，harness 的介面語言就是日常接觸 AI 的第一線，在地化值得認真做。

## 一天之內做了什麼

據原文 FB 貼文，這次在地化涵蓋幾個層次：文件改為英、中（繁）對照；Web UI 的介面字串全部換成台灣用語；網站加上 `/zh-TW/` 路由；再搭一條翻譯 pipeline，讓後續上游新增的字串可以半自動進翻譯流程。一天能完成，關鍵不是打字快，而是工具選對了。

## zhtw-mcp：教育部標準進 MCP

[zhtw-mcp](https://github.com/sysprog21/zhtw-mcp) 是一個專門管繁體中文的語言 linter，透過 Model Context Protocol 接進 AI 編程助理，在中國大陸用語漂進成品之前就攔下來。它強制執行三項台灣官方標準：教育部《重訂標點符號手冊》修訂版的標點、《國字標準字體》的字形，以及以 OpenCC 的 TWPhrases/TWVariants 資料集為基礎的兩岸詞彙轉換——超過 1100 條詞彙規則和 15 條大小寫規則直接編譯進二進位檔。遇到難以機判的詞，它會反過來透過 MCP Sampling 問執行中的 AI 助理，不需要額外的 API key。

實際體感是：讓模型寫繁中時，最常見的破功不是簡體字，而是「信息」「視頻」「默認」「軟件」這種簡繁轉換正確、但台灣沒人這樣講的詞。傳統翻譯 pipeline 靠字串表，zhtw-mcp 則是把語感檢查做成一個工具，直接嵌進 agent 的工作迴圈。

## 「把軟體殺掉」：台灣軟體用語考據

這是六月初另一則貼文的主題，趁這次一併收進來：各位有聽過「把軟體（殺）掉」這種說法嗎？我從小就很常聽到這種解除安裝的用法，而且感覺意義上更接近「清理乾淨」——不是把捷徑刪一刪，而是連殘留的設定檔、快取一起送走。

仔細想，這個「殺」有跡可循。Unix 世界裡 `kill` 本來就是標準操作，`kill -9` 送出訊號終止 process，工程師口語說「把 process 殺掉」完全自然；工作管理員裡的「結束工作」、命令列的「終止」，骨子裡都是同一個 kill。當「殺掉 process」是日常，「把軟體殺掉」（徹底移除）很可能是這個語感外溢到整個軟體的結果——殺，意味著乾淨俐落、不留活口，正好對應解除安裝時「清理乾淨」的期待。

這也說明了為什麼機器翻譯永遠做不出這種用語：它不在任何官方詞彙表裡，卻是台灣工程文化真實的一部分。Web UI 在地化如果只做到「软体→軟體」的字串替換，就會錯過這一層；這正是我認為 zhtw-mcp 這類工具、加上人工校訂仍然無法被完全取代的原因。

## 小結

這次貢獻規模不大，但兩件事讓我覺得值得寫下來：一是 DeepSeek-Harness 這種一線開源專案，從 repo 建立到有繁中介面，距離其實只有一個週末；二是繁體中文在地化的品質，愈來愈取決於工具鏈——教育部標準、OpenCC 詞表、MCP linter——而不是單靠人力逐字校對。至於「殺掉軟體」這類只有在地人才懂的用語，就留給每一輪人工校訂當作小小的樂趣吧。
