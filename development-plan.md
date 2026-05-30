# Autonomous Finance Operations — Phased Development Plan

> Project: 490-autonomous-finance-operations · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four data-model suggestions. The database design adopts **Data Model Suggestion 3 (Hybrid Relational + JSONB)** as the persistence backbone — normalised typed columns for money, approvals, and GL postings, with JSONB for AI reasoning chains, extraction output, policy parameters, and ERP metadata. The **append-only immutable audit log** pattern from Suggestion 2 (event sourcing) is applied specifically to the agent-action ledger, which is the platform's core compliance differentiator. Suggestions 1 and 4 informed the relational integrity rules and the time-partitioning strategy for high-volume tables.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The core value is LLM-driven document extraction, GL-coding prediction, payment-probability scoring, and reconciliation fuzzy-matching — all ML/LLM-heavy. Python has the strongest ecosystem (anthropic SDK, scikit-learn, pdfplumber, the Temporal Python SDK) for these workloads. |
| API framework | FastAPI | Native Pydantic v2 models, automatic **OpenAPI 3.1** generation (required by `standards.md`), async support for ERP/bank/LLM I/O, and dependency-injection that maps cleanly onto auth + tenant scoping. |
| Durable workflow engine | Temporal (Python SDK) | `research.md` mandates durable execution for multi-step financial workflows spanning hours to days, with idempotent steps to prevent duplicate payments. Temporal persists workflow state across restarts and provides built-in retries, timers, and signals (used for human approval gates). |
| Database | PostgreSQL 16 | Hybrid relational+JSONB model needs ACID, foreign keys, `CHECK` constraints, table partitioning, GIN indexes on JSONB, and row-level security for multi-tenancy. SQLite is unacceptable for regulated financial workloads. |
| ORM / migrations | SQLAlchemy 2.0 + Alembic | Typed declarative models, async engine, and versioned migrations. Alembic migrations are an explicit Definition-of-Done item per phase. |
| Cache / queue broker | Redis 7 | Temporal task-queue backing is separate; Redis serves rate-limited LLM call coordination, sanction-list caching, idempotency-key storage, and pub/sub for the oversight console's live feed. |
| LLM provider abstraction | `anthropic` SDK behind a provider interface | Document extraction, GL coding, dunning copy, and reasoning-chain generation use Claude. A thin `LLMProvider` protocol allows swapping/adding providers and enables deterministic mocking in tests. |
| Document parsing | `pdfplumber` + `pypdfium2` (digital PDF), `pillow` (image normalisation), multi-modal LLM for extraction | Handles digital PDFs, scanned images, and email bodies. EDI 810/820 handled by a dedicated `pyx12`-style parser module. |
| EDI parsing | Custom `edi/` module (X12 810/820/850/856) | `standards.md` requires EDI 810 ingest and EDI 820 emit. No mature permissively-licensed Python X12 lib covers all sets; a focused parser/serialiser is built in-house. |
| Bank statement parsing | Custom parsers for BAI2, MT940, CAMT.053 | `standards.md` requires all three. CAMT.053 is ISO 20022 XML (parsed with `lxml`); BAI2 and MT940 are line-format text parsers. |
| Frontend | Next.js 15 (App Router) + TypeScript + shadcn/ui + Tailwind | The **Human Oversight Console** is a core MVP feature (live workflow dashboard, exception queues, approve/pause/redirect). Server Components for data-heavy tables; client components for the live feed via SSE/WebSocket. |
| Auth | OAuth 2.0 + OIDC (Authlib server side), JWT (RFC 7519) sessions | `standards.md` requires OAuth 2.0 client-credentials for ERP/bank M2M (RFC 7523), OIDC for enterprise SSO (Okta/Entra/Ping), and JWT identity claims for audit attribution. |
| Sanction screening | Pluggable `SanctionScreener` over OFAC SDN + HM Treasury consolidated lists | Lists ingested daily into Postgres + Redis; screening is a non-negotiable payment-safety gate. |
| FX rates | ECB reference-rate feed adapter behind `RateProvider` | `research.md` prefers external rate feeds over a proprietary engine. |
| MCP server | `mcp` Python SDK (spec 2025-11-25) | Exposes governed finance tools (`get_invoice_status`, `get_cash_position`, `list_open_payables`, `trigger_payment_run`, `get_bank_reconciliation_exceptions`) to external AI clients. |
| Testing | pytest + pytest-asyncio + testcontainers (Postgres/Redis) + Temporal test env | Unit + mocked-integration + real-dependency integration via ephemeral containers; Temporal provides a time-skipping test environment. |
| Code quality | ruff (lint+format), mypy (strict), bandit (security) | `standards.md` flags OWASP API Top 10 and PCI DSS automated scanning; bandit + dependency audit are part of DoD. |
| Packaging | uv + pyproject.toml | Fast, reproducible dependency resolution and lockfile. |
| Containerisation | Docker + docker-compose | Self-hosted/hybrid deployment per `README.md` (data-residency requirements). Compose brings up Postgres, Redis, Temporal, API, worker, web. |
| Money type | `decimal.Decimal` everywhere; `NUMERIC(18,4)` in DB | Never use float for money. Pydantic `condecimal` validators enforce precision at the boundary. |

### Project Structure

