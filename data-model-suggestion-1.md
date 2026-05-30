# Data Model Suggestion 1: Normalized Relational Model (PostgreSQL)

## Approach

A traditional third-normal-form (3NF+) relational schema in PostgreSQL. Every entity has its own table with proper foreign keys, check constraints, and indexes. This model prioritises data integrity, referential consistency, and query predictability -- all critical properties for a finance operations platform where a misplaced decimal or orphaned record can cause real monetary loss.

## Why This Suits Autonomous Finance Operations

Finance is inherently relational: invoices reference suppliers, purchase orders, and GL accounts; payments reference invoices and bank accounts; reconciliation ties bank statement lines to journal entries. A normalised schema enforces these relationships at the database level, preventing the class of bugs where an agent posts a payment against a non-existent invoice or codes an expense to a deleted cost centre. PostgreSQL's ACID guarantees, row-level security, and mature tooling ecosystem make it the default choice for regulated financial workloads.

## Trade-offs

- **Strengths:** Rock-solid referential integrity, well-understood by finance/ERP developers, excellent tooling for migrations and reporting, straightforward SOX audit queries.
- **Weaknesses:** Schema evolution requires migrations (ALTER TABLE) which can be slow on large tables. Deeply nested or variable-structure data (e.g. vendor-specific invoice metadata) requires extra join tables or nullable columns. High-write audit logging can become a bottleneck without partitioning.
- **Scalability:** Vertical scaling handles most mid-market workloads. For enterprise scale, use table partitioning (by date for transactions, by tenant for multi-tenancy), read replicas for reporting, and connection pooling (PgBouncer).

---

## Schema Definition

