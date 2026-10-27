---
title: "Headscale v0.23 升級與重構筆記：把自架 fork 的客製功能搬回上游新架構"
description: "Headscale v0.23 名義上是重構釋出版，實際上動了 policy、notifier、資料庫結構與整套 API 命名。這篇整理我從 0.22.3-dev fork 升級到 0.23 的檔案搬遷地圖、policy 系統改動，以及自製 ACL API、tag-lock、透明子網路等客製功能在上游的對照與移植取捨。"
pubDate: 2025-07-10T07:20:22+08:00
category: self-hosting
tags: ["headscale", "tailscale", "vpn", "self-hosting", "go", "refactor"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1752103222
---

> 本文由 2025-07-10 的 HackMD 遷移評估筆記於 2026-10-27 擴寫而成。當時我維護著一份多實例部署用的 Headscale fork，正在盤點怎麼把客製功能搬上剛定型的 v0.23；文中的「0.23-dev」指的是評估當下的開發版代碼，觀察皆來自讀碼與 changelog，不是 benchmark 數據。版本事實以 [v0.23.0 release notes](https://github.com/juanfont/headscale/releases/tag/v0.23.0) 為準。

自己 fork 一套 Headscale 最大的成本不是寫功能，是**跟上游**。我的 fork 加了多實例部署需要的東西：控制面板用的 ACL 操作 API、活躍 ACL reload、tag-lock 訂閱分級、透明子網路廣播。這些都散在 `hscontrol/` 的單檔結構裡，改起來很方便——直到上游宣布 v0.23。

## 「本來只是要整理代碼」的一版

[v0.23.0 的 release notes](https://github.com/juanfont/headscale/blob/v0.23.0/CHANGELOG.md) 開頭寫得很老實：這一版原本「主要是代碼重組與重構，大幅提升可維護性」，結果變成一個「比預期大得多、長得多」的 release 週期。從 2023 年 9 月 [Machine→Node 改名 PR #1553](https://github.com/juanfont/headscale/pull/1553) 合併起算，到 2024 年 9 月才出正式版，整整一年。

對 fork 維護者來說，這代表兩件事：一是幾乎每個客製補丁的落點檔案都搬家了，二是幾個 breaking change 會直接打在自製功能上。

## 檔案搬家地圖

v0.23 把 `hscontrol/` 的大雜燴拆成職責分明的模組，我自己的搬遷對照表（上游也建議逐 PR 檢視）：

| 舊位置（0.22.3） | 新位置（0.23） |
| --- | --- |
| `hscontrol/acls.go` | `hscontrol/policy/acls.go` |
| `hscontrol/config.go` | `hscontrol/types/config.go` |
| `hscontrol/db.go` | `hscontrol/db/db.go` |
| `hscontrol/machine.go` | `hscontrol/db/node.go`（改名） |
| `hscontrol/routes.go` | `db/routes.go` + `types/routes.go` |
| `hscontrol/preauth_keys.go` | `db/preauth_keys.go` + `types/preauth_key.go` |
| `hscontrol/protocol_common.go` | `hscontrol/auth.go` |
| `hscontrol/protocol_common_poll.go` | `hscontrol/poll.go` |
| `proto/.../machine.proto` | `proto/.../node.proto`（改名） |

新結構的好處很實際：`db/`、`types/`、`policy/`、`mapper/`、`notifier/` 各自帶測試（`*_test.go` 密度明顯提高），`types/testdata/` 甚至放著 0.22.3→0.23 遷移用的 sqlite 樣本。壞處是：**我每一個補丁的 merge conflict 都要重解一次**。

## Policy 系統：YAML 沒了，換 HuJSON

對我衝擊最大的是 [policy 格式](https://github.com/juanfont/headscale/pull/1792)：YAML 政策檔不再支援，唯一格式是 [HuJSON](https://github.com/tailscale/hujson)——就是 Tailscale 官方那套「帶註解、允許行尾逗號的 JSON」（JWCC）。跟進 Tailscale 的政策語法是正確方向，但代表我的控制面板產生政策的方式要重寫產生器。

API 層面則是從我的 `ACLCreateGroup`、`ACLGroupAddUser` 這種細粒度端點，收斂成通用的 `GetPolicy`/`SetPolicy`：政策以字串進出，DB 模式下存進 `policies` 表、**每次 SetPolicy 建一筆新紀錄（版本化）**，並在 SetPolicy 內接上 notifier 主動推送。麻煩在於上游的 `SetPolicy` 當時只支援 DB 模式，而我的叢集環境需要 File 與 DB 兩種模式並存——File 模式得自己做原子寫入避免多實例寫壞檔案。

## Notifier 與 Delta Updates：輪詢時代的結束

0.22 之前，節點更新靠固定間隔輪詢；v0.23 把串流邏輯整個重寫（[`poll.go`](https://github.com/juanfont/headscale/blob/main/hscontrol/poll.go)），改成 [notifier](https://github.com/juanfont/headscale/tree/main/hscontrol/notifier) + Go channels：每個連線節點一條專屬 channel 收 `StateUpdate`，變更即時推播。搭配 mapper 模組還能送 **delta updates**——只通知新增/異動/移除的節點，不再每次吐整份網路狀態。

更新流程變成一條直線：

```
SetPolicy() → Validate → Store → Update In-Memory → NotifyAll() → Node Channels
```

對我的用途這是紅利：過去為了「控制面板改 ACL 後立刻生效」自己寫的 active reload，上游的 `notifier.NotifyAll` 直接涵蓋了。批量策略也合理——patch 合併送、完整更新立即送，channel 重連時替換、timeout 防死鎖。

## 其他 breaking changes 速記

- **DERP**：啟動時至少要有一個 DERP、內嵌 DERP 需要 `derp.server.private_key_path`，設定錯了直接拒絕啟動（#1611）。
- **Docker**：映像改用 [ko](https://github.com/ko-build/ko) 建，entrypoint 從 shell 換成 `headscale` 本身——`headscale serve` 要寫成 `serve`，`/var/lib/headscale`、`/var/run/headscale` 也不再自動建立。
- **CLI**：除 `serve` 外所有指令只要最小設定檔就能跑；stdout/stderr 收斂成結果走 stdout、錯誤走 stderr（#2109）。
- **資料庫**：設定結構重整（#1700）、改用版本化 migration、SQLite 預設開 WAL。release note 特別提醒**跨版本務必備份資料庫**。
- **DNS**：MagicDNS 網域不再含使用者名稱（可用 `use_username_in_magic_dns` 暫時還原）；`dns.base_domain` 不得與 `server_url` 重疊。

## 自製功能移植盤點

fork 的存亡就在這張表：

| 客製功能 | 上游狀態 | 移植落點 | 備註 |
| --- | --- | --- | --- |
| ACL API + 活躍 reload | ✅ 概念已收斂 | `policy/`、`db/policy.go`、`notifier/` | 換成 `GetPolicy`/`SetPolicy` + `NotifyAll` 重接 |
| Tag-lock 訂閱分級 | ❌ 未上游 | `db/preauth_keys.go`、`types/preauth_key.go` | 補丁需重寫，DB 端 tag 過濾要重接 |
| 透明子網路模式 | ❌ 未上游 | `poll.go`、`db/routes.go` | 要深度整合新 notifier + mapper |
| 主動斷線 API | ❌ 未上游 | `db/node.go`、`grpcv1.go` | 得跟著新 notifier 生命週期重做 |
| 依 tag 指定自訂 IP | ❌ 未上游 | `db/node.go`、`types/preauth_key.go` | IP 配置邏輯需重移植 |
| 群組路由隔離 | ⚠️ 部分涵蓋 | `db/routes.go`、`notifier/` | delta updates 或可部分取代 |
| 資料庫連線優化 | ✅ 上游改善 | `db/`、`notifier/` | 模組化後順手解決 |
| YAML 環境變數支援 | ✅ 已支援 | `types/config.go` | `${ENV}` 開箱即用 |

結論是：**八個客製功能裡三個可以刪掉、兩個要重寫、三個要認真重移植**。這就是跟上游一年的代價。

## 移植方案的取捨

以 ACL API 為例，我評估過三條路：

1. **泛化 `SetPolicy`、舊 API 當 wrapper**——先讓 `SetPolicy` 支援 File 模式（含原子寫入），再把所有 `ACL*` API 重寫成「GetPolicy → 記憶體內改 → SetPolicy」的薄殼。驗證、儲存、推播全部集中在一處，DRY、好 debug。缺點是要先投資把 File 模式做對。
2. **每個舊 API 各自寫雙模式 if/else**——最直覺的直譯，但檔案 I/O、DB I/O、驗證、推播的邏輯會在幾十個函式裡重複，一個 bug 要修幾十次，行為還容易不一致。
3. **把舊 helper（`copyACLConfig` 等）改成 mode-aware**——等於把舊的「pending changes」工作流硬塞進新架構，重造一層 `SetPolicy` 已經解決的抽象，最尷尬的中間態。

我選了 1。理由很樸素：v0.23 的整個設計意圖就是「所有寫入走單一入口」，逆著架構做事，下一次升級會再痛一次。

## 給同樣在 fork 的你

三點收穫。第一，**fork 前先看上游 roadmap**——我加的 ACL API、env var 支援、連線優化，上游後來都用更乾淨的方式做了，等於白繳學費；真正無可取代的只有 tag-lock 和透明模式這種商業邏輯。第二，**v0.23 這種「重構版」對 fork 是最痛也最值得的**：痛在全部補丁要重解，值得在新架構下重接之後，notify/mapper 的邊界反而讓客製程式碼更薄。第三，誠實聲明證據等級：本文的移植判斷基於 0.23-dev 當下的代碼與 changelog，正式版與後續 0.23.x 的細節可能再有變動，動手前請以你手上那個 tag 的源碼為準。
