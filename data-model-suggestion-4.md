# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Procurement Automation · Created: 2026-05-12

## Philosophy

This model combines conventional relational tables for transactional CRUD operations (purchase orders, invoices, goods receipts) with a property graph layer for relationship-heavy queries that are awkward or slow in pure relational models. The graph layer models the network of connections between suppliers, buyers, items, contracts, cost centres, and approvers as nodes and edges in a lightweight graph structure implemented via PostgreSQL tables (no separate graph database required).

Procurement is inherently relationship-rich. Questions like "which suppliers share the same beneficial owner?", "what is the shortest approval path for this requisition?", "which cost centres have overlapping supplier dependencies?", and "trace the full lifecycle of this spend from requisition through payment" are graph traversal problems that relational JOINs handle poorly at scale. Property graphs excel at these queries because they follow relationships directly rather than joining on foreign keys through multiple intermediate tables.

This architecture is inspired by fraud detection platforms (Neo4j in banking), supply chain visibility tools (which model supplier tiers as graphs), and enterprise knowledge graphs. In the procurement context, the graph layer enables capabilities that are differentiating in the market: supplier concentration risk analysis (are too many critical items sourced from a single supplier network?), conflict-of-interest detection (does the approver have a relationship with the supplier?), spend flow visualization, and AI-powered supplier recommendation based on network proximity to high-performing suppliers.

The key principle: use relational tables for what they do best (transactions, three-way matching, audit trails, budget calculations) and the graph layer for what graphs do best (relationship discovery, path traversal, network analysis, impact assessment).

**Best for:** Organisations focused on supplier risk management, spend network analysis, conflict-of-interest detection, multi-tier supply chain visibility, and AI-powered procurement intelligence that exploits relationship patterns.

**Trade-offs:**
- (+) Powerful relationship queries: supplier networks, approval chains, spend flow analysis
- (+) Conflict-of-interest and fraud detection queries are natural graph traversals
- (+) Multi-tier supply chain visibility without complex recursive CTEs
- (+) AI/ML features can exploit graph embeddings for supplier recommendation and risk scoring
- (+) No separate graph database required — implemented in PostgreSQL with ltree or adjacency tables
- (-) Two mental models (relational + graph) increase developer cognitive load
- (-) Graph query optimization requires graph-specific expertise
- (-) Data must be maintained in both relational and graph representations (synchronization)
- (-) Graph queries on very large networks (millions of edges) may require dedicated graph infrastructure
- (-) Overkill for small deployments with simple supplier relationships

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| UBL 2.1 (ISO/IEC 19845) | PO and invoice relational tables align with UBL Order/Invoice structures; graph nodes link documents to their lifecycle participants |
| PEPPOL BIS 3.0 | PEPPOL participants modeled as graph nodes with edges to the documents they exchange; enables network analysis of the PEPPOL supplier community |
| EN 16931 | Invoice compliance data stored on relational invoice table; graph edges link invoices to their tax jurisdiction nodes |
| UNSPSC | UNSPSC taxonomy modeled as a graph hierarchy (segment -> family -> class -> commodity) for traversal queries like "find all suppliers who provide items in the IT segment" |
| ISO 3166 | Countries and subdivisions modeled as graph nodes; edges connect suppliers and organisations to their jurisdictions |
| ISO 17442 (LEI) | Legal Entity Identifiers stored as node properties; edges model corporate ownership and subsidiary relationships |
| GS1 | Item nodes carry GTIN properties; edges link items to their GS1 product classification |
| ISO 20022 | Payment nodes in the graph link invoices to bank transaction references for end-to-end traceability |

---

## Graph Layer (Property Graph in PostgreSQL)

