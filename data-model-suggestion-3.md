# Data Model Suggestion 3: Hybrid Relational + JSONB Model (PostgreSQL)

## Approach

A PostgreSQL schema that keeps core financial entities in normalised relational tables for integrity and query performance, while using JSONB columns for semi-structured and variable data: AI reasoning chains, extraction results, policy parameters, ERP-specific metadata, and vendor-specific invoice fields. This approach captures the best of both worlds -- relational rigour where it matters (money, approvals, GL postings) and schema flexibility where data structures vary by context (AI outputs, multi-ERP integration, document extraction).

## Why This Suits Autonomous Finance Operations

1. **Variable document structures.** Invoice extraction from PDF, email, EDI 810, and scanned images produces different field sets. A JSONB `extracted_fields` column accommodates any vendor format without schema migrations.
2. **AI reasoning is inherently semi-structured.** Agent reasoning chains, confidence breakdowns, and policy evaluations vary by agent type and version. JSONB stores these naturally while GIN indexes enable querying ("find all invoices where the agent cited policy X").
3. **Multi-ERP metadata.** NetSuite, SAP, Xero, and QuickBooks each have different entity identifiers, custom fields, and sync payloads. A JSONB `erp_metadata` column per entity avoids an explosion of nullable ERP-specific columns.
4. **Policy parameters evolve.** Governance policies have different parameter shapes (threshold amounts, role lists, category exclusions, time windows). JSONB handles this without a key-value table.
5. **Core financial data stays relational.** Amounts, dates, account codes, approval chains, and payment statuses remain in typed columns with constraints, indexes, and foreign keys -- no compromises on financial integrity.

## Trade-offs

- **Strengths:** Single database technology (PostgreSQL). No impedance mismatch between "structured" and "flexible" data. GIN indexes on JSONB provide fast lookups. Easier schema evolution than pure 3NF for rapidly changing AI features.
- **Weaknesses:** JSONB data lacks referential integrity (no FK inside JSON). Complex JSONB queries can be slower than joins on indexed columns. Developers must maintain discipline about what goes in JSONB vs. typed columns -- the boundary can blur over time.
- **Scalability:** Same as normalised PostgreSQL -- partitioning, read replicas, connection pooling. JSONB columns add storage overhead compared to normalised tables but compress well.

---

## Schema Definition

