# Atlans Technical Reference

The detailed reference that used to live in the main [README](../README.md): configuration, services, the executor protocol, the API surface and the inner workings of the engine. For the big picture, start with [architecture.md](architecture.md); to install, follow [self-hosting.md](self-hosting.md).

---

## Contents

- [Environment Variables](#environment-variables)
- [Docker Services](#docker-services)
- [Executor System](#executor-system)
- [API — Endpoint Reference](#api--endpoint-reference)
- [Authentication](#authentication)
- [Workflow Engine](#workflow-engine)
- [Scheduling](#scheduling)
- [Real-Time Events](#real-time-events)
- [Multi-tenancy](#multi-tenancy)
- [Monitoring](#monitoring)
- [Makefile](#makefile)
- [CI/CD](#cicd--github-actions)

---

## Environment Variables

Copy `.env.example` to `.env` and adjust the values. The main ones:

### Database and Infrastructure

| Variable | Description | Example |
|---|---|---|
| `DATABASE_URL` | Async URL of the **external** PostgreSQL (with postgis + uuid-ossp) | `postgresql+asyncpg://user:pass@host:5432/atlansdb` |
| `REDIS_PASSWORD` | Redis password | Generate with `python -c "import secrets; print(secrets.token_hex(32))"` |
| `POOL_SIZE` | PostgreSQL pool size, **per worker** (code default: `8`) | `8` |
| `MAX_OVERFLOW` | Extra connections allowed, **per worker** (default: `5`) | `5` |

> **Connection ceiling.** The two values above are per process, and the production API runs with 4 workers:
> the total is `workers × (POOL_SIZE + MAX_OVERFLOW)` = **52** with the code defaults. Check `SHOW max_connections`
> on your Postgres and leave headroom for migrations, `psql` and other clients.

### Security and Authentication

| Variable | Description | Example |
|---|---|---|
| `APP_SECRET` | Secret for signing internal JWT tokens | Generate with `python -c "import secrets; print(secrets.token_urlsafe(48))"` |
| `AUTH_SECRET` | Auth.js (NextAuth) key | Generate with `openssl rand -base64 32` |
| `FERNET_KEY` | Fernet key for credential encryption (rotation via `FERNET_KEYS`) | `Fernet.generate_key()` |
| `EXECUTOR_SIGNING_KEY` | Seed of the Ed25519 key that signs jobs and that executors pin at enrollment (**required**: if empty, no executor enrolls and no job is dispatched; `make bootstrap` generates it) | `openssl rand -base64 32` |
| `EXECUTOR_POLICY_ROUTING` | Turns on routing by per-workspace execution policy (`on`/`off`) | `off` |
| `ALLOWED_ORIGINS` | Comma-separated CORS origins | `https://app.exemplo.com` |
| `STEPCA_ROOT_FINGERPRINT` / `STEPCA_PROVISIONER_PASSWORD` | Required in prod for the API to sign OTTs on step-ca | — |

### MinIO (Object Storage)

| Variable | Description | Example |
|---|---|---|
| `MINIO_ENDPOINT` | Internal MinIO URL (inside Docker) | `http://minio:9000` |
| `MINIO_EXTERNAL_ENDPOINT` | External URL reachable by the browser | `http://localhost:9000` |
| `MINIO_ROOT_USER` | MinIO root user (**required**, no default) | — |
| `MINIO_ROOT_PASSWORD` | MinIO root password (**required**, no default) | — |
| `MINIO_BUCKET` | Default bucket for the Drive | `atlans-drive` |
| `MINIO_PRESIGN_EXPIRY` | Lifetime of presigned URLs (seconds); without the variable the compose uses `3600`, `.env.example` writes `900` | `900` |
| `WEBHOOK_RESPONSE_INLINE_LIMIT` | A ResponseNode body larger than N bytes goes to MinIO instead of traveling inline. It is read by the **executor** (`flow/`), in `executor/.env`, not in the server's `.env` | `1048576` |

### Frontend

| Variable | Description | Example |
|---|---|---|
| `API_INTERNA` | Internal API URL (Next.js server side) | `http://api:8000` |
| `API_PORT` | API port visible to the browser (the compose passes it to the web app as `NEXT_PUBLIC_API_PORT`) | `8000` |
| `NOME_NA_TELA` | The name the screen shows (sidebar, sign-in, tab); empty = `Atlans`. The form with the domain is the holder's trademark ([TRADEMARKS.md](../TRADEMARKS.md)) | `Minha Instalação` |
| `AUTH_URL` | Public URL of the app (Auth.js redirect) | `https://app.exemplo.com` |

> All variables are documented in [`.env.example`](../.env.example).

---

## Docker Services

The `docker-compose.yml` does **not** include PostgreSQL — the database is external (see Prerequisites).

### Development (`--profile dev`)

| Service | Image | Port | Description |
|---|---|---|---|
| `redis` | `valkey/valkey:8-alpine` (`REDIS_IMAGE`) | — (internal) | Cache and pub/sub |
| `api` | `Dockerfile.api` | `8000` | FastAPI with hot reload |
| `web-dev` | `web/Dockerfile.ui` | `3000` | Next.js in dev mode |
| `minio` | `minio/minio` | `9000` / `9001` | S3 object storage + console |

### Production (`--profile prod`)

| Service | Image | Port | Description |
|---|---|---|---|
| `redis` | `valkey/valkey:8-alpine` (`REDIS_IMAGE`) | — (internal) | Cache and pub/sub |
| `api-prod` | `Dockerfile.api` | — (via Traefik) | FastAPI with 4 workers |
| `web-prod` | `web/Dockerfile.ui` | — (via Traefik) | Optimized Next.js build |
| `step-ca` | `smallstep/step-ca:0.27.0` | `9000` (internal) | Internal CA that issues the executors' mTLS certs |
| `traefik` | `traefik:v3.6` | `80` / `443` | Reverse proxy + TLS + mTLS termination |
| `minio` | `minio/minio` | — (via Traefik) | S3 object storage |

### Docker Networks

| Network | Use |
|---|---|
| `backend` | Internal communication between API, Redis, MinIO and step-ca |
| `dev-net` | Communication between `api` and `web-dev` (development) |
| `proxy-net` | Services exposed via Traefik (production) |

---

## Executor System

Atlans delegates workflow execution to **external executors** connected over WebSocket on **mTLS**. There is no Celery — all computation happens on the executors.

### Executor Types

| Type | Description |
|---|---|
| **`default`** | Part of the platform's **default pool**, available to any user as a fallback. There can be **more than one** (managed through `POST /executores/set-default` / `unset-default`). |
| **`dedicated`** | Explicitly assigned to users/workspaces. A workspace can pin its executor through the routing policy (`EXECUTOR_POLICY_ROUTING`). |

Visibility is governed by the `is_default` flag (platform pool) and by per-user/per-workspace assignments — not by a three-valued enum.

### Onboarding (OTP + mTLS)

The executor uses neither an API key nor a JWT. It enrolls with a one-time password (OTP) and receives an **mTLS certificate** issued by the internal CA (step-ca):

```
┌──────────┐  POST /executores/            ┌──────────┐  POST /executores/{id}/enroll-otp  ┌──────────┐
│  Absent  │───────────────────────────▶│ pending  │──────────────────────────────────▶│  (OTP)   │
└──────────┘  (creates entry)            └──────────┘  (admin issues 24h OTP)           └────┬─────┘
                                                                                              │ POST /executores/enroll
                                                                                              │ (CSR → mTLS cert)
                                                                                         ┌────▼─────┐
                                                                                         │  active  │
                                                                                         └────┬─────┘
                                          DELETE /executores/{id}  (revokes)                  │  renews via
                                          ◀───────────────────────────────────────────────── ┘  POST /executores/renew-cert
```

The installation script (`GET /executores/install`) pins the CA certificate by SHA-256 to block a CA swap. Details in [docs/mtls-bootstrap.md](mtls-bootstrap.md).

### WebSocket Protocol

**Connection:** `WS /ws/executores/{executor_id}` — authenticated by **mTLS certificate** (validated by Traefik against the internal CA). There is no token in the URL.

| Direction | Message | Description |
|---|---|---|
| `Servidor → Executor` | `job` | Job encrypted (X25519+AES-GCM) and signed (Ed25519) |
| `Servidor → Executor` | `control` | Control plane: revocation, shutdown, config, artifact purge |
| `Servidor → Executor` | `cancel` | Cancels a running job |
| `Servidor → Executor` | `drive_event` | Drive events |
| `Servidor → Executor` | `error` | Rejection of an executor message (`reason` and detail); the executor logs it at WARNING |
| `Executor → Servidor` | `ack` | The job arrived and entered the local queue |
| `Executor → Servidor` | `heartbeat` | Keepalive (every 30s) |
| `Executor → Servidor` | `capacity` | Current capacity (running/queued) — every 10s |
| `Executor → Servidor` | `node_event` | Per-node execution progress |
| `Executor → Servidor` | `job_result` | Final job result |

### Back-pressure

Each executor reports its capacity through `capacity` messages, based on the environment variables of the executor's **host**:

- `EXECUTOR_MAX_CONCURRENT` — concurrent runs (default: `4`).
- `EXECUTOR_MAX_QUEUE_SIZE` — maximum local queue (default: `50`).

The server checks the reported capacity before dispatching and returns **503** when `(queued + running) ≥ (max_concurrent + max_queue)`. To change the effective limits, edit the executor's `.env` and restart the container — the fields stored in the DB are only informative for the UI while the executor is offline.

---

## API — Endpoint Reference

> Interactive documentation: `http://localhost:8000/docs` (Swagger UI).
> Resource routes use the id's **hash** (`{id_hash}`) and are defined without a trailing slash (a trailing slash produces a 307 redirect).

### Authentication (`/auth`)

| Method | Route | Description |
|---|---|---|
| `POST` | `/auth/register` | Creates an account (default workspace created automatically) |
| `POST` | `/auth/login` | Authenticates and returns access + refresh tokens |
| `POST` | `/auth/refresh` | Renews the access token |
| `GET` | `/auth/me` | Data of the authenticated user |
| `POST` | `/auth/logout` | Ends the session |
| `POST` | `/auth/verify-email` · `/auth/resend-verification` | Email verification |
| `POST` | `/auth/forgot-password` · `/auth/reset-password` | Password recovery |

### Workflows (`/workflows`)

| Method | Route | Description |
|---|---|---|
| `GET` | `/workflows` | Lists workflows in the user's workspaces |
| `POST` | `/workflows` | Creates a workflow |
| `GET` · `PUT` · `DELETE` | `/workflows/{id_hash}` | Get / update / delete |
| `POST` | `/workflows/{id_hash}/execute` | Triggers a run (optional inputs) |
| `GET` | `/workflows/{id_hash}/versions` | Version history |
| `POST` | `/workflows/{id_hash}/versions/{n}/restore` | Restores a version |
| `POST` | `/workflows/{id_hash}/runs/{run_id}/retry` | Re-runs a failed run |
| `POST` | `/workflows/{id_hash}/duplicate` | Duplicates the workflow |
| `POST` | `/workflows/{id_hash}/move` · `/move/preview` | Moves to another workspace (with dry run) |

### Executors (`/executores`)

| Method | Route | Description |
|---|---|---|
| `POST` | `/executores/` | Creates an executor (`pending` record) |
| `GET` | `/executores/` · `/my` | Lists all / those accessible to the user |
| `GET` | `/executores/{id}/status` · `/{id}/workspaces` | Status / workspaces |
| `DELETE` | `/executores/{id}` | Revokes an executor |
| `POST` | `/executores/{id}/enroll-otp` | (Admin) issues an enrollment OTP (24h, single use) |
| `POST` | `/executores/enroll` | Enrollment: CSR → mTLS certificate |
| `POST` | `/executores/renew-cert` | mTLS certificate renewal |
| `GET` | `/executores/ca-bundle` · `/server-public-key` | Internal CA root / server's Ed25519 public key |
| `GET` | `/executores/install` · `/install/windows` | Onboarding script / desktop app (Windows) |
| `POST` | `/executores/set-default` · `/unset-default` | (Admin) manages the default pool |

### Schedules (`/workflows/{id_hash}/schedules`)

| Method | Route | Description |
|---|---|---|
| `PUT` | `/workflows/{id_hash}/schedules/{job_id}` | Updates (the Home uses it to pause/resume) |

Schedules are born from the `ScheduleTrigger` node when the workflow is saved (or through the MCP
trigger tools); the user's list, across all workspaces, comes from `GET /me/schedules`.

### Observability (`/observability`)

| Method | Route | Description |
|---|---|---|
| `GET` | `/observability/metrics` | General metrics (with workspace/period filters) |
| `GET` | `/observability/metrics/workflows` · `/metrics/executores` | Metrics per workflow / per executor |
| `GET` | `/observability/runs-by-day` | Historical series of runs per day |
| `GET` | `/observability/runs` · `/runs/{run_id}` | List of runs / detail of a run |

### Credentials (`/credentials`) · Portal (`/artifacts`) · Drive (`/drive`)

| Method | Route | Description |
|---|---|---|
| `GET` · `POST` · `DELETE` | `/credentials/`, `/credentials/types`, `/credentials/test`, `/credentials/{id}` | CRUD + connectivity test |
| `GET` | `/artifacts/portal/{workflow_hash}` | Data for a workflow's public portal |
| `GET` | `/artifacts/tiles/{workflow_hash}/{layer_key}/{z}/{x}/{y}.pbf` | MVT tiles |
| `GET` · `POST` · `DELETE` | `/drive/`, `/drive/upload`, `/drive/{file_id}/download`, `/drive/{file_id}` | List / upload / download / delete |

**Drive size ceiling.** `PlatformFileSettings.max_size_mb` (default 200 MB, editable via
`PUT /drive/settings`) applies at both ends of the presigned-URL upload: when the URL is requested,
against the **declared** size, and at confirmation, against the **actual** size measured in storage —
whoever sends more than they declared gets `413` on confirm. When the object over the ceiling is a
Drive upload not yet accepted, the rejection also deletes the object and the pending record, because
the bytes are already in storage and the periodic cleanup only removes the row. Run artifacts
(key under `artifacts/`) do not go through this ceiling — they never did —, and so they are still
accepted regardless of size.

### Workspaces (`/workspaces`)

| Method | Route | Description |
|---|---|---|
| `GET` · `POST` | `/workspaces/` | List / create |
| `PUT` · `DELETE` | `/workspaces/{id_hash}` | Update / move to trash (soft delete) |

The `DELETE` is soft: the row stays with `deleted_at`, the workflows are deactivated and their schedules stop; Drive files, however, are removed immediately. Restoring/discarding permanently are platform admin actions (`/admin/workspaces/{trash,restore,purge}`) — restore does **not** reactivate schedules.

### Execution, Webhook and Status

| Method | Route | Description |
|---|---|---|
| `POST` | `/webhook/execute/{id_hash}` | Triggers a run via an external webhook (authenticated by the WebhookTrigger credential) — asynchronous 202 |
| `GET` | `/nodes/` | Lists the available nodes and their schemas |
| `GET` | `/ping` | Healthcheck (`{"status": "ok"}`) |

### WebSocket

| Endpoint | Authentication | Description |
|---|---|---|
| `/ws/workflow/{run_id}` | JWT in the 1st frame | Real-time run events |
| `/ws/executores/{executor_id}` | **mTLS** certificate | Persistent executor connection |
| `/ws/telemetry` | `?token=<JWT>` | VM metrics (CPU, RAM, disk) |

### MCP Server (`/mcp`)

Exact route `https://<PUBLIC_HOST>/mcp` (no trailing slash), outside Swagger and the JWT:
it speaks **MCP** over HTTP and authenticates with a **personal access token** in the header
`Authorization: Bearer atl_pat_…`, created under Configurações (Settings) → Tokens de acesso (Access tokens).
It exposes tools for discovery and reading (workspaces, workflows, node catalog,
credentials and Drive), for data sources (search the catalog of pre-mapped WFS
layers, probe and register a new one), for building (validate, create,
update, activate and publish to the portal) and for execution (trigger with optional
waiting, follow, read the log, cancel and re-run), plus authoring guide resources
and ready-made prompts — all within the token's scope, never beyond what
the account could already reach.

URL, scopes, tools, limits, errors and connection snippets per client:
[docs/mcp.md](mcp.md). The source catalog (the seed in `catalogo/`, the
assistant's "catalog first" rule, per-endpoint verification):
[docs/sources.md](sources.md).

---

## Authentication

The system uses **JWT** with two tokens and brute-force protection.

| Token | Lifetime | Use |
|---|---|---|
| **Access Token** | 30 minutes (fixed) | Header `Authorization: Bearer <token>` |
| **Refresh Token** | 2 days | Only in `POST /auth/refresh` |

### Brute-force Protection

- **Per-IP rate limit**: 5 req/min on login (via slowapi).
- **Per-username lockout**: 5 failed attempts → 15-minute lockout (via Redis).
- The frontend shows the remaining lockout time and disables the form.

In the frontend (NextAuth), the access token lives in `session.user.access_token`.

---

## Workflow Engine

The executor processes DAG graphs in **topological order**, respecting the dependencies between nodes. Execution is restricted to the nodes **reachable from the triggers** (and their ancestors); isolated nodes and orphan edges are discarded before the run.

### Node Categories (63 in total)

| Category | Count | Examples (real registry names) |
|---|---|---|
| 🎯 **trigger** | 5 | `WebhookTrigger`, `ScheduleTrigger`, `FileTrigger`, `GeofenceTrigger`, `SubWorkflowInput` |
| 📥 **datasource** | 8 | `DatabaseQuery`, `DatabaseSpatialQuery`, `ReadGeoJSON`, `ReadShapefile`, `ReadGeoParquet`, `ReadCSVWithCoords`, `WFS`, `DataInput` |
| 🗺️ **spatial** | 21 | `Buffer`, `Clip`, `Dissolve`, `SpatialJoin`, `TransformCRS`, `ComputeArea`, `Simplify`, `ValidateGeometry`, `IntersectionNode`, `UnionNode`, `Heatmap`, … |
| ⚡ **action** | 9 | `HttpRequest`, `SetFields`, `AttributeFilter`, `AttributeJoin`, `OverlapPercentage`, `GeocodeNode`, `PythonScript`, `Sort`, `RemoveDuplicates` |
| 🔀 **control** | 7 | `Conditional`, `Switch`, `JinjaBranch`, `Merge`, `Loop`, `SubWorkflow`, `ChangeDetector` |
| 📤 **output** | 13 | `SaveGeoJSON`, `SaveToPostGIS`, `SaveToShapefile`, `SaveToGeoParquet`, `SaveToS3`, `SendEmail`, `SendWebhook`, `PublishMap`, `CartaImagem`, `Response`, `DataOutput`, `SubWorkflowOutput`, `SaveToPostgres` |

To create a node, see [docs/creating-nodes.md](creating-nodes.md).

### Result reuse (pinning)

There is no cache driven by an environment variable. Reuse is done by **pinning**: the output of a node marked with `cache: true` is serialized to MinIO (`pin-cache/{workspace}/…`) and reused in subsequent runs; the node's event comes with `cache_hit: true`. Large outputs still use **spill-to-disk** (Parquet) during the run to save memory.

---

## Scheduling

The **AsyncScheduler** replaces Celery Beat — it is a native `asyncio` loop inside the API process.

```mermaid
graph TD
    SCHED["AsyncScheduler<br/>(loop every 30s)"] -->|"reads active schedules"| DB[(PostgreSQL)]
    SCHED -->|"checks next_run_at"| CHECK{Time to trigger?}
    CHECK -->|Yes| EXEC["WorkflowService<br/>.start_analysis()"]
    CHECK -->|No| WAIT["Waits for the next cycle"]
    EXEC -->|"selects executor"| AGENT["Executor via WebSocket"]
    EXEC -->|"updates last_run_at<br/>and next_run_at"| DB

    style SCHED fill:#ff6f00,color:#fff
    style EXEC fill:#1a237e,color:#fff
```

| Strategy | Field | Example |
|---|---|---|
| **cron** | `cron_expression` | `0 8 * * 1-5` (Mon–Fri at 08:00) |
| **interval** | `interval` + `unit` | `30` + `minutes` |
| **rrule** | `rrule_expression` | `FREQ=WEEKLY;BYDAY=MO,WE,FR` (RFC 5545) |

The `next_run_at` and `last_run_at` fields are persisted in the `schedules` table on every trigger, so they survive restarts.

---

## Real-Time Events

The platform uses **Redis pub/sub** + **WebSocket** for event streaming:

```mermaid
sequenceDiagram
    participant Executor
    participant API
    participant Redis
    participant Browser

    Executor->>API: node_event (via WS mTLS /ws/executores/{id})
    API->>Redis: PUBLISH workflow:{run_id}:events {event}
    Redis->>API: Subscriber receives event
    API->>Browser: WS /ws/workflow/{run_id} → {event}
    Note over Browser: Canvas updates the nodes'<br/>status in real time
```

Each node event carries a `status` (`kind = lifecycle`); there are also the `stdout` and `debug` kinds, and a sentinel node event `__workflow_complete__` that signals the end of the workflow.

| `status` field | Meaning |
|---|---|
| `started` | Node started execution |
| `completed` | Node finished successfully (with `cache_hit: true` when it came from the pin) |
| `failed` | Node failed (carries `error`) |

The `/ws/workflow/{run_id}` endpoint requires a JWT in the **first frame**; the history (`workflow:{run_id}:history`) is replayed after subscribing.

---

## Multi-tenancy

Atlans isolates data by **workspaces**. Each workspace groups workflows, credentials, assigned executors and files.

- Each user can belong to multiple workspaces.
- Per-member roles: **viewer**, **editor**, **owner**.
- A default workspace is created automatically when the user registers.
- `default` executors (platform pool) are accessible to everyone.

---

## Monitoring

| Service | URL | Description |
|---|---|---|
| Swagger UI | `http://localhost:8000/docs` | Interactive API documentation |
| MinIO Console | `http://localhost:9001` | S3 object management |
| WS Telemetry | `/ws/telemetry?token=<JWT>` | Real-time CPU, memory and disk |

The API exposes aggregated run metrics (general, per workflow, per executor and a per-day time series) — see the `/observability` page in the web app and [docs/specs/metrics-history.md](specs/metrics-history.md).

---

## Makefile

| Command | Description |
|---|---|
| `make bootstrap` | Creates the `step-ca-data` volume, `secrets/` and `.env` with strong secrets |
| `make bootstrap-stepca` | Captures fingerprint + intermediate, raises step-ca's certificate lifetime and issues the `AGENTS_HOST` cert |
| `make up-dev` / `make up-prod` | Starts the stack in dev (hot reload) / prod (Traefik + TLS) |
| `make down` | Stops and removes the containers |
| `make logs` / `logs-dev` / `logs-prod` | Real-time logs (last 100 lines) |
| `make restart` / `restart-prod` | Restarts the dev / prod stack |
| `make build-dev` / `build-prod` | Builds the containers |
| `make smoke` | Checks API, Redis, MinIO (and step-ca/DNS-only in prod) |
| `make seed-admin` | Creates the initial admin (via `api-prod`; in dev use `docker compose exec api …`) |
| `make backup-stepca` | Backs up the internal CA (keeps the 14 most recent) |

---

## CI/CD — GitHub Actions

| Workflow | Trigger | Output |
|---|---|---|
| **CI** (`ci.yml`) | push / PR | secrets-scan, backend (`ruff` + `pytest`), backend and frontend without extensions (the core alone), dependency audit (informational), frontend (`lint` + `vitest` + `build`) and desktop (`typecheck` + `vitest` + lock) |
| **Desktop** (`desktop-windows.yml`) | tag `desktop/v*` | Windows installer (NSIS) in a GitHub Release |
| **Executor Docker** (`executor-docker.yml`) | tag `executor/v*` | Executor Docker image (`.tar.gz`) in a GitHub Release |

**Deploy:** it is up to each installation — update the code, bring up the images and run the migrations ([docs/operations.md](operations.md#update-the-installation)). Migrations do not run on their own: apply `alembic upgrade head` after a version that changes the schema.
