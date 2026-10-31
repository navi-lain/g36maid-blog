---
title: "替北區高中資能競架 Online Judge：CMS v1.5 在 ESXi VM 上的建置實錄"
description: "由 2025-10-14 的 HackMD 建置共編筆記擴寫。接到月薪五千替北區高中資訊能力競賽架 OJ 的委託：盤點 Dell PowerEdge R750/R650 與 ESXi 架構、更正原稿版本記錄（實際為 CMS v1.5.x，2025-10 當時最新穩定版 v1.5.1）、完整保留親手驗證過的 Arch 與 Ubuntu 兩套安裝指南（isolate 沙箱、PostgreSQL 權限三坑：schema owner、pg_largeobject、pg_hba.conf），並誠實列出 TODO 清單未完項（壓測、防作弊、自動化）。人物已去識別化。"
pubDate: 2025-10-14 05:56:08 +0800
category: linux
tags: ["cms", "online-judge", "arch-linux", "ubuntu", "postgresql", "isolate", "部署", "競賽系統"]
draft: false
source: "hackmd"
sourceType: "note"
sourceTimestamp: 1760392568
---

> 本文擴寫自 2025-10-14 的內部共筆（HackMD 專案筆記），於 2026-10-31 整理成文；版本與連結以 2025 年 10 月查證當下為準。

## 五千塊一個月，來幫高中生架個 Online Judge？

2025 年 10 月，我收到系上的委託信，內容很短：「系上需要一位同學協助建置系統（類似 Online Judge），薪資 5000/月，三個月，工作內容是架設北區高中生資訊能力競賽的伺服器。」我和另一位同學（ryan）接下這個案子，對接窗口是系上老師。

打開現況才發現是從半成品開始：機器是現成的，系統「已經有人裝了一版」，但版本不明、文件不存在、能不能撐住正式賽沒人知道。以下記錄盤點到部署的過程——重心放在兩套親手跑過的安裝指南（Arch 與 Ubuntu），沒做完的部分也會誠實列出。

## 專案現況與架構總覽

硬體是系上既有的兩台 Dell PowerEdge，跑在 ESXi 上的 Linux VM，儲存走 NFS：

- **R750**：32 核心 Xeon Gold 6326、128 GB RAM——主力評測機
- **R650**：24 核心 Xeon Silver 4310、64 GB RAM——備援／測試機

