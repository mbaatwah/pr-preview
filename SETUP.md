# PR Preview Infrastructure — Setup Guide

## Architecture Overview

```
                    ┌─────────────────────────────────────────┐
                    │            Ubuntu VPS                   │
                    │                                         │
  *.pr.mydomain.com│  ┌─────────┐    ┌──────────────────┐   │
  ──────────────────┼─►│  Caddy  │    │  Shared Postgres │   │
  (wildcard DNS)    │  │(proxy + │    │     16-alpine    │   │
                    │  │wildcard │    └────────┬─────────┘   │
                    │  │  cert)  │             │              │
                    │  └────┬────┘             │              │
                    │       │ caddy network    │              │
                    │  ┌────┴─────────────────┴──────────┐   │
                    │  │  PR 42:  rails-pr-42 + vite-pr-42│   │
                    │  │  PR 87:  rails-pr-87 + vite-pr-87│   │
                    │  │  ...                             │   │
                    │  └──────────────────────────────────┘   │
                    └─────────────────────────────────────────┘
```

- **Caddy**: Single wildcard cert (`*.pr.mydomain.com`) via Cloudflare DNS challenge
- **Postgres**: One shared instance; each PR gets a logical database (`pr_42`, `pr_87`, etc.)
- **PR containers**: Rails + Vite per PR, joined to the `caddy` network for label-based routing

---

## 1. VPS Bootstrap

Run these commands on a fresh Ubuntu VPS:

```bash
# Install Docker + Compose
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER

# Log out and back in for group to take effect, then:
docker --version
docker compose version

# Create the ingress network (required by caddy-docker-proxy)
docker network create caddy --ipv6

# Create the project directory
sudo mkdir -p /opt/pr-preview
sudo chown $USER:$USER /opt/pr-preview

# Clone this repo
cd /opt/pr-preview
git clone <your-infra-repo-url> .

# Create the .env file from the template
cp .env.example .env
nano .env   # Fill in CLOUDFLARE_API_TOKEN and POSTGRES_PASSWORD

# Start the base infrastructure
docker compose up -d

# Verify
docker compose ps          # Should show caddy + postgres running
docker logs caddy          # Should show "caddy-docker-proxy" started
docker exec postgres pg_isready -U postgres  # Should say "accepting connections"
```

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
   - Copy the token value — you'll need it for `.env` and GitHub Secrets

---

## 3. GitHub Secrets Configuration

In your **app repository** (not the infra repo), go to Settings → Secrets and variables → Actions:

| Secret Name | Value | Used By |
|---|---|---|
| `VPS_HOST` | Your VPS IP or hostname | Both workflows |
| `VPS_USER` | SSH username (e.g. `ubuntu`) | Both workflows |
| `VPS_SSH_KEY` | Full private SSH key content | Both workflows |
| `POSTGRES_PASSWORD` | Same password as in VPS `.env` | pr-open.yml (constructs DATABASE_URL) |

### Setting up SSH access

```bash
# On your local machine, generate a deploy key (if you don't have one)
ssh-keygen -t ed25519 -f ~/.ssh/vps-deploy -C "github-actions-deploy"

# Copy the public key to the VPS
ssh-copy-id -i ~/.ssh/vps-deploy.pub ubuntu@<VPS_IP>

# Add the private key as VPS_SSH_KEY GitHub Secret
cat ~/.ssh/vps-deploy | pbcopy   # macOS
cat ~/.ssh/vps-deploy | xclip -selection clipboard  # Linux
# Paste into GitHub Secret value
```

---

## 4. App Repository Requirements

Your app repository needs these Dockerfiles at its root:

### Dockerfile.rails

```dockerfile
FROM ruby:3.3-slim

WORKDIR /rails

# Install dependencies
RUN apt-get update -qq && \
    apt-get install -y build-essential libpq-dev && \
    rm -rf /var/lib/apt/lists/*

COPY Gemfile Gemfile.lock ./
RUN bundle install

COPY . .

EXPOSE 3000

CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0", "-p", "3000"]
```

### Dockerfile.vite

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

### nginx.conf (for Vite container)

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
2. SSHs into VPS, clones/updates the PR branch
3. Builds `rails-app` and `vite-app` Docker images tagged with PR number + SHA
4. Creates logical database `pr_N` on shared Postgres
5. Starts Rails + Vite containers with Caddy routing labels:
   - `pr-42.pr.mydomain.com` → Vite container (frontend)
   - `api-pr-42.pr.mydomain.com` → Rails container (API)
6. Runs `bin/rails db:prepare` inside the Rails container
7. Caddy auto-discovers the new containers via Docker labels (5s polling)

### PR Closed

1. GitHub Actions triggers `pr-close.yml`
2. SSHs into VPS
3. Drops the logical database `pr_N` from shared Postgres
4. Runs `docker compose down -v` to remove containers and anonymous volumes
5. Removes the built images

---

## 6. Troubleshooting

### Check Caddy routing table

```bash
docker exec caddy cat /config/caddy/Caddyfile.autosave
```

### Check if a PR container is reachable

```bash
docker exec caddy wget -qO- http://rails-pr-42:3000/health 2>/dev/null
```

### View Caddy logs

```bash
docker logs caddy --tail 50
```

### View Rails container logs

```bash
docker logs rails-pr-42 --tail 50
```

### List all PR databases

```bash
docker exec postgres psql -U postgres -c "\l" | grep pr_
```

### Manually clean up a stuck PR

```bash
PR=42
docker exec postgres psql -U postgres -c "DROP DATABASE IF EXISTS pr_${PR};"
cd /opt/pr-preview
docker compose -f docker-compose.pr.yml -p pr-${PR} down -v
```

### Force Caddy to reload

```bash
docker kill --signal=SIGHUP caddy
```
