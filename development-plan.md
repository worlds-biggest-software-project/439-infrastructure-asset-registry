# Infrastructure Asset Registry — Phased Development Plan

> Project: 439-infrastructure-asset-registry · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-1.md` (the normalised relational PostgreSQL + PostGIS model, adopted as the canonical schema). Where class-specific attributes need flexibility, the JSONB pattern from `data-model-suggestion-3.md` is incorporated rather than the rigid EAV table. Event sourcing (suggestion 2) and the graph/time-series polyglot (suggestion 4) are explicitly rejected for the MVP on complexity grounds; the audit-log table satisfies regulatory traceability without them.

---

## Product Summary

An open-source, AI-native platform that gives municipalities, transportation departments, and utilities a structured, GIS-linked inventory of physical infrastructure assets (roads, bridges, pipes, culverts, signals, facilities). It records location, condition, maintenance history, and financial data per asset, and connects condition-inspection workflows to capital planning so agencies make defensible, condition-based investment decisions. The core differentiator is vendor-neutral GIS (no Esri ArcGIS licence — PostGIS, GeoJSON, OGC API Features) combined with transparent, explainable AI for condition scoring, deterioration forecasting, natural-language querying, and automated GASB 34 / grant reporting.

**Primary personas**: public works director, asset management coordinator, field inspector (mobile, offline), finance officer (GASB 34), municipal CIO, and citizen (request portal).

**Deployment model**: self-hosted, cloud, or hybrid — Docker Compose for single-node, container images for orchestrated deployment. Multi-tenant by `organization_id` for hosted offerings.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | Domain is data + GIS + ML/LLM heavy. Best ecosystem for PostGIS (GeoAlchemy2, Shapely, pyproj), computer vision (torch/ultralytics), and LLM SDKs. Aligns with pygeoapi reference implementation cited in standards.md. |
| API framework | FastAPI | Native OpenAPI 3.1 generation (standards.md requires OAS 3.1), Pydantic v2 validation (JSON Schema 2020-12), async for I/O-bound LLM/GIS calls, ASGI. |
| Data validation | Pydantic v2 | Emits JSON Schema 2020-12 directly; one model layer drives request/response validation and OpenAPI. |
| Database | PostgreSQL 16 + PostGIS 3.4 | Adopted per data-model-suggestion-1. Native spatial types + GIST indexes; OGC-compatible (`ST_AsGeoJSON`, WKT/WKB); `ltree` for hierarchy; JSONB for flexible per-class attributes. No Esri dependency — the explicit market gap. Extensions: `postgis`, `ltree`, `pg_trgm`, `uuid-ossp`/`pgcrypto`. |
| ORM / spatial mapping | SQLAlchemy 2.0 + GeoAlchemy2 | Mature, async-capable, maps PostGIS geometry columns to Shapely objects. |
| Migrations | Alembic | Version-controlled schema with PostGIS-aware autogenerate. Replaces Flyway/Liquibase suggestion for a Python stack. |
| Task queue | Celery + Redis | Async workloads: AI condition scoring (vision inference), GIS sync jobs, deterioration model runs, scenario simulations, report generation, bulk import. Redis doubles as result backend and cache. |
| Object storage | MinIO (S3-compatible) | Inspection photos/videos kept out of the DB (per scaling notes). S3 in cloud, MinIO self-hosted. |
| LLM provider abstraction | LiteLLM | Vendor-neutral routing across OpenAI/Anthropic/local Ollama — matches the open-source, no-lock-in ethos. NL query, report drafting, import field mapping, NL work-order creation. |
| Computer vision | Ultralytics YOLO (detection) + torch | Distress detection on pavement/structure photos. Bounding boxes feed an explainable score; field inspectors can challenge results (transparency requirement). |
| Web frontend | React 18 + TypeScript + Vite | SPA dashboard. Map-centric UI mirrors incumbents. |
| Mapping (web) | MapLibre GL JS + Leaflet fallback | Open-source vector/raster mapping, no Esri/Mapbox token lock-in. Renders GeoJSON / vector tiles from the API. |
| Mobile inspection | React Native (Expo) + WatermelonDB | Offline-first field inspections with photo capture; local SQLite store syncs to API when connectivity returns. Addresses the "battery-draining, crash-prone incumbent app" gap. |
| AuthN / AuthZ | OAuth 2.0 + OIDC (Authlib), RBAC | standards.md mandates OAuth2/OIDC for federated municipal identity; RBAC roles from the data model (admin/manager/supervisor/inspector/field_crew/finance/viewer/citizen). |
| Vector tiles | pg_tileserv or Tegola | Serve MVT tiles directly from PostGIS for network-level map performance. |
| Reporting / PDF | WeasyPrint (HTML→PDF) | GASB 34 and grant report templates rendered server-side. |
| MCP server | Official Python MCP SDK | standards.md flags an early-mover opportunity: expose asset/condition/capital data to AI agents via Model Context Protocol. |
| Testing | pytest + pytest-asyncio + testcontainers + Playwright | Unit + integration (real Postgres/PostGIS via testcontainers) + frontend E2E. |
| Code quality | ruff (lint+format), mypy, pre-commit | Standard modern Python toolchain. |
| Package manager | uv | Fast, reproducible Python dependency management and locking. |
| Containerisation | Docker + Docker Compose | Self-hosted single-node and orchestrated deployment. |
| CI | GitHub Actions | Lint, type-check, test matrix, Docker build, OpenAPI export. |

### Standards Alignment (from standards.md)

- **OGC API Features (Parts 1 & 4)** — spatial query/CRUD interface; all spatial responses are GeoJSON `FeatureCollection`s (RFC 7946).
- **OpenAPI 3.1 / JSON Schema 2020-12** — every endpoint documented and validated.
- **ASTM D6433 (PCI)**, **FHWA NBI**, **AASHTO CoRe**, **ACI** — pluggable condition frameworks; never hard-coded.
- **GASB 34** — both depreciation and modified approaches; self-service report generator.
- **ISO 55001** — SAMP/AMP artefacts, documented-information traceability via `audit_log`.
- **W3C PROV-O** — audit-trail provenance semantics.
- **OAuth 2.0 / OIDC**, **OWASP API Security Top 10 (2023)**, **NIST SP 800-53** — security baseline.
- **GML import/export** — legacy municipal GIS interoperability.
- **MCP** — AI-agent access surface.

### Project Structure

```
infrastructure-asset-registry/
├── pyproject.toml                  # uv project, ruff/mypy/pytest config
├── uv.lock
├── Dockerfile                      # API + worker image
├── docker-compose.yml              # postgres+postgis, redis, minio, api, worker, web
├── docker-compose.dev.yml
├── alembic.ini
├── .github/workflows/ci.yml
├── README.md
├── migrations/                     # Alembic versions
│   ├── env.py
│   └── versions/
├── src/
│   └── iar/                        # Infrastructure Asset Registry package
│       ├── __init__.py
│       ├── main.py                 # FastAPI app factory, router registration
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py          # async engine, session factory
│       │   ├── base.py             # declarative base, mixins (audit cols, org_id)
│       │   └── models/             # SQLAlchemy models grouped by domain
│       │       ├── org.py          # organizations, departments, users, permissions
│       │       ├── taxonomy.py     # asset_classes, asset_class_attributes
│       │       ├── asset.py        # assets + type-specific tables
│       │       ├── condition.py    # frameworks, bands, distress_types, inspections
│       │       ├── workorder.py    # work_orders, costs, labor, templates
│       │       ├── finance.py      # financials, gasb34, capital_projects, scenarios
│       │       ├── citizen.py      # citizen_requests
│       │       └── audit.py        # audit_log, import_batches, gis_data_sources
│       ├── schemas/                # Pydantic request/response models per domain
│       ├── api/
│       │   ├── deps.py             # auth, db session, current_user, org scoping
│       │   ├── routers/            # one router module per domain
│       │   └── ogcapi/             # OGC API Features endpoints
│       ├── services/               # business logic (no HTTP/DB-session coupling)
│       │   ├── condition/          # PCI, NBI, ACI scorers (framework plugins)
│       │   ├── deterioration/      # deterioration models + scenario engine
│       │   ├── importing/          # spreadsheet/GIS import + AI field mapping
│       │   ├── reporting/          # GASB 34, grant, NL report generation
│       │   ├── ai/                 # LLM + CV clients, prompt templates
│       │   └── geo/                # GeoJSON/GML conversion, spatial helpers
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks/              # async tasks per domain
│       ├── auth/                   # OAuth2/OIDC, RBAC policy
│       ├── mcp/                    # MCP server exposing registry tools
│       └── core/                   # errors, pagination, logging, security utils
├── web/                            # React + TS + Vite SPA
│   ├── package.json
│   └── src/
│       ├── api/                    # generated client from OpenAPI
│       ├── map/                    # MapLibre components
│       ├── pages/
│       └── components/
├── mobile/                         # React Native (Expo) offline inspection app
│   └── src/
├── tests/
│   ├── conftest.py                 # testcontainers postgres+postgis fixtures
│   ├── unit/
│   ├── integration/
│   ├── fixtures/                   # sample GeoJSON, PCI inspection data, CSVs
│   └── e2e/
└── docs/
    ├── openapi.json                # exported spec
    └── data-model.md
