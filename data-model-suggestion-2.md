# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Procurement Automation · Created: 2026-05-12

## Philosophy

This model treats every change in the procurement system as an immutable domain event stored in an append-only event store. The event store is the single source of truth; all current-state views (purchase orders, invoices, supplier records) are materialised projections rebuilt from event replay. This architecture natively satisfies the SOX audit trail requirement because every state transition — from requisition creation through three-way matching to payment — is permanently recorded with full context, actor identity, and timestamp.

Event sourcing is proven in financial systems (banking ledgers, trading platforms, blockchain), regulatory compliance platforms, and increasingly in enterprise procurement. Coupa's "community intelligence" engine processes billions of spend events; GEP SMART's AI classification operates on streams of categorisation events. The pattern is particularly powerful for procurement because the domain naturally consists of discrete business events: "supplier submitted quote," "PO approved by CFO," "goods received at warehouse," "invoice matched to PO line," "payment initiated." Each of these is a first-class event rather than a side-effect field update.

The CQRS (Command Query Responsibility Segregation) companion pattern separates write operations (commands that produce events) from read operations (queries against materialised views). This enables the procurement platform to optimise reads independently — a spend analytics dashboard reads from a denormalized spend cube, while the three-way matching engine reads from a match-optimised projection, and the audit trail reads directly from the raw event store.

**Best for:** Organisations requiring complete, tamper-evident audit trails; AI/ML teams that need full procurement event history for training models; deployments where temporal queries ("what was the PO status on March 15?") and regulatory compliance are primary concerns.

**Trade-offs:**
- (+) Complete, immutable audit trail is inherent — not bolted on
- (+) Temporal queries are trivial: replay events up to any point in time
- (+) AI/ML training on full event history without separate ETL pipeline
- (+) Independent scaling of write (event append) and read (projection query) paths
- (+) Schema evolution is simpler: new event types can be added without altering existing tables
- (-) Higher complexity: developers must think in events, not CRUD
- (-) Eventual consistency between event store and projections requires careful design
- (-) Projections must be rebuilt if the read schema changes (can be slow for large event stores)
- (-) Debugging requires event replay tooling; you cannot simply "UPDATE" a row to fix data
- (-) Storage grows faster than mutable-state models (events are never deleted)
- (-) More infrastructure: requires event bus, projection workers, and snapshot management

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| UBL 2.1 (ISO/IEC 19845) | UBL document XML/JSON stored as event payloads when POs and invoices are transmitted/received via PEPPOL |
| PEPPOL BIS 3.0 | PEPPOL message exchange events include full BIS document payloads; transmission IDs tracked as event metadata |
| EN 16931 | Invoice received events carry EN 16931 compliance validation results as part of the event payload |
| UNSPSC | Spend classification events record original and corrected UNSPSC codes, enabling temporal analysis of classification accuracy |
| ISO 20022 | Payment initiation events carry ISO 20022 pain.001 message structures as payloads |
| OCSF (Open Cybersecurity Schema Framework) | Event envelope structure (actor, action, object, timestamp, context) inspired by OCSF event categories |
| CloudEvents 1.0 (CNCF) | Event metadata envelope follows CloudEvents specification for interoperability with event-driven middleware |
| SOX Section 404 | Immutable event store directly satisfies internal control over financial reporting requirements |
| EDI X12 / EDIFACT | EDI document exchange events store original EDI payloads alongside parsed structured data |

---

## Event Store (Core)