```sql
-- ============================================================
-- PROPERTY GRAPH LAYER
-- ============================================================
-- A lightweight property graph implemented in PostgreSQL.
-- Nodes represent entities; edges represent relationships.
-- Both carry typed properties as JSONB.
--
-- This avoids the operational overhead of a separate graph database
-- while enabling graph traversal queries via recursive CTEs.

CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       TEXT NOT NULL,
    -- Node types:
    --   'supplier', 'organization', 'user', 'item', 'cost_centre',
    --   'purchase_order', 'invoice', 'goods_receipt', 'payment',
    --   'unspsc_segment', 'unspsc_family', 'unspsc_class', 'unspsc_commodity',
    --   'country', 'company_group', 'beneficial_owner', 'contract'
    entity_id       UUID,                            -- FK to the relational entity (nullable for graph-only nodes)
    label           TEXT NOT NULL,                    -- human-readable label (e.g., supplier name)
    properties      JSONB NOT NULL DEFAULT '{}',     -- node-specific properties
    -- Example for supplier node:
    -- {"risk_score": 72.5, "total_spend": 185000.00, "country": "DE", "status": "active"}
    -- Example for item node:
    -- {"unspsc": "56101504", "name": "Ergonomic chair", "unit_price_avg": 159.00}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gnode_tenant_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gnode_entity ON graph_node(entity_id);
CREATE INDEX idx_gnode_label ON graph_node(tenant_id, label);
CREATE INDEX idx_gnode_properties ON graph_node USING GIN (properties);

CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_node(id),
    target_node_id  UUID NOT NULL REFERENCES graph_node(id),
    edge_type       TEXT NOT NULL,
    -- Edge types:
    --   'SUPPLIES_TO'          supplier -> organization
    --   'SUPPLIED_BY'          purchase_order -> supplier
    --   'REQUESTED_BY'         purchase_requisition -> user
    --   'APPROVED_BY'          purchase_order -> user
    --   'CONTAINS_ITEM'        purchase_order -> item
    --   'CATEGORIZED_AS'       item -> unspsc_commodity
    --   'PARENT_CATEGORY'      unspsc_commodity -> unspsc_class -> unspsc_family -> unspsc_segment
    --   'MATCHED_TO'           invoice -> purchase_order
    --   'RECEIVED_FOR'         goods_receipt -> purchase_order
    --   'PAID_VIA'             invoice -> payment
    --   'CHARGED_TO'           purchase_order -> cost_centre
    --   'SUBSIDIARY_OF'        organization -> company_group
    --   'OWNED_BY'             supplier -> beneficial_owner
    --   'LOCATED_IN'           supplier -> country
    --   'REPORTS_TO'           user -> user (approval hierarchy)
    --   'COMPETES_WITH'        supplier -> supplier (same category)
    --   'CONFLICT_OF_INTEREST' user -> supplier
    properties      JSONB NOT NULL DEFAULT '{}',     -- edge-specific properties
    -- Example for SUPPLIES_TO edge:
    -- {"since": "2024-01-15", "total_orders": 42, "total_spend": 185000.00, "avg_lead_time_days": 12}
    -- Example for APPROVED_BY edge:
    -- {"approved_at": "2026-05-10T14:30:00Z", "step": 2, "amount": 9460.50}
    weight          NUMERIC(10,4) DEFAULT 1.0,       -- for weighted graph algorithms
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT now(),
    valid_to        TIMESTAMPTZ,                     -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gedge_source ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_gedge_target ON graph_edge(target_node_id, edge_type);
CREATE INDEX idx_gedge_type ON graph_edge(tenant_id, edge_type);
CREATE INDEX idx_gedge_tenant ON graph_edge(tenant_id);
CREATE INDEX idx_gedge_properties ON graph_edge USING GIN (properties);
CREATE INDEX idx_gedge_valid ON graph_edge(valid_from, valid_to);
```

---

## Relational Transaction Layer

