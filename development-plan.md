# Data Retention & Archiving — Phased Development Plan

> Project: 434-data-retention-archiving · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. It targets the MVP and v1.1 scope defined in `features.md`: a policy engine, legal hold management, core connectors (Microsoft 365, Google Workspace, local file storage), WORM-compliant immutable storage, full-text search, defensible disposition, and compliance reporting — with AI-native classification, hold scoping, and disposition narratives layered on top as the differentiating advantage.

**Data model decision:** This plan adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB on PostgreSQL)** as the primary schema. It gives full relational integrity and indexing for the stable compliance entities (policies, holds, dispositions, audit) while absorbing the heterogeneous, per-connector metadata into `JSONB` columns — avoiding both the table explosion of the fully normalised model (DM1) and the operational weight of event-sourcing (DM2) or the multi-engine lakehouse (DM4). The tamper-evident hash-chained `audit_log` and `deletion_certificates` patterns are taken from DM1/DM3 verbatim. DM4's lakehouse and DM2's event store are noted as deferred scaling paths, not MVP architecture.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **Python 3.12** | The differentiating value is AI-native classification, hold scoping, and disposition narratives — all LLM- and NLP-heavy. Python has the strongest LLM/embedding ecosystem (`anthropic`, `openai`, `sentence-transformers`) plus mature SDKs for every required connector (`msgraph-sdk`, `google-api-python-client`, `boto3`). |
| API framework | **FastAPI** | Async-first (needed for connector I/O and long-running ingestion), generates an **OpenAPI 3.1** spec automatically (a stated v1.1 requirement and a `standards.md` target), and integrates Pydantic v2 for request/response validation. |
| Data validation / models | **Pydantic v2** | Single source of truth for API schemas, connector config schemas, and JSONB payload validation mirrored from the DB `pg_jsonschema` constraints. |
| Database | **PostgreSQL 16** | Mandated by the chosen data model. Provides JSONB + GIN indexing, declarative LIST/RANGE partitioning, Row-Level Security for multi-tenancy, native FTS for the MVP, and `pg_jsonschema` for JSONB validation. Single engine = one operational runbook. |
| DB access | **SQLAlchemy 2.0 (async) + asyncpg** | Async ORM with explicit Core/typed queries for the hot paths (record ingestion, disposition scans). Avoids hand-rolled SQL while keeping raw SQL escape hatches for JSONB GIN queries. |
| Migrations | **Alembic** | Native to the SQLAlchemy ecosystem; versioned, reviewable, supports the partition + RLS + trigger DDL the schema requires. (`standards.md`/DM3 suggested Flyway, but Alembic keeps the stack single-language.) |
| Task queue | **Celery + Redis** | Connector syncs, AI classification, disposition execution, hold placement, and report generation are all long-running async jobs that must survive process restarts and retry idempotently. Redis doubles as the broker and a result/cache backend. |
| Scheduler | **Celery Beat** | Periodic retention-expiry scans, scheduled compliance reports, custodian reminder escalation, and incremental connector sync polling. |
| Object storage | **S3 API with Object Lock**; **MinIO** for self-hosted/dev | WORM compliance for SEC 17a-4 / FINRA / CFTC 1.31. A single `S3Client` abstraction targets AWS S3, Azure Blob (S3-compat layer), or MinIO so deployment mode is config-driven. COMPLIANCE and GOVERNANCE lock modes both supported. |
| Full-text search | **PostgreSQL FTS (MVP)** → **OpenSearch (scale)** | Graduated approach from DM3. The search layer is abstracted behind a `SearchBackend` interface so the OpenSearch swap is additive, not a rewrite. |
| Trusted timestamping | **RFC 3161 TSA client** | Disposition certificates and archived-record hashes are anchored to an external RFC 3161 Time-Stamp Authority for legal admissibility (per `standards.md`). |
| Auth (platform) | **OAuth 2.0 / OIDC** via `authlib`; JWT sessions | Compliance/legal/IT/auditor RBAC. Custodian self-service portal uses OIDC. |
| Auth (connectors) | **OAuth 2.0 client-credentials / 3-legged** | Required by Microsoft Graph and Google Workspace APIs (`standards.md`). Tokens stored encrypted. |
| Secrets / encryption | **Envelope encryption** (app-layer AES-256-GCM) + KMS-pluggable | Connector credentials and `connection_config` JSONB encrypted at rest; per-tenant data keys. Pluggable KMS (AWS KMS, Vault, local dev key). |
| LLM provider | **Anthropic Claude** via `anthropic` SDK, provider-abstracted | Powers classification, hold scoping, disposition narratives, policy authoring, conflict resolution. Abstracted behind an `LLMProvider` interface so OpenAI/local models can be substituted. Prompt caching enabled. |
| Embeddings | **sentence-transformers (local)** or provider embeddings | Semantic similarity for hold-scope custodian/source suggestion and near-dedup. Local model keeps PII out of third-party APIs by default. |
| MCP server | **`mcp` Python SDK** | Exposes query/policy-coverage/hold/report tools to AI assistants — the `standards.md`-flagged differentiation opportunity. |
| Frontend | **React 18 + TypeScript + Vite**, **shadcn/ui + Tailwind** | Admin dashboard (policies, holds, dispositions, search, reporting) and custodian portal. SPA talking to the REST API. |
| Containerisation | **Docker + docker-compose** | Self-hosted is a first-class deployment mode (README). Compose wires Postgres, Redis, MinIO, API, worker, beat, and web. |
| Testing | **pytest + pytest-asyncio + testcontainers** | Real Postgres/MinIO/Redis via testcontainers for integration tests; mocked connectors and LLM for unit tests. `vitest` + React Testing Library for frontend. |
| Code quality | **ruff** (lint+format), **mypy** (strict), **pre-commit** | Enforced in CI. Frontend: `eslint` + `prettier` + `tsc`. |
| Package manager | **uv** (Python), **pnpm** (frontend) | Fast, reproducible lockfiles. |
| CI | **GitHub Actions** | Lint → type-check → unit → integration (testcontainers) → docker build → OpenAPI spec diff. |

### Project Structure