```

---

## Phase 1: Foundation & Core Asset Registry

### Purpose
Establish the project skeleton, database with PostGIS, configuration, migrations, and the core asset/organisation/taxonomy data model with CRUD APIs. After this phase an agency can register organisations, departments, asset classes, and assets with spatial geometry, and retrieve them — the irreducible heart of an asset registry.

### Tasks

#### 1.1 — Project scaffolding & tooling

**What**: Create the uv project, Docker Compose stack, linting/type/test config, and CI.

**Design**:
- `pyproject.toml` declares dependencies (fastapi, uvicorn, sqlalchemy[asyncio], geoalchemy2, asyncpg, alembic, pydantic-settings, shapely, celery[redis], boto3, litellm, authlib, python-multipart).
- `config.py` — Pydantic `Settings`:
  ```python
  class Settings(BaseSettings):
      database_url: str = "postgresql+asyncpg://iar:iar@localhost:5432/iar"
      redis_url: str = "redis://localhost:6379/0"
      s3_endpoint: str = "http://localhost:9000"
      s3_bucket: str = "iar-media"
      jwt_secret: str
      oidc_issuer: str | None = None
      llm_model: str = "gpt-4o-mini"
      environment: Literal["dev", "test", "prod"] = "dev"
      model_config = SettingsConfigDict(env_prefix="IAR_", env_file=".env")
  ```
- `docker-compose.yml` services: `postgis/postgis:16-3.4`, `redis:7`, `minio`, `api`, `worker`, `web`.
- `main.py` — `create_app()` factory registering routers, exception handlers, CORS, OpenAPI metadata (`openapi_version="3.1.0"`).
- Health endpoint `GET /healthz` → `{"status":"ok","db":bool,"redis":bool}`.

**Testing**:
- `Unit: Settings loads from env with IAR_ prefix → correct typed values`
- `Unit: missing required jwt_secret → ValidationError`
- `Integration: GET /healthz with stack up → 200, db=true, redis=true`
- `Integration: docker compose up → all services healthy (CI smoke test)`

#### 1.2 — Database base, mixins, and migrations bootstrap

**What**: Declarative base, shared mixins, Alembic configured for PostGIS.

**Design**:
- `OrgScopedMixin` (`organization_id: Mapped[UUID]`), `AuditMixin` (`created_at`, `updated_at`, `created_by`, `updated_by`), `UUIDPKMixin` (`id: Mapped[UUID] = mapped_column(default=uuid4)`).
- Alembic `env.py` enables `postgis`, `ltree`, `pg_trgm`, `pgcrypto` extensions in the first migration; teaches autogenerate about `geoalchemy2` types.
- Initial migration `0001_extensions_and_org.py`.

**Testing**:
- `Integration (real PG): run migrations on clean DB → extensions installed, tables exist`
- `Integration: alembic downgrade then upgrade → idempotent`

#### 1.3 — Organisations, departments, users, permissions

**What**: Multi-tenant org model and user/permission tables with CRUD.

**Design**: Implement `organizations`, `departments`, `users`, `user_permissions` exactly per data-model-suggestion-1 §"Organizations and Users". SQLAlchemy models + Pydantic schemas (`OrganizationCreate/Read/Update`, etc.). Routers:
- `POST/GET/PATCH/DELETE /organizations`
- `POST/GET/PATCH/DELETE /organizations/{org_id}/departments`
- `POST/GET/PATCH /organizations/{org_id}/users`
Enforce `UNIQUE(organization_id, code)` for departments; `users.email` globally unique. `org_type`, `user_role`, `department_type` validated against the CHECK enums via Pydantic `Literal`/`Enum`.

**Testing**:
- `Unit: OrganizationCreate with invalid org_type → ValidationError`
- `Integration: create org → 201 with UUID; duplicate dept code in same org → 409`
- `Integration: create user with existing email → 409`
- `Integration: list departments scoped to org_id → only that org's rows`

