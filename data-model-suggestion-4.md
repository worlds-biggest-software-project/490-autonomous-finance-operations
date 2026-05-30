# Data Model Suggestion 4: Time-Series + Graph Hybrid (TimescaleDB + Apache AGE)

## Approach

A dual-paradigm architecture pairing **TimescaleDB** (PostgreSQL extension for time-series data) for high-volume transactional and financial metrics with **Apache AGE** (a PostgreSQL graph extension) for modelling approval chains, counterparty relationships, agent governance flows, and fraud/anomaly detection through graph traversal. Both run as extensions on a single PostgreSQL instance, avoiding operational overhead of separate database engines.

## Why This Suits Autonomous Finance Operations

Finance operations is fundamentally a domain of two data shapes that relational models handle awkwardly:

1. **Time-series data is everywhere.** Bank transactions, cash balances, FX rates, agent audit events, payment clearing timestamps, AR ageing snapshots, and 13-week cash forecasts are all time-indexed. TimescaleDB's hypertables automatically partition by time, compress old data 10-20x, and provide native continuous aggregates -- enabling queries like "rolling 7-day average AP throughput" or "cash position at every hour for the last quarter" without manual materialized views.

2. **Relationships and flows are first-class concerns.** Approval chains, segregation of duties (SOD), supplier-invoice-PO-payment-bank transaction lineage, and agent decision graphs are naturally modelled as graph traversals. "Can user X approve this invoice given that user Y already created the PO?" is a graph query. "Show me every entity touched by payment P from invoice to bank clearing" is a path traversal. Relational JOINs can answer these questions but become unwieldy as the chain length grows.

3. **AI agent observability benefits from both paradigms.** Agent actions over time (time-series) reveal performance trends, confidence degradation, and anomaly spikes. Agent decision chains (graph) reveal reasoning dependencies, policy citation networks, and override patterns.

## Trade-offs

- **Strengths:** Native time-series compression and partitioning for financial transaction history (90%+ storage reduction on historical data). Sub-millisecond graph traversals for approval chains and lineage queries. Both run in PostgreSQL -- single backup, single connection pool, single operational model. Continuous aggregates eliminate the need for periodic batch rollups of financial metrics.
- **Weaknesses:** Two query paradigms increase cognitive load for developers. Apache AGE is less mature than Neo4j for complex graph analytics. TimescaleDB compression makes individual record updates expensive (requires decompression). The team needs expertise in both Cypher (graph queries) and SQL.
- **Scalability:** TimescaleDB supports multi-node for horizontal scaling of time-series data. Graph data typically stays smaller (entities + relationships vs. billions of metric points) and scales vertically. For extreme graph workloads, Apache AGE could be replaced with Neo4j at the cost of operational complexity.

---

## TimescaleDB Schema: Time-Series Financial Data

### Core Setup

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS age;

-- Load AGE
LOAD 'age';
SET search_path = ag_catalog, "$user", public;
```

### Reference Tables (Standard PostgreSQL)

These are low-cardinality, slowly-changing reference entities that stay in regular PostgreSQL tables.

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    legal_entity_id VARCHAR(50),
    base_currency   CHAR(3) NOT NULL DEFAULT 'USD',
    timezone        VARCHAR(50) NOT NULL DEFAULT 'UTC',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE counterparty (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    type            VARCHAR(10) NOT NULL CHECK (type IN ('supplier','customer','both')),
    legal_name      VARCHAR(255) NOT NULL,
    tax_id          VARCHAR(50),
    payment_terms   INTEGER DEFAULT 30,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    risk_score      NUMERIC(5,2),
    sanction_status VARCHAR(20) DEFAULT 'clear',
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    bank_accounts   JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE gl_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    account_code    VARCHAR(20) NOT NULL,
    account_name    VARCHAR(255) NOT NULL,
    account_type    VARCHAR(20) NOT NULL
                    CHECK (account_type IN ('asset','liability','equity','revenue','expense')),
    parent_id       UUID REFERENCES gl_account(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (organisation_id, account_code)
);

CREATE TABLE bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    bank_name       VARCHAR(255) NOT NULL,
    account_number  VARCHAR(50),
    iban            VARCHAR(34),
    currency        CHAR(3) NOT NULL,
    gl_account_id   UUID REFERENCES gl_account(id),
    is_active       BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE governance_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    policy_code     VARCHAR(50) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    policy_type     VARCHAR(30) NOT NULL,
    parameters      JSONB NOT NULL DEFAULT '{}',
    version         INTEGER NOT NULL DEFAULT 1,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    effective_from  DATE NOT NULL,
    UNIQUE (organisation_id, policy_code, version)
);

CREATE TABLE fiscal_period (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    period_name     VARCHAR(20) NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'open',
    UNIQUE (organisation_id, period_name)
);
```