```sql
-- ============================================================
-- EVENT STORE — Single source of truth
-- ============================================================

-- The event store is an append-only table. No UPDATE or DELETE operations
-- are ever performed on this table. All current state is derived from
-- replaying events in sequence.

CREATE TABLE event_store (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,                    -- 'purchase_requisition','purchase_order','invoice','supplier','goods_receipt','sourcing_event','payment'
    stream_id       UUID NOT NULL,                    -- aggregate root ID (e.g., PO ID)
    event_type      TEXT NOT NULL,                    -- e.g. 'PurchaseOrderCreated','LineItemAdded','InvoiceMatched'
    event_version   INTEGER NOT NULL,                 -- monotonically increasing per stream
    payload         JSONB NOT NULL,                   -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',      -- actor, IP, correlation IDs, channel
    -- CloudEvents-inspired envelope fields
    ce_source       TEXT NOT NULL DEFAULT 'procurement-api', -- system that produced the event
    ce_subject      TEXT,                             -- human-readable subject (e.g., 'PO-2026-00142')
    correlation_id  UUID,                             -- links related events across streams
    causation_id    UUID,                             -- the event that caused this event
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Ensure exactly-once event ordering per stream
    UNIQUE(stream_id, event_version)
);

-- Primary query patterns:
-- 1. Replay a single aggregate: WHERE stream_id = $1 ORDER BY event_version
-- 2. Global ordered replay: WHERE tenant_id = $1 ORDER BY created_at, event_id
-- 3. Event type filtering: WHERE event_type = $1 AND tenant_id = $2

CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_tenant_time ON event_store(tenant_id, created_at);
CREATE INDEX idx_event_type ON event_store(event_type, tenant_id, created_at);
CREATE INDEX idx_event_correlation ON event_store(correlation_id);

-- Partition by month for performance at scale
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- ============================================================
-- EVENT SNAPSHOTS — Periodic state checkpoints to avoid full replay
-- ============================================================

CREATE TABLE event_snapshot (
    stream_id       UUID NOT NULL,
    stream_type     TEXT NOT NULL,
    snapshot_version INTEGER NOT NULL,                -- event_version at which snapshot was taken
    state           JSONB NOT NULL,                   -- serialised aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (stream_id, snapshot_version)
);

-- ============================================================
-- OUTBOX — Guarantees at-least-once event publication
-- ============================================================

CREATE TABLE event_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES event_store(event_id),
    destination     TEXT NOT NULL,                    -- 'projections','notifications','webhooks','analytics'
    published       BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    retry_count     SMALLINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_outbox_unpublished ON event_outbox(published, destination) WHERE published = false;
```

---

## Event Type Catalogue

```sql
-- ============================================================
-- EVENT TYPE REGISTRY (documentation table, not enforced via FK)
-- ============================================================

CREATE TABLE event_type_registry (
    event_type      TEXT PRIMARY KEY,
    stream_type     TEXT NOT NULL,
    description     TEXT NOT NULL,
    payload_schema  JSONB NOT NULL,                   -- JSON Schema for the event payload
    version         INTEGER NOT NULL DEFAULT 1,
    deprecated      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Example event type registrations:
-- INSERT INTO event_type_registry VALUES
-- ('PurchaseRequisitionCreated', 'purchase_requisition', 'A new purchase requisition was submitted', '{"type":"object","properties":{"title":{"type":"string"},"requester_id":{"type":"string","format":"uuid"},"lines":{"type":"array"}}}', 1, false, now()),
-- ('PurchaseRequisitionApproved', 'purchase_requisition', 'A requisition was approved by an approver', '{"type":"object","properties":{"approver_id":{"type":"string","format":"uuid"},"step":{"type":"integer"}}}', 1, false, now()),
-- ('PurchaseOrderCreated', 'purchase_order', 'A PO was generated from a requisition or manually', '{"type":"object","properties":{"po_number":{"type":"string"},"supplier_id":{"type":"string","format":"uuid"},"lines":{"type":"array"}}}', 1, false, now()),
-- ('PurchaseOrderSentToSupplier', 'purchase_order', 'PO was transmitted to supplier via email/PEPPOL/EDI', '{"type":"object","properties":{"channel":{"type":"string"},"transmission_id":{"type":"string"}}}', 1, false, now()),
-- ('GoodsReceived', 'goods_receipt', 'Physical goods were received at a location', '{"type":"object","properties":{"po_id":{"type":"string","format":"uuid"},"lines":{"type":"array"}}}', 1, false, now()),
-- ('InvoiceReceived', 'invoice', 'A vendor invoice was received via any channel', '{"type":"object","properties":{"invoice_number":{"type":"string"},"supplier_id":{"type":"string","format":"uuid"},"total":{"type":"number"}}}', 1, false, now()),
-- ('InvoiceAutoMatched', 'invoice', 'AI successfully matched invoice to PO and GRN', '{"type":"object","properties":{"match_score":{"type":"number"},"po_id":{"type":"string","format":"uuid"}}}', 1, false, now()),
-- ('MatchExceptionRaised', 'invoice', 'Three-way matching found a discrepancy outside tolerance', '{"type":"object","properties":{"exception_type":{"type":"string"},"variance":{"type":"number"}}}', 1, false, now()),
-- ('MatchExceptionResolved', 'invoice', 'Exception was resolved by AI or human', '{"type":"object","properties":{"resolved_by":{"type":"string"},"resolution":{"type":"string"}}}', 1, false, now()),
-- ('PaymentInitiated', 'payment', 'Payment was submitted to payment rails', '{"type":"object","properties":{"amount":{"type":"number"},"currency":{"type":"string"}}}', 1, false, now()),
-- ('SpendClassified', 'invoice', 'AI classified a line item against UNSPSC taxonomy', '{"type":"object","properties":{"unspsc_code":{"type":"string"},"confidence":{"type":"number"}}}', 1, false, now());
```

