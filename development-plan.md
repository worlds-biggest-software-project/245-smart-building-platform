# Smart Building Platform — Phased Development Plan

> Project: 245-smart-building-platform · Created: 2026-05-29
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises the research in `README.md`, `research.md`, `features.md`, `standards.md`, and `data-model-suggestion-1.md` (the entity-centric normalised relational model). The relational model was chosen as the canonical implementation target because it (a) aligns directly with the Brick Schema, Haystack, and BACnet standards required by the project, (b) is the well-understood path for facility management software, and (c) supports the TimescaleDB partitioning required for telemetry at scale.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | Strong ecosystem for BACnet (`BAC0`), MQTT (`paho-mqtt`), OPC UA (`asyncua`), Modbus (`pymodbus`), Brick Schema (`brickschema`), LLM SDKs (`openai`, `anthropic`), and time-series ML (`scikit-learn`, `prophet`). Equally good for API and ingestion workloads. |
| API framework | FastAPI 0.115+ | Native async, auto-generated OpenAPI 3.1 spec (required per features.md), Pydantic v2 validation, WebSocket support for real-time telemetry streaming. Matches the API-first posture of Siemens Building X and Facilio. |
| ORM / DB toolkit | SQLAlchemy 2.0 + Alembic | Mature async ORM, supports PostgreSQL-specific features (JSONB, GIST EXCLUDE, recursive CTEs) needed by the data model. Alembic provides migration discipline. |
| Database — operational | PostgreSQL 16 | Required for GIST EXCLUDE constraints (space bookings), JSONB, recursive CTEs, and Row-Level Security. Available as managed service everywhere. |
| Database — telemetry | TimescaleDB 2.16 extension on PostgreSQL | Telemetry is the dominant data volume. Hypertables, continuous aggregates, compression, and retention policies are required (per data-model-suggestion-1). Avoids running a separate Influx/Prometheus stack. |
| Cache / pub-sub | Redis 7 | Real-time fan-out of telemetry to WebSocket subscribers, rate limiting, idempotency keys, session state, and the broker between ingestion workers and analytics. |
| Task queue | Celery 5.4 with Redis broker | Background work for protocol pollers, AI inference, alarm evaluation, periodic rollups, notifications. Beat scheduler covers cron-style polling. |
| MQTT broker | Eclipse Mosquitto 2.x | ISO/IEC 20922-compliant, TLS-ready, well-known, low ops overhead. Used both as a sensor ingress and an internal telemetry fan-out where it's a better fit than Redis. |
| Frontend | Next.js 15 (App Router) + React 19 + TypeScript | Server-rendered dashboards for portfolio/site/building views, client components for real-time charts. Matches the standard SaaS pattern and the team's existing experience implied by the project workspace. |
| UI components | shadcn/ui + Tailwind CSS 4 | Compose dashboards quickly without locking into a heavyweight design system; remains modifiable. |
| Charts | Apache ECharts via `echarts-for-react` | High-cardinality time-series, gauges, heatmaps, sankey diagrams (energy flow) — ECharts is significantly more capable than Recharts for this domain. |
| Auth | Authlib (server) + OIDC | Implements OAuth 2.0 / OpenID Connect 1.0 (RFC 6749) as required by standards.md. Supports local password, OIDC (Auth0, Okta, Keycloak), and SAML via downstream extension. |
| LLM provider | Anthropic Claude (Sonnet/Haiku) via official SDK, with provider-agnostic facade | Used for natural-language-to-API query translation, fault explanation, recommendation rationale. Facade allows OpenAI or local Ollama swap. |
| Vector store | pgvector extension | Stores embeddings for ontology terms, equipment descriptions, and historical fault patterns. Avoids a separate vector DB. |
| Validation / schemas | Pydantic v2 | Pairs with FastAPI, provides JSON-Schema export for OpenAPI. |
| Testing — backend | pytest, pytest-asyncio, pytest-httpx, testcontainers | testcontainers spins up real PostgreSQL+TimescaleDB, Redis, and Mosquitto for integration tests. |
| Testing — frontend | Vitest + React Testing Library + Playwright | Vitest for component tests, Playwright for E2E flows. |
| Linting / formatting | Ruff (lint + format) + mypy (strict) for Python; ESLint + Prettier + tsc for TS | Single-tool Python toolchain; standard TS toolchain. |
| Containerisation | Docker + Docker Compose for dev; production Helm chart | Self-hosted is the explicit deployment posture in README.md. Compose covers single-node demos; Helm covers multi-node production. |
| CI/CD | GitHub Actions | Matrix builds (Python, Node), testcontainers, image builds, Helm chart publish. |
| Package management | uv (Python), pnpm (Node) | Fast, deterministic. |
| Observability | OpenTelemetry SDK → OTLP collector → Prometheus + Grafana + Loki | Open-source, vendor-neutral telemetry. Matches the open-source ethos of the project. |
| Ontology library | `brickschema` (Brick), `phable` (Haystack v4 REST client/parser), `pydtdl` (DTDL v3 parsing) | First-class Python support exists for all three. Avoid hand-rolling parsers. |
| MCP server | `mcp` Python SDK (Anthropic) | Exposes building-data tools to LLM agents per the "MCP server option" called out in README.md (backlog) — wired in from Phase 8. |
| Standards targets | BACnet (ISO 16484-5 / ASHRAE 135-2024 incl. BACnet/SC), MQTT (ISO/IEC 20922), OPC UA (IEC 62541), Brick Schema, Project Haystack v4, DTDL v3, OpenAPI 3.1, OAuth 2.0 / OIDC, ISO 52120, OWASP API Top 10 | Drawn from `standards.md`. The platform implements each at the layer indicated in the plan below. |

### Project Structure

```
smart-building-platform/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── docker-compose.yml                 # full local dev stack
├── docker-compose.test.yml            # ephemeral test stack
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.frontend
├── helm/
│   └── smart-building-platform/       # Helm chart for production
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── publish-images.yml
│       └── publish-helm.yml
├── infra/
│   ├── mosquitto/                     # broker config + ACLs
│   ├── postgres/                      # init scripts (timescaledb + pgvector extensions)
│   └── grafana/                       # provisioning dashboards
├── apps/
│   ├── api/                           # FastAPI service
│   │   ├── pyproject.toml
│   │   ├── alembic.ini
│   │   ├── alembic/
│   │   │   └── versions/
│   │   ├── src/sbp_api/
│   │   │   ├── __init__.py
│   │   │   ├── main.py                # FastAPI app factory
│   │   │   ├── config.py              # Pydantic Settings
│   │   │   ├── db/
│   │   │   │   ├── base.py            # SQLAlchemy Declarative base
│   │   │   │   ├── session.py
│   │   │   │   ├── models/            # ORM models per domain
│   │   │   │   │   ├── identity.py
│   │   │   │   │   ├── spatial.py
│   │   │   │   │   ├── equipment.py
│   │   │   │   │   ├── telemetry.py
│   │   │   │   │   ├── energy.py
│   │   │   │   │   ├── alarms.py
│   │   │   │   │   ├── work_orders.py
│   │   │   │   │   ├── occupancy.py
│   │   │   │   │   ├── iaq.py
│   │   │   │   │   ├── tenants_exp.py
│   │   │   │   │   ├── data_sources.py
│   │   │   │   │   ├── ai.py
│   │   │   │   │   ├── audit.py
│   │   │   │   │   └── haystack_tags.py
│   │   │   │   ├── rls.py             # RLS policy helpers
│   │   │   │   └── seeds/             # reference data (brick types, haystack tags, units)
│   │   │   ├── schemas/               # Pydantic v2 request/response models
│   │   │   ├── routers/               # FastAPI routers per resource
│   │   │   │   ├── auth.py
│   │   │   │   ├── sites.py
│   │   │   │   ├── buildings.py
│   │   │   │   ├── floors.py
│   │   │   │   ├── zones.py
│   │   │   │   ├── spaces.py
│   │   │   │   ├── equipment.py
│   │   │   │   ├── points.py
│   │   │   │   ├── telemetry.py
│   │   │   │   ├── energy.py
│   │   │   │   ├── alarms.py
│   │   │   │   ├── work_orders.py
│   │   │   │   ├── occupancy.py
│   │   │   │   ├── bookings.py
│   │   │   │   ├── data_sources.py
│   │   │   │   ├── nl_query.py
│   │   │   │   ├── recommendations.py
│   │   │   │   ├── webhooks.py
│   │   │   │   └── ws.py              # WebSocket router
│   │   │   ├── services/              # business logic / orchestration
│   │   │   │   ├── alarms.py
│   │   │   │   ├── rollups.py
│   │   │   │   ├── energy.py
│   │   │   │   ├── recommendations.py
│   │   │   │   ├── nl_query.py
│   │   │   │   └── bookings.py
│   │   │   ├── security/
│   │   │   │   ├── auth.py            # OIDC + local
│   │   │   │   ├── rbac.py
│   │   │   │   └── audit.py
│   │   │   ├── observability/
│   │   │   │   ├── logging.py
│   │   │   │   ├── metrics.py
│   │   │   │   └── tracing.py
│   │   │   └── lifespan.py
│   │   └── tests/
│   │       ├── unit/
│   │       ├── integration/
│   │       └── e2e/
│   ├── worker/                        # Celery workers + Beat
│   │   ├── pyproject.toml
│   │   └── src/sbp_worker/
│   │       ├── celery_app.py
│   │       ├── tasks/
│   │       │   ├── ingest_bacnet.py
│   │       │   ├── ingest_mqtt.py
│   │       │   ├── ingest_opcua.py
│   │       │   ├── ingest_modbus.py
│   │       │   ├── ingest_rest.py
│   │       │   ├── rollups.py
│   │       │   ├── alarm_eval.py
│   │       │   ├── notifications.py
│   │       │   ├── ai_optimisation.py
│   │       │   ├── ai_fault_detection.py
│   │       │   ├── ai_energy_forecast.py
│   │       │   └── ai_space_insights.py
│   │       ├── connectors/            # protocol clients
│   │       │   ├── bacnet.py
│   │       │   ├── mqtt.py
│   │       │   ├── opcua.py
│   │       │   ├── modbus.py
│   │       │   └── rest.py
│   │       └── tests/
│   ├── mcp_server/                    # Phase 10 — MCP server
│   │   ├── pyproject.toml
│   │   └── src/sbp_mcp/
│   │       ├── server.py
│   │       └── tools/
│   └── frontend/                      # Next.js dashboard
│       ├── package.json
│       ├── next.config.ts
│       ├── tailwind.config.ts
│       ├── src/
│       │   ├── app/                   # App Router
│       │   │   ├── (auth)/
│       │   │   ├── (dashboard)/
│       │   │   │   ├── portfolio/
│       │   │   │   ├── sites/[siteId]/
│       │   │   │   ├── buildings/[buildingId]/
│       │   │   │   ├── floors/[floorId]/
│       │   │   │   ├── alarms/
│       │   │   │   ├── work-orders/
│       │   │   │   ├── energy/
│       │   │   │   ├── recommendations/
│       │   │   │   ├── nl-query/
│       │   │   │   └── settings/
│       │   │   └── api/               # BFF only when needed
│       │   ├── components/
│       │   │   ├── charts/
│       │   │   ├── alarms/
│       │   │   ├── layouts/
│       │   │   └── ui/                # shadcn primitives
│       │   ├── lib/
│       │   │   ├── api-client.ts      # auto-generated from OpenAPI
│       │   │   ├── ws-client.ts
│       │   │   └── auth.ts
│       │   └── styles/
│       └── tests/
├── packages/
│   ├── ontology/                      # shared Python lib for Brick/Haystack/DTDL
│   │   ├── pyproject.toml
│   │   └── src/sbp_ontology/
│   │       ├── brick.py
│   │       ├── haystack.py
│   │       ├── dtdl.py
│   │       └── mapping.py
│   ├── normaliser/                    # shared Python lib for unit conversion + point mapping
│   │   ├── pyproject.toml
│   │   └── src/sbp_normaliser/
│   ├── ai/                            # shared Python lib for LLM facade + ML models
│   │   ├── pyproject.toml
│   │   └── src/sbp_ai/
│   │       ├── llm.py                 # provider-neutral facade
│   │       ├── prompts/
│   │       ├── nl_to_sql.py
│   │       ├── fault_detection.py
│   │       ├── optimisation.py
│   │       └── forecasting.py
│   └── openapi-client-ts/             # generated TS client (CI step)
├── scripts/
│   ├── dev-up.sh
│   ├── seed-demo-data.py
│   └── generate-openapi-client.sh
└── docs/
    ├── architecture.md
    ├── ontology-mapping.md
    ├── connector-development.md
    └── deployment.md
```

