# Next.js Static Export — Docker, GHCR & GitHub Actions CI/CD

A production-ready guide for building a Next.js app, containerizing it with Docker, and shipping it automatically using GitHub Actions.

## What You Will Build

A single-page Next.js app that is exported as pure static HTML/CSS/JS and served by a rootless Nginx container. No Node.js runtime is needed after the build. GitHub Actions handles testing, building, and pushing the Docker image to GitHub Container Registry automatically.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 16 (App Router, static export) |
| Styling | Tailwind CSS v4 |
| Language | TypeScript |
| Package manager | pnpm |
| Container runtime | Docker (multi-stage build) |
| Web server | Nginx (nginx-unprivileged, rootless) |
| Registry | GitHub Container Registry (GHCR) |
| CI/CD | GitHub Actions |
| Notifications | Slack Incoming Webhooks |

## Quick Start

### Development

```bash
# Install dependencies
pnpm install

# Start dev server
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000).

### Available Scripts

| Command | What it does |
|---------|------------|
| `pnpm dev` | Dev server with hot reload |
| `pnpm build` | Generates static export to `/out` |
| `pnpm lint` | Runs ESLint |
| `pnpm test:ci` | Lint + build (used by CI) |

## Local Docker Testing

Test the production build locally:

```bash
# Build and run the container
docker compose up --build

# Access the app
# Open http://localhost:8080

# Stop the container
docker compose down
```

The app is served by Nginx on port 8080, exactly as it will be in production.

## GitHub Actions CI/CD Pipeline

### How It Works

**On every Pull Request:**
- Runs `pnpm lint && pnpm build` to validate the code

**On push to `main`:**
- Runs tests (lint + build)
- Builds Docker image with Buildx
- Logs in to GHCR
- Pushes image with two tags:
  - `ghcr.io/<owner>/<repo>:latest`
  - `ghcr.io/<owner>/<repo>:sha-<commit>`
- Sends Slack notification with deployment status

### Setup Instructions

#### 1. Push to GitHub

```bash
git init
git add .
git commit -m "initial commit"
git remote add origin https://github.com/<your-username>/<your-repo>.git
git push -u origin main
```

#### 2. Configure Slack Notifications (Optional)

If you want deployment notifications:

1. **Create a Slack app:**
   - Go to https://api.slack.com/apps → Create New App → From scratch
   - Name it (e.g., `my-app-ci`)
   - Select your workspace → Create App

2. **Enable Incoming Webhooks:**
   - In your app settings, select **Incoming Webhooks** from the left sidebar
   - Toggle **Activate Incoming Webhooks** to On
   - Click **Add New Webhook to Workspace**, pick a channel, then click **Allow**
   - Copy the webhook URL (format: `https://hooks.slack.com/services/<YOUR_WEBHOOK_URL>`)

3. **Add the secret to GitHub:**
   - Repo → Settings → Secrets and variables → Actions
   - New repository secret
   - Name: `SLACK_WEBHOOK_URL`
   - Value: paste the webhook URL

⚠️ **Keep the webhook URL secret** — Slack actively revokes leaked secrets.

#### 3. No Other Secrets Needed

The workflow uses GitHub's built-in `GITHUB_TOKEN` to push to GHCR. No manual setup required.

## Docker Architecture

### Multi-Stage Build

The `Dockerfile` has three stages:

1. **Dependencies** — Install pnpm packages
2. **Builder** — Compile Next.js to static HTML/CSS/JS
3. **Runner** — Serve files with rootless Nginx

This approach minimizes the final image size by excluding Node.js, source code, and build artifacts.

### Nginx Configuration

The `nginx.conf` is tuned for static sites:
- Rootless mode on port 8080
- Try files fallback for SPA routing
- 1-year cache headers for `_next` assets
- Proper mimetype handling

## Project Structure

```
.
├── .github/
│   └── workflows/
│       └── deploy.yml           # GitHub Actions pipeline
├── app/
│   ├── globals.css              # Global Tailwind styles
│   ├── layout.tsx               # Root layout
│   └── page.tsx                 # Landing page
├── public/                       # Static assets
├── Dockerfile                    # Multi-stage Docker build
├── compose.yml                   # Docker Compose for local testing
├── nginx.conf                    # Nginx config (rootless, port 8080)
├── next.config.ts                # Static export config
├── package.json                  # Scripts and dependencies
├── pnpm-lock.yaml                # Locked dependency tree
└── tsconfig.json                 # TypeScript config
```

## Workflow Status & Logs

After pushing to GitHub:

1. Go to your repo → **Actions** tab
2. Click on the latest workflow run
3. See real-time logs for test & build steps
4. If Slack is configured, check your Slack channel for deployment messages

## Customization

### Update Node.js Version

Edit `Dockerfile`:

```dockerfile
ARG NODE_VERSION=24.13.0-slim
```

### Change Nginx Port

Edit `nginx.conf`:

```nginx
server {
    listen 8080;  # Change this
    ...
}
```

And update `compose.yml`:

```yaml
ports:
  - "8080:8080"  # Change host port if needed
```

### Modify Build Cache

Edit `.github/workflows/deploy.yml`:

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

## Troubleshooting

### Docker build fails locally

```bash
# Clean up Docker artifacts and rebuild
docker compose down -v
docker compose up --build
```

### GHCR push fails

- Verify `GITHUB_TOKEN` is available (it's automatic)
- Check repo permissions in GitHub settings
- Ensure you've pushed to `main` (not another branch)

### Slack notifications not sending

```bash
# Test the webhook URL manually
curl -X POST -H 'Content-type: application/json' \
  --data '{"text":"Test"}' \
  $SLACK_WEBHOOK_URL
```

### Static export not generating

Verify `next.config.ts`:

```typescript
export const output = "export";
export const images = { unoptimized: true };
```

## Next Steps

- Review each file to understand the setup
- Customize the landing page in `app/page.tsx`
- Add more routes in the `app/` directory
- Deploy to your preferred container orchestration platform (Kubernetes, AWS ECS, etc.)

## Learn More

- [Next.js Static Export Documentation](https://nextjs.org/docs/app/building-your-application/deploying/static-exports)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [GitHub Actions Workflows](https://docs.github.com/en/actions/using-workflows)
- [GHCR Documentation](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)

