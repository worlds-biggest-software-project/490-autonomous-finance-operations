# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Approach

An event-sourced architecture with Command Query Responsibility Segregation (CQRS). Every state change in the system is captured as an immutable domain event. Commands mutate state by appending events; read models (projections) are built from the event stream and optimised for specific query patterns. The event store is the single source of truth; all current-state views are derived.

## Why This Suits Autonomous Finance Operations

Autonomous finance operations is one of the strongest natural fits for event sourcing:

1. **Immutable audit trail by design.** SOX and IFRS compliance demand that every action is logged with who, what, when, and why. In event sourcing the audit trail IS the data -- it is not a secondary log that can drift from reality.
2. **Agent reasoning becomes a first-class record.** Each AI agent decision is a domain event containing the full reasoning chain, policy citations, and confidence scores. Reconstructing exactly why an invoice was auto-approved at any point in time is a simple event replay.
3. **Temporal queries are free.** "What was the AR ageing at close of business on May 15?" is answered by replaying events up to that timestamp -- no need for snapshot tables or slowly changing dimensions.
4. **Idempotency for durable workflows.** Temporal.io workflows that span hours or days need idempotent steps. Event-sourced aggregates naturally support this: replaying the same command against the same event stream produces no duplicate side effects.
5. **Independent read scaling.** AP clerks, AR collectors, treasury analysts, and auditors all need different views of the same data. CQRS lets each projection be optimised independently.

## Trade-offs

- **Complexity:** Event sourcing adds conceptual overhead. Developers must think in terms of events and projections rather than CRUD. Schema evolution (adding fields to events) requires careful versioning.
- **Eventual consistency:** Read projections lag behind writes. For most finance workflows this is acceptable (sub-second lag), but real-time balance displays may need special handling.
- **Storage growth:** Every state change is stored forever. A high-volume AP operation could generate millions of events per month. Snapshotting mitigates replay cost.
- **Querying raw events is hard:** Ad-hoc reporting requires well-designed projections. You cannot simply JOIN events like relational tables.

---

## Event Store Schema

The event store itself is a simple append-only table (PostgreSQL, EventStoreDB, or similar).

```sql
-- Core event store (PostgreSQL implementation)
CREATE TABLE event_store (
    global_position   BIGSERIAL PRIMARY KEY,
    stream_id         VARCHAR(255) NOT NULL,          -- e.g. 'Invoice-abc123'
    stream_position   INTEGER NOT NULL,
    event_type        VARCHAR(255) NOT NULL,           -- e.g. 'InvoiceReceived'
    event_data        JSONB NOT NULL,
    metadata          JSONB NOT NULL DEFAULT '{}',     -- correlation_id, causation_id, agent_id
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, stream_position)
);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_stream ON event_store(stream_id);
CREATE INDEX idx_event_created ON event_store(created_at);

-- Snapshot store for aggregate rehydration optimisation
CREATE TABLE event_snapshot (
    stream_id         VARCHAR(255) PRIMARY KEY,
    stream_position   INTEGER NOT NULL,
    snapshot_data     JSONB NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Domain Events

### Invoice Aggregate Events

```jsonc
// InvoiceReceived
{
  "event_type": "InvoiceReceived",
  "stream_id": "Invoice-550e8400-e29b",
  "data": {
    "organisation_id": "org-123",
    "direction": "ap",
    "invoice_number": "INV-2026-0042",
    "counterparty_id": "sup-456",
    "currency": "USD",
    "subtotal": 15000.00,
    "tax_amount": 1350.00,
    "total_amount": 16350.00,
    "due_date": "2026-06-28",
    "source_document_url": "s3://docs/inv-2026-0042.pdf"
  },
  "metadata": {
    "correlation_id": "wf-789",
    "agent_id": "intake-agent-v3",
    "timestamp": "2026-05-29T10:00:00Z"
  }
}

// InvoiceFieldsExtracted
{
  "event_type": "InvoiceFieldsExtracted",
  "data": {
    "lines": [
      {"line": 1, "description": "Consulting services", "amount": 15000.00, "gl_account": "6100", "confidence": 0.94}
    ],
    "extraction_model": "doc-ai-v4.2",
    "overall_confidence": 0.96
  },
  "metadata": {
    "agent_id": "extraction-agent",
    "reasoning": "Matched vendor template with 96% confidence. GL code 6100 predicted from 14 prior invoices from this supplier."
  }
}