---

## Phase 1: Foundation, Tooling, and Multi-Tenancy Skeleton

### Purpose

Establish the repository, container stack, CI, and the minimum viable database + API skeleton that everything else attaches to. After this phase, a developer can clone the repo, run `docker compose up`, hit a health-check endpoint, sign in as a seeded user, and see an empty (but tenant-scoped) site list. This phase exists to remove all ambiguity about wiring before any domain logic is written.

### Tasks

#### 1.1 — Monorepo skeleton, tooling, and CI

**What**: Create the monorepo directory layout, configure uv/pnpm workspaces, Ruff/mypy/ESLint/Prettier/tsc/Vitest/pytest, and a baseline GitHub Actions pipeline.

**Design**:
- `pyproject.toml` at repo root declares a uv workspace covering `apps/api`, `apps/worker`, `apps/mcp_server`, `packages/ontology`, `packages/normaliser`, `packages/ai`.
- `pnpm-workspace.yaml` covers `apps/frontend` and `packages/openapi-client-ts`.
- Single `ruff.toml` and `mypy.ini` at repo root; strict mode for both.
- `.editorconfig`, `.gitignore`, `.dockerignore`, `LICENSE` (Apache 2.0 placeholder — final decision deferred per README), `CONTRIBUTING.md` (copied template).
- GitHub Actions `ci.yml`:
  ```yaml
  jobs:
    python-lint-test:
      strategy: { matrix: { python: ["3.12"] } }
      services: { postgres: timescale/timescaledb:2.16.1-pg16, redis: redis:7, mosquitto: eclipse-mosquitto:2 }
      steps: [checkout, setup-uv, uv sync, ruff check, ruff format --check, mypy, pytest -m "not slow"]
    frontend-lint-test:
      steps: [checkout, setup-node, pnpm install, pnpm lint, pnpm typecheck, pnpm test, pnpm build]
    docker-build:
      needs: [python-lint-test, frontend-lint-test]
      steps: [build api/worker/frontend images]
  ```
- A `Makefile` exposing `make dev`, `make test`, `make lint`, `make migrate`, `make seed`.

**Testing**:
- Unit: `ruff check .` returns 0 on a clean checkout.
- Unit: `mypy` returns 0 on the empty `sbp_api` package.
- Integration: GitHub Actions workflow runs to completion on a no-op commit.
- Integration: `docker compose -f docker-compose.test.yml up -d` brings up Postgres/Redis/Mosquitto and `pg_isready` succeeds within 30s.

#### 1.2 — `docker-compose.yml` for local development

**What**: A single Compose file that brings up the full stack (Postgres+TimescaleDB+pgvector, Redis, Mosquitto, API, worker, beat, frontend, OTLP collector, Grafana, Loki, Prometheus) with hot reload.

**Design**:
- Services:
  - `postgres`: image `timescale/timescaledb-ha:pg16-latest`, env `POSTGRES_DB=sbp`, init script installing `timescaledb`, `pgvector`, `pg_trgm`, `btree_gist`.
  - `redis`: image `redis:7-alpine`.
  - `mosquitto`: image `eclipse-mosquitto:2`, config bind-mounted from `infra/mosquitto/`.
  - `api`: built from `Dockerfile.api`, command `uvicorn sbp_api.main:app --reload`, depends_on healthy postgres/redis.
  - `worker`: built from `Dockerfile.worker`, command `celery -A sbp_worker.celery_app worker -l info`.
  - `beat`: same image, command `celery -A sbp_worker.celery_app beat -l info`.
  - `frontend`: built from `Dockerfile.frontend`, command `pnpm dev --port 3000`.
  - `otel-collector`, `prometheus`, `loki`, `grafana` for observability (provisioned dashboards in `infra/grafana/`).
- Single `.env.example` containing all required vars (`DATABASE_URL`, `REDIS_URL`, `MQTT_URL`, `JWT_SECRET`, `OIDC_*`, `LLM_PROVIDER`, `ANTHROPIC_API_KEY`, etc).
- `scripts/dev-up.sh` wraps `docker compose up` with optional `--seed` flag invoking `scripts/seed-demo-data.py`.

**Testing**:
- Integration: `docker compose up -d` followed by `curl http://localhost:8000/healthz` returns `{"status":"ok"}` within 30s.
- Integration: `curl http://localhost:3000` returns HTTP 200 with the Next.js landing page.
- Integration: Grafana available at `http://localhost:3001`, default dashboard loads.

#### 1.3 — Database baseline + multi-tenancy + RLS

**What**: Alembic configuration, the first migration creating tenants/users/roles/permissions/role_permissions/user_roles tables, and PostgreSQL Row-Level Security helpers.

**Design**:
- Alembic initialised with async template, single migration `0001_baseline.py` containing the **Multi-Tenancy and Identity** SQL from data-model-suggestion-1, verbatim, plus:
  ```sql
  CREATE EXTENSION IF NOT EXISTS pgcrypto;
  CREATE EXTENSION IF NOT EXISTS pg_trgm;
  CREATE EXTENSION IF NOT EXISTS btree_gist;
  CREATE EXTENSION IF NOT EXISTS timescaledb;
  CREATE EXTENSION IF NOT EXISTS vector;
  ```
- `sbp_api.db.rls.set_tenant(tenant_id: UUID)` issues `SET LOCAL app.tenant_id = '<uuid>'` on the current connection. SQLAlchemy `before_cursor_execute` event applies this on every request from a `tenant_id` resolved during authentication.
- A reusable PostgreSQL function `current_tenant_id()` reads `app.tenant_id`. Every operational table (added in later phases) will receive a policy of the form:
  ```sql
  CREATE POLICY tenant_isolation ON <table>
    USING (tenant_id = current_tenant_id());
  ALTER TABLE <table> ENABLE ROW LEVEL SECURITY;
  ```
- `sbp_api.security.auth` issues JWTs whose payload includes `tenant_id`, `user_id`, and an `aud` of `sbp-api`.
- ORM models in `sbp_api.db.models.identity`:
  ```python
  class Tenant(Base):
      __tablename__ = "tenants"
      id: Mapped[UUID] = mapped_column(PGUUID(as_uuid=True), primary_key=True, server_default=text("gen_random_uuid()"))
      name: Mapped[str] = mapped_column(String(255))
      slug: Mapped[str] = mapped_column(String(100), unique=True)
      subscription_tier: Mapped[str] = mapped_column(String(50), default="standard")
      settings: Mapped[dict] = mapped_column(JSONB, default=dict)
      created_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True), server_default=func.now())
      updated_at: Mapped[datetime] = mapped_column(TIMESTAMP(timezone=True), server_default=func.now(), onupdate=func.now())
  ```
  (User, Role, Permission, RolePermission, UserRole modelled equivalently from the SQL.)

