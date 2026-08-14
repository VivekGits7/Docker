# MinIO

S3-compatible object storage used by Langfuse for event uploads and media files. It runs as its own compose stack on its own Docker network (`minio-network`), fully independent from the Langfuse stack.

The default bucket is created automatically on startup, so Langfuse never hits a missing bucket.

## Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | The MinIO service, volume, and network definition |
| `.env.example` | Template, copy to `.env` and fill in |
| `.env` | Real credentials, gitignored, never commit |

## Setup

```bash
cd minio
cp .env.example .env

# Generate a strong password and put it in .env
openssl rand -base64 24

docker compose up -d
```

The `minio-network` is created automatically by this stack. Start MinIO **before** the Langfuse stack, Langfuse declares this network as external.

## Configuration (.env)

| Variable | What it is | Notes |
|----------|-----------|-------|
| `MINIO_ROOT_USER` | Root user, doubles as the S3 access key | Must match `MINIO_ACCESS_KEY` in `../langfuse/.env` |
| `MINIO_ROOT_PASSWORD` | Root password, doubles as the S3 secret key | Must match `MINIO_SECRET_KEY` in `../langfuse/.env` |
| `MINIO_DEFAULT_BUCKET` | Bucket created on startup | Must match `MINIO_BUCKET` in `../langfuse/.env`, default `langfuse` |
| `MINIO_NETWORK` | Name of this stack's Docker network | Must match `MINIO_NETWORK` in `../langfuse/.env` |

## Ports

| Mapping | What it is | Who uses it |
|---------|-----------|-------------|
| `9090 -> 9000` | S3 API | Browsers opening presigned media URLs, so this one must be reachable from outside. Langfuse containers use `http://minio:9000` over the Docker network instead |
| `9091 -> 9001` | Web admin console | You, for browsing buckets and objects. Log in with the root user and password |

Container-to-container traffic never touches the host ports, Langfuse reaches the API at **`http://minio:9000`** via the service name on `minio-network`.

## Verify it works

```bash
docker ps --filter name=minio --format '{{.Names}}  {{.Status}}'
curl -s http://localhost:9090/minio/health/live -o /dev/null -w '%{http_code}\n'   # 200 = good
```

Open the console at `http://localhost:9091` (or `http://VM_IP:9091` if reachable) and check the `langfuse` bucket exists. After Langfuse has processed some traces you should see objects under `events/` and `media/`.

## Day-2 operations

```bash
docker compose logs -f minio          # follow logs
docker compose restart minio          # restart
docker compose down                   # stop, data survives in the minio-data volume
docker compose down -v                # stop AND wipe all stored objects, full reset
```

To read the current credentials without printing the whole file:

```bash
grep MINIO_ROOT .env
```

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Langfuse logs show `AccessDenied` or signature errors | `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` in `../langfuse/.env` do not match the root user and password here |
| Langfuse logs show `NoSuchBucket` | `MINIO_BUCKET` in `../langfuse/.env` does not match `MINIO_DEFAULT_BUCKET` here |
| Media files do not load in the Langfuse UI | `S3_MEDIA_EXTERNAL_ENDPOINT` in `../langfuse/.env` must be a browser-reachable URL, on a VM that is `http://VM_IP:9090`, and port 9090 must be open in the firewall |
| Credential change not applying | Run `docker compose up -d` (recreate), not just `restart` |

## Security notes

- On a public VM, open only port **9090** in the firewall (needed for presigned media URLs). Keep **9091** closed, or bind it to loopback with `"127.0.0.1:9091:9001"` and reach the console through an SSH tunnel: `ssh -L 9091:127.0.0.1:9091 user@VM_IP`. The console is a full admin panel behind root credentials.
- Real credentials live only in `.env` (gitignored, `chmod 600`), never in `.env.example`.

Full deployment walkthrough: [../docs/SELF_HOSTING.md](../docs/SELF_HOSTING.md)