```
autonomous-finance-operations/
├── pyproject.toml
├── uv.lock
├── Dockerfile                         # API + worker image
├── docker-compose.yml                 # postgres, redis, temporal, api, worker, web
├── alembic.ini
├── .env.example
├── README.md
├── migrations/                        # Alembic versioned migrations
│   └── versions/
├── config/
│   └── default.yaml                   # base config; env vars override
├── src/
│   └── afo/
│       ├── __init__.py
│       ├── main.py                    # FastAPI app factory
│       ├── settings.py                # Pydantic Settings (env + yaml)
│       ├── deps.py                    # DI: db session, current_user, tenant scope
│       ├── db/
│       │   ├── base.py                # SQLAlchemy declarative base, mixins
│       │   ├── session.py             # async engine + session maker
│       │   └── models/                # one module per aggregate
│       │       ├── organisation.py
│       │       ├── counterparty.py
│       │       ├── gl.py              # gl_account, cost_centre, fiscal_period
│       │       ├── purchase_order.py
│       │       ├── invoice.py
│       │       ├── matching.py
│       │       ├── approval.py
│       │       ├── payment.py
│       │       ├── banking.py
│       │       ├── journal.py
│       │       ├── collections.py
│       │       ├── close.py
│       │       ├── forecast.py
│       │       ├── governance.py
│       │       ├── audit.py           # append-only agent_audit_log (partitioned)
│       │       └── erp.py
│       ├── schemas/                   # Pydantic request/response models
│       ├── api/
│       │   ├── router.py              # mounts all routers under /api/v1
│       │   └── routes/                # invoices, payments, banking, approvals, ...
│       ├── domain/                    # pure business logic (no I/O)
│       │   ├── money.py
│       │   ├── matching.py            # 3-way match engine
│       │   ├── coding.py              # GL coding prediction
│       │   ├── policy.py              # policy-as-code evaluation
│       │   ├── reconciliation.py      # fuzzy matcher
│       │   ├── duplicate.py           # duplicate payment detection
│       │   ├── forecast.py            # 13-week cash model
│       │   └── collections.py         # payment-probability scoring
│       ├── ai/
│       │   ├── provider.py            # LLMProvider protocol + AnthropicProvider
│       │   ├── extraction.py          # multi-modal invoice extraction
│       │   ├── prompts/               # versioned prompt templates
│       │   └── reasoning.py           # reasoning-chain capture
│       ├── workflows/                 # Temporal workflows + activities
│       │   ├── invoice_workflow.py
│       │   ├── payment_workflow.py
│       │   ├── collections_workflow.py
│       │   ├── close_workflow.py
│       │   └── activities/
│       ├── integrations/
│       │   ├── erp/                   # base + netsuite, quickbooks, xero, sap, oracle
│       │   ├── banking/               # bai2, mt940, camt053 parsers
│       │   ├── edi/                   # x12 810/820/850/856
│       │   ├── sanctions/             # ofac, hm_treasury loaders + screener
│       │   └── fx/                    # ecb rate provider
│       ├── audit/
│       │   └── ledger.py              # append-only audit writer (hash-chained)
│       ├── mcp/
│       │   └── server.py              # MCP server exposing finance tools
│       └── worker.py                  # Temporal worker entrypoint
├── web/                               # Next.js oversight console
│   ├── package.json
│   └── app/
└── tests/
    ├── unit/
    ├── integration/
    ├── e2e/
    └── fixtures/                      # sample invoices, BAI2/MT940/CAMT files, EDI 810
```

The structure groups by **concern** (domain logic, AI, workflows, integrations, audit), not by phase, so every later phase adds modules without restructuring.

---

## Phase 1: Foundation, Data Model & Audit Ledger

### Purpose
Establish the project skeleton, configuration, database, multi-tenant scoping, and the immutable audit ledger. Auditability is the platform's core differentiator (`README.md`, `standards.md` SOX §404), so the agent-action ledger exists from day one. After this phase the application boots, connects to Postgres/Redis, exposes a health endpoint and OpenAPI docs, and can persist and verify tamper-evident audit records.

### Tasks

#### 1.1 — Project scaffold, settings, and containerisation

**What**: Bootstrap the repo with FastAPI, uv, Docker Compose, and layered configuration.

**Design**:
- `settings.py` uses `pydantic_settings.BaseSettings`, loading `config/default.yaml` then overriding with env vars.
```python
class Settings(BaseSettings):
    env: Literal["dev", "test", "prod"] = "dev"
    database_url: PostgresDsn
    redis_url: RedisDsn
    temporal_host: str = "localhost:7233"
    anthropic_api_key: SecretStr
    jwt_secret: SecretStr
    jwt_algorithm: str = "HS256"
    payment_auto_approve_threshold: Decimal = Decimal("5000")
    extraction_confidence_floor: Decimal = Decimal("0.85")
    model_config = SettingsConfigDict(env_prefix="AFO_", env_nested_delimiter="__")
```
- `main.py` exposes `create_app() -> FastAPI`, mounting `/api/v1` router, `/health`, and `/docs` (OpenAPI 3.1).
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `temporal` + `temporal-ui`, `api`, `worker`, `web`.
- `GET /health` returns `{"status":"ok","db":bool,"redis":bool,"temporal":bool}`.

**Testing**:
- `Unit: Settings loads defaults when env unset → payment_auto_approve_threshold == Decimal("5000")`.
- `Unit: AFO_PAYMENT_AUTO_APPROVE_THRESHOLD=10000 env → Decimal("10000")`.
- `Integration: GET /health with containers up → 200, all booleans true`.
- `Integration: GET /docs → 200, OpenAPI version field == "3.1.0"`.

#### 1.2 — Core SQLAlchemy base, mixins, and tenant model

**What**: Declarative base with shared mixins and the `organisation` / `fiscal_period` tables.

**Design**:
- Mixins: `UUIDPrimaryKeyMixin` (`id: Mapped[UUID]` default `uuid4`), `TimestampMixin` (`created_at`, `updated_at` with `server_default=func.now()`), `TenantMixin` (`organisation_id` FK + index).
- Tables from Suggestion 1: `organisation`, `fiscal_period` (status `open|closing|closed|locked`).
- Row-level security pattern: every tenant-scoped query goes through `deps.tenant_scope()` which injects `WHERE organisation_id = :tenant`.