```
data-retention-archiving/
├── pyproject.toml                  # uv-managed; deps + ruff + mypy config
├── uv.lock
├── Dockerfile                      # multi-stage: api / worker images
├── docker-compose.yml              # postgres, redis, minio, api, worker, beat, web
├── docker-compose.dev.yml          # dev overrides (volume mounts, hot reload)
├── alembic.ini
├── .pre-commit-config.yaml
├── .github/workflows/ci.yml
├── README.md
├── migrations/                     # Alembic versions (schema, partitions, RLS, triggers)
│   ├── env.py
│   └── versions/
├── src/
│   └── dra/                        # "data retention & archiving" package
│       ├── __init__.py
│       ├── config.py               # Pydantic Settings (env-driven)
│       ├── main.py                 # FastAPI app factory
│       ├── db/
│       │   ├── engine.py           # async engine, session, RLS context manager
│       │   ├── models/             # SQLAlchemy ORM models (one module per domain)
│       │   │   ├── org.py  policy.py  record.py  hold.py
│       │   │   ├── disposition.py  ediscovery.py  audit.py  report.py
│       │   └── partitions.py       # tenant + monthly partition management
│       ├── schemas/                # Pydantic request/response + JSONB payload models
│       │   ├── policy.py  hold.py  record.py  disposition.py  search.py ...
│       │   └── jsonb/              # per-connector source_metadata schemas
│       ├── api/
│       │   ├── deps.py             # auth, RBAC, tenant context, pagination
│       │   ├── routers/            # one router per resource
│       │   │   ├── policies.py  holds.py  records.py  search.py
│       │   │   ├── disposition.py  reports.py  sources.py  custodians.py
│       │   │   └── auth.py  custodian_portal.py
│       │   └── errors.py           # RFC 7807 problem+json handlers
│       ├── services/               # business logic (framework-agnostic)
│       │   ├── policy_engine.py    # assignment, expiry computation, conflict detection
│       │   ├── retention.py        # expiry scan, lifecycle transitions
│       │   ├── holds.py            # placement, release, custodian orchestration
│       │   ├── disposition.py      # batch build, approval chain, defensible delete
│       │   ├── search.py           # query compilation against SearchBackend
│       │   ├── ediscovery.py       # collections, productions, chain of custody
│       │   ├── classification.py   # AI data-class assignment
│       │   ├── hold_scoping.py     # AI custodian/source suggestion
│       │   ├── narrative.py        # AI disposition narrative + policy authoring
│       │   ├── conflict.py         # AI multi-jurisdiction conflict resolution
│       │   └── reporting.py        # report generation + coverage-gap detection
│       ├── connectors/
│       │   ├── base.py             # Connector ABC + capability descriptors
│       │   ├── registry.py         # source_type -> connector class
│       │   ├── m365.py  gworkspace.py  local_fs.py        # MVP
│       │   ├── slack.py  gdrive.py  box.py  dropbox.py  salesforce.py  # v1.1
│       ├── storage/
│       │   ├── backend.py          # S3Client (Object Lock), tier management
│       │   └── crypto.py           # envelope encryption, KMS providers
│       ├── search_backend/
│       │   ├── base.py             # SearchBackend interface
│       │   ├── postgres.py         # FTS implementation (MVP)
│       │   └── opensearch.py       # scale implementation (v1.1+)
│       ├── audit/
│       │   ├── chain.py            # HMAC-SHA256 hash-chain + verification
│       │   └── timestamp.py        # RFC 3161 TSA client
│       ├── llm/
│       │   ├── provider.py         # LLMProvider ABC (Claude default)
│       │   ├── prompts/            # versioned prompt templates
│       │   └── embeddings.py
│       ├── tasks/                  # Celery task definitions
│       │   ├── celery_app.py  sync.py  classify.py  disposition.py
│       │   ├── holds.py  reports.py  beat_schedule.py
│       ├── mcp/
│       │   └── server.py           # MCP server exposing read/hold/report tools
│       └── templates/              # email / notification / certificate templates
├── web/                            # React + TS admin dashboard + custodian portal
│   ├── package.json  vite.config.ts  tsconfig.json
│   └── src/{pages,components,api,hooks,lib}/
└── tests/
    ├── conftest.py                 # testcontainers fixtures (pg, minio, redis)
    ├── unit/  integration/  e2e/  fixtures/
    └── fixtures/{emls,docs,slack_exports,policies}/
```

The structure is grouped by concern (connectors, services, storage, search, llm) so each phase adds modules without restructuring. Connectors, search backends, LLM providers, and KMS providers are all behind interfaces, so new implementations are additive.

---

## Phase 1: Foundation — Project Skeleton, Config, Database, Auth, Audit Spine

### Purpose
Establish the runnable skeleton every later phase builds on: the FastAPI app, configuration, the PostgreSQL schema with multi-tenant RLS, the tamper-evident audit log, authentication/RBAC, and CI. After this phase the platform boots, authenticates a user, isolates tenants at the database level, and writes verifiable hash-chained audit entries — the compliance backbone everything else depends on.

### Tasks

#### 1.1 — Project scaffold, config, and tooling

**What**: A `uv`-managed Python package that boots a FastAPI app with health checks, structured logging, and environment-driven settings.

**Design**:
- `dra/config.py` using `pydantic-settings`:
```python
class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="DRA_", env_file=".env")
    environment: Literal["dev", "staging", "prod"] = "dev"
    database_url: PostgresDsn
    redis_url: RedisDsn
    s3_endpoint: str | None = None        # None => AWS; set for MinIO
    s3_region: str = "us-east-1"
    s3_access_key: SecretStr
    s3_secret_key: SecretStr
    object_lock_mode: Literal["COMPLIANCE", "GOVERNANCE"] = "COMPLIANCE"
    kms_provider: Literal["local", "aws", "vault"] = "local"
    master_key: SecretStr                  # local KMS dev key
    llm_provider: Literal["anthropic", "openai"] = "anthropic"
    anthropic_api_key: SecretStr | None = None
    jwt_secret: SecretStr
    tsa_url: str | None = None             # RFC 3161 endpoint
```
- `dra/main.py` exposes `create_app() -> FastAPI`; mounts `/healthz` (liveness) and `/readyz` (DB + Redis + S3 ping).
- Structured JSON logging via `structlog`; request-ID middleware.
- `pyproject.toml` configures ruff (line length 100, isort), mypy (`strict = true`), pytest.

**Testing**:
- `Unit: Settings loads from env vars with DRA_ prefix → correct typed values`
- `Unit: missing required DRA_DATABASE_URL → ValidationError naming the field`
- `Integration: GET /healthz → 200 {"status":"ok"}`
- `Integration (testcontainers): GET /readyz with all deps up → 200; with Redis down → 503 listing the failed dependency`

#### 1.2 — Core database schema, partitioning, and RLS (Alembic)

**What**: The full Phase-1 schema (organisations, users, data_sources, retention_policies, regulatory_frameworks) plus tenant LIST-partitioning helpers and Row-Level Security, as Alembic migrations.

**Design**:
- Implement tables exactly per **DM3**: `organisations`, `users` (role CHECK constraint over the seven roles), `data_sources` (relational sync fields + JSONB `connection_config`/`sync_state`/`capabilities`), `regulatory_frameworks` (JSONB `requirements`), `retention_policies` (relational core + JSONB `scope_conditions`/`approval_workflow`/`jurisdiction_overrides`).
- Enable `pgcrypto`, `pg_jsonschema` extensions.
- RLS: every tenant table gets `ENABLE ROW LEVEL SECURITY` and a policy `USING (organisation_id = current_setting('app.current_org_id')::UUID)`.
- `dra/db/engine.py` provides:
```python
@asynccontextmanager
async def tenant_session(org_id: UUID | None) -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as s:
        if org_id:
            await s.execute(text("SET LOCAL app.current_org_id = :oid"), {"oid": str(org_id)})
        yield s
```
- `dra/db/partitions.py`: `create_tenant_partition(org_id)` creates `archived_records` LIST partition + first monthly RANGE sub-partition (used in Phase 3; stubbed here).
- Seed migration loads system regulatory frameworks: GDPR, HIPAA, SOX, PCI-DSS, FINRA, SEC_17A4, CFTC_1_31, DPDPA with their `requirements` JSONB and citations from `standards.md`.