#### 1.4 — Asset classification taxonomy

**What**: `asset_classes` hierarchy and `asset_class_attributes` definitions.

**Design**: Per data-model-suggestion-1 §"Asset Classification Taxonomy". `asset_classes.parent_id` self-reference; `asset_category ∈ {horizontal,vertical,point,linear,area}`; `depreciation_method` and `default_useful_life_years` for GASB. `asset_class_attributes` defines custom fields (name, label, data_type, unit, required, select_options[]). Routers under `/organizations/{org_id}/asset-classes`. Seed a starter taxonomy fixture (Roads, Bridges, Pipes, Signals, Facilities, Culverts).

**Testing**:
- `Unit: asset_category not in enum → ValidationError`
- `Integration: create child class referencing parent → hierarchy_level computed`
- `Integration: seed fixture loads 6 root classes with attributes`

#### 1.5 — Core asset entity with spatial geometry

**What**: The `assets` table with point/line/polygon geometry, hierarchy (ltree), JSONB extended attributes, and CRUD.

**Design**:
- Model per data-model-suggestion-1 §"Core Asset Registry": identification, hierarchy (`parent_asset_id`, `hierarchy_level`, `hierarchy_path LTREE`), location, three geometry columns (`geom_point GEOMETRY(Point,4326)`, `geom_line`, `geom_polygon`), physical characteristics, lifecycle, status, criticality, denormalised `current_condition_*`.
- **Deviation from suggestion-1**: replace the EAV `asset_attributes` table with `assets.custom_attributes JSONB` (validated against the relevant `asset_class_attributes` definitions at the service layer) — adopting suggestion-3's flexibility to avoid the documented EAV performance penalty. Keep `asset_class_attributes` as the schema-of-record.
- GIST indexes on all three geometry columns; GIST on `hierarchy_path`; composite indexes per the model.
- Pydantic schemas accept/emit geometry as GeoJSON (RFC 7946); service converts GeoJSON ↔ Shapely/WKB.
- Endpoints: `POST/GET/PATCH/DELETE /organizations/{org_id}/assets`, with filters `?asset_class_id=&status=&bbox=minx,miny,maxx,maxy&parent_asset_id=`.
- `bbox` filter → `ST_Intersects(geom, ST_MakeEnvelope(...,4326))`.
- Cursor pagination (RFC 8288 `Link` header).