---

## Example Event Payloads

```jsonc
// Event: PurchaseOrderCreated
{
  "event_type": "PurchaseOrderCreated",
  "stream_type": "purchase_order",
  "stream_id": "a1b2c3d4-...",
  "payload": {
    "po_number": "PO-2026-00142",
    "supplier_id": "s1s2s3s4-...",
    "supplier_name": "Ergonomic Solutions GmbH",
    "requisition_id": "r1r2r3r4-...",
    "currency": "EUR",
    "lines": [
      {
        "line_number": 1,
        "description": "Ergonomic office chair, adjustable height",
        "item_id": "i1i2i3i4-...",
        "quantity": 50,
        "unit_price": 159.00,
        "unit_of_measure": "EA",
        "unspsc_code": "56101504",
        "cost_centre_code": "CC-BERLIN-OPS"
      }
    ],
    "subtotal": 7950.00,
    "tax_amount": 1510.50,
    "total_amount": 9460.50,
    "payment_terms_days": 30,
    "expected_delivery_date": "2026-06-15",
    "created_by": "u1u2u3u4-..."
  },
  "metadata": {
    "actor_id": "u1u2u3u4-...",
    "actor_type": "user",
    "ip_address": "192.168.1.42",
    "channel": "web",
    "correlation_id": "corr-001-..."
  }
}

// Event: InvoiceAutoMatched
{
  "event_type": "InvoiceAutoMatched",
  "stream_type": "invoice",
  "stream_id": "inv-uuid-...",
  "payload": {
    "invoice_number": "INV-2026-8847",
    "purchase_order_id": "a1b2c3d4-...",
    "match_results": [
      {
        "invoice_line": 1,
        "po_line": 1,
        "grn_line": 1,
        "invoiced_qty": 50,
        "received_qty": 50,
        "ordered_qty": 50,
        "invoiced_price": 159.00,
        "po_price": 159.00,
        "price_variance": 0.00,
        "quantity_variance": 0,
        "match_status": "matched"
      }
    ],
    "overall_match_score": 1.0,
    "auto_approved": true,
    "tolerance_rule_applied": "default-2pct"
  },
  "metadata": {
    "actor_type": "system",
    "ai_model_version": "match-v3.2",
    "processing_time_ms": 142
  }
}

// Event: SpendClassified
{
  "event_type": "SpendClassified",
  "stream_type": "invoice",
  "stream_id": "inv-uuid-...",
  "payload": {
    "line_number": 1,
    "raw_description": "Ergonomic office chair, adjustable height",
    "classified_unspsc": "56101504",
    "unspsc_path": "Furniture > Office Furniture > Seating > Task and operator seating",
    "confidence": 0.9847,
    "alternative_codes": [
      {"code": "56101505", "confidence": 0.0098, "name": "Executive seating"},
      {"code": "56101503", "confidence": 0.0031, "name": "Side chairs"}
    ],
    "model_version": "classify-v2.1"
  },
  "metadata": {
    "actor_type": "ai",
    "ai_model_version": "classify-v2.1"
  }
}
```