**Testing**:
- `Integration (testcontainers): alembic upgrade head → all tables, extensions, RLS policies present (query pg_policies, pg_extension)`
- `Integration: insert two orgs; set app.current_org_id to org A; SELECT users → only org A rows returned`
- `Integration: attempt cross-tenant UPDATE under org A's RLS context → 0 rows affected`
- `Unit: retention_policies insert with invalid scope_conditions shape → CHECK/pg_jsonschema violation`
- `Integration: seed migration → 8 regulatory_frameworks rows with correct codes and JSONB requirements`

#### 1.3 — Tamper-evident audit log with hash chaining

**What**: An append-only, monthly-partitioned `audit_log` table with HMAC-SHA256 hash chaining and a verifiable chain, plus an `AuditWriter` service.

**Design**:
- Table per DM3 §8: relational filter columns + JSONB `event_details` + `previous_hash`/`entry_hash`; `PARTITION BY RANGE (occurred_at)`; UPDATE/DELETE-blocking triggers (`prevent_audit_modification`).
- `dra/audit/chain.py`:
```python
def compute_entry_hash(prev_hash: bytes | None, entry: AuditEntry, hmac_key: bytes) -> bytes:
    payload = canonical_json({
        "organisation_id": str(entry.org_id), "event_type": entry.event_type,
        "actor_id": str(entry.actor_id) if entry.actor_id else None,
        "target_type": entry.target_type, "target_id": str(entry.target_id),
        "summary": entry.summary, "details": entry.details,
        "occurred_at": entry.occurred_at.isoformat(),
        "previous_hash": prev_hash.hex() if prev_hash else "",
    })
    return hmac.new(hmac_key, payload, hashlib.sha256).digest()

async def verify_chain(session, org_id: UUID, hmac_key: bytes) -> ChainVerifyResult: ...
```
- `AuditWriter.record(event_type, category, actor, target, summary, details)` fetches the latest `entry_hash` per org under an advisory lock (serialises chain writes), computes the new hash, inserts. Categories/event_types enumerated as constants (`record.ingested`, `hold.placed`, `policy.created`, `disposition.certificate_issued`, ...).
- HMAC key sourced from KMS (Phase 1 uses local key); document that key compromise breaks non-repudiation, mitigated by periodic Merkle-root anchoring to RFC 3161 TSA (Phase 7).

**Testing**:
- `Unit: compute_entry_hash is deterministic for identical input; differs when any field changes`
- `Integration: write 3 sequential audit entries → each previous_hash equals prior entry_hash`
- `Integration: verify_chain over a clean chain → valid=True`
- `Integration: directly corrupt one entry's summary via superuser bypass → verify_chain returns valid=False with the breaking sequence number`
- `Integration: UPDATE audit_log row → raises "Audit log is append-only"`

#### 1.4 — Authentication, RBAC, and tenant context middleware

**What**: OIDC/JWT login, role-based dependency guards, and middleware that resolves the tenant and sets the RLS context per request.

**Design**:
- `dra/api/routers/auth.py`: `/auth/login` (OIDC code exchange via `authlib`) and password fallback for local admin; issues JWT `{sub, org_id, role, exp}`.
- `dra/api/deps.py`:
```python
async def current_user(token: str = Depends(bearer)) -> AuthUser: ...
def require_role(*roles: Role) -> Callable: ...   # 403 if user.role not in roles
async def tenant_db(user=Depends(current_user)) -> AsyncSession:  # opens tenant_session(user.org_id)
```
- RBAC matrix encoded as a table: e.g. policy create/approve → `compliance_admin`; hold issue/release → `legal_admin`; disposition execute → `compliance_admin` + `records_manager`; read-only → `auditor`.
- Every state-changing endpoint emits an audit entry via `AuditWriter`.

**Testing**:
- `Unit: require_role(legal_admin) with compliance_admin token → HTTPException 403`
- `Integration: login with valid OIDC mock → JWT with correct org_id and role`
- `Integration: request with expired JWT → 401`
- `Integration: compliance_admin creates policy → 201 and audit_log has policy.created entry with actor_id`

#### 1.5 — Docker, compose, and CI

**What**: Containerisation and a green CI pipeline.

**Design**:
- Multi-stage `Dockerfile` (builder installs via `uv`; runtime slim image; same image runs api/worker by command).
- `docker-compose.yml`: `postgres:16`, `redis:7`, `minio` (with `mc` init creating a bucket with Object Lock enabled), `api`, `worker`, `beat`, `web`. Healthchecks gate startup ordering.
- `.github/workflows/ci.yml`: ruff → mypy → pytest unit → pytest integration (testcontainers) → docker build → export OpenAPI and diff against committed `openapi.json`.

**Testing**:
- `E2E: docker compose up → /readyz returns 200 within 60s`
- `CI: ruff and mypy pass with zero findings on the skeleton`
- `Integration: MinIO bucket created with ObjectLockEnabled=Enabled (head-bucket object-lock-configuration)`

---

## Phase 2: Retention Policy Engine

### Purpose
Implement the heart of proactive compliance: defining retention schedules, computing per-record expiry, resolving which policy governs a record, and detecting policy conflicts. This is one half of the core value proposition and is buildable before any data is ingested (policies are evaluated against records in Phase 3, but the engine, API, and conflict logic stand alone and are unit-testable with synthetic records).

### Tasks

#### 2.1 — Policy CRUD, templates, and lifecycle

**What**: REST endpoints to create, version, activate, suspend, and retire retention policies, plus pre-built compliance templates.

**Design**:
- Pydantic `RetentionPolicyIn`/`Out` mirroring DM3 `retention_policies`, including typed `ScopeConditions` (data_classes, source_types, business_units, jurisdictions, content_types, `custom_conditions: list[Condition]`) and `ApprovalWorkflow` (ordered stages by role with timeout_days, escalation_policy, minimum_approvals).
- `Condition` model: `{field: str, operator: Literal["equals","not_equals","greater_than","less_than","contains","in"], value: Any}` — `field` is a dotted path into record columns or `source_metadata`/`enrichment_data`.
- Endpoints:
  - `POST /policies` (draft) · `PATCH /policies/{id}` · `POST /policies/{id}/submit` · `POST /policies/{id}/approve` · `POST /policies/{id}/suspend` · `POST /policies/{id}/retire`
  - `GET /policies` (filter by status/framework) · `GET /policies/{id}`
  - `GET /policy-templates` · `POST /policies/from-template/{code}`
- Templates seeded for GDPR (storage limitation, erasure-aware), HIPAA (6yr docs, configurable medical-record period), SOX (7yr), PCI-DSS, FINRA (6yr, WORM), SEC 17a-4 (3/6yr, immediate-access window), DPDPA — each pre-fills `retention_period_days`, `disposition_action`, `jurisdiction_overrides`, and `regulatory_citation`.
- Status transitions enforced by a state machine: `draft → pending_review → active → {suspended → active, retired}`.

**Testing**:
- `Unit: state machine rejects draft → active (must pass through pending_review/approve)`
- `Unit: ScopeConditions with unknown operator → ValidationError`
- `Integration: POST /policies/from-template/GDPR → policy with retention_period_days and citation populated`
- `Integration: approve by compliance_admin → status active, approved_by/approved_at set, audit entry policy.approved`
- `Integration: non-compliance_admin approve → 403`

#### 2.2 — Expiry computation and policy assignment

**What**: A `PolicyEngine` that, given a record, determines all matching policies, computes expiry per the longest-retention-wins rule, and writes `record_policy_assignments`.

