# Immich Blue-Green Migration Runbook

This is the simplified plan for improving the live Immich stack without modifying the currently running directory in place.

The running Immich stack is on an Ubuntu 24.04 LTS server backed by a ZFS pool. This repository contains the compose files, but the live containers and data are not on this workstation.

## Verdict

The blue-green idea is good, but with one rule:

**Do not point the new `immich-v2` stack at the old live volumes while the old stack is running.**

Use cloned upload/database data for testing. At final cutover, stop the old stack, take one final DB backup and one final filesystem sync, then start `immich-v2` from that final consistent state.

This gives you:

- old Immich still available during most of the work
- a separate `photos-v2` Tailscale hostname for testing
- no nginx in the new stack
- a clean rollback path by stopping `immich-v2` and starting the old stack again
- no in-place mutation of the old compose directory or DB directory during testing

## Important Rules

- Do not run `docker compose down -v`.
- Do not run both stacks against the same Postgres data directory.
- Do not run both stacks with write access to the same `UPLOAD_LOCATION`.
- Do not delete the old stack after cutover. Keep it stopped until the new stack has been stable for several days.
- Keep the old internal upload mount target `/usr/src/app/upload` for the first migration. Do not combine this with a `/data` mount migration.

## Source And Server Variables

Run these commands on the Ubuntu server.

```sh
export OLD_STACK_DIR="/opt/stacks/immich"
export NEW_STACK_DIR="/opt/stacks/immich-v2"
export BACKUP_ROOT="/tank/backups/immich-migration-$(date +%Y%m%d-%H%M%S)"
export IMMICH_ZFS_DATASET="tank/appdata/immich"
export OLD_TS_HOSTNAME="photos"
export NEW_TS_HOSTNAME="photos-v2"
```

Change the paths and dataset names to match the server.

Load the old stack env:

```sh
cd "$OLD_STACK_DIR"
set -a
. ./.env
set +a
```

Capture the old values:

```sh
export OLD_UPLOAD_LOCATION="$UPLOAD_LOCATION"
export OLD_DB_DATA_LOCATION="$DB_DATA_LOCATION"
export OLD_DB_USERNAME="$DB_USERNAME"
export OLD_DB_DATABASE_NAME="$DB_DATABASE_NAME"
```

Create new v2 paths:

```sh
mkdir -p "$BACKUP_ROOT"
mkdir -p "$NEW_STACK_DIR"
mkdir -p "$NEW_STACK_DIR/upload"
mkdir -p "$NEW_STACK_DIR/postgres"
mkdir -p "$NEW_STACK_DIR/tailscale-state"
```

## Phase 1: Preflight

Run on the Ubuntu server.

```sh
cd "$OLD_STACK_DIR"
docker compose --env-file .env -f compose.yml ps
docker compose --env-file .env -f compose.yml config > "$BACKUP_ROOT/old-compose.rendered.yml"
docker compose --env-file .env -f compose.yml images > "$BACKUP_ROOT/old-images.txt"
```

Confirm OS, Docker, and ZFS:

```sh
lsb_release -a
docker version
docker compose version
zpool status
zfs list -o name,mountpoint,used,available | sort
df -hT "$OLD_UPLOAD_LOCATION" "$OLD_DB_DATA_LOCATION" "$OLD_STACK_DIR"
findmnt -T "$OLD_UPLOAD_LOCATION"
findmnt -T "$OLD_DB_DATA_LOCATION"
```

Confirm DB health:

```sh
docker exec immich_postgres psql \
  --dbname="$OLD_DB_DATABASE_NAME" \
  --username="$OLD_DB_USERNAME" \
  --command="SELECT version();"

docker exec immich_postgres psql \
  --dbname="$OLD_DB_DATABASE_NAME" \
  --username="$OLD_DB_USERNAME" \
  --command="SELECT extname, extversion FROM pg_extension ORDER BY extname;"

docker exec immich_postgres psql \
  --dbname=postgres \
  --username="$OLD_DB_USERNAME" \
  --command="SELECT datname, checksum_failures, checksum_last_failure FROM pg_stat_database WHERE datname IS NOT NULL ORDER BY datname;"
```