**Testing**:
- `Unit: model instance gets uuid id and created_at on flush (in-memory async session)`.
- `Integration (testcontainer): insert organisation → row persisted with default base_currency='USD'`.
- `Integration: fiscal_period with invalid status → IntegrityError (CHECK constraint)`.

#### 1.3 — Immutable, hash-chained agent audit ledger

**What**: Append-only `agent_audit_log` partitioned by month, with a hash chain making tampering detectable.

**Design**:
- Table per Suggestion 1 `agent_audit_log` plus two columns for tamper-evidence: `prev_hash CHAR(64)`, `entry_hash CHAR(64) NOT NULL`. Partition by `RANGE (created_at)`; an Alembic helper creates the next 13 monthly partitions.
- `audit/ledger.py`:
```python
class AuditEntry(BaseModel):
    organisation_id: UUID
    agent_id: str
    action: str
    entity_type: str
    entity_id: UUID
    reasoning_chain: str
    policy_citations: list[str]
    input_summary: str
    output_summary: str
    confidence: Decimal | None
    decision: Literal["auto_approved","escalated","rejected","deferred"]

async def append(session, entry: AuditEntry) -> AuditRecord:
    prev = await _latest_hash(session, entry.organisation_id)
    payload = canonical_json(entry)          # sorted keys, no whitespace
    entry_hash = sha256(prev + payload + created_at_iso)
    # INSERT ... never UPDATE/DELETE
```
- A DB trigger (`BEFORE UPDATE OR DELETE`) raises an exception to enforce append-only at the storage layer.
- `verify_chain(session, organisation_id) -> ChainVerification` recomputes hashes and reports the first broken link.

**Testing**:
- `Unit: append computes entry_hash = sha256(prev_hash + payload + ts); two appends chain correctly`.
- `Integration: UPDATE on agent_audit_log row → DB raises (trigger)`.
- `Integration: append 100 entries, verify_chain → valid=True`.
- `Integration: manually corrupt one row's reasoning_chain via raw SQL bypassing trigger, verify_chain → valid=False, first_broken=<position>`.

#### 1.4 — AuthN/AuthZ: JWT sessions, OIDC, OAuth client-credentials

**What**: Authentication dependencies and role model.

**Design**:
- `User` and `Role` tables; roles: `ap_clerk`, `ap_manager`, `ar_collector`, `controller`, `treasurer`, `cfo`, `auditor`, `admin`.
- `deps.current_user()` validates a JWT (RFC 7519), returns `AuthContext(user_id, org_id, roles)`.
- OIDC login via Authlib (`/auth/oidc/login`, `/auth/oidc/callback`); identity claims (`sub`, `email`) recorded for audit attribution.
- `require_role(*roles)` dependency → 403 on mismatch (OWASP API1 BOLA / API5 BFLA mitigation).
- Segregation-of-duties primitive `assert_not_self_approval(creator_id, approver_id)`.

**Testing**:
- `Unit: valid JWT → AuthContext with correct roles`.
- `Unit: expired JWT → 401`.
- `Unit: require_role('controller') with ap_clerk token → 403`.
- `Unit: assert_not_self_approval(u, u) → raises SoDViolation`.
- `Integration (mocked OIDC): callback with valid code → session JWT issued`.

---

## Phase 2: Master Data & Chart of Accounts

### Purpose
Build the reference data that every downstream workflow validates against: counterparties (suppliers/customers) with bank accounts, the chart of accounts, cost centres, and purchase orders. Extraction validation, 3-way matching, GL coding, and sanction screening all depend on this layer existing first.

### Tasks

#### 2.1 — Counterparty & bank-account management with dual-approval changes

**What**: CRUD for suppliers/customers and their bank accounts, with mandatory dual approval on creation and bank-detail changes.

**Design**:
- Tables `counterparty`, `counterparty_bank_account` (Suggestion 1). Add JSONB `erp_metadata` (Suggestion 3) for per-ERP identifiers.
- Endpoints: `POST/GET/PATCH /api/v1/counterparties`, `POST /api/v1/counterparties/{id}/bank-accounts`.
- New vendor creation and bank-account add/change create a `pending_change` requiring a second approver (mitigation from `research.md` risk table). Status flow: `proposed → approved → applied | rejected`.
- Every applied change writes an `AuditEntry`.

**Testing**:
- `Unit: bank-account change request creates pending_change in 'proposed' state, not applied`.
- `Unit: same user approving own proposed change → SoDViolation`.
- `Integration: second approver approves → bank account row written, audit entry appended`.
- `Integration: counterparty with invalid type='vendor' → 422 (only supplier|customer|both)`.

#### 2.2 — Chart of accounts, cost centres, fiscal periods

**What**: GL account tree, cost centres, and fiscal-period lifecycle.

**Design**:
- Tables `gl_account` (self-referencing `parent_id`, type asset|liability|equity|revenue|expense), `cost_centre`.
- `POST /api/v1/gl-accounts`, `GET /api/v1/gl-accounts?type=expense&active=true`.
- Period transitions enforce ordering: cannot post to a `closed`/`locked` period (used by Phase 8).

**Testing**:
- `Unit: gl_account with duplicate (org, account_code) → IntegrityError`.
- `Unit: cyclic parent_id assignment → rejected by validator`.
- `Integration: posting attempt to locked period → 409 PeriodLockedError`.

#### 2.3 — Purchase orders & lines

**What**: PO and PO-line storage to support 3-way matching.