### Organisation and Configuration

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_entity_id VARCHAR(50),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',  -- locale, date formats, feature flags
    erp_config      JSONB NOT NULL DEFAULT '{}',  -- ERP-specific connection and mapping config
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE fiscal_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    period_name     VARCHAR(20) NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','closing','closed','locked')),
    close_metadata  JSONB DEFAULT '{}',           -- close timestamps, certifier info, notes
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, period_name)
);
CREATE INDEX idx_fiscal_period_org ON fiscal_period(organisation_id, status);
```

### Counterparties

```sql
CREATE TABLE counterparty (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    type            VARCHAR(10) NOT NULL CHECK (type IN ('supplier','customer','both')),
    legal_name      VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    tax_id          VARCHAR(50),
    payment_terms   INTEGER DEFAULT 30,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    risk_score      NUMERIC(5,2),
    sanction_status VARCHAR(20) DEFAULT 'clear'
                    CHECK (sanction_status IN ('clear','flagged','blocked')),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','blocked')),
    -- JSONB for variable counterparty data
    bank_accounts   JSONB NOT NULL DEFAULT '[]',  -- array of bank account objects
    contact_info    JSONB NOT NULL DEFAULT '{}',  -- emails, phones, addresses by type
    tax_forms       JSONB NOT NULL DEFAULT '{}',  -- W-9, W-8BEN, VAT registrations
    erp_refs        JSONB NOT NULL DEFAULT '{}',  -- {"netsuite": "VND-001", "xero": "abc-123"}
    custom_fields   JSONB NOT NULL DEFAULT '{}',  -- organisation-specific extensions
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_counterparty_org_type ON counterparty(organisation_id, type);
CREATE INDEX idx_counterparty_tax_id ON counterparty(tax_id);
CREATE INDEX idx_counterparty_erp_refs ON counterparty USING GIN (erp_refs);
```

### Chart of Accounts

```sql
CREATE TABLE gl_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    account_code    VARCHAR(20) NOT NULL,
    account_name    VARCHAR(255) NOT NULL,
    account_type    VARCHAR(20) NOT NULL
                    CHECK (account_type IN ('asset','liability','equity','revenue','expense')),
    parent_id       UUID REFERENCES gl_account(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    erp_mappings    JSONB NOT NULL DEFAULT '{}',  -- {"netsuite": {"internal_id": 142}, "xero": {"code": "200"}}
    tags            JSONB NOT NULL DEFAULT '[]',  -- ["capex","recurring","discretionary"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, account_code)
);
CREATE INDEX idx_gl_account_org ON gl_account(organisation_id, account_type);
CREATE INDEX idx_gl_account_tags ON gl_account USING GIN (tags);

CREATE TABLE cost_centre (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            VARCHAR(20) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    metadata        JSONB NOT NULL DEFAULT '{}',
    UNIQUE (organisation_id, code)
);
```

### Purchase Orders

```sql
CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    po_number       VARCHAR(50) NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES counterparty(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','approved','partially_received','received','closed','cancelled')),
    currency        CHAR(3) NOT NULL,
    total_amount    NUMERIC(18,4) NOT NULL DEFAULT 0,
    issued_date     DATE,
    lines           JSONB NOT NULL DEFAULT '[]',  -- array of line items with qty, price, GL, cost centre
    receipt_history JSONB NOT NULL DEFAULT '[]',  -- goods receipt records
    erp_refs        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, po_number)
);
CREATE INDEX idx_po_org_status ON purchase_order(organisation_id, status);
CREATE INDEX idx_po_supplier ON purchase_order(supplier_id);
```

### Invoices

```sql
CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    direction       VARCHAR(2) NOT NULL CHECK (direction IN ('ap','ar')),
    invoice_number  VARCHAR(100) NOT NULL,
    counterparty_id UUID NOT NULL REFERENCES counterparty(id),
    purchase_order_id UUID REFERENCES purchase_order(id),
    fiscal_period_id UUID REFERENCES fiscal_period(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'received'
                    CHECK (status IN (
                        'received','extracting','extracted','matching',
                        'matched','exception','pending_approval','approved',
                        'scheduled','paid','disputed','cancelled','written_off'
                    )),
    currency        CHAR(3) NOT NULL,
    subtotal        NUMERIC(18,4) NOT NULL,
    tax_amount      NUMERIC(18,4) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(18,4) NOT NULL,
    due_date        DATE,
    received_date   DATE NOT NULL DEFAULT CURRENT_DATE,

    -- JSONB columns for variable/AI-generated data
    extracted_fields JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "raw_fields": {"vendor_name": "Acme Corp", "po_ref": "PO-123", ...},
    --   "confidence_scores": {"vendor_name": 0.99, "total": 0.97, "line_items": 0.91},
    --   "extraction_model": "doc-ai-v4.2",
    --   "source_format": "pdf",
    --   "ocr_text": "..."
    -- }

    line_items      JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"line": 1, "description": "Consulting", "qty": 1, "unit_price": 15000,
    --    "amount": 15000, "gl_account_id": "uuid", "cost_centre_id": "uuid",
    --    "coding_confidence": 0.94, "coding_reasoning": "Matched 14 prior invoices"}
    -- ]

    match_results   JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "match_type": "three_way",
    --   "overall_status": "matched",
    --   "checks": [
    --     {"type": "po_match", "status": "matched", "variance_pct": 0.0},
    --     {"type": "price_match", "status": "tolerance", "expected": 14800, "actual": 15000}
    --   ]
    -- }

    approval_history JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"action": "auto_approved", "agent_id": "ap-agent", "policy": "pol-001",
    --    "reasoning": "...", "at": "2026-05-29T10:00:00Z"},
    --   {"action": "overridden", "by": "user-controller", "reason": "...", "at": "..."}
    -- ]

    source_document_url TEXT,
    erp_refs        JSONB NOT NULL DEFAULT '{}',
    tags            JSONB NOT NULL DEFAULT '[]',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_invoice_org_direction ON invoice(organisation_id, direction);
