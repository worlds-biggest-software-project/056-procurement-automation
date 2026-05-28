# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Procurement Automation · Created: 2026-05-12

## Philosophy

This model follows the classical normalized relational approach where every domain concept gets its own dedicated table with well-defined foreign key relationships. The procurement domain has a clear, well-understood entity structure — purchase requisitions, purchase orders, goods receipts, invoices, suppliers, and spend categories — that maps naturally to a normalized schema. The source of truth is always the relational tables, and referential integrity is enforced at the database level.

Real-world systems that use this pattern include SAP S/4HANA's procurement tables, ERPNext/Frappe's DocType-per-table design, and Oracle Fusion Procurement. These systems prioritize data integrity, complex cross-entity joins (especially for three-way matching), and compliance with regulatory requirements that demand clear data lineage.

This approach is best suited for teams that value data integrity above all else, need complex cross-entity queries (e.g., "show me all invoices matched to POs from supplier X in Q3 that exceeded budget by more than 5%"), and operate in regulatory environments where auditors need to understand the data model. It produces the most tables but each table has a clear, single purpose.

**Best for:** Teams building a production-grade procurement platform where data integrity, regulatory compliance, and complex analytical queries are top priorities.

**Trade-offs:**
- (+) Maximum data integrity via foreign keys and constraints
- (+) Complex cross-entity queries are natural and performant with proper indexing
- (+) Easy for auditors and new developers to understand — one table per concept
- (+) Standards-aligned: table structures map directly to UBL/PEPPOL document types
- (-) High table count (~45-55 tables) increases migration complexity
- (-) Schema changes require ALTER TABLE + migration scripts for every new field
- (-) Multi-jurisdiction variability (different tax rules, document requirements) creates sparse columns or requires many junction tables
- (-) Less flexible for rapid prototyping; adding a new entity type requires DDL changes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| UBL 2.1 (ISO/IEC 19845) | Purchase order and invoice table structures mirror UBL Order and Invoice document types; line items map to UBL OrderLine |
| PEPPOL BIS 3.0 | Supplier and buyer party tables include PEPPOL participant identifiers; invoice fields align with PEPPOL BIS Billing constraints |
| EN 16931 | Invoice and credit note tables contain all mandatory semantic fields from the European e-invoicing standard |
| UNSPSC | Four-level taxonomy hierarchy (segment/family/class/commodity) stored in dedicated lookup tables referenced by line items |
| ISO 3166 | Country and subdivision codes used for addresses, tax jurisdictions, and supplier registration |
| ISO 4217 | Currency codes stored as CHAR(3) on all monetary amount fields |
| ISO 20022 | Payment instruction tables align with ISO 20022 pain.001 (payment initiation) message fields |
| GS1 (GTIN/SSCC) | Item master includes GTIN field; goods receipt lines support SSCC for pallet-level tracking |
| EDI X12 850 / EDIFACT ORDERS | PO and invoice tables include fields for EDI transaction set identifiers and interchange control numbers |

---

## Core Identity & Multi-Tenancy

```sql
-- ============================================================
-- TENANT & ORGANIZATION
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',  -- 'free','starter','professional','enterprise'
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,                            -- VAT number, EIN, etc.
    peppol_id       TEXT,                            -- PEPPOL participant identifier
    lei             TEXT,                            -- Legal Entity Identifier (ISO 17442)
    default_currency CHAR(3) NOT NULL DEFAULT 'USD', -- ISO 4217
    fiscal_year_start_month SMALLINT NOT NULL DEFAULT 1,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_organization_tenant ON organization(tenant_id);

-- ============================================================
-- USERS & ROLES (RBAC)
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    password_hash   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE TABLE role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,                   -- 'procurement_admin','approver','requester','ap_clerk','viewer'
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE user_role (
    user_id         UUID NOT NULL REFERENCES app_user(id),
    role_id         UUID NOT NULL REFERENCES role(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by      UUID REFERENCES app_user(id),
    PRIMARY KEY (user_id, role_id, organization_id)
);
```

---

## Supplier Management