**Testing**:
- Unit: Pydantic `Settings` rejects missing `DATABASE_URL` with a clear error.
- Integration (testcontainers): `alembic upgrade head` creates all tables; introspecting `information_schema.tables` returns the expected 6.
- Integration: Inserting two tenants, two users (one per tenant); querying without setting `app.tenant_id` returns 0 rows from a future RLS-enabled table; setting `app.tenant_id` to tenant A returns only A's rows.
- Unit: JWT round-trip — sign with `tenant_id` claim, decode, assert payload matches.

#### 1.4 — Authentication: local password + OIDC

**What**: `/auth/login`, `/auth/refresh`, `/auth/logout`, `/auth/me` endpoints; OIDC callback flow via Authlib.

**Design**:
- Local-password login uses Argon2 (`argon2-cffi`).
- OIDC providers configured per tenant in `tenants.settings.oidc` (`{ "issuer": "...", "client_id": "...", "client_secret_env": "..." }`).
- Endpoint signatures:
  - `POST /auth/login` → `{ "email": str, "password": str, "tenant_slug": str }` → `{ "access_token", "refresh_token", "user": {...} }`
  - `POST /auth/refresh` → `{ "refresh_token": str }` → `{ "access_token", "refresh_token" }`
  - `GET /auth/oidc/{provider}/login?tenant_slug=...` → 302 redirect to IdP
  - `GET /auth/oidc/{provider}/callback?code=...&state=...` → exchanges code, creates/updates user, issues JWT
  - `GET /auth/me` → returns current user + tenant + roles
- Refresh tokens stored in Redis (`refresh:{jti}` → `{ user_id, tenant_id, expires_at }`), revocable.
- FastAPI dependency `current_user()` extracts JWT, sets `app.tenant_id` on the request-scoped DB session, returns the `User` ORM object.

**Testing**:
- Unit: Argon2 hash verification round-trip.
- Unit: JWT created at time T expires after configured TTL (mocked time).
- Integration: Local login with correct credentials returns 200 + tokens; wrong password returns 401.
- Integration (mocked IdP): OIDC callback with valid `code` exchanges to access_token, creates a user with `auth_provider='oidc'`.
- Integration: Hitting `/auth/me` without a token returns 401; with valid token returns the seeded user.

#### 1.5 — Observability skeleton

**What**: Structured logging, Prometheus metrics, and OpenTelemetry tracing for the API and worker.

**Design**:
- `structlog` configured to emit JSON in production, console in dev. All log lines include `tenant_id`, `user_id`, `request_id`.
- `prometheus-fastapi-instrumentator` exposes `/metrics`.
- OTel SDK auto-instruments FastAPI, SQLAlchemy, Celery, Redis. Exporter targets the local OTLP collector. Sampling: parent-based with `TRACES_SAMPLER_ARG=0.1` in prod, `1.0` in dev.
- Pre-provisioned Grafana dashboards: API RED (rate/errors/duration), Postgres connection pool, Celery queue depth.

