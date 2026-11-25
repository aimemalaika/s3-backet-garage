# Ulk File Server — Garage object storage

A self-hosted, S3-compatible object store for the Ulk project, running a single-node
[Garage](https://garagehq.deuxfleurs.fr/) cluster in Docker. Data is persisted to host
folders so it survives container restarts and recreations.

- **Engine:** Garage `v2.6.0` (S3-compatible)
- **Bucket:** `ulk`
- **Mode:** single node, replication factor 1 (no redundancy — dev/local)

---

## Contents

- [Prerequisites](#prerequisites)
- [Project layout](#project-layout)
- [Quick start](#quick-start)
- [Connecting over S3](#connecting-over-s3)
- [Admin CLI](#admin-cli)
- [Web Admin UI](#web-admin-ui)
- [Secrets & configuration](#secrets--configuration)
- [Persistence & backups](#persistence--backups)
- [Ports](#ports)
- [Troubleshooting](#troubleshooting)
- [Rebuilding from scratch](#rebuilding-from-scratch)

---

## Prerequisites

- Docker + Docker Compose
- (optional) `aws` CLI for interacting with the bucket from the shell

---

## Project layout

```
file-server/
├── docker-compose.yml   # Garage service definition
├── garage.toml          # Garage config (NO secrets — they come from env)
├── .env                 # secrets + client credentials  (gitignored)
├── .gitignore           # ignores .env, meta/, data/
├── README.md
├── meta/                # persisted cluster + bucket metadata (LMDB)  (gitignored)
└── data/                # persisted object blocks                     (gitignored)
```

Only `docker-compose.yml`, `garage.toml`, `.gitignore`, and `README.md` are meant to be
committed. `.env`, `meta/`, and `data/` stay local.

---

## Quick start

```sh
docker compose up -d        # start Garage (reads secrets from .env)
docker compose ps           # check it's healthy
docker compose logs -f      # follow logs
docker compose down         # stop (data in ./meta and ./data is kept)
```

The cluster layout, the `ulk` bucket, and an access key are already provisioned and live
in `./meta`, so a plain `up -d` is all you need on an existing checkout. If you're
starting from empty `meta/`/`data/`, see [Rebuilding from scratch](#rebuilding-from-scratch).

---

## Connecting over S3

Credentials live in `.env`. With the `AWS_*`/`S3_*` values available in your environment,
point any S3 tool at the local endpoint.

```sh
# list
aws --endpoint-url "$S3_ENDPOINT" s3 ls "s3://$S3_BUCKET/"

# upload / download
aws --endpoint-url "$S3_ENDPOINT" s3 cp ./localfile "s3://$S3_BUCKET/path/to/key"
aws --endpoint-url "$S3_ENDPOINT" s3 cp "s3://$S3_BUCKET/path/to/key" ./localfile
```

| Setting     | Value                   |
|-------------|-------------------------|
| Endpoint    | `http://localhost:3900` |
| Region      | `garage`                |
| Bucket      | `ulk`                   |
| Path style  | required (not virtual-hosted) |

For SDKs (boto3, aws-sdk-js, etc.): use the same endpoint/region/key, and enable
**path-style** addressing.

---

## Admin CLI

Garage ships a `garage` binary inside the container:

```sh
docker compose exec garage /garage status              # cluster + node health
docker compose exec garage /garage bucket list
docker compose exec garage /garage bucket info ulk     # size, objects, keys
docker compose exec garage /garage key list
docker compose exec garage /garage key info file-server-key
```

Create another bucket and grant the existing key:

```sh
docker compose exec garage /garage bucket create my-bucket
docker compose exec garage /garage bucket allow --read --write --owner \
  my-bucket --key GKf93c4d2b42c68aacf285813f
```

---

## Web Admin UI

A separate Garage Web Admin container provides a browser dashboard (buckets, keys, object
browsing). It talks to Garage's **Admin API** on port `3903` and logs in with the
**admin token** (`GARAGE_ADMIN_TOKEN` in `.env`).

> The Web Admin UI is not yet part of `docker-compose.yml`. If you want it to start
> automatically alongside Garage, it can be folded into the compose stack.

---

## Secrets & configuration

No secrets are stored in committed files. Garage reads them from environment variables
that `docker-compose.yml` injects from `.env`:

| Variable             | Used for                                  |
|----------------------|-------------------------------------------|
| `GARAGE_RPC_SECRET`  | node-to-node RPC auth (`rpc_secret`)      |
| `GARAGE_ADMIN_TOKEN` | Admin API + Web Admin login (`admin_token`) |
| `AWS_ACCESS_KEY_ID`  | S3 access key                             |
| `AWS_SECRET_ACCESS_KEY` | S3 secret key                          |

`garage.toml` references these with comments instead of literal values.

**These values were generated for local dev.** Rotate them before using this setup
anywhere shared or production-facing:

```sh
openssl rand -hex 32      # new GARAGE_RPC_SECRET
openssl rand -base64 32   # new GARAGE_ADMIN_TOKEN
# new S3 key:
docker compose exec garage /garage key create new-key
```

---

## Persistence & backups

Durability comes from two bind mounts declared in `docker-compose.yml`:

- `./meta` → `/var/lib/garage/meta` — bucket/key/layout metadata (LMDB)
- `./data` → `/var/lib/garage/data` — the actual object bytes

`docker compose down` and `up` preserve these. **Deleting either folder destroys the
cluster and all objects.** To back up, stop the container and copy both folders together
(they must stay consistent):

```sh
docker compose down
tar czf garage-backup-$(date +%F).tgz meta data
docker compose up -d
```

---

## Ports

| Port | Purpose                      |
|------|------------------------------|
| 3900 | S3 API                       |
| 3901 | RPC (node-to-node)           |
| 3902 | S3 web (static site hosting) |
| 3903 | Admin API                    |

---

## Troubleshooting

**Container won't start / secret errors** — ensure `.env` exists and is loaded. Compose
reads `.env` automatically for `${GARAGE_RPC_SECRET}` / `${GARAGE_ADMIN_TOKEN}`
substitution; missing values make Garage refuse to boot.

**`NO ROLE ASSIGNED` in `garage status`** — the cluster layout isn't applied. See
[Rebuilding from scratch](#rebuilding-from-scratch).

**S3 calls return 403 / SignatureDoesNotMatch** — check region is `garage`, path-style
addressing is on, and the key has permission (`garage bucket info ulk`).

**Admin UI login fails** — confirm the admin API is reachable and the token matches:

```sh
curl -H "Authorization: Bearer $GARAGE_ADMIN_TOKEN" http://localhost:3903/v2/GetClusterHealth
```

---

## Rebuilding from scratch

If `meta/`/`data/` are empty (fresh clone, or you wiped them), provision the cluster:

```sh
docker compose up -d

# 1. find the node id
NODE=$(docker compose exec -T garage /garage node id -q | cut -d@ -f1)

# 2. assign it to a zone with capacity, then apply
docker compose exec -T garage /garage layout assign -z dc1 -c 10G "$NODE"
docker compose exec -T garage /garage layout apply --version 1

# 3. create the bucket + an access key, and link them
docker compose exec -T garage /garage bucket create ulk
docker compose exec -T garage /garage key create file-server-key    # note the printed Key ID + Secret
docker compose exec -T garage /garage bucket allow --read --write --owner ulk --key <KEY_ID>
```

Then put the printed key/secret into `.env` as `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`.
