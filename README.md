# HomeLab

Docker Compose stacks for a personal homelab.

This repo is organized as one directory per service. Each stack keeps its own
`compose.yml`, optional `.env.example`, and any service-specific notes or helper
files. The goal is to keep each service self-contained while following a common
pattern across the repo.

## Repository Layout

- [SearXNG](/Users/jibanez/projects/HomeLab/SearXNG): private metasearch stack
- [forgejo](/Users/jibanez/projects/HomeLab/forgejo): self-hosted Git forge
- [immich](/Users/jibanez/projects/HomeLab/immich): photo library stack and migration runbook
- [jellyfin](/Users/jibanez/projects/HomeLab/jellyfin): media server
- [nodejs_dev](/Users/jibanez/projects/HomeLab/nodejs_dev): Node.js development container with PostgreSQL
- [pihole](/Users/jibanez/projects/HomeLab/pihole): Pi-hole with Cloudflare upstream DNS
- [stirling_pdf](/Users/jibanez/projects/HomeLab/stirling_pdf): PDF tools stack
- [wordpress](/Users/jibanez/projects/HomeLab/wordpress): WordPress with MariaDB and Redis

## Shared Conventions

- Each service lives in its own directory.
- Compose files are named `compose.yml`.
- Example environment files are named `.env.example` when present.
- Persistent data paths are usually configurable through environment variables.
- Services that should only be reachable on the tailnet use a dedicated
  Tailscale sidecar container.
- Healthchecks are added where the image provides a meaningful local probe.
- Generated state and local data directories are ignored in `.gitignore`.

## Typical Workflow

1. Go into the service directory you want to work on.
2. Review `compose.yml` and `.env.example`.
3. Create a real `.env` with the values for that host.
4. Render the compose config before starting anything:

```sh
docker compose --env-file .env -f compose.yml config
```

5. Start the stack:

```sh
docker compose --env-file .env -f compose.yml up -d
```

## Secret Safety

This repo includes both CI and local secret scanning:

- GitHub Actions runs Gitleaks on pushes, pull requests, and manual dispatches.
- A local pre-commit hook template is available in `.githooks/pre-commit`.

Install the local hook once per clone:

```sh
./scripts/install-git-hooks.sh
```

The local hook expects `gitleaks` to be installed. On macOS:

```sh
brew install gitleaks
```