CREATE INDEX idx_invoice_status ON invoice(organisation_id, status);
CREATE INDEX idx_invoice_counterparty ON invoice(counterparty_id);
CREATE INDEX idx_invoice_due_date ON invoice(due_date);
CREATE INDEX idx_invoice_erp_refs ON invoice USING GIN (erp_refs);
CREATE INDEX idx_invoice_match_status ON invoice USING GIN (match_results jsonb_path_ops);
CREATE INDEX idx_invoice_tags ON invoice USING GIN (tags);
-- Partial index for active AP invoices needing attention
CREATE INDEX idx_invoice_ap_active ON invoice(organisation_id, due_date)
    WHERE direction = 'ap' AND status NOT IN ('paid','cancelled','written_off');
```

### Payments

```sql
CREATE TABLE payment_batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    batch_reference VARCHAR(50) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','approved','submitted','processing',
                                      'completed','partially_failed','failed')),
    payment_date    DATE NOT NULL,
    total_amount    NUMERIC(18,4) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL,
    summary         JSONB NOT NULL DEFAULT '{}',  -- payment count, method breakdown, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    submitted_at    TIMESTAMPTZ,
    UNIQUE (organisation_id, batch_reference)
);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id        UUID REFERENCES payment_batch(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    direction       VARCHAR(2) NOT NULL CHECK (direction IN ('ap','ar')),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    counterparty_id UUID NOT NULL REFERENCES counterparty(id),
    amount          NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    fx_rate         NUMERIC(18,8) DEFAULT 1.0,
    payment_method  VARCHAR(20) NOT NULL
                    CHECK (payment_method IN ('ach','wire','check','card','virtual_card')),
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','approved','submitted','cleared',
                                      'failed','reversed','cancelled')),
    duplicate_check_hash VARCHAR(64),
    -- JSONB for compliance and processing details
    compliance      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "sanction_screened": true, "sanction_result": "clear",
    --   "duplicate_check": {"checked": true, "result": "unique", "hash": "sha256:..."},
    --   "screening_timestamp": "2026-05-29T10:00:00Z"
    -- }

    processing      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "submitted_at": "...", "cleared_at": "...", "bank_ref": "...",
    --   "failure_reason": null, "retry_count": 0
    -- }

    erp_refs        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_payment_invoice ON payment(invoice_id);