```sql
-- ============================================================
-- TENANT & USERS (same as other models)
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    config          JSONB NOT NULL DEFAULT '{}',
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
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph layer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    roles           JSONB NOT NULL DEFAULT '[]',
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph layer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

-- ============================================================
-- SUPPLIERS (relational master + graph node link)
-- ============================================================

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    peppol_id       TEXT,
    lei             TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    risk_score      NUMERIC(5,2),
    risk_updated_at TIMESTAMPTZ,
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    payment_terms_days INTEGER NOT NULL DEFAULT 30,
    contacts        JSONB NOT NULL DEFAULT '[]',
    addresses       JSONB NOT NULL DEFAULT '[]',
    bank_accounts   JSONB NOT NULL DEFAULT '[]',
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph layer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_tenant ON supplier(tenant_id, status);
CREATE INDEX idx_supplier_name ON supplier(tenant_id, name);
CREATE INDEX idx_supplier_risk ON supplier(tenant_id, risk_score);

-- ============================================================
-- ITEMS & UNSPSC
-- ============================================================

CREATE TABLE unspsc_commodity (
    code            CHAR(8) PRIMARY KEY,
    segment_code    CHAR(2) NOT NULL,
    segment_name    TEXT NOT NULL,
    family_code     CHAR(4) NOT NULL,
    family_name     TEXT NOT NULL,
    class_code      CHAR(6) NOT NULL,
    class_name      TEXT NOT NULL,
    commodity_name  TEXT NOT NULL,
    graph_node_id   UUID REFERENCES graph_node(id)
);

CREATE TABLE item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    sku             TEXT,
    gtin            TEXT,
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    graph_node_id   UUID REFERENCES graph_node(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_item_tenant ON item(tenant_id);
CREATE INDEX idx_item_unspsc ON item(unspsc_code);

-- ============================================================
-- COST CENTRES & BUDGETS
-- ============================================================

CREATE TABLE cost_centre (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES cost_centre(id),
    graph_node_id   UUID REFERENCES graph_node(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE TABLE budget (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    cost_centre_id  UUID NOT NULL REFERENCES cost_centre(id),
    fiscal_year     SMALLINT NOT NULL,
    amount          NUMERIC(18,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    committed       NUMERIC(18,2) NOT NULL DEFAULT 0,
    spent           NUMERIC(18,2) NOT NULL DEFAULT 0,
    alert_threshold_pct NUMERIC(5,2) DEFAULT 80.00,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(cost_centre_id, fiscal_year)
);

CREATE TABLE gl_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);
```

---

## Purchase Orders, GRNs, and Invoices

