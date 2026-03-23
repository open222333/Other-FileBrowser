# Other-FileBrowser

```
FileBrowser 檔案管理器 + Nginx 反向代理 Docker 環境
```

## 目錄

- [Other-FileBrowser](#other-filebrowser)
  - [目錄](#目錄)
  - [專案說明](#專案說明)
  - [架構概覽](#架構概覽)
  - [目錄結構](#目錄結構)
  - [設定檔說明](#設定檔說明)
  - [IP 白名單設定](#ip-白名單設定)
  - [執行流程](#執行流程)
  - [用法](#用法)
  - [建議注意事項](#建議注意事項)

## 專案說明

本專案以 Docker Compose 建立 FileBrowser 檔案管理服務，搭配 Nginx 反向代理作為對外入口，並透過 IP 白名單限制僅允許指定 IP 存取，提高安全性。

## 架構概覽

```
Client (允許的 IP)
  └─ Nginx (80 / 443)  ← 反向代理 + IP 白名單過濾
       └─ FileBrowser (內部 :80)  ← 檔案瀏覽與管理
            └─ ./filebrowser/  ← 實際提供瀏覽的目錄
```

| 服務 | Image | 說明 |
|------|-------|------|
| filebrowser | filebrowser/filebrowser:latest | 檔案瀏覽、上傳、下載、管理 |
| nginx | nginx | 反向代理，負責對外連接埠與 IP 過濾 |

## 目錄結構

```
Other-FileBrowser/
├── docker-compose.yml
├── filebrowser/          # 提供瀏覽的根目錄（掛載至容器 /srv）
├── database/             # FileBrowser SQLite 資料庫
├── settings/             # FileBrowser 設定檔目錄（掛載至容器 /config）
└── conf/
    ├── nginx/
    │   ├── nginx.conf           # Nginx 主設定
    │   └── conf.d/
    │       └── allow_ip.conf    # IP 白名單設定
    └── config.ini.default       # 應用設定範本
```

## 設定檔說明

啟動前需複製設定範本：

```bash
cp conf/config.ini.default conf/config.ini
```

Nginx 主設定（`conf/nginx/nginx.conf`）引用 IP 白名單，並將請求代理至 FileBrowser 容器：

```nginx
http {
  server {
    listen 80;
    include /etc/nginx/conf.d/allow_ip.conf;
    location / {
      proxy_pass http://filebrowser:80;
    }
  }
}
```

## IP 白名單設定

編輯 `conf/nginx/conf.d/allow_ip.conf`，加入允許存取的 IP 或 IP 段：

```nginx
# 允許內網段
allow 192.168.1.0/24;

# 允許單一 IP
# allow 1.2.3.4;

# 拒絕其他所有來源
deny all;
```

修改後重新載入 Nginx 設定：

```bash
docker compose exec nginx nginx -s reload
```

## 執行流程

1. 複製設定範本：`cp conf/config.ini.default conf/config.ini`
2. 設定 IP 白名單：編輯 `conf/nginx/conf.d/allow_ip.conf`
3. 啟動服務：`docker compose up -d`
4. 瀏覽器開啟 `http://localhost`，以預設帳密登入後立即修改密碼
5. 將要管理的檔案放入 `./filebrowser/` 目錄即可透過 Web UI 存取

## 用法

```bash
# 啟動所有服務（背景執行）
docker compose up -d

# 查看服務狀態
docker compose ps

# 查看日誌
docker compose logs -f

# 停止服務
docker compose down
```

存取網址：`http://localhost`

預設帳號：`admin`
預設密碼：`admin`（**首次登入後請立即至設定頁面修改密碼**）

## 建議注意事項

- **立即修改預設密碼**：FileBrowser 預設帳密為 `admin / admin`，首次登入後務必修改。
- **IP 白名單維護**：`allow_ip.conf` 僅允許指定來源，若需從新 IP 存取，記得更新此檔案並 reload Nginx。
- **HTTPS 設定**：目前範本僅含 HTTP（port 80）設定，對外服務建議配置 SSL 憑證並啟用 HTTPS（443）。
- **資料備份**：`./filebrowser/` 與 `./database/` 目錄為資料核心，請定期備份。
- **`./filebrowser/` 目錄內容**：此目錄會直接暴露於 Web UI，請避免放入敏感檔案或機密資料。