軟體選了 [CMS（Contest Management System）](https://cms-dev.github.io/)：2012 年 IOI 官方使用的評測系統，[AGPL-3.0 授權](https://github.com/cms-dev/cms)，分散式架構——ContestWebServer、AdminWebServer、Worker、ResourceService 等服務可拆到不同機器。它只支援 PostgreSQL——[內部依賴 Large Object 機制](https://cms.readthedocs.io/en/v1.5/Introduction.html)存題目與提交檔案。

先修正原稿的兩個錯誤：原稿寫「CMS 系統（版本 cms 12）」——CMS 根本沒有這種版本號，實際部署的是 **v1.5 branch**；2025 年 10 月當時最新穩定版是 [v1.5.1](https://github.com/cms-dev/cms/releases/tag/v1.5.1)（2025-06-07 發布，修復 archive.py 的[安全漏洞](https://github.com/cms-dev/cms/pull/1344)），v1.5.0 則是 2024 年 12 月底發布。原稿還寫「似乎是透過 Docker 安裝」——repo 有 `docker/` 目錄與社群映像，但我們最後直接在 VM 上裸裝，下面兩份指南就是這麼來的。

沙箱方面，CMS 用 [isolate](https://github.com/ioi/isolate)（IOI 官方沙箱，靠 cgroups + namespace 隔離不可信程式），官方的 `prerequisites.py` 會幫你裝好，截至 2025 年仍在活躍維護。

## 在 Arch Linux 上安裝 CMS

這套流程我在 Arch 機器上從零跑到能匯入比賽，每一步都驗證過；我的主力機本來就是 Arch（[安裝筆記另文](/blog/arch-install-notes-2025/)）。

### 1. 安裝系統依賴

推薦用 `paru` 之類的 AUR helper，一次裝完官方庫和 AUR 的套件：

```bash
# 使用 paru 一次安裝所有官方與 AUR 的套件
paru -S --needed base-devel jdk8-openjdk fpc \
  postgresql postgresql-client python libcap \
  postgresql-libs libcups libyaml python-pip \
  libcgroup cppreference-doc-en-html
```

> **Note:**
> 如果不想一次裝完，可以分開處理：
>
> 1. 先用 `pacman` 安裝官方套件。
> 2. 再手動從 AUR 安裝 `libcgroup` 和 `cppreference`。

#### 可選套件

若需要 Web Server、PHP 管理後台或更多語言支援：

```bash
paru -S --needed nginx php php-fpm phppgadmin texlive-core \
  a2ps ghc rust mono pypy3
```

### 2. 下載 CMS 原始碼

```bash
git clone --recursive --branch v1.5 --single-branch https://github.com/cms-dev/cms.git
cd cms
```

### 3. 設定 Python 虛擬環境並安裝 CMS

這裡用 `uv` 代替 pip——依賴解析快到一個誇張，體感就是爽：

```bash
# 建立 venv
uv venv ~/cms_venv

# 啟用 venv
source ~/cms_venv/bin/activate

# 安裝依賴，然後安裝 cms 本體
uv pip install -r requirements.txt
uv pip install .
```

### 4. 設定系統環境（isolate 沙箱）

CMS 靠 isolate 沙箱執行選手提交的程式碼，跑官方 script 完成設定：

```bash
sudo python3 prerequisites.py
```

過程中會問是否把你加入 `cmsuser` group：

- **回答 `Y`**：script 自動處理。
- **回答 `N`**：手動加入：

  ```bash
  sudo usermod -a -G cmsuser $USER
  ```

無論哪種方式，都要重新載入群組設定：

```bash
newgrp cmsuser
```

用 `groups` 確認自己已在 `cmsuser` 群組中。

### 5. 建立 PostgreSQL 資料庫與帳號

```bash
# 切換到 postgres 系統帳號
sudo -u postgres -i

# 建立 cmsuser 並設定密碼
createuser --pwprompt cmsuser

# 建立 cmsdb，擁有者設為 cmsuser
createdb --owner=cmsuser cmsdb

# 調整 schema 權限
psql -d cmsdb -c "ALTER SCHEMA public OWNER TO cmsuser"
psql -d cmsdb -c "GRANT SELECT ON pg_largeobject TO cmsuser"

# 退回原本的 shell
exit
```

`GRANT SELECT ON pg_largeobject` 不是裝飾：CMS 把題目檔案、提交原始碼都塞在 Large Object 裡，新版 PostgreSQL 預設收緊了對它的存取，漏掉這行 `cmsInitDB` 會直接炸給你看。

### 6. 設定 PostgreSQL 連線權限（pg_hba.conf）

```bash
# 找到 pg_hba.conf 的路徑
HBA_CONF=$(sudo -u postgres psql -Atc "SHOW hba_file")
echo "PostgreSQL HBA config file is at: $HBA_CONF"

sudo nano "$HBA_CONF"
```

在檔案**最上方**加入（pg_hba.conf 是第一條符合就生效，放下面會被預設規則先擋掉）：

```ini
# TYPE  DATABASE        USER            ADDRESS                 METHOD
host    cmsdb           cmsuser         127.0.0.1/32            md5
host    cmsdb           cmsuser         ::1/128                 md5
```

```bash
sudo systemctl restart postgresql
```

### 7. 複製並設定 CMS config

```bash
sudo cp config/cms.conf.sample /usr/local/etc/cms.conf
sudo cp config/cms.ranking.conf.sample /usr/local/etc/cms.ranking.conf
```

編輯 `/usr/local/etc/cms.conf`，把 `database` 欄位填入步驟 5 設定的密碼：

```json
// ...
"database": "postgresql+psycopg2://cmsuser:YOUR_PASSWORD_HERE@127.0.0.1:5432/cmsdb",
// ...
```

### 8. 初始化資料庫與建立管理員

```bash
# 記得啟用 venv
source ~/cms_venv/bin/activate

# 初始化 DB
cmsInitDB

# 新增管理員
cmsAddAdmin admin
```

### 9. 啟動 CMS 服務

```bash
# 啟動所有在 cms.conf 中定義的 resource services（例如評測機）
cmsLogService
cmsResourceService -a
```
```bash
# 啟動管理員用的 Web Server
cmsAdminWebServer
```
```bash
# 啟動公開用的 Ranking Server
cmsRankingWebServer
```

> **Note:**
> 為了方便管理，可以把這些服務註冊成 systemd units；但臨時架設或測試，手動啟動就夠了。

### 10.（可選）匯入比賽範例

驗收安裝最快的方式是灌官方範例：

```bash
# 下載範例
git clone https://github.com/cms-dev/con_test.git
cd con_test

# 記得啟用 venv
source ~/cms_venv/bin/activate

# 匯入選手和比賽
cmsImportUser --all -L italy_yaml .
cmsImportContest -i -L italy_yaml .
```

匯入成功後，用 `admin` 帳號登入後台，比賽應該已經出現了。

## Ubuntu 版安裝流程

正式環境的 VM 是 Ubuntu，所以同一套流程再走一遍。差異主要是套件名稱和 `sudo -u postgres` 的寫法：

### 1. 安裝系統套件

```bash
sudo apt update
sudo apt install -y build-essential openjdk-8-jdk fpc postgresql postgresql-client \
    python3 python3-venv python3-pip libcap-dev libcups2-dev libyaml-dev \
    nginx php php-fpm texlive-latex-base a2ps ghc rustc mono-runtime pypy3
```

> 💡 若 `openjdk-8-jdk` 無法安裝（新版 Ubuntu 已移除），改用：
>
> ```bash
> sudo apt install -y openjdk-11-jdk
> ```

### 2. 下載 CMS 專案

```bash
git clone --recursive --branch v1.5 --single-branch https://github.com/cms-dev/cms.git
cd cms
```

### 3. 建立 Python 虛擬環境並安裝 CMS

```bash
python3 -m venv ~/cms_venv
source ~/cms_venv/bin/activate
pip install -r requirements.txt
pip install .
```

### 4. 安裝系統環境（isolate）

```bash
sudo python3 prerequisites.py
```

若要自己管理群組，選 **N** 然後：

```bash
sudo usermod -a -G cmsuser $USER
newgrp cmsuser
```

### 5. 建立資料庫

```bash
sudo -u postgres createuser --pwprompt cmsuser
sudo -u postgres createdb --owner=cmsuser cmsdb
sudo -u postgres psql -d cmsdb -c "ALTER SCHEMA public OWNER TO cmsuser"
sudo -u postgres psql -d cmsdb -c "GRANT SELECT ON pg_largeobject TO cmsuser"
```

### 6. 設定資料庫權限（pg_hba.conf）

```bash
sudo -u postgres psql -Atc "SHOW hba_file"
# 假設輸出 /etc/postgresql/14/main/pg_hba.conf
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

在檔案開頭加上：

```
host    cmsdb   cmsuser   127.0.0.1/32   md5
host    cmsdb   cmsuser   ::1/128        md5
```

```bash
sudo systemctl reload postgresql
```

### 7. 複製設定檔

```bash
sudo cp ~/cms/config/cms.conf.sample /usr/local/etc/cms.conf
sudo cp ~/cms/config/cms.ranking.conf.sample /usr/local/etc/cms.ranking.conf
```

編輯 `/usr/local/etc/cms.conf`：

```bash
sudo nano /usr/local/etc/cms.conf
```

修改資料庫連線資訊：

```
"database": "postgresql+psycopg2://cmsuser:<你的密碼>@127.0.0.1:5432/cmsdb"
```

### 8. 初始化資料庫與建立管理員

```bash
source ~/cms_venv/bin/activate
cmsInitDB
cmsAddAdmin admin
```

### 9. 啟動 CMS 服務

```bash
cmsLogService
cmsResourceService -a
```
```bash
cmsAdminWebServer
```
```bash
cmsRankingWebServer
```

### 10. 匯入比賽範例（可選）

```bash
git clone https://github.com/cms-dev/con_test.git
cd con_test
cmsImportUser --all -L italy_yaml .
cmsImportContest -i -L italy_yaml .
```

## 踩雷筆記

- **`pg_hba.conf` 的順序就是一切**：第一條符合就生效——把 `cmsdb` 條目放尾端會被先命中的預設規則擋掉，症狀是密碼正確卻連不上。
- **群組變更不會熱套用**：`usermod` 加完 `cmsuser` 後，當前 shell 不會自動生效，要 `newgrp` 或重新登入。我卡在這裡浪費了半小時。
- **`pg_largeobject` 權限**：漏了這行 `cmsInitDB` 會失敗，錯誤訊息卻長得很像密碼錯誤，容易查錯方向。
- **Java 版本**：CMS 文件以 JDK 8 為基準，Ubuntu 新版裝不到就換 11，實測評測正常。
- **服務間通訊沒有加密認證**：[官方文件明講](https://cms.readthedocs.io/en/v1.5/Introduction.html)，除了 Ranking Web Server 外，各服務之間、服務與資料庫之間都沒有認證機制，必須用防火牆隔出獨立網路。對正式賽來說這是硬需求，不是選配。

## 誠實說：還有很多沒做完

照原稿 TODO List 誠實說：**系統優化整個區塊是空的**——效能分析、資安檢查（隔離、權限、防作弊）、架構調整全都沒做。**壓力測試也還沒跑**，「大量選手同時提交會發生什麼事」——當時我們還沒有答案。自動化部署腳本、帳號／題目管理流程、排錯工具同樣未完成；文件倒是先寫了（配置規格書＋維運 Wiki）。正式賽支援和賽後備份更是連碰都沒碰。

換句話說，這篇是「部署到能動」的記錄，不是「能上正式賽」的保證。

個人評價：CMS 的優點是 IOI 級別的實戰履歷與分散式架構；缺點是安裝步驟多且文件分散、只支援 PostgreSQL、設定不利容器化、以及上面那些「自己負責」的安全面向。對月薪五千的案子來說它仍是最合理的選擇——只是離「放心開賽」還有一段路要走。

## 延伸閱讀

- [CMS 官網](https://cms-dev.github.io/)／[原始碼](https://github.com/cms-dev/cms)（v1.5 branch，AGPL-3.0）／[v1.5 文件](https://cms.readthedocs.io/en/v1.5/)
- [isolate 沙箱](https://github.com/ioi/isolate)（IOI 官方）
- [cms-dev/con_test](https://github.com/cms-dev/con_test)：驗收安裝用的官方範例比賽
- 站內：[我的 Arch Linux 安裝筆記](/blog/arch-install-notes-2025/)、[Arch 的 ricing 心得](/blog/linux-ricing-productivity/)