**Design**:
- Tables `purchase_order`, `purchase_order_line` (Suggestion 1), with `received_qty` for goods-receipt tracking.
- `POST /api/v1/purchase-orders`, status `draft|approved|partially_received|received|closed|cancelled`.

**Testing**:
- `Unit: PO total recomputed = sum(line.qty * line.unit_price)`.
- `Integration: duplicate (org, po_number) → IntegrityError`.

---

## Phase 3: Intelligent Invoice Processing (Core Value — Part 1)

### Purpose
The heart of the product. Ingest invoices in any format, extract structured fields with confidence scoring via multi-modal LLM, validate against master data, and predict GL coding. This is the AI-native replacement for template OCR (`README.md` AI-Native Advantage). After this phase a raw PDF/image/email/EDI becomes a validated, GL-coded `invoice` record with a captured reasoning chain.

### Tasks

#### 3.1 — Invoice & invoice-line model with extraction JSONB

**What**: Persist invoices and lines with hybrid relational+JSONB fields.

**Design**:
- Tables `invoice`, `invoice_line` (Suggestion 1) plus JSONB `extracted_fields` and JSONB `extraction_reasoning` (Suggestion 3). Status enum exactly as Suggestion 1 (`received → extracting → extracted → matching → matched → exception → pending_approval → approved → scheduled → paid | disputed | cancelled | written_off`).
- `extraction_confidence NUMERIC(5,4)`; lines carry `coding_confidence`.

**Testing**:
- `Unit: invoice status transition validator rejects received → paid (illegal jump)`.
- `Integration: insert invoice with extracted_fields JSONB, GIN query "fields containing vendor_template" → row found`.

#### 3.2 — Document ingestion & normalisation pipeline

**What**: Accept uploads (PDF, image, email .eml, EDI 810) and normalise to a common `RawDocument`.

**Design**:
- `POST /api/v1/invoices/ingest` (multipart) → stores raw doc, returns `invoice_id`, sets status `received`.
- Format detection by content sniff. EDI 810 routed to `integrations/edi`; PDFs/images to multimodal extraction; `.eml` parsed for body + attachments.
```python
@dataclass
class RawDocument:
    invoice_id: UUID
    media_type: Literal["pdf","image","email","edi810"]
    pages: list[bytes] | None      # image bytes per page for multimodal
    text: str | None
    source_uri: str
```

**Testing**:
- `Unit: sniff EDI 810 envelope (ISA*...*810) → media_type='edi810'`.
- `Integration: POST sample PDF (fixtures/inv_digital.pdf) → 201, status='received'`.
- `Integration: POST malformed file → 422 UnsupportedDocument`.

#### 3.3 — Multi-modal LLM extraction with confidence scoring

**What**: Extract header + line fields using Claude multimodal, returning per-field confidence.

**Design**:
- `ai/provider.py`:
```python
class LLMProvider(Protocol):
    async def extract(self, doc: RawDocument, schema: dict) -> ExtractionResult: ...

class ExtractionResult(BaseModel):
    invoice_number: FieldValue[str]
    invoice_date: FieldValue[date]
    due_date: FieldValue[date]
    currency: FieldValue[str]
    subtotal: FieldValue[Decimal]
    tax_amount: FieldValue[Decimal]
    total_amount: FieldValue[Decimal]
    supplier_name: FieldValue[str]
    lines: list[ExtractedLine]
    overall_confidence: Decimal
    reasoning: str

class FieldValue(BaseModel, Generic[T]):
    value: T | None
    confidence: Decimal     # 0..1
```
- Prompt template `prompts/extraction_v1.txt` (system): *"You are a finance document extraction agent. Extract the fields defined in the JSON schema. For each field return a value and a confidence 0–1 reflecting how certain you are from the document evidence. Never guess amounts; if illegible set value null and confidence 0. Return ONLY valid JSON matching the schema."* The schema (JSON Schema Draft 2020-12, per `standards.md`) is supplied in the user turn.
- Tool-use / structured-output mode requested so the model returns schema-valid JSON; response re-validated with Pydantic.
- Whole `reasoning` plus per-field confidences written to `extraction_reasoning` JSONB and to an `AuditEntry` (action `invoice.extracted`).

**Testing**:
- `Unit (mocked provider): provider returns canned ExtractionResult → invoice fields + extraction_confidence persisted`.
- `Unit: provider returns JSON missing 'total_amount' → ValidationError surfaced as ExtractionFailed, status='exception'`.
- `Unit: line amount returned as float string "15000.00" → coerced to Decimal exactly`.
- `Integration (real LLM, marked @pytest.mark.live): fixtures/inv_digital.pdf → overall_confidence > 0.8` (opt-in, skipped in CI default).

#### 3.4 — Master-data validation & confidence gating

**What**: Validate extracted supplier/amounts against `counterparty` master data and route low-confidence invoices to human review.

**Design**:
- `domain/` validator matches `supplier_name` to a `counterparty` (exact tax_id, else fuzzy name ≥ 0.9); arithmetic check `subtotal + tax_amount == total_amount` within ±0.01 tolerance.
- If `overall_confidence < settings.extraction_confidence_floor` OR any required field null OR arithmetic fails → status `exception`, reason recorded; else status `extracted`.

**Testing**:
- `Unit: confidence 0.70 (floor 0.85) → status='exception', reason='low_confidence'`.
- `Unit: subtotal 100 + tax 9 ≠ total 110 → status='exception', reason='arithmetic_mismatch'`.
- `Unit: supplier matches counterparty by tax_id → counterparty_id populated`.

#### 3.5 — Self-learning GL coding engine

**What**: Predict GL account + cost centre per line, learning from this supplier's prior invoices.

