# PR Preview Infrastructure

Lightweight, self-hosted PR preview environments using Caddy, Docker, and a GitHub Actions self-hosted runner.

Every pull request gets an isolated ephemeral stack (Rails + Vite + Postgres) with automatic TLS and a unique subdomain.

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
│  │  Caddy (proxy)  │    │  GitHub Actions Runner Agent  │  │
│  │  + wildcard TLS │    │  (connects outbound to GH)    │  │
│  └────────┬────────┘    └───────────────┬───────────────┘  │
│           │ caddy network               │ checkout + build │
│  ┌────────┴─────────────────────────────┴───────────────┐  │
│  │  PR 42: rails-pr-42 + vite-pr-42 + postgres-pr-42   │  │
│  │  PR 87: rails-pr-87 + vite-pr-87 + postgres-pr-87   │  │
│  │  ...                                                │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## What This Does

- **On PR open/update**: Builds Docker images, spins up containers, runs migrations, and exposes the preview at `https://pr-<number>.pr.mydomain.com`.
- **On PR close**: Tears down containers, drops the per-PR database, and cleans up images.
- **Zero shared state**: Each PR gets its own Postgres container and volume. No database collisions.
- **Automatic TLS**: Caddy handles wildcard certificates via Cloudflare DNS challenge.

## What's in This Repo

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Permanent base infrastructure — Caddy reverse proxy only |
| `docker-compose.pr.yml` | Reference compose template for per-PR stacks (app repo should provide its own) |
| `Dockerfile.caddy` | Custom Caddy build with `caddy-docker-proxy` and Cloudflare DNS plugins |
| `.github/workflows/pr-open.yml` | Spin up a preview environment on the self-hosted runner |
| `.github/workflows/pr-close.yml` | Tear down a preview environment on the self-hosted runner |
| `SETUP.md` | Full setup guide: VPS bootstrap, runner installation, Cloudflare DNS, troubleshooting |

## Quick Start

1. **Bootstrap the VPS**: Follow [SETUP.md](SETUP.md#1-vps-bootstrap--base-infrastructure) to install Docker, create the Caddy network, and start the base infrastructure.
2. **Install the self-hosted runner**: Follow [SETUP.md](SETUP.md#3-install-self-hosted-github-actions-runner) to register the runner with your app repo.
3. **Set up DNS**: Add a `*.pr` A record in Cloudflare pointing to your VPS IP.
4. **Add the compose file to your app repo**: Copy the example from [SETUP.md](SETUP.md#docker-composepryml) into your application repository.

## Requirements

- Ubuntu VPS with Docker & Docker Compose
- Cloudflare-managed domain
- GitHub repository with a self-hosted runner registered
- Application repository must provide:
  - `docker-compose.pr.yml`
  - `Dockerfile.rails`
  - `Dockerfile.vite`

## Security

> **This setup uses a self-hosted runner and should only be used with private repositories.**
>
> Public repositories allow arbitrary code execution from forks, which is dangerous on self-hosted infrastructure. If your repo is public, consider a webhook-based deployment model instead.

## License

MIT
