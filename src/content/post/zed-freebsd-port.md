---
title: "用 AUR 思維替 Zed Editor 做非官方 FreeBSD port"
slug: zed-freebsd-port
description: "以 Arch Linux AUR 的打包思維,把 Zed Editor 帶到 FreeBSD 14.2:整理上游 patch、踩 webrtc-sys 預編譯 binary 404 的雷,並規劃成 editors/zed-editor port 的過程。"
pubDate: 2025-04-28 20:22:26 +0800
tags: ["freebsd", "zed", "rust", "ports", "aur", "gpui", "webrtc"]
category: linux
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1745842946
draft: false
media:
  - zed-freebsd-pr20480.png
---

> 擴寫自 2025-04-28 的隨筆

在 FreeBSD 上想用 Zed,`pkg search zed` 找不到任何東西。官方只發 macOS 和 Linux 版,而我日常主力是 FreeBSD 14.2-RELEASE (amd64),總不能為了一個編輯器天天切到 Linux 機器。乾脆自己動手做一個非官方 port。

![CalebHeydon 在 Zed PR #20480 附上的截圖：Zed 跑在 FreeBSD 14.1（KDE Plasma），右側 neofetch 顯示系統資訊](/media/2025-04/zed-freebsd-pr20480.png)

## 背景:Zed 離 FreeBSD 其實沒有很遠

Zed 是 Rust 寫的編輯器,UI 用自家的 GPUI 框架。它對非主流平台的障礙主要是兩層:GPUI 底層的平台綁定(windows、event loop、繪圖後端),以及一堆假設 `target_os = "linux"` 的依賴。

社群這半年動得很快:

- syobocat 在 [Discussion #10247](https://github.com/zed-industries/zed/discussions/10247) 貼出 Zed 跑在 FreeBSD 上的截圖
- [Issue #15309](https://github.com/zed-industries/zed/issues/15309) 是官方的 FreeBSD 支援追蹤 issue
- Caleb Heydon 的 [PR #20480](https://github.com/zed-industries/zed/pull/20480) 「Add initial FreeBSD support」:做法樸素到有趣——把所有檢查 `target_os = "linux"` 的地方加上 `target_os = "freebsd"`,就能建置、能跑
- [Discussion #25601](https://github.com/zed-industries/zed/discussions/25601) 則在處理 remote session(zed-remote-server)的 FreeBSD 支援

也就是說「能不能編譯」已經有人證明了,缺的是一個讓使用者一行指令裝起來的管道。我開了 [Discussion #29550](https://github.com/zed-industries/zed/discussions/29550) 說明要做非官方 port,也獲得 Zed 團隊的正面回應。目標:

1. 在 FreeBSD 14.2 (amd64) 原生編譯並執行 Zed
2. 打包成 FreeBSD port,路徑就叫 `editors/zed-editor`
3. 持續追蹤上游、維護 patch
4. 適時把修好東西回饋上游

## 1. 用 AUR 思維切入 ports

我是 Arch Linux 使用者,很習慣 AUR 的生態:社群成員自己維護 PKGBUILD、帶著自己的 patch,不等官方。FreeBSD ports 結構上其實很像——一個目錄、一個 Makefile、`distinfo` 和 `files/` 下的 patch,只是多了 maintainer 責任和 QA 流程。所以我的做法是:先在自己的 fork 快速迭代,patch 齊了再整理成 port 提交。

```sh
git clone --depth=1 https://github.com/zed-industries/zed.git
cd zed
```

## 2. 環境與第一次編譯

照 Zed 的開發文件,Linux 上是跑 `script/linux` 裝依賴——踩雷一號:那支 script 是給 Debian/Ubuntu 用的(`apt-get install ...`),在 FreeBSD 上直接爛掉。正確姿勢是手動把依賴裝齊:

```sh
pkg install rust cmake clang pkgconf expat fontconfig \
    alsa-lib vulkan-headers wayland git
rustup target add x86_64-unknown-freebsd
```

Rust 對 `x86_64-unknown-freebsd` 是 Tier 2 with host tools 等級的目標,rustup 直接給,這點比想像中省事。接著 `cargo build` 起跑。

## 3. webrtc-sys 的 404:第一道真正的牆

編譯很快死在 `webrtc-sys` 的 build script:

```text
thread 'main' panicked at webrtc-sys/build.rs:89:45:
called `Result::unwrap()` on an `Err` value:
  "failed to download webrtc: 404 Not Found"
```

去看 Zed 用的 [livekit-rust-sdks fork](https://github.com/zed-industries/livekit-rust-sdks) 的 `build.rs` 才明白:它會依 target 去 GitHub Releases 下載**預先編譯好的 WebRTC C++ 靜態庫**,但清單裡只有 linux x64 和 macOS 這幾個 triple,freebsd 的 URL 根本不存在,於是 404。我把完整錯誤[貼了 gist](https://gist.github.com/G36maid/5b64c7bd801368a208e6750432e24cbc)。

要根治得自己用 GN 把 WebRTC 編出 FreeBSD 版,工程不小;務實的做法是先用 `LK_CUSTOM_WEBRTC` 繞過、或 patch 掉 livekit 相關 crate,犧牲 audio call 和螢幕分享——反正 Zed 的核心編輯體驗不依賴它。

## 4. 包成 port

編過之後就是把成果翻譯成 ports 格式:`USES=cargo` 會處理 crates 的抓取與 checksum,`CARGO_CRATES` 列出依賴,自家的修正放 `files/patch-*`。骨架大致:

```makefile
PORTNAME=	zed
DISTVERSION=	0.x.x
CATEGORIES=	editors
MASTER_SITES=	https://github.com/zed-industries/zed/archive/
DISTNAME=	v${DISTVERSION}

MAINTAINER=	me@example.com
COMMENT=	High-performance collaborative code editor
USES=		cargo
USE_GITHUB=	yes
GH_ACCOUNT=	zed-industries
GH_PROJECT=	zed
```

## Why / Trade-off

AUR 式的「先 fork 快跑、再整理成 port」最大的好處是迭代快:上游 commit 天天變,patch 用 `git am` 串著走就好;等到穩定才固化成 `files/` 裡的 patch。代價是維護者要獨自扛著 upstream 隨時改格式的風險,而且 `webrtc-sys` 這種「build 時從 release 抓 binary」的依賴,天生跟 ports 的「所有東西都要可重現、可驗 hash」哲學打架,得額外把 WebRTC 源碼納進 `distinfo`,這是後續最硬的工程。

## 踩雷清單

- `script/linux` 是 Ubuntu 專用,FreeBSD 要手動裝依賴
- webrtc-sys 下載預編譯 binary,freebsd triple 直接 404
- PR #20480 提醒:要加 `RUSTFLAGS="-C link-dead-code"` 才能正常連結
- `project/environment.rs` 裡呼叫 `/bin/sh -l`,FreeBSD 的 sh 不吃 `-l`
- `ashpd` 的檔案選擇器在 FreeBSD(至少 KDE)不動,好在有 fallback dialog;丟回收桶功能也是壞的

## 個人評價

走完一輪,最大感想是 Rust 生態對 FreeBSD 比預期友善——rustup 直接可用、大部分 crate 都能過,真正的牆集中在「需要預編譯 C++ binary」的少數依賴。GPUI 的平台抽象也做得不錯,一個「到處加 `target_os = "freebsd"`」的 PR 就能動,代表它沒有把 Linux 假設焊死在架構裡。這次當然還沒全打通,但方向明確:webrtc 一關打通、port 就能成形。等手上的 patch 洗乾淨,會把 `editors/zed-editor` 提上 FreeBSD Bug Tracker,也希望 upstream 哪天直接把 FreeBSD 收編成支援平台。
