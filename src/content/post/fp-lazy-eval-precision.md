---
title: "「電腦能儲存的最高精度是多少？」——一個 FP 腦患者的無限精度答案"
description: "修 Storage Systems 時被問『電腦能儲存的最高精度是多少』，深陷 FP 的我答了『無限』。幾年後認真展開這個極端理論派答案：符號計算、惰性求值與 computable reals 為什麼站得住腳，以及當年沒算的 space leak 帳單。"
pubDate: 2026-02-19T02:50:24+08:00
category: essay
tags: [haskell, lazy-evaluation, floating-point, symbolic-computation, exact-arithmetic]
draft: false
source: facebook
sourceType: 貼文
sourceTimestamp: 1771440624
---

本文擴寫自 2026-02-19 的 Facebook 貼文，原稿是修 Storage Systems 時的一段課堂問答。以下保留原始對話與當時的語氣，再把那個「極端理論派」答案認真展開一次，最後誠實回頭算帳。

## 原始對話

之前修張原豪教授的 Storage Systems 時，他突然拋出一個問題：「你覺得電腦能儲存的最高精度是多少？」

當時我正好深陷 Functional Programming 課程的坑，腦子裡全是 Lambda Calculus，於是給出了一個極端理論派的思路：

「是無限。如果我們不存『數值（value）』，而是存『算式（expression）』，再搭配 Lazy Evaluation（惰性求值），那我們就能在讀取時輸出任意想要的精度。」

現在回想起來，教授當時的表情大概是在想：「這學生是在供三小……？」

幾年後重想這個答案，我發現它在理論上真的站得住腳——只是實務上的代價，當年的我完全沒算。以下先替當年的自己辯護，再回來認賠。

## 先立術語：value、expression 與 thunk

- **value**：已經算好、放在記憶體裡的東西。IEEE 754 的雙精度浮點數就是一種 value：64 bits、其中 52 bits 是尾數，精度上限是寫死在表示法裡的。
- **expression**：一段「怎麼算」的描述，例如 `sqrt 2`。它本身不是數，是尚未執行的計算。
- **lazy evaluation（惰性求值）**：運算式的求值延到「第一次被需要」才發生，而且只算到被需要的那一格。實作上，尚未求值的運算式在 heap 上是一個 **thunk**——一個等著被觸發的計算閉包。

## 為什麼「無限」在理論上成立

Haskell 是預設惰性的語言，`Integer` 是任意精度整數、`Rational` 是精確分數。把兩件事疊起來，就得到課堂上那句狂話的技術版本：

```haskell
-- 存的是算式不是數值：這一行執行完什麼都沒算
approx = iterate (\x -> (x + 2 / x) / 2) 1  -- sqrt 2 的牛頓法迭代
-- 用到第 n 個元素才真的算前 n 步；要幾位精度，取幾位
```

`iterate` 建出一條**無限串流（infinite stream）**：理論上沒有盡頭的列表，但因為惰性，記憶體裡只存在被**強迫求值（force）**過的片段。John Hughes 的經典論文 *Why Functional Programming Matters* [1] 正是用無限串流加牛頓法示範：把一個數表示成「位數的無限列表」，每迭代一次正確位數多一位，要 1000 位就取前 1000 位。**精度從此是讀取端的選項，不是儲存端的限制**——這就是當年那句「讀取時輸出任意精度」的意思。

這也不是嘴砲，這條路在數學軟體裡是日常：

- **符號計算（symbolic computation）**：以 symPy [2] 為代表。`sqrt(2)` 在電腦裡就是那個符號本身，化簡、微分、代數操作全程精確，直到最後一行 `evalf(50)` 才浮點化成 50 位。
- **代數數（algebraic number）**：`sqrt(2)` 這種「整係數多項式的根」可以被完整精確地表示與比較——symPy 的 `CRootOf` 就是這種物件。
- **Computable reals / exact real arithmetic**：更極端的一支，把實數定義成「可計算到任意精度的函數」。iRRAM（C++）[3]、Haskell `numbers` 套件的 `CReal` [4] 都是真實存在的實作。

對照組很清楚：IEEE 754 浮點把無限多的實數壓進 64 bits，`0.1 + 0.2 != 0.3` 不是 bug，是這種表示法的宿命。存 expression 而不存 value，等於把誤差從系統裡趕出去——代價只是，永遠不算完。

## 誠實回頭檢討：教授問的其實是實務題

當年的我講得理直氣壯，但這個答案在 Storage Systems 的課堂上，代價至少有三層：

1. **thunk 積累與 space leak**。惰性最大的坑就是著名的 space leak：消費串流的程式若不嚴格（strict），thunk 只堆不算，記憶體曲線一路爬升。教科書級案例是 `foldl` 對長列表積累百萬層 thunk，得換成嚴格版 `foldl'` 才救得回來 [5]。「先不算」的自由，是用 heap 上的債換來的。
2. **精確算術每一筆都是錢**。任意精度整數與有理數的每個運算，都比硬體浮點慢一到數個數量級；要維護 1000 位正確位數，中間過程的成本是實打實的 CPU 時間。符號計算系統沒有人拿來跑迴圈，不是沒有原因。
3. **教授問的可能根本不是這題**。在 Storage Systems 的語境下，「最高精度」更可能的標準答案是：磁碟上的一個 byte 就是 8 bits，該做的是挑對表示法——float64 的 15–17 位有效數字、定點數、或乾脆存字串。我拿 lambda calculus 回答了一個儲存格式設計題，難怪教授的表情寫著「這學生是在供三小」。

所以結論大概是：那個答案在理論上是對的——exact real arithmetic 學界認真做了三十年 [3]，就是它的學術版本；但作為儲存課堂上的回答，它像把一台圖靈機搬去回答「你家硬碟多大」。理論滿分，實務掛科。

幾年後我仍然慶幸自己曾這樣答過。只是現在開口前，會先多問一句：「老師，你問的是數學上的精度，還是錢跟時間上的精度？」

## References

1. John Hughes, *Why Functional Programming Matters* (1989) — <https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf>
2. symPy — <https://www.sympy.org/>
3. iRRAM: Exact Real Arithmetic in C++ — <https://irram.uni-trier.de/>
4. Haskell `numbers` package（`Data.Number.CReal`）— <https://hackage.haskell.org/package/numbers>
5. Haskell Wiki: Foldr Foldl Foldl' 與 space leak — <https://wiki.haskell.org/Foldr_Foldl_Foldl'>
