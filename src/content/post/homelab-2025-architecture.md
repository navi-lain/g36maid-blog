---
title: "🏠 Homelab 2025 規劃：把一頁工具清單變成一個能跑的架構"
description: "從 2025 年 10 月的一頁 Homelab 規劃清單出發，整理當時的選型脈絡：虛擬化與儲存的取捨、Traefik 與 Vaultwarden 的自動化思維、Cloudflare Tunnel 加 Headscale 的內外分流，以及媒體、檔案與本地 LLM 的工具選擇，還有清單沒寫到的盲點。"
pubDate: 2025-10-07T21:24:38+08:00
category: self-hosting
tags: ["homelab", "proxmox", "docker", "self-hosting", "tailscale", "jellyfin", "immich", "llm"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1759843478
---

> 本文由 2025-10-07 的 HackMD 規劃清單於 2026-10-29 擴寫而成。原稿是一頁以 checkbox 條列的工具評選，打勾的項目代表當時已定案或實際跑起來的選擇；本文補上每個取捨背後的理由，「沒勾」的項目也照實保留。當時的版本脈絡以 2025 年 10 月為準。

「要不要自己蓋一個 homelab」這個問題，我想了兩年，真正動手是 2025 年。當時在 HackMD 開了一頁規劃清單，把想得到的層面——虛擬化、儲存、容器管理、監控、網路入口、媒體、檔案、開發、AI——全部列成 checkbox，一項一項比對候選工具。現在回頭看，這頁清單最值錢的不是結論，而是每個項目旁邊「為什麼不選另一個」的理由。

## 虛擬化與儲存：先定地基

清單第一節是 Hypervisor 與 NAS OS 的取捨，唯一打勾的是 [Proxmox VE](https://www.proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-0)。寫清單時 PVE 剛出 9.0（2025 年 8 月，底層換上 Debian 13 "Trixie"），KVM 加 LXC 的組合對我剛好：該隔離的服務開 VM，輕量的開 LXC 容器，同一個 web 介面全管。

儲存那格我遲遲沒勾。[TrueNAS SCALE](https://www.truenas.com/blog/truenas-goldeye-25-10/) 的賣點是原生 ZFS；2025 年 TrueNAS 把 CORE 和 SCALE 整併成 Community Edition，一年一大版的節奏也定了下來。但它本質上是整套作業系統——要嘛吃掉整台機器，要嘛變成 PVE 裡的一個 VM，兩種都有點彆扭。[Unraid](https://unraid.net/) 的彈性硬碟陣列很誘人，但授權是付費的，parity 保護下的寫入效能也一直是老話題。最後我的折衷是：PVE 直接上 ZFS，當時的資料量還不需要一套專門的 NAS OS。也是這個決定，讓我後來認真去研究 CoW 檔案系統遇到斷電時的行為，做了[一次模擬 RAID5 斷電的實驗](/blog/btrfs-raid5-power-cut-lab/)。

## Admin 層：能自動化的就不要手動

容器管理勾了 Portainer——不是它最強，而是圖形化到可以被家人託管。反向代理勾了 [Traefik](https://doc.traefik.io/traefik/) 而不是更熱門的 Nginx Proxy Manager，關鍵差異在服務發現方式：Traefik 直接讀 Docker labels，新容器加上兩行 label 就自動掛上路由與 Let's Encrypt 憑證，不必進 UI 點滑鼠。對會頻繁開關容器的 homelab，這個自動化值得初期設定成本。密碼管理勾了 [Vaultwarden](https://github.com/dani-garcia/vaultwarden)——Bitwarden API 的 Rust 實作，一個容器、幾十 MB 記憶體就搞定全家的密碼同步。

SSO 那格（Authelia 對 Authentik）最後是空的：兩者都要替每個應用綁 forward auth 設定，工程量不小，我決定先靠 Cloudflare Tunnel 加上服務自己的密碼頂著，之後再說。

## 網路入口：內外分流

這節是整份清單裡想最清楚的。對外服務勾了 [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)：家裡沒有固定公網 IP，也不想開 port，Tunnel 用 outbound 連線把內部服務掛到自己的 domain 上，路由器上不需要任何 inbound 規則。遠端管理則走 Tailscale 生態：[Tailscale](https://tailscale.com/) 與開源控制伺服器 [Headscale](https://github.com/juanfont/headscale) 都打了勾，讓整個 mesh 網路不依賴 Tailscale 的 SaaS。這套組合後來維護出不少心得，整理在[另一篇 Headscale v0.23 升級筆記](/blog/headscale-v023-upgrade/)；實際的網路拓樸長相則記錄在[網路架構盤點](/blog/home-network-topology-audit/)。

防火牆勾了 OPNsense——介面現代、更新週期快，勝過以穩定著稱但步調慢的 pfSense。DNS 廣告過濾在 AdGuard Home 和 Pi-hole 之間徘徊，這格清單上到今天還是空的。

個人儀表板勾了 [Homepage](https://gethomepage.dev/)，理由和 Traefik 一脈相承：它可以透過各服務的 API 整合上百種應用，狀態直接長在首頁上，而不是一堆靜態連結。清單裡也列了 Homarr、Dashy、Heimdall、Homer 一整排替代品，但結論很早就收斂了——與其比介面美醜，不如比誰能真的接上自己的服務。

## 監控、媒體與日常

監控勾了兩個互補的：[Netdata](https://github.com/netdata/netdata) 管每秒級的即時排查，圖表華麗；[Beszel](https://beszel.dev/) 管輕量的容器與主機儀表板。Prometheus 加 Grafana 那格留白——長期趨勢的需求當時還不存在，不想為了架構圖先架一套。

媒體這節幾乎全勾：[Jellyfin](https://jellyfin.org/)（完全開源，這點對我沒得妥協）、[Immich](https://github.com/immich-app/immich) 當 Google Photos 的替代品、輕量的 Filebrowser、功能全面的 Nextcloud、去中心化的 Syncthing，還有 Stirling-PDF 這種小而美的網頁工具。*arr 自動下載套件（Sonarr/Radarr 那一族）沒勾，那是我不想碰的灰色地帶。

AI 那節的勾選狀態很誠實：整節沒有任何勾。構想是 Ollama 跑本地 LLM、Open WebUI 當前端、ComfyUI 畫圖、vLLM 是「等有張像樣的 GPU 再說」的進階項。以當時的 GPU 行情，這節整節延後，先讓清單躺著。

清單最後一節是生活工具，全部沒勾但都值得記一筆：Mealie 管食譜、Actual Budget 管記帳（本地優先、重視隱私）、Home Assistant 當智慧家庭中樞、NocoDB 把資料庫變成試算表。這節的存在本身就說明了一件事——homelab 的終點不是技術，是「我想要什麼樣的生活」。工具清單可以列一百個，但每一格背後其實是同一個問題：這件事，我想不想把它放在自己的機器上？

## 清單沒寫的事

回頭看，這份規劃最大的盲點是它太像軟體採購清單：備份策略（3-2-1）、UPS、硬體預算、以及「服務掛了誰會發現」，都沒有一格。後來實際把監控和備份補齊的過程，比選哪個 dashboard 難多了。如果你也在寫自己的 homelab 規劃，建議把清單從「工具名稱」升級成「這個工具掛掉時我怎麼知道、怎麼救」——checkbox 勾完只是開始。