**Design**:
```python
class PolicyEngine:
    def matches(self, policy: RetentionPolicy, record: RecordView) -> MatchResult:
        # evaluate scope_conditions (class/source/jurisdiction/content_type) AND custom_conditions
        ...
    def compute_expiry(self, policy, record) -> datetime:
        base = {"creation_date": record.source_created_at,
                "ingestion_date": record.ingested_at,
                "last_modified_date": record.source_modified_at}[policy.retention_trigger]
        return base + timedelta(days=policy.effective_retention_days(record.jurisdiction))
    def assign(self, record, policies) -> list[Assignment]:
        # governing = longest computed_expiry; set is_governing=True on that one
```
- `effective_retention_days` consults `jurisdiction_overrides` first, then base `retention_period_days`.
- Governing-policy rule: the assignment with the latest `computed_expiry` wins (retention always trumps shorter periods); `archived_records.governing_policy_id` and `retention_expiry` updated to match.
- Assignment reasoning captured in `assignment_details` JSONB (matched_conditions, trigger_date, trigger_type, classification_method/confidence) for defensibility.

**Testing**:
- `Unit: record matching two policies (5yr, 7yr) → governing = 7yr policy, retention_expiry = created + 7yr`
- `Unit: jurisdiction_override EU=1825d overrides base 2555d for an EU record`
- `Unit: custom_condition source_metadata.folder == "Inbox" → matches only Inbox records`
- `Unit: no matching policy → empty assignments, retention_expiry = NULL (becomes a coverage gap)`
- `Integration: assign persists record_policy_assignments with exactly one is_governing=True`

#### 2.3 — Policy conflict detection

**What**: Detect and surface conflicts between overlapping policies (retention overlap, jurisdiction conflict, GDPR-erasure vs retention, action conflict).

**Design**:
- `ConflictDetector.scan(org_id)` compares active policies pairwise over overlapping scopes:
  - `erasure_vs_retention`: a GDPR-erasure-flagged policy overlaps a mandatory-retention policy (e.g. FINRA) on the same data class/jurisdiction set.
  - `retention_overlap`: same scope, differing `retention_period_days`.
  - `action_conflict`: same scope, one `delete` vs another `archive_cold`.
- Writes `policy_conflicts` rows (open/resolved/acknowledged). Surfaced via `GET /policies/conflicts`.
- AI-assisted reconciliation deferred to Phase 8; here it is deterministic detection only.

**Testing**:
- `Unit: GDPR-erasure policy + FINRA-6yr policy on same scope → erasure_vs_retention conflict`
- `Unit: two policies, disjoint scopes → no conflict`
- `Integration: scan persists policy_conflicts and GET /policies/conflicts returns them`

---

## Phase 3: Connector Framework, Ingestion & Immutable Storage

### Purpose
Make the platform actually archive data. This phase defines the connector abstraction, builds the first three MVP connectors (local filesystem, Microsoft 365, Google Workspace), stores content in WORM-compliant object storage, and writes record metadata to the hybrid `archived_records` table — at which point policies from Phase 2 can be assigned to real records. This is the second half of the core value proposition.

### Tasks

#### 3.1 — Immutable object storage layer

**What**: An `S3Client` wrapper that writes content with Object Lock retention and supports tiering, plus envelope encryption.

**Design**:
```python
class StorageBackend:
    async def put_record(self, org_id, key, content: bytes, content_type: str,
                         retain_until: datetime, lock_mode: str) -> StoredObject:
        # boto3 put_object with ObjectLockMode, ObjectLockRetainUntilDate
    async def get_record(self, bucket, key) -> bytes: ...
    async def set_tier(self, bucket, key, tier: Literal["hot","warm","cold","glacier"]): ...
    async def delete_record(self, bucket, key, bypass_governance: bool=False): ...  # only in GOVERNANCE
```
- Keys: `{org_id}/{ingested_month}/{record_id}` so deletion and lifecycle map cleanly to records.
- `dra/storage/crypto.py`: per-tenant data key (DEK) wrapped by KMS master key; content encrypted AES-256-GCM before upload; `body_hash` (SHA-256 of plaintext) stored in DB for integrity.
- COMPLIANCE mode = no early deletion possible (enforced by S3); GOVERNANCE allows privileged bypass (audited).

**Testing**:
- `Integration (MinIO): put_record with retain_until → object present; delete before retain_until in COMPLIANCE → S3 error (object locked)`
- `Integration (MinIO): GOVERNANCE bypass delete → succeeds and writes audit entry storage.governance_bypass`
- `Unit: encrypt then decrypt round-trips; body_hash matches plaintext SHA-256`

#### 3.2 — Connector abstraction and registry

**What**: A `Connector` ABC defining the contract every data source implements, plus a registry mapping `source_type` → connector.

**Design**:
```python
class Connector(ABC):
    source_type: ClassVar[str]
    capabilities: ClassVar[Capabilities]  # supports_hold, supports_incremental_sync, max_items, rate_limit
    @abstractmethod
    async def validate_config(self, cfg: dict) -> None: ...
    @abstractmethod
    async def iter_items(self, cfg: dict, since: SyncState | None) -> AsyncIterator[SourceItem]: ...
    @abstractmethod
    async def fetch_content(self, cfg: dict, item: SourceItem) -> bytes: ...
    async def place_hold(self, cfg, item_ids) -> None: ...     # default: app-side hold only
    async def release_hold(self, cfg, item_ids) -> None: ...

@dataclass
class SourceItem:
    original_id: str; content_type: str; subject: str | None
    source_created_at: datetime | None; source_metadata: dict; size_bytes: int
```
- `registry.py`: decorator `@register` populates `source_type -> class`. `connection_config`/`sync_state`/`capabilities` JSONB shapes validated by per-connector Pydantic models in `schemas/jsonb/`.

**Testing**:
- `Unit: registry resolves "microsoft_365" to M365Connector`
- `Unit: connector with missing required config field → validate_config raises with field name`

#### 3.3 — Ingestion pipeline and `archived_records` write path

**What**: A Celery sync task that pulls items from a connector, stores content immutably, extracts searchable text, computes hashes, and writes records — then triggers policy assignment.

**Design**:
- `tasks/sync.py::sync_data_source(source_id)`:
  1. Load `data_sources` row; instantiate connector; decrypt config.
  2. `iter_items(since=sync_state)` → for each item: `fetch_content` → encrypt+`put_record` (Object Lock retain_until = max anticipated retention, refined after policy assignment) → extract plaintext (`body_text`) via format handlers (eml, docx, pdf, html, plain) → compute `body_hash`.
  3. Insert into `archived_records` (creating the tenant LIST partition + monthly RANGE sub-partition on demand via `partitions.py`).
  4. Enqueue `assign_policies(record_id)` (Phase 2 engine) and `classify_record(record_id)` (Phase 6, no-op until then).
  5. Update `sync_state` (delta token/cursor) and source counters; emit `record.ingested` audit entries (batched).
- Idempotency: unique `(organisation_id, data_source_id, original_id)` guard → re-sync updates, never duplicates.
- Incremental sync uses connector delta tokens stored in `sync_state` JSONB.