```sql
-- ============================================================
-- SUPPLIER MASTER
-- ============================================================

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    duns_number     TEXT,                            -- Dun & Bradstreet identifier
    peppol_id       TEXT,                            -- PEPPOL participant ID
    lei             TEXT,                            -- ISO 17442 Legal Entity Identifier
    status          TEXT NOT NULL DEFAULT 'active',  -- 'active','inactive','blocked','pending_approval'
    risk_score      NUMERIC(5,2),                    -- 0.00 - 100.00
    risk_updated_at TIMESTAMPTZ,
    payment_terms_days INTEGER NOT NULL DEFAULT 30,
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    website         TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_tenant ON supplier(tenant_id);
CREATE INDEX idx_supplier_status ON supplier(tenant_id, status);
CREATE INDEX idx_supplier_name ON supplier(tenant_id, name);

CREATE TABLE supplier_contact (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    name            TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    role            TEXT,                             -- 'primary','billing','sales','technical'
    is_primary      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_contact_supplier ON supplier_contact(supplier_id);

CREATE TABLE supplier_address (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    address_type    TEXT NOT NULL DEFAULT 'billing', -- 'billing','shipping','registered'
    street_line_1   TEXT NOT NULL,
    street_line_2   TEXT,
    city            TEXT NOT NULL,
    state_province  TEXT,
    postal_code     TEXT,
    country_code    CHAR(2) NOT NULL,                -- ISO 3166-1 alpha-2
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_address_supplier ON supplier_address(supplier_id);

CREATE TABLE supplier_bank_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    bank_name       TEXT NOT NULL,
    account_name    TEXT NOT NULL,
    account_number_encrypted TEXT NOT NULL,           -- encrypted at application layer
    routing_number  TEXT,
    iban            TEXT,
    swift_bic       TEXT,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    is_default      BOOLEAN NOT NULL DEFAULT false,
    verified_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_bank_supplier ON supplier_bank_account(supplier_id);

CREATE TABLE supplier_risk_signal (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    signal_type     TEXT NOT NULL,                    -- 'financial','cyber','esg','geopolitical','news'
    severity        TEXT NOT NULL,                    -- 'low','medium','high','critical'
    title           TEXT NOT NULL,
    description     TEXT,
    source_url      TEXT,
    detected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_risk_supplier ON supplier_risk_signal(supplier_id);
CREATE INDEX idx_supplier_risk_severity ON supplier_risk_signal(supplier_id, severity);

CREATE TABLE supplier_performance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    on_time_delivery_pct  NUMERIC(5,2),
    quality_score         NUMERIC(5,2),
    price_competitiveness NUMERIC(5,2),
    responsiveness_score  NUMERIC(5,2),
    overall_score         NUMERIC(5,2),
    total_pos             INTEGER NOT NULL DEFAULT 0,
    total_spend           NUMERIC(18,2) NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_perf_supplier ON supplier_performance(supplier_id, period_start);
```

---

## Spend Classification (UNSPSC)

```sql
-- ============================================================
-- UNSPSC TAXONOMY (Reference Data)
-- ============================================================

CREATE TABLE unspsc_segment (
    code            CHAR(2) PRIMARY KEY,             -- e.g. '43'
    name            TEXT NOT NULL                     -- e.g. 'Information Technology Broadcasting and Telecommunications'
);

CREATE TABLE unspsc_family (
    code            CHAR(4) PRIMARY KEY,             -- e.g. '4321'
    segment_code    CHAR(2) NOT NULL REFERENCES unspsc_segment(code),
    name            TEXT NOT NULL
);

CREATE TABLE unspsc_class (
    code            CHAR(6) PRIMARY KEY,             -- e.g. '432115'
    family_code     CHAR(4) NOT NULL REFERENCES unspsc_family(code),
    name            TEXT NOT NULL
);

CREATE TABLE unspsc_commodity (
    code            CHAR(8) PRIMARY KEY,             -- e.g. '43211501'
    class_code      CHAR(6) NOT NULL REFERENCES unspsc_class(code),
    name            TEXT NOT NULL
);

-- ============================================================
-- ITEM MASTER
-- ============================================================

CREATE TABLE item (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    sku             TEXT,
    gtin            TEXT,                            -- GS1 Global Trade Item Number
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    unspsc_confidence NUMERIC(5,4),                  -- AI classification confidence 0.0000-1.0000
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',      -- 'EA','KG','L','HR','MTH', etc.
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_item_tenant ON item(tenant_id);
CREATE INDEX idx_item_unspsc ON item(unspsc_code);
CREATE INDEX idx_item_sku ON item(tenant_id, sku);
```