**Testing**:
- `Unit: GeoJSON Point → Shapely → WKB round-trips with SRID 4326`
- `Unit: asset with custom_attributes missing a required class attribute → 422 with field name`
- `Integration: create road asset with LineString geometry → 201, ST_AsGeoJSON returns equivalent coords`
- `Integration: bbox filter returns only assets intersecting envelope`
- `Integration: duplicate (org_id, asset_number) → 409`
- `Integration: pagination Link header next/prev correct`

#### 1.6 — Type-specific asset detail tables

**What**: `asset_roads`, `asset_bridges`, `asset_pipes`, `asset_signals`, `asset_facilities`, `asset_culverts` (table-per-subclass).

**Design**: Implement all six exactly per data-model-suggestion-1 §"Type-Specific Asset Tables", each keyed `asset_id PK→assets.id ON DELETE CASCADE`. Service resolves the detail table from the asset's `asset_class.asset_category` + class code. Nested in asset payload: `POST /assets` body includes optional `road`/`bridge`/`pipe`/... sub-object; reads embed it. NBI-relevant fields (`nbi_structure_number`, `fracture_critical`, `scour_critical`, `waterway_adequacy`) present on bridges for later NBI scoring.

**Testing**:
- `Unit: bridge sub-object with bridge_type not in enum → ValidationError`
- `Integration: create bridge asset → row in assets AND asset_bridges; GET embeds bridge fields`
- `Integration: delete asset → cascade removes asset_bridges row`

### Definition of Done
All §"Definition of Done (per phase)" items; assets with each geometry type create/read/update/delete cleanly; OpenAPI spec exports.

---

## Phase 2: Authentication, RBAC & Audit Trail

### Purpose
Add OAuth2/OIDC authentication, role-based authorisation, and the comprehensive audit log. Required before any multi-user or regulated deployment; satisfies OWASP API Security Top 10 (broken object-level/function-level authorisation) and ISO 55001 / GASB 34 traceability.

### Tasks

#### 2.1 — OAuth2 / OIDC authentication

**What**: Token-based auth supporting local password grant and external OIDC providers.

**Design**:
- `auth/` using Authlib. Local: `POST /auth/token` (OAuth2 password grant) → JWT access (15 min) + refresh (7 d). External: OIDC authorization-code flow with `oidc_issuer` discovery; map IdP claims → `users`.
- JWT claims: `sub` (user_id), `org` (organization_id), `role`, `exp`. Signed with `jwt_secret` (HS256 dev) or RS256 in prod.
- `api/deps.py`: `get_current_user()` validates token, loads user, attaches to request.

**Testing**:
- `Unit: expired token → 401`
- `Unit: token with tampered signature → 401`
- `Integration: valid credentials → access+refresh tokens; refresh → new access token`
- `Integration (mocked OIDC): auth-code callback creates/links user`

#### 2.2 — RBAC policy & object-level authorisation

**What**: Enforce role permissions and org-scoping on every endpoint.

**Design**:
- Permission matrix mapping `(role, resource, action)` → allow/deny. e.g. `inspector` may create inspections but not delete assets; `finance` reads financials; `citizen` may only create citizen_requests and read public dashboards; `viewer` read-only.
- Dependency `require(permission: str)` and `enforce_org(resource.organization_id == current_user.org)` — addresses OWASP API1 (BOLA) and API5 (BFLA).
- Optional row-level: PostgreSQL RLS policies keyed on a `SET app.current_org` GUC for defence-in-depth.

**Testing**:
- `Unit: permission matrix denies inspector→delete asset`
- `Integration: user from org A requests org B asset by id → 404 (not 403, no existence leak)`
- `Integration: citizen creating asset → 403`
- `Integration: finance reading financials → 200`

#### 2.3 — Audit log with PROV-O semantics

**What**: Append-only `audit_log` capturing INSERT/UPDATE/DELETE with old/new values and actor.

**Design**: Table per data-model-suggestion-1 §"Audit Trail" (`old_values`/`new_values` JSONB, `changed_fields[]`, `user_id`, `ip_address`, `user_agent`). Implemented via SQLAlchemy session event listeners (`after_flush`) capturing dirty objects — no per-endpoint code. Range-partition by month. Endpoint `GET /organizations/{org_id}/audit?table=&record_id=&from=&to=`. Map fields to PROV-O (`prov:Activity`, `prov:Agent`, `prov:wasGeneratedBy`) in export.

**Testing**:
- `Integration: update asset.material → audit row with old/new value + changed_fields=['material']`
- `Integration: audit query by record_id returns full change history ordered by time`
- `Unit: PROV-O export of an audit row → valid JSON-LD with prov: terms`

### Definition of Done
Auth required on all non-public endpoints; RBAC enforced; every write produces an audit entry; partition rotation tested.

---

## Phase 3: Condition Inspection & Scoring Engine

### Purpose
Deliver the platform's analytical heart: pluggable multi-standard condition frameworks (PCI, NBI, ACI), inspection recording with distress observations and photo media, and computed condition scores that feed deterioration and planning. This is the primary value proposition and ships early per the planning principle.

### Tasks

#### 3.1 — Condition framework registry

**What**: `condition_frameworks`, `condition_rating_bands`, `distress_types` with seed data for PCI and NBI.

