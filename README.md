# Formbricks — self-hosted (Komodo stack)

Self-hosts [Formbricks](https://formbricks.com) v5 as a [Komodo](https://komo.do)
**Stack**, fronted by a **Cloudflare Tunnel** for public HTTPS. Postgres is bundled
inside the stack.

This repo is **compose-only**. The Komodo `ResourceSync` that declares/points at
this stack lives in a separate project.

## What's in the stack

| Service              | Image                              | Purpose                                    |
| -------------------- | ---------------------------------- | ------------------------------------------ |
| `postgres`           | `pgvector/pgvector:pg18`           | Database (bundled, persisted volume)       |
| `redis`              | `valkey/valkey` (pinned digest)    | Cache, rate limiting, audit log            |
| `formbricks-migrate` | `ghcr.io/formbricks/formbricks`    | Runs Prisma migrations, then exits         |
| `formbricks`         | `ghcr.io/formbricks/formbricks`    | The web app (port 3000)                    |
| `hub-migrate`        | `ghcr.io/formbricks/hub`           | Runs Hub DB migrations, then exits         |
| `hub`                | `ghcr.io/formbricks/hub`           | Hub API (internal, port 8080)              |
| `cube`               | `cubejs/cube:v1.6.6`               | Analytics semantic layer (internal, 4000)  |

`hub` and `cube` are part of the **v5 baseline** — the app depends on them. The
optional AI services (`taxonomy`, `vllm`/Qwen) are intentionally omitted; see
[Enabling AI](#enabling-ai-optional) to add them.

Config files `cube/cube.js` and `cube/schema/FeedbackRecords.js` are vendored
verbatim from the Formbricks 5.1.4 release and mounted into the Cube container.

## Files

```
compose.yaml           # the stack (Komodo's default file_paths)
.env.example           # template to paste into the Komodo Stack "Environment" field
cube/                  # Cube config (required by the cube service)
saml-connection/       # runtime mount for SAML (kept empty via .gitkeep)
```

> This repo does **not** contain a `.env`. Komodo writes one on the server from the
> Stack's **Environment** field (see below); `.env` is gitignored so secrets are
> never committed.

## Prerequisites

- A Komodo Server (Periphery) with Docker + Docker Compose.
- A Cloudflare account with a Tunnel, running as a **separate** `cloudflared` stack
  on the same Docker host.
- A DNS hostname routed by the tunnel (placeholder used here: `forms.example.com`).

## Setup

### 1. Create the external tunnel network (once, on the server)

Komodo does **not** create external networks. On the Komodo server host, create the
shared network the tunnel uses (name must match `PROXY_NETWORK`, default `cloudflared`):

```bash
docker network create cloudflared
```

### 2. Prepare the environment values

Start from `.env.example` and generate the secrets — you'll paste the result into
Komodo, not commit it:

```bash
for k in NEXTAUTH_SECRET ENCRYPTION_KEY CRON_SECRET HUB_API_KEY CUBEJS_API_SECRET; do
  echo "$k=$(openssl rand -hex 32)"
done
```

Set `WEBAPP_URL` and `NEXTAUTH_URL` to your Cloudflare hostname (both identical,
e.g. `https://forms.example.com`) and pick a strong `POSTGRES_PASSWORD` **before
first boot** (changing it later means migrating the DB).

### 3. Create the Komodo Stack

In Komodo → **Stacks → New**, configure:

| Komodo field          | Value                                                    |
| --------------------- | -------------------------------------------------------- |
| Source / provider     | Git repo → this repository, branch `main`                |
| **Run Directory**     | `.` (repo root — where `compose.yaml` and `cube/` live)  |
| **File Paths**        | `compose.yaml` (Komodo's default)                        |
| **Environment**       | paste the filled values from step 2 (the `.env` content) |
| Env File Path         | `.env` (default — leave as is)                            |

Deploy. Komodo clones the repo, writes the Environment to `.env` in the run
directory, and runs `docker compose up -d --env-file .env`. The `${VAR}` refs in
`compose.yaml` resolve from that file, and the app container additionally loads it
via `env_file` so optional vars reach Formbricks.

> The Komodo **Stack resource definition** (ResourceSync TOML) lives in your other
> project. The relevant fields for it are: `run_directory = "."`,
> `file_paths = ["compose.yaml"]`, `env_file_path = ".env"`, and the `environment`
> block. `git_provider` / `repo` / `branch` point at this repository.

First boot ordering is automatic: `postgres`/`redis` become healthy →
`formbricks-migrate` → `hub-migrate` → `cube`/`hub` → `formbricks`.

### 4. Wire the Cloudflare Tunnel (separate stack)

The external network from step 1 is shared between this stack and your `cloudflared`
stack. The `formbricks` web container joins it (port 3000); datastores and internal
APIs stay on the private `default` network. A `127.0.0.1:3000` host bind exists as a
fallback (not exposed publicly).

In your `cloudflared` compose stack (a separate Komodo Stack), join the same external
network and point the tunnel ingress at the container:

```yaml
# cloudflared stack
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    command: tunnel run
    environment:
      TUNNEL_TOKEN: ${TUNNEL_TOKEN}
    networks: [proxy]
networks:
  proxy:
    name: cloudflared        # must match PROXY_NETWORK
    external: true
```

Then in the Cloudflare Zero Trust dashboard (or tunnel config), route
`forms.example.com` → `http://formbricks:3000`.

### 5. First run

Visit `https://forms.example.com` — you should land on the Formbricks setup wizard.
Create the first (owner) account. Self-hosted signup is invite-only by default;
the first user created via the wizard becomes the owner.

## Email (optional but recommended)

Password reset and email verification are **disabled** until SMTP is configured.
Add the `SMTP_*` values and set `EMAIL_VERIFICATION_DISABLED=0` /
`PASSWORD_RESET_DISABLED=0` in the Komodo Stack **Environment** field, then redeploy.

## Updating

- **Track latest** (default): redeploy the stack in Komodo — it pulls the newest
  `:latest` images for Formbricks/Hub together and re-runs the idempotent migrations.
- **Pin a version**: set `FB_IMAGE_REF=:5.1.4` (and optionally `HUB_IMAGE_REF`) in
  the Komodo **Environment** field, then redeploy. Keep Formbricks and Hub tags
  compatible for your release.

Migrations run automatically on every deploy via the `*-migrate` services.

## Backups

All state is in the `postgres` docker volume (plus the `redis` volume, which is
just cache). Back it up with `pg_dump` — run this from the Komodo Stack terminal or
on the server host:

```bash
docker exec -i formbricks-postgres-1 pg_dump -U postgres formbricks | gzip > formbricks-$(date +%F).sql.gz
```

(Container name follows Komodo's `<stack>-<service>-N` pattern; adjust the prefix to
your stack name, or use `docker compose -p <stack> exec ...`.)

## Enabling AI (optional)

The bundled Qwen/vLLM (GPU) and taxonomy services are not included here to keep the
baseline lean. To add them, copy the `taxonomy` / `vllm` service definitions and
the `qwen`/`taxonomy` compose profiles from the upstream
[`docker/docker-compose.yml`](https://github.com/formbricks/formbricks/blob/main/docker/docker-compose.yml)
and set the corresponding `AI_*` / `TAXONOMY_*` / `COMPOSE_PROFILES` env vars.

## Notes / decisions

- `cube` and `hub` are required baseline services in Formbricks v5 — removing them
  breaks the app's startup dependency chain.
- Redis is **not** published to the host (internal-only).
- Postgres uses the pg18 data path (`/var/lib/postgresql`), matching upstream.