---

## Materialised Read Projections (CQRS Query Side)

```sql
-- ============================================================
-- PROJECTION: Current Purchase Order State
-- ============================================================
-- This table is rebuilt from events. It is NOT the source of truth.
-- Mark it clearly so developers never write to it directly.

CREATE TABLE proj_purchase_order (
    id              UUID PRIMARY KEY,                 -- same as stream_id
    tenant_id       UUID NOT NULL,
    po_number       TEXT NOT NULL,
    supplier_id     UUID NOT NULL,
    supplier_name   TEXT NOT NULL,
    requisition_id  UUID,
    status          TEXT NOT NULL,
    order_date      DATE NOT NULL,
    expected_delivery_date DATE,
    currency        CHAR(3) NOT NULL,
    subtotal        NUMERIC(18,2) NOT NULL,
    tax_amount      NUMERIC(18,2) NOT NULL,
    total_amount    NUMERIC(18,2) NOT NULL,
    cost_centre_code TEXT,
    created_by      UUID NOT NULL,
    sent_at         TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,              -- track projection freshness
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);

CREATE INDEX idx_proj_po_tenant ON proj_purchase_order(tenant_id, status);
CREATE INDEX idx_proj_po_supplier ON proj_purchase_order(supplier_id);

CREATE TABLE proj_purchase_order_line (
    id              UUID PRIMARY KEY,
    purchase_order_id UUID NOT NULL REFERENCES proj_purchase_order(id),
    line_number     SMALLINT NOT NULL,
    description     TEXT NOT NULL,
    quantity_ordered NUMERIC(18,4) NOT NULL,
    quantity_received NUMERIC(18,4) NOT NULL DEFAULT 0,
    quantity_invoiced NUMERIC(18,4) NOT NULL DEFAULT 0,
    unit_price      NUMERIC(18,4) NOT NULL,
    line_total      NUMERIC(18,2) NOT NULL,
    unspsc_code     CHAR(8),
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ============================================================
-- PROJECTION: Current Invoice State with Match Status
-- ============================================================

CREATE TABLE proj_invoice (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    invoice_number  TEXT NOT NULL,
    supplier_id     UUID NOT NULL,
    supplier_name   TEXT NOT NULL,
    purchase_order_id UUID,
    po_number       TEXT,
    status          TEXT NOT NULL,
    match_status    TEXT NOT NULL,
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency        CHAR(3) NOT NULL,
    total_amount    NUMERIC(18,2) NOT NULL,
    match_score     NUMERIC(5,4),
    exception_count INTEGER NOT NULL DEFAULT 0,
    payment_status  TEXT NOT NULL DEFAULT 'unpaid',
    paid_amount     NUMERIC(18,2) NOT NULL DEFAULT 0,
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_inv_tenant ON proj_invoice(tenant_id, status);
CREATE INDEX idx_proj_inv_match ON proj_invoice(tenant_id, match_status);
CREATE INDEX idx_proj_inv_due ON proj_invoice(tenant_id, due_date);

-- ============================================================
-- PROJECTION: Supplier Master (current state)
-- ============================================================

CREATE TABLE proj_supplier (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    peppol_id       TEXT,
    status          TEXT NOT NULL,
    risk_score      NUMERIC(5,2),
    risk_updated_at TIMESTAMPTZ,
    total_pos       INTEGER NOT NULL DEFAULT 0,
    total_spend     NUMERIC(18,2) NOT NULL DEFAULT 0,
    on_time_delivery_pct NUMERIC(5,2),
    last_event_version INTEGER NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_proj_supplier_tenant ON proj_supplier(tenant_id, status);

-- ============================================================
-- PROJECTION: Spend Analytics Cube (denormalized for dashboards)
-- ============================================================

CREATE TABLE proj_spend_cube (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    fiscal_year     SMALLINT NOT NULL,
    fiscal_quarter  SMALLINT NOT NULL,
    fiscal_month    SMALLINT NOT NULL,
    supplier_id     UUID NOT NULL,
    supplier_name   TEXT NOT NULL,
    cost_centre_code TEXT,
    gl_account_code TEXT,
    unspsc_segment  CHAR(2),
    unspsc_family   CHAR(4),
    unspsc_class    CHAR(6),
    unspsc_commodity CHAR(8),
    currency        CHAR(3) NOT NULL,
    po_count        INTEGER NOT NULL DEFAULT 0,
    invoice_count   INTEGER NOT NULL DEFAULT 0,
    total_ordered   NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_invoiced  NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_paid      NUMERIC(18,2) NOT NULL DEFAULT 0,
    avg_match_score NUMERIC(5,4),
    exception_count INTEGER NOT NULL DEFAULT 0,
    auto_resolved_count INTEGER NOT NULL DEFAULT 0,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_spend_cube_tenant_period ON proj_spend_cube(tenant_id, fiscal_year, fiscal_month);
CREATE INDEX idx_spend_cube_supplier ON proj_spend_cube(tenant_id, supplier_id);
CREATE INDEX idx_spend_cube_unspsc ON proj_spend_cube(tenant_id, unspsc_segment);

-- ============================================================
-- PROJECTION: Approval Queue (active approvals for dashboard)
-- ============================================================

CREATE TABLE proj_approval_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    entity_type     TEXT NOT NULL,                    -- 'purchase_requisition','purchase_order','invoice'
    entity_id       UUID NOT NULL,
    entity_ref      TEXT NOT NULL,                    -- e.g., 'PR-2026-00042'
    title           TEXT NOT NULL,
    total_amount    NUMERIC(18,2),
    currency        CHAR(3),
    requester_name  TEXT NOT NULL,
    approver_id     UUID NOT NULL,
    step_order      SMALLINT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    submitted_at    TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_approval_queue_approver ON proj_approval_queue(approver_id, status);

-- ============================================================
-- PROJECTION: Match Exception Queue
-- ============================================================

CREATE TABLE proj_match_exception_queue (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    invoice_id      UUID NOT NULL,
    invoice_number  TEXT NOT NULL,
    supplier_name   TEXT NOT NULL,
    exception_type  TEXT NOT NULL,
    severity        TEXT NOT NULL,
    description     TEXT NOT NULL,
    ai_resolution   TEXT,
    ai_confidence   NUMERIC(5,4),
    raised_at       TIMESTAMPTZ NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_exception_queue_tenant ON proj_match_exception_queue(tenant_id, severity);
```