**Design**: Tables per data-model-suggestion-1 §"Condition Inspection System". `score_direction ∈ {asc,desc}` (PCI asc 0–100; NBI desc 0–9). Seed:
- **PCI (ASTM D6433)**: bands Good(86–100), Satisfactory(71–85), Fair(56–70), Poor(41–55), Very Poor(26–40), Serious(11–25), Failed(0–10); 19 asphalt + 20 concrete distress types with severity levels {low,medium,high}.
- **NBI (FHWA)**: 0–9 element ratings; bands Good(7–9), Fair(5–6), Poor(0–4).
- **ACI**: generic 0–100.
Frameworks are data, never hard-coded (per standards.md "fragmentation" note). `GET /condition-frameworks`, `POST` for custom org frameworks.

**Testing**:
- `Integration: PCI seed loads 7 bands + 39 distress types`
- `Unit: band lookup score=62 (PCI) → 'Fair'`
- `Unit: NBI score=4 (desc) → 'Poor'`

#### 3.2 — Pluggable scorer service (PCI deduct-value algorithm)

**What**: A `ConditionScorer` plugin interface with a correct ASTM D6433 PCI implementation.

**Design**:
```python
class ConditionScorer(Protocol):
    framework_code: str
    def score(self, observations: list[DistressObservation],
              sample_area_m2: float) -> ScoreResult: ...

@dataclass
class DistressObservation:
    distress_code: str
    severity: Literal["low","medium","high"]
    quantity: float            # m2 / linear m / count per distress unit
@dataclass
class ScoreResult:
    score: float
    rating_band: str
    deduct_values: dict[str, float]
    method: str
```
PCI: density% → deduct value (from committed `deduct_curves.json`) → corrected deduct value (CDV via max-CDV iteration) → `PCI = 100 - max_CDV`. NBI scorer = min element rating (configurable rollup). Registry `get_scorer(framework_code)`.

**Testing**:
- `Unit: known ASTM D6433 worked example → PCI within ±2 of published value (fixture-based)`
- `Unit: no distresses → PCI=100, band Good`
- `Unit: unknown framework_code → KeyError surfaced as 422`

#### 3.3 — Inspection recording, distresses & media

**What**: `inspections`, `inspection_distresses`, `inspection_media` with submit/review workflow and S3 photo upload.

**Design**: Tables per the data model. Inspection state machine: `draft → submitted → reviewed → approved | rejected`. On `approved`, the scorer computes `overall_score`/`rating_band` and a DB trigger (or service) updates `assets.current_condition_*`. Endpoints:
- `POST /assets/{id}/inspections` (draft) — `inspection_type`, `framework_code`, distresses[].
- `POST /inspections/{id}/media` — multipart upload → MinIO, store key + `taken_at` + `geom_point`.
- `POST /inspections/{id}/submit`, `/approve`, `/reject`.
Media stored in object storage; only references in DB.

**Testing**:
- `Integration: create inspection with PCI distresses → overall_score computed on approve`
- `Integration: approve inspection → assets.current_condition_score updated`
- `Integration: upload photo → object in MinIO, row in inspection_media`
- `Unit: reject from 'approved' state → 409 invalid transition`

### Definition of Done
PCI and NBI scoring verified against fixtures; inspection lifecycle enforced; approving an inspection updates denormalised asset condition; media in object storage.

---

## Phase 4: Work Order Management

### Purpose
Connect maintenance activity to assets so every work order updates service history and cost. Table-stakes for any EAM; depends on assets (Phase 1) and links to inspections (Phase 3).

### Tasks

#### 4.1 — Work orders, costs, labor

**What**: `work_orders`, `work_order_costs`, `work_order_labor`, with lifecycle and cost rollup to asset financials.

**Design**: Tables per data-model-suggestion-1 §"Work Order Management". State machine: `open → assigned → in_progress → (on_hold) → completed → closed | cancelled`. `work_type`, `priority` enums. Endpoints `POST/GET/PATCH /assets/{id}/work-orders`, `/work-orders/{id}/costs`, `/work-orders/{id}/labor`, `/work-orders/{id}/transition`. Completing a WO sums costs into `asset_financials.total_maintenance_cost`. WO may reference triggering `inspection_id`.

**Testing**:
- `Unit: invalid transition open→closed → 409`
- `Integration: complete WO with costs → asset total_maintenance_cost increases by sum`
- `Integration: WO created from inspection links inspection_id`

#### 4.2 — Preventive maintenance templates & scheduling

**What**: `work_order_templates` with iCal RRULE recurrence and a Celery beat job that generates due WOs.

**Design**: Template holds `recurrence_rule` (RFC 5545 RRULE), `task_checklist[]`, estimates. Celery beat task `generate_pm_work_orders` runs daily, expands RRULEs against last-generated dates, creates WOs for matching assets (by `asset_class_id`).

**Testing**:
- `Unit: RRULE FREQ=MONTHLY expands to correct next date`
- `Integration (mocked clock): beat run on due date → WO generated; non-due date → none`

### Definition of Done
WO lifecycle and costing work end-to-end; PM scheduler generates WOs; asset financial rollups correct.

---

## Phase 5: GIS Interface — OGC API Features & Vector Tiles

### Purpose
Expose assets through the standards-compliant spatial API (OGC API Features) and serve vector tiles for network-level map rendering — the vendor-neutral GIS capability that is the project's flagship differentiator. Can be developed in parallel with Phases 4 and 6 once Phase 1 lands.

### Tasks