**Testing**:
- `Integration (testcontainers + MinIO): sync a fixture local_fs dir of 5 files → 5 archived_records, 5 locked objects, body_hash set, policy assignment enqueued`
- `Unit: re-ingesting same original_id → updates existing row, no duplicate (idempotency)`
- `Integration: text extraction for .docx/.pdf fixtures → body_text non-empty and FTS-indexed`
- `Integration: sync_state delta token persisted; second sync only fetches items newer than token (mocked connector)`

#### 3.4 — MVP connectors: Local FS, Microsoft 365, Google Workspace

**What**: Three concrete connectors.

**Design**:
- **Local FS** (`local_fs.py`): walks a configured root; `content_type` inferred from extension; `source_metadata` = `{path, mtime, owner, permissions}`. No OAuth. Great for tests/dev.
- **Microsoft 365** (`m365.py`): Microsoft Graph via `msgraph-sdk`, OAuth 2.0 client-credentials (`standards.md`). Covers Exchange mail, SharePoint/OneDrive files, Teams messages. `iter_items` uses Graph delta queries; `source_metadata` per DM3 email/document examples; `place_hold` uses Graph in-place hold where available, else app-side `hold_records`.
- **Google Workspace** (`gworkspace.py`): `google-api-python-client`, OAuth 2.0 with domain-wide delegation. Gmail + Drive. Delta via Gmail historyId / Drive changes feed.
- Each ships a JSON schema for its `source_metadata` and registers a `pg_jsonschema` CHECK fragment (email schema per DM3 §"JSON Schema Validation").

**Testing**:
- `Integration: local_fs end-to-end (above)`
- `Integration (mocked Graph SDK): M365 iter_items paginates delta pages → SourceItems with correct email source_metadata; CHECK constraint accepts the payload`
- `Integration (mocked Graph): invalid email metadata (missing message_id) → pg_jsonschema CHECK rejects insert`
- `Integration (mocked Google API): gworkspace Gmail sync → records with thread metadata`
- `Real (optional, gated by env creds): M365 sandbox tenant sync of <10 messages`

---

## Phase 4: Full-Text Search & eDiscovery Foundations

### Purpose
Deliver indexed search across archived content — a table-stakes feature and the entry point for legal hold scoping (Phase 5) and eDiscovery collection. Built on PostgreSQL FTS behind a `SearchBackend` interface so OpenSearch can be substituted at scale without API changes.

### Tasks

#### 4.1 — SearchBackend interface and PostgreSQL FTS implementation

**What**: A query-compilation layer translating a structured query spec into an indexed search.

**Design**:
```python
class SearchBackend(ABC):
    @abstractmethod
    async def search(self, org_id: UUID, spec: QuerySpec, page: Page) -> SearchResults: ...

class QuerySpec(BaseModel):
    boolean_query: str | None          # parsed to tsquery
    date_range: DateRange | None
    content_types: list[str] = []
    data_sources: list[UUID] = []
    custodians: list[UUID] = []
    data_classes: list[str] = []
    source_metadata_filters: list[MetaFilter] = []   # JSONB path filters
    exclude_keywords: list[str] = []
    file_size_range: SizeRange | None
    sort: Sort = Sort(field="source_created_at", order="desc")
```
- `postgres.py`: builds a parameterised SQL query combining `to_tsvector @@ websearch_to_tsquery`, B-tree predicates on relational columns, and JSONB containment (`source_metadata @> ...`) for `source_metadata_filters`. Pagination via keyset (id, source_created_at).
- Boolean query parsing: support AND/OR/NOT and quoted phrases → `websearch_to_tsquery`/`tsquery`.

**Testing**:
- `Unit: QuerySpec "(merger OR acquisition) AND NOT draft" → correct tsquery string`
- `Integration: index 100 fixture records; search "contract" with date_range → only matching records, correct count`
- `Integration: source_metadata_filter folder=Inbox → only Inbox emails`
- `Integration: keyset pagination returns stable, non-overlapping pages`

#### 4.2 — Search API and saved queries

**What**: `POST /search` and saved-query persistence.

**Design**:
- `POST /search` accepts `QuerySpec`, returns paginated hits with highlighted snippets and `record` summaries (never raw locked content unless authorised).
- `search_queries` persisted (DM3 §7) with `query_spec` JSONB; `is_saved` flag; optional `matter_id` association.
- Every search emits an audit entry (`search.executed`) with the spec — important for eDiscovery defensibility.

**Testing**:
- `Integration: POST /search → 200 with hits and snippets; audit search.executed recorded`
- `Integration: save a query then GET /search-queries → returns it scoped to tenant`
- `Integration (RLS): user from org B cannot retrieve org A's saved query → 404`

#### 4.3 — eDiscovery collections and chain of custody

**What**: Build collections from search results and track chain of custody — the EDRM Identification→Collection stages.

**Design**:
- `POST /matters/{id}/collections` from a `search_query_id` → snapshots matching `record_id`s into `collection_items` (status `pending`).
- `chain_of_custody` rows written on every action (`collected`, `reviewed`, `tagged`, `produced`) with `content_hash` at each step (DM1 §7 pattern) — proves integrity per EDRM.
- Review coding (`responsive`/`privileged`/etc.) stored in `collection_items.review_annotations` JSONB.

**Testing**:
- `Integration: create collection from saved query → collection_items mirror the search hits; chain_of_custody "collected" rows written`
- `Unit: content_hash recorded in custody equals record body_hash at collection time`
- `Integration: tag an item responsive → review_annotations updated, custody "tagged" entry added`

---

## Phase 5: Legal Hold Management

### Purpose
Implement the eDiscovery preservation half of the product: place custodians and data sources under hold, suspend disposition for held records, notify and track custodian acknowledgement, and release holds with full audit documentation — the EDRM Preservation stage. Cross-platform orchestration (one hold spanning M365 + Google + Slack) is a key differentiator addressed here.

### Tasks

#### 5.1 — Matters, custodians, and hold CRUD

**What**: Endpoints for legal matters, custodian registry, and hold lifecycle.

**Design**:
- `legal_matters` (JSONB `matter_details` per matter_type), `custodians` (JSONB `data_source_access` map), `legal_holds` (JSONB `scope_definition`) per DM3 §5.
- Endpoints: `POST/GET /matters`, `POST/GET /matters/{id}/holds`, `POST /custodians`, `POST /holds/{id}/issue`, `POST /holds/{id}/release`.
- Hold state machine: `draft → active → released`/`expired`. Issue/release restricted to `legal_admin`.

**Testing**:
- `Integration: create matter + draft hold → 201; issue by legal_admin → status active, audit hold.issued`
- `Unit: release a draft (never-issued) hold → 409 invalid transition`
- `Integration (RBAC): compliance_admin attempts hold issue → 403`

#### 5.2 — Hold placement: suspend disposition for in-scope records

**What**: Resolve a hold's scope to concrete records, write `hold_records`, and increment `hold_count` so retention/disposition skips them.

**Design**:
- `tasks/holds.py::apply_hold(hold_id)`:
  1. Compile `scope_definition` into a `QuerySpec`; run `SearchBackend.search` to enumerate in-scope `record_id`s.
  2. Insert `hold_records`; increment `archived_records.hold_count`; set `lifecycle_status='hold'`.
  3. For connectors with `supports_hold`, call `connector.place_hold` (e.g. M365 in-place hold) for belt-and-braces preservation at source.