### Organisation and Tenant

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_entity_id VARCHAR(50),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE fiscal_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    period_name     VARCHAR(20) NOT NULL,        -- e.g. '2026-Q2', '2026-05'
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','closing','closed','locked')),
    closed_at       TIMESTAMPTZ,
    closed_by       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, period_name)
);
CREATE INDEX idx_fiscal_period_org_status ON fiscal_period(organisation_id, status);
```

### Counterparties (Suppliers and Customers)

```sql
CREATE TABLE counterparty (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    type            VARCHAR(10) NOT NULL CHECK (type IN ('supplier','customer','both')),
    legal_name      VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    tax_id          VARCHAR(50),
    payment_terms   INTEGER DEFAULT 30,          -- net days
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    risk_score      NUMERIC(5,2),
    sanction_status VARCHAR(20) DEFAULT 'clear'
                    CHECK (sanction_status IN ('clear','flagged','blocked')),
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','inactive','blocked')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_counterparty_org_type ON counterparty(organisation_id, type);
CREATE INDEX idx_counterparty_tax_id ON counterparty(tax_id);

CREATE TABLE counterparty_bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    counterparty_id UUID NOT NULL REFERENCES counterparty(id),
    bank_name       VARCHAR(255),
    account_number  VARCHAR(50),
    routing_number  VARCHAR(50),
    iban            VARCHAR(34),
    swift_bic       VARCHAR(11),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_cp_bank_counterparty ON counterparty_bank_account(counterparty_id);
```

### Chart of Accounts and GL

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, account_code)
);
CREATE INDEX idx_gl_account_org_type ON gl_account(organisation_id, account_type);

CREATE TABLE cost_centre (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    code            VARCHAR(20) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, po_number)
);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id),
    line_number     INTEGER NOT NULL,
    description     TEXT NOT NULL,
    quantity        NUMERIC(18,4) NOT NULL,
    unit_price      NUMERIC(18,4) NOT NULL,
    gl_account_id   UUID REFERENCES gl_account(id),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    received_qty    NUMERIC(18,4) NOT NULL DEFAULT 0,
    UNIQUE (purchase_order_id, line_number)
);
```

### Invoices (AP and AR)

```sql
CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    direction       VARCHAR(2) NOT NULL CHECK (direction IN ('ap','ar')),
    invoice_number  VARCHAR(100) NOT NULL,
    counterparty_id UUID NOT NULL REFERENCES counterparty(id),
    purchase_order_id UUID REFERENCES purchase_order(id),
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
    extraction_confidence NUMERIC(5,4),          -- 0.0000 to 1.0000
    match_score     NUMERIC(5,4),
    gl_coding_auto  BOOLEAN DEFAULT false,
    fiscal_period_id UUID REFERENCES fiscal_period(id),
    source_document_url TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_invoice_org_direction ON invoice(organisation_id, direction);
CREATE INDEX idx_invoice_status ON invoice(organisation_id, status);
CREATE INDEX idx_invoice_counterparty ON invoice(counterparty_id);
CREATE INDEX idx_invoice_due_date ON invoice(due_date);

CREATE TABLE invoice_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    line_number     INTEGER NOT NULL,
    description     TEXT,
    quantity        NUMERIC(18,4),
    unit_price      NUMERIC(18,4),
    amount          NUMERIC(18,4) NOT NULL,
    gl_account_id   UUID REFERENCES gl_account(id),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    po_line_id      UUID REFERENCES purchase_order_line(id),
    coding_confidence NUMERIC(5,4),
    UNIQUE (invoice_id, line_number)
);
```

### Three-Way Matching

```sql
CREATE TABLE match_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    po_line_id      UUID REFERENCES purchase_order_line(id),
    invoice_line_id UUID REFERENCES invoice_line(id),
    match_type      VARCHAR(20) NOT NULL
                    CHECK (match_type IN ('po_match','receipt_match','price_match','quantity_match')),
    status          VARCHAR(20) NOT NULL
                    CHECK (status IN ('matched','tolerance','exception')),
    expected_value  NUMERIC(18,4),
    actual_value    NUMERIC(18,4),
    variance_pct    NUMERIC(8,4),
    resolved_by     UUID,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_match_result_invoice ON match_result(invoice_id);
```

### Approval Workflow

```sql
CREATE TABLE approval_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            VARCHAR(255) NOT NULL,
    entity_type     VARCHAR(20) NOT NULL CHECK (entity_type IN ('invoice','payment','journal')),
    min_amount      NUMERIC(18,4),
    max_amount      NUMERIC(18,4),
    required_role   VARCHAR(50),
    auto_approve    BOOLEAN NOT NULL DEFAULT false,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    version         INTEGER NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES approval_policy(id),
    entity_type     VARCHAR(20) NOT NULL,
    entity_id       UUID NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','approved','rejected','escalated','expired')),
    requested_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    decided_at      TIMESTAMPTZ,
    decided_by      UUID,
    decision_note   TEXT
);
CREATE INDEX idx_approval_entity ON approval_request(entity_type, entity_id);
CREATE INDEX idx_approval_status ON approval_request(status);
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
    sanction_screened BOOLEAN NOT NULL DEFAULT false,
    erp_reference   VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    cleared_at      TIMESTAMPTZ
);
CREATE INDEX idx_payment_invoice ON payment(invoice_id);
CREATE INDEX idx_payment_status ON payment(organisation_id, status);
CREATE INDEX idx_payment_dup_hash ON payment(duplicate_check_hash);
```

### Bank Accounts and Reconciliation

```sql
CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    bank_name       VARCHAR(255) NOT NULL,
    account_number  VARCHAR(50),
    iban            VARCHAR(34),
    currency        CHAR(3) NOT NULL,
    current_balance NUMERIC(18,4),
    balance_as_of   TIMESTAMPTZ,
    gl_account_id   UUID REFERENCES gl_account(id),
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
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    raw_file_url    TEXT
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
    match_confidence NUMERIC(5,4),
    reconciled_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_bank_txn_account_date ON bank_transaction(bank_account_id, transaction_date);
CREATE INDEX idx_bank_txn_unreconciled ON bank_transaction(bank_account_id) WHERE NOT reconciled;
```

### Journal Entries (GL Postings)

```sql
CREATE TABLE journal_entry (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    entry_number    VARCHAR(50) NOT NULL,
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    entry_type      VARCHAR(20) NOT NULL
                    CHECK (entry_type IN ('standard','accrual','reversal','adjustment','closing')),
    description     TEXT,
    source_type     VARCHAR(20),                 -- 'invoice','payment','agent','manual'
    source_id       UUID,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft','pending_approval','posted','reversed')),
    posted_at       TIMESTAMPTZ,
    posted_by       UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, entry_number)
);

CREATE TABLE journal_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_entry_id UUID NOT NULL REFERENCES journal_entry(id),
    gl_account_id   UUID NOT NULL REFERENCES gl_account(id),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    debit_amount    NUMERIC(18,4) NOT NULL DEFAULT 0,
    credit_amount   NUMERIC(18,4) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL,
    fx_rate         NUMERIC(18,8) DEFAULT 1.0,
    base_debit      NUMERIC(18,4) NOT NULL DEFAULT 0,
    base_credit     NUMERIC(18,4) NOT NULL DEFAULT 0,
    description     TEXT,
    CHECK (debit_amount >= 0 AND credit_amount >= 0),
    CHECK (NOT (debit_amount > 0 AND credit_amount > 0))
);
CREATE INDEX idx_journal_line_account ON journal_line(gl_account_id);
```

### Cash Flow Forecasting

```sql
CREATE TABLE cash_forecast (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    forecast_date   DATE NOT NULL,
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    scenario        VARCHAR(10) NOT NULL DEFAULT 'base'
                    CHECK (scenario IN ('best','base','worst')),
    horizon_weeks   INTEGER NOT NULL DEFAULT 13,
    model_version   VARCHAR(20)
);

CREATE TABLE cash_forecast_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    forecast_id     UUID NOT NULL REFERENCES cash_forecast(id),
    week_start      DATE NOT NULL,
    category        VARCHAR(30) NOT NULL,        -- 'ap_payments','ar_receipts','payroll','other_inflow','other_outflow'
    currency        CHAR(3) NOT NULL,
    projected_amount NUMERIC(18,4) NOT NULL,
    confidence      NUMERIC(5,4)
);
CREATE INDEX idx_forecast_line_forecast ON cash_forecast_line(forecast_id);
```

### Collections and Dunning (AR)

```sql
CREATE TABLE collection_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    customer_id     UUID NOT NULL REFERENCES counterparty(id),
    priority_score  NUMERIC(8,4),                -- ML-generated payment probability
    days_overdue    INTEGER NOT NULL DEFAULT 0,
    dunning_level   INTEGER NOT NULL DEFAULT 0,
    next_action_date DATE,
    status          VARCHAR(20) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active','promised','disputed','escalated','resolved')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_collection_priority ON collection_queue(organisation_id, priority_score DESC);

CREATE TABLE dunning_action (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id   UUID NOT NULL REFERENCES collection_queue(id),
    action_type     VARCHAR(20) NOT NULL
                    CHECK (action_type IN ('email','call','letter','portal_notice','escalation')),
    template_id     VARCHAR(50),
    sent_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    response        TEXT,
    responded_at    TIMESTAMPTZ
);

CREATE TABLE dispute (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    counterparty_id UUID NOT NULL REFERENCES counterparty(id),
    reason_code     VARCHAR(50) NOT NULL,
    disputed_amount NUMERIC(18,4) NOT NULL,
    description     TEXT,
    status          VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','investigating','resolved','written_off')),
    resolution_note TEXT,
    root_cause      VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ
);
```

### Month-End Close

```sql
CREATE TABLE close_checklist (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    fiscal_period_id UUID NOT NULL REFERENCES fiscal_period(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'not_started'
                    CHECK (status IN ('not_started','in_progress','completed','certified')),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    certified_by    UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE close_task (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    checklist_id    UUID NOT NULL REFERENCES close_checklist(id),
    task_order      INTEGER NOT NULL,
    name            VARCHAR(255) NOT NULL,
    task_type       VARCHAR(30) NOT NULL
                    CHECK (task_type IN ('accrual','reversal','reconciliation','review','sign_off')),
    assigned_agent  VARCHAR(50),                 -- 'ai_agent' or user role
    status          VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','running','completed','failed','skipped')),
    journal_entry_id UUID REFERENCES journal_entry(id),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    notes           TEXT
);
```

### Agent Governance and Audit Trail

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
    parameters      TEXT,                        -- serialised JSON of thresholds and rules
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, policy_code, version)
);