#### 5.1 — OGC API Features endpoints

**What**: Implement OGC API Features Part 1 (Core) read and Part 4 (CRUD) for asset collections.

**Design**: Per standards.md. Endpoints:
- `GET /ogc/` (landing), `GET /ogc/conformance`, `GET /ogc/collections`, `GET /ogc/collections/{collectionId}` (one collection per asset class), `GET /ogc/collections/{collectionId}/items?bbox=&datetime=&limit=&...`, `GET .../items/{featureId}`.
- Part 4: `POST/PUT/PATCH/DELETE .../items` (RBAC-gated).
- Responses are GeoJSON `FeatureCollection`/`Feature` (RFC 7946); `numberMatched`/`numberReturned`; RFC 8288 `Link` rels (`self`, `next`, `alternate`). Content negotiation JSON/HTML.
- OpenAPI 3.1 description served at `/ogc/api`.

**Testing**:
- `Integration: GET /ogc/conformance → lists Core + CRUD conformance class URIs`
- `Integration: items?bbox=... → valid GeoJSON FeatureCollection, geometries within bbox`
- `Integration: POST item as inspector → 201; as citizen → 403`
- `Validation: response validates against OGC API Features OpenAPI schema (fixture)`

#### 5.2 — Vector tile service & GML import/export

**What**: MVT tiles from PostGIS and GML 3.2 import/export for legacy interop.

**Design**: `GET /tiles/{collection}/{z}/{x}/{y}.pbf` via `ST_AsMVT`/`ST_AsMVTGeom`, condition-coloured by rating band. `geo/` service: GeoJSON↔GML (ISO 19136) using ogr2ogr bindings or `lxml`; `POST /import/gml`, `GET /export/gml?collection=`.

**Testing**:
- `Integration: tile request → valid MVT decodable by mapbox-vector-tile`
- `Unit: GeoJSON LineString → GML 3.2 → GeoJSON round-trips`

### Definition of Done
OGC conformance classes advertised and honoured; tiles render in MapLibre; GML round-trips.

---

## Phase 6: Capital Planning, Deterioration & GASB 34 Reporting

### Purpose
Turn condition + financial data into defensible investment decisions: deterioration forecasting, scenario modelling with cost-of-deferral, capital projects, and automated GASB 34 reports — addressing the documented "modified approach tooling gap". Requires Phases 1, 3.

### Tasks

#### 6.1 — Asset financials & GASB 34 records

**What**: `asset_financials`, `gasb34_condition_assessments`, depreciation engine.

**Design**: Tables per data-model-suggestion-1 §"Financial and Capital Planning". Straight-line + declining-balance depreciation service computing `accumulated_depreciation`/`net_book_value`. `gasb34_condition_assessments` with generated `condition_met`. Endpoints under `/organizations/{org_id}/financials` and `/gasb34/assessments`.

**Testing**:
- `Unit: straight-line, cost=1M, life=40 → annual=25k; year 10 net book=750k`
- `Integration: assessment actual≥target → condition_met true`

#### 6.2 — Deterioration models & scenario engine

**What**: `deterioration_models` and `planning_scenarios` with a multi-year simulation.

**Design**: Model types {linear, exponential, markov, weibull}; parameters in JSONB; fitted from `inspection` history per asset class (least-squares for linear/exponential). Scenario engine projects condition forward over `analysis_period_years` under an `annual_budget`, applying treatments by priority, computing `total_backlog_end`, `avg_condition_end`, and `cost_of_deferral` (do-nothing backlog minus funded backlog). Long runs dispatched to Celery. `POST /scenarios/{id}/run` → async; `GET /scenarios/{id}` returns results.

**Testing**:
- `Unit: linear model fit to declining scores → negative slope coefficient`
- `Unit: scenario with $0 budget → backlog grows, avg condition declines monotonically`
- `Integration: run scenario → status running→completed, results populated`

#### 6.3 — Capital projects & prioritisation

**What**: `capital_projects`, `capital_project_assets`, priority scoring.

**Design**: Tables per the data model. Priority score = f(condition, criticality, risk_score, cost_of_deferral). Endpoints `/capital-projects` CRUD + `POST /capital-projects/prioritise` returning a ranked CIP candidate list.

**Testing**:
- `Integration: link assets to project → costs aggregate`
- `Unit: prioritisation ranks low-condition high-criticality asset above good-condition asset`

#### 6.4 — GASB 34 & grant report generation

**What**: Server-rendered PDF reports for GASB 34 (both approaches) and grant applications.

**Design**: Jinja2 HTML templates → WeasyPrint PDF. `POST /reports/gasb34?year=&approach=depreciation|modified` builds from financials/assessments. Uses `v_gasb34_condition_summary` view. Grant template pulls condition + capital plan. Reports queued via Celery, retrievable from object storage.

**Testing**:
- `Integration: GASB34 modified-approach report → PDF with target vs actual condition table, compliance status`
- `Integration: depreciation-approach report → net book value rollup by class`

### Definition of Done
Depreciation/scenario math verified; GASB 34 PDFs generate for both approaches; capital prioritisation ranks defensibly.

---

## Phase 7: AI-Native Capabilities

### Purpose
Deliver the explainable, vendor-neutral AI that differentiates the platform: computer-vision condition scoring from photos, natural-language asset querying, AI report drafting, and NL work-order creation. Requires Phases 3, 4, 6.