**Design**:
- `domain/coding.py`: for each line, gather the supplier's historical line → gl_account distribution; if a dominant code (>70% of prior lines with similar description embedding) exists, assign it with that frequency as confidence. Otherwise call LLM with the supplier's coding history as context (few-shot). Target <5% exception rate (features.md v1.1).
- Sets `invoice.gl_coding_auto = true` when all lines coded above threshold; lines below threshold flag for review.

**Testing**:
- `Unit: supplier with 14 prior lines all coded 6100 → new line predicted 6100, confidence ≥ 0.9`.
- `Unit: no history + LLM mock returns code 6200 conf 0.6 (floor 0.7) → line flagged for review`.
- `Unit: user correction of a coded line is appended to history and changes next prediction`.

---

## Phase 4: 3-Way Matching, Policy Engine & Approval Workflow (Core Value — Part 2)

### Purpose
Turn validated invoices into approved-or-escalated outcomes autonomously. Implements 3-way PO/GR/invoice matching with tolerance handling, a policy-as-code governance layer (the gap identified in `features.md`), and learned approval routing. After this phase an invoice flows automatically to `approved` (within policy) or to a human queue with an AI-generated resolution suggestion.

### Tasks

#### 4.1 — Policy-as-code governance layer

**What**: Versioned, auditable policies defining what agents may do autonomously.

**Design**:
- Tables `governance_policy` (Suggestion 1) with JSONB `parameters` (Suggestion 3). Types: `approval_threshold`, `sod`, `auto_post_limit`, `sanction_screen`, `duplicate_detect`, `fx_tolerance`, `match_tolerance`.
- `domain/policy.py`:
```python
class PolicyDecision(BaseModel):
    allowed: bool
    requires_human: bool
    citations: list[str]   # policy_code-vN
    reason: str

def evaluate(policies: list[GovernancePolicy], context: dict) -> PolicyDecision: ...
```
- Policies are immutable once published (new version on change); `effective_from`/`effective_to` window selects the active version. Publication writes an `AuditEntry`.

**Testing**:
- `Unit: invoice $450, threshold policy max_amount 5000 → allowed=True, requires_human=False`.
- `Unit: invoice $20000 → requires_human=True, citation includes 'ap-auto-approve-v3'`.
- `Unit: two policy versions, context date selects effective version`.
- `Unit: capex category excluded by policy → requires_human=True`.

#### 4.2 — 3-way matching engine

**What**: Match invoice lines to PO lines and goods receipts with configurable tolerances.

**Design**:
- Table `match_result` (Suggestion 1). `domain/matching.py` performs `po_match`, `receipt_match`, `price_match`, `quantity_match`; each yields `matched | tolerance | exception` based on `match_tolerance` policy (e.g. price ±2%, qty exact).
- Overall: all matched → `matched`; any within tolerance → `matched_with_tolerance`; any exception → invoice status `exception`.

**Testing**:
- `Unit: invoiced price 15000 vs PO 14800 (tol 2%) → variance 1.35% → 'tolerance'`.
- `Unit: invoiced qty 12 vs received 10 → quantity_match 'exception'`.
- `Unit: no PO linked + policy requires PO → exception, reason='missing_po'`.

#### 4.3 — AI resolution suggestions for exceptions

**What**: For each match/extraction exception, generate a human-readable resolution suggestion with policy citation.

**Design**:
- LLM call with the exception context + relevant policies → `ResolutionSuggestion(summary, recommended_action, citations, confidence)`. Stored on the exception and shown in the oversight console. Action written to `AuditEntry` with `decision='deferred'`.

**Testing**:
- `Unit (mocked): price-tolerance exception → suggestion text references the variance and the tolerance policy`.
- `Unit: suggestion missing citations → rejected, logged as low-quality, falls back to generic message`.

#### 4.4 — Approval routing with learned approver prediction

**What**: Route invoices through approval chains by amount/vendor/cost-centre, predicting the likely approver.

**Design**:
- Tables `approval_policy`, `approval_request` (Suggestion 1).
- Routing: policy selects required role; learned predictor ranks candidate approvers by historical approvals on similar (cost_centre, amount band, supplier). Recalculate routing if GL coding changes (Vic.ai pattern from features.md).
- `auto_approve=true` + within policy → `InvoiceAutoApproved` path: status `approved`, `AuditEntry decision='auto_approved'` with full reasoning chain.

**Testing**:
- `Unit: $450 invoice, auto_approve policy → status='approved', audit decision='auto_approved'`.
- `Unit: $20000 → approval_request created in 'pending' for ap_manager`.
- `Unit: re-coding changes cost centre → routing recalculated to new approver`.
- `Integration: approver from different org cannot see request → 404 (tenant scope)`.

---

## Phase 5: Durable Workflows & Human Oversight Console

### Purpose
Wrap the invoice lifecycle in a Temporal workflow so it survives restarts, retries idempotently, and pauses for human signals. Build the oversight console where humans monitor in-flight workflows and exercise pause/override/redirect — both are MVP must-haves (`features.md`). After this phase, ingestion-to-approval runs as one durable, observable, controllable workflow.

### Tasks

#### 5.1 — Invoice processing Temporal workflow

**What**: Orchestrate extract → validate → code → match → approve as a durable workflow.

**Design**:
- `workflows/invoice_workflow.py` `InvoiceWorkflow.run(invoice_id)` calls activities `extract`, `validate`, `code`, `match`, `evaluate_policy`. Each activity is idempotent (keyed by `invoice_id` + step) so retries never double-write.
- Human approval implemented as `await workflow.wait_condition()` on an `approval_signal`; a timer escalates after configurable SLA.
- Pause/redirect handled via signals `pause`, `resume`, `redirect(target_role)`.