---

## Budgets & Cost Centres

```sql
-- ============================================================
-- BUDGET MANAGEMENT
-- ============================================================

CREATE TABLE cost_centre (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES cost_centre(id),  -- hierarchical
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);

CREATE INDEX idx_cost_centre_parent ON cost_centre(parent_id);

CREATE TABLE budget (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    cost_centre_id  UUID NOT NULL REFERENCES cost_centre(id),
    fiscal_year     SMALLINT NOT NULL,
    period_type     TEXT NOT NULL DEFAULT 'annual',   -- 'annual','quarterly','monthly'
    period_number   SMALLINT NOT NULL DEFAULT 1,      -- 1 for annual, 1-4 quarterly, 1-12 monthly
    amount          NUMERIC(18,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    committed       NUMERIC(18,2) NOT NULL DEFAULT 0, -- POs issued but not yet invoiced
    spent           NUMERIC(18,2) NOT NULL DEFAULT 0, -- invoices approved for payment
    alert_threshold_pct NUMERIC(5,2) DEFAULT 80.00,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(cost_centre_id, fiscal_year, period_type, period_number)
);

CREATE TABLE gl_account (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    account_type    TEXT NOT NULL,                    -- 'expense','asset','liability'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, code)
);
```

---

## Purchase Requisitions & Approval Workflows

```sql
-- ============================================================
-- APPROVAL WORKFLOW CONFIGURATION
-- ============================================================

CREATE TABLE approval_policy (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    priority        INTEGER NOT NULL DEFAULT 0,       -- higher = evaluated first
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE approval_policy_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES approval_policy(id),
    condition_field TEXT NOT NULL,                    -- 'total_amount','category','supplier_risk','cost_centre'
    condition_operator TEXT NOT NULL,                 -- 'gt','gte','lt','lte','eq','in','contains'
    condition_value TEXT NOT NULL,                    -- e.g. '10000' or '["IT","Marketing"]'
    step_order      SMALLINT NOT NULL,
    approver_user_id UUID REFERENCES app_user(id),
    approver_role   TEXT,                             -- alternative: role-based approval
    is_mandatory    BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_approval_rule_policy ON approval_policy_rule(policy_id);

-- ============================================================
-- PURCHASE REQUISITIONS
-- ============================================================

CREATE TABLE purchase_requisition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    requisition_number TEXT NOT NULL,
    requester_id    UUID NOT NULL REFERENCES app_user(id),
    status          TEXT NOT NULL DEFAULT 'draft',   -- 'draft','pending_approval','approved','rejected','cancelled','converted'
    title           TEXT NOT NULL,
    description     TEXT,
    original_text   TEXT,                            -- natural language input from conversational intake
    cost_centre_id  UUID REFERENCES cost_centre(id),
    gl_account_id   UUID REFERENCES gl_account(id),
    needed_by_date  DATE,
    total_estimated NUMERIC(18,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    priority        TEXT NOT NULL DEFAULT 'normal',  -- 'low','normal','high','urgent'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, requisition_number)
);

CREATE INDEX idx_pr_tenant_status ON purchase_requisition(tenant_id, status);
CREATE INDEX idx_pr_requester ON purchase_requisition(requester_id);

CREATE TABLE purchase_requisition_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    requisition_id  UUID NOT NULL REFERENCES purchase_requisition(id),
    line_number     SMALLINT NOT NULL,
    item_id         UUID REFERENCES item(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(18,4) NOT NULL,
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    estimated_unit_price NUMERIC(18,4),
    estimated_total NUMERIC(18,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    unspsc_confidence NUMERIC(5,4),                  -- AI classification confidence
    suggested_supplier_id UUID REFERENCES supplier(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(requisition_id, line_number)
);

CREATE INDEX idx_pr_line_requisition ON purchase_requisition_line(requisition_id);

CREATE TABLE approval_request (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     TEXT NOT NULL,                    -- 'purchase_requisition','purchase_order','invoice'
    entity_id       UUID NOT NULL,                    -- FK to the approvable entity
    policy_id       UUID REFERENCES approval_policy(id),
    step_order      SMALLINT NOT NULL,
    approver_id     UUID NOT NULL REFERENCES app_user(id),
    status          TEXT NOT NULL DEFAULT 'pending',  -- 'pending','approved','rejected','delegated'
    decision_at     TIMESTAMPTZ,
    decision_note   TEXT,
    delegated_to    UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_approval_entity ON approval_request(entity_type, entity_id);
CREATE INDEX idx_approval_approver ON approval_request(approver_id, status);
```