```sql
-- ============================================================
-- PURCHASE ORDERS
-- ============================================================

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    po_number       TEXT NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    status          TEXT NOT NULL DEFAULT 'draft',
    order_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_delivery_date DATE,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(18,2) NOT NULL DEFAULT 0,
    cost_centre_id  UUID REFERENCES cost_centre(id),
    payment_terms_days INTEGER NOT NULL DEFAULT 30,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    transmission_data JSONB,
    tax_details     JSONB,
    approvals       JSONB NOT NULL DEFAULT '[]',
    notes           TEXT,
    graph_node_id   UUID REFERENCES graph_node(id),  -- link to graph layer
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);

CREATE INDEX idx_po_tenant_status ON purchase_order(tenant_id, status);
CREATE INDEX idx_po_supplier ON purchase_order(supplier_id);
CREATE INDEX idx_po_date ON purchase_order(tenant_id, order_date);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    item_id         UUID REFERENCES item(id),
    description     TEXT NOT NULL,
    quantity_ordered NUMERIC(18,4) NOT NULL,
    quantity_received NUMERIC(18,4) NOT NULL DEFAULT 0,
    quantity_invoiced NUMERIC(18,4) NOT NULL DEFAULT 0,
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    unit_price      NUMERIC(18,4) NOT NULL,
    line_total      NUMERIC(18,2) NOT NULL,
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    gl_account_id   UUID REFERENCES gl_account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(purchase_order_id, line_number)
);

CREATE INDEX idx_po_line_po ON purchase_order_line(purchase_order_id);

-- ============================================================
-- GOODS RECEIPTS
-- ============================================================

CREATE TABLE goods_receipt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    grn_number      TEXT NOT NULL,
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    receipt_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    status          TEXT NOT NULL DEFAULT 'draft',
    received_by     UUID NOT NULL REFERENCES app_user(id),
    delivery_data   JSONB,
    graph_node_id   UUID REFERENCES graph_node(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, grn_number)
);

CREATE INDEX idx_grn_po ON goods_receipt(purchase_order_id);

CREATE TABLE goods_receipt_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goods_receipt_id UUID NOT NULL REFERENCES goods_receipt(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    po_line_id      UUID NOT NULL REFERENCES purchase_order_line(id),
    quantity_received NUMERIC(18,4) NOT NULL,
    quantity_accepted NUMERIC(18,4) NOT NULL,
    quantity_rejected NUMERIC(18,4) NOT NULL DEFAULT 0,
    rejection_reason TEXT,
    inspection_data JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(goods_receipt_id, line_number)
);

CREATE INDEX idx_grn_line_grn ON goods_receipt_line(goods_receipt_id);
CREATE INDEX idx_grn_line_po_line ON goods_receipt_line(po_line_id);

-- ============================================================
-- INVOICES
-- ============================================================

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    invoice_number  TEXT NOT NULL,
    internal_ref    TEXT NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    purchase_order_id UUID REFERENCES purchase_order(id),
    status          TEXT NOT NULL DEFAULT 'received',
    match_status    TEXT NOT NULL DEFAULT 'pending',
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18,2) NOT NULL,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(18,2) NOT NULL,
    match_score     NUMERIC(5,4),
    source_data     JSONB,
    tax_details     JSONB,
    payment_data    JSONB,
    graph_node_id   UUID REFERENCES graph_node(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, supplier_id, invoice_number)
);

CREATE INDEX idx_inv_tenant_status ON invoice(tenant_id, status);
CREATE INDEX idx_inv_match ON invoice(tenant_id, match_status);
CREATE INDEX idx_inv_supplier ON invoice(supplier_id);
CREATE INDEX idx_inv_po ON invoice(purchase_order_id);

CREATE TABLE invoice_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    po_line_id      UUID REFERENCES purchase_order_line(id),
    grn_line_id     UUID REFERENCES goods_receipt_line(id),
    item_id         UUID REFERENCES item(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(18,4) NOT NULL,
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    unit_price      NUMERIC(18,4) NOT NULL,
    line_total      NUMERIC(18,2) NOT NULL,
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    match_result    JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(invoice_id, line_number)
);

CREATE INDEX idx_inv_line_invoice ON invoice_line(invoice_id);
CREATE INDEX idx_inv_line_po_line ON invoice_line(po_line_id);

-- ============================================================
-- PAYMENTS
-- ============================================================

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    amount          NUMERIC(18,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'pending',
    payment_method  TEXT NOT NULL DEFAULT 'bank_transfer',
    payment_reference TEXT,
    payment_date    TIMESTAMPTZ,
    iso20022_data   JSONB,
    graph_node_id   UUID REFERENCES graph_node(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_payment_invoice ON payment(invoice_id);
CREATE INDEX idx_payment_supplier ON payment(supplier_id);
```

---

## Audit Trail

```sql
-- ============================================================
-- AUDIT LOG
-- ============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,
    actor_id        UUID,
    actor_type      TEXT NOT NULL DEFAULT 'user',
    changes         JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at);
```

---

## Graph Synchronization Triggers

