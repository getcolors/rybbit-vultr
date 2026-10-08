# rybbit-vultr

Desired state for a production-oriented single-node Rybbit web & product analytics deployment on Vultr.

As of 8 October 2026, this is the retained rollback instance. Production DNS
points to Hetzner (`2.31.12.220`), and this host's Caddy forwards late requests
there over verified HTTPS. The original application, databases and SSH identity
remain intact at `78.141.212.24`.

**Do not run `create` or `delete`.** The production DNS binding was removed from
this deployment's state and imported into Hetzner's state on 8 October 2026;
the live record was preserved and the target DNS plan showed no changes.
This source DNS state has no managed records, but its retained desired
configuration still names production and would attempt a competing create.
Keep this profile frozen pending an explicit retirement decision. See
`../rybbit/README.md` and the
[live deployment record](https://wiki.pocketcontext.com/#/page/deployment-profile-rybbit-hetzner).

Rollback requires suspending Hetzner convergence first, because its desired DNS
state would otherwise undo a manual rollback. Then restore `/opt/rybbit/Caddyfile`
**in place**, preserving its
inode, from the host's protected `/var/backups/rybbit-cutover/Caddyfile.before`,
validating/reloading Caddy, then
pointing the existing production A record back to `78.141.212.24` with proxying
preserved. DNS rollback alone would otherwise keep forwarding to Hetzner.
Writes accepted only by Hetzner will not appear here. The source R2 backup job's
previous HTTP 401 remains unresolved; retain the protected local migration
archive. Do not retire this instance without a separate decision.

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
