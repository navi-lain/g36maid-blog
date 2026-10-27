---
title: "「牛來」現形記：匿名前沿模型 Ox Alpha 的指紋偵查與認領結局"
description: "2026-08-20 匿名前沿模型 Ox Alpha 突然在 OpenRouter 與 OpenCode 上線：1M 上下文、原生多模態、免費一週。本文整理社群 36 小時內的 tokenizer 與影片編碼器指紋分析，以及 8/26 智譜認領 GLM-5.3-Flash 的結局——指紋哪些猜對了、哪些猜錯了，誠實對帳。"
pubDate: 2026-08-21T15:29:23+08:00
category: ai
tags: ["ai", "模型", "glm", "openrouter", "agent", "指紋分析"]
draft: false
source: "facebook"
sourceType: "相片"
sourceTimestamp: 1787297363
---

> 本文由 2026-08-21 的 Facebook 貼文於 2026-10-27 擴寫而成。原稿寫於 Ox Alpha 還是匿名模型的當下，社群只知道它「大概是誰」；如今結局已揭曉，所以這篇會做兩件事：還原當時的指紋偵查過程，以及事後對帳——**哪些指紋猜對了、哪些猜錯了**。文中嚴格區分「社群指紋推測」與「官方確認」兩級證據，不確定的地方會明說。

一個沒有署名、沒有論文、沒有官方頁面的前沿模型，突然掛在 API 目錄上免費給你用——你敢不敢把 production 流量灌進去？

2026 年 8 月 20 日，OpenRouter 和 OpenCode 同時上架了一款匿名模型 **Ox Alpha**，掛在 OpenRouter 的 "Stealth" provider 底下。中文圈看著它的宣傳海報（一頭站在冰原城堡上的黃牛玩偶），直接暱稱它「**牛來**」。定位是高效 Coding + 長時程 Agent + 真實生產環境，核心規格在當時是頂規：1M token 上下文、原生多模態（文字 + 圖片 + 影片輸入）、最大輸出 131,072 tokens。OpenCode 宣布免費開放約一週、日服務容量上看 100T tokens；OpenRouter 這邊輸入輸出價格直接掛零。

![「OX IS COMING」宣傳海報：冰原城堡上站著一頭藍角黃牛玩偶，配黑色牛頭旗——中文圈「牛來」綽號的由來](/media/2026-08/122173115660958395.jpg)

![OpenCode 與 OpenRouter 的 X 貼文截圖：100 萬上下文、多模態、零資料保留、寬鬆速率限制，OpenRouter 頁面標示 context 1.05M、Released Aug 20, 2026](/media/2026-08/122173116242958395.jpg)

## 指紋偵查：36 小時內的三路證據

沒人簽名，於是社群自己動手驗。這場偵查最漂亮的地方在於**用了三條互相獨立的證據鏈**，而不只是「感覺很像 GLM」。

