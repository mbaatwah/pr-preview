# PR Preview Infrastructure

Lightweight, self-hosted PR preview environments using Traefik, Docker, and a GitHub Actions self-hosted runner.

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

## What This Does

- **On PR open/update**: Builds Docker images, spins up containers, runs migrations, and exposes the preview at `https://pr-<number>.pr.<your-domain>.com`.
- **On PR close**: Tears down containers, drops the per-PR database, and cleans up images.
- **Zero shared state**: Each PR gets its own Postgres container and volume. No database collisions.
- **Automatic TLS**: Traefik requests a certificate for each preview subdomain on demand via Let's Encrypt.

## What's in This Repo

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Permanent base infrastructure — Traefik reverse proxy only |
| `docker-compose.dashboard.yml` | Optional dashboard overlay with basic auth |
| `docker-compose.pr.yml` | Reference compose template for per-PR stacks (app repo should provide its own) |
| `.github/workflows/pr-open.yml` | Spin up a preview environment on the self-hosted runner |
| `.github/workflows/pr-close.yml` | Tear down a preview environment on the self-hosted runner |
| `SETUP.md` | Full setup guide: VPS bootstrap, runner installation, DNS, troubleshooting |

## Quick Start

1. **Bootstrap the VPS**: Follow [SETUP.md](SETUP.md#1-vps-bootstrap--base-infrastructure) to install Docker, create the Traefik network, and start the base infrastructure.
2. **Install the self-hosted runner**: Follow [SETUP.md](SETUP.md#3-install-self-hosted-github-actions-runner) to register the runner with your app repo.
3. **Set up DNS**: Add a `*.pr` A record pointing to your VPS IP.
4. **Add the compose file to your app repo**: Copy the example from [SETUP.md](SETUP.md#docker-composepryml) into your application repository.

## Requirements

- Ubuntu VPS with Docker & Docker Compose
- A domain with DNS A record support
- GitHub repository with a self-hosted runner registered
- Application repository must provide:
  - `docker-compose.pr.yml` (must define a `rails` service; see constraints in SETUP.md)
  - `Dockerfile.rails`
  - `Dockerfile.vite`

> **Note:** The fallback `docker-compose.pr.yml` in this repository does **not** include a database service. If your application requires Postgres (or any database), you must provide your own compose file.

## Security

> **This setup uses a self-hosted runner and should only be used with private repositories.**
>
> Public repositories allow arbitrary code execution from forks, which is dangerous on self-hosted infrastructure. If your repo is public, consider a webhook-based deployment model instead.

### Network Isolation

All PR stacks share the same Docker network (`traefik`). Containers from different PRs can reach each other by name. Do not place sensitive data or services on this shared network without additional isolation measures.

### Database Credentials

The example `docker-compose.pr.yml` uses `${POSTGRES_PASSWORD}` injected at runtime. Never hardcode passwords in your compose file or commit them to version control. Set credentials via GitHub Secrets or environment variables.

## Known Limitations

- **No concurrency control**: Multiple rapid pushes to the same PR can cause overlapping workflow runs that interfere with each other. See SETUP.md for a `concurrency` fix.
- **Fragile teardown**: If a branch is deleted immediately after merge, `pr-close.yml` may fail to check out the code needed for teardown.
- **No rollback**: The previous environment is destroyed before the new one is verified. A failed build or migration leaves the PR without a working preview.

## License

MIT
