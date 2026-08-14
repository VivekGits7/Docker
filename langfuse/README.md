# Langfuse

Self-hosted Langfuse v4: the web UI, the background worker, and ClickHouse. Postgres, Redis, and MinIO are **not** started here, they run as separate services:

- **Postgres**, already deployed on the host, reached over the shared `langfuse-network`
- **Redis**, its own stack in [`../redis`](../redis), reached over `redis-network`
- **MinIO**, its own stack in [`../minio`](../minio), reached over `minio-network`

```
                    ┌─────────────────────────────────────┐
                    │           langfuse stack            │
                    │  langfuse-web ── langfuse-worker    │
                    │         │              │            │
                    │   langfuse-clickhouse (internal)    │
                    └───┬──────────┬──────────┬───────────┘
     langfuse-network ──┘          │          └── redis-network
            │              minio-network                │
       [ postgres ]           [ minio ]           [ redis-cache ]
      already running        ../minio stack       ../redis stack
```

## Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Web, worker, and ClickHouse services |
| `.env.example` | Template, copy to `.env` and fill in |
| `.env` | Real credentials, gitignored, never commit |

## Prerequisites (do these first, in order)

1. **Create the shared network and wire Postgres onto it** (one time):

   ```bash
   docker network create langfuse-network
   docker network connect langfuse-network <your-postgres-container>
   ```

2. **Create the Langfuse database** inside the running Postgres:

   ```bash
   docker exec -it <your-postgres-container> psql -U <postgres-user> -c "CREATE DATABASE langfuse;"
   ```

3. **Start Redis**: `cd ../redis && docker compose up -d`
4. **Start MinIO**: `cd ../minio && docker compose up -d`

This stack declares all three networks as `external`, so Redis and MinIO must be up before it starts.

## Setup

```bash
cd langfuse
cp .env.example .env
# fill in every value, see the credential guide below
docker compose up -d
```

First boot takes a couple of minutes: ClickHouse comes up, then the web container runs database migrations and the headless init. Watch it with:

```bash
docker compose logs -f langfuse-web
```

When you see `✓ Ready`, open `http://localhost:3000` (or `http://VM_IP:3000`) and log in with `INIT_USER_EMAIL` / `INIT_USER_PASSWORD`.

## Where every credential comes from

### Networks

| Variable | How to get it |
|----------|---------------|
| `DOCKER_NETWORK` | The shared network you created in prerequisites, check with `docker network ls`, default `langfuse-network` |
| `MINIO_NETWORK` | Must match `MINIO_NETWORK` in `../minio/.env`, default `minio-network` |
| `REDIS_NETWORK` | Must match `REDIS_NETWORK` in `../redis/.env`, default `redis-network` |

### Postgres (the already running container)

| Variable | How to get it |
|----------|---------------|
| `POSTGRES_HOST` | The container name: `docker ps --format '{{.Names}}  {{.Image}}' \| grep -i postgres` |
| `POSTGRES_PORT` | Always `5432` here, the container-internal port, never the published host port |
| `POSTGRES_USER` | `docker exec <postgres-container> env \| grep POSTGRES_USER` |
| `POSTGRES_PASSWORD` | `docker exec <postgres-container> env \| grep POSTGRES_PASSWORD` |
| `POSTGRES_DB` | The database you created for Langfuse, `langfuse` |

### Redis (the ../redis stack)

| Variable | How to get it |
|----------|---------------|
| `REDIS_HOST` | Always `redis`, the compose service name on `redis-network`. Never a host IP, the firewall will eat it and you get `ETIMEDOUT` |
| `REDIS_PORT` | Always `6379`, the container port, not the published `6380` |
| `REDIS_PASSWORD` | `grep REDIS_PASSWORD ../redis/.env` |

### MinIO (the ../minio stack)

| Variable | How to get it |
|----------|---------------|
| `MINIO_ENDPOINT` | Always `http://minio:9000`, service name over `minio-network` |
| `MINIO_BUCKET` | `grep MINIO_DEFAULT_BUCKET ../minio/.env`, default `langfuse` |
| `MINIO_ACCESS_KEY` | The root user: `grep MINIO_ROOT_USER ../minio/.env` |
| `MINIO_SECRET_KEY` | The root password: `grep MINIO_ROOT_PASSWORD ../minio/.env` |
| `S3_MEDIA_EXTERNAL_ENDPOINT` | Browser-reachable MinIO URL. Local: `http://localhost:9090`. VM: `http://VM_IP:9090` |

### Generated secrets (make them yourself)

| Variable | Command |
|----------|---------|
| `NEXTAUTH_SECRET` | `openssl rand -base64 32` |
| `SALT` | `openssl rand -base64 32` |
| `ENCRYPTION_KEY` | `openssl rand -hex 32` (must be exactly 64 hex chars) |
| `CLICKHOUSE_PASSWORD` | `openssl rand -base64 24` |
| `INIT_USER_PASSWORD` | Anything strong, this is your UI login |

### API keys (headless init)

| Variable | Rules |
|----------|-------|
| `PUBLIC_KEY` | Must start with `pk-lf-`, e.g. `pk-lf-$(openssl rand -hex 16)` |
| `SECRET_KEY` | Must start with `sk-lf-`, e.g. `sk-lf-$(openssl rand -hex 16)` |

The same pair goes into your app's `.env` for the SDK. After first boot the UI is the source of truth: **Project → Settings → API Keys**. If the SDK gets a 401, the keys in the UI win, not the ones in this file.

## Things you must know before first boot

- **`SALT` and `ENCRYPTION_KEY` are permanent.** They encrypt data at rest. Changing them after first boot breaks decryption of existing data. Set them once, back them up, never touch them again.
- **`NEXTAUTH_URL` must exactly match the browser URL** you use to open Langfuse, scheme, host, and port. Mismatch = broken login redirects.
- **Headless init (`INIT_*`) runs only once, on an empty database.** Changing those values later does nothing, manage users and keys in the UI instead.