**第一路：tokenizer 指紋。** 做法很樸素——把同一組字串（英文、中文、程式碼、emoji 混打）丟給候選模型，數回傳的 prompt token 數。Tokenizer 是模型身分證：同一個詞表切出來的 token 數必然一致，而且難以偽裝。結果（[@sushsrinivasan 的探針串](https://www.digitalapplied.com/blog/openrouter-stealth-model-census-who-they-turned-out-to-be)：10 個候選模型、11 組探針）顯示 Ox Alpha 與 GLM 系列 **11/11 全數吻合**，其他任何實驗室最多過 4 關。另一組測試（[@aitrackerbot](https://www.aimodeling.com/en/news/slug/ox-alpha-glm-5-3-stealth-zhipu)）用 25 組多樣 prompt 對比，得到更精確的結論：Ox Alpha 的 token 數**始終等於 GLM-5.3 加上一個恆定的 +75 offset**——這個常數很像 OpenRouter 或上游包裝層多塞的一段隱藏 system prompt，不影響身分判定。

**第二路：影片編碼器指紋。** 這招比較少見：對 Ox Alpha 餵四段受控影片，逐 token 對帳。結果與智譜自家的 GLM-5V-Turbo **完全一致**，而且同時命中三個獨立的設計選擇：幀率無關的取樣（FPS-invariant sampling）、約每秒 147 token 的時長計費、逐幀解析度縮放。一個 token 數對上可能是巧合，三個獨立設計選擇同時對上，機率上很難是別家。

**第三路：API 行為指紋。** 故意傳不合法的 `reasoning_effort` 參數，看它怎麼報錯。Ox Alpha 的反應和 GLM-5.3 高度接近：強制開啟 thinking、只接受 `low / high / max`。整組 API profile——1,048,576 context、131,072 max output、temperature=1、top_p=0.95——也和 GLM-5.3 幾乎重合。

## 排除法：為什麼不是小米 MiMo

當時另一個熱門猜測是小米 MiMo。但同一輪測試把它排除了：Ox Alpha 在**所有路由上都拒絕音訊輸入**（行為像 GLM-5V 家族），而 MiMo v2.5 原生接受音訊——端點行為對不上。Qwen 3.8 Max 和 GLM-4.6V 也被同一套指紋流程排除了。於是到 8 月 22 日為止，各路分析的收斂結論是：**智譜 GLM-5.x 系列的多模態版本**，置信度從 70–80%、90% 到轉述中 Ben Davis 的約 99% 都有——方向一致，數字各憑方法。

## 證據等級對帳：指紋對了什麼、錯了什麼

| 主張 | 證據等級 | 結局 |
| --- | --- | --- |
| 背後是智譜 GLM 系列 | 社群指紋推測（tokenizer/影片/API 三路獨立） | ✅ 官方認領證實 |
| Tokenizer 與 GLM-5.3 幾乎一致 | 社群指紋推測（多組獨立測試） | ✅ 與認領結果相符 |
| 具體變體是 GLM-5.3V（視覺版） | 社群推測 | ❌ 猜錯：是重新訓練的 **GLM-5.3-Flash** |
| 免費期結束前後公布身份 | 過往 stealth 模型慣例歸納 | ✅ 8/26 認領，免費期後 |

必須誠實記一筆：原稿（8/21）押的是「GLM-5.3 多模態變體 5.3V」，因為當時公開版 GLM-5.3 是純文字模型（8 月 14 日發布），大家自然猜「補上視覺的那版」。結果智譜交出來的是 **GLM-5.3-Flash**——一個 320B 總參數、約 18B 激活的 MoE，Sparse Attention + Linear Attention 混合架構，用 30T token 多模態預訓練資料重訓的 Flash 級模型。**指紋方法論全對（家族判定正確），但變體名猜錯了**——指紋能告訴你「是誰家的」，告訴不了你「是哪一顆」。事後看，這也是指紋分析的固有極限：它比對的是 tokenizer 和推理行為，而這些在同家族的不同變體之間本來就可能共享。

## 結局：8 月 26 日，官方認領

8 月 26 日，智譜（Z.ai）正式認領：Ox Alpha 就是 **GLM-5.3-Flash**，並且**同日發布 MIT 授權的開源權重**。`stealth/ox-alpha` 這個 ID 隨之退役（現在呼叫它只會得到零端點），路由改為 `z-ai/glm-5.3-flash`，價格落在約 $0.15/M 輸入、$0.50/M 輸出——Artificial Analysis 評測 57 分，只比旗艦 GLM-5.3 的 60 分低 3 分，成本約十分之一。

值得記錄的生態事實：有人對 OpenRouter 歷年 stealth 模型做了完整普查（[digitalapplied 的 census](https://www.digitalapplied.com/blog/openrouter-stealth-model-census-who-they-turned-out-to-be)），截至 Ox Alpha 共 14 例，歷史 13 例中 7 例獲得正式認領——而且**每一次認領都來自第一方管道（平台或實驗室自己開口），指紋分析從未是認領的機制**。指紋的價值不在「逼出答案」，而在答案揭曉前讓開發者能做出有依據的判斷。

## 想回去測的人：兩個注意事項

1. **隱私條款兩條路不一樣。** 當時社群流傳「Zero Data Retention」，但仔細看是分裂的：OpenCode Zen 路線是真零保留；OpenRouter 的 stealth 頁面措辭是「prompts and completions are retained by the provider and are not used for training」——**保留但不用於訓練**，兩者不是同一個承諾（第三方報導甚至指出 OpenRouter 當時的 EULA 授權將資料用於訓練）。要跑敏感工作負載前，先把兩邊條款讀清楚。
2. **`stealth/ox-alpha` 已死。** 現在要用，model ID 是 `z-ai/glm-5.3-flash`，權重在 Hugging Face 上以 MIT 授權公開，想自己跑也有得選。

## 結論

回頭看，「牛來」這一週最有趣的不是榜單分數，而是它示範了一整套**匿名模型的歸因流程**：tokenizer 指紋鎖定家族、影片編碼器指紋補上第二獨立證據、API 行為指紋收尾、排除法砍掉熱門錯誤答案——最後由官方認領收束，且過程中指紋結論被大體證實、細節被修正（5.3V → 5.3-Flash）。對使用者來說，教訓很務實：免費的前沿模型可以玩，但「哪條路線、什麼條款」比「它可能是誰」更值得先看。至於下一頭牛什麼時候來？歷史基數說：大概率一個月內，而且大概率又會被三天內驗明正身。

## 延伸閱讀

- [OpenRouter Stealth Models: Who They Turned Out To Be](https://www.digitalapplied.com/blog/openrouter-stealth-model-census-who-they-turned-out-to-be) — 14 個 stealth 模型的完整普查
- [Ox Alpha fingerprints point to GLM-5.3](https://www.aimodeling.com/en/news/slug/ox-alpha-glm-5-3-stealth-zhipu) — 指紋分析細節與結論邊界
- [GLM-5.3-Flash 發布：旗艦能力，十分之一成本](https://cloud.tencent.com/developer/article/2733239) — 認領後的模型本體分析
- [神秘免费模型 Ox Alpha：一场 24 小时的指纹侦探战](https://x.com/lizikk_zhu/article/2090837612978397653) — 中文圈的偵查實錄
