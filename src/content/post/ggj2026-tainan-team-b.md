---
title: "在審查之下說話：GGJ 2026 台南 Team B 的四十八小時"
description: "Global Game Jam 2026 主題揭曉是 Mask。我們在台南，用一份 HackMD 筆記、一套自創的二十幾個符號的語言，和一個假的 Telegram 聊天室，做了一款關於敏感詞的遊戲。"
pubDate: 2026-01-31T09:43:34+08:00
category: acg
tags: ["game-jam", "ggj2026", "game-dev", "rust", "vue", "conlang"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1769823814
---

> 本文擴寫自 2026-01-31 的 HackMD 協作筆記（GGJ2026 Tainan Team B），擴寫於 2026-10-28。時間線依原筆記與 [project-babel repo](https://github.com/G36maid/project-babel) 的 git 歷史整理。

## Jam 才開始一天，就掛上 🔴

GGJ 2026 的主題揭曉是 **Mask（面具）**。我們那份共筆的第一行是狀態列：`Status: 🟡 Prototyping / 🔴 Crunch Time`——時間 1 月 31 日上午 9 點 43 分。Jam 明明 30 號才開始，我們卻已經同時標記了 prototyping 和 crunch。事後回看，這個狀態列意外地誠實：整場 jam 我們從來沒有離開過這兩個狀態的疊加。

當天早上 10 點 38 分 repo 開出來，四分鐘後第一個 commit 進來——`docs: Add GGJ2026 Tainan Team B docs`。第一個被收進版本控制的東西，是那份共筆本身。

## 主題的三條線

發想夜的素材很雜。《Papers, Please》教我們「關口審查」的壓迫感，《Chants of Sennaar》教我們「用有限符號學一種語言」的樂趣，然後有人丟了迴形針那支〈如何科學的消滅敏感詞〉的影片，共筆裡只寫了一句話，但那句話決定了整個遊戲：

> 現實世界總是充滿了敏感詞。

腦力激盪自然分成兩組：一組往 factory/logic 去（台積電、光刻機、Zachtronics 式的方塊工廠），一組往 narrative/crypto 去（語言、匿名、防火牆、敏感詞）。前者後來長成「方案 A：Project Litho」——四人分飾 CMYK 四個通道，在輸送帶上對 voxel cube 做光刻與上色，良率達標過關，甚至認真討論要在機台旁邊放乖乖。後者就是「方案 B：Project Babel」——四個玩家身處四個國家，各有各的禁字表，訊息過伺服器時被審查遮蔽，你要在殘缺的對話裡把語言跟禁詞清單一起解出來。

共筆的待辦清單裡，Phase 1 六個項目只有一個打了勾：「決定方案 B」。

## 為什麼是聊天室，不是光刻廠

這是整場 jam 最關鍵的一個決定，值得深寫。選 B 不是因為故事比較感人，而是盤點完 48 小時的產能後的誠實結論：

- **Cartridge Ready** 這個 diversifier 要 10MB 以內。Babel 是純文字＋符號，天生命小；Litho 的 voxel 美術光是資源量就先輸一半。
- **語音輸入**是 Babel 的殺手級賣點，而瀏覽器的 Web Speech API 不用訓練任何模型。共筆裡那個 trick 我現在想還是覺得聰明：把自創語言的詞設計成「英文語音辨識容易命中的發音」——北約音標風格的雙音節詞，`SpeechRecognition` 直接辨成英文再 mapping 回符號，零成本繞過自訂語音模型的深坑。
- 最現實的一條：共筆裡有一整節「緊急切割方案」——如果剩 4 小時做不完，砍連線、砍 3D、砍關卡。那份切割清單其實是為方案 A 寫的遺書。選 B，等於一開始就把最容易炸的 3D 產線整條拆掉。

## 二十幾個符號的語言，跟一份禁詞 JSON

選了 Babel，馬上掉進語言設計的坑。我們把《Chants of Sennaar》五個民族的詞彙表整份整理進共筆——信徒的 SVO、戰士的 OSV、詩人靠「非-」否定詞造反義詞、煉金術士的完整數字系統——然後照樣畫出自己的符號語言：每個符號對應一個人造語音單字。共筆裡一下寫 25 個符號、一下寫 26 個，等到真正落成 JSON 的時候是 28 個 normal 詞（你、我、他、講、什麼、是、不……），外加成對的 censored 清單：`["capital", "communism"]`、`["democracy", "oligarchy", "dictatorship"]`、`["freedom", "limit"]`。這份清單本身就是遊戲的世界觀：國家不禁止「人」跟「神」，禁止的是「資本」跟「共產」、「自由」跟「限制」。語言的邊界就是思想的邊界，這句話在 jam 裡是可以玩的機制。

另一個後來證明是對的決定：**不做遊戲感 UI**。面具這個主題，最好的詮釋是把遊戲偽裝成一個你不會多想的東西——我們直接找了 Telegram UI 的元件庫，做出一個以假亂真的聊天室介面，審查就發生在這個「看起來只是通訊軟體」的介面裡。git log 裡那一排 `ChatHeader`、`MessageBubble`、`TelegramLayout` 的 commit，就是在打造面具本身。

還有一個誠實的落差要記下來：共筆的 tech stack 那節只寫了兩個字——「gleam」。最後 repo 的語言統計，Rust 佔了八萬多 bytes 排第一，前端是 Vue。計畫與實作之間隔著 48 小時，這在 jam 裡是常態，寫下來只是提醒自己共筆不是承諾。

## 最後一天的時間線

git log 不會說謊。2 月 1 日（皆 UTC+8）：上午還在補 send_note 與玩家筆記追蹤，中午 12 點前後在調審查機制與 log，下午 14:41 victory system 進來，14:57 SVG 符號渲染完成，15:06 release 文件，15:38 前端排版，16:38 `disable shadow ban and enable sender ban`——對，連「該禁發言者還是該陰影禁言」都是截止前一小時才吵出結論的。整場 jam 73 個 commit，全部擠在 1 月 31 日到 2 月 1 日。

jam 結束一週後的 2 月 8 日，我們又回來了一波：把遊戲邏輯抽成 `GameRules` trait、伺服器與具體 ChatRoom 解耦、`UserAction` 拆成 System/Game 兩軸、最後一個 PR 用 `Arc<FilterConfig>` 換掉 `Box::leak` 修記憶體洩漏。評審看到的遊戲在 2 月 1 日就凍結了；但我們自己知道，那個遊戲真正的形狀是那個禮拜才整出來的。

## 後記

做這款遊戲有個繞不開的諷刺：為了講一個反審查的故事，我們得先認真寫出一個好用的審查引擎——替換邏輯、遮蔽規則、ban 的粒度，一項都不能含糊。共筆裡那句「現實世界總是充滿了敏感詞」是發想夜的起點，最後變成了我們親手實作過的東西。面具戴久了會長進臉裡；還好這一次，面具是我們自己做的，也是我們自己拆的。

### 延伸閱讀

- [project-babel（GitHub）](https://github.com/G36maid/project-babel)
- [GGJ 2026 主題 Mask 與官方反思](https://globalgamejam.org/news/reflections-ggj-2026-theme)
- [GGJ 2026 Diversifiers 官方清單](https://globalgamejam.org/global-game-jam-2026-diversifiers-have-arrived)
- [Chants of Sennaar（Steam）](https://store.steampowered.com/app/1931770/Chants_of_Sennaar/)、[Papers, Please（Steam）](https://store.steampowered.com/app/239030/Papers_Please)
