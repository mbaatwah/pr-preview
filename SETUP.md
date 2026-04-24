# PR Preview Infrastructure — Setup Guide

> **⚠️ Security Warning**
>
> Self-hosted runners should **only be used with private repositories**.
> GitHub explicitly warns against using them with public repositories because
> any fork can open a PR and execute arbitrary code on your VPS.
> If your repo is public, use GitHub-hosted runners with a webhook-based
> deployment model instead.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      GitHub (Cloud)                         │
│  PR opened / updated / closed ──► GitHub Actions queue      │
└────────────────────────┬────────────────────────────────────┘
                         │ job dispatch
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                     Ubuntu VPS                              │
│                                                             │
│  ┌─────────────────┐    ┌───────────────────────────────┐  │
│  │ Traefik (proxy) │    │  GitHub Actions Runner Agent  │
│  │ + auto TLS      │    │  (connects outbound to GH)    │
│  └────────┬────────┘    └───────────────┬───────────────┘  │
│           │ traefik network             │ checkout + build │
│  ┌────────┴─────────────────────────────┴───────────────┐  │
│  │  PR 42: rails-pr-42 + vite-pr-42 + postgres-pr-42   │  │
│  │  PR 87: rails-pr-87 + vite-pr-87 + postgres-pr-87   │  │
│  │  ...                                                │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

- **Traefik**: On-demand certificates for each preview subdomain via Cloudflare DNS challenge
- **Self-hosted runner**: A GitHub Actions agent running on the VPS itself. Jobs
  are dispatched from GitHub, but Docker commands run natively on the machine.
- **PR stacks**: Each PR is self-contained. The application repository defines its own
  services (Rails, Vite, Postgres, Redis, etc.) in `docker-compose.pr.yml`.
- **Isolation**: Every PR gets its own containers and volumes. No shared database.

---

## 1. VPS Bootstrap — Base Infrastructure

Run these commands on a fresh Ubuntu VPS:

```bash
# Install Docker + Compose
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Log out and back in for group to take effect, then:
docker --version
docker compose version

# Create the ingress network (required by Traefik Docker provider)
docker network create traefik --ipv6

# Create the project directory
sudo mkdir -p /opt/pr-preview
sudo chown $USER:$USER /opt/pr-preview

# Clone this (infra) repo
cd /opt/pr-preview
git clone <your-infra-repo-url> .

# Create the .env file from the template
cp .env.example .env
nano .env   # Fill in CLOUDFLARE_API_TOKEN and TRAEFIK_ACME_EMAIL

# Start the base infrastructure
docker compose up -d

# Verify
docker compose ps          # Should show traefik running
docker logs traefik        # Should show "Starting provider *docker.Provider"
```

### Optional: Enable the Traefik Dashboard

If you want the Traefik dashboard exposed with basic auth:

```bash
# Generate a basic auth hash (replace 'yourpassword')
htpasswd -nb admin yourpassword | sed -e s/\\$/\\$\\$/g

# Paste the result into .env as TRAEFIK_DASHBOARD_AUTH
# Then start with the dashboard overlay:
docker compose -f docker-compose.yml -f docker-compose.dashboard.yml up -d
```

The dashboard will be available at `https://traefik.pr.<your-domain>.com` (or whatever you set in `TRAEFIK_DASHBOARD_DOMAIN`).

---

## 2. Cloudflare DNS Setup

In your Cloudflare dashboard for `mydomain.com`:

1. **DNS Record**: Add an A record:
   - Name: `*.pr`
   - IPv4: `<your VPS IP>`
   - Proxy: **OFF** (DNS-only, grey cloud) — required for ACME DNS challenge

2. **API Token**: Create a scoped API token at https://dash.cloudflare.com/profile/api-tokens
   - Permissions:
     - **Zone → Zone → Read**
     - **Zone → DNS → Edit**
   - Zone Resources: Include → Specific zone → `mydomain.com`
   - Copy the token value — you'll need it for the `.env` file

---

## 3. Install Self-Hosted GitHub Actions Runner

The runner is a long-lived agent on the VPS that connects to GitHub and waits for
jobs. When a PR event occurs, GitHub dispatches the workflow to this runner,
which executes it directly on the VPS.

### Create a dedicated user (recommended)

```bash
sudo useradd -m -s /bin/bash github-runner
sudo usermod -aG docker github-runner
sudo su - github-runner
```

### Download and configure the runner

```bash
mkdir ~/actions-runner && cd ~/actions-runner

# Download the latest runner (check https://github.com/actions/runner/releases)
RUNNER_VERSION="2.317.0"
curl -o "actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz" \
  -L "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"

tar xzf "actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"

# Configure (get the token from your app repo:
# Settings → Actions → Runners → New self-hosted runner)
./config.sh --url https://github.com/<OWNER>/<REPO> --token <RUNNER_TOKEN>

# Install and start as a systemd service
sudo ./svc.sh install
sudo ./svc.sh start
```