## Verify it works

```bash
docker compose ps
curl -s http://localhost:3000/api/public/health   # {"status":"OK"}
```

All three containers should show `(healthy)` or `Up`. The web healthcheck probes `127.0.0.1` on purpose, the Alpine image can resolve `localhost` to IPv6 `::1` where the app is not listening.

## One-shot health check (all services)

Paste this whole block from this folder. It checks every container, hits every health endpoint, prints all credentials, and verifies the API keys against the database.

```bash
VM_IP=72.60.220.164   # or localhost when running locally

echo "================= CONTAINERS ================="
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' | grep -Ei 'NAMES|langfuse|minio|redis|postgres'
echo
echo "================= HEALTH ================="
printf 'langfuse-web : '; curl -s http://localhost:3000/api/public/health; echo
printf 'redis        : '; docker exec redis-cache redis-cli ping 2>&1
printf 'clickhouse   : '; docker exec langfuse-clickhouse wget -qO- http://localhost:8123/ping 2>&1
printf 'minio        : '; curl -s -o /dev/null -w 'HTTP %{http_code}\n' http://localhost:9090/minio/health/live
PG=$(docker ps --format '{{.Names}}' | grep -i postgres | head -1)
printf 'postgres     : '; docker exec "$PG" pg_isready -U postgres 2>&1
echo
echo "================= LANGFUSE UI LOGIN ================="
echo "URL: http://$VM_IP:3000"
grep -E '^INIT_USER_(EMAIL|PASSWORD)=' .env
echo
echo "================= API KEYS (go in your app's .env) ================="
grep -E '^(PUBLIC_KEY|SECRET_KEY)=' .env
printf 'keys valid against DB: '
curl -s -o /dev/null -w 'HTTP %{http_code}\n' \
  -u "$(grep '^PUBLIC_KEY=' .env | cut -d= -f2):$(grep '^SECRET_KEY=' .env | cut -d= -f2)" \
  http://localhost:3000/api/public/projects
echo
echo "================= MINIO ================="
echo "Console: http://$VM_IP:9091 | S3: http://$VM_IP:9090"
grep -E '^MINIO_ROOT_(USER|PASSWORD)=' ../minio/.env
echo
echo "================= REDIS ================="
echo "Inside docker: redis:6379 | From the host: redis-cli -p 6380"
grep '^REDIS_PASSWORD=' ../redis/.env
```

What a perfect run looks like:

| Check | Good output | If it is not |
|-------|-------------|--------------|
| Containers | all `Up ... (healthy)`: langfuse-web, langfuse-worker, langfuse-clickhouse, minio, redis-cache, postgres | `docker compose up -d` in that service's folder, then `docker compose logs -f` |
| langfuse-web | `{"status":"OK",...}` | `docker compose logs -f langfuse-web` here |
| redis | `PONG` | `cd ../redis && docker compose up -d` |
| clickhouse | `Ok.` | usually still booting or out of RAM, wait 2 minutes |
| minio | `HTTP 200` | `cd ../minio && docker compose up -d` |
| postgres | `accepting connections` | the existing Postgres container is down, restart it |
| keys valid against DB | `HTTP 200` | 401 means init ran on an earlier boot with older keys, the UI (Project → Settings → API Keys) is the source of truth |

Full VM edition with firewall checks and laptop-side reachability tests: [../docs/VM_HEALTH_CHECK.md](../docs/VM_HEALTH_CHECK.md)

## Day-2 operations

```bash
docker compose logs -f langfuse-web       # web logs
docker compose logs -f langfuse-worker    # worker logs
docker compose pull && docker compose up -d   # update to latest v4 images
docker compose down                       # stop, ClickHouse data survives in volumes
docker compose down -v                    # DANGER, wipes ClickHouse traces
```

Postgres holds users, projects, and API keys. ClickHouse holds the trace analytics. MinIO holds raw events and media. Back up all three, see the ops section of the self-hosting guide.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `network langfuse-network declared as external, but could not be found` | Run the prerequisite `docker network create langfuse-network` and connect Postgres to it |
| `network minio-network / redis-network ... could not be found` | Start the `../minio` or `../redis` stack first, they own those networks |
| Redis `connect ETIMEDOUT` | `REDIS_HOST` is an IP instead of `redis`, or `REDIS_PORT` is `6380` instead of `6379` |
| Web container `(unhealthy)` but `curl` on the health endpoint returns OK | Old healthcheck probing `localhost`, make sure the compose file matches this repo (probes `127.0.0.1`) and `docker compose up -d` |
| Login loop or redirect to the wrong host | `NEXTAUTH_URL` does not match the URL in your browser |
| SDK gets 401 | Keys in the UI differ from `PUBLIC_KEY`/`SECRET_KEY` here, copy the pair from Project → Settings → API Keys |
| Browser cannot reach `:3000` on a VM but `curl` on the VM works | Provider firewall, open TCP 3000 (and 9090 for media) in the hosting panel and ufw |

## Security notes

- Real credentials live only in `.env` (gitignored, `chmod 600`). Placeholders only in `.env.example`.
- On a public VM open only ports 22, 3000, and 9090. Redis 6380, MinIO console 9091, and any published Postgres port stay closed.
- Any secret ever pasted into a chat, ticket, or screenshot is burned, rotate it.

Full deployment walkthrough: [../docs/SELF_HOSTING.md](../docs/SELF_HOSTING.md) · VM status one-shot: [../docs/VM_HEALTH_CHECK.md](../docs/VM_HEALTH_CHECK.md)
