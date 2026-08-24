# rybbit-vultr

Desired state for a production-oriented single-node Rybbit web & product analytics deployment on Vultr.

## Architecture

- **Domain**: `https://rybbit.getcolors.ai` (Cloudflare-proxied)
- **Location**: `ams` (Amsterdam)
- **Instance**: `vc2-2c-4gb` on Ubuntu 24.04
- **Databases**:
  - PostgreSQL 17 (`postgres:17-alpine`)
  - ClickHouse 24.8 (`clickhouse/clickhouse-server:24.8-alpine`)
  - Redis (`redis:8.6.4-alpine`)
- **Ingress**: Caddy origin TLS terminating on 80/443 (+ UDP 443 for HTTP/3)
- **Backups**: Daily systemd timer uploading archives to the shared `rybbit-backup` R2 bucket under the `rybbit-vultr/` prefix

The Rybbit application images are pinned by digest; move backend and client
together when updating them.

## Usage

```sh
./green build
./green create --dry-run
./green create
```

The machine SSH keypair is `~/.ssh/rybbit-vultr`(`.pub`), named by profile per
`workspace/standards/ssh-keypair.md`. The `rybbit-vultr` entry in
`~/.ssh/config` matches both the alias and the instance address, so converges
and ad-hoc `ssh` authenticate with it directly; no agent is required.

## Operations & Verification

```sh
# Health check
curl -fsS https://rybbit.getcolors.ai/api/health

# Send synthetic test event
curl -fsS -X POST -H 'content-type: application/json' \
  --data '{"name":"pageview","site_id":"benchmark","data":{"path":"/test"}}' \
  https://rybbit.getcolors.ai/api/track

# Run backup service on host (use the instance address: the hostname is proxied)
ssh root@SERVER 'systemctl start rybbit-backup.service'
ssh root@SERVER 'systemctl status rybbit-backup.timer'
```

## Recovery

Persistent data lives under `/var/lib/rybbit` on the instance and survives
restarts. Nightly archives (PostgreSQL dump + ClickHouse `BACKUP`, restore-
drilled before upload) land in `r2:rybbit-backup/rybbit-vultr/`. This
single-node design is durable but not highly available.

## License

MIT License. Copyright (c) 2026 getcolors.