### Verify

```bash
sudo systemctl status actions.runner.<OWNER>-<REPO>.<HOSTNAME>.service
```

You should also see the runner listed as "Online" in your repo's
Settings → Actions → Runners page.

---

## 4. App Repository Requirements

Your app repository needs these files at its root:

### Domain & Environment Variables

Before copying the examples below, replace all occurrences of `<your-domain>.com` with your actual domain. The following environment variables should be configured in your repository settings (GitHub Secrets or Variables) or in a `.env` file:

| Variable | Purpose | Example |
|----------|---------|---------|
| `POSTGRES_PASSWORD` | Database password for the per-PR Postgres container | (generate a strong password) |

> **Note:** The `POSTGRES_PASSWORD` example uses `${POSTGRES_PASSWORD}` in the compose file. Set this via GitHub Secrets (`secrets.POSTGRES_PASSWORD`) and pass it into the workflow environment, or export it on the runner before the workflow runs.

### `docker-compose.pr.yml`

This file defines the full per-PR stack. The workflow injects `PR_NUMBER` and `IMAGE_TAG`.

> **⚠️ Important constraints:**
> - The Rails service **must be named `rails`** (the workflow hardcodes `docker exec rails-pr-${PR_NUMBER}`).
> - If you change `container_name` for the Rails service, the database preparation step will fail.
> - The fallback `docker-compose.pr.yml` in this repo **does not include a database service**. If your app needs Postgres, you must provide your own compose file (recommended).

```yaml
services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres-pr-${PR_NUMBER}
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - traefik

  rails:
    image: rails-app:${IMAGE_TAG}
    container_name: rails-pr-${PR_NUMBER}
    environment:
      DATABASE_URL: postgres://postgres:${POSTGRES_PASSWORD}@postgres-pr-${PR_NUMBER}:5432/app
      RAILS_ENV: production
      RAILS_LOG_TO_STDOUT: "true"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.rails-pr-${PR_NUMBER}.rule=Host(`api-pr-${PR_NUMBER}.pr.<your-domain>.com`)"
      - "traefik.http.routers.rails-pr-${PR_NUMBER}.entrypoints=websecure"
      - "traefik.http.routers.rails-pr-${PR_NUMBER}.tls.certresolver=letsencrypt"
      - "traefik.http.services.rails-pr-${PR_NUMBER}.loadbalancer.server.port=3000"

  vite:
    image: vite-app:${IMAGE_TAG}
    container_name: vite-pr-${PR_NUMBER}
    environment:
      VITE_API_URL: https://api-pr-${PR_NUMBER}.pr.<your-domain>.com
    networks:
      - traefik
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.vite-pr-${PR_NUMBER}.rule=Host(`pr-${PR_NUMBER}.pr.<your-domain>.com`)"
      - "traefik.http.routers.vite-pr-${PR_NUMBER}.entrypoints=websecure"
      - "traefik.http.routers.vite-pr-${PR_NUMBER}.tls.certresolver=letsencrypt"
      - "traefik.http.services.vite-pr-${PR_NUMBER}.loadbalancer.server.port=80"

networks:
  traefik:
    external: true

volumes:
  postgres_data: {}
```

### `Dockerfile.rails`

```dockerfile
FROM ruby:3.3-slim

WORKDIR /rails

RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

EXPOSE 3000

CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0", "-p", "3000"]
```

### `Dockerfile.vite`

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### `nginx.conf` (for Vite container)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

---

## 5. How It Works — End to End

### PR Opened / Updated

1. GitHub Actions triggers `pr-open.yml`
2. The job is dispatched to the self-hosted runner on the VPS
3. The runner checks out the PR code directly on the VPS
4. Builds `rails-app` and `vite-app` Docker images tagged with PR number + SHA
5. Starts the PR stack using the app repo's `docker-compose.pr.yml`:
   - `pr-42.pr.mydomain.com` → Vite container (frontend)
   - `api-pr-42.pr.mydomain.com` → Rails container (API)
6. Runs `bin/rails db:prepare` inside the Rails container
7. Traefik discovers the new containers via Docker labels and updates its routing table

### PR Closed

1. GitHub Actions triggers `pr-close.yml`
2. The runner checks out the PR code and locates the compose file
3. Runs `docker compose down -v` to remove containers and volumes (including the per-PR database)
4. Removes all built images tagged for this PR

---

## 6. Security Considerations

### Network Isolation

All PR stacks attach to the same external `traefik` network. Containers from different PRs can reach each other by container name (e.g., `postgres-pr-87` is reachable from `rails-pr-42`). If you run PRs from untrusted contributors or with sensitive data, consider additional isolation:

- Use per-PR Docker networks bridged to Traefik.
- Randomize container names or use non-predictable hostnames.
- Avoid placing sensitive services on the shared `traefik` network.

### Self-Hosted Runner Security

As noted at the top of this guide, self-hosted runners should **only be used with private repositories**. Public forks can open PRs and execute arbitrary code on your VPS. If your repository is public, use a webhook-based deployment model with GitHub-hosted runners instead.

### Database Passwords

Never commit database passwords to your repository. Use GitHub Secrets or environment variables injected at runtime. The examples in this guide use `${POSTGRES_PASSWORD}` as a placeholder.

---

## 7. Known Limitations

### No Concurrency Control (Race Condition)

The example workflows do not declare a `concurrency` group. If a PR receives multiple pushes in quick succession, multiple workflow runs can execute concurrently on the self-hosted runner. Because every run for the same PR uses the identical `COMPOSE_PROJECT` name, they can interfere with each other (e.g., one run tearing down containers that another just started).

**Fix:** Add a concurrency block to `pr-open.yml`:
```yaml
concurrency:
  group: pr-preview-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

### Fragile Teardown on Branch Deletion

`pr-close.yml` checks out the PR branch using `head.sha`. If the branch is deleted immediately after merge (common with "delete branch on merge" enabled), the checkout can fail. Additionally, if `docker-compose.pr.yml` changed between the last deployment and the close event, `docker compose down` may not match the running containers.

**Mitigation:** Ensure "delete branch on merge" is disabled, or implement a stateless teardown that targets containers by project label without checking out code.

### No Rollback on Failed Updates

`pr-open.yml` tears down the previous environment *before* building images and starting the new stack. If the new build fails or a migration errors, the PR is left with no working preview environment.

**Mitigation:** Test builds locally before pushing, or implement a blue-green deployment approach (build and verify the new stack before swapping).

---

## 8. Maintenance

### Update the runner

GitHub releases runner updates periodically. The runner will log warnings when an
update is available.

```bash
sudo su - github-runner
cd ~/actions-runner
./svc.sh stop

# Download the new version (replace X.Y.Z)
RUNNER_VERSION="X.Y.Z"
curl -o "actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz" \
  -L "https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"

# Extract over the existing installation
tar xzf "actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz"

./svc.sh start
```

### Monitor disk usage

Docker images, build cache, and runner workspaces accumulate over time:

```bash
# Check Docker disk usage
docker system df -v

# Prune dangling images (safe)
docker image prune -f

# Prune build cache and unused builder layers (run periodically)
docker builder prune -f

# Check runner workspace size
du -sh ~/actions-runner/_work
```

> **Note:** The `pr-close.yml` cleanup step removes tagged images, but it does not remove untagged intermediate layers produced by multi-stage builds. Running `docker builder prune -f` periodically prevents gradual disk accumulation.

---

## 9. Troubleshooting

### Check runner status

```bash
sudo systemctl status actions.runner.<OWNER>-<REPO>.<HOSTNAME>.service
sudo journalctl -u actions.runner.<OWNER>-<REPO>.<HOSTNAME>.service -f
```

### Check Traefik routing table

If the dashboard is enabled, visit `https://traefik.pr.<your-domain>.com`.

From the command line (requires a temporary curl container since the Traefik image has no shell):

```bash
docker run --rm --network container:traefik curlimages/curl -s \
  http://localhost:8080/api/rawdata | jq .
```

### Check if a PR container is reachable

```bash
# From the host, using the shared network
docker run --rm --network traefik curlimages/curl -s \
  http://rails-pr-42:3000/health
```

### View Traefik logs

```bash
docker logs traefik --tail 50
```

### View Rails container logs

```bash
docker logs rails-pr-42 --tail 50
```

### Manually clean up a stuck PR

```bash
PR=42

# Tear down by project name (does not require the compose file):
docker compose -p pr-${PR} down -v

# Or, if you have the app repo cloned locally:
cd /path/to/your-app-repo
docker compose -f docker-compose.pr.yml -p pr-${PR} down -v

# Clean up images
docker images --format '{{.Repository}}:{{.Tag}}' | grep -E ":pr-${PR}-[a-f0-9]{40}$" | xargs -r docker rmi

# Prune any leftover build cache
docker builder prune -f
```

### Restart Traefik

```bash
docker restart traefik
```

### Runner is offline in GitHub UI

1. Check the service is running: `sudo systemctl status actions.runner...`
2. Check network connectivity from the VPS: `curl -I https://github.com`
3. If the registration token expired, re-run `./config.sh` with a new token
4. Check logs: `sudo journalctl -u actions.runner...`