CREATE TABLE agent_audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    agent_id        VARCHAR(100) NOT NULL,
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    reasoning_chain TEXT NOT NULL,               -- human-readable AI reasoning
    policy_citations TEXT,                       -- which policies were evaluated
    input_summary   TEXT,
    output_summary  TEXT,
    confidence      NUMERIC(5,4),
    decision        VARCHAR(20) NOT NULL
                    CHECK (decision IN ('auto_approved','escalated','rejected','deferred')),
    overridden_by   UUID,
    override_reason TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions for the audit log
CREATE TABLE agent_audit_log_2026_01 PARTITION OF agent_audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... additional partitions created by automation

CREATE INDEX idx_audit_log_entity ON agent_audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_agent ON agent_audit_log(agent_id, created_at);
CREATE INDEX idx_audit_log_org_date ON agent_audit_log(organisation_id, created_at);
```

### ERP Sync Tracking

```sql
CREATE TABLE erp_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    erp_type        VARCHAR(20) NOT NULL
                    CHECK (erp_type IN ('netsuite','quickbooks','xero','sap','oracle_fusion')),
    connection_name VARCHAR(255),
    sync_mode       VARCHAR(10) NOT NULL CHECK (sync_mode IN ('realtime','batch')),
    is_active       BOOLEAN NOT NULL DEFAULT true,
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
    error_message   TEXT,
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_erp_sync_connection ON erp_sync_log(connection_id, synced_at);
```

---

## Migration Path

This schema can be introduced incrementally: start with organisation, counterparty, gl_account, and invoice tables for MVP (AP automation). Add payment, bank reconciliation, and journal entry tables for v1.0. Layer on collections, cash forecasting, and close management as the platform expands. The agent_audit_log table should be present from day one since auditability is a core differentiator. Table partitioning on audit_log and bank_transaction by date keeps query performance stable as data grows.