### Tasks

#### 7.1 — Computer-vision condition assessment

**What**: Detect pavement/structure distresses in inspection photos and produce an explainable AI score with confidence and bounding boxes, flagged for human review.

**Design**: Celery task `assess_condition_cv(media_id)`: load image from S3 → YOLO distress-detection model → map detections to framework `distress_types` → run the same PCI/NBI scorer → write `inspection_media.ai_score`, `ai_confidence`, `ai_analysis` (detected distresses + bounding boxes). `requires_human_review=true` when confidence < threshold. Inspector can accept/override (transparency requirement — they can challenge the AI).

**Testing**:
- `Unit (mocked model): detections → mapped distresses → scorer → ai_score set`
- `Integration: low-confidence result → flagged for review, not auto-approved`

#### 7.2 — Natural-language query & MCP server

**What**: NL → safe parameterised query over assets/conditions/work orders, plus an MCP server exposing registry tools to AI agents.

**Design**: `POST /ai/query {question}` → LLM (LiteLLM) with a tool/function schema mapping to whitelisted, parameterised query builders (never raw SQL) — mitigates injection. Returns structured results + a plain-language summary. MCP server (`mcp/`) exposes tools `search_assets`, `get_asset_condition_history`, `list_open_work_orders`, `get_capital_plan`, `generate_report` over stdio/HTTP for Claude and other agents.

System prompt template (NL query):
```
You translate municipal asset questions into calls to the provided tools.
Never invent asset numbers. Only use tool outputs. If the question is
ambiguous, ask one clarifying question. Always cite asset_number and
condition framework in answers.
```

**Testing**:
- `Integration (mocked LLM): "show roads below PCI 40" → calls search_assets(class=road, max_score=40)`
- `Unit: attempted SQL injection in question → no raw SQL executed (tool-only path)`
- `Integration: MCP search_assets tool returns GeoJSON-free structured rows`

#### 7.3 — AI report drafting & NL work-order creation

**What**: Plain-language report synthesis for non-technical stakeholders and WO creation from field voice/text notes.

**Design**: `POST /ai/reports/summarise` takes a dataset (e.g. a GASB result or scenario) → LLM drafts a councillor-friendly narrative with explicit figures (no hallucinated numbers — values injected, not generated). `POST /ai/work-orders/from-note {text}` → LLM extracts `{asset hint, work_type, priority, title, description}`, resolves asset by spatial/text match, creates a draft WO for confirmation.

**Testing**:
- `Integration (mocked LLM): scenario data → summary contains the exact backlog figure passed in`
- `Integration: note "pothole on Main St, urgent" → draft WO work_type=corrective, priority=high, asset matched`

### Definition of Done
CV scoring is explainable and review-gated; NL query is injection-safe and tool-bounded; MCP server passes a reference client handshake; AI reports never fabricate figures.

---

## Phase 8: Import, Migration & GIS Sync

### Purpose
Lower the adoption barrier with AI-assisted migration of legacy spreadsheets/GIS and scheduled sync from external GIS sources — directly targeting the "data import and cleansing assistance" gap and reducing implementation time.

### Tasks

#### 8.1 — AI-assisted spreadsheet/CSV import wizard

**What**: Upload a CSV/XLSX, AI proposes column→field mappings, user confirms, rows import with lineage.

**Design**: `import_batches` per the data model. `POST /import/analyse` (file) → sniff headers + sample rows → LLM proposes mapping to `assets` + `asset_class_attributes` fields with confidence. `POST /import/commit {batch_id, mapping}` → Celery validates & inserts via staging, recording `success_count`/`error_count`/`error_log`.

**Testing**:
- `Integration (mocked LLM): messy headers → proposed mapping covers required asset fields`
- `Integration: commit with 2 invalid rows → success_count=N-2, error_log lists rows`

#### 8.2 — External GIS source sync

**What**: `gis_data_sources` + scheduled pull from OGC API Features / WFS / GeoJSON / shapefile / GeoPackage.

**Design**: Table per the data model; Celery beat reads `sync_schedule` cron, fetches features, upserts assets matched by `external_id` or spatial proximity (`ST_DWithin`). Supports `ogc_api_features`, `wfs`, `geojson_url`, `shapefile`, `geopackage` (via GDAL/ogr2ogr).

**Testing**:
- `Integration (mocked OGC server): sync → new assets created, existing matched by external_id updated`
- `Unit: cron schedule parses; due/not-due correctly evaluated`

### Definition of Done
CSV and GIS imports create assets with traceable batches; scheduled sync upserts idempotently.

---

## Phase 9: Web Dashboard & Map UI

### Purpose
Deliver the map-centric web application — the primary surface for managers, finance, and supervisors — consuming the API and OGC/tile services. Requires Phases 1–6; parallelable with Phase 10.

### Tasks

#### 9.1 — App shell, auth, generated API client

**What**: React+Vite shell with OIDC/OAuth login, routing, and a typed client generated from `openapi.json`.

**Design**: Login via `POST /auth/token`/OIDC; token in memory + refresh cookie. `openapi-typescript` generates the client. Role-aware navigation.

**Testing**:
- `E2E (Playwright): login → dashboard; expired token → re-auth`

