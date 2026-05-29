# Deploy OpenClaw to Coolify (Build on GitHub Actions)

Guide to deploy OpenClaw on a Coolify server, building Docker images on GitHub Actions to keep the server lightweight.

## Architecture

```
GitHub Actions (build image) --> GHCR (store image) --> Coolify (pull & run)
```

- **GitHub Actions** builds a Docker image on every push to `main`
- The image is pushed to **GitHub Container Registry (GHCR)**
- **Coolify** pulls the pre-built image from GHCR and runs it on your server

## Step 1: Configure the GitHub Repository

### 1.1. Allow GitHub Actions to push packages

Go to repo Settings > Actions > General:
- Enable "Read and write permissions" under Workflow permissions

Go to repo Settings > Packages:
- Set package visibility as needed (public or private)

### 1.2. Trigger the first build

Push code to the `main` branch, or go to the **Actions** tab > select the **"Build & Push Docker Image"** workflow > **Run workflow**.

The image will be pushed to: `ghcr.io/<your-github-username>/openclaw:latest`

## Step 2: Configure Coolify

### 2.1. Create a new project on Coolify

1. Log in to the Coolify dashboard
2. Create a **New Project** and name it (e.g., `openclaw`)
3. Select an environment (e.g., `production`)

### 2.2. Add a Docker Compose service

1. In the project, click **+ Add New Resource**
2. Select **Docker Compose**
3. Paste the contents of `docker-compose.coolify.yml` into the editor

### 2.3. Configure environment variables

In Coolify, add the following environment variables:

```env
# Image to pull (required — set to your fork's GHCR image)
OPENCLAW_IMAGE=ghcr.io/<your-github-username>/openclaw:latest

# Security token for the gateway (required)
OPENCLAW_GATEWAY_TOKEN=your-secret-token-here

# Timezone (default: UTC)
OPENCLAW_TZ=UTC

# Port (default: 18789)
OPENCLAW_GATEWAY_PORT=18789
```

### 2.4. Configure domain/proxy (optional)

In Coolify, you can configure:
- **Domain**: Assign a domain to the service (e.g., `openclaw.yourdomain.com`)
- **SSL**: Coolify automatically provisions SSL via Let's Encrypt
- **Proxy port**: Point to port `18789` of the container

## Step 3: Auto-deploy on new image push

### Option 1: Webhook (recommended)

1. In Coolify, copy the **Webhook URL** of the resource
2. In the GitHub repo, go to Settings > Webhooks > Add webhook
3. Paste the Coolify webhook URL
4. Select event: `Packages` or `Workflow runs`

### Option 2: Polling

In Coolify resource settings, enable **Check for updates** with an appropriate interval (e.g., 5 minutes).

### Option 3: Add a deploy step to GitHub Actions

Add the following step to the end of the `.github/workflows/build-and-push.yml` workflow:

```yaml
      - name: Trigger Coolify deploy
        if: github.ref == 'refs/heads/main'
        run: |
          curl -s "${{ secrets.COOLIFY_WEBHOOK_URL }}"
```

Then add the `COOLIFY_WEBHOOK_URL` secret in repo Settings > Secrets and variables > Actions.

## Step 4: Verify the deployment

### Health check

```bash
curl https://openclaw.yourdomain.com/healthz
```

### View logs on Coolify

In the Coolify dashboard, select the resource and go to the **Logs** tab.

## Advanced configuration

### Add extensions

Edit `build-args` in `.github/workflows/build-and-push.yml`:

```yaml
          build-args: |
            OPENCLAW_EXTENSIONS=telegram,discord,slack
```

### Add browser automation

```yaml
          build-args: |
            OPENCLAW_INSTALL_BROWSER=1
```

### Multi-architecture (amd64 + arm64)

If your server runs ARM, add `linux/arm64` to platforms:

```yaml
          platforms: linux/amd64,linux/arm64
```

> Note: Multi-arch builds take significantly longer.

## Troubleshooting

### Image pull failed

- Check package visibility: repo Settings > Packages
- If private, add GHCR credentials in Coolify

### Container crash/restart loop

- Check logs in Coolify
- Ensure `OPENCLAW_GATEWAY_TOKEN` is set
- Check memory: OpenClaw requires at least 512 MB RAM

### Build failed on GitHub Actions

- Check the Actions tab in the repo
- Ensure the Dockerfile has not been incorrectly modified
