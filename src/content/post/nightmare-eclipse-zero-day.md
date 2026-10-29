---
title: "「噩夢日蝕」Nightmare Eclipse——接連公開 Windows 零日漏洞的人講出自己的故事"
description: "前微軟員工 Abdelhamid Naceri 公開自己的身份與被解雇的經過：一封語焉不詳的解約信、一場敗訴的勞動訴訟、12 個零日漏洞，以及一套徹底失靈的通報制度。"
pubDate: 2026-09-13T23:47:31+08:00
category: security
tags: ["漏洞", "windows", "zero-day", "microsoft", "security", "vulnerability-disclosure"]
draft: false
source: "facebook"
sourceType: "相片"
sourceTimestamp: 1789314451
---

> 本文由 2026-09-13 的 Facebook 貼文草稿（轉譯自 International Cyber Digest 的 X 貼文）於 2026-10-29 擴寫而成，並補上後續追蹤報導與事實查核。原稿的情緒語氣盡量保留；事實層面以查核結果修正。

你聽過有人用一整排零日漏洞跟一家公司吵架嗎？2026 年 4 月起，一個自稱「Nightmare Eclipse」（又作 Chaotic Eclipse、MSNightmare）的帳號開始接連公開 Windows 與微軟產品的零日漏洞，而且專挑 Patch Tuesday 前後發布，讓微軟幾乎沒時間反應。2026 年 9 月中，這個人決定講出自己的故事——他是前微軟員工，本名阿卜杜勒哈米德·納塞里（Abdelhamid Naceri）。

## 故事的起點：一封語焉不詳的信

納塞里並不是第一次出現在資安新聞裡。早在 2020 年他就透過 Trend Micro 的 Zero-Day Initiative 通報過 Windows 提權漏洞（當時微軟甚至沒有出修補），2021 年又回報過能讓系統無法開機的 Kaspersky 漏洞。換句話說，他曾經是走「正當管道」的模範通報者。

但根據他貼出的信件，MSRC 工程副總裁湯姆·加拉格爾（Tom Gallagher）在 2024 年 9 月 20 日與他碰面，隔天以 email 告知他被解雇，理由是「公司發現潛在的資安事件」、他「對外部第三方分享漏洞資訊」——語焉不詳，具體細節至今沒有公開。納塞里說，加拉格爾還當面警告會把他列入黑名單、給他極差的評價，讓他再也找不到工作。

![Tom Gallagher 於 2024-09-20 寄給 Naceri 的信件截圖：信中稱公司發現「潛在資安事件」、其行為危及公司與客戶，要求一週內回覆 HR 的解約提案](/media/2026-09/122175872768958395.jpg)

![同一組貼文裡的另一張圖片：Nightmare Eclipse 的頭像](/media/2026-09/122175872744958395.jpg)

這種時候大家通常會說：換家公司不就好了？但納塞里沒有歐盟護照，而他當時距離拿到歐盟永久居留權只差幾個月。微軟完全沒有通融那幾個月，而是用一個至今沒有說清的理由將他開除；訴訟過程中微軟提出 55,000 歐元外加一年薪水作為和解條件，要求他撤告——他要的其實只是一個正式的解釋，和留在德國的可能性。

## 敗訴、以及被撕掉的敘事

納塞里在德國以「不當解雇」控告微軟。這場官司最終以敗訴收場：據他自己的說法，德國勞動法院維持了解雇決定，他幾乎沒有拿到補償，還背上超過 20 萬美元的律師費。他說這場糾紛對他的精神狀態造成嚴重影響，目前正在精神醫院接受治療；他在 2026 年 7、8 月的部落格文章裡寫過自殺念頭。

這裡必須誠實標注證據等級：以上過程全部來自納塞里的單方敘事（以及那張信件截圖），微軟從未公開證實全部細節。而且他自己就示範過為什麼要打折——他曾宣稱微軟反過來告他，後來承認「MS never sued, I lied」。單方說法加上說謊紀錄，不構成定論；但微軟也確實始終拒絕說明「資安事件」的具體內容，雙方各選了一條最不透明的路。

