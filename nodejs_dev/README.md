# Node.js Development Stack

Start the dev container:

```sh
docker compose --env-file .env -f compose.yml up -d
```

Run commands inside the workspace:

```sh
docker compose -f compose.yml exec node-dev npm create vite@latest .
docker compose -f compose.yml exec node-dev npm install
docker compose -f compose.yml exec node-dev npm run dev -- --host 0.0.0.0
```

Project files live in `workspace/` and are mounted at `/workspace` inside the container.

The Postgres service is available to Node apps at `postgres:5432`. The Node
container also receives `DATABASE_URL` from the values in `.env`.

The sample `.env` uses `POSTGRES_DATA_TARGET=/var/lib/postgresql`, which matches
the official PostgreSQL 18+ image layout used by `postgres:latest`. If you pin
Postgres to 16 or 17, set `POSTGRES_DATA_TARGET=/var/lib/postgresql/data` before
creating the database volume.