---

## Tenant & Identity (Command Side)

```sql
-- ============================================================
-- TENANT & USER (Mutable reference data — not event-sourced)
-- These tables are mutable because they represent system configuration,
-- not business domain state. Only domain aggregates are event-sourced.
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    peppol_id       TEXT,
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- ============================================================
-- REFERENCE DATA (UNSPSC, currencies, etc.) — also mutable
-- ============================================================

CREATE TABLE unspsc_taxonomy (
    code            CHAR(8) PRIMARY KEY,
    segment_code    CHAR(2) NOT NULL,
    segment_name    TEXT NOT NULL,
    family_code     CHAR(4) NOT NULL,
    family_name     TEXT NOT NULL,
    class_code      CHAR(6) NOT NULL,
    class_name      TEXT NOT NULL,
    commodity_name  TEXT NOT NULL
);

-- Flattened taxonomy for simpler lookups; trades normalization for query speed

CREATE INDEX idx_unspsc_segment ON unspsc_taxonomy(segment_code);
CREATE INDEX idx_unspsc_family ON unspsc_taxonomy(family_code);

-- ============================================================
-- APPROVAL POLICY CONFIGURATION — mutable configuration
-- ============================================================

CREATE TABLE approval_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    rules           JSONB NOT NULL,                   -- array of condition/action rules
    -- Example: [{"condition":"amount > 10000","approver_role":"cfo","mandatory":true}]
    is_active       BOOLEAN NOT NULL DEFAULT true,
    priority        INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE match_tolerance_config (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    match_type      TEXT NOT NULL,                    -- 'price','quantity','total'
    tolerance_type  TEXT NOT NULL,                    -- 'percentage','absolute'
    tolerance_value NUMERIC(18,4) NOT NULL,
    auto_approve    BOOLEAN NOT NULL DEFAULT true,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Command Handlers (Application Layer — Pseudocode)

```python
# Example: How a PurchaseOrder aggregate processes commands