## 12 個零日，與一套失靈的制度

從 2026 年 4 月到 9 月，納塞里累積了 12 份零日報告：BlueHammer、RedSun、UnDefend、YellowKey、GreenPlasma、MiniPlasma、RoguePlanet、GreatXML、LegacyHive、ShieldBreak、ShieldCrash，以及最後的 BigDiskBuster。標的橫跨 Defender 提權（ShieldCrash 可以在微軟修掉 ShieldBreak 之後幾天內再度繞過）、BitLocker（YellowKey、GreenPlasma 能碰受保護的磁碟），到塞滿磁碟空間讓 Defender 無法更新簽章的 DoS。微軟修了其中一部分——ShieldBreak（CVE-2026-69414）、RoguePlanet、YellowKey、GreenPlasma、MiniPlasma——其餘至今沒有官方修補。

他的存在感大到微軟在 5 月的 MSRC 部落格文中威脅要聯合「全球執法機關」打擊獨立漏洞公開，被資安社群罵翻之後又收回。

> 「如果我沒有把靈魂傾注在那份工作裡、熬過無數荒謬的不眠之夜，我早就把它放下了……」——納塞里

他曾經深愛微軟。他對 ISMG 說，公開身份「與微軟無關，我知道微軟無論發生什麼都不會改變立場。我只是想找一份新工作。」

## 一點感想

我不會說他接連釋出零日的做法是「正確」的——未通報就公開 exploit，風險直接落在裝了 Windows 的每個人身上，這筆帳不該由微軟獨扛。但這個案例戳破了一個舒服的假設：協調式漏洞揭露（CVD）是「正確管道」，走不通就是研究者的錯。實際上管道是公司維護的，公司可以刪帳號、不回信、不付錢，然後把「沒有走管道」當成解雇的理由。當管道的守門人同時是糾紛的當事人，公開就成了被逼出來的最後手段。

另一個更冷的事實是：解雇可以撤銷存取權，撤不掉專業知識。帳號關掉只需要幾分鐘，但一個人對產品弱點的理解不會隨著 offboarding 過期。一家公司怎麼對待離職的安全研究員，本身就會決定這些知識最後流向哪裡——微軟這堂課的學費，是 20 萬美元、12 個零日，和一個住進精神醫院的人。

## References

- SecurityWeek: [Nightmare Eclipse Drops New Microsoft Defender Exploit After Revealing Identity](https://www.securityweek.com/nightmare-eclipse-drops-new-microsoft-defender-exploit-after-revealing-identity/)
- Bank Info Security: [Nightmare Eclipse Reveals Name, Story Behind MS Zero-Days](https://www.bankinfosecurity.com/nightmare-eclipse-reveals-name-story-behind-ms-zero-days-a-32907)
- BleepingComputer: [New Windows Defender zero-day blocks Microsoft antivirus updates](https://www.bleepingcomputer.com/news/security/new-windows-defender-zero-day-blocks-microsoft-antivirus-updates/)
- International Cyber Digest（原稿來源）: [X 貼文含解約信截圖](https://x.com/IntCyberDigest/status/2099104399867928703)
- Microsoft MSRC: [A Shared Responsibility: Protecting Customers Through Coordinated Vulnerability Disclosure](https://www.microsoft.com/en-us/msrc/blog/2026/05/a-shared-responsibility-protecting-customers-through-coordinated-vulnerability-disclosure)（後續於 [X 上收回](https://x.com/msftsecresponse/status/2061293718942908925)）
- Hive Security: [Nightmare Eclipse Revealed: What a Former Employee Can Still Do to Your Security](https://hivesecurity.gitlab.io/blog/nightmare-eclipse-former-employee-insider-risk/)
- MSRC 修正諮詢: [CVE-2026-69414](https://msrc.microsoft.com/update-guide/en-US/advisory/CVE-2026-69414)
