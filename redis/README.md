# Redis

Dedicated Redis 7 instance used by Langfuse as its queue and cache. It runs as its own compose stack on its own Docker network (`redis-network`) so it stays independent from any other Redis already running on the machine.

Langfuse requires Redis 7 or newer with `maxmemory-policy noeviction`, which is why the existing Redis 6 container on the VM cannot be reused.

## Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | The Redis service, volume, and network definition |
| `.env.example` | Template, copy to `.env` and fill in |
| `.env` | Real credentials, gitignored, never commit |

## Setup

```bash
cd redis
cp .env.example .env

# Generate a strong password and put it in .env
openssl rand -base64 24

docker compose up -d
```

That's it. The `redis-network` is created automatically by this stack.

## Configuration (.env)

| Variable | What it is | Notes |
|----------|-----------|-------|
| `REDIS_PASSWORD` | The `requirepass` password | Must match `REDIS_PASSWORD` in `../langfuse/.env` |
| `REDIS_NETWORK` | Name of this stack's Docker network | Default `redis-network`, must match `REDIS_NETWORK` in `../langfuse/.env` |

## How other containers reach it

- Containers on `redis-network` (Langfuse web and worker) connect to **`redis:6379`** using the compose service name. They never use the host port.
- From the host, the port is **6380** (`6380:6379` mapping) to avoid clashing with any other Redis on the machine. The container side is always 6379.
- The container is named `redis-cache` because the plain name `redis` was already taken by another container on the VM. DNS-wise both the service name `redis` and `redis-cache` resolve on the network.

## Verify it works

```bash
docker ps --filter name=redis-cache --format '{{.Names}}  {{.Status}}'
docker exec redis-cache redis-cli ping        # PONG = good, auth comes from REDISCLI_AUTH
docker exec redis-cache redis-cli config get maxmemory-policy   # must say noeviction
```

## Day-2 operations

```bash
docker compose logs -f redis          # follow logs
docker compose restart redis          # restart
docker compose down                   # stop, data survives in the redis-data volume
docker compose down -v                # stop AND wipe data, full reset
```

To read the current password without printing the whole file:

```bash
grep REDIS_PASSWORD .env
```

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Memory overcommit must be enabled` warning in logs | `sudo sysctl vm.overcommit_memory=1` then persist it: `echo 'vm.overcommit_memory = 1' \| sudo tee /etc/sysctl.d/99-redis.conf` |
| `Conflict. The container name "/redis" is already in use` | Another container owns that name. This stack already uses `redis-cache`, make sure your compose file matches this repo |
| Langfuse logs show `NOAUTH` or `WRONGPASS` | `REDIS_PASSWORD` in `../langfuse/.env` does not match the one here |
| Langfuse logs show `connect ETIMEDOUT` | Langfuse is pointing at an IP instead of the service name. Set `REDIS_HOST=redis` and `REDIS_PORT=6379` in `../langfuse/.env` |
| Password change not applying | The password is baked into the container command, run `docker compose up -d` (recreate), not just `restart` |

## Security notes

- Never put the real password in `.env.example`, it is committed. Real values live only in `.env` (gitignored, `chmod 600`).
- Keep host port 6380 closed in the VM firewall. Only Langfuse needs Redis and it talks over the Docker network.

Full deployment walkthrough: [../docs/SELF_HOSTING.md](../docs/SELF_HOSTING.md)