---

## Purchase Orders

```sql
-- ============================================================
-- PURCHASE ORDERS
-- ============================================================

CREATE TABLE purchase_order (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    po_number       TEXT NOT NULL,
    requisition_id  UUID REFERENCES purchase_requisition(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    status          TEXT NOT NULL DEFAULT 'draft',   -- 'draft','pending_approval','approved','sent','acknowledged','partially_received','received','invoiced','closed','cancelled'
    order_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_delivery_date DATE,
    ship_to_address_id UUID REFERENCES supplier_address(id), -- reuse address table pattern for org addresses
    bill_to_address_id UUID REFERENCES supplier_address(id),
    payment_terms_days INTEGER NOT NULL DEFAULT 30,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    shipping_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(18,2) NOT NULL DEFAULT 0,
    cost_centre_id  UUID REFERENCES cost_centre(id),
    gl_account_id   UUID REFERENCES gl_account(id),
    notes           TEXT,
    internal_notes  TEXT,
    -- EDI/e-procurement fields
    edi_interchange_id TEXT,                          -- EDI X12/EDIFACT interchange control number
    ubl_document_id TEXT,                             -- UBL Order UUID
    peppol_transmission_id TEXT,                      -- PEPPOL message ID
    sent_at         TIMESTAMPTZ,
    acknowledged_at TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);

CREATE INDEX idx_po_tenant_status ON purchase_order(tenant_id, status);
CREATE INDEX idx_po_supplier ON purchase_order(supplier_id);
CREATE INDEX idx_po_requisition ON purchase_order(requisition_id);
CREATE INDEX idx_po_order_date ON purchase_order(tenant_id, order_date);

CREATE TABLE purchase_order_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id),
    line_number     SMALLINT NOT NULL,
    requisition_line_id UUID REFERENCES purchase_requisition_line(id),
    item_id         UUID REFERENCES item(id),
    description     TEXT NOT NULL,
    quantity_ordered NUMERIC(18,4) NOT NULL,
    quantity_received NUMERIC(18,4) NOT NULL DEFAULT 0,  -- denormalized for quick matching
    quantity_invoiced NUMERIC(18,4) NOT NULL DEFAULT 0,  -- denormalized for quick matching
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    unit_price      NUMERIC(18,4) NOT NULL,
    tax_rate        NUMERIC(8,4) DEFAULT 0,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    discount_pct    NUMERIC(5,2) DEFAULT 0,
    line_total      NUMERIC(18,2) NOT NULL,
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    gl_account_id   UUID REFERENCES gl_account(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(purchase_order_id, line_number)
);

CREATE INDEX idx_po_line_po ON purchase_order_line(purchase_order_id);
CREATE INDEX idx_po_line_item ON purchase_order_line(item_id);
```

---

## Goods Receipt

```sql
-- ============================================================
-- GOODS RECEIPT NOTES (GRN)
-- ============================================================

CREATE TABLE goods_receipt (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    grn_number      TEXT NOT NULL,
    purchase_order_id UUID NOT NULL REFERENCES purchase_order(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    receipt_date    DATE NOT NULL DEFAULT CURRENT_DATE,
    status          TEXT NOT NULL DEFAULT 'draft',   -- 'draft','confirmed','partial','returned'
    received_by     UUID NOT NULL REFERENCES app_user(id),
    delivery_note_number TEXT,                        -- supplier's delivery note reference
    sscc_code       TEXT,                             -- GS1 Serial Shipping Container Code
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, grn_number)
);

CREATE INDEX idx_grn_po ON goods_receipt(purchase_order_id);
CREATE INDEX idx_grn_tenant ON goods_receipt(tenant_id, receipt_date);

CREATE TABLE goods_receipt_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goods_receipt_id UUID NOT NULL REFERENCES goods_receipt(id),
    line_number     SMALLINT NOT NULL,
    po_line_id      UUID NOT NULL REFERENCES purchase_order_line(id),
    quantity_received NUMERIC(18,4) NOT NULL,
    quantity_accepted NUMERIC(18,4) NOT NULL,
    quantity_rejected NUMERIC(18,4) NOT NULL DEFAULT 0,
    rejection_reason TEXT,
    condition       TEXT NOT NULL DEFAULT 'good',     -- 'good','damaged','defective'
    gtin            TEXT,                             -- scanned GS1 barcode
    batch_number    TEXT,
    serial_numbers  TEXT[],                           -- array of serial numbers for serialized items
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(goods_receipt_id, line_number)
);

CREATE INDEX idx_grn_line_grn ON goods_receipt_line(goods_receipt_id);
CREATE INDEX idx_grn_line_po_line ON goods_receipt_line(po_line_id);
```