If any database has non-zero checksum failures, stop and investigate.

## Phase 2: Create Two Full Backups And A DB Dump

This phase intentionally pauses writes so DB and filesystem are consistent.

Stop only write-producing app containers first:

```sh
cd "$OLD_STACK_DIR"
docker compose --env-file .env -f compose.yml stop immich-server immich-machine-learning immich_nginx
docker compose --env-file .env -f compose.yml ps
```

Create the logical DB dump while Postgres is still running:

```sh
export BACKUP_ID="$(date +%Y%m%d-%H%M%S)"

docker exec -t immich_postgres pg_dump \
  --clean \
  --if-exists \
  --dbname="$OLD_DB_DATABASE_NAME" \
  --username="$OLD_DB_USERNAME" \
  | gzip > "$BACKUP_ROOT/immich-db-$BACKUP_ID.sql.gz"

gzip -t "$BACKUP_ROOT/immich-db-$BACKUP_ID.sql.gz"
ls -lh "$BACKUP_ROOT/immich-db-$BACKUP_ID.sql.gz"
```

Stop the rest of the old stack briefly for consistent raw directory backups:

```sh
docker compose --env-file .env -f compose.yml stop database ts-authkey-immich
docker compose --env-file .env -f compose.yml ps
```

Take a ZFS snapshot as backup copy 1:

```sh
export SNAPSHOT_NAME="pre-immich-v2-$BACKUP_ID"
zfs snapshot -r "$IMMICH_ZFS_DATASET@$SNAPSHOT_NAME"
zfs list -t snapshot -r "$IMMICH_ZFS_DATASET" | grep "$SNAPSHOT_NAME"
```

Create a second full backup copy with `rsync`. This should go to storage outside the active Immich dataset.

```sh
mkdir -p "$BACKUP_ROOT/upload-copy"
mkdir -p "$BACKUP_ROOT/db-dir-copy"
mkdir -p "$BACKUP_ROOT/stack-files"

rsync -aHAX --numeric-ids --info=progress2 \
  "$OLD_UPLOAD_LOCATION"/ \
  "$BACKUP_ROOT/upload-copy"/

rsync -aHAX --numeric-ids --info=progress2 \
  "$OLD_DB_DATA_LOCATION"/ \
  "$BACKUP_ROOT/db-dir-copy"/

cp -a "$OLD_STACK_DIR/compose.yml" "$BACKUP_ROOT/stack-files"/
cp -a "$OLD_STACK_DIR/.env" "$BACKUP_ROOT/stack-files"/
cp -a "$OLD_STACK_DIR/hwaccel.transcoding.yml" "$BACKUP_ROOT/stack-files"/
cp -a "$OLD_STACK_DIR/hwaccel.ml.yml" "$BACKUP_ROOT/stack-files"/
test -f "$OLD_STACK_DIR/nginx.conf" && cp -a "$OLD_STACK_DIR/nginx.conf" "$BACKUP_ROOT/stack-files"/ || true
```

Restart the old stack so the family can keep using Immich while `immich-v2` is prepared:

```sh
cd "$OLD_STACK_DIR"
docker compose --env-file .env -f compose.yml up -d
docker compose --env-file .env -f compose.yml ps
```

Verify the backup folders:

```sh
find "$BACKUP_ROOT/upload-copy" -maxdepth 1 -type d -print | sort
du -sh "$BACKUP_ROOT/upload-copy" "$BACKUP_ROOT/db-dir-copy" "$BACKUP_ROOT/stack-files"
```

Expected upload folders commonly include:

- `backups`
- `encoded-video`
- `library`
- `profile`
- `thumbs`
- `upload`

## Phase 3: Build The Separate `immich-v2` Stack

The files in this repository directory now represent the intended `immich-v2` stack. You can copy them to `$NEW_STACK_DIR` on the Ubuntu server, then adjust `.env` on the server.

Create a new directory and copy only config files:

```sh
mkdir -p "$NEW_STACK_DIR"
cp /path/to/repo/immich/compose.yml "$NEW_STACK_DIR/"
cp /path/to/repo/immich/.env.example "$NEW_STACK_DIR/"
cp "$OLD_STACK_DIR/hwaccel.transcoding.yml" "$NEW_STACK_DIR/"
cp "$OLD_STACK_DIR/hwaccel.ml.yml" "$NEW_STACK_DIR/"
```

Create `immich-v2/.env` from the example or overwrite it with the old secrets and new v2 paths:

```sh
cat > "$NEW_STACK_DIR/.env" <<EOF
UPLOAD_LOCATION=$NEW_STACK_DIR/upload
DB_DATA_LOCATION=$NEW_STACK_DIR/postgres
TZ=${TZ:-America/Puerto_Rico}
IMMICH_VERSION=v2
DB_PASSWORD=$DB_PASSWORD
DB_USERNAME=$DB_USERNAME
DB_DATABASE_NAME=$DB_DATABASE_NAME
TS_KEY=$TS_KEY
IMMICH_TS_HOSTNAME=$NEW_TS_HOSTNAME
EOF
```

The copied `compose.yml` should already use unique container names, a unique compose project name, a unique Tailscale hostname, and no nginx. The expected shape is:

```sh
cat > "$NEW_STACK_DIR/compose.yml" <<'EOF'
name: immich-v2

services:
  ts-authkey-immich-v2:
    image: tailscale/tailscale:latest
    container_name: immich-v2-tailscale
    hostname: ${IMMICH_TS_HOSTNAME:-photos-v2}
    environment:
      - TS_AUTHKEY=${TS_KEY:?}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
      - TS_AUTH_ONCE=true
    volumes:
      - ./tailscale-state:/var/lib/tailscale
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
    healthcheck:
      test: ["CMD", "tailscale", "status", "--peers=false"]
      interval: 10s
      timeout: 5s
      retries: 12
      start_period: 10s
    restart: unless-stopped

  immich-server:
    container_name: immich-v2-server
    image: ghcr.io/immich-app/immich-server:${IMMICH_VERSION:-v2}
    extends:
      file: hwaccel.transcoding.yml
      service: nvenc
    volumes:
      - ${UPLOAD_LOCATION}:/usr/src/app/upload
      - /etc/localtime:/etc/localtime:ro
    env_file:
      - .env
    network_mode: service:ts-authkey-immich-v2
    depends_on:
      valkey:
        condition: service_healthy
      database:
        condition: service_healthy
      ts-authkey-immich-v2:
        condition: service_healthy
    restart: always
    healthcheck:
      disable: false

  immich-machine-learning:
    container_name: immich-v2-machine-learning
    image: ghcr.io/immich-app/immich-machine-learning:${IMMICH_VERSION:-v2}-cuda
    extends:
      file: hwaccel.ml.yml
      service: cuda
    volumes:
      - model-cache:/cache
    env_file:
      - .env
    restart: always
    healthcheck:
      disable: false

  valkey:
    container_name: immich-v2-valkey
    image: docker.io/valkey/valkey:9@sha256:3b55fbaa0cd93cf0d9d961f405e4dfcc70efe325e2d84da207a0a8e6d8fde4f9
    healthcheck:
      test: valkey-cli ping || exit 1
    restart: always

  database:
    container_name: immich-v2-postgres
    image: ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0
    environment:
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_DB: ${DB_DATABASE_NAME}
      POSTGRES_INITDB_ARGS: '--data-checksums'
    volumes:
      - ${DB_DATA_LOCATION}:/var/lib/postgresql/data
    shm_size: 128mb
    restart: always
    healthcheck:
      disable: false

volumes:
  model-cache:
EOF
```

Render the new stack:

```sh
cd "$NEW_STACK_DIR"
docker compose --env-file .env -f compose.yml config > "$BACKUP_ROOT/immich-v2-compose.rendered.yml"
grep -n "immich_nginx" "$BACKUP_ROOT/immich-v2-compose.rendered.yml" && echo "nginx found: stop" || echo "nginx absent"
grep -n "container_name: immich-v2" "$BACKUP_ROOT/immich-v2-compose.rendered.yml"
grep -n "hostname: photos-v2" "$BACKUP_ROOT/immich-v2-compose.rendered.yml"
grep -n "target: /usr/src/app/upload" "$BACKUP_ROOT/immich-v2-compose.rendered.yml"
```

## Phase 4: Load Cloned Files And Restore DB Into `immich-v2`

Copy the old upload backup into the new upload directory:

```sh
rsync -aHAX --numeric-ids --info=progress2 \
  "$BACKUP_ROOT/upload-copy"/ \
  "$NEW_STACK_DIR/upload"/
```

Start only the new database and cache:

```sh
cd "$NEW_STACK_DIR"
docker compose --env-file .env -f compose.yml pull
docker compose --env-file .env -f compose.yml up -d database valkey
docker compose --env-file .env -f compose.yml ps
```

Wait for the new DB to be healthy:

```sh
watch -n 2 'docker compose --env-file .env -f compose.yml ps'
```

Restore the old DB dump into the new DB:

```sh
gunzip --stdout "$BACKUP_ROOT/immich-db-$BACKUP_ID.sql.gz" \
  | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
  | docker exec -i immich-v2-postgres psql \
      --dbname="$DB_DATABASE_NAME" \
      --username="$DB_USERNAME" \
      --single-transaction \
      --set ON_ERROR_STOP=on
```

If this restore fails with errors related to `vectors`, `pgvecto.rs`, `vectorchord`, extensions, indexes, or schemas, stop here. Do not hand-edit the dump until you understand the error.

At that point, use one of these safer paths:

- use Immich's official restore-from-backup flow in the new `photos-v2` instance
- follow Immich's official pgvecto.rs to VectorChord migration guide
- temporarily use the old `tensorchord/pgvecto-rs` database image in `immich-v2`, prove the blue-green networking/filesystem plan first, and do the database modernization as a separate maintenance window

The blue-green plan and the database-extension migration are separate concerns. Keep them separate if the restore is not clean.

Start the full new stack:

```sh
docker compose --env-file .env -f compose.yml up -d
docker compose --env-file .env -f compose.yml ps
docker compose --env-file .env -f compose.yml logs -f --tail=300 immich-server
```

Open the new stack from a tailnet device:

```text
http://photos-v2:2283
```

If MagicDNS is not available:

```sh
docker exec immich-v2-tailscale tailscale ip -4
```

Then open:

```text
http://<photos-v2-tailscale-ip>:2283
```

Verify:

- Login works.
- Timeline loads.
- Albums load.
- Search loads.
- A recent photo opens.
- A video plays.
- Mobile app can connect to `photos-v2`.
- Upload one test item to `photos-v2`.
- Admin jobs and logs show no DB or storage integrity errors.

## Phase 5: Final Cutover

Do this only after `photos-v2` works on cloned data.

Stop writes to the old stack:

```sh
cd "$OLD_STACK_DIR"
docker compose --env-file .env -f compose.yml stop immich-server immich-machine-learning immich_nginx
```

Create a final DB dump:

```sh
export FINAL_ID="$(date +%Y%m%d-%H%M%S)"

docker exec -t immich_postgres pg_dump \
  --clean \
  --if-exists \
  --dbname="$OLD_DB_DATABASE_NAME" \
  --username="$OLD_DB_USERNAME" \
  | gzip > "$BACKUP_ROOT/immich-db-final-$FINAL_ID.sql.gz"

gzip -t "$BACKUP_ROOT/immich-db-final-$FINAL_ID.sql.gz"
```

Final sync old uploads into the new upload directory:

```sh
rsync -aHAX --numeric-ids --delete --info=progress2 \
  "$OLD_UPLOAD_LOCATION"/ \
  "$NEW_STACK_DIR/upload"/
```

Reset only the new v2 database directory and restore the final DB dump. This does not touch the old database directory.