// InvoiceMatchCompleted
{
  "event_type": "InvoiceMatchCompleted",
  "data": {
    "match_type": "three_way",
    "po_id": "po-321",
    "match_results": [
      {"check": "po_match", "status": "matched", "variance_pct": 0.0},
      {"check": "receipt_match", "status": "matched", "received_qty": 1, "invoiced_qty": 1},
      {"check": "price_match", "status": "tolerance", "expected": 14800.00, "actual": 15000.00, "variance_pct": 1.35}
    ],
    "overall_status": "matched_with_tolerance"
  }
}

// InvoiceApprovalRequested
{
  "event_type": "InvoiceApprovalRequested",
  "data": {
    "policy_id": "pol-ap-001",
    "policy_version": 3,
    "reason": "Amount exceeds auto-approval threshold of $10,000",
    "requested_approver_role": "ap_manager"
  }
}

// InvoiceApproved
{
  "event_type": "InvoiceApproved",
  "data": {
    "approved_by": "user-jane-doe",
    "approval_type": "manual",
    "note": "Verified with project manager"
  }
}

// InvoiceAutoApproved
{
  "event_type": "InvoiceAutoApproved",
  "data": {
    "policy_id": "pol-ap-002",
    "reasoning_chain": "Amount $450 below auto-approve threshold $5,000. Supplier verified. No duplicate detected. Sanction screen clear.",
    "policy_citations": ["pol-ap-002-v2", "pol-sanction-001-v1", "pol-dup-001-v1"],
    "confidence": 0.98
  }
}
```

### Payment Aggregate Events

```jsonc
// PaymentScheduled
{
  "event_type": "PaymentScheduled",
  "data": {
    "invoice_id": "Invoice-550e8400",
    "counterparty_id": "sup-456",
    "amount": 16350.00,
    "currency": "USD",
    "payment_method": "ach",
    "scheduled_date": "2026-06-01",
    "batch_id": "batch-2026-06-01-001"
  }
}

// PaymentDuplicateDetected
{
  "event_type": "PaymentDuplicateDetected",
  "data": {
    "original_payment_id": "Payment-existing-123",
    "duplicate_hash": "sha256:abc123",
    "action_taken": "blocked",
    "reasoning": "Same supplier, same amount, same invoice number within 7-day window"
  }
}

// PaymentSanctionScreened
{
  "event_type": "PaymentSanctionScreened",
  "data": {
    "screen_provider": "ofac",
    "result": "clear",
    "screened_entities": ["sup-456"],
    "screened_at": "2026-05-29T10:05:00Z"
  }
}

// PaymentSubmitted, PaymentCleared, PaymentFailed, PaymentReversed ...
```

### Bank Reconciliation Events

```jsonc
// BankStatementImported
{
  "event_type": "BankStatementImported",
  "data": {
    "bank_account_id": "ba-001",
    "format": "bai2",
    "statement_date": "2026-05-28",
    "opening_balance": 245000.00,
    "closing_balance": 231450.00,
    "transaction_count": 47
  }
}

// BankTransactionMatched
{
  "event_type": "BankTransactionMatched",
  "data": {
    "bank_transaction_ref": "BT-2026-05-28-014",
    "matched_payment_id": "Payment-abc",
    "match_method": "ai_fuzzy",
    "confidence": 0.92,
    "reasoning": "Amount exact match. Reference partial match (8/12 chars). Date within 2-day window."
  }
}

// BankTransactionUnmatched -- flagged for manual review
```

### Collections and Dunning Events

```jsonc
// CollectionItemPrioritised
{
  "event_type": "CollectionItemPrioritised",
  "data": {
    "invoice_id": "Invoice-ar-789",
    "customer_id": "cust-012",
    "payment_probability": 0.35,
    "days_overdue": 42,
    "priority_score": 87.5,
    "scoring_model": "collections-ml-v2"
  }
}

// DunningEmailSent
{
  "event_type": "DunningEmailSent",
  "data": {
    "collection_id": "col-345",
    "template": "dunning-level-2",
    "recipient": "ap@customer.com",
    "strategy": "adaptive",
    "reasoning": "Customer responded to email (not phone) in 3 of 4 prior cases. Escalating to level 2 after 14 days no response."
  }
}

// DisputeRaised, DisputeResolved, PaymentPromiseRecorded ...
```

### Month-End Close Events

```jsonc
// CloseChecklistInitiated
{
  "event_type": "CloseChecklistInitiated",
  "data": {
    "fiscal_period": "2026-05",
    "task_count": 24,
    "tasks": ["accrue_utilities","accrue_rent","reverse_prior_accruals","reconcile_bank","reconcile_intercompany","post_depreciation","review_ar_ageing","certify"]
  }
}