---

## Invoices & Three-Way Matching

```sql
-- ============================================================
-- INVOICES
-- ============================================================

CREATE TABLE invoice (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    invoice_number  TEXT NOT NULL,                    -- supplier's invoice number
    internal_ref    TEXT NOT NULL,                    -- system-generated reference
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    purchase_order_id UUID REFERENCES purchase_order(id),
    status          TEXT NOT NULL DEFAULT 'received', -- 'received','matched','exception','approved','paid','cancelled','credited'
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    received_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18,2) NOT NULL,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    shipping_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    discount_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(18,2) NOT NULL,
    payment_terms_days INTEGER,
    -- E-invoicing compliance fields
    ubl_invoice_id  TEXT,                             -- UBL Invoice/CreditNote UUID
    peppol_transmission_id TEXT,
    en16931_compliant BOOLEAN DEFAULT false,
    edi_transaction_set TEXT,                         -- '810' for X12, 'INVOIC' for EDIFACT
    -- Source document
    source_type     TEXT NOT NULL DEFAULT 'manual',  -- 'manual','email','peppol','edi','api','ocr'
    source_document_url TEXT,                         -- link to original PDF/XML
    ocr_confidence  NUMERIC(5,4),                    -- AI extraction confidence
    -- Matching
    match_status    TEXT NOT NULL DEFAULT 'pending',  -- 'pending','auto_matched','manual_matched','exception','no_po'
    match_score     NUMERIC(5,4),                    -- AI match confidence 0-1
    matched_at      TIMESTAMPTZ,
    matched_by      UUID REFERENCES app_user(id),    -- NULL if auto-matched
    -- Payment
    payment_status  TEXT NOT NULL DEFAULT 'unpaid',  -- 'unpaid','scheduled','paid','partial'
    paid_amount     NUMERIC(18,2) NOT NULL DEFAULT 0,
    paid_at         TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, supplier_id, invoice_number)
);

CREATE INDEX idx_invoice_tenant_status ON invoice(tenant_id, status);
CREATE INDEX idx_invoice_supplier ON invoice(supplier_id);
CREATE INDEX idx_invoice_po ON invoice(purchase_order_id);
CREATE INDEX idx_invoice_match_status ON invoice(tenant_id, match_status);
CREATE INDEX idx_invoice_due_date ON invoice(tenant_id, due_date);

CREATE TABLE invoice_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    line_number     SMALLINT NOT NULL,
    po_line_id      UUID REFERENCES purchase_order_line(id),
    grn_line_id     UUID REFERENCES goods_receipt_line(id),
    item_id         UUID REFERENCES item(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(18,4) NOT NULL,
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    unit_price      NUMERIC(18,4) NOT NULL,
    tax_rate        NUMERIC(8,4) DEFAULT 0,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    line_total      NUMERIC(18,2) NOT NULL,
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    unspsc_confidence NUMERIC(5,4),
    gl_account_id   UUID REFERENCES gl_account(id),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    -- Three-way match fields
    match_status    TEXT NOT NULL DEFAULT 'pending', -- 'pending','matched','price_variance','quantity_variance','both_variance','exception'
    price_variance  NUMERIC(18,4) DEFAULT 0,         -- invoice price - PO price
    quantity_variance NUMERIC(18,4) DEFAULT 0,       -- invoice qty - received qty
    variance_pct    NUMERIC(8,4) DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(invoice_id, line_number)
);

CREATE INDEX idx_invoice_line_invoice ON invoice_line(invoice_id);
CREATE INDEX idx_invoice_line_po_line ON invoice_line(po_line_id);
CREATE INDEX idx_invoice_line_match ON invoice_line(match_status);

-- ============================================================
-- THREE-WAY MATCH CONFIGURATION
-- ============================================================

CREATE TABLE match_tolerance_rule (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    match_type      TEXT NOT NULL,                    -- 'price','quantity','total'
    tolerance_type  TEXT NOT NULL,                    -- 'percentage','absolute'
    tolerance_value NUMERIC(18,4) NOT NULL,          -- e.g. 2.00 for 2% or 5.00 for $5
    currency        CHAR(3),                          -- required if tolerance_type = 'absolute'
    auto_approve    BOOLEAN NOT NULL DEFAULT true,    -- auto-approve if within tolerance
    applies_to_category TEXT,                         -- UNSPSC segment or NULL for all
    priority        INTEGER NOT NULL DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_match_tolerance_tenant ON match_tolerance_rule(tenant_id, is_active);

-- ============================================================
-- MATCH EXCEPTIONS (for AI resolution and human review)
-- ============================================================

CREATE TABLE match_exception (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    invoice_line_id UUID REFERENCES invoice_line(id),
    exception_type  TEXT NOT NULL,                    -- 'price_over_tolerance','quantity_mismatch','no_grn','no_po','duplicate_invoice','tax_mismatch'
    severity        TEXT NOT NULL DEFAULT 'medium',   -- 'low','medium','high','critical'
    description     TEXT NOT NULL,
    -- AI resolution
    ai_resolution   TEXT,                             -- AI-suggested resolution
    ai_confidence   NUMERIC(5,4),
    ai_reasoning    TEXT,                             -- explanation of AI decision
    -- Human resolution
    resolution      TEXT,                             -- 'approved','rejected','adjusted','escalated'
    resolved_by     UUID REFERENCES app_user(id),
    resolved_at     TIMESTAMPTZ,
    resolution_note TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_match_exception_invoice ON match_exception(invoice_id);
CREATE INDEX idx_match_exception_status ON match_exception(tenant_id, resolution);
```

