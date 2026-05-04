# Day 35 — 部署紀錄：spots-list (地點找找看)

## 架構

```
用戶
  ↓ HTTPS
Cloudflare (DNS + DDoS 保護)
  ├── findingaspot.org → Cloudflare Pages (前端 React SPA)
  └── api.findingaspot.org → EC2 Elastic IP
                                ↓ HTTP port 80
                              Nginx (反向代理)
                                ↓
                              Docker container (127.0.0.1:3001)
                              NestJS
                                ↓
                              Supabase PostgreSQL
```

## 部署流程（monorepo：frontend + backend 在同個 repo）

### 1. EC2 準備

```bash
# 安裝 Docker
sudo dnf install -y docker
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker ec2-user
# exit 再重連讓 group 生效

# 安裝 Nginx
sudo dnf install -y nginx
sudo systemctl enable nginx
sudo systemctl start nginx
```

### 2. Nginx 設定（反向代理）

```bash
printf 'server {\n    listen 80;\n    server_name api.findingaspot.org;\n\n    location / {\n        proxy_pass http://127.0.0.1:3001;\n        proxy_set_header Host $host;\n        proxy_set_header X-Real-IP $remote_addr;\n        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;\n        proxy_set_header X-Forwarded-Proto $scheme;\n    }\n}\n' | sudo tee /etc/nginx/conf.d/api.conf

sudo nginx -t
sudo systemctl reload nginx
mkdir -p /home/ec2-user/spots
```

### 3. AWS 設定

- 配置 **Elastic IP** 並綁到 EC2（否則 IP 會因重啟而改變）
- **Security Group** Inbound rules：
  - SSH port 22: 0.0.0.0/0
  - HTTP port 80: 0.0.0.0/0

### 4. GitHub Actions CI/CD

`.github/workflows/deploy.yml` 觸發條件：push to master（backend/ 有改動）

流程：
1. Build Docker image（build context: `./backend`）
2. Push 到 GHCR（ghcr.io/username/repo:latest）
3. SSH 進 EC2 → 寫 .env → docker pull → 停舊 container → 啟新 container

GitHub Secrets 需要設定：
- `EC2_HOST` — Elastic IP
- `EC2_SSH_KEY` — PEM 檔內容
- `DATABASE_URL` — Supabase Session Pooler URL
- `FRONTEND_URL` — https://findingaspot.org
- `GHCR_TOKEN` — GitHub PAT (read:packages)

### 5. Cloudflare DNS

- `api.findingaspot.org` → A record → EC2 Elastic IP（Proxied 開啟）

### 6. Cloudflare SSL/TLS

設為 **Flexible**（EC2 只有 HTTP，Cloudflare 負責 HTTPS 終結）

### 7. Cloudflare Pages（前端）

- Build command: `cd frontend && npm run build`
- Output directory: `frontend/dist`
- 環境變數：`VITE_API_URL=https://api.findingaspot.org`

---

## 踩到的坑

### 坑 1：`Cannot find module '/app/dist/main'`

**原因：** Dockerfile CMD 是 `["node", "dist/main"]`，但 `nest build` 實際輸出到 `dist/src/main.js`（因為 tsconfig 沒有設 rootDir，TypeScript 保留了 src/ 目錄結構）

**修正：** `CMD ["node", "dist/src/main"]`

**教訓：** 部署前先在本機確認 `npm run build` 的輸出路徑：
```bash
find backend/dist -name "main.js"
```

---

### 坑 2：Cloudflare 522 錯誤（Connection Timed Out）

**原因：** Cloudflare SSL/TLS 預設是 **Full** 模式，會嘗試連 EC2 的 HTTPS port 443。但 EC2 只有 Nginx 監聽 port 80，port 443 沒開。

**修正：** Cloudflare SSL/TLS → Overview → 改成 **Flexible**

**教訓：**
- Flexible = Cloudflare 到 EC2 用 HTTP（EC2 不需要 SSL 憑證）
- Full = Cloudflare 到 EC2 用 HTTPS（EC2 需要 SSL 憑證，e.g., Certbot）
- 不裝 Certbot 就用 Flexible

---

### 坑 3：EC2 SSH 用舊 IP 連不進去

**原因：** 配置 Elastic IP 後，舊的 auto-assigned IP 失效，要用新的 Elastic IP 連線

**修正：** `ssh -i ~/.ssh/key.pem ec2-user@<ELASTIC_IP>`

---

### 坑 4：GitHub Actions 跑失敗（missing server host）

**原因：** GitHub Secrets 還沒設定就先 push 觸發 workflow，`EC2_HOST` secret 不存在

**修正：** 設好所有 Secrets 後在 Actions 頁面 **Re-run all jobs**

---

### 坑 5：EC2 Container 一直 Restarting（health check 誤判）

**原因：** `docker ps | grep -q spots-backend` 在 container crash 重啟的瞬間也會回傳 true，導致 health check 誤判為成功

**教訓：** 簡單的 process 存在檢查不等於 app 健康。可以加 `curl http://127.0.0.1:3001/health` 檢查 app 真正啟動。

---

## Nginx vs AWS ALB

| | Nginx（我們用的）| AWS ALB |
|--|--|--|
| 費用 | 免費（裝在 EC2）| $16-20/月起跳 |
| 適合 | side project、單台 EC2 | 多台 EC2、需要 auto scaling |
| 設定 | 手動設定 conf | 走 AWS 設定流程 |

Side project 用 Nginx 是正確選擇。

---

## Cloudflare Pages vs 自架

前端 React SPA 放 Cloudflare Pages 的優點：
- 免費
- 全球 CDN
- 自動 HTTPS
- Git 連接後 push 自動部署
- 不需要管 server

缺點：只能放靜態檔案，沒辦法跑 SSR（Next.js ISR/SSR 需要另外設定）。