- Disposition scan (Phase 6) excludes any record with `hold_count > 0` — enforced both in the SQL `idx_records_expiry` partial index predicate and an explicit guard.
- Release: `release_hold(hold_id)` decrements `hold_count`, clears `hold_records.released_at`, and only returns records to `active` when `hold_count` reaches 0.

**Testing**:
- `Integration: apply hold over a keyword scope → matching records get hold_count=1, lifecycle hold; non-matching untouched`
- `Integration: record under two holds → releasing one leaves hold_count=1 and status hold`
- `Integration: disposition scan skips held records (verified in Phase 6 test, asserted here via query)`
- `Integration (mocked M365): apply_hold calls connector.place_hold for supports_hold sources`

#### 5.3 — Custodian notifications, acknowledgement, and reminders

**What**: Notify custodians, track acknowledgement, and escalate reminders.

**Design**:
- `hold_custodian_assignments` with `notification_history` JSONB (DM3 §5). Notification templates in `dra/templates/`.
- On `issue`, enqueue `notify_custodian` per assignment (email + in-app); record `initial_notice` in history; status `notified`.
- Custodian portal endpoint `POST /portal/holds/{id}/acknowledge` (OIDC-authed custodian) → status `acknowledged`, `acknowledged_at`, IP recorded.
- Celery Beat `escalate_reminders` daily: for assignments `notified` and unacknowledged past N days, send reminder, increment `reminder_count`, escalate after threshold.

**Testing**:
- `Integration: issue hold with 3 custodians → 3 notify tasks enqueued, statuses notified, history initial_notice`
- `Integration: custodian acknowledges → status acknowledged, IP and timestamp recorded, audit hold.custodian_acknowledged`
- `Unit: reminder escalation selects only assignments past timeout and unacknowledged`

---

## Phase 6: Disposition & Defensible Deletion

### Purpose
Close the lifecycle: identify records past retention, route them through a multi-stage approval chain, execute NIST 800-88-conformant deletion that respects legal holds, and issue tamper-evident, RFC-3161-timestamped deletion certificates with human-readable narratives. This is where the platform proves defensibility.

### Tasks

#### 6.1 — Retention expiry scan and disposition batch assembly

**What**: A scheduled scan that finds eligible records and assembles review batches per policy.

**Design**:
- Celery Beat `retention_scan` (daily): selects `archived_records` where `lifecycle_status='active' AND hold_count=0 AND retention_expiry <= now()` (uses partial index `idx_records_expiry`), grouped by `governing_policy_id`.
- Creates a `disposition_batches` row per policy with `batch_summary` JSONB (counts by content_type/source/data_class, date range, est. cost savings) and `disposition_items` (status `pending`). Sets records `lifecycle_status='pending_disposition'`.
- If `policy.requires_approval=false`, route directly to execution; else `pending_review`.

**Testing**:
- `Integration: 10 expired unheld records under one policy → one batch, 10 items, batch_summary counts correct`
- `Integration: expired but held record (hold_count>0) → excluded from batch`
- `Unit: batch_summary aggregation groups correctly by content_type and data_class`

#### 6.2 — Approval chain workflow

**What**: Multi-stage approval per the policy's `approval_workflow`, with exceptions and timeout escalation.

**Design**:
- On batch creation, materialise `disposition_approvals` rows from `approval_workflow.stages` (ordered, role-targeted).
- `POST /disposition/batches/{id}/decision` (approver) records `approved`/`rejected`/`deferred` + comments. Batch advances only when each `required` stage approves in order. Reviewer may except individual items → `disposition_items.status='excepted'` with reason (e.g. ongoing business relevance flagged in Phase 8).
- Timeout: Beat job applies `escalation_policy` (`auto_approve_after_timeout` or escalate to next role).
- A rejection sets batch `rejected`; records return to `active`.

**Testing**:
- `Integration: two-stage workflow; first approves, second approves → batch approved`
- `Integration: first stage rejects → batch rejected, records back to active`
- `Integration: except one item → that item excepted, remains, others proceed`
- `Unit: auto_approve_after_timeout advances a stage past its timeout_days`

#### 6.3 — Defensible deletion execution

**What**: Execute approved deletions in a verifiable, hold-safe, NIST 800-88-aligned manner.

**Design**:
- `tasks/disposition.py::execute_batch(batch_id)` per item:
  1. Re-check `hold_count=0` immediately before deletion (race guard) — held items auto-excepted.
  2. Storage deletion: in GOVERNANCE mode delete the object (audited); in COMPLIANCE mode the object self-expires at Object Lock retain_until and we record a tombstone and verify inaccessibility — never claim deletion of a still-locked object (honest narrative).
  3. Purge from search index; write a `tombstone` retaining only non-content metadata (id, hashes, policy, timestamps) for audit.
  4. Set `archived_records.lifecycle_status='deleted'`; `disposition_items.status='deleted'`, `deletion_verified=true`.
  5. Append to `disposition_batches.execution_log` JSONB (validation → storage_deletion → index_removal → metadata_tombstone) per DM3 §6.
- Deletion method recorded as NIST 800-88 `Purge` (cryptographic erasure via DEK destruction) or `Destroy`, referenced in the certificate.

**Testing**:
- `Integration (MinIO, GOVERNANCE): execute batch → objects deleted, search index purged, records lifecycle deleted, execution_log has all four steps`
- `Integration: item placed under hold between batch creation and execution → auto-excepted, not deleted`
- `Integration (COMPLIANCE): object still locked → recorded as tombstoned-pending-expiry, narrative reflects honest status`
- `Unit: cryptographic erasure destroys the per-record DEK; subsequent decrypt fails`

#### 6.4 — Deletion certificates with hash chain and RFC 3161 timestamp

**What**: Issue a tamper-evident certificate per executed batch.

**Design**:
- `deletion_certificates` per DM3 §6: `content_hash` (SHA-256 of `certificate_content` JSONB), `previous_cert_hash`, `chain_sequence` — a per-org certificate hash chain (mirrors audit chain).
- `certificate_content` JSONB carries the full narrative, approval chain, data summary, and storage verification.
- The `content_hash` is submitted to an RFC 3161 TSA (`audit/timestamp.py`); the returned timestamp token is stored, proving certificate existence time for admissibility.
- `GET /disposition/certificates/{id}` renders human-readable + downloadable PDF.

**Testing**:
- `Integration: issue two certificates → second.previous_cert_hash == first.content_hash, chain_sequence increments`
- `Unit: tampering with certificate_content → recomputed content_hash mismatches stored hash (verification fails)`
- `Integration (mocked TSA): timestamp token attached and verifiable against TSA cert`
- `Integration: certificate_content.disposition_narrative references the governing policy and approvers`

---

## Phase 7: Compliance Reporting, Coverage-Gap Detection & Audit Verification

### Purpose
Give compliance officers the dashboards and scheduled reports that demonstrate audit readiness, surface data stores with no retention coverage, and let auditors independently verify the integrity of the audit and certificate chains. After this phase the MVP feature set is complete.

### Tasks

#### 7.1 — Report generation and scheduling

**What**: Generate and schedule the report types in DM3 §9 (policy_adherence, hold_status, disposition_summary, coverage_gap, audit_readiness, storage_analytics).