### Hypertables: Time-Series Transaction Data

```sql
-- Invoice events (every state change is a time-series point)
CREATE TABLE invoice_event (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    invoice_id      UUID NOT NULL,
    direction       VARCHAR(2) NOT NULL,           -- 'ap' or 'ar'
    invoice_number  VARCHAR(100) NOT NULL,
    counterparty_id UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,           -- 'received','extracted','matched','approved','paid',...
    status_before   VARCHAR(30),
    status_after    VARCHAR(30) NOT NULL,
    amount          NUMERIC(18,4),
    currency        CHAR(3),
    agent_id        VARCHAR(100),
    confidence      NUMERIC(5,4),
    detail          JSONB NOT NULL DEFAULT '{}'     -- event-specific payload
);
SELECT create_hypertable('invoice_event', 'time');
CREATE INDEX idx_inv_event_org ON invoice_event(organisation_id, time DESC);
CREATE INDEX idx_inv_event_invoice ON invoice_event(invoice_id, time DESC);
CREATE INDEX idx_inv_event_type ON invoice_event(event_type, time DESC);

-- Payment events
CREATE TABLE payment_event (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    payment_id      UUID NOT NULL,
    invoice_id      UUID NOT NULL,
    counterparty_id UUID NOT NULL,
    event_type      VARCHAR(50) NOT NULL,           -- 'scheduled','submitted','cleared','failed','reversed'
    amount          NUMERIC(18,4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    payment_method  VARCHAR(20),
    batch_id        UUID,
    detail          JSONB NOT NULL DEFAULT '{}'
);
SELECT create_hypertable('payment_event', 'time');
CREATE INDEX idx_pay_event_org ON payment_event(organisation_id, time DESC);
CREATE INDEX idx_pay_event_payment ON payment_event(payment_id, time DESC);

-- Bank transactions (high-volume time-series from statement imports)
CREATE TABLE bank_transaction (
    time            TIMESTAMPTZ NOT NULL,           -- transaction_date as timestamp
    bank_account_id UUID NOT NULL,
    organisation_id UUID NOT NULL,
    statement_id    UUID,
    amount          NUMERIC(18,4) NOT NULL,
    direction       VARCHAR(6) NOT NULL,            -- 'debit' or 'credit'
    reference       TEXT,
    description     TEXT,
    bank_ref        VARCHAR(100),
    reconciled      BOOLEAN NOT NULL DEFAULT false,
    matched_payment_id UUID,
    match_confidence NUMERIC(5,4),
    match_detail    JSONB NOT NULL DEFAULT '{}'
);
SELECT create_hypertable('bank_transaction', 'time');
CREATE INDEX idx_bank_txn_account ON bank_transaction(bank_account_id, time DESC);
CREATE INDEX idx_bank_txn_unreconciled ON bank_transaction(bank_account_id, time DESC)
    WHERE NOT reconciled;

-- Cash position snapshots (captured daily or hourly)
CREATE TABLE cash_position (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    bank_account_id UUID NOT NULL,
    currency        CHAR(3) NOT NULL,
    balance         NUMERIC(18,4) NOT NULL,
    pending_inflows NUMERIC(18,4) DEFAULT 0,
    pending_outflows NUMERIC(18,4) DEFAULT 0,
    projected_balance NUMERIC(18,4)
);
SELECT create_hypertable('cash_position', 'time');
CREATE INDEX idx_cash_pos_org ON cash_position(organisation_id, time DESC);

-- FX rate feed (time-series of exchange rates)
CREATE TABLE fx_rate (
    time            TIMESTAMPTZ NOT NULL,
    base_currency   CHAR(3) NOT NULL,
    quote_currency  CHAR(3) NOT NULL,
    rate            NUMERIC(18,8) NOT NULL,
    source          VARCHAR(20) NOT NULL DEFAULT 'ecb'
);
SELECT create_hypertable('fx_rate', 'time');
CREATE INDEX idx_fx_pair ON fx_rate(base_currency, quote_currency, time DESC);

-- Agent audit events (time-series of all AI agent actions)
CREATE TABLE agent_audit_event (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    agent_id        VARCHAR(100) NOT NULL,
    action          VARCHAR(100) NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,
    entity_id       UUID NOT NULL,
    decision        VARCHAR(20) NOT NULL,
    confidence      NUMERIC(5,4),
    reasoning       JSONB NOT NULL DEFAULT '{}',
    policy_citations JSONB NOT NULL DEFAULT '[]',
    duration_ms     INTEGER,
    was_overridden  BOOLEAN DEFAULT false,
    override_by     UUID
);
SELECT create_hypertable('agent_audit_event', 'time');
CREATE INDEX idx_audit_org ON agent_audit_event(organisation_id, time DESC);
CREATE INDEX idx_audit_agent ON agent_audit_event(agent_id, time DESC);
CREATE INDEX idx_audit_entity ON agent_audit_event(entity_type, entity_id, time DESC);

-- Collection scoring history (tracks how ML scores evolve per AR item)
CREATE TABLE collection_score (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    invoice_id      UUID NOT NULL,
    customer_id     UUID NOT NULL,
    payment_probability NUMERIC(5,4) NOT NULL,
    priority_score  NUMERIC(8,4) NOT NULL,
    days_overdue    INTEGER NOT NULL,
    model_version   VARCHAR(20),
    factors         JSONB NOT NULL DEFAULT '{}'
);
SELECT create_hypertable('collection_score', 'time');
CREATE INDEX idx_coll_score_invoice ON collection_score(invoice_id, time DESC);
```

