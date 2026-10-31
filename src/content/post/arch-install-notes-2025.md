---
title: "Arch Linux PC 安裝筆記 (2025)"
description: "7800X3D + RTX 3070 Ti 的 Arch 安裝全記錄：archinstall 逐項選擇、Hyprland + nvidia-open、Timeshift + grub-btrfs 快照救援鏈，以及 Btrfs 儲存池在 raid5 上踩雷後改走 RAID0/RAID1 的誠實筆記。"
pubDate: 2026-06-21 15:27:28 +0800
category: linux
tags: ["arch", "archinstall", "btrfs", "hyprland", "nvidia", "linux", "grub", "steam"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1782026848
---

> 本文擴寫自 2026-06-21 的 HackMD 安裝筆記（原稿性質：装机速記），於 2026-10-31 整理成文；版本與連結以查證當下為準。

六月幫自己組了台新桌機：AMD Ryzen 7 7800X3D 配 RTX 3070 Ti，系統碟一顆 512GB NVMe，另抓三顆 1TB SSD（兩 NVMe + 一顆 SATA MX500）要湊 Btrfs 儲存池。目標很明確：遊戲（Steam + Proton）和開發共用一台 Arch，而且要能優雅地搞砸——也就是快照救援要能從 GRUB 直接開。這篇是安裝筆記的擴寫整理，記錄每個選擇的原因和後來發現踩的雷。

## 用 archinstall 裝，不丟臉

手動裝 Arch 我會，但重複勞動沒有樂趣。archinstall 現在的引導式流程已經很成熟（[官方 repo](https://github.com/archlinux/archinstall)），而且它的選項都能匯出成 JSON 設定檔（`--config` / `--disk-layout`），之後重裝可以整份重放。逐項記錄我的選擇：

```text
鏡像地區      Taiwan
附加套件庫    multilib      # Steam 需要 32-bit 函式庫
磁碟          系統碟整顆 wipe，格式 Btrfs
開機管理員    GRUB
Swap          啟用（zram）
時區          Asia/Taipei，啟用 NTP
網路          複製 ISO 的 NetworkManager 設定
聲音          PipeWire
帳號          建立我的帳號並加入 wheel 群組（sudo）
額外套件      git vim tmux btop fastfetch
```

幾個值得說的理由：

- **Btrfs + 預設子卷結構**：archinstall 會問要不要用預設子卷配置，答 yes 的話它建 `@`、`@home`、`@log`、`@pkg`、`@.snapshots`（見 [archinstall 源碼的預設清單](https://github.com/archlinux/archinstall/blob/master/archinstall/lib/disk/disk_menu.py)），掛載選項帶 `compress`。這個佈局剛好就是 Timeshift + grub-btrfs 想要的形狀，後面會用到。
- **multilib**：不勾這個 Steam 裝不起來，`lib32-*` 全靠它。
- **GRUB 而非 systemd-boot**：純粹為了 grub-btrfs 的開機進快照選單。
- Profile 選了 **Hyprland** 當視窗管理器，顯示驅動選 **nvidia-open**（後述）。草稿裡寫的 `bashtop` 現在都改用 btop 了；`pacman -S nerd-fonts` 之後我補裝了思源黑體/宋體的 `-tw-` 變體和 [paru](https://github.com/Morganamilo/paru) 當 AUR helper。

## 安裝後設定：Hyprland、輸入法、dotfiles

Hyprland 的設定我另外寫過心得（[Ricing 與生產力](/blog/linux-ricing-productivity/)），這裡只講安裝層面的事。中文輸入法用 Fcitx5 + [McBopomofo](https://github.com/openvanilla/fcitx5-mcbopomofo)——從 macOS 時代就習慣的組字邏輯，AUR 直接有打包：

```bash
paru -S fcitx5 fcitx5-mcbopomofo fcitx5-configtool
paru -S adobe-source-han-serif-tw-fonts adobe-source-han-sans-tw-fonts
```

Wayland 下記得設環境變數讓 GTK/Qt 都走 fcitx5：

```bash
# /etc/environment
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
```

dotfiles 放在 [github.com/G36maid/dotfiles](https://github.com/G36maid/dotfiles)，涵蓋 Hyprland、zsh + [zimfw](https://github.com/zimfw/zimfw)、tmux。誠實講：當時那份還沒更新到位，我後來才慢慢補齊，別照抄。

## Timeshift + grub-btrfs：搞砸了也能開機回去

這套是整台機器的保險絲，跟著 [Stephen 的 btrfs 影片](https://www.youtube.com/watch?v=V1wxgWU0j0E)配置的：

```bash
paru -S timeshift grub-btrfs inotify-tools timeshift-autosnap
```

- **timeshift**：GUI 設定快照類型選 *BTRFS*，快照位置指到系統碟。它用子卷快照建還原點，CoW 所以秒建、不佔雙倍空間。
- **grub-btrfs**：掃描系統上的 btrfs 快照，在 GRUB 選單生成「開機進入快照」的子選單。唯讀快照開機需要它的 initramfs hook 處理（[文件](https://github.com/Antynea/grub-btrfs)寫得很清楚）。
- **grub-btrfsd**：新版 grub-btrfs 用一個 daemon 監看快照目錄，快照增刪時自動重產 `grub-btrfs.cfg`。它底層靠 [inotify-tools](https://github.com/rfaquino/Inotify-tools) 的 `inotifywait` 監聽事件。用 Timeshift 的話不用手填路徑，開 `--timeshift-auto` 就會自動偵測（Timeshift ≥ 22.06）：

  ```bash
  sudo systemctl edit --full grub-btrfsd
  # /etc/conf.d/grub-btrfsd：
  #   optional_args="--syslog --timeshift-auto"
  sudo systemctl enable --now grub-btrfsd
  ```

- **timeshift-autosnap**：裝完自帶 pacman hook，**每次 pacman 交易前**先建一顆 Timeshift 快照（設定在 `/etc/timeshift-autosnap.conf`）。升級炸了就重開機選升級前的快照，回到可開機狀態再慢慢修。

這套組合唯一要注意：快照不是備份。系統碟死掉就什麼都沒了，重要資料還是要離機一份。

## Steam 與 NVIDIA：nvidia-open 已是預設

```bash
paru -S steam lib32-nvidia-utils
```

`nvidia-utils` 本體會被驅動包拉進來，Steam 的 32-bit 遊戲靠 `lib32-nvidia-utils`。驅動這裡記錄一下當時查證的結論：3070 Ti 是 **Ampere**（GA104），完全在 [open GPU kernel modules](https://github.com/NVIDIA/open-gpu-kernel-modules) 的支援範圍——open 模組依賴 Turing 才導入的 GSP 韌體，所以只支援 Turing 之後的卡，Maxwell/Pascal/Volta 必須留在閉源版。而從 [560 系列起，開源核心模組已是 NVIDIA 預設且建議的安裝選項](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/610/kernel-modules.html)。也就是說 archinstall 的 nvidia-open 選項對這張卡不只是可用，根本是正解；實際遊戲（Proton + DXVK/VKD3D）跑起來沒遇到需要閉源版的場景。

## Btrfs 儲存池：我在 raid5 上撞牆的經過

三顆 1TB 的定位是遊戲、媒體、VM 這種「可重取得」資料，系統碟不參與。原筆記裡我留了兩行 `mkfs.btrfs`，一行 `-d raid5 -m raid1`、一行只有 `-d raid5`——這就是踩雷現場：一開始手癢想上 raid5 拿空間效率，後來查證才認清現實。

[btrfs 官方 status 頁](https://btrfs.readthedocs.io/en/stable/Status.html)對 RAID56 的評級至今仍是 **unstable**（"do not use for other than testing purposes"），[mkfs.btrfs(8)](https://btrfs.readthedocs.io/en/latest/mkfs.btrfs.8.html) 直接寫 "RAID5/6 has known problems and should not be used in production"。write hole、partial stripe 寫入中斷電後 parity 失同步這類問題，社群吵了很多年；kernel 6.14 進了 write-intent bitmap，但「有修復機制」不等於「已 stabilized」。後來我甚至為此專門做了一輪斷電實測，完整記錄在[這篇 Btrfs RAID5 斷電實驗](/blog/btrfs-raid5-power-cut-lab/)——復原行為比想像中好，但 4K 隨機寫慢到不忍看。

所以儲存池最終定案是標題寫的那組：**RAID0 資料 + RAID1 metadata**——讀寫吞吐拿滿，中繼資料仍有兩份，資料層的風險用「這些資料本來就可在別處重取得 + 定期備份」消化。真要玩 raid56 的話，至少 metadata 要上 `raid1c3`（三份），這是 mkfs 文件裡給 RAID56 的建議組合。

```bash
# 最終採用：RAID0 資料，RAID1 metadata
sudo mkfs.btrfs -f -d raid0 -m raid1 \
  /dev/nvme1n1 /dev/nvme2n1 /dev/disk/by-id/ata-CT1000MX500SSD1_2302E69CE85A
```

注意 SATA 碟我用 `/dev/disk/by-id/` 路徑——裝置名順序開機後可能變，by-id 才是穩定識別。接著切子卷並掛載：

```bash
sudo mount /dev/nvme1n1 /mnt
sudo btrfs subvolume create /mnt/@games
sudo btrfs subvolume create /mnt/@media
sudo btrfs subvolume create /mnt/@vms
sudo umount /mnt
```

```bash
# /etc/fstab
UUID=<UUID>  /mnt/games  btrfs  noatime,compress=zstd,subvol=@games  0 0
UUID=<UUID>  /mnt/media  btrfs  noatime,compress=zstd,subvol=@media  0 0
UUID=<UUID>  /mnt/vms    btrfs  noatime,compress=zstd,subvol=@vms    0 0
```

```bash
sudo systemctl daemon-reload && sudo mount -a
sudo chown -R $USER:$USER /mnt/games /mnt/media /mnt/vms
```

`compress=zstd` 對遊戲資產和 VM 映像省空間有感；`noatime` 少一堆無意義寫入。

## 誠實評價

跑了幾個月的結論：

- **值得的**：7800X3D + 3070 Ti 在 Arch + Proton 下遊戲體驗非常順；快照救援鏈（Timeshift + autosnap + grub-btrfsd）真的救過我幾次 pacman 升級；RAID0 池當遊戲庫載入快。
- **缺點照列**：Hyprland 在 NVIDIA 上偶爾還是有些邊角案例要自己調；Btrfs raid5 想都別想，空間效率就認了 raid0 的風險；我的 dotfiles 整理進度永遠落後實際機器。
- 如果重來一次：架構不變，但我會第一天就把 `--config` JSON 留好——第二次重裝就能 30 分鐘全自動。

同一天整理的另一篇：[線上 Judge 部署記](/blog/cms-online-judge-deployment/)。家裡網路整體架構則寫在[ homelab 2025 ](/blog/homelab-2025-architecture/)；遊戲登入相關的坑另見[ FH6 + Arch + Proton 登入問題](/blog/fh6-arch-proton-login/)。