**Testing**:
- Integration: `curl /metrics` returns Prometheus exposition format including `http_requests_total`.
- Integration: A request with `X-Request-Id: test-123` appears in logs with `request_id=test-123`.
- Integration: Trace appears in the collector when API → DB call is made (verified via the collector's debug exporter).

---

## Phase 2: Spatial Hierarchy and Reference Ontology

### Purpose

Build out the canonical spatial model (`portfolio → site → building → floor → zone → space`) and seed the reference ontology tables (equipment types, point types, units, Haystack tag definitions). After this phase, the platform has a queryable model of a building's physical and semantic structure, expressible via the REST API and viewable in the frontend's portfolio tree. This is the structural foundation for every later phase.

### Tasks

#### 2.1 — Spatial schema migration

**What**: Migration `0002_spatial.py` creating `portfolios`, `sites`, `buildings`, `floors`, `zones`, `spaces` per data-model-suggestion-1 §Spatial Hierarchy, plus RLS policies.

**Design**: Copy the SQL verbatim. Apply tenant RLS policy to each table. Add ORM models in `sbp_api.db.models.spatial`. Each model exposes a `to_brick_iri()` method that returns a stable IRI string of the form `https://example.com/<tenant_slug>/<entity_type>/<id>` for downstream RDF export.

**Testing**:
- Integration: Insert a portfolio → site → building → floor → zone → space chain; verify all FKs hold.
- Integration: Insert a site with invalid `country_code` (`'USA'`) → CHECK constraint fails with the expected message.
- Integration: With RLS enabled and `app.tenant_id` unset, SELECT returns 0; with the right tenant, returns the inserted row.

#### 2.2 — Reference data: equipment types, point types, units

**What**: Migration `0003_reference.py` creating `equipment_types`, `point_types`, `units_of_measure`, plus a deterministic seed loader.

**Design**:
- Seed data lives in `apps/api/src/sbp_api/db/seeds/`:
  - `equipment_types.yaml` — at least 40 common HVAC/lighting/electrical/access types with `brick_class_uri` and `haystack_marker` populated (AHU, VAV, CAV, FCU, Chiller, Boiler, Cooling_Tower, Pump, Fan, Lighting_Panel, Lighting_Fixture, Occupancy_Sensor, Card_Reader, …).
  - `point_types.yaml` — at least 80 common points (Zone_Air_Temperature_Sensor, Zone_Air_Temperature_Setpoint, Discharge_Air_Flow_Sensor, CO2_Sensor, Occupancy_Count_Sensor, Electric_Energy_Sensor, …).
  - `units_of_measure.yaml` — QUDT-aligned subset (degC, degF, K, Pa, kPa, CFM, L/s, m³/h, ppm, %RH, lux, W, kW, kWh, A, V, Hz, kg, kgCO2eq).
- Loader command `python -m sbp_api seed reference --upsert` iterates each YAML, upserts by unique key.
- Brick IRIs validated against the `brickschema` package's loaded ontology graph at seed time; failures log warnings.

**Testing**:
- Unit: YAML parsing produces the expected count of records.
- Integration: Running the seed twice is idempotent (same row count).
- Unit: Each seeded `brick_class_uri` is reachable in the Brick graph (`brickschema.Graph().load_file('Brick.ttl')` then SPARQL ASK).
- Unit: Unit conversion factors round-trip correctly (e.g., `degF→K` then `K→degF` returns the original value within 1e-9).

#### 2.3 — Spatial REST API

**What**: CRUD endpoints for portfolios, sites, buildings, floors, zones, spaces; nested under `/api/v1`.

**Design**:
- Resource shape (example for site):
  ```
  POST   /api/v1/sites                       → create
  GET    /api/v1/sites                       → list (cursor-paginated)
  GET    /api/v1/sites/{site_id}             → retrieve
  PATCH  /api/v1/sites/{site_id}             → partial update
  DELETE /api/v1/sites/{site_id}             → soft-delete (sets `deleted_at`)
  GET    /api/v1/sites/{site_id}/buildings   → nested list
  ```
- Pagination: `?limit=50&cursor=<opaque>`; cursor encodes `(created_at, id)`. Response includes `next_cursor`.
- Filtering: `?country_code=US&q=campus` (q does ILIKE on `name`).
- Response envelope follows JSON:API-lite: `{ "data": [...], "meta": { "next_cursor": "..." } }`.
- All write endpoints require `permission(resource_type='site', action='write')`.
- OpenAPI 3.1 generated by FastAPI; CI publishes `openapi.json` and regenerates `packages/openapi-client-ts`.

**Testing**:
- Unit: `POST /api/v1/sites` with missing `country_code` → 422 with field path.
- Integration: Create site → list → retrieve → patch name → list again shows new name.
- Integration: User from tenant A cannot retrieve a site owned by tenant B (404, not 403, to avoid disclosure).
- Integration: Pagination with 120 sites, `limit=50` returns 3 pages; cursors are stable across requests.
- E2E (Playwright): Create a site through the API, log into the frontend, see it in the sidebar tree.

#### 2.4 — Frontend: portfolio tree and entity views

**What**: Next.js routes for `/portfolio`, `/sites/[siteId]`, `/buildings/[buildingId]`, `/floors/[floorId]` with read-only views populated from the API.

**Design**:
- Layout: persistent left sidebar (`PortfolioTree` component) showing portfolios → sites → buildings → floors as a collapsible tree (`@radix-ui/react-collapsible`).
- Pages use React Server Components for initial load; client components only for the interactive tree.
- `lib/api-client.ts` is generated from `openapi.json` via `openapi-typescript-codegen` (`scripts/generate-openapi-client.sh`, invoked in CI and pre-commit).
- An `<EntityHeader>` shows breadcrumbs (Portfolio › Site › Building › Floor) and key facts (area, country, year built).

**Testing**:
- Unit (Vitest): `<PortfolioTree>` with mocked data renders all nodes and collapses on click.
- E2E (Playwright): Sign in, click site in tree, URL updates, breadcrumb shows expected path, header shows expected name.

---

## Phase 3: Equipment, Points, and Telemetry Storage

### Purpose

Add the equipment and point models — the "things being measured and controlled" — and the TimescaleDB-backed telemetry store. After this phase, telemetry can be written to the database (via a direct API for testing) and queried back with time-range and aggregation parameters. The platform now has a complete data plane for the core value loop, ready for protocol ingestion in Phase 4.

### Tasks

#### 3.1 — Equipment and points schema

**What**: Migration `0004_equipment_points.py` creating `equipment` and `points` tables per data-model-suggestion-1 §Equipment and Points, plus RLS.

**Design**: Verbatim SQL from the data model suggestion. ORM models in `sbp_api.db.models.equipment`. The `equipment` model exposes `descendants()` using a recursive CTE for parent→child traversal:
```python
async def descendants(self, session: AsyncSession) -> list["Equipment"]:
    cte = (
        select(Equipment).where(Equipment.id == self.id)
        .cte(name="equip_tree", recursive=True)
    )
    cte = cte.union_all(
        select(Equipment).join(cte, Equipment.parent_equipment_id == cte.c.id)
    )
    return (await session.execute(select(Equipment).join(cte, Equipment.id == cte.c.id))).scalars().all()
```

**Testing**:
- Integration: Create AHU + 3 VAVs as children + 6 grand-child sensors; `descendants()` returns 10 rows in DFS order.
- Integration: Deleting an AHU with FK-restricted children returns 409 (handled at API layer).
- Unit: Insert point with `bacnet_object_instance` but no `bacnet_object_type` → application-level validation fails.

#### 3.2 — Telemetry hypertable + rollups

**What**: Migration `0005_telemetry.py` creating `telemetry`, `telemetry_hourly`, `telemetry_daily` as TimescaleDB hypertables and continuous aggregates.

**Design**:
- Migration runs:
  ```sql
  CREATE TABLE telemetry (...);  -- per data-model-suggestion-1
  SELECT create_hypertable('telemetry', 'time', chunk_time_interval => INTERVAL '1 day');
  ALTER TABLE telemetry SET (timescaledb.compress, timescaledb.compress_segmentby = 'point_id');
  SELECT add_compression_policy('telemetry', INTERVAL '7 days');
  SELECT add_retention_policy('telemetry', INTERVAL '365 days');

  CREATE MATERIALIZED VIEW telemetry_hourly_cagg
    WITH (timescaledb.continuous) AS
    SELECT time_bucket('1 hour', time) AS bucket, point_id,
           MIN(value_numeric) AS min_value,
           MAX(value_numeric) AS max_value,
           AVG(value_numeric) AS avg_value,
           COUNT(*) AS sample_count
    FROM telemetry GROUP BY bucket, point_id;
  SELECT add_continuous_aggregate_policy('telemetry_hourly_cagg',
    start_offset => INTERVAL '3 days', end_offset => INTERVAL '1 hour', schedule_interval => INTERVAL '15 minutes');
  -- Same pattern for telemetry_daily_cagg
  ```
- The denormalised `points.current_value` is updated on each insert via a trigger:
  ```sql
  CREATE FUNCTION update_point_current_value() RETURNS TRIGGER AS $$
  BEGIN
    UPDATE points SET current_value = NEW.value_numeric,
                      current_value_str = NEW.value_string,
                      current_quality = NEW.quality,
                      current_timestamp = NEW.time
    WHERE id = NEW.point_id AND (current_timestamp IS NULL OR NEW.time > current_timestamp);
    RETURN NEW;
  END; $$ LANGUAGE plpgsql;
  CREATE TRIGGER trg_telemetry_current_value AFTER INSERT ON telemetry
    FOR EACH ROW EXECUTE FUNCTION update_point_current_value();
  ```

**Testing**:
- Integration: Insert 100 telemetry rows for 1 point; `points.current_value` matches the latest by `time`.
- Integration: Insert an out-of-order row (older timestamp); `current_value` is unchanged.
- Integration: Run `CALL refresh_continuous_aggregate('telemetry_hourly_cagg', NULL, NULL)`; querying the CAGG returns the expected aggregations.
- Integration: Verify `add_retention_policy` is registered (`SELECT * FROM timescaledb_information.jobs WHERE proc_name='policy_retention'` returns 1 row).

#### 3.3 — Equipment, point, and telemetry REST API

**What**: Endpoints to manage equipment/points and to ingest/query telemetry.

**Design**:
- `POST /api/v1/buildings/{building_id}/equipment` — create equipment.
- `POST /api/v1/equipment/{equipment_id}/points` — create points.
- `POST /api/v1/telemetry:ingest` — bulk insert (NDJSON or JSON array body, up to 10k rows):
  ```json
  { "items": [ { "point_id": "<uuid>", "time": "2026-05-29T10:00:00Z", "value_numeric": 22.5, "quality": "good" } ] }
  ```
  Response: `{ "accepted": 9998, "rejected": [{"index": 17, "reason": "unknown point_id"}, ...] }`.
- `GET /api/v1/telemetry?point_id=...&from=...&to=...&aggregation=raw|hour|day&limit=...` — query.
- Bulk insert uses `asyncpg.copy_records_to_table` for throughput; expected ≥10k rows/sec/worker.
- An `Idempotency-Key` header is honoured on ingest (cached in Redis 24h).

**Testing**:
- Integration: Bulk insert 10k rows → `accepted=10000`.
- Integration: Replay same bulk insert with same `Idempotency-Key` → returns cached response, no duplicates in DB.
- Integration: Query with `aggregation=hour` returns rows from the CAGG; `aggregation=raw` returns rows from `telemetry`.
- Integration: Query with `to` before `from` returns 422.

#### 3.4 — WebSocket: real-time telemetry stream

**What**: `WS /api/v1/ws/telemetry` — subscribe to live point updates.

**Design**:
- Client sends `{"action":"subscribe", "point_ids":["..."]}` after connecting (with `Authorization: Bearer <jwt>` in connect headers).
- Server subscribes the websocket to Redis Pub/Sub channel `telemetry:<point_id>`.
- Telemetry ingest publishes to Redis after writing to TimescaleDB:
  ```python
  await redis.publish(f"telemetry:{point_id}", json.dumps({"time": ..., "value": ...}))
  ```
- Server enforces RLS at subscribe time: rejects point_ids not owned by the connecting user's tenant.
- Heartbeat: server pings every 30s; client disconnects on 90s silence.

**Testing**:
- Integration (real Redis): Client subscribes to 5 points, ingest publishes to 3 → client receives exactly 3 messages.
- Integration: Cross-tenant subscription attempt rejected with 4403 close code.
- Integration: Heartbeat interval observed via mock clock.

---

## Phase 4: Multi-Protocol Ingestion (BACnet, MQTT, OPC UA, Modbus, REST)

### Purpose

Implement the protocol connectors and the data-source/point mapping infrastructure that let a real building's BMS, IoT gateway, or cloud API feed the platform. After this phase, configuring a data source in the API causes the worker to begin polling/subscribing and writing telemetry that lights up the dashboards built in Phase 3. This is the core differentiator from "yet another dashboard tool".

### Tasks

#### 4.1 — Data source schema and configuration API

**What**: Migration `0006_data_sources.py` creating `data_sources` and `point_data_source_map`; CRUD API.

**Design**: Verbatim SQL from data-model-suggestion-1 §Data Sources. ORM models. API:
- `POST /api/v1/buildings/{id}/data-sources` — body shape varies by `protocol`; validated by a discriminated Pydantic union:
  ```python
  class BacnetIPConfig(BaseModel):
      protocol: Literal["bacnet_ip"]
      ip: IPvAnyAddress
      port: int = 47808
      network: int = 0
  class MQTTConfig(BaseModel):
      protocol: Literal["mqtt"]
      broker: str
      port: int = 8883
      tls: bool = True
      username: str | None = None
      password_env: str | None = None
      topic_prefix: str
  # OPC UA, Modbus TCP, REST API equivalents
  DataSourceConfig = Annotated[Union[BacnetIPConfig, MQTTConfig, OPCUAConfig, ModbusTCPConfig, RestAPIConfig], Field(discriminator="protocol")]
  ```
- `POST /api/v1/data-sources/{id}/points` — bind a point to a source with `source_address` (e.g., BACnet `analog-input,12`, MQTT topic, OPC UA NodeId).
- `POST /api/v1/data-sources/{id}:test` — performs a connection check and returns `{status, details}` without persisting telemetry.

**Testing**:
- Unit: Discriminator routes the right Pydantic class for each protocol.
- Integration: `:test` against a mock BACnet device returns `status=ok`; against an unreachable IP returns `status=error` with message.
- Integration: RLS prevents cross-tenant data source visibility.

#### 4.2 — BACnet connector (incl. BACnet/SC)

**What**: Celery task `ingest_bacnet` that discovers, polls, and subscribes (COV) to BACnet devices.

**Design**:
- Uses `BAC0` for BACnet/IP and the BAC0 BACnet/SC mode (ASHRAE 135-2024) for secured deployments; certificate paths configured via `data_sources.connection_config.tls`.
- Discovery: `BAC0.lite.whois()` populates a candidate equipment list returned via `POST /api/v1/data-sources/{id}:discover`. Operator can review and bind.
- Polling: each mapped point is polled at its `polling_interval_sec`. COV subscription used where the device supports it.
- Each reading is written via the worker's internal `telemetry_writer` (which batches and `COPY`s every 250ms or 500 rows).
- Errors classified: `transient` (retry with exp backoff up to 1h), `permanent` (mark data_source.status=error, alert).

**Testing**:
- Unit (mocked BAC0): Polling produces `TelemetryRecord` objects with correct `point_id` and `value_numeric`.
- Integration (BAC0 simulator): Configure a BAC0 mock device exposing 5 analog-input objects; data source `:discover` finds all 5; binding 3 of them produces telemetry within 60s.
- Unit: Permanent error after 5 failed cycles marks `data_source.status='error'` and emits `bacnet_connector_error` log.

#### 4.3 — MQTT connector

**What**: Celery task `ingest_mqtt` that maintains a persistent MQTT subscription per data source.

**Design**:
- `paho-mqtt` async client; auto-reconnect with backoff.
- Topic→point mapping resolved via `point_data_source_map.source_address` (topic glob like `building/+/ahu/+/temp`).
- A configurable JSON path extracts the value: `connection_config.value_jsonpath` (default `$.value`).
- TLS-mandatory for ISO/IEC 20922 deployments; certificate auth supported.
- QoS 1 minimum; deduplicated by `(topic, message_id, timestamp)` in Redis with 5-minute TTL.

**Testing**:
- Integration (testcontainers Mosquitto): Publish 100 messages to `building/A/ahu/1/temp`; worker writes 100 telemetry rows in <5s.
- Integration: Disconnect broker, publish 50 messages while disconnected, reconnect → all 50 received (Mosquitto persistence on).
- Unit: Malformed JSON message logged and dropped; metric `mqtt_malformed_total` incremented.

#### 4.4 — OPC UA connector

**What**: Celery task `ingest_opcua` using `asyncua`.

**Design**:
- Supports client-server (subscribe to monitored items) and Pub/Sub patterns (IEC 62541 Part 14) where the server exposes them.
- NodeId mapping stored in `point_data_source_map.source_address` (e.g., `ns=2;s=AHU1.SupplyAirTemp`).
- Security: prefers `SignAndEncrypt` with `Basic256Sha256`; falls back to `None` only if explicitly enabled.

**Testing**:
- Integration (open62541 docker server): Create monitored items for 3 nodes; verify telemetry within 2 sample periods.
- Unit: Security policy `None` requires explicit `allow_unencrypted: true` in config; otherwise rejected.

#### 4.5 — Modbus TCP connector

**What**: Celery task `ingest_modbus` using `pymodbus` async client.

**Design**:
- Register-based polling. `source_address` format: `coil:100`, `holding:42`, `input:7`, `discrete:3`.
- Scaling expression on `point_data_source_map.transform_expr` (default `value`): evaluated with `simpleeval` (safe AST eval, no builtins) — e.g., `value * 0.1 - 273.15` to convert deciKelvin to Celsius.
- Batching: combines contiguous register reads into a single `read_holding_registers(start, count)`.

**Testing**:
- Integration (pymodbus simulator): Poll 10 holding registers; receive 10 telemetry rows.
- Unit: `transform_expr` `value * 0.1 - 273.15` applied to 2982 returns 25.05.
- Unit: Unsafe expression (`__import__('os').system('rm -rf')`) rejected by `simpleeval`.

#### 4.6 — REST API / cloud-to-cloud connector

**What**: Celery task `ingest_rest` for polling third-party vendor APIs (e.g., a Kaiterra IAQ feed).

**Design**:
- Configurable per source:
  ```python
  class RestAPIConfig(BaseModel):
      protocol: Literal["rest_api"]
      base_url: HttpUrl
      auth_type: Literal["none", "api_key", "bearer", "oauth2"]
      auth_config: dict
      poll_endpoint: str
      poll_interval_sec: int = 60
      response_jsonpath: str = "$.data[*]"     # iterates records
      time_field: str = "timestamp"
      value_field: str = "value"
      external_id_field: str = "sensor_id"      # mapped to point.source_address
  ```
- Rate-limiting honoured via `Retry-After` header.
- Pagination: cursor follow loop; max 10 pages per cycle.

**Testing**:
- Integration (pytest-httpx): Mock 3-page response with 50 readings each; worker writes 150 telemetry rows.
- Integration: 429 with `Retry-After: 5` triggers a delayed retry (verified via mocked sleep).
- Unit: OAuth 2.0 client-credentials flow obtains a token, caches it until expiry minus 60s.

#### 4.7 — Connector supervisor + health UI

**What**: `data_sources.status` lifecycle and a Settings → Data Sources page showing per-connector health.

**Design**:
- A supervisor task (every 30s) inspects last-seen timestamps and updates `status` to `connected` (seen within 2× poll interval), `error` (5 consecutive failures), or `disabled` (operator-disabled).
- Frontend page lists data sources, badges by status, shows last error, exposes "Test connection" and "Discover" buttons.

**Testing**:
- Integration: Disable a source via API → supervisor sets status to `disabled` within 30s.
- E2E (Playwright): Configure an MQTT source via the UI against testcontainers Mosquitto, see status transition `configured → connected` within 30s.

---

## Phase 5: Normalisation, Ontology Mapping, and Semantic Queries

### Purpose

Translate raw points and equipment into a Brick Schema / Haystack normalised view so the platform can answer ontology-aware queries ("all CO₂ sensors in meeting rooms on floor 3") regardless of underlying naming conventions. After this phase, the platform exposes a stable, semantically meaningful API on top of heterogeneous BMS data — matching Mapped's positioning, but built-in.

### Tasks

#### 5.1 — Brick Schema graph store

**What**: An in-process Brick graph derived from the operational tables, exposed via a SPARQL endpoint.

**Design**:
- The `packages/ontology/sbp_ontology/brick.py` module builds an `rdflib.Graph` lazily from the DB:
  - Each `site` → `brick:Site`; `building` → `brick:Building`; `floor` → `brick:Floor`; `zone` → `brick:HVAC_Zone`/`Lighting_Zone`/etc. based on `zone_type`; `space` → `brick:Room`.
  - Each `equipment` → instance of its `equipment_types.brick_class_uri`.
  - Each `point` → instance of its `point_types.brick_class_uri`; linked to its equipment via `brick:isPointOf`.
  - Containment: `brick:hasPart` between equipment and parent equipment, `brick:hasLocation` between equipment and zone/space.
- Cached in Redis (`brick:graph:{tenant_id}:{version}`) and invalidated when the underlying tables change (`pg_notify('brick_invalidate', tenant_id)`).
- SPARQL endpoint: `POST /api/v1/semantic/sparql` body `{"query": "..."}` returns JSON results.

**Testing**:
- Unit: Building a graph from a known fixture produces the expected number of triples.
- Integration: SPARQL `SELECT ?p WHERE { ?p a brick:CO2_Level_Sensor . }` returns the IDs of all CO₂ points.
- Integration: Insert a new point → `pg_notify` triggers cache invalidation; next query reflects the new point.

#### 5.2 — Haystack v4 REST endpoint

**What**: Implement the Haystack `read`, `hisRead`, `nav` ops at `/api/v1/haystack/*`.

**Design**:
- Uses `phable` to encode responses in Zinc and JSON.
- `read` filter (`equip and ahu`) translated to SQL via a small filter parser: tokens map to existence checks on `entity_tags` (joined with `haystack_tag_definitions`).
- `hisRead` returns telemetry from the appropriate aggregation level based on the time range (`raw` < 24h, `hour` < 30d, `day` otherwise).
- Authentication: same JWT bearer as the main API; Haystack `auth` op forwards to the OIDC/local flow.

**Testing**:
- Unit: Filter parser handles `equip and ahu`, `point and temp and not setpoint`, `siteRef==@123`.
- Integration: `read` for `co2 and sensor` returns the seeded CO₂ points in Zinc.
- Integration: `hisRead` for the last 24h returns raw values; for 60d returns daily aggregates.

#### 5.3 — DTDL export

**What**: `GET /api/v1/buildings/{id}/dtdl` exports the building as an array of DTDL v3 interfaces and a twin instance graph.

**Design**:
- Uses `pydtdl` for serialisation.
- Mapping: each `equipment_types` row becomes a DTDL interface (`@id` from `dtdl_model_id` or generated `dtmi:com:sbp:<type>;1`). Each equipment instance becomes a Digital Twin with telemetry channels from its points.
- Output is valid against the WillowTwin DTDL building ontology.

**Testing**:
- Unit: Exported interfaces validate against the DTDL JSON Schema.
- Integration: Round-trip — export a building, parse the JSON, count interfaces equals distinct equipment types.

#### 5.4 — Unit normalisation service

**What**: A shared `packages/normaliser/sbp_normaliser` library that converts a value+unit to a canonical SI unit and back.

**Design**:
- Conversion table seeded from `units_of_measure` (factor + offset).
- API:
  ```python
  def to_si(value: float, unit_symbol: str) -> tuple[float, str]: ...
  def from_si(value: float, target_symbol: str) -> float: ...
  def convert(value: float, from_unit: str, to_unit: str) -> float: ...
  ```
- Used by ingestion (to normalise incoming readings) and by API responses (for user-preferred display unit per `users.settings.units`).

**Testing**:
- Unit: 100°F → SI (K) → 100°F is exact within 1e-9.
- Unit: 1 kWh → SI (J) returns 3,600,000.
- Unit: Unknown unit raises `UnknownUnitError`.

---

## Phase 6: Alarms, Notifications, and Work Orders

### Purpose

Turn data into action. This phase implements the operational backbone of any building platform: configurable alarms, multi-channel notifications, and a work-order system. After this phase, an out-of-bound CO₂ reading triggers an alarm, escalates through configured channels, and (optionally) auto-creates a work order assigned to a maintenance technician.

### Tasks

#### 6.1 — Alarms schema and evaluation engine

**What**: Migration `0007_alarms.py` creating alarm tables; a Celery worker that evaluates alarm definitions against new telemetry.

**Design**:
- Verbatim SQL from data-model-suggestion-1 §Alarms.
- Evaluation triggered by a Redis stream `telemetry:eval` populated alongside Pub/Sub in Phase 3.3.
- Engine matches each incoming reading against alarm definitions whose `condition_config` references the point type. Conditions:
  ```python
  class ThresholdCondition(BaseModel):
      type: Literal["threshold"]
      operator: Literal[">", ">=", "<", "<=", "==", "!="]
      value: float
      duration_seconds: int = 0           # condition must hold continuously
  class RateOfChangeCondition(BaseModel):
      type: Literal["rate_of_change"]
      delta: float
      window_seconds: int
  class OfflineCondition(BaseModel):
      type: Literal["offline"]
      max_silence_seconds: int
  class PatternCondition(BaseModel):
      type: Literal["pattern"]
      sequence: list[dict]                # ordered conditions
  ```
- State machine for an `alarm` row: `active → acknowledged → resolved`; `active → suppressed → active`.
- Sliding-window state per (definition, point) cached in Redis to avoid full table scans.

**Testing**:
- Unit: Threshold condition `> 800 ppm for 15 min` fires only after sustained breach (mocked sliding window).
- Integration: Seed an alarm def, ingest 20 telemetry points crossing the threshold, alarm row created with `state=active`.
- Integration: Ingest a recovery reading → alarm transitions to `resolved` (auto-resolve on clearing).
- Integration: Acknowledge via API → `acknowledged_by` and `acknowledged_at` set; audit log row written.

#### 6.2 — Notification dispatch

**What**: Multi-channel notification delivery (email, SMS, webhook, Slack, Teams, push).

**Design**:
- `notification_channels` table from data-model-suggestion-1.
- Adapters in `apps/worker/src/sbp_worker/notifications/`:
  - `email.py` (SMTP, `aiosmtplib`)
  - `sms.py` (Twilio + AWS SNS adapters)
  - `webhook.py` (HTTPS POST with HMAC-SHA256 signature in `X-SBP-Signature`)
  - `slack.py` (Incoming Webhook + Slack Web API)
  - `teams.py` (Adaptive Card to Incoming Webhook)
  - `push.py` (Web Push via VAPID; APNs/FCM stubs)
- Escalation: when an alarm fires, the engine schedules per-level dispatches based on `alarm_escalation_rules.delay_minutes`; cancels remaining schedules on ack/resolve.
- Templates rendered with Jinja2 from `apps/worker/src/sbp_worker/templates/`.

**Testing**:
- Integration (MailHog testcontainer): Trigger an email channel → MailHog receives a message with the expected subject and body.
- Integration: Trigger a webhook → mock server verifies HMAC signature.
- Unit: Escalation level 2 schedules a job for `triggered_at + delay_minutes`.
- Integration: Acknowledging the alarm before level 2 fires cancels the queued job.

#### 6.3 — Work orders

**What**: Migration `0008_work_orders.py` and CRUD/workflow API for work orders.

**Design**:
- Verbatim SQL from data-model-suggestion-1 §Maintenance.
- API:
  - `POST /api/v1/work-orders` (manual create)
  - `POST /api/v1/work-orders/from-alarm/{alarm_id}` (auto on alarm if `alarm_definition.auto_ticket=true`)
  - `PATCH /api/v1/work-orders/{id}` (status, assignment, notes)
  - `POST /api/v1/work-orders/{id}/comments`
- State transitions enforced server-side: `open → assigned → in_progress → completed`; `cancelled` from any non-terminal state; `on_hold` ↔ `in_progress`.
- iCal feed `GET /api/v1/work-orders/calendar.ics?token=...` for technician calendars.

**Testing**:
- Integration: `auto_ticket=true` alarm fires → exactly one work order created with `alarm_id` populated.
- Integration: Invalid transition (`completed → open`) returns 409.
- Integration: Calendar feed contains all open work orders for the requesting technician.

#### 6.4 — Frontend: alarms console + work-order board

**What**: Pages `/alarms` (filterable list with bulk ack) and `/work-orders` (Kanban by status).

**Design**:
- Alarms page: server-rendered initial list, WebSocket subscription `WS /api/v1/ws/alarms` for live updates. Filters: severity, state, building, time range, category. Bulk select → "Acknowledge" / "Add note".
- Work-order Kanban with drag-and-drop (`@dnd-kit/core`); drop triggers PATCH; optimistic UI with rollback on failure.

**Testing**:
- E2E (Playwright): Trigger alarm via API, observe it appear in the UI within 5s, click "Acknowledge", confirm row updates.
- E2E: Drag a work order from "Assigned" to "In Progress", reload, status persists.
- Unit (Vitest): Optimistic UI rolls back when the PATCH returns 409.

---

## Phase 7: Energy, Sustainability, and Carbon Forecasting

### Purpose

Add the energy-management plane required for Scope 1/2 reporting (per features.md should-have) and the carbon forecasting that's missing in most competitors. After this phase, building operators can see consumption, demand, cost, and emissions; sustainability teams can run scenario models for proposed operational changes.

### Tasks

#### 7.1 — Energy meters and readings

**What**: Migration `0009_energy.py` (energy_meters, energy_readings, carbon_factors).

**Design**: Verbatim SQL from data-model-suggestion-1 §Energy. Seed `carbon_factors` from IEA 2025 dataset and EPA eGRID (CSV under `apps/api/src/sbp_api/db/seeds/carbon_factors/`) — at least 50 country/region rows.

**Testing**:
- Integration: Insert a meter, ingest 24h of hourly readings → daily consumption query returns the expected sum.
- Unit: Carbon factor lookup for `(country='US', region='CAMX', energy_type='electricity', date='2026-01-01')` returns the correct row.

#### 7.2 — Energy + emissions API

**What**: REST endpoints for energy KPIs and Scope 1/2 emissions.

**Design**:
- `GET /api/v1/buildings/{id}/energy?from=...&to=...&granularity=hour|day|month` →
  ```json
  {
    "totals": { "kwh": 12345.6, "cost": 1879.30, "currency": "USD" },
    "by_meter": [ { "meter_id": "...", "type": "electricity", "kwh": ... } ],
    "series": [ { "time": "...", "kwh": ... } ]
  }
  ```
- `GET /api/v1/buildings/{id}/emissions?from=...&to=...&scope=1|2|both` →
  ```json
  { "scope1_kgco2e": ..., "scope2_kgco2e": ..., "by_source": [...] }
  ```
- ISO 52120 BACS class displayed at site/building level: `GET /api/v1/sites/{id}/bacs-class` returns the stored class plus an "indicative compliance" computed from observed control patterns.

**Testing**:
- Integration: Energy query aggregates correctly at each granularity.
- Integration: Emissions calculation uses the temporally correct carbon factor (factor valid from 2024-01-01 to 2024-12-31 used for readings on 2024-06-15; new factor from 2025-01-01 used for readings on 2025-06-15).
- Unit: Currency-formatted output respects the user's locale preference.

#### 7.3 — Carbon and energy forecasting

**What**: A forecasting service producing 24h–365d energy and CO₂ projections, including scenario "what-if" runs.

**Design**:
- `packages/ai/src/sbp_ai/forecasting.py`:
  - Baseline: Facebook Prophet for each meter using 90 days of history + weather covariates (NOAA / Open-Meteo).
  - Optional gradient-boosted model (`lightgbm`) trained nightly per building.
  - Outputs include `p10`, `p50`, `p90` percentiles and a `model_version` reference.
- API:
  - `POST /api/v1/buildings/{id}/forecasts/energy` body `{ "horizon_hours": 168, "covariates": { "occupancy_factor": 1.0, "setback_temp_delta_c": 2.0 } }`.
  - Scenarios re-run with overridden covariates; comparison endpoint returns `{ "baseline": [...], "scenario": [...], "delta": [...] }`.
- Background job nightly retrains models per building; results cached in `ai_models.performance_metrics`.

**Testing**:
- Unit: Forecasting service called with insufficient history (< 14 days) returns `{"status": "insufficient_data"}`.
- Integration: With 90 days of synthetic seasonal data, MAPE on the next 7 days < 15%.
- Integration: Scenario with `setback_temp_delta_c=2` produces a lower forecast than baseline (sign check, not magnitude).

#### 7.4 — Frontend: energy dashboard

**What**: `/energy` page with portfolio→building rollups, time-series charts, sub-meter breakdown, emissions, and scenario panel.

**Design**:
- Server-rendered KPI cards (total kWh, cost, kgCO₂e, intensity per m²).
- ECharts time-series with toggleable series (electricity, gas, water, steam).
- Sankey diagram: building → sub-meters → equipment groups.
- Scenario panel with sliders for setback delta, occupancy factor, schedule shift; live API call refreshes the forecast chart.

**Testing**:
- E2E (Playwright): Page loads, all KPIs populated, scenario slider movement updates the forecast chart within 3s.
- Unit (Vitest): KPI card shows "—" when data is unavailable instead of throwing.

---

## Phase 8: AI Layer — Optimisation, Fault Detection, NL Query

### Purpose

Deliver the AI-native advantage that defines the project. After this phase, the platform autonomously optimises HVAC and lighting setpoints; surfaces multivariate fault patterns ahead of threshold alerts; and answers natural-language questions about building data. This is what differentiates the platform from incumbents that retrofit GenAI onto rule-based stacks.

### Tasks

#### 8.1 — LLM facade and prompt registry

**What**: `packages/ai/sbp_ai/llm.py` — a provider-neutral facade for chat completions, embeddings, and tool calling.

**Design**:
- Concrete adapters: `AnthropicAdapter`, `OpenAIAdapter`, `OllamaAdapter`. All conform to:
  ```python
  class LLM(Protocol):
      async def complete(self, system: str, messages: list[Message], tools: list[Tool] | None = None,
                          max_tokens: int = 1024, temperature: float = 0.2) -> Completion: ...
      async def embed(self, texts: list[str]) -> list[list[float]]: ...
  ```
- Prompt registry at `packages/ai/sbp_ai/prompts/`:
  - `nl_to_query.system.md` — system prompt for natural-language → API query translation; includes ontology context.
  - `fault_explain.system.md` — system prompt for human-readable explanations of detected faults.
  - `recommendation_rationale.system.md` — system prompt for recommendation summaries.
  - Each prompt versioned with a `version: N` front-matter; `LLM.complete()` records the version in the response metadata.

**Testing**:
- Unit (mocked): Each adapter returns a normalised `Completion`.
- Unit: Embedding round-trip preserves dimension (e.g., 1024 for Claude embeddings).
- Unit: Prompt registry loads the correct version on demand.

#### 8.2 — Natural-language query

**What**: `POST /api/v1/nl-query` translates a user's question into a structured query and returns the answer with the SQL/SPARQL the model produced.

**Design**:
- Two-stage pipeline:
  1. **Plan**: LLM with tools `list_buildings`, `list_equipment`, `list_points`, `query_telemetry`, `sparql_brick` selects a tool and parameters.
  2. **Execute**: server runs the tool against the database and returns results.
- All queries run with the requesting user's RLS context — the model has no privileged access.
- System prompt includes a compact ontology summary (top 50 Brick classes, Haystack markers) and table-of-tools schema.
- Response shape:
  ```json
  {
    "answer": "Meeting room 5B exceeded 800 ppm CO2 on 4 days last week.",
    "tool_calls": [ { "name": "query_telemetry", "args": {...}, "result_rows": 4 } ],
    "model_version": "claude-opus-4.7",
    "prompt_version": 3,
    "tokens": { "input": 1234, "output": 87 }
  }
  ```
- Result caching: identical (user, question) within 60s returns cached.

**Testing**:
- Unit (mocked LLM): Question "show CO₂ > 800 on floor 5 last week" routes through `query_telemetry` with the right filters.
- Integration: Real LLM (gated via env var `RUN_LLM_TESTS=1`) answers a deterministic seeded scenario within tolerance.
- Integration: User from tenant A cannot extract tenant B data even if the model's SQL would attempt it (RLS blocks).

#### 8.3 — Predictive fault detection

**What**: A multivariate anomaly detector running per equipment, producing `ai_recommendations` of type `maintenance_alert`.

**Design**:
- `packages/ai/sbp_ai/fault_detection.py`:
  - Per-equipment feature window (last 24h) of all related points.
  - Model: `IsolationForest` baseline, plus per-equipment-type residual models (autoencoder for AHUs trained on historical normal operation).
  - Rule-mining layer on top: ASHRAE Guideline 36 fault patterns (e.g., simultaneous heating + cooling in an AHU) encoded as declarative rules.
- Outputs an `ai_recommendation` row with `confidence`, `estimated_impact`, and a structured `evidence` JSON pointing to the contributing telemetry windows.
- A worker schedule runs every 15 minutes per equipment with new telemetry.

**Testing**:
- Unit: Rule "supply air temp rising while cooling coil command > 80%" fires on the corresponding fixture.
- Integration: Seed 7 days of normal AHU telemetry + 2 hours of simulated fault data → `IsolationForest` flags the fault window.
- Integration: Recommendation row created with `recommendation_type='maintenance_alert'`, `status='pending'`.

#### 8.4 — HVAC and lighting setpoint optimisation

**What**: Worker `ai_optimisation` periodically computes setpoint adjustments balancing comfort, energy cost, and occupancy.

**Design**:
- Inputs: current zone temp, setpoint, occupancy (Phase 9), weather forecast, day-ahead energy price (from `energy_prices` table populated by an external feed), comfort bounds from `comfort_preferences`.
- Approach: model-predictive control (MPC) over a 4-hour horizon using a linear thermal model fitted nightly per zone, optimised with `cvxpy`.
- Output: a list of recommended setpoint changes; if the zone equipment is bound to a writable BACnet object and `tenant.settings.autonomous_control=true`, the worker writes the setpoint via the BACnet connector. Otherwise it creates an `ai_recommendation` for human approval.
- Every change is logged to `audit_log` with `action='override_setpoint'` and recorded as telemetry `source='ai_override'`.

**Testing**:
- Unit: MPC with synthetic thermal model and known forecast produces lower energy than a constant setpoint.
- Integration: Recommendation flow — autonomous control off → recommendation appears in UI; user accepts → setpoint write attempted.
- Integration: With autonomous control on and a BACnet mock writable object, the write occurs and is audited.

#### 8.5 — Space utilisation insights

**What**: Worker `ai_space_insights` analyses occupancy + bookings and emits weekly recommendations.

**Design**:
- Clustering of spaces by usage pattern (`scikit-learn` KMeans on weekly occupancy vectors).
- Rule layer: "space utilised < 20% during business hours for 4 weeks" → recommend consolidation; "booked but unoccupied > 30% of the time" → recommend auto-release.
- Each recommendation includes a rationale generated by the LLM facade using the `recommendation_rationale` prompt.

**Testing**:
- Integration: Seed occupancy showing chronic under-utilisation → recommendation generated with `recommendation_type='space_optimization'`.
- Unit: Rationale generation receives the structured evidence object and produces non-empty text.

#### 8.6 — Frontend: AI surfaces

**What**: `/nl-query` chat surface; `/recommendations` page; explanatory tooltips on alarms.

**Design**:
- Chat UI streams the answer as it generates (`fetch` with `text/event-stream`).
- Recommendations page groups by `recommendation_type`; each card shows confidence, estimated impact, evidence drill-down (links to charts), and accept/reject/snooze actions.
- Alarm rows include a "Why?" button that calls a `POST /api/v1/alarms/{id}/explain` endpoint using the `fault_explain` prompt.

**Testing**:
- E2E (Playwright): Ask "Which buildings used the most energy last week?" in the NL chat → response renders streaming, then settles with a coherent answer.
- E2E: Click "Accept" on a setpoint recommendation → status flips to `accepted` in DB.

---

## Phase 9: Occupancy, IAQ, Bookings, Tenant Experience

### Purpose

Add the occupant-facing capabilities: privacy-preserving occupancy ingestion, indoor air quality monitoring, room bookings with auto-release, and per-tenant lease/occupant tracking. After this phase, the platform supports the workplace-experience use cases that Cohesion and Occuspace target.

### Tasks

#### 9.1 — Occupancy and IAQ schemas + ingestion

**What**: Migration `0010_occupancy_iaq.py` (occupancy_sensors, occupancy_readings, iaq_readings).

**Design**: Verbatim SQL from data-model-suggestion-1. Both reading tables are hypertables.

Connector additions:
- An Occuspace-style passive WiFi/BLE feed: a new REST connector preset under `apps/worker/src/sbp_worker/connectors/presets/occuspace.py` that maps Occuspace API responses to `occupancy_readings`.
- A Kaiterra-style IAQ feed preset that writes to `iaq_readings` and `telemetry`.

**Testing**:
- Integration (mocked Occuspace API): Periodic poll writes `occupancy_readings` rows; `is_privacy_safe=true` enforced when `sensor_type='passive_wifi'`.
- Integration: IAQ readings populate composite `iaq_score` calculated as the configured weighted average of CO₂, PM2.5, TVOC bands.

#### 9.2 — Bookings with auto-release

**What**: Migration `0011_bookings.py` + booking API + auto-release worker.

**Design**:
- Verbatim SQL from data-model-suggestion-1, including the GIST EXCLUDE constraint preventing overlapping confirmed bookings.
- API:
  - `POST /api/v1/spaces/{id}/bookings` `{ "starts_at", "ends_at", "title" }` → 201 or 409 on overlap.
  - `DELETE /api/v1/bookings/{id}` → cancels.
- Auto-release worker (Celery Beat every minute): for any `confirmed` booking whose `auto_release_minutes` has elapsed past `starts_at` with `occupancy_readings.occupant_count==0` for the space, transition to `auto_released` and notify the booker.

**Testing**:
- Integration: Two overlapping confirmed bookings → second returns 409.
- Integration: A booking with no occupancy after `auto_release_minutes` is auto-released; a booking with occupancy is not.
- Unit: Notification template renders correctly for the release event.

#### 9.3 — Tenant experience: leases, occupants, preferences

**What**: Migration `0012_tenant_experience.py` (building_tenants, building_tenant_spaces, comfort_preferences); API.

**Design**: Verbatim SQL. API includes `POST /api/v1/buildings/{id}/tenants`, `POST /api/v1/tenants/{id}/spaces` (assigns spaces), `GET/PUT /api/v1/me/comfort-preferences`.

**Testing**:
- Integration: Assigning a space to a building tenant prevents another building tenant from booking it (policy enforced at API layer).
- Integration: Updating comfort preferences for a user with active setpoint optimisation triggers a fresh optimisation run.

#### 9.4 — Occupant mobile-friendly portal

**What**: A subset of pages under `/portal/*` optimised for mobile: book a room, see IAQ in your area, set comfort preferences, access (Phase 11+).

**Design**:
- PWA manifest, offline shell, push notifications via web push.
- Room list shows IAQ score chips (green/amber/red) and live availability.

**Testing**:
- E2E (Playwright mobile viewport): Book a room from the portal, receive a confirmation push, verify auto-release after the configured window with no occupancy.

---

## Phase 10: Public API, Webhooks, MCP Server, Developer Portal

### Purpose

Make the platform an integration substrate. After this phase, third parties can register apps, subscribe to events, and connect agentic AI workflows via MCP — matching the developer-platform posture of Siemens Building X and Honeywell Forge while remaining open-source.

### Tasks

#### 10.1 — API keys and OAuth 2.0 client registration

**What**: Self-service developer portal under `/developers/*`; API keys and OAuth 2.0 client-credentials for machine-to-machine.

**Design**:
- New tables `api_clients` (`client_id`, `client_secret_hash`, `scopes`, `rate_limit_rps`), `api_keys` (`key_prefix`, `key_hash`, `scopes`, `expires_at`).
- Endpoints: `POST /api/v1/developers/clients`, `POST /oauth/token` (client_credentials grant).
- Rate limiting via Redis token bucket per client.
- All scopes use `<resource>:<action>` form (`buildings:read`, `telemetry:write`).

**Testing**:
- Integration: Create a client, exchange credentials for an access token, call a scoped endpoint.
- Integration: Exceeding `rate_limit_rps` returns 429 with `Retry-After`.

#### 10.2 — Webhooks

**What**: Subscribe to platform events (alarms, recommendations, work-order updates) via HTTPS webhooks.

**Design**:
- `POST /api/v1/webhooks` `{ "url", "events": ["alarm.triggered", ...], "secret" }`.
- Delivery worker signs payloads with HMAC-SHA256 over the JSON body using the subscription secret, header `X-SBP-Signature: t=<timestamp>,v1=<hex>`.
- At-least-once delivery with exponential backoff (1s, 5s, 30s, 5m, 30m, 6h, abandon after 24h).
- Delivery attempts visible via `GET /api/v1/webhooks/{id}/deliveries`.

**Testing**:
- Integration: Trigger alarm → mock receiver gets a POST with valid signature.
- Integration: Receiver returns 500 three times then 200 → exactly one logical delivery, four attempts recorded.

#### 10.3 — MCP server

**What**: A separate process (`apps/mcp_server/`) exposing building-data tools via the Model Context Protocol so external LLM agents (Claude Desktop, Cursor, agentic workflows) can query and act on the platform.

**Design**:
- Tools exported:
  - `list_buildings`, `list_equipment(building_id)`, `list_points(equipment_id)`
  - `query_telemetry(point_ids, from, to, aggregation)`
  - `query_sparql(query)`
  - `list_alarms(filters)`, `acknowledge_alarm(id, note)`
  - `create_work_order(building_id, title, ...)`, `update_work_order(...)`
  - `apply_recommendation(id)` (gated by user confirmation flow)
  - `forecast_energy(building_id, horizon_hours, covariates)`
- Authentication: each MCP session presents an OAuth 2.0 access token; all DB calls go through the same RLS pathway as the REST API.
- Transport: stdio for local, SSE for hosted.

**Testing**:
- Unit (MCP SDK harness): `list_buildings` returns expected schema.
- Integration: Connect Claude Desktop to the local MCP server, list tools, call `query_telemetry`, receive normalised result.

#### 10.4 — Developer portal pages

**What**: Pages at `/developers` for docs (rendered from OpenAPI + Markdown), clients, keys, webhook subscriptions, and a sandbox playground.

**Design**:
- `redoc` for the OpenAPI spec.
- Try-it-out playground embeds the API base URL and the user's currently selected client credential.

**Testing**:
- E2E (Playwright): Create a client in the UI, copy credentials, use them in the playground, see a successful response.

---

## Phase 11: Multi-Site Portfolio, Benchmarking, Hardening, and Release

### Purpose

Round the platform into a production-ready 1.0: portfolio-level dashboards and cross-building benchmarking, security hardening per OWASP API Top 10 / IEC 62443, performance testing at realistic scale, documentation, and the v1.0 release artefacts.

### Tasks

#### 11.1 — Portfolio dashboards and benchmarking

**What**: `/portfolio` view that aggregates KPIs across buildings, and a benchmarking view comparing same-typology buildings.

**Design**:
- Portfolio KPIs (energy, EUI kWh/m², emissions intensity, comfort score, open critical alarms, open work orders) computed via materialised view refreshed hourly.
- Benchmarking: peer set defined by `building_type` and area band. Box plots show each metric distribution; the focal building is highlighted.

**Testing**:
- Integration: Aggregations match per-building rollups summed.
- E2E: Switch a building in/out of the peer set, percentiles recompute.

#### 11.2 — Security hardening

**What**: Apply OWASP API Security Top 10 + IEC 62443 controls and produce a hardening report.

**Design**:
- API1: Object-level authorisation tests for every resource endpoint.
- API3: Request schemas validated against Pydantic; response models prevent over-exposure (explicit `response_model` on every route).
- API4: Rate limiting per route group, configurable.
- API7: CSP + Trusted Types on the frontend; CORS allowlist.
- API8: Secrets only via env or HashiCorp Vault; no secrets in DB.
- API10: SBOM generated with `syft` per release; container images signed with `cosign`.
- IEC 62443: Network segmentation guidance in `docs/deployment.md`; mandatory TLS 1.3 for ingress; BACnet/SC required for any actuating connection.
- Vulnerability scan in CI: `trivy fs` and `trivy image`; fail on HIGH/CRITICAL.

**Testing**:
- Security: Run OWASP ZAP baseline against the running stack in CI; fail on new findings.
- Unit: Object-level authorisation matrix test (parametrised over resource × role × action).
- Integration: Attempt to read tenant B data with tenant A token → 404 in all cases.

#### 11.3 — Performance and capacity testing

**What**: Load tests demonstrating the platform meets target capacities, with a published reference architecture per scale tier.

**Design**:
- Targets:
  - Ingest: 50,000 telemetry rows/sec sustained on a single 4-vCPU writer.
  - API: p95 < 200ms for read endpoints under 500 concurrent users.
  - WebSocket: 10,000 concurrent subscribers per node.
- Tooling: `locust` for HTTP/WebSocket; a synthetic telemetry generator for ingest.
- Reference architectures in `docs/deployment.md`: single-node demo, mid (3-node), portfolio (HA Postgres + Redis Cluster + multi-worker pool).

**Testing**:
- Performance: Locust run for 30 min at target load; report archived as a CI artefact.
- Soak: 12h stability run; no memory growth > 5%.

#### 11.4 — Documentation, examples, release

**What**: Public docs site, a working "demo building" seed, the 1.0 GitHub release.

**Design**:
- Docs site at `docs/` rendered with Docusaurus; deployed via GitHub Pages.
- `scripts/seed-demo-data.py` creates a tenant, 2 sites, 5 buildings, 200 equipment, 1,500 points, and 30 days of synthetic telemetry, occupancy, and energy.
- Release workflow: tag `v1.0.0` triggers image builds, Helm chart publish, GitHub Release with CHANGELOG and signed SBOM attachments.

**Testing**:
- E2E: Fresh user clones repo, runs `make dev && make seed`, hits dashboards within 5 minutes.
- Manual: Release rehearsal on a release-candidate tag confirms all artefacts publish.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, tooling, multi-tenancy        ─── required by everything
    │
Phase 2: Spatial hierarchy + reference ontology    ─── requires 1
    │
Phase 3: Equipment, points, telemetry              ─── requires 2
    │
Phase 4: Multi-protocol ingestion                  ─── requires 3
    │
Phase 5: Normalisation, ontology mapping           ─── requires 3 (can parallel with late 4)
    │
    ├── Phase 6: Alarms, notifications, work orders   ─── requires 3 (parallel with 5)
    ├── Phase 7: Energy + carbon forecasting          ─── requires 3 (parallel with 5, 6)
    └── Phase 9: Occupancy, IAQ, bookings, tenants    ─── requires 3 (parallel with 5, 6, 7)
         │
Phase 8: AI layer — optimisation, faults, NL query ─── requires 5, 6, 7, 9
    │
Phase 10: Public API, webhooks, MCP, dev portal   ─── requires 6 (and benefits from 8)
    │
Phase 11: Portfolio, benchmarking, hardening, GA   ─── requires all
```

Phases 5, 6, 7, 9 are the most parallelisable; in a 4-developer team the natural assignment is one engineer per phase after Phase 4 ships.

**Scope estimate**: Large. ~11 phases · ~50 tasks · realistic delivery 9–14 months for a small team (3–5 engineers + part-time front-end designer) given the breadth of protocol integrations and the AI surface.

---

## Definition of Done (per phase)

Every phase is "done" only when every item below is satisfied:

1. All tasks implemented as specified (Design section).
2. All unit, integration, and E2E tests listed in the phase pass in CI.
3. Ruff lint + format pass; mypy strict passes; ESLint + tsc pass.
4. Alembic migrations are reversible (`alembic downgrade -1` then `upgrade head` succeeds on a populated database).
5. Docker images for any new service build and run under `docker compose up`.
6. The new endpoints appear in the auto-generated OpenAPI 3.1 spec and in the regenerated TypeScript client.
7. Any new RLS-enabled tables have automated cross-tenant isolation tests.
8. New config options documented in `docs/` and represented in `.env.example`.
9. New Celery tasks have explicit retry, idempotency, and dead-letter behaviour documented and tested.
10. New external integrations (LLM, protocols, vendor APIs) ship with a feature-flag and a documented disable path.
11. New frontend pages pass accessibility checks (`@axe-core/playwright`) with no critical violations.
12. A short "phase release notes" section is appended to `CHANGELOG.md`.