### Continuous Aggregates

```sql
-- Daily AP throughput metrics
CREATE MATERIALIZED VIEW daily_ap_metrics
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    organisation_id,
    event_type,
    COUNT(*) AS event_count,
    SUM(amount) AS total_amount,
    AVG(confidence) AS avg_confidence
FROM invoice_event
WHERE direction = 'ap'
GROUP BY day, organisation_id, event_type;

-- Hourly cash position summary
CREATE MATERIALIZED VIEW hourly_cash_summary
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS hour,
    organisation_id,
    currency,
    last(balance, time) AS latest_balance,
    SUM(pending_inflows) AS total_pending_in,
    SUM(pending_outflows) AS total_pending_out
FROM cash_position
GROUP BY hour, organisation_id, currency;

-- Weekly agent performance metrics
CREATE MATERIALIZED VIEW weekly_agent_performance
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 week', time) AS week,
    organisation_id,
    agent_id,
    COUNT(*) AS total_decisions,
    COUNT(*) FILTER (WHERE decision = 'auto_approved') AS auto_approved,
    COUNT(*) FILTER (WHERE was_overridden) AS overridden,
    AVG(confidence) AS avg_confidence,
    AVG(duration_ms) AS avg_duration_ms
FROM agent_audit_event
GROUP BY week, organisation_id, agent_id;

-- Daily reconciliation match rate
CREATE MATERIALIZED VIEW daily_recon_metrics
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS day,
    bank_account_id,
    COUNT(*) AS total_transactions,
    COUNT(*) FILTER (WHERE reconciled) AS matched,
    AVG(match_confidence) FILTER (WHERE reconciled) AS avg_match_confidence
FROM bank_transaction
GROUP BY day, bank_account_id;
```

### Compression Policies

```sql
-- Compress data older than 30 days for storage efficiency
ALTER TABLE invoice_event SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'organisation_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('invoice_event', INTERVAL '30 days');

ALTER TABLE payment_event SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'organisation_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('payment_event', INTERVAL '30 days');

ALTER TABLE bank_transaction SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'bank_account_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('bank_transaction', INTERVAL '30 days');

ALTER TABLE agent_audit_event SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'organisation_id',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('agent_audit_event', INTERVAL '30 days');

ALTER TABLE fx_rate SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'base_currency, quote_currency',
    timescaledb.compress_orderby = 'time DESC'
);
SELECT add_compression_policy('fx_rate', INTERVAL '7 days');
```

---

## Apache AGE Graph Schema: Relationships and Flows

### Graph Setup

```sql
SELECT create_graph('finance_ops');
```

### Vertex (Node) Types