---

## Sourcing (RFQ/RFP)

```sql
-- ============================================================
-- STRATEGIC SOURCING
-- ============================================================

CREATE TABLE sourcing_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    event_number    TEXT NOT NULL,
    event_type      TEXT NOT NULL,                    -- 'rfq','rfp','rfi','reverse_auction'
    title           TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'draft',   -- 'draft','published','evaluation','awarded','closed','cancelled'
    published_at    TIMESTAMPTZ,
    response_deadline TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, event_number)
);

CREATE TABLE sourcing_event_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_event(id),
    line_number     SMALLINT NOT NULL,
    item_id         UUID REFERENCES item(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(18,4) NOT NULL,
    unit_of_measure TEXT NOT NULL DEFAULT 'EA',
    target_price    NUMERIC(18,4),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(sourcing_event_id, line_number)
);

CREATE TABLE sourcing_invitation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_event(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    status          TEXT NOT NULL DEFAULT 'invited', -- 'invited','viewed','responded','declined'
    invited_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    responded_at    TIMESTAMPTZ,
    UNIQUE(sourcing_event_id, supplier_id)
);

CREATE TABLE sourcing_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sourcing_event_id UUID NOT NULL REFERENCES sourcing_event(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    status          TEXT NOT NULL DEFAULT 'submitted', -- 'submitted','evaluated','awarded','rejected'
    total_amount    NUMERIC(18,2),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    lead_time_days  INTEGER,
    notes           TEXT,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    evaluated_at    TIMESTAMPTZ
);

CREATE TABLE sourcing_response_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id     UUID NOT NULL REFERENCES sourcing_response(id),
    event_line_id   UUID NOT NULL REFERENCES sourcing_event_line(id),
    unit_price      NUMERIC(18,4) NOT NULL,
    quantity_offered NUMERIC(18,4) NOT NULL,
    lead_time_days  INTEGER,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Payments

```sql
-- ============================================================
-- PAYMENT PROCESSING
-- ============================================================

