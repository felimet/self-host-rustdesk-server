# Server 架設指南 (Docker)

本文件說明如何使用 Docker Compose 部署 RustDesk Server（社群版），搭配 Cloudflare Tunnel 保護 Web Console。

> 使用映像檔：[lejianwen/rustdesk-server-s6](https://hub.docker.com/r/lejianwen/rustdesk-server-s6)（hbbs + hbbr + API 整合）

> ⚠️ **核心配置警告**：
> 本文件中提及的 `docker-compose.yml` 與 `.env` 參數設定（包含網路結構、通訊埠映射、信任代理清單等）皆**直接攸關系統的網路安全與防爆破防禦機制**。在不充分了解其底層運作原理的情況下，**請務必保持目前的預設值**。任何未經深入了解的輕率更改（例如隨意更改代理透傳設定），極有可能導致防護失效或大範圍的阻斷服務（DoS）。

詳細後台設定請參考 [RustDesk 官方文件](https://rustdesk.com/docs/zh-tw/self-host/rustdesk-server-pro/console/)。社群版與官方有些許差異。

## 目錄

- [系統需求](#系統需求)
- [部署步驟](#部署步驟)
- [Port 說明與防火牆](#port-說明與防火牆)
- [路由器 Port Forwarding](#路由器-port-forwarding)
- [Cloudflare Tunnel 設定](#cloudflare-tunnel-設定)
- [安全配置說明](#安全配置說明)
- [帳號安全](#帳號安全)
- [NAS 部署注意事項](#nas-部署注意事項)
- [維運操作](#維運操作)
- [故障排除](#故障排除)

---

## 系統需求

| 項目 | 最低需求 |
|------|---------|
| CPU | 1 vCPU |
| 記憶體 | 512 MB |
| 硬碟 | 1 GB（日誌與資料庫） |
| 作業系統 | 任何支援 Docker 的 Linux / NAS |
| Docker | Docker Engine 20.10+ / Docker Compose V2 |
| 網路 | 具公網 IP 或 DDNS 域名 |
| Cloudflare | 帳號 + 一個託管域名（免費方案即可） |

---

## 部署步驟

### 1. 建立 Cloudflare Tunnel

1. 登入 [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com)
2. **Networks** → **Connectors** → **Cloudflare Tunnels**
3. **Create a tunnel** → 選擇 **Cloudflared** → **Next**
4. 輸入 Tunnel 名稱（例如 `rustdesk-server`）
5. **Save tunnel**
6. 選擇環境 **Docker**，複製 Token 值（`eyJhIjoiNWFi...` 格式）

### 2. 設定 Tunnel Public Hostname

在 Tunnel 設定中加入路由，讓 Web Console 可透過 HTTPS 存取：

| 欄位 | 值 |
|------|-----|
| Subdomain | `rustdesk-console`（或自訂） |
| Domain | `example.com` |
| Type | `HTTP` |
| URL | `127.0.0.1:21114` |

### 3. 設定環境變數

複製 `.env` 範本並編輯：

```bash
# 產生 JWT 金鑰
openssl rand -base64 256
```

```env
# ── 必填 ──
RUSTDESK_DOMAIN=rustdesk-console.example.com
RUSTDESK_API_SERVER=https://rustdesk-console.example.com
CLOUDFLARE_TUNNEL_TOKEN=eyJhIjoiNWFi...
RUSTDESK_API_JWT_KEY=<openssl rand -base64 256 的輸出>

# ── 選填（預設值已在 docker-compose.yml 中設定）──
MUST_LOGIN=N                        # Y = 強制登入
RUSTDESK_API_APP_REGISTER=false     # true = 開放註冊
```

> `.env` 包含敏感 Token，已加入 `.gitignore`，切勿 commit。

#### 環境變數完整參考

`docker-compose.yml` 或 `.env` 中的環境變數與設定檔可一一對應，前綴皆為 `RUSTDESK_API_`。以下分類列出可配置的參數及說明：

##### 📌 系統與通用設定

| 變數名稱 | 說明 | 預設 / 範例 |
|------|------|------|
| `TZ` | 時區 | `Asia/Taipei` |
| `RUSTDESK_API_LANG` | 介面語言 | `zh-TW`, `en` |

##### 📌 App 與帳號管理

| 變數名稱 | 說明 | 預設 / 範例 |
|------|------|------|
| `RUSTDESK_API_APP_WEB_CLIENT` | 是否啟用 Web-Client；`1` (啟用), `0` (不啟用) | `1` |
| `RUSTDESK_API_APP_REGISTER` | 是否開啟註冊功能；`true` (開啟), `false` (關閉) | `false` |
| `RUSTDESK_API_APP_REGISTER_STATUS` | 註冊使用者的預設狀態；`1` (啟用), `2` (停用) | `1` |
| `RUSTDESK_API_APP_DISABLE_PWD_LOGIN`| 是否禁用密碼登入 (僅允許 OAuth)；`true` 或 `false` | `false` |
| `RUSTDESK_API_APP_TOKEN_EXPIRE` | Token 有效時長 | `168h` |
| `RUSTDESK_API_APP_CAPTCHA_THRESHOLD`| 驗證碼觸發條件；`-1` 不啟用，`0` 永遠啟用，`>0` 錯誤 N 次後啟用 | `3` |
| `RUSTDESK_API_APP_BAN_THRESHOLD` | 封禁 IP 觸發條件；`0` 不啟用，`>0` 登入錯誤 N 次後封禁 IP | `0` (防爆破建議設 `10`) |
| `RUSTDESK_API_APP_SHOW_SWAGGER` | 是否開放 Swagger API 文件；`1` (開放)，`0` (不開放) | `0` |

##### 📌 後台介面 (Admin)

| 變數名稱 | 說明 | 預設 / 範例 |
|------|------|------|
| `RUSTDESK_API_ADMIN_TITLE` | 後台標題自訂名稱 | `RustDesk Api Admin` |
| `RUSTDESK_API_ADMIN_HELLO` | 後台登入歡迎語 (支援 HTML 語法) | （空白） |
| `RUSTDESK_API_ADMIN_HELLO_FILE` | 後台歡迎語檔案；若內容長建議放入檔案，會覆寫上述變數 | `/app/data/hello.html` |

##### 📌 資料庫設定 (GORM / MySQL)

| 變數名稱 | 說明 | 預設 / 範例 |
|------|------|------|
| `RUSTDESK_API_GORM_TYPE` | 資料庫類型；使用 SQLite 則不需配置底下 MySQL 變數 | `sqlite` (或 `mysql`) |
| `RUSTDESK_API_GORM_MAX_IDLE_CONNS` | 資料庫最大空閒連線數 | `10` |
| `RUSTDESK_API_GORM_MAX_OPEN_CONNS` | 資料庫最大打開連線數 | `100` |
| `RUSTDESK_API_MYSQL_USERNAME` | MySQL 使用者名稱 (僅 GORM_TYPE=mysql 時需要) | `root` |
| `RUSTDESK_API_MYSQL_PASSWORD` | MySQL 密碼 | `111111` |
| `RUSTDESK_API_MYSQL_ADDR` | MySQL 連線位址 | `192.168.1.66:3306` |
| `RUSTDESK_API_MYSQL_DBNAME` | MySQL 資料庫名稱 | `rustdesk` |

##### 📌 RustDesk Server 及金鑰配置

| 變數名稱 | 說明 | 預設 / 範例 |
|------|------|------|
| `RUSTDESK_API_RUSTDESK_PERSONAL` | 是否啟用個人版 API；`1` (免登入即可用地址簿同步), `0` (裝置必須登入帳號才能使用雲端功能) | `1` |
| `RUSTDESK_API_RUSTDESK_ID_SERVER` | Rustdesk 的 ID 伺服器位址 (綁定 Port 21116) | `domain.com:21116` |
| `RUSTDESK_API_RUSTDESK_RELAY_SERVER`| Rustdesk 的中繼伺服器位址 (綁定 Port 21117) | `domain.com:21117` |
| `RUSTDESK_API_RUSTDESK_API_SERVER` | Rustdesk 的 API 伺服器位址 (走 HTTPS Tunnel 的網址) | `https://console.com` |
| `RUSTDESK_API_RUSTDESK_KEY` | Rustdesk 的公鑰字串本體 (與下方二選一) | `123456...` |
| `RUSTDESK_API_RUSTDESK_KEY_FILE` | Rustdesk 存放公鑰的檔案路徑 (通常直接吃容器內檔案) | `/data/id_ed25519.pub` |
| `RUSTDESK_API_RUSTDESK_WEBCLIENT_MAGIC_QUERYONLINE` | Web Client V2 是否啟用新線上狀態查詢；`1` (啟用), `0` (不啟用) | `0` |
| `RUSTDESK_API_RUSTDESK_WS_HOST` | 自訂 Websocket Host | `wss://...` |

##### 📌 Proxy 及 JWT 認證配置

| 變數名稱 | 說明 | 預設 / 範例 |
|------|------|------|
| `RUSTDESK_API_PROXY_ENABLE` | 是否啟用 Proxy 代理；`true` 或 `false` | `false` |
| `RUSTDESK_API_PROXY_HOST` | 代理位址 | `http://127.0.0.1:1080` |
| `RUSTDESK_API_GIN_TRUST_PROXY` | 信任的反向代理 IP 列表，以 `,` 分隔；預設為信任所有 | `192.168.1.2` |
| `RUSTDESK_API_JWT_KEY` | 自訂 JWT 簽發金鑰 (產生隨機字串填入) | （亂數金鑰） |
| `RUSTDESK_API_JWT_EXPIRE_DURATION` | JWT Token 有效時間 | `6h` |

### 4. 建立資料夾與啟動服務

> **⚠️ 避免權限錯誤（特別是 NAS 用戶）**：
> 在執行啟動指令前，請**務必**手動建立資料夾，以防 Docker daemon 以 root 身分自動建立導致後續 SQLite 初始化失敗或權限不足（Permission Denied）。

```bash
mkdir -p ./data/server ./data/api
chmod -R 777 ./data   # 確保容器內程序具有寫入權限，若注重安全請設定為 1000:1000

docker compose up -d
```

首次啟動時，S6 容器會自動產生 Ed25519 金鑰對：

```
data/
├── server/              # hbbs/hbbr 資料
│   ├── id_ed25519       # 私鑰（絕勿外洩）
│   ├── id_ed25519.pub   # 公鑰（配發給客戶端）
│   └── db_v2.sqlite3    # 設備 ID 資料庫
└── api/                 # rustdesk-api 資料
    └── rustdesk-api.db  # 帳號 / 地址簿 / 日誌（SQLite）
```

### 5. 取得公鑰

或至 `./data/server/id_ed25519.pub` 檔案查看 (可以用記事本打開)
- **終端機**：輸入 `cat ./data/server/id_ed25519.pub`
- **SSH**：輸入 `sudo cat ./data/server/id_ed25519.pub`
- **NAS GUI**：進入 Container Manager，打開 `rustdesk` 容器的「日誌 (Logs)」頁籤查看。

```bash
cat ./data/server/id_ed25519.pub
```

記下輸出的公鑰字串，後續客戶端設定需要使用。

### 6. 驗證服務狀態

```bash
# 確認兩個容器（rustdesk, cloudflared）都在運行
docker compose ps

# 檢查日誌
docker compose logs -f rustdesk
docker compose logs -f cloudflared
```

正常啟動後 hbbs 日誌應顯示：

```
INFO  Listening on tcp/udp :21116
INFO  Listening on tcp :21115, extra port for NAT test
INFO  Listening on tcp :21114, extra port for API
INFO  Listening on websocket :21118
```

### 7. 登入 Web Console

瀏覽器開啟 Cloudflare Tunnel 域名：

```text
https://rustdesk-console.example.com
```

> **🔑 首次預設登入憑證**
> - **預設使用者名稱**：`admin`
> - **密碼與 Key 獲取方式**：
>   - **終端機**：輸入 `docker logs rustdesk`
>   - **SSH**：輸入 `sudo docker logs rustdesk`
>   - **NAS GUI**：進入 Container Manager，打開 `rustdesk` 容器的「日誌 (Logs)」頁籤查看。
> 
> ![Admin Login](../images/admin.png)
> 
> 🚨 **安全警告：首次登入後請務必立即至後台修改為高強度密碼！**

---

## Port 說明與防火牆

### Port 清單

| Port | 協定 | 服務 | 用途 | 必要性 | 需開放？ |
|------|------|------|------|--------|:--------:|
| 21114 | TCP | hbbs | Web Console / API | Pro 核心 | **不需要**（走 Tunnel） |
| 21115 | TCP | hbbs | NAT 類型偵測 | 必要 | 是 |
| 21116 | TCP | hbbs | TCP 打洞 | 必要 | 是 |
| 21116 | **UDP** | hbbs | ID 註冊與心跳 | **核心** | **是** |
| 21117 | TCP | hbbr | Relay 中繼轉發 | 必要 | 是 |
| 21118 | TCP | hbbs | WebSocket | 可選 | Web 客戶端用 |
| 21119 | TCP | hbbr | WebSocket Relay | 可選 | Web 客戶端用 |

### 防火牆設定範例

**UFW (Ubuntu)**：

```bash
sudo ufw allow 21115/tcp comment 'RustDesk NAT test'
sudo ufw allow 21116/tcp comment 'RustDesk TCP hole punch'
sudo ufw allow 21116/udp comment 'RustDesk ID registration'
sudo ufw allow 21117/tcp comment 'RustDesk relay'
# 不需要：sudo ufw allow 21114/tcp  ← Web Console 走 Cloudflare Tunnel
sudo ufw --force enable
```

**firewall-cmd (CentOS / RHEL / Fedora)**：

```bash
sudo firewall-cmd --permanent --add-port=21115/tcp
sudo firewall-cmd --permanent --add-port=21116/tcp
sudo firewall-cmd --permanent --add-port=21116/udp
sudo firewall-cmd --permanent --add-port=21117/tcp
sudo firewall-cmd --reload
```

---

## 路由器 Port Forwarding

NAS 或家用伺服器位於 NAT 之後，**必須**在路由器設定 Port Forwarding：

| 外部 Port | 協定 | 轉發至 | 備註 |
|-----------|------|--------|------|
| 21115 | TCP | `<伺服器內網 IP>:21115` | |
| 21116 | **TCP** | `<伺服器內網 IP>:21116` | |
| 21116 | **UDP** | `<伺服器內網 IP>:21116` | 許多路由器預設只轉 TCP，需額外設定 UDP |
| 21117 | TCP | `<伺服器內網 IP>:21117` | |
| 21118 | TCP | `<伺服器內網 IP>:21118` | 可選（Web 客戶端） |
| 21119 | TCP | `<伺服器內網 IP>:21119` | 可選（Web 客戶端） |

> **21114/tcp 不需要做 Port Forwarding** — Web Console 完全走 Cloudflare Tunnel。

注意事項：
- 伺服器內網 IP 應設為**靜態 IP** 或 DHCP 保留
- 若公網 IP 為浮動，需設定 DDNS 並確認 DNS A 記錄自動更新

---

## Cloudflare Tunnel 設定

### 為何使用 Cloudflare Tunnel

| 比較 | IP 直連 21114 | Nginx + TLS | Cloudflare Tunnel |
|------|:------------:|:-----------:|:-----------------:|
| 加密 | 無（HTTP 明文） | TLS | TLS |
| 管理憑證 | N/A | 需手動更新 | 自動 |
| 入站 Port | 需開 21114 | 需開 80/443 | **不需開任何 port** |
| WAF/DDoS | 無 | 無 | 內建 |
| 隱藏源站 IP | 否 | 否 | 是 |

### cloudflared 容器說明

`docker-compose.yml` 中的 `cloudflared` 服務：

```yaml
cloudflared:
  image: cloudflare/cloudflared:latest
  command: tunnel run --token ${CLOUDFLARE_TUNNEL_TOKEN}
  network_mode: "service:rustdesk"  # 共享 rustdesk 容器的網路
  restart: unless-stopped
```

- 使用 `network_mode: "service:rustdesk"` 共享 rustdesk 容器的網路 namespace，可存取 127.0.0.1:21114
- Token 從 `.env` 讀取，不寫死在 compose 檔中
- `cloudflared` 僅建立**出站連線**至 Cloudflare Edge，不監聽任何入站 port
- 定期輪換 Token：Cloudflare Dashboard → Tunnel → 重新產生

### 關於 Cloudflare Access

> ⚠️ **不建議**在此 Tunnel 域名上啟用 Cloudflare Access（Zero Trust 存取策略）。
>
> RustDesk 客戶端的 API 呼叫（地址簿同步、裝置管理等）走 HTTP REST，
> 無法處理 Cloudflare Access 的 OAuth/Email 驗證流程，會直接失敗。
>
> 安全性已由以下層級保障：
> - Cloudflare Tunnel（TLS 加密 + 隱藏源站 IP）
> - Cloudflare WAF / DDoS 防護
> - rustdesk-api 內建帳號密碼 + JWT 驗證

---

## 安全配置說明

### 社群版 vs 官方開源版設計差異

| 項目 | 官方開源版 | 社群版 (lejianwen) |
|------|--------|--------|
| Server 映像檔 | `rustdesk/rustdesk-server` | `lejianwen/rustdesk-server-s6` |
| Web Console / API | 無 | 有（rustdesk-api 內建） |
| hbbs command | `hbbs -r <domain>:21117 -k _` | S6 自動管理 |
| 強制登入才能連線 | 無 | 有（`MUST_LOGIN=Y`） |
| JWT 驗證 | 無 | 有（`RUSTDESK_API_JWT_KEY`） |
| 裝置數量限制 | 無 | **無** |
| 網路模式 | bridge 或 host | bridge 或 host 皆可 |

### 容器加固

| 措施 | 作用 | 套用對象 |
|------|------|---------|
| `read_only: true` | 容器檔案系統唯讀，阻擋惡意寫入 | cloudflared |
| `tmpfs: /tmp` | 暫存使用記憶體，不落地磁碟 | cloudflared |
| `no-new-privileges` | 禁止容器內程序提權 | rustdesk, cloudflared |

### 金鑰安全

- `id_ed25519`（私鑰）**絕不可外洩或傳輸**，僅留在伺服器 `data/server/` 目錄
- `id_ed25519.pub`（公鑰）需安全傳遞給客戶端使用者
- 若懷疑金鑰洩漏，刪除 `data/server/id_ed25519*` 後重啟服務即可重新生成

---

## 帳號安全

### 修改預設密碼

首次登入 Web Console 後**立即修改**。預設帳密 `admin / admin` 為公開資訊。

### 緊急恢復

```bash
# 重置管理員密碼（透過 rustdesk-api CLI）
docker exec -it rustdesk /app/rustdesk-api resetpass
```

---

## NAS 部署注意事項

### 檔案權限地雷 (Permission Denied)

在群暉 (Synology) 或 QNAP 部署時，若交由 Docker 自動建立掛載卷 (Volumes) 資料夾，將會以 `root` 身分建立，導致 API 容器內的 user 權限無法建立 SQLite 資料庫。

**解決方案**：
在啟動 Docker 容器前，請先手動建立資料夾並賦予權限（可透過 NAS 內的 File Station / 檔案總管 GUI 介面或 SSH 操作）：
1. 建立 `data` 資料夾。
2. 在 `data` 內建立 `server` 與 `api` 兩個子資料夾。
3. 將這三個資料夾的權限設定為允許「Everyone 讀寫」或明確賦予容器執行使用者的權限（GUI 右鍵設定屬性，或 SSH 執行 `chmod -R 777 data`）。

### 網路模式

社群版不強制 `network_mode: "host"`，可使用標準 bridge 網路 + port mapping。
這讓 NAS 部署更容易：

- **Synology DSM Container Manager**：直接導入 `docker-compose.yml`
- **QNAP Container Station**：支援 Docker Compose
- **TrueNAS**：原生 Docker/Podman 支援

### DDNS

若你的公網 IP 為浮動（如家用寬頻），需確認：

- DNS A 記錄搭配 DDNS 自動更新
- Cloudflare Tunnel **不受 IP 變動影響**（outbound-only），Web Console 永遠可存取
- 但客戶端直連的 `rustdesk-server.example.com` 必須解析到正確的公網 IP

---

## 維運操作

### 更新映像檔

```bash
docker compose pull
docker compose up -d
```

### 備份

需備份 `data/` 目錄，包含金鑰對與資料庫：

```bash
tar -czf rustdesk-backup-$(date +%Y%m%d).tar.gz ./data/
```

### 查看連線日誌

> ⚠️ **警告**：請**不要**透過 NAS 介面裡的「新增終端機 (Terminal)」並試圖在裡面執行 `docker compose` 指令（會出現 `executable file not found` 錯誤）。如果你使用 GUI 管理器，請直接點選該容器的 **「日誌 (Log)」** 頁籤查看。

```bash
# 終端機：
docker logs -f rustdesk     # S6 整合容器 (hbbs + hbbr + API) 及其密碼/Key
docker logs -f cloudflared  # Cloudflare Tunnel

# SSH：
sudo docker logs -f rustdesk
sudo docker logs -f cloudflared
```

### 重新生成金鑰

```bash
docker compose down
rm ./data/server/id_ed25519 ./data/server/id_ed25519.pub
docker compose up -d
# 重新取得新公鑰並更新到所有客戶端
cat ./data/server/id_ed25519.pub
```

---

## 故障排除

### 客戶端顯示「尚未就緒：請檢查您的網路連線」

**常見原因**：

1. 確認映像檔為 `lejianwen/rustdesk-server-s6`
2. **API 伺服器欄位錯誤** — 應填 `https://rustdesk-console.example.com`（Tunnel 域名），而非 `http://...:21114`
3. **Tunnel 未啟動** — 執行 `docker compose logs cloudflared` 檢查

### 客戶端無法連線（ID 註冊失敗）

1. **檢查 UDP 21116 是否開通**：

   ```bash
   # 從外網測試
   nc -zuv rustdesk-server.example.com 21116
   ```

2. **路由器 Port Forwarding** — 確認 UDP 21116 已設定（很多路由器預設只轉 TCP）
3. **確認客戶端的 ID Server 與 Key 是否正確**

### Web Console 無法存取

1. **確認 rustdesk 和 cloudflared 容器都在運行**：

   ```bash
   docker compose ps
   ```

2. **確認 Tunnel Token 正確**：

   ```bash
   docker compose logs cloudflared | head -20
   ```

3. **確認 Cloudflare Tunnel Public Hostname 設定正確**（指向 `http://127.0.0.1:21114`）

### 連線頻繁斷開

- 檢查伺服器頻寬是否足夠
- 檢查容器是否被 OOM Kill：

  ```bash
  docker inspect rustdesk | grep -i oom
  ```

### 日誌顯示 `key mismatch`

客戶端填入的 Key 與伺服器的 `id_ed25519.pub` 不一致。重新複製公鑰並更新客戶端設定。