**Design**:
- `services/reporting.py` builds each report from live queries + materialised views; snapshots result into `compliance_reports.data_snapshot` JSONB (immutable point-in-time evidence).
- `POST /reports` (one-off) and scheduled via `schedule_config` JSONB (cron, recipients, format) executed by Beat. PDF/CSV export.
- Dashboard aggregate endpoint `GET /dashboard` returns active policy/hold/upcoming-disposition counts and recent audit events.

**Testing**:
- `Integration: generate disposition_summary → data_snapshot matches underlying batch data`
- `Integration: scheduled report cron registered in Beat; manual trigger emails recipients (mocked SMTP)`
- `Integration: GET /dashboard → correct counts across seeded fixtures`

#### 7.2 — Coverage-gap detection

**What**: Continuously identify data sources / record sets with no governing policy or stale/conflicting coverage.

**Design**:
- `services/reporting.py::detect_gaps(org_id)`: finds `data_sources` whose records include rows with `governing_policy_id IS NULL`, or content_types not covered by any active policy; writes `coverage_gaps` with `gap_details` JSONB (uncovered counts, suggested policies, risk_level, regulatory_exposure).
- Beat `coverage_scan` daily; surfaced on dashboard and via `GET /coverage-gaps`.

**Testing**:
- `Integration: ingest records of a content_type with no matching policy → coverage_gap no_policy created with uncovered_record_count`
- `Integration: add covering policy and reassign → gap resolved_at set on next scan`

#### 7.3 — Audit & certificate chain verification endpoint

**What**: Let auditors verify chain integrity independently.

**Design**:
- `GET /audit/verify` (auditor/compliance_admin) → runs `verify_chain` over `audit_log` and `deletion_certificates`; returns `{valid, entries_checked, first_break_sequence?}`.
- Periodic Beat `anchor_chains`: computes a Merkle root over the period's audit entries and timestamps it via RFC 3161 TSA — external anchoring that survives HMAC-key compromise.

**Testing**:
- `Integration: verify endpoint over clean chains → valid=True`
- `Integration: superuser-injected tamper → valid=False with breaking sequence`
- `Integration (mocked TSA): anchor_chains stores a timestamped Merkle root`

---

## Phase 8: AI-Native Layer (v1.1 Differentiators)

### Purpose
Deliver the differentiation that justifies the project's existence: autonomous content classification, AI hold-scoping, human-readable disposition narratives, AI-assisted policy authoring, and multi-jurisdiction conflict reconciliation. Each is additive — it enriches data and assists humans, never bypassing the deterministic compliance controls of Phases 2–7.

### Tasks

#### 8.1 — LLM provider abstraction and prompt registry

**What**: A provider-agnostic LLM interface with versioned prompts and prompt caching.

**Design**:
```python
class LLMProvider(ABC):
    @abstractmethod
    async def complete(self, system: str, messages: list[Msg], *, schema: type[BaseModel] | None,
                       cache: bool = True) -> LLMResult: ...
class AnthropicProvider(LLMProvider): ...   # uses anthropic SDK, prompt caching on system+few-shot
```
- Prompts versioned under `dra/llm/prompts/` (e.g. `classify_v1.txt`), each tagged with a `model_version` recorded into the JSONB it produces — so every AI output is traceable and reproducible for audit.
- Structured outputs validated against Pydantic schemas (tool-use / JSON mode).

**Testing**:
- `Unit (mocked SDK): complete returns schema-validated object; invalid JSON → retried then raises`
- `Unit: model_version stamped into result metadata`

#### 8.2 — Autonomous content classification

**What**: Classify ingested records into regulatory data classes (pii, phi, financial, confidential, general) without manual rules.

**Design**:
- `services/classification.py::classify_record`: combines a fast local PII/PHI detector (regex + `presidio`-style entity recognition) with an LLM pass for ambiguous/contextual classes; writes `data_class`, `ai_confidence`, and `enrichment_data` (entities, pii_detected counts, topics, summary, classification_reasoning) per DM3 §3.
- Triggered from the ingestion pipeline (the `classify_record` enqueue stubbed in 3.3 becomes live). Low-confidence results flagged for human verification (`classification_verified=false`).
- Re-runs policy assignment (Phase 2) after classification, since `data_class` drives scope matching.

**Testing**:
- `Unit (mocked LLM): an email containing an SSN → data_class pii, enrichment_data.pii_detected.ssn>0`
- `Integration: classify then reassign → policy whose scope is data_classes:["pii"] now governs the record`
- `Unit: confidence below threshold → classification_verified=false, surfaced for review`

#### 8.3 — AI hold scoping

**What**: Given a matter description, suggest custodians and data sources likely relevant.

**Design**:
- `services/hold_scoping.py::suggest_scope(matter_id, description)`: extracts keywords/entities via LLM, runs semantic + keyword search to rank custodians (by message volume matching keywords in date range) and data sources; writes `legal_holds.ai_scope_suggestions` JSONB (suggested_custodians with relevance_score + reason, suggested_data_sources, estimated_record_count/storage, model_version) per DM3 §5.
- Surfaced in the hold-creation UI; the legal admin accepts/edits before issuing — AI suggests, human decides.

**Testing**:
- `Unit (mocked LLM + seeded records): suggest_scope ranks the custodian with most keyword-matching messages first with a reason string`
- `Integration: suggestions persisted to ai_scope_suggestions; estimated_record_count matches a real search count`

#### 8.4 — Disposition narratives, policy authoring, and conflict reconciliation

**What**: Three LLM-assisted authoring features.

**Design**:
- **Disposition narrative**: `services/narrative.py` generates the human-readable `disposition_narrative` for certificates (Phase 6.4) from structured batch facts — grounded strictly in the batch's data (no hallucinated figures; numbers passed in, prose generated).
- **Policy authoring**: `POST /policies/draft-from-description` → LLM drafts a `RetentionPolicyIn` (period, trigger, disposition action, scope, jurisdiction_overrides, citation) from a plain-language regulatory description; returned as a *draft* for human review.
- **Conflict reconciliation**: extends Phase 2.3 — for each detected `policy_conflict`, the LLM proposes a reconciled resolution (e.g. "retain to satisfy FINRA 6yr, then honour GDPR erasure"), stored in `policy_conflicts.resolution` for human approval.

**Testing**:
- `Unit (mocked LLM): narrative includes exactly the record counts/policy/approvers passed in; assert no numeric values absent from input appear in output`
- `Integration: draft-from-description "retain trade emails 7 years per FINRA" → draft policy with retention_period_days=2555 and FINRA citation, status draft`
- `Unit: conflict reconciliation proposal written to policy_conflicts.resolution, status stays open until human approves`

---

## Phase 9: Extended Connectors & OpenAPI/MCP Surface (v1.1)

### Purpose
Broaden data-source coverage and harden the public integration surface — the v1.1 connectors plus the published OpenAPI spec and MCP server that make the platform programmable and AI-assistant-accessible.

### Tasks

#### 9.1 — Extended connectors: Slack, Google Drive, Box, Dropbox, Salesforce

**What**: Five additional connectors using the Phase 3 abstraction — zero schema migration (metadata goes to `source_metadata` JSONB).