CREATE INDEX idx_payment_status ON payment(organisation_id, status);
CREATE INDEX idx_payment_dup_hash ON payment(duplicate_check_hash);
CREATE INDEX idx_payment_compliance ON payment USING GIN (compliance jsonb_path_ops);
```

### Bank Accounts and Reconciliation

```sql
CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    bank_name       VARCHAR(255) NOT NULL,
    account_number  VARCHAR(50),
    iban            VARCHAR(34),
    swift_bic       VARCHAR(11),
    currency        CHAR(3) NOT NULL,
    current_balance NUMERIC(18,4),
    balance_as_of   TIMESTAMPTZ,
    gl_account_id   UUID REFERENCES gl_account(id),
    connection_config JSONB NOT NULL DEFAULT '{}',  -- bank feed credentials, format preferences
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE bank_statement (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    statement_date  DATE NOT NULL,
    format          VARCHAR(10) NOT NULL CHECK (format IN ('bai2','mt940','camt053','csv')),
    opening_balance NUMERIC(18,4) NOT NULL,
    closing_balance NUMERIC(18,4) NOT NULL,
    raw_file_url    TEXT,
    parse_metadata  JSONB NOT NULL DEFAULT '{}',  -- parser version, warnings, format-specific fields
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE bank_transaction (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id    UUID NOT NULL REFERENCES bank_statement(id),
    bank_account_id UUID NOT NULL REFERENCES bank_account(id),
    transaction_date DATE NOT NULL,
    value_date      DATE,
    amount          NUMERIC(18,4) NOT NULL,
    direction       VARCHAR(6) NOT NULL CHECK (direction IN ('debit','credit')),
    reference       TEXT,
    description     TEXT,
    bank_ref        VARCHAR(100),
    reconciled      BOOLEAN NOT NULL DEFAULT false,
    matched_payment_id UUID REFERENCES payment(id),

    -- JSONB for AI matching details
    match_detail    JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "confidence": 0.92,
    --   "method": "ai_fuzzy",
    --   "reasoning": "Amount exact match. Reference partial (8/12 chars). Date within 2-day window.",
    --   "candidate_scores": [
    --     {"payment_id": "uuid", "score": 0.92},
    --     {"payment_id": "uuid", "score": 0.45}
    --   ]
    -- }

    reconciled_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bank_txn_account_date ON bank_transaction(bank_account_id, transaction_date);
CREATE INDEX idx_bank_txn_unreconciled ON bank_transaction(bank_account_id) WHERE NOT reconciled;
CREATE INDEX idx_bank_txn_match ON bank_transaction USING GIN (match_detail jsonb_path_ops);
```

### Journal Entries

```sql
CREATE TABLE journal_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    entry_number    VARCHAR(50) NOT NULL,
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    entry_type      VARCHAR(20) NOT NULL
                    CHECK (entry_type IN ('standard','accrual','reversal','adjustment','closing')),
    description     TEXT,
    source_type     VARCHAR(20),
    source_id       UUID,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','pending_approval','posted','reversed')),
    -- JSONB for line items (typically 2-10 lines per entry)
    lines           JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"gl_account_id": "uuid", "gl_code": "6200", "cost_centre_id": "uuid",
    --    "debit": 8500.00, "credit": 0, "currency": "USD", "fx_rate": 1.0,
    --    "base_debit": 8500.00, "base_credit": 0, "description": "Monthly rent"},
    --   {"gl_account_id": "uuid", "gl_code": "2100", ...}
    -- ]

    total_debits    NUMERIC(18,4) NOT NULL DEFAULT 0,   -- denormalised for fast validation
    total_credits   NUMERIC(18,4) NOT NULL DEFAULT 0,
    agent_context   JSONB NOT NULL DEFAULT '{}',         -- agent reasoning if auto-posted
    erp_refs        JSONB NOT NULL DEFAULT '{}',
    posted_at       TIMESTAMPTZ,
    posted_by       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, entry_number),
    CHECK (total_debits = total_credits)                 -- enforce balanced entries
);
CREATE INDEX idx_journal_org_period ON journal_entry(organisation_id, fiscal_period_id);
CREATE INDEX idx_journal_status ON journal_entry(status);
CREATE INDEX idx_journal_lines ON journal_entry USING GIN (lines);
```

### Cash Flow Forecasting

```sql
CREATE TABLE cash_forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    forecast_date   DATE NOT NULL,
    scenario        VARCHAR(10) NOT NULL DEFAULT 'base'
                    CHECK (scenario IN ('best','base','worst')),
    horizon_weeks   INTEGER NOT NULL DEFAULT 13,
    model_version   VARCHAR(20),
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- JSONB for the full weekly breakdown
    weekly_projections JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"week_start": "2026-06-02", "ap_outflows": -85000, "ar_inflows": 120000,
    --    "payroll": -65000, "other": -5000, "net": -35000, "cumulative": 180000,
    --    "confidence": 0.85},
    --   ...
    -- ]

    summary         JSONB NOT NULL DEFAULT '{}',
    -- {"total_inflows": 1560000, "total_outflows": -1340000, "min_balance": 95000,
    --  "min_balance_week": "2026-07-14", "fx_exposure": {"EUR": -45000, "GBP": 12000}}

    model_inputs    JSONB NOT NULL DEFAULT '{}'   -- what data fed the forecast
);
CREATE INDEX idx_forecast_org_date ON cash_forecast(organisation_id, forecast_date);
```

### Collections and Disputes (AR)

```sql
CREATE TABLE collection_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    customer_id     UUID NOT NULL REFERENCES counterparty(id),
    priority_score  NUMERIC(8,4),
    days_overdue    INTEGER NOT NULL DEFAULT 0,
    dunning_level   INTEGER NOT NULL DEFAULT 0,
    next_action_date DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','promised','disputed','escalated','resolved')),

    -- JSONB for interaction history and AI strategy
    outreach_history JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"type": "email", "template": "dunning-2", "sent_at": "...",
    --    "response": "promised_payment", "responded_at": "...",
    --    "strategy_reasoning": "Customer responds to email, not phone. Escalating tone."}
    -- ]

    scoring_detail  JSONB NOT NULL DEFAULT '{}',
    -- {"model": "collections-ml-v2", "payment_probability": 0.35,
    --  "factors": {"days_overdue": 0.3, "payment_history": 0.25, "amount": 0.15, "industry": 0.1}}

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_collection_priority ON collection_queue(organisation_id, priority_score DESC);
CREATE INDEX idx_collection_status ON collection_queue(organisation_id, status);

