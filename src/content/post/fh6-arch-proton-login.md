---
title: "為了在 Arch Linux 上登入 Forza Horizon 6：五種 Proton、3.2GB log，與一個被 Microsoft 標記的 token"
description: "Forza Horizon 6 在 Arch Linux 上登入失敗，換了五種 Proton、刪 prefix 重建、重裝 vcredist、撈 3.2GB log 逐行 grep，最後發現兇手是 Microsoft 帳號被標記為可疑的 device-auth token——去 account.live.com 重掃 QR 就好了。"
pubDate: 2026-06-25T23:43:23+08:00
category: linux
tags: ["linux", "arch", "proton", "wine", "steam", "遊戲"]
draft: false
source: "facebook"
sourceType: "相片"
sourceTimestamp: 1782402203
---

> 本文由 2026-06-25 的 Facebook 短文（相片）於 2026-10-27 擴寫而成。時間軸、版本名稱與結論皆出自當日原稿；擴寫補充的 Wine/Proton 背景說明屬一般性知識，非當日另行實測的數據。環境：Arch Linux、Steam + Proton。

## 症狀：遊戲裝好了，帳號登不進去

Forza Horizon 6 買了、裝了、能跑——然後卡在登入。微軟帳號的授權流程轉了又轉，轉完就是進不去。這種「遊戲本身沒事、帳號服務有事」的症狀在 Windows 上偶爾也會遇到，但玩家的直覺永遠是先懷疑自己桌上的環境：Linux + Proton，嫌疑最大。

於是我開始了一天的排除法。

## 第一輪：換了五種 Proton

Proton 版本輪詢是 Linux 玩家的標準反射動作，這次一口氣換了五種：

- CachyOS 內建的 5 月版
- CachyOS 6 月版
- Valve 的 Proton Experimental
- Proton Hotfix
- GE-Proton（社群版）

每一種都重啟、重登、看轉圈，然後繼續轉圈。五種全滅。

這裡必須誠實記一筆：**換 Proton 本身是合理的排查步驟**——不同版本的 Wine stub 覆蓋度、steamapi 介接、網路相關 DLL 都有差異，登入類問題換版本確實偶爾會好。但當五種版本行為完全一致時，正確的推論不是「再找第六種」，而是「問題不在 Proton」。當下的我沒有接受這個推論。

## 第二輪：刪 prefix 重建、重裝 vcredist

接著是環境層的懷疑。把該遊戲的 compatdata prefix 整個刪掉重建，讓 Proton 從零生成新的 `drive_c`，再手動把 vcredist 重裝一輪——理論上可以排除「prefix 汙染」與「runtime 套件損壞」兩種可能。

結果一樣。prefix 是新的，vcredist 是好的，登入是壞的。

## 第三輪：3.2GB 的 Proton log，逐行 grep

終於想起來開 log。Proton 的除錯慣例是設 `PROTON_LOG=1`，它會把該遊戲整個 Wine session 的輸出傾成一個文字檔；遊戲跑一次，log 漲到 **3.2GB**。然後就是逐行 grep 的夜晚：auth、login、xsts、token、error，關鍵字換了一輪又一輪，看著 Wine 在畫面後面不斷重試網路請求、不斷失敗。

log 裡確實有東西反覆出現：`netprofm`。看起來非常有嫌疑。

## netprofm semi-stub：一條漂亮的紅鯡魚

`netprofm` 是 Windows 的「網路清單服務」（Network List Manager），遊戲常用它判斷「現在到底有沒有連上網路」。Wine 對它的實作是 semi-stub——介面在、能回話，但行為不完整。看到這裡很容易拼出一個完美故事：

> 「Wine 的 netprofm 是半殘的，所以遊戲以為自己離線，所以登入服務不啟動。」

這個故事結構完整、技術上講得通、還有 log 當證據——**然而是錯的**。netprofm 的確是 semi-stub，這點沒錯；但它不是這次登入失敗的原因。它只是整個系統裡最容易被怪罪的那個環節，一條標準的紅鯡魚。跟它大眼瞪小眼了一個下午，毫無進展。

## 真正的解法：帳號被標記「可疑」，不是遊戲壞掉

繞了一大圈回到原點後才去想一個一直被忽略的對象：**Microsoft 帳號本身**。

到 [account.live.com](https://account.live.com) 開「活動記錄」（Activity），發現帳號的 device-auth token 已經進入了「可疑」狀態——Microsoft 的風險偵測把這台裝置的登入判定為需要重新驗證，於是遊戲端的授權請求每次都被擋下。在 Windows 上，系統通常會跳通知引導你重新驗證；在 Linux 上沒有那層引導，你只會看到遊戲轉圈。

解法樸素到令人火大：

1. 開 `account.live.com` / Activity
2. 點「保護您的帳戶」
3. 重新掃 QR code 驗證身分
4. 通過

然後遊戲裡的登入一次就過了。**五種 Proton、prefix、vcredist、3.2GB log，沒有一樣是兇手。**

## 十五分鐘雲端同步，與本週已結束的 Festival Playlist

登入成功還沒結束：等雲端存檔同步，整整十五分鐘。同步完，遊戲端出現了「選擇存檔」——本機有一個 12 分 47 秒前的存檔，雲端有一個 9 天 7 小時前的存檔（135 台車、XP 1,863,446、點數 2,192,399），二選一。

![終於登入後的「選擇存檔」畫面：本機存檔（ARCHFW13，12 分 47 秒前，282 KB）對上雲端存檔（G36ARCHPC，9 天 7 小時前，7 MB、135 台車、XP 1,863,446、點數 2,192,399），選項是本機存檔／雲端存檔／離線遊玩。畫面上緣的 MangoHud 與 Steam 介面證明這確實是 Arch Linux 上的 Proton session——FPS 30，反正這是選單畫面。](/media/2026-06/122166645698958395.jpg)

選了雲端，進到遊戲的瞬間，發現**本週的 Festival Playlist 已經結束了** 🙂🙂🙂

折騰一整天換來的獎品，是眼睜睜看著本週賽季獎勵從指縫間溜走。至少現在能玩了。拜託別再壞了。

## 結論：排查順序的教訓

把這次的時間軸攤開，教訓很清楚——**排查順序錯了**。正確的順序應該是：

1. **先查帳號側**：`account.live.com` / Activity 看有沒有安全性挑戰、token 被標記。微軟的風險偵測不會在 Linux 上主動通知你，它只會默默擋你的授權。
2. **再看服務側**：遊戲登入服務本身（Xbox Live / XSTS）有沒有掛。
3. **最後才是本機環境**：Proton 版本、prefix、log。

本機環境最容易折騰、最有「做了很多事」的感覺，但在這種「五種 Proton 行為完全一致」的案例裡，它恰恰是最不可能的答案。3.2GB 的 log 教會我的事：log 裡最顯眼的錯誤（netprofm semi-stub）未必是原因，它可能只是系統裡最誠實的自白者——「我的網路狀態回報能力很差」——但擋你登入的是帳號風控，不是它。

下次再有「Linux 上遊戲登入失敗」，第一個動作是開瀏覽器登 account.live.com，不是開終端機。