```sql
-- ============================================================
-- GRAPH SYNC: Automatically create/update graph nodes and edges
-- when relational entities are created or modified.
-- ============================================================

-- Function: Create a graph node for a new supplier
CREATE OR REPLACE FUNCTION sync_supplier_graph_node()
RETURNS TRIGGER AS $$
DECLARE
    node_id UUID;
BEGIN
    -- Create or update the graph node
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id,
        'supplier',
        NEW.id,
        NEW.name,
        jsonb_build_object(
            'risk_score', NEW.risk_score,
            'status', NEW.status,
            'currency', NEW.default_currency,
            'tax_id', NEW.tax_id,
            'peppol_id', NEW.peppol_id,
            'lei', NEW.lei
        )
    )
    ON CONFLICT (id) DO UPDATE SET
        label = EXCLUDED.label,
        properties = EXCLUDED.properties,
        updated_at = now()
    RETURNING id INTO node_id;

    -- Link the relational record to the graph node
    UPDATE supplier SET graph_node_id = node_id WHERE id = NEW.id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_supplier_graph_sync
AFTER INSERT ON supplier
FOR EACH ROW EXECUTE FUNCTION sync_supplier_graph_node();

-- Function: Create graph edges when a PO is created
CREATE OR REPLACE FUNCTION sync_po_graph_edges()
RETURNS TRIGGER AS $$
DECLARE
    po_node_id UUID;
    supplier_node_id UUID;
    user_node_id UUID;
    cc_node_id UUID;
BEGIN
    -- Get referenced graph nodes
    SELECT graph_node_id INTO supplier_node_id FROM supplier WHERE id = NEW.supplier_id;
    SELECT graph_node_id INTO user_node_id FROM app_user WHERE id = NEW.created_by;
    IF NEW.cost_centre_id IS NOT NULL THEN
        SELECT graph_node_id INTO cc_node_id FROM cost_centre WHERE id = NEW.cost_centre_id;
    END IF;

    -- Create PO graph node
    INSERT INTO graph_node (tenant_id, node_type, entity_id, label, properties)
    VALUES (
        NEW.tenant_id,
        'purchase_order',
        NEW.id,
        NEW.po_number,
        jsonb_build_object(
            'status', NEW.status,
            'total_amount', NEW.total_amount,
            'currency', NEW.currency,
            'order_date', NEW.order_date
        )
    )
    RETURNING id INTO po_node_id;

    UPDATE purchase_order SET graph_node_id = po_node_id WHERE id = NEW.id;

    -- Create edges
    IF supplier_node_id IS NOT NULL THEN
        INSERT INTO graph_edge (tenant_id, source_node_id, target_node_id, edge_type, properties, weight)
        VALUES (NEW.tenant_id, po_node_id, supplier_node_id, 'SUPPLIED_BY',
                jsonb_build_object('amount', NEW.total_amount, 'date', NEW.order_date),
                NEW.total_amount);
    END IF;

    IF user_node_id IS NOT NULL THEN
        INSERT INTO graph_edge (tenant_id, source_node_id, target_node_id, edge_type, properties)
        VALUES (NEW.tenant_id, po_node_id, user_node_id, 'REQUESTED_BY',
                jsonb_build_object('date', NEW.order_date));
    END IF;

    IF cc_node_id IS NOT NULL THEN
        INSERT INTO graph_edge (tenant_id, source_node_id, target_node_id, edge_type, properties, weight)
        VALUES (NEW.tenant_id, po_node_id, cc_node_id, 'CHARGED_TO',
                jsonb_build_object('amount', NEW.total_amount),
                NEW.total_amount);
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_po_graph_sync
AFTER INSERT ON purchase_order
FOR EACH ROW EXECUTE FUNCTION sync_po_graph_edges();
```

---

## Graph Query Examples