CREATE TABLE dispute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    counterparty_id UUID NOT NULL REFERENCES counterparty(id),
    reason_code     VARCHAR(50) NOT NULL,
    disputed_amount NUMERIC(18,4) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','investigating','resolved','written_off')),
    resolution      JSONB NOT NULL DEFAULT '{}',
    -- {"resolution_type": "credit_note", "credit_amount": 500, "root_cause": "pricing_error",
    --  "resolved_by": "user-ar-mgr", "notes": "Supplier acknowledged incorrect rate"}
    timeline        JSONB NOT NULL DEFAULT '[]',  -- array of status changes with timestamps
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
CREATE INDEX idx_dispute_org_status ON dispute(organisation_id, status);
```

### Month-End Close

```sql
CREATE TABLE close_checklist (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'not_started'
                    CHECK (status IN ('not_started','in_progress','completed','certified')),
    -- JSONB for the full task list with flexible task definitions
    tasks           JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"task_id": "uuid", "order": 1, "name": "Accrue utilities", "type": "accrual",
    --    "assigned_to": "ai_agent", "status": "completed", "journal_entry_id": "uuid",
    --    "started_at": "...", "completed_at": "...",
    --    "agent_notes": "Posted $3,200 based on avg of prior 3 months. Variance < 5%."},
    --   ...
    -- ]
    summary         JSONB NOT NULL DEFAULT '{}',  -- progress stats, blockers
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    certified_by    UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_close_org_period ON close_checklist(organisation_id, fiscal_period_id);
```

### Agent Governance and Audit

```sql
CREATE TABLE governance_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    policy_code     VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    policy_type     VARCHAR(30) NOT NULL
                    CHECK (policy_type IN ('approval_threshold','sod','auto_post_limit',
                                           'sanction_screen','duplicate_detect','fx_tolerance')),
    -- JSONB for flexible policy parameters
    parameters      JSONB NOT NULL DEFAULT '{}',
    -- Example for approval_threshold:
    -- {"max_auto_amount": 5000, "currency": "USD", "require_po_match": true,
    --  "excluded_categories": ["capex"], "escalation_role": "ap_manager"}
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    change_history  JSONB NOT NULL DEFAULT '[]',  -- version diffs
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, policy_code, version)
);
CREATE INDEX idx_policy_org_active ON governance_policy(organisation_id) WHERE is_active;
CREATE INDEX idx_policy_params ON governance_policy USING GIN (parameters jsonb_path_ops);