**Testing**:
- `Integration (Temporal test env, time-skip): happy path → workflow completes, invoice status 'approved'`.
- `Integration: activity fails once then succeeds on retry → no duplicate invoice_line rows (idempotency)`.
- `Integration: pause signal → workflow halts; resume → continues`.
- `Integration: approval timer fires → escalation activity invoked`.

#### 5.2 — Oversight console API (live feed + queues)

**What**: APIs powering the dashboard: in-flight workflows, exception queues, pending approvals, agent activity feed.

**Design**:
- `GET /api/v1/workflows/in-flight`, `GET /api/v1/exceptions`, `GET /api/v1/approvals?status=pending`, `GET /api/v1/activity` (reads from the audit ledger).
- `POST /api/v1/workflows/{id}/pause|resume|redirect`, `POST /api/v1/approvals/{id}/decide` (sends Temporal signal; enforces SoD and role).
- Live updates over SSE `GET /api/v1/activity/stream` fed by Redis pub/sub on each audit append.

**Testing**:
- `Integration: pause via API → Temporal signal sent, workflow paused, audit entry written`.
- `Integration: decide approval with self-approval → 409 SoDViolation`.
- `Integration: SSE stream receives event within 1s of an audit append`.

#### 5.3 — Oversight console web UI (Next.js)

**What**: Dashboard UI for workflows, exceptions, approvals, and the live agent feed.

**Design**:
- Server Components for the queue tables; a client component subscribes to the SSE feed. Approve/Reject/Pause/Redirect buttons call the Phase 5.2 APIs. Each list row links to a detail drawer showing the full reasoning chain and policy citations.
- Auth via OIDC session cookie; role-gated actions hidden for unauthorised roles.

**Testing**:
- `E2E (Playwright, mocked API): load dashboard → in-flight workflows render`.
- `E2E: click Approve on a pending invoice → API called, row moves to approved list`.
- `E2E: live feed shows a new agent action pushed via SSE`.

---

## Phase 6: Payment Safety, Execution Staging & Sanction Screening

### Purpose
The highest-risk part of the system. Stage approved invoices into payment batches with mandatory human gates above threshold, duplicate detection, and OFAC/HM Treasury sanction screening — all MVP must-haves (`features.md`, `research.md` risk table). Payments are staged for ERP submission (Phase 7), never fired blindly at bank APIs.

### Tasks

#### 6.1 — Payment & batch model with idempotency

**What**: Persist payments and batches with a duplicate-detection hash and idempotency keys.

**Design**:
- Tables `payment_batch`, `payment` (Suggestion 1). `duplicate_check_hash = sha256(counterparty_id|amount|currency|invoice_number)`. Idempotency keys stored in Redis to make `SchedulePayment` safe under retries.

**Testing**:
- `Unit: same (supplier, amount, invoice_number) → identical duplicate_check_hash`.
- `Integration: scheduling same payment twice with same idempotency key → single payment row`.

#### 6.2 — Duplicate payment detection

**What**: Block payments matching a prior payment within a configurable window.

**Design**:
- `domain/duplicate.py`: on schedule, query payments with same `duplicate_check_hash` within N days (policy `duplicate_detect`). On hit → block, status stays unscheduled, `AuditEntry decision='rejected'`.

**Testing**:
- `Unit: duplicate within 7-day window → blocked`.
- `Unit: same hash 30 days apart, window 7 → allowed`.

#### 6.3 — Sanction screening (OFAC / HM Treasury)

**What**: Screen counterparty against sanction lists before any payment is staged.

**Design**:
- `integrations/sanctions/`: daily loader ingests OFAC SDN + HM Treasury consolidated list into Postgres + Redis. `SanctionScreener.screen(counterparty) -> ScreenResult(status: clear|flagged|blocked, matches)` uses normalised-name fuzzy match + alias check. `flagged`/`blocked` halts payment, escalates, writes audit entry.

**Testing**:
- `Unit: counterparty 'BLOCKED ENTITY LLC' present on SDN fixture → status='blocked'`.
- `Unit: near-match alias above 0.92 → status='flagged'`.
- `Unit: clean counterparty → status='clear', payment proceeds`.

#### 6.4 — Payment Temporal workflow with mandatory human gate

**What**: Durable payment workflow: dedup → sanction → threshold gate → stage.

**Design**:
- `workflows/payment_workflow.py`: runs dedup + sanction activities, then if `amount > settings.payment_auto_approve_threshold` waits for human approval signal (non-negotiable gate, `research.md`). On approval, stages payment (status `approved`) ready for ERP submission. SoD: the approver cannot be the invoice approver.

**Testing**:
- `Integration: payment below threshold, clear sanction, no dup → staged automatically`.
- `Integration: payment above threshold → waits; human signal approves → staged`.
- `Integration: sanction 'blocked' → workflow terminates, never staged, audit decision='rejected'`.
- `Integration: invoice approver == payment approver → SoDViolation`.

---

## Phase 7: ERP Integration Layer

### Purpose
Connect the platform bi-directionally to ERPs through an ERP-agnostic adapter interface (the open data-layer gap in `features.md`). MVP ships NetSuite, QuickBooks, and Xero; the same interface supports SAP and Oracle Fusion in v1.0. Payments are written to the ERP as approved payment runs, not submitted to banks.

### Tasks

#### 7.1 — ERP connector interface & sync tracking

**What**: Abstract connector protocol plus connection/sync-log tables.