```sql
-- ============================================================
-- GRAPH QUERIES (PostgreSQL recursive CTEs)
-- ============================================================

-- 1. SUPPLIER CONCENTRATION RISK
-- Find all suppliers that a single cost centre depends on,
-- ranked by total spend through that cost centre

SELECT
    s.id AS supplier_id,
    s_node.label AS supplier_name,
    COUNT(DISTINCT po_edge.source_node_id) AS po_count,
    SUM((po_edge.properties->>'amount')::NUMERIC) AS total_spend,
    s.risk_score
FROM graph_edge cc_edge
JOIN graph_node po_node ON po_node.id = cc_edge.source_node_id
JOIN graph_edge po_edge ON po_edge.source_node_id = po_node.id AND po_edge.edge_type = 'SUPPLIED_BY'
JOIN graph_node s_node ON s_node.id = po_edge.target_node_id
JOIN supplier s ON s.id = s_node.entity_id
WHERE cc_edge.target_node_id = $1     -- cost centre graph node ID
  AND cc_edge.edge_type = 'CHARGED_TO'
  AND cc_edge.tenant_id = $2
GROUP BY s.id, s_node.label, s.risk_score
ORDER BY total_spend DESC;

-- 2. CONFLICT OF INTEREST DETECTION
-- Find any path between an approver and a supplier (up to 3 hops)
-- that could indicate a conflict of interest

WITH RECURSIVE coi_paths AS (
    -- Start from the approver's node
    SELECT
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        ARRAY[e.source_node_id, e.target_node_id] AS path,
        1 AS depth
    FROM graph_edge e
    WHERE e.source_node_id = $1     -- approver's graph node ID
      AND e.tenant_id = $2
      AND e.edge_type IN ('OWNED_BY', 'REPORTS_TO', 'CONFLICT_OF_INTEREST', 'SUBSIDIARY_OF')

    UNION ALL

    SELECT
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        p.path || e.target_node_id,
        p.depth + 1
    FROM graph_edge e
    JOIN coi_paths p ON e.source_node_id = p.target_node_id
    WHERE p.depth < 3
      AND e.target_node_id <> ALL(p.path)  -- prevent cycles
      AND e.tenant_id = $2
)
SELECT
    p.path,
    p.depth,
    array_agg(n.label ORDER BY array_position(p.path, n.id)) AS path_labels
FROM coi_paths p
JOIN graph_node n ON n.id = ANY(p.path)
WHERE p.target_node_id = $3         -- supplier's graph node ID
GROUP BY p.path, p.depth
ORDER BY p.depth;

-- 3. FULL PROCUREMENT LIFECYCLE TRACE
-- Trace a purchase from requisition through payment via graph traversal

WITH RECURSIVE lifecycle AS (
    -- Start from a specific PO node
    SELECT
        n.id AS node_id,
        n.node_type,
        n.label,
        n.properties,
        e.edge_type AS relationship,
        0 AS depth,
        ARRAY[n.id] AS visited
    FROM graph_node n
    LEFT JOIN graph_edge e ON e.source_node_id = n.id
    WHERE n.entity_id = $1           -- PO UUID
      AND n.node_type = 'purchase_order'

    UNION ALL

    SELECT
        n2.id,
        n2.node_type,
        n2.label,
        n2.properties,
        e2.edge_type,
        l.depth + 1,
        l.visited || n2.id
    FROM lifecycle l
    JOIN graph_edge e2 ON e2.source_node_id = l.node_id
    JOIN graph_node n2 ON n2.id = e2.target_node_id
    WHERE l.depth < 5
      AND n2.id <> ALL(l.visited)
)
SELECT node_type, label, relationship, properties, depth
FROM lifecycle
ORDER BY depth, node_type;

-- 4. SUPPLIER NETWORK ANALYSIS
-- Find all suppliers within 2 hops of a given supplier
-- (shared owners, subsidiaries, parent companies)

WITH RECURSIVE supplier_network AS (
    SELECT
        e.target_node_id AS node_id,
        e.edge_type,
        1 AS depth
    FROM graph_edge e
    WHERE e.source_node_id = $1      -- supplier's graph node ID
      AND e.edge_type IN ('OWNED_BY', 'SUBSIDIARY_OF', 'COMPETES_WITH')
      AND e.tenant_id = $2

    UNION ALL

    SELECT
        e.target_node_id,
        e.edge_type,
        sn.depth + 1
    FROM graph_edge e
    JOIN supplier_network sn ON e.source_node_id = sn.node_id
    WHERE sn.depth < 2
      AND e.edge_type IN ('OWNED_BY', 'SUBSIDIARY_OF', 'COMPETES_WITH')
      AND e.tenant_id = $2
)
SELECT DISTINCT
    n.label AS entity_name,
    n.node_type,
    sn.edge_type AS relationship,
    sn.depth,
    n.properties
FROM supplier_network sn
JOIN graph_node n ON n.id = sn.node_id;

-- 5. SPEND FLOW ANALYSIS
-- Which categories of items flow through which cost centres,
-- supplied by which suppliers? (multi-dimensional graph query)

SELECT
    cc_node.label AS cost_centre,
    s_node.label AS supplier,
    cat_node.label AS unspsc_category,
    COUNT(DISTINCT po_node.id) AS order_count,
    SUM((supply_edge.properties->>'amount')::NUMERIC) AS total_spend
FROM graph_edge charge_edge
JOIN graph_node po_node ON po_node.id = charge_edge.source_node_id AND po_node.node_type = 'purchase_order'
JOIN graph_node cc_node ON cc_node.id = charge_edge.target_node_id AND cc_node.node_type = 'cost_centre'
JOIN graph_edge supply_edge ON supply_edge.source_node_id = po_node.id AND supply_edge.edge_type = 'SUPPLIED_BY'
JOIN graph_node s_node ON s_node.id = supply_edge.target_node_id AND s_node.node_type = 'supplier'
LEFT JOIN graph_edge item_edge ON item_edge.source_node_id = po_node.id AND item_edge.edge_type = 'CONTAINS_ITEM'
LEFT JOIN graph_node item_node ON item_node.id = item_edge.target_node_id
LEFT JOIN graph_edge cat_edge ON cat_edge.source_node_id = item_node.id AND cat_edge.edge_type = 'CATEGORIZED_AS'
LEFT JOIN graph_node cat_node ON cat_node.id = cat_edge.target_node_id
WHERE charge_edge.edge_type = 'CHARGED_TO'
  AND charge_edge.tenant_id = $1
GROUP BY cc_node.label, s_node.label, cat_node.label
ORDER BY total_spend DESC;

-- 6. APPROVAL CHAIN HIERARCHY
-- Traverse the user hierarchy to find the approval chain for a spend amount

WITH RECURSIVE approval_chain AS (
    SELECT
        n.id AS node_id,
        n.entity_id AS user_id,
        n.label AS user_name,
        (n.properties->>'approval_limit')::NUMERIC AS approval_limit,
        0 AS level
    FROM graph_node n
    WHERE n.entity_id = $1           -- requesting user ID
      AND n.node_type = 'user'

    UNION ALL

    SELECT
        mgr_node.id,
        mgr_node.entity_id,
        mgr_node.label,
        (mgr_node.properties->>'approval_limit')::NUMERIC,
        ac.level + 1
    FROM approval_chain ac
    JOIN graph_edge e ON e.source_node_id = ac.node_id AND e.edge_type = 'REPORTS_TO'
    JOIN graph_node mgr_node ON mgr_node.id = e.target_node_id
    WHERE ac.level < 10
)
SELECT user_id, user_name, approval_limit, level
FROM approval_chain
WHERE approval_limit >= $2           -- spend amount requiring approval
ORDER BY level
LIMIT 1;                            -- first person in chain with sufficient authority
```

