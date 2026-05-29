# Deploy OpenClaw lên Coolify (Build trên GitHub Actions)

Hướng dẫn deploy OpenClaw lên Coolify server, build Docker image trên GitHub Actions để giảm tải cho server.

## Kiến trúc

```
GitHub Actions (build image) --> GHCR (lưu image) --> Coolify (pull & run)
```

- **GitHub Actions** build Docker image mỗi khi push code lên `main`
- Image được push lên **GitHub Container Registry (GHCR)**
- **Coolify** pull image từ GHCR và chạy trên server

## Bước 1: Cấu hình GitHub Repository

### 1.1. Cho phép GitHub Actions push packages

Vào repo Settings > Actions > General:
- Đảm bảo "Read and write permissions" được bật trong Workflow permissions

Vào repo Settings > Packages:
- Đảm bảo package visibility phù hợp (public hoặc private)

### 1.2. Trigger build lần đầu

Push code lên branch `main` hoặc vào tab **Actions** > chọn workflow **"Build & Push Docker Image"** > **Run workflow**.

Image sẽ được push lên: `ghcr.io/huuluong6768-jpg/openclaw:latest`

## Bước 2: Cấu hình Coolify

### 2.1. Tạo project mới trên Coolify

1. Đăng nhập Coolify dashboard
2. Tạo **New Project** > đặt tên (vd: `openclaw`)
3. Chọn environment (vd: `production`)

### 2.2. Thêm service Docker Compose

1. Trong project, chọn **+ Add New Resource**
2. Chọn **Docker Compose**
3. Paste nội dung file `docker-compose.coolify.yml` vào editor

### 2.3. Cấu hình environment variables

Trong Coolify, thêm các environment variables sau:

```env
# Token bảo mật cho gateway (bắt buộc)
OPENCLAW_GATEWAY_TOKEN=your-secret-token-here

# Timezone
OPENCLAW_TZ=Asia/Ho_Chi_Minh

# Port (mặc định 18789)
OPENCLAW_GATEWAY_PORT=18789
```

### 2.4. Cấu hình domain/proxy (tùy chọn)

Trong Coolify, bạn có thể cấu hình:
- **Domain**: Gán domain cho service (vd: `openclaw.yourdomain.com`)
- **SSL**: Coolify tự động cấp SSL qua Let's Encrypt
- **Proxy port**: Trỏ đến port `18789` của container

## Bước 3: Auto-deploy khi có image mới

### Cách 1: Webhook (khuyên dùng)

1. Trong Coolify, copy **Webhook URL** của resource
2. Trong GitHub repo, vào Settings > Webhooks > Add webhook
3. Paste Coolify webhook URL
4. Chọn event: `Packages` hoặc `Workflow runs`

### Cách 2: Polling

Trong Coolify resource settings, bật **Check for updates** với interval phù hợp (vd: 5 phút).

### Cách 3: Thêm step deploy vào GitHub Actions

Thêm step sau vào cuối workflow `.github/workflows/build-and-push.yml`:

```yaml
      - name: Trigger Coolify deploy
        if: github.ref == 'refs/heads/main'
        run: |
          curl -s "${{ secrets.COOLIFY_WEBHOOK_URL }}"
```

Sau đó thêm secret `COOLIFY_WEBHOOK_URL` trong repo Settings > Secrets > Actions.

## Bước 4: Kiểm tra deployment

### Health check

```bash
curl https://openclaw.yourdomain.com/healthz
```

### Xem logs trên Coolify

Trong Coolify dashboard > chọn resource > tab **Logs**.

## Cấu hình nâng cao

### Thêm extensions

Sửa `build-args` trong `.github/workflows/build-and-push.yml`:

```yaml
          build-args: |
            OPENCLAW_EXTENSIONS=telegram,discord,slack
```

### Thêm browser automation

```yaml
          build-args: |
            OPENCLAW_INSTALL_BROWSER=1
```

### Multi-architecture (amd64 + arm64)

Nếu server chạy ARM, thêm `linux/arm64` vào platforms:

```yaml
          platforms: linux/amd64,linux/arm64
```

> Lưu ý: Build multi-arch sẽ lâu hơn đáng kể.

## Troubleshooting

### Image pull failed

- Kiểm tra package visibility: repo Settings > Packages
- Nếu private, cần thêm GHCR credentials trong Coolify

### Container crash/restart loop

- Kiểm tra logs trong Coolify
- Đảm bảo `OPENCLAW_GATEWAY_TOKEN` đã được set
- Kiểm tra memory: OpenClaw cần tối thiểu 512MB RAM

### Build failed trên GitHub Actions

- Kiểm tra tab Actions trong repo
- Đảm bảo Dockerfile không bị modify sai