**Design**:
- Tables `erp_connection`, `erp_sync_log` (Suggestion 1).
```python
class ERPConnector(Protocol):
    async def pull_master_data(self) -> MasterDataBatch: ...   # vendors, GL, cost centres
    async def push_invoice(self, inv: Invoice) -> ErpRef: ...
    async def push_payment_run(self, batch: PaymentBatch) -> ErpRef: ...
    async def pull_purchase_orders(self) -> list[PurchaseOrder]: ...
```
- OAuth 2.0 token management per connector (RFC 6749/7523); `erp_metadata` JSONB holds ERP-native IDs.
- Conflict handling: outbound push records `status` `synced|failed|conflict`; conflicts surface to oversight console.

**Testing**:
- `Unit: sync_log records inbound/outbound with erp_ref`.
- `Unit: token expiry triggers refresh before call`.

#### 7.2 — NetSuite, QuickBooks, Xero connectors (MVP)

**What**: Concrete connectors using each platform's REST API + OAuth 2.0.

**Design**:
- NetSuite: SuiteTalk REST + SuiteQL, Token-Based Auth/OAuth 2.0 (SOAP deprecated 2026.1 per `standards.md`).
- QuickBooks: REST/JSON, OAuth 2.0 auth-code + refresh tokens; map Invoice↔Bill, vendor, payment.
- Xero: REST/JSON, OAuth 2.0 + PKCE; webhooks for invoice/payment/bank-transaction events.
- Each maps to/from the internal hybrid model; pagination via `Link` header (RFC 8288).

**Testing**:
- `Integration (mocked HTTP via respx): NetSuite pull_master_data → counterparties + gl_accounts upserted`.
- `Integration (mocked): QuickBooks push_invoice → POST /bill called, erp_ref stored`.
- `Integration (mocked): Xero webhook 'invoice.updated' → local invoice resynced`.
- `Integration (mocked): 401 from ERP → token refresh attempted then retried`.

#### 7.3 — EDI 810 ingest / EDI 820 emit

**What**: Parse inbound EDI 810 invoices; emit EDI 820 remittance advice.

**Design**:
- `integrations/edi/`: X12 parser for 810 (and 850/856 for matching context), serialiser for 820. Maps EDI segments to internal invoice/payment models.

**Testing**:
- `Unit: parse fixtures/edi810_sample.x12 → invoice header + lines correct`.
- `Unit: emit EDI 820 for a payment batch → round-trips through parser to equal totals`.

---

## Phase 8: AR Collections, Reconciliation, Forecasting & Close (v1.1 Capabilities)

### Purpose
Extend coverage to the AR, treasury, and close functions that make this the only open platform spanning AP+AR+treasury+close (`features.md` underserved area). Implements bank reconciliation (MVP must-have), collections, 13-week forecasting, and month-end close acceleration.

### Tasks

#### 8.1 — Bank statement ingest (BAI2 / MT940 / CAMT.053) & AI reconciliation

**What**: Parse all three statement formats and fuzzy-match transactions to payments/ledger entries.

**Design**:
- Tables `bank_account`, `bank_statement`, `bank_transaction` (Suggestion 1). Parsers in `integrations/banking/` (BAI2/MT940 text, CAMT.053 via `lxml`).
- `domain/reconciliation.py`: match by exact amount + date window + fuzzy payee/reference (token-set ratio). Confidence ≥ policy threshold → auto-match; else surface as unmatched. Optional Plaid adapter for direct bank feeds (`standards.md`).

**Testing**:
- `Unit: parse fixtures/sample.bai2 → opening/closing balances and N transactions`.
- `Unit: parse fixtures/sample.camt053.xml → balances match`.
- `Unit: exact amount + 8/12 ref chars + 1-day gap → matched, confidence ≈ 0.9`.
- `Integration: unmatched transaction → appears in reconciliation exceptions queue`.

#### 8.2 — AR collections with payment-probability scoring & adaptive dunning

**What**: Prioritised collections queue and learning dunning workflow.

**Design**:
- Tables `collection_queue`, `dunning_action`, `dispute` (Suggestion 1). `domain/collections.py` scores payment probability (scikit-learn gradient-boosted model on ageing, payment history, dispute flags); `priority_score` orders the queue.
- `workflows/collections_workflow.py`: adaptive outreach — channel/timing chosen from prior response history (Billtrust pattern); LLM drafts dunning email; each action audited.

**Testing**:
- `Unit: 42 days overdue + low historical pay rate → high priority_score`.
- `Unit: customer responds to email not phone in 3/4 cases → next action channel='email'`.
- `Integration: dunning workflow advances level after no-response timer`.

#### 8.3 — Rolling 13-week cash-flow forecast

**What**: Daily-refreshed 13-week forecast from AP/AR ageing, payroll, committed spend, with scenarios.

**Design**:
- Tables `cash_forecast`, `cash_forecast_line` (Suggestion 1). `domain/forecast.py` buckets expected AP outflows (by due date + payment probability), AR inflows (by collection probability), and committed spend across 13 weekly buckets; produces `best|base|worst` scenarios. A Temporal cron workflow regenerates daily.

**Testing**:
- `Unit: invoice due in week 3 → projected outflow in correct bucket`.
- `Unit: worst-case applies haircut to AR inflows`.
- `Integration: cron workflow produces a fresh forecast row daily`.

#### 8.4 — Month-end close acceleration

**What**: AI-driven close checklist that auto-posts accrual/prepayment journals and balances debits/credits.

**Design**:
- Tables `journal_entry`, `journal_line` (Suggestion 1; `CHECK` enforces non-negative, no both-sided lines), `close_checklist`, `close_task`. `workflows/close_workflow.py` runs tasks in order; accrual agent posts standard journals citing source agreements; controller certifies. Posting blocked on `closed`/`locked` periods (Phase 2.2). AI-posted entries carry a distinct `source_type='agent'` so auditors review them as a group (`research.md` mitigation).