// AccrualJournalPosted
{
  "event_type": "AccrualJournalPosted",
  "data": {
    "close_task_id": "task-accrue-rent",
    "journal_entry_stream": "Journal-je-2026-05-001",
    "lines": [
      {"gl_account": "6200", "debit": 8500.00, "credit": 0},
      {"gl_account": "2100", "debit": 0, "credit": 8500.00}
    ],
    "reasoning": "Monthly rent accrual based on lease agreement LS-2025-003. Amount unchanged from prior 4 months.",
    "agent_id": "close-agent-v2"
  }
}

// PeriodClosed, PeriodLocked
```

### Governance Policy Events

```jsonc
// GovernancePolicyPublished
{
  "event_type": "GovernancePolicyPublished",
  "data": {
    "policy_code": "ap-auto-approve",
    "version": 3,
    "parameters": {
      "max_amount": 5000,
      "require_po_match": true,
      "require_sanction_screen": true,
      "excluded_categories": ["capex"]
    },
    "effective_from": "2026-06-01",
    "published_by": "user-cfo"
  }
}

// AgentOverridden
{
  "event_type": "AgentOverridden",
  "data": {
    "original_decision": "auto_approved",
    "override_decision": "rejected",
    "overridden_by": "user-controller",
    "reason": "Supplier under review pending updated W-9",
    "entity_type": "invoice",
    "entity_id": "Invoice-550e8400"
  }
}
```

---

## Command Handlers

```text
Commands (write side):
  ReceiveInvoice       -> validates, emits InvoiceReceived
  ExtractInvoiceFields -> runs doc AI, emits InvoiceFieldsExtracted
  MatchInvoice         -> runs 3-way match, emits InvoiceMatchCompleted
  RequestApproval      -> evaluates policies, emits InvoiceApprovalRequested or InvoiceAutoApproved
  ApproveInvoice       -> human approval, emits InvoiceApproved
  SchedulePayment      -> validates, dedup checks, emits PaymentScheduled
  SubmitPayment        -> sends to ERP/bank, emits PaymentSubmitted
  ImportBankStatement  -> parses BAI2/MT940/CAMT053, emits BankStatementImported + N BankTransactionRecorded
  ReconcileTransaction -> AI matching, emits BankTransactionMatched or BankTransactionUnmatched
  PrioritiseCollections -> ML scoring, emits CollectionItemPrioritised for each AR item
  SendDunning          -> adaptive outreach, emits DunningEmailSent
  InitiateClose        -> creates checklist, emits CloseChecklistInitiated
  PostAccrualJournal   -> agent posts journal, emits AccrualJournalPosted
  PublishPolicy        -> CFO publishes governance rule, emits GovernancePolicyPublished
  OverrideAgent        -> human overrides agent decision, emits AgentOverridden
```

---

## Read Projections (Query Side)

Projections are rebuilt from the event stream and stored in PostgreSQL tables optimised for reads.

```sql
-- Projection: Current invoice state (materialised from invoice events)
CREATE TABLE proj_invoice (
    invoice_id          UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    direction           VARCHAR(2) NOT NULL,
    invoice_number      VARCHAR(100) NOT NULL,
    counterparty_id     UUID NOT NULL,
    counterparty_name   VARCHAR(255),           -- denormalised for fast display
    status              VARCHAR(30) NOT NULL,
    currency            CHAR(3) NOT NULL,
    total_amount        NUMERIC(18,4) NOT NULL,
    due_date            DATE,
    extraction_confidence NUMERIC(5,4),
    match_status        VARCHAR(30),
    approval_status     VARCHAR(20),
    approved_by         VARCHAR(100),
    payment_status      VARCHAR(20),
    days_overdue        INTEGER GENERATED ALWAYS AS (GREATEST(0, CURRENT_DATE - due_date)) STORED,
    last_event_at       TIMESTAMPTZ NOT NULL,
    last_event_position BIGINT NOT NULL          -- for projection tracking
);
CREATE INDEX idx_proj_inv_org_status ON proj_invoice(organisation_id, status);
CREATE INDEX idx_proj_inv_overdue ON proj_invoice(organisation_id, days_overdue DESC) WHERE direction = 'ar';

-- Projection: AR ageing summary (rebuilt from payment/invoice events)
CREATE TABLE proj_ar_ageing (
    organisation_id     UUID NOT NULL,
    customer_id         UUID NOT NULL,
    customer_name       VARCHAR(255),
    currency            CHAR(3) NOT NULL,
    current_amount      NUMERIC(18,4) DEFAULT 0,
    days_1_30           NUMERIC(18,4) DEFAULT 0,
    days_31_60          NUMERIC(18,4) DEFAULT 0,
    days_61_90          NUMERIC(18,4) DEFAULT 0,
    days_over_90        NUMERIC(18,4) DEFAULT 0,
    total_outstanding   NUMERIC(18,4) DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organisation_id, customer_id, currency)
);