CREATE TABLE agent_audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    agent_id        VARCHAR(100) NOT NULL,
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    -- JSONB for the full reasoning and context
    reasoning       JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "chain": ["Received invoice INV-2026-0042 from Acme Corp",
    --             "Extracted 1 line item with 96% confidence",
    --             "Matched to PO-321, price variance 1.35% within 2% tolerance",
    --             "Amount $16,350 exceeds auto-approve threshold $10,000",
    --             "Escalating to ap_manager per policy pol-ap-001 v3"],
    --   "policy_citations": [{"code": "pol-ap-001", "version": 3, "clause": "max_auto_amount"}],
    --   "confidence": 0.97,
    --   "model_version": "ap-agent-v3.1",
    --   "input_hash": "sha256:...",
    --   "duration_ms": 1250
    -- }
    decision        VARCHAR(20) NOT NULL
                    CHECK (decision IN ('auto_approved','escalated','rejected','deferred')),
    overridden_by   UUID,
    override_detail JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE agent_audit_log_default PARTITION OF agent_audit_log DEFAULT;

CREATE INDEX idx_audit_entity ON agent_audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_agent_time ON agent_audit_log(agent_id, created_at);
CREATE INDEX idx_audit_org_time ON agent_audit_log(organisation_id, created_at);
CREATE INDEX idx_audit_reasoning ON agent_audit_log USING GIN (reasoning jsonb_path_ops);
```

### ERP Integration

```sql
CREATE TABLE erp_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    erp_type        VARCHAR(20) NOT NULL
                    CHECK (erp_type IN ('netsuite','quickbooks','xero','sap','oracle_fusion')),
    connection_name VARCHAR(255),
    sync_mode       VARCHAR(10) NOT NULL CHECK (sync_mode IN ('realtime','batch')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    config          JSONB NOT NULL DEFAULT '{}',  -- API endpoints, auth refs, field mappings
    field_mappings  JSONB NOT NULL DEFAULT '{}',  -- maps platform fields to ERP-specific fields
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE erp_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES erp_connection(id),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('inbound','outbound')),
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID,
    erp_ref         VARCHAR(100),
    status          VARCHAR(20) NOT NULL
                    CHECK (status IN ('pending','synced','failed','conflict')),
    -- JSONB for sync payload and error details
    payload         JSONB NOT NULL DEFAULT '{}',   -- the data sent/received
    error_detail    JSONB,                         -- structured error info if failed
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_erp_sync_conn_time ON erp_sync_log(connection_id, synced_at);
CREATE INDEX idx_erp_sync_status ON erp_sync_log(status) WHERE status != 'synced';
```

---

## JSONB Query Examples

```sql
-- Find invoices where AI confidence was below threshold
SELECT id, invoice_number, extracted_fields->>'extraction_model' AS model,
       (extracted_fields->'confidence_scores'->>'total')::numeric AS confidence
FROM invoice
WHERE (extracted_fields->'confidence_scores'->>'total')::numeric < 0.90
  AND status = 'extracted';

-- Find all audit entries citing a specific policy
SELECT id, agent_id, action, decision, created_at
FROM agent_audit_log
WHERE reasoning @> '{"policy_citations": [{"code": "pol-ap-001"}]}';

-- Query collections where payment probability is low
SELECT cq.id, cq.priority_score, cq.days_overdue,
       (cq.scoring_detail->>'payment_probability')::numeric AS prob
FROM collection_queue cq
WHERE (cq.scoring_detail->>'payment_probability')::numeric < 0.40
  AND cq.status = 'active'
ORDER BY cq.priority_score DESC;

-- Find unreconciled bank transactions where AI had multiple close candidates
SELECT id, amount, description,
       match_detail->'candidate_scores' AS candidates
FROM bank_transaction
WHERE NOT reconciled
  AND jsonb_array_length(match_detail->'candidate_scores') > 1;
```

---

## Migration Path

This schema works well as an evolution from the pure normalised model (Suggestion 1). Start with typed columns for all core data. As the AI layer matures and agent output schemas stabilise, some JSONB columns could be "promoted" to typed columns if query patterns justify it. Conversely, if a new ERP integration requires fields not yet modelled, they go into the JSONB `erp_refs` or `custom_fields` columns immediately with no migration required. The GIN indexes on JSONB columns should be reviewed periodically -- remove indexes on paths that are never queried and add indexes on paths that appear in frequent WHERE clauses.