---

## Graph-Powered AI Features

```sql
-- ============================================================
-- AI / ML SUPPORT TABLES
-- ============================================================

-- Graph embeddings for ML (generated by graph neural network)
CREATE TABLE graph_embedding (
    node_id         UUID NOT NULL REFERENCES graph_node(id),
    model_version   TEXT NOT NULL,
    embedding       VECTOR(128),                     -- pgvector extension
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (node_id, model_version)
);

-- Requires pgvector extension:
-- CREATE EXTENSION IF NOT EXISTS vector;

CREATE INDEX idx_embedding_vector ON graph_embedding USING ivfflat (embedding vector_cosine_ops);

-- Use case: "Find suppliers similar to our best-performing supplier"
-- SELECT
--     n.label AS supplier_name,
--     n.properties->>'risk_score' AS risk_score,
--     1 - (e1.embedding <=> e2.embedding) AS similarity
-- FROM graph_embedding e1
-- JOIN graph_embedding e2 ON e2.model_version = e1.model_version
-- JOIN graph_node n ON n.id = e2.node_id AND n.node_type = 'supplier'
-- WHERE e1.node_id = $1  -- best supplier's node ID
--   AND e2.node_id != e1.node_id
-- ORDER BY e1.embedding <=> e2.embedding
-- LIMIT 10;

-- Spend classification with AI context
CREATE TABLE spend_classification_feedback (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    entity_type     TEXT NOT NULL,
    entity_id       UUID NOT NULL,
    original_unspsc CHAR(8),
    corrected_unspsc CHAR(8) NOT NULL,
    corrected_by    UUID NOT NULL,
    description_text TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_class_feedback_tenant ON spend_classification_feedback(tenant_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge (core property graph) |
| Graph AI | 2 | graph_embedding, spend_classification_feedback |
| Tenant & Identity | 3 | tenant, organization, app_user |
| Supplier | 1 | supplier (with JSONB for contacts/addresses) |
| Reference Data | 2 | unspsc_commodity, item |
| Budgets | 3 | cost_centre, budget, gl_account |
| Purchase Orders | 2 | purchase_order, purchase_order_line |
| Goods Receipt | 2 | goods_receipt, goods_receipt_line |
| Invoices | 2 | invoice, invoice_line |
| Payments | 1 | payment |
| Audit | 1 | audit_log |
| **Total** | **~21** | Plus graph sync triggers |

---

## Key Design Decisions

1. **Graph implemented in PostgreSQL, not a separate database** — using two tables (`graph_node`, `graph_edge`) with JSONB properties provides graph capabilities without the operational overhead of running Neo4j or Amazon Neptune alongside PostgreSQL. Recursive CTEs handle traversal queries up to 5-10 hops efficiently. For deployments that outgrow this, the graph layer can be migrated to a dedicated graph database without changing the relational layer.

2. **Every relational entity has a `graph_node_id` FK** — this bidirectional link between the relational and graph layers enables seamless queries that start in one layer and finish in the other. For example, a relational query finds invoices with exceptions, then a graph query traces each exception back through the supplier network.

3. **Graph edges are temporal** — `valid_from` and `valid_to` on edges enable temporal graph queries: "who were the approvers for this cost centre last quarter?" or "which suppliers were active in our network in 2025?" This is essential for procurement compliance where historical relationships matter.

4. **Edge weights encode spend amounts** — the `weight` column on `graph_edge` is set to the monetary amount for SUPPLIED_BY and CHARGED_TO edges. This enables weighted graph algorithms: shortest path by cost, maximum flow analysis, and spend-weighted centrality calculations.

5. **Graph synchronization via triggers** — PostgreSQL triggers automatically create graph nodes and edges when relational entities are created. This keeps the graph layer in sync without requiring application code changes for every entity type. The trade-off is trigger complexity, but it ensures the graph is always consistent with relational data.

6. **Graph embeddings for AI** — the `graph_embedding` table stores vector representations of graph nodes generated by a graph neural network (GNN). These embeddings encode structural relationships (supplier A is similar to supplier B because they supply similar items to similar cost centres) and power ML features like supplier recommendation and anomaly detection. Uses pgvector for efficient similarity search.

7. **UNSPSC taxonomy as graph hierarchy** — rather than just a flat lookup table, UNSPSC categories are modeled as graph nodes with PARENT_CATEGORY edges. This enables traversal queries like "find all suppliers who provide items in the IT segment (including all sub-categories)" without complex LIKE queries on UNSPSC codes.

8. **Conflict-of-interest as a first-class edge type** — the CONFLICT_OF_INTEREST edge type between user and supplier nodes enables automated COI detection during the approval workflow. When a PO requires approval, the system queries for any path between the approver and the supplier within 3 hops.

9. **Relational tables handle the transactional workload** — the graph layer is for analytical and relationship queries, not for CRUD operations. Three-way matching, budget calculations, and PO status updates all happen in the relational layer. The graph layer is updated asynchronously and is eventually consistent.

10. **Graph enables multi-tier supply chain visibility** — by modeling supplier ownership chains (SUBSIDIARY_OF, OWNED_BY edges), the graph layer reveals supplier concentration risks that are invisible in a flat relational model. If three apparently independent suppliers are all subsidiaries of the same parent company, the graph makes this visible immediately.
