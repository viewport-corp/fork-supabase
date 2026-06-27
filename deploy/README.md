# Viewport LLC — Supabase self-host deploy (service #410)

Internal-only, gated deployment of self-hosted Supabase onto the **new** Docker
engine via the Dokploy API. NO DNS, NO public subdomain, NO :80/:443 — loopback
binding + health verification only.

- Fork: `viewport-corp/fork-supabase` (fork of `supabase/supabase`)
- Branch: `viewport/deploy`
- Base compose: `docker/docker-compose.yml` (upstream, source of truth)
- Overlay: `deploy/docker-compose.yml` (loopback-only port override)
- Tracking issue: viewport-ops #410 (Part of #404)

## Stack (verified from `docker/docker-compose.yml`, master)

11 containers + 2 named volumes (`db-config`, `deno-cache`):

| Service | Image:tag |
|---|---|
| db | `supabase/postgres:17.6.1.136` |
| studio | `supabase/studio:2026.06.03-sha-0bca601` |
| kong | `kong/kong:3.9.1` |
| auth | `supabase/gotrue:v2.189.0` |
| rest | `postgrest/postgrest:v14.12` |
| realtime | `supabase/realtime:v2.102.3` |
| storage | `supabase/storage-api:v1.60.4` |
| imgproxy | `darthsim/imgproxy:v3.30.1` |
| meta | `supabase/postgres-meta:v0.96.6` |
| functions | `supabase/edge-runtime:v1.74.0` |
| supavisor | `supabase/supavisor:2.9.5` |

**No `analytics` / `vector` service** in the base compose (moved upstream to the
optional `docker-compose.logs.yml`). Deploy the base compose ONLY — do not chain
the logs / s3 / proxy overlays.

## Ports

Only two services publish ports. The overlay narrows both to `127.0.0.1`:

- kong: `127.0.0.1:8000` (single API front door) + `127.0.0.1:8443`
- supavisor: `127.0.0.1:5432` (session) + `127.0.0.1:6543` (transaction)

## Persistence

Bind mounts under `docker/volumes/` hold all state and config and MUST persist
across redeploys:

- `volumes/db/data` — Postgres data
- `volumes/storage` — uploaded files (storage + imgproxy)
- `volumes/functions` — Edge Functions code
- `volumes/api/kong.yml`, `volumes/db/*.sql`, `volumes/pooler/pooler.exs`,
  `volumes/snippets`

Because these ship inside `docker/`, the Dokploy compose context must be the
`docker/` tree on a persistent Dokploy volume (feed via `sourceType: git` so the
bind-mounted config files come with the checkout — `raw` would lose them).

## Secrets — generate on the VPS, never commit

Run on the VPS, never reuse the demo JWTs:

```
cd docker
# 1. choose JWT_SECRET / POSTGRES_PASSWORD, then:
bash utils/generate-keys.sh        # ANON_KEY + SERVICE_ROLE_KEY signed by JWT_SECRET
bash utils/add-new-auth-keys.sh    # JWT_KEYS / JWT_JWKS / *_ASYMMETRIC / PUBLISHABLE / SECRET
```

Must be regenerated (never ship `.env.example` defaults):
`POSTGRES_PASSWORD`, `JWT_SECRET`, `ANON_KEY`, `SERVICE_ROLE_KEY`,
`DASHBOARD_USERNAME`/`DASHBOARD_PASSWORD`, `SECRET_KEY_BASE` (64),
`VAULT_ENC_KEY` (exactly 32), `PG_META_CRYPTO_KEY` (>=32),
S3 keys if used, plus the asymmetric-JWT set above.

URLs for the gated stage: `SUPABASE_PUBLIC_URL` / `API_EXTERNAL_URL` ->
`http://<vps-internal>:8000`; `SITE_URL` -> internal placeholder.
Keep `COMPOSE_FILE=docker-compose.yml` (base only).

All secrets go into the Dokploy env store — never into this branch.


## CRITICAL — port env vars must be BARE numbers; loopback comes from the overlay

`POSTGRES_PORT` is dual-purpose in the base compose: it is the supavisor host
port mapping AND the Postgres server `PGPORT` / all internal connection URLs
(`db:${POSTGRES_PORT}`). Putting a host-IP prefix in it
(`POSTGRES_PORT=127.0.0.1:5432`) makes Postgres fail at init with
`FATAL: invalid value for parameter "port"`, so the whole stack aborts on the
`db` healthcheck.

Therefore:
- Env port vars stay BARE: `POSTGRES_PORT=5432`, `POOLER_PROXY_PORT_TRANSACTION=6543`,
  `KONG_HTTP_PORT=8000`, `KONG_HTTPS_PORT=8443`.
- Loopback binding is applied ONLY by the overlay, which uses `ports: !override`
  so it REPLACES the base ports list (a plain merge would APPEND, leaving the
  public `0.0.0.0` mapping in place and double-binding the port).
- Deploy ONE file: `deploy/docker-compose.deploy.yml`. It uses Compose
  `include:` to pull in the untouched base and `ports: !override` to replace the
  two host port lists with loopback-only bindings. (Dokploy passes an explicit
  `-f <composePath>`, which ignores the `COMPOSE_FILE` env chain and a second
  `-f` overlay — so a single self-contained file is required.)

## Deploy via Dokploy API (Stage 2 plan)

Auth: `x-api-key: $DOKPLOY_API_KEY` (from `/srv/viewport/secrets/platformx.env`)
against `http://194.163.153.171:3001`. New engine only
(`DOCKER_HOST=unix:///var/run/docker-viewport.sock`); old engine read-only.

1. `GET /api/project.all` -> match department NAME **"Shared Data and Storage"**
   -> `projectId`; resolve `environmentId` live from that project (newer Dokploy
   `compose.create` requires it — do not assume).
2. `POST /api/compose.create` — `{ name, appName, projectId, environmentId,
   composeType: "docker-compose" }` -> `composeId`.
3. `POST /api/compose.update` — `sourceType: git` pointing at this fork
   `viewport/deploy`, `composePath: deploy/docker-compose.deploy.yml`; supply the full
   secret `env` block. (Use the overlay for loopback override when applied.)
4. `POST /api/compose.deploy` — `{ composeId }`; then poll `compose.one` /
   `readLogs` and `docker ps` on the new engine.

## Internal verification (no public edge)

```
curl -fsS http://127.0.0.1:8000/                       # Kong / Studio front door
pg_isready -h 127.0.0.1 -p 5432                         # Supavisor session
pg_isready -h 127.0.0.1 -p 6543                         # Supavisor transaction
docker ps --filter name=supabase                       # all 11 healthy
```