#### 9.2 — Map view, asset detail, inspection & work-order screens

**What**: MapLibre map with condition-coloured layers from vector tiles, asset detail panel, inspection and WO management.

**Design**: MapLibre renders `/tiles/...`; click feature → asset detail (condition history chart, financials, WOs). Inspection form mirrors framework distress types. WO board by status.

**Testing**:
- `E2E: click asset on map → detail panel with current condition`
- `E2E: create work order from asset → appears on WO board`

#### 9.3 — Capital planning, GASB dashboards & NL query box

**What**: Scenario runner UI, GASB 34 dashboard, and an NL query input.

**Design**: Scenario form → `POST /scenarios/run`, poll, chart backlog/condition curves. GASB dashboard from `v_gasb34_condition_summary`. NL query box → `/ai/query`, render summary + results table.

**Testing**:
- `E2E: run scenario → condition/backlog chart renders`
- `E2E (mocked AI): NL query returns table + summary`

### Definition of Done
Manager workflows usable end-to-end in browser; map performant at network scale via tiles.

---

## Phase 10: Mobile Inspection App, Citizen Portal & Hardening

### Purpose
Complete the field-to-citizen loop with an offline-capable mobile inspection app and a public request portal, then harden for production. Mobile app and portal parallelable; hardening last.

### Tasks

#### 10.1 — Offline mobile inspection app

**What**: React Native (Expo) app: download assigned assets, record inspections + photos offline, sync on reconnect.

**Design**: WatermelonDB local store mirrors assets/inspection schema; photos cached locally then uploaded to `/inspections/{id}/media` on sync. Conflict policy: server timestamps win; local drafts never lost. Targets the incumbent reliability gap (stable, low-battery, simple).

**Testing**:
- `Integration: airplane-mode inspection → queued; on reconnect → synced, media uploaded`
- `E2E (Detox): capture photo offline → appears server-side after sync`

#### 10.2 — Citizen request portal

**What**: Public `citizen_requests` submission with photo + location, spatial auto-link to nearest asset, status tracking.

**Design**: Tables per data-model-suggestion-1 §"Citizen and Stakeholder Access". `POST /public/requests` (unauthenticated, rate-limited, captcha) → store, `ST_DWithin` link to nearest asset, optionally spawn a WO. Public status lookup by `request_number`. Community transparency dashboard (condition + planned works).

**Testing**:
- `Integration: request near a road → asset_id linked by proximity`
- `Integration: rate limit exceeded → 429`

#### 10.3 — Production hardening

**What**: Rate limiting, security headers, NIST/OWASP review, observability, backups, deployment manifests.

**Design**: Per-IP/-token rate limiting (Redis); OWASP API Top-10 checklist pass; structured logging + OpenTelemetry traces; `pgBackRest`/WAL archiving docs; Docker healthchecks; optional FIPS-validated crypto note for federal deployments; secrets via env/secret manager.

**Testing**:
- `Integration: burst requests → 429 after threshold`
- `Security: automated OWASP API checks (e.g., authz fuzzing) pass`
- `Integration: restore from backup → data intact`

### Definition of Done
Offline mobile inspection round-trips; citizen requests link to assets; rate-limiting, observability, and backup/restore verified; OWASP checklist clean.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Core Asset Registry      ─── required by everything
    │
Phase 2: Auth, RBAC & Audit                     ─── requires Phase 1
    │
Phase 3: Condition Inspection & Scoring         ─── requires Phase 1 (uses 2)   ◀ core value
    │
    ├── Phase 4: Work Order Management           ─── requires 1,3
    ├── Phase 5: GIS / OGC API & Vector Tiles    ─── requires 1   (parallel with 4,6)
    └── Phase 6: Capital Planning / GASB 34      ─── requires 1,3 (parallel with 4,5)
         │
Phase 7: AI-Native Capabilities                 ─── requires 3,4,6
Phase 8: Import, Migration & GIS Sync           ─── requires 1,5 (parallel with 7)
    │
Phase 9: Web Dashboard & Map UI                 ─── requires 1-6 (parallel with 10)
Phase 10: Mobile App, Citizen Portal, Hardening ─── requires 3,4 (+ portal needs 5)
```

**Parallelism opportunities**:
- After Phase 3: Phases 4, 5, and 6 can be built concurrently.
- Phase 7 (AI) and Phase 8 (import/sync) can proceed concurrently.
- Phase 9 (web) and Phase 10 (mobile/portal) can proceed concurrently once their backend dependencies land.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`), including testcontainers PostGIS integration tests.
3. Linting and formatting pass (`ruff check`, `ruff format --check`).
4. Type checking passes (`mypy src/`).
5. Docker build succeeds; `docker compose up` brings the stack to healthy.
6. The phase's feature works end-to-end (demonstrable via API call, E2E test, or UI).
7. New configuration options documented in `README.md` / `.env.example`.
8. New API endpoints appear in the auto-generated OpenAPI 3.1 spec (`docs/openapi.json` regenerated in CI).
9. Database migrations created with Alembic and verified to upgrade/downgrade cleanly.
10. New writes produce audit-log entries (from Phase 2 onward).
11. Standards conformance checked where applicable (OGC conformance classes, GeoJSON validity, OpenAPI validity, GASB report contents).
```
