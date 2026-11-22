# SILO - S3 Interface Libre Object

Self-hosted deployment of [SILO](https://silo.pgsty.com/), an S3-compatible object storage server, running with Docker Compose on a Proxmox host (`silo-1`) and reachable over Tailscale.

## Repository layout

```
.
├── .github/workflows/
│   ├── gitguardian.yml        # Secret scanning on every push / PR
│   ├── proxmox-docker.yml     # Deploys the Docker stack on push to main
│   └── proxmox-system.yml     # Weekly OS upgrade + reboot of the host
└── docker/
    ├── .env                   # Compose project name and compose file
    ├── Makefile               # Bootstrap, run, update and secret helpers
    ├── compose.silo.yml       # SILO service definition
    ├── environment/           # Plain (non-secret) env files per stage
    │   ├── silo.default.env
    │   └── silo.production.env
    └── secret/                # GPG-encrypted (AES256) secret env files per stage
        ├── silo.default.env.gpg
        └── silo.production.env.gpg
```

## Service

| Item        | Value                                         |
|-------------|-----------------------------------------------|
| Image       | `docker.io/pgsty/silo:latest`                 |
| S3 API      | port `9000`                                   |
| Web console | port `9001`                                   |
| Data        | `docker/storage/silo/data` → `/data`          |
| Network     | shares the network of a `tailscale` container |
| Healthcheck | `silo healthcheck ready`                      |

Environment is loaded in this order (later files override earlier ones):

1. `environment/silo.default.env`
2. `environment/silo.${STAGE}.env`
3. `storage/secret/silo.default.env` (decrypted)
4. `storage/secret/silo.${STAGE}.env` (decrypted)

`STAGE` defaults to `default`; set `STAGE=production` on the production host.

## Prerequisites

- Docker with the Compose plugin
- `make`, `gpg` and `openssl`
- A running container named `tailscale` (SILO uses `network_mode: container:tailscale`)
- The secret key used to encrypt the secrets, at `/etc/default/secret.key` (override with `SECRET_KEY=...`)
- [`ggshield`](https://github.com/GitGuardian/ggshield) (optional, for `make scan`)

## Getting started

All commands run from the `docker/` directory.

```sh
cd docker

# Create the storage directory (outside the repo), symlink it as ./storage,
# and decrypt the secrets into storage/secret/
STORAGE_PATH=/path/to/storage make bootstrap

# Start the stack and wait until it is healthy
STAGE=production make start
```

## Make targets

| Target              | Description                                                                      |
|---------------------|----------------------------------------------------------------------------------|
| `bootstrap`         | Create storage at `$STORAGE_PATH`, symlink it to `./storage`, decrypt secrets    |
| `start`             | `docker compose up -d --wait`                                                    |
| `stop`              | `docker compose down`                                                            |
| `update`            | Stop, `git pull --rebase`, pull images, decrypt secrets, start, prune Docker     |
| `listen`            | Follow container logs                                                            |
| `create-secret-key` | Generate a new random key at `$SECRET_KEY`                                       |
| `encrypt-secret`    | Encrypt every file in `storage/secret/` into `secret/*.gpg`                      |
| `decrypt-secret`    | Decrypt every `secret/*.gpg` into `storage/secret/`                              |
| `scan`              | Scan the repository for leaked secrets with `ggshield`                           |

## Managing secrets

Secrets (e.g. `MINIO_ROOT_PASSWORD`) are never committed in plain text. Plain
files live in `storage/secret/` (git-ignored); only the encrypted `.gpg`
copies in `secret/` are committed.

```sh
cd docker
make decrypt-secret                       # get the current plain files
$EDITOR storage/secret/silo.production.env
make encrypt-secret                       # re-encrypt into secret/
make scan                                 # make sure nothing leaked
git add secret/ && git commit -m "Update secret"
```

## CI/CD

- **GitGuardian** — scans every push and pull request for secrets. Requires the `GITGUARDIAN_API_KEY` repository secret.
- **Proxmox (Docker)** — on every push to `main`, joins the tailnet and runs `make update` on `silo-1` in `/docker/services/<repo-name>/docker`.
- **Proxmox (System)** — every Tuesday at 00:00 UTC (or manually), runs a non-interactive `apt full-upgrade` + `autoremove` on `silo-1`, reboots it, and checks it comes back.

The Proxmox workflows use the `proxmox` environment and need these secrets:

| Secret               | Purpose                                 |
|----------------------|-----------------------------------------|
| `TS_OAUTH_CLIENT_ID` | Tailscale OAuth client (tag `tag:github`) |
| `TS_AUDIENCE`        | Tailscale workload identity audience    |
| `SSH_PASSWORD`       | Password for `user@silo-1`              |
