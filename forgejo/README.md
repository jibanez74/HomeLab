# Forgejo Stack

This stack runs Forgejo behind a Tailscale sidecar with PostgreSQL.

It does not publish host ports. Access the web UI from a tailnet device:

```text
http://forgejo:3000/
```

Start:

```sh
docker compose --env-file .env -f compose.yml up -d
```

The first web visit will complete Forgejo setup and admin-user creation.