CREATE TABLE payment_batch (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    batch_number    TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft',   -- 'draft','approved','submitted','processing','completed','failed'
    payment_method  TEXT NOT NULL DEFAULT 'bank_transfer', -- 'bank_transfer','check','ach','wire','virtual_card'
    total_amount    NUMERIC(18,2) NOT NULL DEFAULT 0,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    payment_date    DATE,
    -- ISO 20022 payment initiation
    iso20022_msg_id TEXT,                             -- pain.001 message ID
    created_by      UUID NOT NULL REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, batch_number)
);

CREATE TABLE payment (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_batch_id UUID REFERENCES payment_batch(id),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    invoice_id      UUID NOT NULL REFERENCES invoice(id),
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    amount          NUMERIC(18,2) NOT NULL,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    status          TEXT NOT NULL DEFAULT 'pending', -- 'pending','processing','completed','failed','reversed'
    payment_reference TEXT,
    payment_date    TIMESTAMPTZ,
    bank_account_id UUID REFERENCES supplier_bank_account(id),
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
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     TEXT NOT NULL,                    -- 'purchase_requisition','purchase_order','invoice','supplier','payment'
    entity_id       UUID NOT NULL,
    action          TEXT NOT NULL,                    -- 'created','updated','deleted','status_changed','approved','rejected','matched','paid'
    actor_id        UUID REFERENCES app_user(id),    -- NULL for system actions
    actor_type      TEXT NOT NULL DEFAULT 'user',    -- 'user','system','ai','api'
    old_values      JSONB,                            -- previous field values
    new_values      JSONB,                            -- new field values
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at);
CREATE INDEX idx_audit_actor ON audit_log(actor_id);

-- Partition by month for performance on large deployments
-- CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

---

## AI Classification & NLP

```sql
-- ============================================================
-- AI SPEND CLASSIFICATION
-- ============================================================

CREATE TABLE spend_classification_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    status          TEXT NOT NULL DEFAULT 'pending', -- 'pending','running','completed','failed'
    total_items     INTEGER NOT NULL DEFAULT 0,
    classified_items INTEGER NOT NULL DEFAULT 0,
    avg_confidence  NUMERIC(5,4),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE spend_classification_feedback (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     TEXT NOT NULL,                    -- 'purchase_requisition_line','invoice_line','item'
    entity_id       UUID NOT NULL,
    original_unspsc CHAR(8),
    corrected_unspsc CHAR(8) NOT NULL REFERENCES unspsc_commodity(code),
    corrected_by    UUID NOT NULL REFERENCES app_user(id),
    description_text TEXT NOT NULL,                   -- the text that was classified
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_classification_feedback_tenant ON spend_classification_feedback(tenant_id);

-- ============================================================
-- CONVERSATIONAL INTAKE HISTORY
-- ============================================================

CREATE TABLE intake_conversation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID NOT NULL REFERENCES app_user(id),
    channel         TEXT NOT NULL DEFAULT 'web',     -- 'web','slack','teams','email'
    requisition_id  UUID REFERENCES purchase_requisition(id),
    status          TEXT NOT NULL DEFAULT 'active',  -- 'active','converted','abandoned'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE intake_message (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conversation_id UUID NOT NULL REFERENCES intake_conversation(id),
    role            TEXT NOT NULL,                    -- 'user','assistant','system'
    content         TEXT NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_intake_msg_conversation ON intake_message(conversation_id);
```

---

## E-Invoicing Document Store

```sql
-- ============================================================
-- DOCUMENT STORAGE (for UBL/EDI/PEPPOL original documents)
-- ============================================================

CREATE TABLE document (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    entity_type     TEXT NOT NULL,                    -- 'purchase_order','invoice','goods_receipt','sourcing_event'
    entity_id       UUID NOT NULL,
    document_type   TEXT NOT NULL,                    -- 'ubl_order','ubl_invoice','edi_850','edi_810','pdf','image','peppol_order','peppol_invoice'
    file_name       TEXT NOT NULL,
    mime_type       TEXT NOT NULL,
    file_size_bytes BIGINT,
    storage_path    TEXT NOT NULL,                    -- S3/GCS path or local filesystem
    checksum_sha256 TEXT,
    uploaded_by     UUID REFERENCES app_user(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_document_entity ON document(entity_type, entity_id);
CREATE INDEX idx_document_tenant ON document(tenant_id);
```

