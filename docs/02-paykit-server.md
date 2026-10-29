# 2. Running paykit-server

Source: [pubky/paykit-server](https://github.com/pubky/paykit-server). It builds from
its `Dockerfile` and needs PostgreSQL.

## Environment

| Variable | Purpose |
|---|---|
| `PAYKIT_CONFIG` | Path of `config.toml` inside the container, e.g. `/home/paykit/config.toml`. |
| `PAYKIT_DATABASE_URL` | Its own Postgres database. |
| `PAYKIT_MASTER_KEY` | 32 bytes, base64url. Encrypts persisted state. **Lose it and the database is unreadable**: keep it in a secrets manager. |

## config.toml, the parts you will edit

```toml
[http]
listen_addr = "0.0.0.0:8080"
trusted_proxy_hops = 1            # one reverse proxy (Traefik…) in front

[signed_services]
# One Ed25519 public key per calling service, pubky-prefixed z-base-32.
trusted_public_keys = [
  "pubky…service-a",
  "pubky…service-b",
]

[setup]
# Exact origins allowed to embed the Creator /setup iframe.
allowed_origins = ["https://service-a.example", "https://service-b.example"]

[paykit]
network = "mainnet"
payment_window = "24h"

[usdt]
rpc_url = "https://arb1.arbitrum.io/rpc"   # enables USDT0 on Arbitrum One
```

Start from `config/paykit-server.example.toml` in the repository for the rest
(electrum, outbox, rate limits…).

**The config is read once, at startup.** Adding a key or an origin does nothing until
the container restarts.

## Health

- `GET /health/live`, `GET /health/ready`, `GET /metrics` (no signature).
- A cheap end-to-end check of a service key: signed `POST /setup/status` with
  `{"creator":"pubky…","accepted_asset":"USDT"}` must answer `200 {"status":"ready"}`.
  A key missing from `trusted_public_keys` answers `401 invalid_signature`.

## Pitfalls we hit

### Redeploying pulled a new version

Building from the repository's default branch means every redeploy ships whatever
`master` is that day. Pin a commit or tag, and upgrade on purpose after testing.

### The file mount that never updated (Dokploy)

The config was mounted as a Dokploy "file mount" with no file name. On the host,
Dokploy stored it as a **file named `files`** instead of `files/<name>`. From then
on, every edit of the mount was saved in Dokploy's database and **never written to
disk** (`mkdir files` fails because `files` is a file), so restarts kept loading the
old config, with no error in the UI.

Symptoms: a key added to `trusted_public_keys` still gets `401 invalid_signature`
after a restart; editing the mount with a file name fails with
`mkdir …/files: File exists`.

Fix, on the host:

```bash
cd /etc/dokploy/applications/<app-name>
cp files /root/paykit-config.backup.toml
mv files config.toml.bak && mkdir files && cp config.toml.bak files/config.toml
```

then give the mount the file name `config.toml` (mount path unchanged), save the new
content, and redeploy. Always set a file name on Dokploy file mounts.

### Restart ≠ reload

On Dokploy, "Reload" did not recreate the container; a mount change needed a
redeploy. Check the container start time before testing.

## Backups

Back up the Postgres database **and** `PAYKIT_MASTER_KEY` together; one without the
other is useless.