```cypher
-- Organisations, users, and counterparties
CREATE (:Organisation {id: 'org-uuid', name: 'Acme Corp'})
CREATE (:User {id: 'user-uuid', name: 'Jane Doe', role: 'ap_manager', org_id: 'org-uuid'})
CREATE (:Supplier {id: 'sup-uuid', name: 'Vendor Inc', risk_score: 2.5})
CREATE (:Customer {id: 'cust-uuid', name: 'Client LLC', payment_probability: 0.75})

-- Financial documents and transactions
CREATE (:PurchaseOrder {id: 'po-uuid', po_number: 'PO-2026-042', amount: 15000, status: 'approved'})
CREATE (:Invoice {id: 'inv-uuid', invoice_number: 'INV-2026-0042', amount: 16350, direction: 'ap', status: 'paid'})
CREATE (:Payment {id: 'pay-uuid', amount: 16350, method: 'ach', status: 'cleared'})
CREATE (:BankTransaction {id: 'bt-uuid', amount: -16350, bank_ref: 'REF-123'})
CREATE (:JournalEntry {id: 'je-uuid', entry_number: 'JE-2026-05-001', type: 'standard'})

-- Governance
CREATE (:Policy {id: 'pol-uuid', code: 'ap-auto-approve', version: 3, max_amount: 5000})
CREATE (:Agent {id: 'ap-agent-v3', type: 'ap_automation', model_version: 'v3.1'})

-- GL structure
CREATE (:GLAccount {id: 'gl-uuid', code: '6200', name: 'Rent Expense', type: 'expense'})
CREATE (:CostCentre {id: 'cc-uuid', code: 'ENG', name: 'Engineering'})
```

### Edge (Relationship) Types

```cypher
-- Document lineage: PO -> Invoice -> Payment -> Bank Transaction -> Journal Entry
CREATE (po)-[:REFERENCES_PO]->(inv)
CREATE (inv)-[:PAID_BY]->(pay)
CREATE (pay)-[:CLEARED_AS]->(bt)
CREATE (pay)-[:POSTED_AS]->(je)

-- Counterparty relationships
CREATE (sup)-[:SUBMITTED]->(inv)
CREATE (org)-[:HAS_SUPPLIER]->(sup)
CREATE (org)-[:HAS_CUSTOMER]->(cust)

-- Approval chain (critical for SOD compliance)
CREATE (user1)-[:CREATED]->(po)
CREATE (user2)-[:APPROVED {at: '2026-05-29T10:00:00Z', policy: 'pol-ap-001'}]->(inv)
CREATE (agent)-[:AUTO_APPROVED {confidence: 0.98, reasoning: '...', policy: 'pol-ap-002'}]->(inv)
CREATE (user3)-[:OVERRODE {reason: 'Supplier under review', at: '2026-05-29T11:00:00Z'}]->(inv)

-- Policy governance
CREATE (agent)-[:GOVERNED_BY]->(policy)
CREATE (policy)-[:APPLIES_TO {entity_type: 'invoice', min_amount: 0, max_amount: 5000}]->(org)

-- GL coding
CREATE (inv)-[:CODED_TO {amount: 15000, confidence: 0.94}]->(gl)
CREATE (inv)-[:ALLOCATED_TO {amount: 15000}]->(cc)

-- Dunning / collections chain
CREATE (agent)-[:SENT_DUNNING {level: 2, template: 'dunning-2', at: '2026-05-29'}]->(cust)
CREATE (cust)-[:DISPUTED {amount: 500, reason: 'pricing_error'}]->(inv)
```

### Example Graph Queries