---

## Three-Way Matching Query Example

```sql
-- Example: Find all invoices with unresolved exceptions for a tenant
SELECT
    i.invoice_number,
    i.total_amount,
    i.invoice_date,
    s.name AS supplier_name,
    po.po_number,
    il.line_number,
    il.description,
    il.quantity AS invoiced_qty,
    pol.quantity_ordered AS ordered_qty,
    grl.quantity_accepted AS received_qty,
    il.unit_price AS invoiced_price,
    pol.unit_price AS po_price,
    il.match_status,
    me.exception_type,
    me.ai_resolution,
    me.ai_confidence
FROM invoice i
JOIN supplier s ON s.id = i.supplier_id
LEFT JOIN purchase_order po ON po.id = i.purchase_order_id
JOIN invoice_line il ON il.invoice_id = i.id
LEFT JOIN purchase_order_line pol ON pol.id = il.po_line_id
LEFT JOIN goods_receipt_line grl ON grl.id = il.grn_line_id
LEFT JOIN match_exception me ON me.invoice_line_id = il.id AND me.resolution IS NULL
WHERE i.tenant_id = $1
  AND i.match_status = 'exception'
ORDER BY i.invoice_date DESC, il.line_number;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 4 | tenant, organization, app_user, role + junction |
| Supplier Management | 6 | supplier, contacts, addresses, bank accounts, risk signals, performance |
| Reference Data (UNSPSC) | 4 | segment, family, class, commodity |
| Items & Budgets | 4 | item, cost_centre, budget, gl_account |
| Requisitions & Approvals | 4 | requisition, lines, approval_policy, approval_policy_rule, approval_request |
| Purchase Orders | 2 | purchase_order, purchase_order_line |
| Goods Receipt | 2 | goods_receipt, goods_receipt_line |
| Invoices & Matching | 4 | invoice, invoice_line, match_tolerance_rule, match_exception |
| Sourcing | 5 | sourcing_event, event_line, invitation, response, response_line |
| Payments | 2 | payment_batch, payment |
| Audit & Documents | 2 | audit_log, document |
| AI & Intake | 4 | classification_job, classification_feedback, intake_conversation, intake_message |
| **Total** | **~43** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — enables distributed ID generation, safe cross-system references, and prevents enumeration attacks. The trade-off is larger indexes versus sequential IDs, but modern PostgreSQL handles this well with gen_random_uuid().

2. **Denormalized quantity counters on PO lines** — `quantity_received` and `quantity_invoiced` on `purchase_order_line` are technically derivable from GRN and invoice lines, but pre-computing them avoids expensive joins during the most frequent operation: checking whether a PO line is fully received/invoiced.

3. **Separate UNSPSC taxonomy tables** — rather than storing UNSPSC codes as opaque strings, the four-level hierarchy is modeled as four tables with foreign keys. This enables roll-up analytics ("show me all IT spend at the segment level") and validates codes at insert time.

4. **Polymorphic approval_request** — uses `entity_type` + `entity_id` to approve requisitions, POs, or invoices through the same workflow engine. This avoids duplicating approval logic across three separate approval tables.

5. **E-invoicing fields on invoice table** — UBL, PEPPOL, and EDI identifiers are first-class columns rather than JSONB, because they are indexed and queried for compliance reporting and duplicate detection.

6. **Match exception table separates AI and human resolution** — the AI proposes a resolution with confidence and reasoning; a human can accept, override, or escalate. This creates a feedback loop for model improvement.

7. **Audit log as append-only with JSONB diffs** — captures old/new values as JSONB for flexibility while keeping entity_type and entity_id as indexed columns for fast lookup. Designed for monthly partitioning at scale.

8. **Supplier bank account encryption at application layer** — sensitive payment data is stored encrypted; the database never sees plaintext account numbers. This aligns with PCI DSS and ISO 27001 requirements.

9. **Tenant-scoped unique constraints** — business identifiers (PO numbers, invoice numbers) are unique within a tenant, not globally. This uses composite unique constraints like `UNIQUE(tenant_id, po_number)`.

10. **Conversational intake stored separately from requisitions** — the raw conversation history is preserved for AI training and audit purposes, linked to the resulting requisition via FK. This keeps the requisition table clean while maintaining full traceability.
