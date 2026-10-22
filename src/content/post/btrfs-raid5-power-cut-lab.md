---
title: "Btrfs RAID5 斷電、拔碟、write hole 實測：把三顆 SSD 折騰到底的實驗紀錄"
description: "在 Arch Linux（kernel 6.14）上用 2×NVMe + 1×SATA SSD 建 Btrfs RAID5，實測斷電、拔碟、傳輸中掉碟、bitrot 與 write hole，最後用 fio 對比 md-raid bitmap：Btrfs RAID5 隨機寫只有 md-raid 的八分之一。"
pubDate: 2025-06-11T05:14:02+08:00
category: linux
tags: ["btrfs", "raid", "mdadm", "fio", "linux", "檔案系統"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1749590042
---

> 本文由 2025-05-20 至 05-25 的 SSDI 課程實驗紀錄（2025-06-11 HackMD note）於 2026-10-26 擴寫而成。所有數字皆為單次實測值：Arch Linux、kernel 6.14.6-zen、btrfs-progs 6.14、Ryzen 7 7800X3D，陣列為 2×NVMe SSD + 1×SATA SSD（各 1TB，CT1000MX500SSD1）。結論僅代表這張桌子上的硬體，不應外推成通用保證。

你的 RAID5 陣列敢不敢在 rsync 跑到一半時直接拔掉一顆碟？官方文件說 Btrfs RAID56「不該用於 production，僅供測試評估」——`mkfs.btrfs` 也會當場警告你。但警告歸警告，實際斷電、拔碟之後到底會發生什麼，文件只說「corner case 不可靠」，沒說多不可靠。這份實驗就是為了把這個問號換成數字。

## 建陣列：safe 與 unsafe 兩種姿勢

Btrfs 官方建議 RAID5/6 **只給資料用，metadata 退回 mirror**：

```bash
# safe：data raid5、metadata raid1c3
sudo mkfs.btrfs -f -d raid5 -m raid1c3 /dev/nvme1n1 /dev/nvme2n1 /dev/sda
# unsafe：連 metadata 都用 raid5（本次實驗主體）
sudo mkfs.btrfs -f -d raid5 -m raid5 /dev/nvme1n1 /dev/nvme2n1 /dev/sda
```

三顆 931.51GiB 碟組成 2.73TiB 陣列，data ratio 1.50——RAID5 的 N-1 容量效率在 Btrfs 上沒打折。掛載用慣例參數 `noatime,compress=zstd`，之後所有測試都以 `btrfs device stats` 全 0 作為健康基準。

## 斷電測試：沒有任何問題

rsync 灌入約 248GB（109,452 個檔案，5m04s，約 815MB/s）中途切電源。重開機後 `btrfs filesystem show -d` 三顆碟都在、`device stats` 全部為 0，結論只有一行：**沒有任何問題**。CoW 加 checksum 的組合對「寫到一半斷電」這種事本來就不太怕——舊資料不會被覆蓋，新資料沒寫完就不算數。

## 拔碟與 replace：degraded 掛載六分鐘回血

模擬碟死是直接 `wipefs -a /dev/sda` 再拔線。degraded 掛載後 `filesystem usage` 如實報出 `Device missing: 931.51GiB`，dmesg 也警告 `devid 3 ... is missing`。換新（其實是接回同一顆）只花一條指令：

```bash
sudo btrfs replace start 3 /dev/disk/by-id/ata-CT1000MX500SSD1_2302E69CE85A /mnt
```

dmesg 記錄 `dev_replace from <missing disk> (devid 3) ... started` 到 `finished` 相隔約 690 秒，iostat 顯示 sda 穩定寫入 159MB/s。replace 完接著跑 scrub，三顆碟 `no errors found`。

## 傳輸中拔碟：停幾秒，然後自己好起來

最狠的一題：rsync 正在跑時拔掉 SATA 線。現象是**傳輸只停頓了幾秒，然後若無其事繼續跑完**——客戶端只感覺到速度瞬間下修，248GB 照樣傳完。dmesg 那邊可熱鬧了：`ata4: SATA link down`、`I/O error, dev sda` 連發、`bdev /dev/sda errs: wr 14` 一路爬升到 17、super block 寫入失敗三次。重新開機接回後，sda 的 stats 停在 `wr 27, rd 3, flush 8`，而 scrub 的帳單是：

```
Error summary:    super=3 verify=12172 csum=13814411
  Corrected:      13826583
  Uncorrectable:  0
```

**1380 萬個錯誤全部自動修復、0 個不可修復**，最後 `balance` 重定位 109/109 個 chunk（9m42s）收尾。這是整個實驗最漂亮的一筆：掉碟當下資料靠剩餘兩顆碟的 parity 撐住，重連後靠 checksum 找出所有過期區塊逐一重建。

## bitrot 與 dd 誤操作：誠實說，測了個寂寞

bitrot 模擬是對 sda 開窗亂寫：`dd if=/dev/urandom of=/dev/sda seek=100000 count=1000 bs=1k`，結果 `corruption_errs` 維持 0；更暴力的整碟 `dd of=/dev/sda` 也「什麼都沒發生」。必須誠實标注證據等級：這兩次 dd 的寫入位置大概率落在**未配置的空間**，根本沒踩到任何有 checksum 保護的 extent——所以「Btrfs 自動修好了」這個結論並不成立，成立的只是「沒寫到東西就沒事」。要正確驗證 bitrot 得用 `btrfs corrupt block` 之類直接改 metadata 的手段，這是本次實驗的盲點。

## write hole：重現失敗，而且是有理由的

把「斷電」與「拔碟」疊加（寫入中斷電、重來、再拔碟），怎麼試都無法重現 write hole。這跟時間點吻合：kernel 6.14 收了 [Btrfs RAID56 write-intent bitmap](https://lwn.net/Articles/902402/)——RAID56 寫入前先把 bitmap flush 到所有裝置的 1MiB 偏移處，斷電重掛載時對 dirty stripe 跑 scrub 復原，正好補上 [官方文件](https://btrfs.readthedocs.io/en/latest/btrfs-man5.html) 承認的那個「未追蹤哪些 stripe 是新是舊」的洞。但同樣要誠實：「沒重現」不等於「證明修好」，只能說在這套環境與這幾種折騰方式下 write hole 沒有現形。

## 加一顆二手 HDD：翻車實錄

05-24 加入二手 Seagate ST1000LM048。`btrfs device add` 後 balance 跑到一半 sda 離線、重開機換 sdb missing，靠 `btrfs device remove missing` + 重複 balance（39m39s，121 chunks）救回。05-25 scrub 直接宣判：sda `read_errors: 12,768,640`、uncorrectable 38,024，`device stats` 寫入錯誤 **6,108,593** 次——原稿的評語是「幹我想說不對勁，又他嗎離線了」，然後「正式死亡」。全閃 RAID 裡混入慢速 HDD 還有第二個問題：所有寫入瓶頸都被那顆碟卡住。結論很樸素：**別把二手 HDD 混進 SSD 陣列**，無論是速度還是壽命。

## fio 對比：Btrfs RAID5 的隨機寫是 md-raid 的 1/8

最後用同一組 fio 參數（4k、4 jobs、60s、`direct=1`）對比三種陣列：

| 工作負載 | Btrfs RAID5 | md-raid5 | md-raid5 + bitmap |
| --- | --- | --- | --- |
| randwrite | **15.3 MiB/s**（3,913 IOPS） | 119 MiB/s（30.6k） | 102 MiB/s（26.1k） |
| seq write | 134 MiB/s（34.4k） | **181 MiB/s（46.4k）** | 147 MiB/s |
| randrw 70/30 | 99.8 / 42.9 MiB/s | 117 / 50.2 MiB/s | 107 / 45.8 MiB/s |
| rsync 147.6GB | **3m09s** | 3m21s | 3m49s |

```bash
fio --name=randwrite_test --directory=/mnt/storage \
    --rw=randwrite --bs=4k --size=512M --numjobs=4 \
    --runtime=60 --time_based --group_reporting --direct=1
```

差距集中在一個地方：**4k 隨機寫**。Btrfs RAID5 只有 15.3MiB/s（3,913 IOPS），是 md-raid 119MiB/s 的八分之一，p99 延遲也被拖到 3.7ms、長尾到 240ms。原因在寫入路徑：Btrfs 的 CoW 讓小寫幾乎總是觸發 RAID56 的 read-modify-write，而 md-raid 可以靠 page cache 合併成 full-stripe 寫。開了 write-intent bitmap 的 md-raid 隨機寫再掉 14%（119→102 MiB/s）——這是 resilver 速度換來的保險費。有趣的是大檔 rsync 反而是 Btrfs 最快（779 vs 729 vs 643 MB/s）：順序大寫走 compress=zstd + full stripe，優勢全回來了。

## 結論

一週折騰下來的誠實評價：Btrfs RAID5 的**復原行為**比文件上那句警告漂亮得多——斷電無傷、傳輸中拔碟只停幾秒、1380 萬個錯誤全數自癒、replace 六分鐘搞定。它輸的是**隨機小寫效能**，以及官方至今仍掛著「僅供評估」的態度。如果負載是遊戲庫、多媒體這種順序大檔，Btrfs RAID5（data）+ raid1c3（metadata）在 kernel 6.14 之後已經是可以認真考慮的選項；如果是資料庫或 VM image，隨機寫八倍的差距會直接勸退。至於 metadata 用 raid5 的 unsafe 姿勢，就留給實驗室吧。