```cypher
-- Full payment lineage: trace from invoice to bank clearing
MATCH path = (inv:Invoice {id: 'inv-uuid'})-[:PAID_BY]->(pay:Payment)-[:CLEARED_AS]->(bt:BankTransaction)
RETURN path;

-- Segregation of duties check: did the same person create PO and approve invoice?
MATCH (u1:User)-[:CREATED]->(po:PurchaseOrder)-[:REFERENCES_PO]->(inv:Invoice)<-[:APPROVED]-(u2:User)
WHERE u1.id = u2.id
RETURN inv.invoice_number AS violation, u1.name AS user_name;

-- Find all invoices auto-approved by a specific agent with overrides
MATCH (a:Agent {id: 'ap-agent-v3'})-[:AUTO_APPROVED]->(inv:Invoice)<-[:OVERRODE]-(u:User)
RETURN inv.invoice_number, inv.amount, u.name AS overrider, u.role;

-- Supplier payment network: which suppliers share bank accounts (fraud detection)
MATCH (s1:Supplier)-[:HAS_BANK_ACCOUNT]->(ba)<-[:HAS_BANK_ACCOUNT]-(s2:Supplier)
WHERE s1.id <> s2.id
RETURN s1.name, s2.name, ba.iban;

-- Policy impact analysis: which invoices were affected by policy version change
MATCH (p:Policy {code: 'ap-auto-approve'})-[:APPLIES_TO]->(org:Organisation),
      (a:Agent)-[:GOVERNED_BY]->(p),
      (a)-[r:AUTO_APPROVED]->(inv:Invoice)
WHERE p.version = 3
RETURN COUNT(inv) AS affected_invoices, SUM(inv.amount) AS total_amount;

-- Collections escalation path
MATCH path = (a:Agent)-[:SENT_DUNNING*1..5]->(c:Customer)
WHERE c.id = 'cust-uuid'
RETURN path ORDER BY length(path) DESC LIMIT 1;
```

---

## Query Patterns Across Both Paradigms

### Time-Series Queries (TimescaleDB)

```sql
-- 13-week rolling cash forecast vs. actuals
SELECT
    time_bucket('1 week', cp.time) AS week,
    cp.currency,
    last(cp.balance, cp.time) AS actual_balance,
    last(cp.projected_balance, cp.time) AS projected
FROM cash_position cp
WHERE cp.organisation_id = 'org-uuid'
  AND cp.time >= now() - INTERVAL '13 weeks'
GROUP BY week, cp.currency
ORDER BY week;

-- Agent confidence trend: is extraction accuracy degrading?
SELECT
    time_bucket('1 day', time) AS day,
    agent_id,
    AVG(confidence) AS avg_confidence,
    COUNT(*) FILTER (WHERE was_overridden) AS overrides
FROM agent_audit_event
WHERE organisation_id = 'org-uuid'
  AND agent_id = 'extraction-agent'
  AND time >= now() - INTERVAL '30 days'
GROUP BY day, agent_id
ORDER BY day;

-- Reconciliation match rate trend
SELECT * FROM daily_recon_metrics
WHERE bank_account_id = 'ba-uuid'
  AND day >= now() - INTERVAL '90 days'
ORDER BY day;

-- FX exposure: latest rates for open multi-currency invoices
SELECT DISTINCT ON (ie.currency)
    ie.currency,
    SUM(ie.amount) OVER (PARTITION BY ie.currency) AS exposure,
    fx.rate AS latest_rate,
    fx.time AS rate_time
FROM invoice_event ie
JOIN LATERAL (
    SELECT rate, time FROM fx_rate
    WHERE base_currency = 'USD' AND quote_currency = ie.currency
    ORDER BY time DESC LIMIT 1
) fx ON true
WHERE ie.organisation_id = 'org-uuid'
  AND ie.status_after NOT IN ('paid','cancelled','written_off')
  AND ie.currency != 'USD';
```

---

## Scalability Considerations

- **Time-series retention:** Use TimescaleDB retention policies to drop raw data older than N years while keeping continuous aggregates indefinitely. Compressed chunks reduce storage by 10-20x.
- **Graph size:** The graph stores entities and relationships, not metrics. A large organisation with 100K invoices/year, 5K suppliers, and 500 users generates a graph with ~500K nodes and ~2M edges -- easily fits in memory on a single node.
- **Hybrid queries:** For queries that span both paradigms (e.g. "show me the payment lineage for invoices where agent confidence dropped below 0.85 this week"), run the time-series filter first, extract entity IDs, then traverse the graph. Application-level orchestration handles this cleanly.
- **Multi-tenancy:** Segment hypertables by `organisation_id` (compress_segmentby). Graph queries include org_id filters at the application layer.

## Migration Path

Start with TimescaleDB hypertables from day one for all event/transaction data -- it is a PostgreSQL extension, so migration from vanilla PostgreSQL is just `CREATE EXTENSION` and `create_hypertable`. Add Apache AGE when approval workflows and lineage queries become a product priority (likely v1.0 when SOD compliance is implemented). The graph can be populated by replaying existing time-series events. If graph query complexity exceeds Apache AGE capabilities, the graph layer can be migrated to Neo4j while keeping TimescaleDB for time-series, using CDC (change data capture) to keep both in sync.
