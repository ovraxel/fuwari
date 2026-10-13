---
title: Chatto 自架完整指南：Docker Compose + Nginx Proxy Manager
published: 2026-10-13
description: 從零部署 Chatto，搭配 Nginx Proxy Manager、LiveKit、備份與推播的完整實戰指南。
image: ''
tags: [Chatto, Docker, 自架, Nginx, LiveKit]
category: 教學
draft: false
---

# Chatto 自架完整指南：Docker Compose + Nginx Proxy Manager

> 本文整理自實際部署過程中的所有問題與解法。範例域名一律用 `chat.example.com`，請替換成你自己的。
> 標註「**未驗證**」的內容，是我沒能在官方文件或實機上確認的部分，使用前請自行核對。

---

## 目錄

1. [架構概覽](#1-架構概覽)
2. [部署前準備](#2-部署前準備)
3. [部署 Chatto（Docker Compose）](#3-部署-chattodocker-compose)
4. [反向代理：Nginx Proxy Manager](#4-反向代理nginx-proxy-manager)
5. [首次登入與管理後台](#5-首次登入與管理後台)
6. [註冊控制與帳號管理](#6-註冊控制與帳號管理)
7. [備份與還原](#7-備份與還原)
8. [推播通知（Web Push）](#8-推播通知web-push)
9. [語音／視訊通話（LiveKit）](#9-語音視訊通話livekit)
10. [客戶端：PWA、桌面版、自編譯 APK](#10-客戶端pwa桌面版自編譯-apk)
11. [升級版本](#11-升級版本)
12. [常見問題與錯誤排查](#12-常見問題與錯誤排查)
13. [指令速查](#13-指令速查)
14. [未驗證事項清單](#14-未驗證事項清單)

---

## 1. 架構概覽

官方 Docker Compose 範例由四個服務組成：

| 服務 | 作用 | 內部埠 |
|---|---|---|
| **Chatto** | 網頁應用與 API、WebSocket | 4000 |
| **NATS** | 儲存所有聊天資料（JetStream） | 4222（**不可對外開放**） |
| **LiveKit** | 語音／視訊通話（WebRTC） | 7880（信令）、7881/3478/50000-50200（媒體） |
| **Caddy** | 自動 HTTPS 與反向代理 | 80/443 |

本指南的情境是：**已有 Nginx Proxy Manager（NPM）佔用 80/443**，所以移除 Caddy，由 NPM 負責 HTTPS 與轉發。

```text
瀏覽器 ──HTTPS──► NPM(80/443) ──► chatto:4000
         ──WSS────► NPM(80/443) ──► livekit:7880
         ──UDP/TCP 直連──────────► LiveKit(7881, 3478, 50000-50200)   ← 不經過 NPM
```

**關鍵觀念：**

- 資料全部在 NATS 的 `nats_data` volume，要定期備份。
- 語音媒體**不能**走 HTTP 反向代理，必須直接開放埠。
- 官方對外部代理的要求只有兩條路由：Chatto 域名 → `chatto:4000`；LiveKit 域名 → `livekit:7880`，兩者都需要公開信任的 TLS 憑證。

---

## 2. 部署前準備

### 2.1 需求

- Docker 與 Compose v2
- 一台有公網 IP 的伺服器
- **SMTP 帳號**（註冊驗證碼、密碼重設都靠郵件發送）

### 2.2 DNS

建立兩筆記錄指向伺服器：

| 用途 | 範例 | 備註 |
|---|---|---|
| Chatto | `chat.example.com` | 用 Cloudflare 時可橘雲 |
| LiveKit | `livekit-chat.example.com` | **必須灰雲（DNS only）** |

> ⚠️ **LiveKit 域名建議用「一層子域名」**（如 `livekit-chat.example.com`），不要用 `livekit.chat.example.com`。
> Cloudflare 免費的 Universal SSL 只涵蓋 `example.com` 與 `*.example.com`，**不涵蓋二層子域名**（`*.chat.example.com`），走橘雲時會導致 `tls: handshake failure`。

### 2.3 防火牆（主機與雲端安全組都要開）

| 埠 | 協定 | 用途 |
|---|---|---|
| 80、443 | TCP | NPM（HTTP/HTTPS） |
| 7881 | TCP | WebRTC 媒體後備 |
| 3478 | UDP | TURN/STUN |
| 50000–50200 | UDP | WebRTC 直連媒體 |

**不要**對外開放 NATS 的 4222。雲廠商安全組最常漏開 UDP。

---

## 3. 部署 Chatto（Docker Compose）

### 3.1 取得官方範例並生成設定

```bash
cd /opt
git clone --depth 1 https://github.com/chattocorp/chatto.git chatto
cd chatto/examples/dockercompose
./init-env.sh chat.example.com admin@example.com
```

`init-env.sh` 會生成 `.env` 和 `livekit.generated.yaml`，裡面的 NATS、Chatto、LiveKit 密鑰互相匹配。官方也說明，生成的 `.env` 會啟用本機 Operator API socket。

> 之後**所有 `docker compose` 指令都要在 `/opt/chatto/examples/dockercompose` 執行**。Compose 專案名稱預設取目錄名，所以網路名稱會是 `dockercompose_default`。

### 3.2 編輯 `.env`

至少要確認這幾項：

```ini
# 第一個帳號的信箱，註冊後會自動成為 owner
CHATTO_OWNERS_EMAILS=admin@example.com

# SMTP（用 465 埠的 SMTPS 時加上 CHATTO_SMTP_TLS=implicit）
CHATTO_SMTP_ENABLED=true
CHATTO_SMTP_HOST=smtp.example.com
CHATTO_SMTP_PORT=587
CHATTO_SMTP_FROM=noreply@example.com
CHATTO_SMTP_USERNAME=...
CHATTO_SMTP_PASSWORD=...

# 瀏覽器直連的 LiveKit 地址：必須是 wss://，且是 LiveKit 的域名
CHATTO_LIVEKIT_URL=wss://livekit-chat.example.com
```

驗證沒有殘留佔位值：

```bash
grep -n -i -E "PUBLIC_URL|LIVEKIT_URL|OWNERS_EMAILS|SMTP_HOST" .env
grep -n "example.com" .env      # 只應出現你自己的域名，不應有範例值
```

### 3.3 從 `compose.yml` 移除 Caddy

NPM 已佔用 80/443，Caddy 必須移除。要刪的內容：

1. 整個 `caddy:` 服務（從 `caddy:` 到它底下最後一個 `- caddy_config:/config`）。
2. 頂層 `volumes:` 裡的 `caddy_data:`、`caddy_config:`。
3. `Caddyfile` 檔案可留可刪，沒有服務掛載它就不影響。

**必須保留：**

- `nats_data:` volume（所有聊天資料都在裡面）。
- `livekit` 服務的 `ports`：
  ```yaml
  ports:
    - "50000-50200:50000-50200/udp"
    - "7881:7881/tcp"
    - "3478:3478/udp"
  ```
- `.env` 裡的 `PUBLIC_URL`（它是 `.env` 變數，不是 compose 裡要刪的東西）。

刪除後驗證：

```bash
grep -n -i caddy compose.yml          # 只應剩註解（例如 "proxied through Caddy"）
docker compose config > /dev/null && echo OK
```

> 常見手誤：只刪了 `caddy:` 標題，留下它底下的 `environment:`、`volumes:`，會被併進上一個服務，造成 `mapping key "environment" already defined`。**凡是在同一個服務裡第二次出現的 key，都是 Caddy 殘留**。

### 3.4 啟動

```bash
docker compose up -d
docker compose ps         # 應看到 chatto、nats、livekit，沒有 caddy
docker compose logs --tail 30 chatto
```

---

## 4. 反向代理：Nginx Proxy Manager

先確認你的 80/443 是誰在接管：

```bash
ss -tlnp | grep -E ':80 |:443 '
docker ps --format '{{.Names}}\t{{.Image}}\t{{.Ports}}'
```

看到 `docker-proxy` 與 `jc21/nginx-proxy-manager` 容器，表示是 **NPM 在容器裡**。此時：

- 主機上沒有 `nginx` 指令（`nginx: command not found` 是正常的）。
- **不需要**手動建立 `/etc/nginx/conf.d/chatto.conf`，所有設定在 NPM 後台完成。

### 4.1 為什麼會 502

**NPM 在容器裡，它填的 `127.0.0.1:4000` 指的是 NPM 容器自己**，不是主機，所以連不到 Chatto。解法是讓 NPM 與 Chatto 共用 Docker 網路，上游直接用服務名 `chatto:4000`。

### 4.2 把 NPM 接進 Chatto 的網路

```bash
docker network ls | grep dockercompose        # 確認網路名稱
docker network connect dockercompose_default npm-app-1

# 測試 NPM 能否連到兩個服務
docker exec npm-app-1 curl -sI http://chatto:4000 | head -1     # 應回 HTTP/1.1 200
docker exec npm-app-1 curl -sI http://livekit:7880 | head -1    # 應回 HTTP/1.1 200
```

> 若回報 `endpoint ... already exists in network`，表示已經接上了，直接測 curl 即可。

### 4.3 在 NPM 後台建立兩個 Proxy Host

後台網址：`http://你的IP:81`。

| 域名 | Scheme | Forward Hostname | Port |
|---|---|---|---|
| `chat.example.com` | http | `chatto` | 4000 |
| `livekit-chat.example.com` | http | `livekit` | 7880 |

兩個都要：

- 勾選 **Websockets Support**
- **SSL 分頁**：Request a new SSL Certificate（Let's Encrypt），勾選 Force SSL、HTTP/2

**Chatto 那個 Host 的 Advanced 分頁**（Custom Nginx Configuration）加入：

```nginx
client_max_body_size 100m;
proxy_read_timeout 3600s;
proxy_send_timeout 3600s;
```

- `client_max_body_size`：預設只有 1MB，不加的話上傳圖片和檔案會失敗。
- `proxy_read_timeout`：避免閒置的 WebSocket 連線被切斷。
- Websockets Support 勾選後，`Upgrade`/`Connection` 標頭 NPM 會自動加，Advanced 裡不用重寫。

> 不要直接改 `/data/nginx/proxy_host/*.conf`（NPM 容器內），後台存檔時會被覆蓋。

### 4.4 讓網路連線永久生效

`docker network connect` 在 NPM 容器重建後會消失，要寫進 NPM 自己的 compose：

```bash
docker inspect npm-app-1 --format '{{index .Config.Labels "com.docker.compose.project.working_dir"}}'
```

進入該目錄，在 NPM 服務下加上（`app` 要換成你實際的服務名稱）：

```yaml
services:
  app:
    # 原有設定不動
    networks:
      - default
      - chatto_net

networks:
  chatto_net:
    external: true
    name: dockercompose_default
```

然後 `docker compose up -d`。

> ⚠️ **重新部署（刪除舊網路）前**，要先 `docker network disconnect <舊網路> npm-app-1`，並把 NPM compose 裡引用的舊網路名稱改成新的，否則會出現 `network ... not found`，NPM 可能無法重建。

### 4.5 如果你用的是主機上的 Nginx（非 NPM）

把 Chatto 的埠映射改成只綁本機：

```yaml
    ports:
      - "127.0.0.1:4000:4000"
```

Nginx 站點設定：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}

server {
    listen 443 ssl http2;
    server_name chat.example.com;
    # ssl_certificate / ssl_certificate_key ...

    client_max_body_size 100m;

    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 3600s;
    }
}
```

CentOS/RHEL 系列若有 SELinux 擋住：`sudo setsebool -P httpd_can_network_connect 1`。

### 4.6 Cloudflare 設定要點

- Chatto 域名：橘雲可以，SSL/TLS 模式用 **Full (strict)**，不要用 Flexible。
- LiveKit 域名：**灰雲**。
- 兩個域名的 DNS 要指向伺服器公網 IP（`dig +short 域名` 對照 `curl -4 ifconfig.me`）。

---

## 5. 首次登入與管理後台

1. 打開 `https://chat.example.com`。
2. 用 `CHATTO_OWNERS_EMAILS` 的信箱註冊，驗證碼會寄到信箱，註冊後自動成為 **owner**。
3. **管理後台**（官方稱 Admin UI）**沒有獨立網址**，是登入後網站內建的功能，需要 owner 或有權限的管理員帳號。依官方文件，它涵蓋成員管理、角色與權限、房間排列、伺服器品牌設定、事件日誌與系統診斷。

> **未驗證**：後台入口在介面上的確切位置與按鈕名稱我沒看過。通常在伺服器名稱旁的選單或用戶選單裡，找 Admin / Server settings 之類的項目。
> 頂部工具列的**齒輪圖示**是 Chatto 的設定（含 Appearance），不是外殼 App 的設定。

**房間結構**：房間（room）可歸入房間群組（room group），群組同時是權限邊界。官方有專頁 *Building Your Community Structure*：<https://docs.chatto.run/guides/operations/community-structure/>（我沒能讀到內容，操作細節請看該頁）。

**owner 帳號沒有 owner 權限時：** 確認 `CHATTO_OWNERS_EMAILS` 包含該帳號的已驗證信箱，**重啟 Chatto**（owner 角色在啟動時套用）：

```bash
docker compose restart chatto
```

---

## 6. 註冊控制與帳號管理

### 6.1 關閉公開註冊

在 `.env` 加入（官方設定）：

```ini
CHATTO_AUTH_DIRECT_REGISTRATION=false
```

套用：

```bash
docker compose up -d --force-recreate chatto
```

**操作順序：先完成你自己的 owner 註冊，再關閉註冊。**

> ⚠️ **值只能是 `true` 或 `false`**，多一個字元就會啟動失敗。曾出現 `falsex` 造成：
> `strconv.ParseBool: parsing "falsex": invalid syntax`，容器起不來，網站 502。
> 建議用 `sed` 寫入而非手打：
> ```bash
> sed -i 's|^CHATTO_AUTH_DIRECT_REGISTRATION=.*|CHATTO_AUTH_DIRECT_REGISTRATION=false|' .env
> grep -n DIRECT_REGISTRATION .env
> ```

關閉後的影響：

- 登入頁不再顯示註冊／Join 入口（網頁、PWA、Android 客戶端都一樣）。
- 官方說明：已設定的 SSO 登入方式仍可使用。
- 要開回來就把值改成 `true` 並重建；開放期間任何知道網址的人都能註冊，用完建議立即關閉。

### 6.2 用 Operator CLI 管理帳號

Operator API 透過容器內的 unix socket（`/tmp/chatto/operator.sock`）運作，**預設不一定啟用**；官方說 `init-env.sh` 生成的 `.env` 會啟用它。

```bash
docker compose exec -u chatto chatto /chatto operator user list
docker compose exec -u chatto chatto /chatto operator user list --search someone@example.com
docker compose exec -u chatto chatto /chatto operator user get USER_ID
docker compose exec -u chatto chatto /chatto operator user update USER_ID --display-name "Alice"
docker compose exec -u chatto chatto /chatto operator user set-password USER_ID
docker compose exec -u chatto chatto /chatto operator user add-email USER_ID --email alice@example.com
docker compose exec -u chatto chatto /chatto operator user role add USER_ID moderator
docker compose exec -u chatto chatto /chatto operator user delete USER_ID --yes
```

- 設定密碼建議用 `--password-stdin` 或 `--password-file`，不要用 `--password`（會留在 shell 歷史）。
- **「建立用戶」的指令**：官方文件提到操作者可以建立用戶，但我沒在文件列出的指令裡看到確切參數（**未驗證**）。請用說明查詢：
  ```bash
  docker compose exec -u chatto chatto /chatto operator user --help
  docker compose exec -u chatto chatto /chatto operator user create --help
  ```
- **安全**：能使用 operator socket 的人等同於 Chatto 的 root 權限。**不要**把 socket 掛載到主機或別的容器，也不要用網路暴露。

> 若出現 `dial unix /tmp/chatto/operator.sock: connect: no such file or directory`，表示 Operator API 沒有啟用。官方設定項為 `operator_api.enabled = true`，環境變數名稱依命名規則推測為 `CHATTO_OPERATOR_API_ENABLED=true`（**未驗證**，請以 `.env.example` 或官方環境變數參考為準）。

---

## 7. 備份與還原

Chatto 內建 `chatto backup`，會把用戶、房間、訊息、NATS 內的附件打包成單一檔案，並支援加密。官方對小型自架的建議是 `--encrypt --include-keys`，單一檔案就是完整備份。

**備份指令不包含設定檔**（`.env`、`livekit.generated.yaml`），這些要另外備份。

### 7.1 一次性準備

官方 compose 的 `chatto` 服務已經有備份目錄掛載：

```yaml
    volumes:
      - ${CHATTO_BACKUP_DIR:-./backups}:/backups
```

所以**不要再重複掛一次 `/backups`**，只需在同一個 `volumes:` 下加上密碼檔：

```yaml
    volumes:
      - ${CHATTO_BACKUP_DIR:-./backups}:/backups
      - ./secrets/backup-passphrase.txt:/run/secrets/chatto-backup-passphrase:ro
```

建立目錄、密碼檔並設定擁有者（容器預設以 `1000:1000` 執行）：

```bash
cd /opt/chatto/examples/dockercompose
mkdir -p backups secrets
openssl rand -base64 32 > secrets/backup-passphrase.txt
chmod 600 secrets/backup-passphrase.txt
chown -R 1000:1000 backups secrets

ls -l secrets/backup-passphrase.txt     # 開頭必須是 -（檔案），不能是 d（目錄）
docker compose up -d
```

> ⚠️ 先建好檔案**再**啟動。如果來源檔案不存在，Docker 會在主機上自動建立同名**目錄**，之後會出現：
> `read /run/secrets/chatto-backup-passphrase: is a directory`
> 修法：`rm -rf secrets/backup-passphrase.txt`，重新產生檔案，再 `docker compose up -d --force-recreate chatto`。

### 7.2 手動備份

```bash
cd /opt/chatto/examples/dockercompose

# 資料
docker compose exec -T -u chatto chatto /chatto backup --encrypt --include-keys \
  --passphrase-file /run/secrets/chatto-backup-passphrase \
  -o /backups/chatto-$(date -u +%Y%m%d-%H%M%S).tar.gz.age

# 設定檔（含密鑰，注意保管）
tar czf backups/config-$(date -u +%Y%m%d).tar.gz \
  .env livekit.generated.yaml compose.yml secrets/backup-passphrase.txt
chmod 600 backups/config-*.tar.gz

ls -lh backups/
```

Chatto 運行中即可備份，不需停機。

> 若不確定是否需要 `-c chatto.toml`：Compose 部署用環境變數設定，我沒驗證過是否需要。報設定相關錯誤時再處理（**未驗證**）。

### 7.3 定時備份

建立 `/opt/chatto-backup.sh`：

```bash
#!/bin/bash
set -euo pipefail

COMPOSE_DIR="/opt/chatto/examples/dockercompose"
RETENTION_DAYS=14

cd "$COMPOSE_DIR"
TS=$(date -u +%Y-%m-%dT%H-%M-%SZ)

docker compose exec -T -u chatto chatto /chatto backup --encrypt --include-keys \
  --passphrase-file /run/secrets/chatto-backup-passphrase \
  -o "/backups/$TS.tar.gz.age"

find "$COMPOSE_DIR/backups" -name "*.tar.gz.age" -mtime +$RETENTION_DAYS -delete
echo "Backup complete: $TS"
```

```bash
chmod +x /opt/chatto-backup.sh
/opt/chatto-backup.sh        # 先手動跑一次確認
(crontab -l 2>/dev/null; echo '30 3 * * * /opt/chatto-backup.sh >> /var/log/chatto-backup.log 2>&1') | crontab -
crontab -l
```

> `exec` 要加 `-T`（cron 沒有 TTY）。

### 7.4 異地備份與密碼保管（最重要）

- **只放在同一台 VPS 上等於沒備份**：用 `rclone`、`rsync` 或 `scp` 同步到另一台機器或雲端儲存（S3、R2、B2 等）。
- **密碼檔 `backup-passphrase.txt` 另外存一份**到密碼管理器，沒有它加密的備份打不開。
- 備份含有解密訊息的金鑰，務必保持加密。

### 7.5 還原（**未驗證**，依官方說明推導）

還原前必須先停止 Chatto，**NATS 要保持運行**：

```bash
docker compose stop chatto
docker compose run --rm -u chatto chatto /chatto restore /backups/某個備份.tar.gz.age \
  --passphrase-file /run/secrets/chatto-backup-passphrase
docker compose start chatto
```

預設遇到已有資料會報錯；覆蓋現有資料需加 `--conflict=overwrite`（**具破壞性**，確認後再用）。建議在測試環境完整驗證一次還原流程。

### 7.6 權限錯誤

出現 `mkdir /backups/.chatto-backup-...: permission denied`：

```bash
ls -ldn backups                                   # 看擁有者數字
docker compose exec -T -u chatto chatto id        # 容器內 chatto 用戶的 uid
chown -R 1000:1000 backups                        # 數字換成上面查到的 uid:gid
```

Docker 自動建立的目錄是 root（`0 0`）擁有，容器內的 `chatto` 用戶（1000）寫不進去。CentOS/RHEL 若仍失敗，掛載改成 `./backups:/backups:z`。

### 7.7 備援：直接打包 volume（**不建議作為主要方式**）

```bash
docker compose stop chatto nats
docker run --rm -v dockercompose_nats_data:/data -v "$PWD/backups":/backup alpine \
  tar czf /backup/nats-data-$(date -u +%Y%m%d).tar.gz -C /data .
docker compose start nats chatto
```

有短暫停機、沒有加密、不是官方建議方式。volume 名稱以 `docker volume ls | grep nats` 為準。

---

## 8. 推播通知（Web Push）

官方說明：**Web Push 在提供 VAPID 金鑰前是關閉的**，沒設定時 Chatto 會把推播介面整個隱藏，所以手機上找不到通知選項，多半是伺服器還沒設定。

### 8.1 伺服器設定

產生金鑰（用 Docker，不必另外裝 Node）：

```bash
docker run --rm node:22-alpine npx --yes web-push generate-vapid-keys
```

在 `.env` 加入四行（**缺一不可**，包括 `CHATTO_PUSH_ENABLED`）：

```ini
CHATTO_PUSH_ENABLED=true
CHATTO_PUSH_VAPID_PUBLIC_KEY=你的公鑰
CHATTO_PUSH_VAPID_PRIVATE_KEY=你的私鑰
CHATTO_PUSH_VAPID_SUBJECT=you@example.com
```

套用並驗證：

```bash
docker compose up -d --force-recreate chatto
docker compose exec chatto env | grep CHATTO_PUSH_ENABLED     # 應輸出 =true
docker compose logs --tail 20 chatto
```

注意事項：

- **私鑰要備份好、不要重新產生**，更換私鑰會讓所有裝置的訂閱失效。
- `CHATTO_PUSH_VAPID_SUBJECT` 是推播服務商聯絡你的方式，**不必與 SMTP 信箱一致**，填一個你收得到信的信箱即可（官方範例是不帶 `mailto:` 的純信箱）。這個值之後可以改，不影響既有訂閱。
- 公開網址（`webserver.url`／`PUBLIC_URL`）要是對外的 HTTPS 來源。
- 如果 compose 的 `chatto` 服務**沒有** `env_file: .env`，而是在 `environment:` 逐行列變數，就要把這四個變數也加進 `environment:`。查詢：`grep -n -E "env_file|CHATTO_PUSH" compose.yml`。

### 8.2 iPhone 的使用方式

iOS 的網頁推播**只對「加入主畫面」的 PWA 有效**，在 Safari 分頁裡直接開是收不到的。需要 iOS / iPadOS 16.4 以上。

1. 用 **Safari** 打開 Chatto 網址並登入。
2. 分享按鈕 →「加入主畫面」。
3. **從主畫面圖示**打開 Chatto。
4. 在 Chatto 設定裡啟用推播，系統詢問時選「允許」（要由使用者點按某個按鈕才會跳出詢問）。
5. iOS 的 **設定 → 通知 → Chatto** 確認已允許。

伺服器設定好後，要**重新打開 App**（官方說明 App 開啟時會更新伺服器上的訂閱），必要時刪除圖示重新加入。

### 8.3 其他說明

- 推播是**每台裝置各自訂閱**。
- 勿擾模式（DND）下不會響鈴也不會推送 Web Push，但通知仍會記錄。
- 房間通知層級預設只對私訊、@提及、討論串回覆通知，要收到所有訊息需把該房間設為「All messages」。
- Mac Safari（macOS 13+）也支援，可用「加入 Dock」。
- **版本提醒（未驗證）**：官方文件對應最新版。若你的伺服器是舊版（如 0.4.25），不確定是否支援 Web Push 及變數名稱是否相同。

---

## 9. 語音／視訊通話（LiveKit）

通話分兩段：

1. **信令**：瀏覽器用 WebSocket 連 LiveKit 域名（經過 NPM）。
2. **媒體**：音訊直接連伺服器的 UDP/TCP 埠（**不經過 NPM**）。

### 9.1 設定檢查清單

- `.env` 的 `CHATTO_LIVEKIT_URL=wss://livekit-chat.example.com`（`wss://`，不是 `ws://`、也不是 `https://`；是 LiveKit 的域名，不是 Chatto 的）。
- `.env` 的 LiveKit 金鑰／密鑰要和 `livekit.generated.yaml` 的 `keys:` 完全一致（`init-env.sh` 已處理）。
- LiveKit 域名 DNS 是**灰雲**，並在 NPM 簽有 Let's Encrypt 憑證。
- 防火牆開放 TCP 7881、UDP 3478、UDP 50000–50200（主機＋雲端安全組）。
- 瀏覽器必須是 **HTTPS** 並允許麥克風權限。

### 9.2 改 LiveKit 域名的步驟

1. 改 `.env`：
   ```bash
   sed -i 's|^CHATTO_LIVEKIT_URL=.*|CHATTO_LIVEKIT_URL=wss://livekit-chat.example.com|' .env
   grep -n CHATTO_LIVEKIT_URL .env
   ```
2. DNS 新增記錄（灰雲），舊記錄可刪。
3. NPM 新增／修改 Proxy Host（`livekit:7880`、Websockets、SSL）。
4. 重建：
   ```bash
   docker compose up -d --force-recreate chatto livekit
   ```
5. 驗證：
   ```bash
   curl -I https://livekit-chat.example.com                    # 應 200
   curl -s https://livekit-chat.example.com                    # LiveKit 健康檢查，應回 OK
   docker compose logs --since 2m chatto | grep -i livekit     # 不應再有新的錯誤
   ```

> `curl -s` 若回一大段 HTML 且含 *Congratulations! Your website is now running on Nginx Proxy Manager*，表示 NPM 沒匹配到這個域名，回的是預設頁面（200 OK 但不是 LiveKit）。檢查 Proxy Host 的 Domain Names 與狀態是否 Online。

### 9.3 症狀對照

| 症狀 | 最可能原因 |
|---|---|
| 點加入語音立刻報錯 | `CHATTO_LIVEKIT_URL` 錯誤、LiveKit 域名 502／憑證問題、金鑰不一致 |
| 能進頻道但沒聲音，之後斷線 | UDP 埠沒開，或 LiveKit 域名走了橘雲 |
| 只有部分網路下不通 | TCP 7881 沒開（阻擋 UDP 的網路會走 TCP 後備） |
| 部分使用者失敗 | 網路完全阻擋 UDP，需要 TURN/TLS（需另外的 TURN 域名與憑證） |

### 9.4 如何不需要語音

不需要通話時可以：移除 `livekit` 服務與其 `depends_on`、刪除 LiveKit 設定檔、移除 `CHATTO_LIVEKIT_*` 變數；也就不需要 LiveKit 子域名與相關埠。

---

## 10. 客戶端：PWA、桌面版、自編譯 APK

### 10.1 官方支援的方式：PWA

目前沒有官方原生手機 App，官方專注在 PWA（含語音通話與推播）。

- **Android（Chrome）**：打開網址 → 選單 →「安裝應用程式」。
- **iOS（Safari）**：分享 →「加入主畫面」（**必須用 Safari**）。
- **桌面**：Chrome/Edge 網址列的「安裝」圖示。

### 10.2 社群 Tauri 客戶端（chatto-tauri）

非官方專案：<https://github.com/teal-bauer/chatto-tauri>。它是桌面與 Android 外殼，在原生 webview 載入你的 Chatto 伺服器，加上背景通知、系統匣、自動啟動等。預設伺服器是 `https://chat.chatto.run`，可在設定裡改（行動版是側欄的 `chatto://` 圖示）。

- **官方 Release 的 APK 預設連官方伺服器**，要自己手動改成你的網址。
- 授權為 AGPL-3.0，修改後對外發布 APK 要公開修改的原始碼。
- 同一支手機上，自編譯與官方 Release 的 APK **套件名稱相同、簽名不同，不能共存**，安裝其中一個前要先解除安裝另一個。

### 10.3 自編譯 APK：把你的伺服器設為預設

**不要在 Termux 或 iSH 上編譯**（Android SDK/NDK 工具鏈跑不起來），也不建議佔用正在服務的 VPS。最省事的做法是 **Fork + GitHub Actions**。

**需要改的三個檔案、四處**（搜尋 `chat.chatto.run` 可找到）：

| 檔案 | 位置 | 說明 |
|---|---|---|
| `src-tauri/src/lib.rs` | 第 23 行 `DEFAULT_SERVER_URL` | Rust 主程式的預設伺服器 |
| `src-tauri/gen/android/app/src/main/java/run/chatto/desktop/NotificationService.kt` | 第 39 行 `DEFAULT_SERVER_URL` | **Android 背景通知服務**的預設值，漏改的話通知可能連到官方伺服器 |
| `src/routes/+page.svelte` | 第 83 行（備援網址）、第 198 行（placeholder） | 輸入框留空時的網址／提示文字 |

README 與 `lib.rs:17` 的註解不用改。在 Fork 裡用網頁編輯（檔案 → 鉛筆圖示 → Commit changes），只換網址，不動其他字元。不要改 `identifier`（Android 專案已用該套件名稱產生，單改設定檔會對不上）。

**編譯：**

1. Fork 的 **Actions** 分頁 → 啟用 workflows。
2. 左側選 **Build** → **Run workflow** → Branch 選 `main`。
3. 這個 workflow 會同時跑 macOS／Linux／Windows／Android，**只需要 `build-android`**，其他可取消。
4. 完成後在該次執行頁面最下方的 **Artifacts** 下載 `chatto-android`（zip），解壓得到 `.apk`。

要點：

- **不要用推送標籤（tag）觸發**，那條路走 `tauri-action` 發佈流程，需要你沒有的 Secrets。
- Android 簽名**不需要任何 Secrets**：workflow 每次執行會自己產生 keystore（密碼 `android`）來簽名。
- 檔名雖含 `-unsigned.apk`，但 `apksigner` 是就地簽名，**檔案其實已簽名**，不必再簽一次。
- **每次編譯的簽名都不同**，所以更新時不能直接覆蓋安裝，要先解除安裝。想要能直接更新，需改 workflow 改用固定 keystore（存在 Secrets）。
- 裝新 APK 前建議先解除安裝舊版，或清除舊 App 的資料，否則可能殘留舊的伺服器設定。
- 如果編譯失敗，點進失敗的 job，看紅色步驟的最後 30 行，常見原因是網頁編輯時誤動了語法。

> **未驗證**：我沒有實際執行過這個編譯流程，預估時間與產物路徑僅供參考。

### 10.4 Android 客戶端「沒有 Join」

最可能是因為你關閉了公開註冊（見第 6 節），登入頁不再顯示註冊入口，不只是 Android，所有客戶端都一樣。已有帳號直接登入即可；要讓別人加入，臨時把 `CHATTO_AUTH_DIRECT_REGISTRATION` 改回 `true`，或用 Operator CLI。

### 10.5 「Appearance／Theme customization」只在官方伺服器有

這個設定頁是**伺服器提供的網頁介面**，不是 APK 的功能。如果官方伺服器有、你的沒有，代表**你的伺服器版本較舊**（例如穩定版 0.4.25，而該面板屬於 0.5.0 測試版系列）。換 APK 解決不了，需升級伺服器（見第 11 節）。

---

## 11. 升級版本

官方文件沒有專門的升級章節，以下是 Compose 部署的標準流程。

### 11.1 流程

```bash
cd /opt/chatto/examples/dockercompose

# 1. 先備份（見第 7 節），並複製一份到 VPS 以外
docker compose exec -T -u chatto chatto /chatto backup --encrypt --include-keys \
  --passphrase-file /run/secrets/chatto-backup-passphrase \
  -o /backups/pre-upgrade-$(date -u +%Y%m%d-%H%M%S).tar.gz.age
tar czf backups/config-pre-upgrade.tar.gz .env livekit.generated.yaml compose.yml

# 2. 看目前版本與映像標籤
grep -n "image:" compose.yml
docker compose exec chatto /chatto --version

# 3. 升級
docker compose pull chatto
docker compose up -d chatto
docker compose logs --tail 30 chatto
docker compose exec chatto /chatto --version
```

`up -d` 只重建有變動的容器，NATS 與 LiveKit 保持運行，網站會有數秒中斷。如果 `image:` 是指定版本號（非 `latest`），要先改成新版本號。LiveKit 也要更新就 `docker compose pull && docker compose up -d`。

### 11.2 注意事項

- `latest` 標籤對應**穩定版**；0.5.0 目前仍是 beta，且 beta 之間可能有破壞性變更（例如 beta.6 有權限相關的 BREAKING CHANGES）。
- 升級可能升級資料格式，**降回舊版不一定能還原**，所以備份不能省。
- 升級前看 [GitHub Releases](https://github.com/chattocorp/chatto/releases) 的更新記錄；映像標籤請到 GitHub Packages 查，**不要猜**。
- 新版可能新增必要的環境變數，日誌報缺少設定時對照 [環境變數參考](https://docs.chatto.run/reference/environment-variables/)。
- **不要直接 `git pull` 範例倉庫**（你已改過 `compose.yml`，會衝突），只比對差異：
  ```bash
  cd /opt/chatto
  git fetch origin
  git diff HEAD origin/main -- examples/dockercompose/compose.yml
  ```

### 11.3 回復

把 `compose.yml` 的 `image:` 標籤改回舊版本，`docker compose pull chatto && docker compose up -d chatto`。若新版已升級資料格式，就用升級前的備份還原。

---

## 12. 常見問題與錯誤排查

### 12.1 網站與反向代理

| 症狀 / 錯誤訊息 | 原因 | 解法 |
|---|---|---|
| 域名 502，但 `127.0.0.1:4000` 正常 | NPM 在容器裡，`127.0.0.1` 指的是它自己 | 讓 NPM 加入 Chatto 網路，上游填 `chatto:4000`（見 4.2） |
| `sudo: nginx: command not found` | 主機上沒有 nginx，80/443 是 NPM 容器接管 | 在 NPM 後台設定，不需裝 nginx（見第 4 節） |
| 頁面只寫 `Bad gateway Error code 502` | 這是 **Cloudflare** 的錯誤頁，源站連不上或後端沒回應 | 依序查容器狀態、NPM 到上游、Cloudflare SSL 模式 |
| 上游填 `https://...:4000` 502 | 4000 是純 HTTP | Scheme 改 `http` |
| `network chatto_default not found` | 舊網路已被刪除，NPM compose 還在引用 | 把 NPM compose 裡的網路名稱改成 `dockercompose_default`，再 `docker compose up -d` |
| `endpoint ... already exists in network` | NPM 已接在該網路上 | 正常，直接測 curl |
| 容器重建後又 502 | `docker network connect` 是臨時的 | 把網路寫進 NPM 的 compose（見 4.4） |
| 上傳圖片／檔案失敗 | `client_max_body_size` 預設只有 1MB | NPM Advanced 加 `client_max_body_size 100m;` |
| 即時訊息不更新、頻繁斷線 | 沒開 WebSocket | Proxy Host 勾選 Websockets Support；加長 `proxy_read_timeout` |

**502 通用排查：**

```bash
docker compose ps
docker compose logs --tail 40 chatto
docker exec npm-app-1 curl -sI http://chatto:4000 | head -1       # 200 = 容器與網路正常
docker logs --tail 40 npm-app-1 2>&1 | grep -i -E "upstream|refused|resolve|502"
docker exec npm-app-1 nginx -t
docker exec npm-app-1 nginx -s reload     # 重建容器後 nginx 偶爾快取舊 IP
```

### 12.2 容器啟動失敗

| 錯誤訊息 | 原因 | 解法 |
|---|---|---|
| `failed to parse environment variables: ... DirectRegistration ... parsing "falsex"` | `.env` 的布林值多了字元 | 值只能是 `true`/`false`，用 `sed` 重寫（見 6.1） |
| `failed to parse compose.yml: mapping key "environment" already defined` | 刪 Caddy 時有殘留區塊被併入上一個服務 | 刪掉第二次出現的 `environment:`／`ports:`／`volumes:` 等 Caddy 殘留；`grep -n -i caddy compose.yml` |
| 80/443 被佔用、啟動失敗 | Caddy 沒移除，和 NPM 搶埠 | 移除 `caddy` 服務與 `caddy_*` volume |
| Chatto 連不上 NATS | `NATS_TOKEN` 與 `CHATTO_NATS_CLIENT_TOKEN` 不一致 | 兩者要相同，重新用 `init-env.sh` 生成最保險 |
| `docker compose` 指令找不到 `compose.yml` | 檔名可能是 `docker-compose.yml` 或目錄不對 | `docker inspect <容器> --format '{{index .Config.Labels "com.docker.compose.project.config_files"}}'` |

### 12.3 備份相關

| 錯誤訊息 | 原因 | 解法 |
|---|---|---|
| `read /run/secrets/chatto-backup-passphrase: is a directory` | 掛載來源不存在，Docker 建成了目錄 | 刪除該目錄、重新產生密碼檔、`--force-recreate`（見 7.1） |
| `mkdir /backups/.chatto-backup-...: permission denied` | `backups/` 擁有者是 root，容器用戶是 1000 | `chown -R 1000:1000 backups` |
| 備份檔 0 bytes 或沒產生 | 指令報錯 | 看輸出的錯誤訊息；CentOS 加 `:z` 掛載選項 |
| cron 備份沒跑 | 缺 `-T`、路徑錯誤 | 腳本內用 `exec -T`，`COMPOSE_DIR` 用絕對路徑，看 `/var/log/chatto-backup.log` |

### 12.4 帳號與管理

| 症狀 | 原因 | 解法 |
|---|---|---|
| `operator.sock: no such file or directory` | Operator API 沒啟用 | 啟用 operator API 後重建（見 6.2）；日常使用不一定需要 |
| 註冊提示郵件未設定 | 未設 SMTP | 設定 `CHATTO_SMTP_*`；465 埠加 `CHATTO_SMTP_TLS=implicit` |
| 收不到驗證碼 | SMTP 設定錯誤 | `docker compose logs chatto \| grep -i smtp` |
| 第一個帳號不是 owner | 信箱不在 `CHATTO_OWNERS_EMAILS` | 補上後 `docker compose restart chatto`（owner 在啟動時套用） |
| 找不到註冊／Join | 已關閉公開註冊 | 見 6.1、10.4 |

### 12.5 語音與通知

| 症狀 / 日誌 | 原因 | 解法 |
|---|---|---|
| `LiveKit listing failed ... tls: handshake failure`（每 30 秒一次） | LiveKit 域名 HTTPS 失敗；多為橘雲＋二層子域名無憑證，或 NPM 沒有該域名的 SSL | 改灰雲，改用一層子域名，NPM 重簽憑證（見 9.2） |
| `LiveKit listing failed ... 502 Bad Gateway` | HTTPS 通了，但 NPM 轉不到 `livekit:7880` | 檢查 Proxy Host（Scheme `http`、Host `livekit`、Port 7880）、NPM 是否在同一網路、`nginx -s reload` |
| `consecutive_failures` 數字很大 | 累計值，不會歸零 | 只看有沒有**新的**警告行，不看數字 |
| 能進頻道但沒聲音 | UDP 埠沒開、LiveKit 走橘雲 | 開放 7881/TCP、3478/UDP、50000-50200/UDP（主機＋安全組） |
| iPhone 收不到通知 | 沒從主畫面圖示開啟、伺服器沒設定 VAPID | 見第 8 節 |
| 設定裡沒有通知選項 | 伺服器沒啟用推播（介面被隱藏） | 設定四個 `CHATTO_PUSH_*` 變數並重建 |

**日誌裡的正常訊息（不需處理）：**

```
"level":"info","prefix":"core.kms","msg":"shredded call encryption key","key_ref":"call.e2ee...."
```

這是通話結束後銷毀該次通話的加密金鑰，是正常清理動作，也代表通話建立與結束的流程在運作。

只看警告與錯誤：

```bash
docker compose logs --since 10m chatto | grep -E '"level":"(warn|error)"'
```

### 12.6 客戶端

| 症狀 | 原因 | 解法 |
|---|---|---|
| APK 沒有 Appearance／主題面板 | 該面板屬於伺服器 0.5.0 系列，你的伺服器較舊 | 升級伺服器（第 11 節）；與 APK 無關 |
| 自編 APK 仍連官方伺服器 | 漏改 Kotlin 檔、或殘留舊設定 | 確認三個檔案都改了；清除 App 資料或重裝 |
| 裝不上自編 APK | 與官方 Release 簽名不同 | 先解除安裝舊版 |
| 官方 Release APK 連的是官方伺服器 | 預設如此 | 在 `chatto://` 設定裡改成你的網址 |
| Android 沒有 Join 選項 | 公開註冊已關閉 | 見 10.4 |

### 12.7 Cloudflare 驗證／Turnstile

我沒有查到 Chatto 內建 Cloudflare Turnstile（登入／註冊人機驗證）的支援，官方文件導覽中也沒有相關設定。替代做法：

- Cloudflare 的 Bot Fight Mode、WAF Rate Limiting（針對登入路徑限速）。
- **不要對整個站套用 Challenge**：Chatto 使用 WebSocket、API 與推播，App 和 API 客戶端遇到挑戰頁面會失敗。
- 公開註冊已關閉的話，機器人註冊的需求其實已經解決。
- 想把網站完全放在 Cloudflare Access 後面，同樣會擋住 App／API 客戶端，只適合只用網頁版的私人伺服器。
- 想讓源站只接受 Cloudflare 流量，可限制 80/443 的來源 IP 為 Cloudflare 段，但 **LiveKit 的媒體埠與灰雲域名不能套用**。

---

## 13. 指令速查

```bash
# 位置
cd /opt/chatto/examples/dockercompose

# 狀態與日誌
docker compose ps
docker compose logs --tail 50 chatto
docker compose logs --since 10m chatto | grep -E '"level":"(warn|error)"'

# 驗證 compose 與最終設定
docker compose config > /dev/null && echo OK
docker compose exec chatto env | grep CHATTO_

# 重啟／重建（改 .env 後用 --force-recreate）
docker compose restart chatto
docker compose up -d --force-recreate chatto

# 版本
docker compose exec chatto /chatto --version

# NPM
docker exec npm-app-1 nginx -t
docker exec npm-app-1 nginx -s reload
docker exec npm-app-1 curl -sI http://chatto:4000 | head -1
docker exec npm-app-1 curl -sI http://livekit:7880 | head -1

# 帳號
docker compose exec -u chatto chatto /chatto operator user list

# 手動備份
docker compose exec -T -u chatto chatto /chatto backup --encrypt --include-keys \
  --passphrase-file /run/secrets/chatto-backup-passphrase \
  -o /backups/chatto-$(date -u +%Y%m%d-%H%M%S).tar.gz.age
```

### 完整清除重裝（**會刪除所有聊天資料**）

```bash
cd /opt/chatto/examples/dockercompose
docker network disconnect dockercompose_default npm-app-1     # 先讓 NPM 離開網路
docker compose down -v --remove-orphans
cd /opt && rm -rf /opt/chatto
docker image rm ghcr.io/chattocorp/chatto:latest
rm -f /opt/chatto-backup.sh
crontab -e                                                      # 刪掉備份那一行
```

**不要用** `docker system prune -a --volumes`，可能清掉其他服務的映像與資料。重裝後記得更新 NPM compose 裡的網路名稱。

---

## 14. 未驗證事項清單

以下是我沒能在官方文件或實機上確認的內容，使用前請自行核對：

1. **管理後台入口與房間建立**在介面上的確切位置與按鈕名稱。
2. **`operator user create`** 的確切參數（官方指令列表沒有列出）。
3. **Operator API 啟用的環境變數名稱**（`CHATTO_OPERATOR_API_ENABLED` 是依命名規則推測）。
4. **還原（restore）指令**的完整參數，以及 `--conflict=overwrite` 的行為，是依官方說明推導，未實測。
5. Compose 部署執行 `chatto backup` 是否需要 `-c` 設定檔參數。
6. 舊版伺服器（如 0.4.25）是否支援 Web Push 及相同的變數名稱。
7. 自編譯 APK 的實際耗時與產物路徑。
8. **映像標籤**：官方是否提供具體版本標籤或 `main` 標籤，請到 GitHub Packages 查看。
9. 官方 Release 的桌面客戶端提供哪些平台。
10. Chatto 登入頁面的路徑（若要用 WAF 規則只針對登入頁）。

### 官方連結

- 文件首頁：<https://docs.chatto.run/>
- Docker Compose：<https://docs.chatto.run/guides/deployment/docker-compose/>
- 備份與還原：<https://docs.chatto.run/guides/operations/backup-restore/>
- Operator CLI：<https://docs.chatto.run/guides/operations/operator-cli/>
- 推播：<https://docs.chatto.run/guides/operations/notifications-web-push/>
- 環境變數：<https://docs.chatto.run/reference/environment-variables/>
- 專案倉庫：<https://github.com/chattocorp/chatto>
- 社群桌面／Android 客戶端：<https://github.com/teal-bauer/chatto-tauri>
```