class PurchaseOrderAggregate:
    """
    Domain aggregate that processes commands and emits events.
    State is rebuilt from events, never read from projections.
    """

    def __init__(self, stream_id: UUID):
        self.stream_id = stream_id
        self.version = 0
        self.status = None
        self.lines = []
        self.total_amount = Decimal("0")

    # --- Command handlers ---

    def create(self, cmd: CreatePurchaseOrderCommand) -> list[Event]:
        if self.status is not None:
            raise DomainError("PO already exists")
        return [PurchaseOrderCreated(
            po_number=cmd.po_number,
            supplier_id=cmd.supplier_id,
            lines=cmd.lines,
            total_amount=cmd.total_amount,
        )]

    def approve(self, cmd: ApprovePurchaseOrderCommand) -> list[Event]:
        if self.status != "pending_approval":
            raise DomainError(f"Cannot approve PO in status {self.status}")
        return [PurchaseOrderApproved(
            approver_id=cmd.approver_id,
            approval_note=cmd.note,
        )]

    def receive_goods(self, cmd: RecordGoodsReceiptCommand) -> list[Event]:
        events = [GoodsReceived(
            grn_number=cmd.grn_number,
            lines=cmd.lines,
        )]
        # Check if all lines fully received
        if self._all_lines_received(cmd.lines):
            events.append(PurchaseOrderFullyReceived())
        return events

    # --- Event applicators (rebuild state from events) ---

    def apply(self, event: Event):
        match event:
            case PurchaseOrderCreated():
                self.status = "draft"
                self.lines = event.payload["lines"]
                self.total_amount = event.payload["total_amount"]
            case PurchaseOrderApproved():
                self.status = "approved"
            case PurchaseOrderFullyReceived():
                self.status = "received"
        self.version += 1
```

---

## Temporal Query Examples

```sql
-- Example 1: What was the status of PO-2026-00142 on March 15, 2026?
-- Replay events up to that date and derive state

SELECT event_type, payload, created_at
FROM event_store
WHERE stream_id = $1           -- PO stream ID
  AND created_at <= '2026-03-15 23:59:59+00'
ORDER BY event_version;

-- The application replays these events through the aggregate to derive
-- the state as of March 15.

-- Example 2: Timeline of all actions on an invoice (full audit trail)

SELECT
    e.event_type,
    e.created_at,
    e.metadata->>'actor_type' AS actor_type,
    u.full_name AS actor_name,
    e.payload
FROM event_store e
LEFT JOIN app_user u ON u.id = (e.metadata->>'actor_id')::UUID
WHERE e.stream_id = $1         -- Invoice stream ID
ORDER BY e.event_version;

-- Example 3: All three-way match exceptions raised in the last 30 days

SELECT
    e.stream_id AS invoice_id,
    e.payload->>'invoice_number' AS invoice_number,
    e.payload->>'exception_type' AS exception_type,
    e.payload->>'variance' AS variance,
    e.created_at AS raised_at
FROM event_store e
WHERE e.tenant_id = $1
  AND e.event_type = 'MatchExceptionRaised'
  AND e.created_at >= now() - INTERVAL '30 days'
ORDER BY e.created_at DESC;

-- Example 4: AI classification accuracy over time
-- (how often was the AI classification corrected by humans?)

SELECT
    DATE_TRUNC('month', corrected.created_at) AS month,
    COUNT(*) AS corrections,
    AVG((classified.payload->>'confidence')::NUMERIC) AS avg_original_confidence
FROM event_store corrected
JOIN event_store classified
  ON classified.stream_id = corrected.stream_id
  AND classified.event_type = 'SpendClassified'
  AND classified.payload->>'line_number' = corrected.payload->>'line_number'