**Design**:
- Each implements `Connector`, registers a `source_metadata` JSON schema, and declares `capabilities` (hold support, incremental sync, rate limits). Slack message metadata per DM3 §3 example (channel, thread_ts, reactions, blocks). Salesforce uses structured-record metadata shape (table/PK/column_values).
- Connector-specific rate limiting honoured via Celery rate limits + `capabilities.rate_limit_per_minute`. Social/SaaS API-degradation fallback noted (`standards.md` social-API risk).

**Testing**:
- `Integration (mocked Slack API): sync a channel export fixture → chat_message records with reaction/thread metadata; CHECK accepts payload`
- `Unit: each new connector resolves via registry and validates its config schema`
- `Integration: adding Slack required no Alembic migration (schema hash unchanged) — assert migrations dir untouched by connector addition`

#### 9.2 — OpenAPI 3.1 spec, client niceties, and eDiscovery export

**What**: Publish the full OpenAPI 3.1 spec and standards-conformant export packages.

**Design**:
- FastAPI auto-generates OpenAPI 3.1; committed `openapi.json` is CI-diffed (Phase 1.5). Redoc/Swagger UI served at `/docs`.
- RFC 7807 problem+json error bodies; RFC 8288 `Link` headers for pagination.
- eDiscovery production export (`productions` from Phase 4/DM3 §7): EML/PST/PDF/native + Concordance/EDRM-XML load file with custody metadata, per `standards.md` (EDRM XML, S/MIME preservation for signed mail).

**Testing**:
- `Integration: GET /openapi.json validates against OpenAPI 3.1 meta-schema`
- `Integration: error response conforms to RFC 7807 (type/title/status/detail)`
- `Integration: export a collection as EML+load-file → archive contains messages + Concordance DAT with custody fields; hashes match chain_of_custody`

#### 9.3 — MCP server

**What**: Expose safe, read-and-controlled tools to AI assistants per `standards.md`'s flagged opportunity.

**Design**:
- `dra/mcp/server.py` (mcp SDK) tools: `search_archive(query_spec)`, `check_policy_coverage(source_id)`, `list_active_holds()`, `place_hold(matter_id, scope)` (gated: returns a *draft* hold requiring human issue), `generate_report(type, params)`.
- Tools run under a scoped service identity with RBAC; all invocations audited (`mcp.tool_invoked`). State-changing tools never bypass approval/issue gates.

**Testing**:
- `Integration: MCP search_archive tool returns same results as REST /search for an identical spec`
- `Integration: MCP place_hold creates a draft hold only (status draft), audit mcp.tool_invoked recorded`
- `Unit: MCP tool call without required scope → permission error`

---

## Phase 10: Web UI — Admin Dashboard & Custodian Portal

### Purpose
Provide the human-facing surfaces: a compliance/legal/IT admin dashboard and a custodian self-service portal. The API is the source of truth; the UI is a typed client over it.

### Tasks

#### 10.1 — Admin dashboard

**What**: React SPA covering policies, holds, search/eDiscovery, disposition review, reporting, and connectors.

**Design**:
- Vite + React + TS, shadcn/ui + Tailwind. Typed API client generated from `openapi.json` (`openapi-typescript`).
- Pages: Dashboard (counts + audit feed), Policies (CRUD, templates, AI draft, conflicts), Holds (create with AI scope suggestions, custodian status), Search & Collections, Disposition (review queue, approve/except, certificates), Reports, Connectors (config + sync status), Audit (chain verification view).
- RBAC-aware nav: routes/actions hidden per role.

**Testing**:
- `Component (vitest + RTL): disposition review queue renders items, approve button disabled for auditor role`
- `Component: policy AI-draft fills the form from a mocked draft-from-description response`
- `E2E (Playwright, mocked API): create policy → appears in list with status draft`

#### 10.2 — Custodian self-service portal

**What**: A scoped portal where custodians acknowledge holds and search their own archive.

**Design**:
- OIDC-authed; custodian sees only their `hold_custodian_assignments` and personal records.
- Acknowledge action (Phase 5.3) records timestamp+IP; personal archive search scoped to the custodian's `data_source_access`.

**Testing**:
- `E2E (mocked API): custodian logs in, sees pending hold, acknowledges → status acknowledged`
- `Component: custodian cannot see other custodians' holds (scoped data only)`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (schema, RLS, audit spine, auth, CI)  ─── required by everything
    │
Phase 2: Retention Policy Engine ───────── requires Phase 1
    │
Phase 3: Connectors / Ingestion / WORM ─── requires Phase 1 (uses Phase 2 engine when present)
    │
Phase 4: Search & eDiscovery foundations ─ requires Phase 3
    │
    ├── Phase 5: Legal Hold Management ──── requires Phase 4  ┐
    └── Phase 6: Disposition & Deletion ── requires Phase 2+3 ┘ (5 and 6 can parallel;
    │                                                            6 must respect 5's holds)
Phase 7: Reporting / Coverage / Audit verify ─ requires Phases 2,3,5,6  → MVP COMPLETE
    │
Phase 8: AI-Native Layer ───────────────── requires Phases 2,3,4,6 (enriches them)
    │
    ├── Phase 9: Extended Connectors / OpenAPI / MCP ─ requires Phase 3 (+8 for MCP hold-scope)
    └── Phase 10: Web UI ───────────────────────────── requires Phases 2–7 APIs (9 for typed client)
```

**Parallelism opportunities:**
- **Phases 5 and 6** can be developed concurrently once Phases 3 and 4 land (both depend on records + search; 6's executor must honour 5's `hold_count` guard, which is a thin contract).
- **Phases 9 and 10** can be developed concurrently after the MVP (Phase 7) — connectors/MCP are backend, UI is frontend, sharing only the OpenAPI contract.
- Within Phase 8, the four AI features (8.2 classification, 8.3 hold scoping, 8.4 narratives/authoring/conflict) are independent once 8.1 lands and can be split across developers.
- **Phase 2** can begin in parallel with **Phase 3** after Phase 1, since the policy engine is unit-testable against synthetic records before real ingestion exists.

**MVP cut line:** Phases 1–7 deliver the full `features.md` MVP (policy engine, legal hold, MVP connectors, WORM storage, search, disposition, reporting). Phases 8–10 deliver the v1.1 differentiators (AI layer, extended connectors, public API/MCP, UI).

---

## Definition of Done (per phase)

A phase is complete only when **all** of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass; new tests cover both happy-path and the enumerated edge cases.
3. `ruff` lint and format pass with zero findings.
4. `mypy --strict` passes with zero errors.
5. The Docker images build and `docker compose up` reaches `/readyz` 200.
6. The phase's feature works end-to-end (demonstrated by at least one integration or E2E test exercising the real path against testcontainers Postgres/MinIO/Redis).
7. New config options are documented in `README.md` and represented in `Settings`.
8. New or changed API endpoints appear in the regenerated `openapi.json`, and the committed spec is updated (CI spec-diff is green).
9. New Alembic migrations are created, are reversible where feasible, and `alembic upgrade head` + `downgrade` round-trips on a clean database.
10. Every state-changing operation introduced in the phase emits a hash-chained `audit_log` entry, and `verify_chain` remains valid after the phase's tests run.
11. Any JSONB column written by the phase has a corresponding `pg_jsonschema` CHECK constraint and a Pydantic schema kept in sync.
12. RLS isolation is asserted for any new tenant-scoped table (a cross-tenant access test proves zero leakage).
```