**Testing**:
- `Unit: journal entry where sum(debits) != sum(credits) → BalanceError`.
- `Unit: line with both debit>0 and credit>0 → CHECK violation`.
- `Integration: close workflow auto-posts rent accrual, journal balanced, audit entry written`.
- `Integration: certify period → status 'closed', further postings rejected`.

---

## Phase 9: MCP Server, OpenAPI Hardening & Compliance Reporting

### Purpose
Expose the platform to external AI clients via MCP, finalise the public API surface against OWASP/PCI standards, and produce auditor-facing compliance outputs (XBRL-tagged close reports, audit-chain verification). Delivers the AI-interoperability and explainability differentiators that satisfy controllers and the EU AI Act high-risk-system requirements (`standards.md`).

### Tasks

#### 9.1 — MCP server exposing governed finance tools

**What**: An MCP server (spec 2025-11-25) exposing read/act tools, all routed through policy + audit.

**Design**:
- `mcp/server.py` tools: `get_invoice_status`, `get_cash_position`, `list_open_payables`, `trigger_payment_run`, `get_bank_reconciliation_exceptions`. Every tool call authenticates the calling agent, evaluates governance policy, and writes an `AuditEntry` with `agent_id` = the MCP client identity. `trigger_payment_run` is subject to the same human-gate workflow as Phase 6.

**Testing**:
- `Integration: MCP get_cash_position → returns current balances, audit entry written`.
- `Integration: MCP trigger_payment_run above threshold → staged pending human gate, not executed`.
- `Unit: unauthenticated MCP client → tool call rejected`.

#### 9.2 — OpenAPI 3.1 hardening & OWASP/PCI checks

**What**: Finalise the OpenAPI spec and add automated security scanning.

**Design**:
- All routes documented with schemas (OpenAPI 3.1). CI runs an OWASP API Top 10 check (BOLA, BFLA, excessive data exposure) and bandit; per `standards.md` PCI DSS 6.2.4 maintains an API inventory artifact generated from the spec.
- Response models stripped of sensitive fields (bank account numbers masked except for authorised roles).

**Testing**:
- `Integration: cross-tenant object access (BOLA) → 404, scanner asserts no leakage`.
- `Integration: bank_account_number masked for non-treasurer role`.
- `CI: bandit + OWASP scan pass with zero high findings`.

#### 9.3 — Compliance reporting: audit-chain verification & XBRL close output

**What**: Auditor endpoints to verify the audit chain and export close reports.

**Design**:
- `GET /api/v1/audit/verify` → runs `verify_chain` (Phase 1.3), returns validity + first broken link.
- `GET /api/v1/close/{period}/report?format=ixbrl` → renders close output as Inline XBRL (iXBRL) per `standards.md`, embedding XBRL tags in audit-ready HTML.

**Testing**:
- `Integration: tamper a partition row → /audit/verify reports invalid with position`.
- `Unit: iXBRL output validates against XBRL schema for tagged monetary facts`.
- `Integration: auditor role can access; ap_clerk → 403`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Data Model & Audit Ledger      ─── required by everything
    │
Phase 2: Master Data & Chart of Accounts            ─── requires Phase 1
    │
Phase 3: Intelligent Invoice Processing             ─── requires Phase 2
    │
Phase 4: Matching, Policy & Approval                ─── requires Phase 3
    │
Phase 5: Durable Workflows & Oversight Console      ─── requires Phase 4
    │
    ├── Phase 6: Payment Safety & Execution Staging  ─── requires Phase 5
    │       │
    │       └── Phase 7: ERP Integration Layer       ─── requires Phase 6 (and Phase 2 master data)
    │
    └── Phase 8: AR / Reconciliation / Forecast / Close ─── requires Phase 5; can parallel with Phase 6+7
            │
Phase 9: MCP, OpenAPI Hardening & Compliance        ─── requires Phases 6, 7, 8
```

Parallelism:
- After Phase 5, **Phase 6 (+7)** and **Phase 8** can be built by separate streams — payments/ERP vs AR/treasury/close — sharing only the audit ledger and workflow infrastructure.
- Within Phase 7, the three MVP connectors (7.2) and the EDI module (7.3) are independent and parallelisable.
- Within Phase 8, sub-tasks 8.1–8.4 are largely independent given the Phase 1–5 foundation.

Scope checkpoints:
- **MVP** completes at end of Phase 7 minus 7.3 deferral acceptable: invoice processing, matching, approval, oversight, payment safety, NetSuite/QuickBooks/Xero, bank reconciliation (pull 8.1 forward if shipping recon in MVP).
- **v1.1** = remainder of Phase 8 (collections, forecast, close).
- **Backlog** = Phase 9 MCP/compliance extras, plus SAP/Oracle connectors, FX hedging, agentic VoIP (not planned here).

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented per their Design sections.
2. All unit tests and mocked-integration tests pass; real-dependency integration tests (testcontainers / Temporal test env) pass in CI; `@pytest.mark.live` LLM/ERP tests pass locally when run.
3. `ruff check` and `ruff format --check` pass with zero errors.
4. `mypy --strict` passes for `src/afo`.
5. `bandit` reports zero high-severity findings; for Phases 6–9, the OWASP API Top 10 check passes.
6. `docker compose up` brings the stack healthy (`/health` all-true) and the phase's feature works end-to-end against it.
7. New configuration options are documented in `.env.example` and `config/default.yaml`.
8. New API endpoints appear in the generated OpenAPI 3.1 spec at `/docs` with request/response schemas.
9. New tables/columns have a corresponding Alembic migration that applies and rolls back cleanly.
10. Every new agent action or state-changing API path writes an `AuditEntry`, and `verify_chain` remains valid after the phase's tests run.
```