WHERE corrected.tenant_id = $1
  AND corrected.event_type = 'SpendClassificationCorrected'
  AND corrected.created_at >= now() - INTERVAL '12 months'
GROUP BY DATE_TRUNC('month', corrected.created_at)
ORDER BY month;
```

---

## Projection Rebuild Process

```sql
-- When a projection schema changes, projections are rebuilt from events.
-- This is a background job, not a blocking migration.

-- Step 1: Truncate the stale projection
TRUNCATE TABLE proj_purchase_order CASCADE;

-- Step 2: Replay all events for the stream type
-- (done in application code, not raw SQL — shown here conceptually)

-- SELECT * FROM event_store
-- WHERE stream_type = 'purchase_order'
-- ORDER BY stream_id, event_version;

-- Step 3: For each stream, rebuild the aggregate and write to projection
-- The application uses snapshots to accelerate: load latest snapshot,
-- then replay only events after snapshot_version.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (core) | 3 | event_store, event_snapshot, event_outbox |
| Event Registry | 1 | event_type_registry (documentation) |
| Reference/Config (mutable) | 6 | tenant, organization, app_user, unspsc_taxonomy, approval_policy, match_tolerance_config |
| Projection: Transactions | 4 | proj_purchase_order, proj_purchase_order_line, proj_invoice, proj_supplier |
| Projection: Analytics | 1 | proj_spend_cube |
| Projection: Queues | 2 | proj_approval_queue, proj_match_exception_queue |
| **Total** | **~17** | Plus projections grow as new read views are needed |

---

## Key Design Decisions

1. **Append-only event store as sole source of truth** — no domain entity has a mutable "current state" table. All current-state views are derived projections. This means the audit trail is not a secondary concern bolted onto the system; it IS the system.

2. **Events carry full context, not just diffs** — each event payload includes enough information to understand the business action without loading the aggregate. For example, `PurchaseOrderCreated` includes supplier name, not just supplier_id. This makes event streams human-readable and simplifies projection building.

3. **Flattened UNSPSC taxonomy** — unlike the normalized model (4 tables), this model uses a single denormalized lookup table for UNSPSC. Since taxonomy is reference data queried by code, a flat table with segment/family/class names is simpler and still supports roll-up queries via the `segment_code`, `family_code`, and `class_code` columns.

4. **CloudEvents-inspired envelope** — event metadata follows the CloudEvents 1.0 specification (source, subject, type) for interoperability. This means events can be published to external systems (Kafka, EventBridge, webhooks) without transformation.

5. **Outbox pattern for reliable event publication** — rather than publishing events directly to a message bus (which risks losing events if the bus is down), events are written to an outbox table in the same transaction as the event store insert. A background worker polls the outbox and publishes to projection builders, notification services, and analytics pipelines.

6. **Snapshot acceleration** — for aggregates with many events (e.g., a supplier with 10,000 PO events over years), periodic snapshots capture the current state so replay only processes events since the last snapshot. Snapshots are an optimization, not a source of truth.

7. **Projections are disposable and rebuildable** — any projection table can be truncated and rebuilt from the event store. This makes schema evolution for read models safe: change the projection schema, rebuild, and deploy. No data migration required.

8. **Mutable reference data separated from event-sourced domain** — tenants, users, UNSPSC taxonomy, and configuration (approval policies, tolerance rules) are stored in conventional mutable tables. Only business domain aggregates (POs, invoices, suppliers, GRNs) are event-sourced. This pragmatic boundary avoids the complexity of event-sourcing simple configuration data.

9. **Correlation IDs link related event streams** — when a requisition generates a PO which generates a GRN which triggers an invoice match, all events share a `correlation_id`. This enables end-to-end traceability of the full procurement lifecycle from a single query.

10. **AI actions are first-class events** — spend classification, auto-matching, and exception resolution by AI agents produce events with `actor_type: "ai"` and model version metadata. This creates a complete record of AI decisions for model governance, accuracy tracking, and regulatory review.