```sh
cd "$NEW_STACK_DIR"
docker compose --env-file .env -f compose.yml stop immich-server immich-machine-learning database valkey

rm -rf "$NEW_STACK_DIR/postgres"/*

docker compose --env-file .env -f compose.yml up -d database valkey
watch -n 2 'docker compose --env-file .env -f compose.yml ps'

gunzip --stdout "$BACKUP_ROOT/immich-db-final-$FINAL_ID.sql.gz" \
  | sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
  | docker exec -i immich-v2-postgres psql \
      --dbname="$DB_DATABASE_NAME" \
      --username="$DB_USERNAME" \
      --single-transaction \
      --set ON_ERROR_STOP=on
```

If the final restore fails for extension or index reasons, do not start v2. Keep the old stack stopped, copy the error into the migration notes, and either fix the restore path or roll back by starting the old stack again.

Start v2:

```sh
docker compose --env-file .env -f compose.yml up -d
docker compose --env-file .env -f compose.yml ps
docker compose --env-file .env -f compose.yml logs --tail=300 immich-server
```

At this point, keep the old stack stopped and use:

```text
http://photos-v2:2283
```

## Phase 6: Optional Hostname Swap

Do this only after `photos-v2` is accepted as the production instance.

Stop the old Tailscale container so the old `photos` node is offline:

```sh
cd "$OLD_STACK_DIR"
docker compose --env-file .env -f compose.yml stop ts-authkey-immich
```

Change v2 `.env`:

```sh
cd "$NEW_STACK_DIR"
cp .env "$BACKUP_ROOT/immich-v2.env.before-hostname-swap"
sed -i 's/^IMMICH_TS_HOSTNAME=.*/IMMICH_TS_HOSTNAME=photos/' .env
```

Force a fresh Tailscale state for v2 so it can register as `photos`:

```sh
docker compose --env-file .env -f compose.yml stop immich-server ts-authkey-immich-v2
mv tailscale-state "$BACKUP_ROOT/tailscale-state-photos-v2-before-hostname-swap"
mkdir -p tailscale-state
docker compose --env-file .env -f compose.yml up -d ts-authkey-immich-v2
docker exec immich-v2-tailscale tailscale status --peers=false
docker exec immich-v2-tailscale tailscale ip -4
docker compose --env-file .env -f compose.yml up -d
```

Open:

```text
http://photos:2283
```

Update mobile apps to the final URL if needed.

## Rollback

For rollback before hostname swap:

```sh
cd "$NEW_STACK_DIR"
docker compose --env-file .env -f compose.yml stop

cd "$OLD_STACK_DIR"
docker compose --env-file .env -f compose.yml up -d
docker compose --env-file .env -f compose.yml ps
```

For rollback after hostname swap:

```sh
cd "$NEW_STACK_DIR"
docker compose --env-file .env -f compose.yml stop

cd "$OLD_STACK_DIR"
docker compose --env-file .env -f compose.yml up -d
```

If the old Tailscale hostname does not return immediately, check the Tailscale admin console for duplicate/offline `photos` nodes and remove the stale one.

Do not delete `immich-v2` until the old stack has been intentionally retired.

## Why This Is Safer Than Pointing At Old Volumes Early

Immich stores metadata and file paths in Postgres. The filesystem alone is not enough to recreate the app state. The database backup and the upload directory must represent the same point in time.

Running old and new Immich against the same writable upload directory can create mismatched thumbnails, encoded videos, backups, and uploads. Running two Postgres containers against the same database directory is unsafe.

The safe pattern is:

1. clone files
2. restore DB into a new DB directory
3. test on `photos-v2`
4. stop old
5. final sync and final DB restore
6. start v2
7. keep old stopped for rollback

## References

- Immich backup and restore docs: https://docs.immich.app/administration/backup-and-restore/
- Immich Docker Compose install docs: https://docs.immich.app/install/docker-compose/
- Immich custom locations docs: https://immich.app/docs/guides/custom-locations
- Immich environment variables docs: https://immich.app/docs/install/environment-variables/