-- Projection: Bank reconciliation dashboard
CREATE TABLE proj_bank_reconciliation (
    bank_account_id     UUID NOT NULL,
    statement_date      DATE NOT NULL,
    total_transactions  INTEGER NOT NULL DEFAULT 0,
    matched_count       INTEGER NOT NULL DEFAULT 0,
    unmatched_count     INTEGER NOT NULL DEFAULT 0,
    match_rate_pct      NUMERIC(5,2),
    unmatched_amount    NUMERIC(18,4) DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (bank_account_id, statement_date)
);

-- Projection: Cash position (rebuilt from payment and bank events)
CREATE TABLE proj_cash_position (
    organisation_id     UUID NOT NULL,
    bank_account_id     UUID NOT NULL,
    currency            CHAR(3) NOT NULL,
    balance             NUMERIC(18,4) NOT NULL,
    as_of               TIMESTAMPTZ NOT NULL,
    pending_outflows    NUMERIC(18,4) DEFAULT 0,
    pending_inflows     NUMERIC(18,4) DEFAULT 0,
    projected_balance   NUMERIC(18,4),
    PRIMARY KEY (organisation_id, bank_account_id)
);

-- Projection: Close checklist progress
CREATE TABLE proj_close_progress (
    organisation_id     UUID NOT NULL,
    fiscal_period       VARCHAR(20) NOT NULL,
    total_tasks         INTEGER NOT NULL DEFAULT 0,
    completed_tasks     INTEGER NOT NULL DEFAULT 0,
    failed_tasks        INTEGER NOT NULL DEFAULT 0,
    progress_pct        NUMERIC(5,2),
    status              VARCHAR(20) NOT NULL,
    last_updated        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organisation_id, fiscal_period)
);

-- Projection: Agent activity feed (for human oversight console)
CREATE TABLE proj_agent_activity (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    event_type          VARCHAR(255) NOT NULL,
    agent_id            VARCHAR(100),
    entity_type         VARCHAR(50) NOT NULL,
    entity_id           UUID NOT NULL,
    summary             TEXT NOT NULL,
    reasoning_chain     TEXT,
    decision            VARCHAR(20),
    confidence          NUMERIC(5,4),
    policy_citations    TEXT[],
    was_overridden      BOOLEAN DEFAULT false,
    occurred_at         TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_proj_agent_org_time ON proj_agent_activity(organisation_id, occurred_at DESC);
CREATE INDEX idx_proj_agent_entity ON proj_agent_activity(entity_type, entity_id);

-- Projection: Governance policy registry (current active policies)
CREATE TABLE proj_active_policies (
    policy_code         VARCHAR(50) NOT NULL,
    organisation_id     UUID NOT NULL,
    version             INTEGER NOT NULL,
    name                VARCHAR(255) NOT NULL,
    parameters          JSONB NOT NULL,
    effective_from      DATE NOT NULL,
    published_by        VARCHAR(100),
    last_updated        TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organisation_id, policy_code)
);
```

---

## Projection Tracking

```sql
-- Tracks the last processed event position for each projection
CREATE TABLE projection_checkpoint (
    projection_name     VARCHAR(100) PRIMARY KEY,
    last_position       BIGINT NOT NULL DEFAULT 0,
    last_updated        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Each projection consumer reads events from `event_store` where `global_position > last_position`, applies the event to its read model, and advances the checkpoint. This supports exactly-once processing semantics within a single database transaction.

---

## Scalability Considerations

- **Event store partitioning:** Partition `event_store` by `created_at` range for archival and query performance.
- **Snapshotting:** Create aggregate snapshots every N events (e.g. every 100) to avoid replaying long streams. Critical for invoices that go through many state transitions.
- **Projection rebuild:** Any projection can be dropped and rebuilt from scratch by replaying the event store. This is the primary migration path -- add a new projection, replay events, switch traffic.
- **Independent scaling:** Read projections can live in separate databases or read replicas. Write throughput is limited to the event store append rate, which can be partitioned by stream_id.
- **Event versioning:** Use event upcasters to transform old event schemas to new ones during replay. Never mutate stored events.

## Migration Path

Start with a PostgreSQL-backed event store for simplicity. If write throughput exceeds single-node PostgreSQL capacity, migrate to EventStoreDB or Apache Kafka as the event backbone while keeping PostgreSQL for projections. The event schemas (JSON) serve as the integration contract -- projections can be rewritten in any technology without affecting the write side.
