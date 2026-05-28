# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Procurement Automation · Created: 2026-05-12

## Philosophy

This model uses a hybrid approach: core relational tables with well-defined columns for the stable, universally-needed fields (PO number, supplier ID, total amount, status), combined with JSONB columns for variable, jurisdiction-specific, or rapidly-evolving fields. The relational backbone provides the query performance and referential integrity needed for three-way matching and spend analytics, while JSONB columns absorb the variability that would otherwise require dozens of sparse columns or complex EAV (Entity-Attribute-Value) patterns.

This philosophy is proven in production at scale. PostgreSQL's JSONB with GIN indexing offers near-relational query performance for structured JSON queries. Platforms like Stripe (payment metadata), Shopify (metafields), and modern ERP systems use this pattern to balance schema stability with per-customer extensibility. In the procurement domain, this approach is particularly valuable because tax structures differ by jurisdiction (EU VAT vs. US sales tax vs. GST), e-invoicing requirements vary by country (PEPPOL in Europe, ZATCA in Saudi Arabia, FEL in Guatemala), and custom fields are a universal buyer requirement that procurement platforms must support without schema changes.

The key insight is that a procurement platform serving the mid-market must accommodate dozens of jurisdiction-specific field requirements without forcing every tenant to see every field. JSONB columns provide a clean separation: the relational columns handle the 80% of fields that are universal, and the JSONB columns handle the 20% that vary by tenant, jurisdiction, or integration.

**Best for:** Teams building a multi-market SaaS platform that must accommodate jurisdiction-specific requirements, custom fields, rapid feature iteration, and diverse integration formats (UBL, EDI, cXML) without continuous schema migrations.

**Trade-offs:**
- (+) Fastest path to MVP — add new field types without ALTER TABLE
- (+) Multi-jurisdiction support without sparse columns (EU VAT fields, US 1099 fields, APAC GST fields all live in JSONB)
- (+) Custom fields for tenants are native — no EAV table needed
- (+) Integration payloads (UBL XML parsed to JSON, EDI translated to JSON) stored in original structure alongside normalized fields
- (+) Fewer tables than a fully normalized model (~30 vs ~45)
- (-) JSONB fields lack database-level referential integrity — validation must happen in application code
- (-) Complex JSONB queries can be slower than indexed relational columns (mitigated by GIN indexes)
- (-) Schema documentation requires discipline — JSONB contents must be documented separately
- (-) JSONB data is harder to report on with standard BI tools that expect flat relational tables
- (-) Risk of "JSONB everything" anti-pattern if discipline lapses — must maintain clear boundaries

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| UBL 2.1 (ISO/IEC 19845) | Full UBL document parsed and stored in `ubl_data JSONB` column alongside normalized relational fields; enables round-trip UBL generation |
| PEPPOL BIS 3.0 | PEPPOL-specific fields (endpoint ID, scheme, process ID) stored in `peppol_data JSONB` on invoice and PO tables |
| EN 16931 | EU e-invoicing semantic fields stored in `tax_details JSONB` with structure matching EN 16931 mandatory field list |
| UNSPSC | UNSPSC codes stored relationally for indexing; the full classification context (confidence, alternatives) stored in `classification_data JSONB` |
| ISO 3166 | Country codes as relational CHAR(2) columns; subdivision hierarchies in `jurisdiction_data JSONB` |
| ISO 4217 | Currency as relational CHAR(3); multi-currency conversion context in `currency_data JSONB` |
| ISO 20022 | Payment instruction details stored in `payment_data JSONB` matching ISO 20022 pain.001 structure |
| EDI X12 / EDIFACT | Original EDI segments stored in `edi_data JSONB` alongside normalized fields for dual-format support |
| cXML | PunchOut and cXML order payloads stored in `cxml_data JSONB` for SAP Ariba network integration |

---

## Core Tables

```sql
-- ============================================================
-- TENANT & MULTI-TENANCY
-- ============================================================

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    subscription_tier TEXT NOT NULL DEFAULT 'free',
    -- Tenant-wide configuration as JSONB — avoids a separate settings table
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "default_currency": "EUR",
    --   "fiscal_year_start": 1,
    --   "peppol_enabled": true,
    --   "peppol_endpoint_id": "0192:987654321",
    --   "edi_enabled": false,
    --   "custom_fields_schema": {
    --     "purchase_order": [
    --       {"name": "project_code", "type": "string", "required": true},
    --       {"name": "carbon_offset", "type": "boolean", "required": false}
    --     ]
    --   },
    --   "approval_delegation_enabled": true,
    --   "match_tolerances": {
    --     "price_pct": 2.0,
    --     "quantity_pct": 5.0,
    --     "auto_approve_below": 500.00
    --   }
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE organization (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    tax_id          TEXT,
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    address         JSONB NOT NULL DEFAULT '{}',
    -- Example address:
    -- {
    --   "street": "Friedrichstr. 123",
    --   "city": "Berlin",
    --   "state": "BE",
    --   "postal_code": "10117",
    --   "country_code": "DE",
    --   "country_name": "Germany"
    -- }
    peppol_data     JSONB,
    -- Example: {"endpoint_id": "0192:987654321", "scheme": "iso6523-actorid-upis"}
    extra           JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_org_tenant ON organization(tenant_id);

-- ============================================================
-- USERS & RBAC
-- ============================================================

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    full_name       TEXT NOT NULL,
    password_hash   TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    roles           JSONB NOT NULL DEFAULT '[]',
    -- Example: ["procurement_admin", "approver"]
    -- Roles are tenant-scoped strings; permissions resolved at app layer
    preferences     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"notification_channel": "slack", "timezone": "Europe/Berlin"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, email)
);

CREATE INDEX idx_user_tenant ON app_user(tenant_id);
CREATE INDEX idx_user_roles ON app_user USING GIN (roles);
```

---

## Supplier Management

```sql
-- ============================================================
-- SUPPLIERS
-- ============================================================

CREATE TABLE supplier (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    legal_name      TEXT,
    status          TEXT NOT NULL DEFAULT 'active',
    -- Core relational fields (indexed, queried frequently)
    tax_id          TEXT,
    default_currency CHAR(3) NOT NULL DEFAULT 'USD',
    payment_terms_days INTEGER NOT NULL DEFAULT 30,
    risk_score      NUMERIC(5,2),
    risk_updated_at TIMESTAMPTZ,
    -- Structured JSONB for variable data
    contacts        JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"name": "Hans Mueller", "email": "hans@ergo.de", "phone": "+4930123456", "role": "primary"},
    --   {"name": "Lisa Schmidt", "email": "lisa@ergo.de", "role": "billing"}
    -- ]
    addresses       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"type": "billing", "street": "Hauptstr. 42", "city": "Munich", "country_code": "DE", "postal_code": "80331", "is_default": true},
    --   {"type": "shipping", "street": "Industriestr. 5", "city": "Stuttgart", "country_code": "DE", "postal_code": "70173"}
    -- ]
    bank_accounts   JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"bank_name": "Deutsche Bank", "iban": "DE89370400440532013000", "swift": "DEUTDEDB", "currency": "EUR", "is_default": true}
    -- ]
    -- Note: in production, bank account numbers should be encrypted at application layer
    identifiers     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"duns": "123456789", "peppol_id": "0192:987654321", "lei": "5493001KJTIIGC8Y1R12"}
    risk_signals    JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"type": "financial", "severity": "medium", "title": "Credit downgrade", "detected_at": "2026-03-15T10:00:00Z"}
    -- ]
    performance     JSONB NOT NULL DEFAULT '{}',
    -- Example: {"on_time_pct": 94.5, "quality_score": 87.2, "total_pos": 42, "total_spend": 185000.00}
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- Tenant-specific custom fields
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_supplier_tenant ON supplier(tenant_id);
CREATE INDEX idx_supplier_status ON supplier(tenant_id, status);
CREATE INDEX idx_supplier_name ON supplier(tenant_id, name);
CREATE INDEX idx_supplier_risk ON supplier(tenant_id, risk_score);
CREATE INDEX idx_supplier_identifiers ON supplier USING GIN (identifiers);

-- Query example: find all suppliers with a PEPPOL ID
-- SELECT * FROM supplier WHERE identifiers ? 'peppol_id' AND tenant_id = $1;

-- Query example: find suppliers with critical risk signals
-- SELECT * FROM supplier
-- WHERE tenant_id = $1
--   AND risk_signals @> '[{"severity": "critical"}]';
```

---

## Spend Classification

```sql
-- ============================================================
-- UNSPSC TAXONOMY (relational — reference data)
-- ============================================================

CREATE TABLE unspsc_commodity (
    code            CHAR(8) PRIMARY KEY,
    segment_code    CHAR(2) NOT NULL,
    segment_name    TEXT NOT NULL,
    family_code     CHAR(4) NOT NULL,
    family_name     TEXT NOT NULL,
    class_code      CHAR(6) NOT NULL,
    class_name      TEXT NOT NULL,
    commodity_name  TEXT NOT NULL
);

CREATE INDEX idx_unspsc_segment ON unspsc_commodity(segment_code);
CREATE INDEX idx_unspsc_family ON unspsc_commodity(family_code);
CREATE INDEX idx_unspsc_class ON unspsc_commodity(class_code);

-- ============================================================
-- ITEMS
-- ============================================================

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
    classification_data JSONB,
    -- Example: {
    --   "confidence": 0.9847,
    --   "model_version": "classify-v2.1",
    --   "classified_at": "2026-05-01T10:00:00Z",
    --   "alternatives": [
    --     {"code": "56101505", "confidence": 0.0098, "name": "Executive seating"}
    --   ],
    --   "feedback_corrections": 0
    -- }
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_item_tenant ON item(tenant_id);
CREATE INDEX idx_item_sku ON item(tenant_id, sku);
CREATE INDEX idx_item_unspsc ON item(unspsc_code);
```

---

## Budgets & Cost Centres

```sql
-- ============================================================
-- COST CENTRES & BUDGETS
-- ============================================================

CREATE TABLE cost_centre (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    code            TEXT NOT NULL,
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES cost_centre(id),
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
    periods         JSONB,
    -- Example (monthly breakdown):
    -- {
    --   "1": {"amount": 10000, "committed": 3000, "spent": 2500},
    --   "2": {"amount": 10000, "committed": 5000, "spent": 4200},
    --   ...
    -- }
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

## Purchase Requisitions

```sql
-- ============================================================
-- PURCHASE REQUISITIONS
-- ============================================================

CREATE TABLE purchase_requisition (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    organization_id UUID NOT NULL REFERENCES organization(id),
    requisition_number TEXT NOT NULL,
    requester_id    UUID NOT NULL REFERENCES app_user(id),
    status          TEXT NOT NULL DEFAULT 'draft',
    title           TEXT NOT NULL,
    description     TEXT,
    original_text   TEXT,                            -- conversational intake raw input
    cost_centre_id  UUID REFERENCES cost_centre(id),
    gl_account_id   UUID REFERENCES gl_account(id),
    needed_by_date  DATE,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    total_estimated NUMERIC(18,2),
    priority        TEXT NOT NULL DEFAULT 'normal',
    -- Lines stored as JSONB array (fewer JOINs for intake/display)
    lines           JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "line_number": 1,
    --     "description": "Ergonomic office chair, adjustable height",
    --     "item_id": "i1i2i3i4-...",
    --     "quantity": 50,
    --     "unit_of_measure": "EA",
    --     "estimated_unit_price": 159.00,
    --     "estimated_total": 7950.00,
    --     "unspsc_code": "56101504",
    --     "unspsc_confidence": 0.9847,
    --     "suggested_supplier_id": "s1s2s3s4-..."
    --   }
    -- ]
    approvals       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"step": 1, "approver_id": "u2u2-...", "status": "approved", "decided_at": "2026-05-10T14:30:00Z", "note": "OK"},
    --   {"step": 2, "approver_id": "u3u3-...", "status": "pending"}
    -- ]
    intake_conversation JSONB,
    -- Stores the conversational intake messages inline
    -- Example: [
    --   {"role": "user", "content": "I need 50 ergonomic chairs for Berlin office by June, ~€8K", "at": "2026-05-10T10:00:00Z"},
    --   {"role": "assistant", "content": "I've created a requisition for 50 ergonomic chairs...", "at": "2026-05-10T10:00:05Z"}
    -- ]
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, requisition_number)
);

CREATE INDEX idx_pr_tenant_status ON purchase_requisition(tenant_id, status);
CREATE INDEX idx_pr_requester ON purchase_requisition(requester_id);
CREATE INDEX idx_pr_lines ON purchase_requisition USING GIN (lines);
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
    status          TEXT NOT NULL DEFAULT 'draft',
    order_date      DATE NOT NULL DEFAULT CURRENT_DATE,
    expected_delivery_date DATE,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18,2) NOT NULL DEFAULT 0,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(18,2) NOT NULL DEFAULT 0,
    cost_centre_id  UUID REFERENCES cost_centre(id),
    gl_account_id   UUID REFERENCES gl_account(id),
    payment_terms_days INTEGER NOT NULL DEFAULT 30,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    -- Ship-to and bill-to as JSONB (avoids separate address junction tables)
    ship_to         JSONB,
    -- Example: {"street": "Friedrichstr. 123", "city": "Berlin", "country_code": "DE", "postal_code": "10117"}
    bill_to         JSONB,
    -- E-invoicing / EDI integration data
    transmission_data JSONB,
    -- Example: {
    --   "channel": "peppol",
    --   "peppol_message_id": "msg-uuid-...",
    --   "peppol_endpoint": "0192:987654321",
    --   "ubl_document_id": "ubl-uuid-...",
    --   "sent_at": "2026-05-11T09:00:00Z",
    --   "acknowledged_at": "2026-05-11T09:15:00Z"
    -- }
    -- OR for EDI:
    -- {
    --   "channel": "edi_x12",
    --   "interchange_control_number": "000001234",
    --   "transaction_set": "850",
    --   "sent_at": "2026-05-11T09:00:00Z"
    -- }
    tax_details     JSONB,
    -- Jurisdiction-specific tax breakdown
    -- EU example: {"vat_rate": 19.0, "vat_amount": 1510.50, "reverse_charge": false, "tax_point_date": "2026-05-11"}
    -- US example: {"sales_tax_rate": 8.875, "sales_tax_amount": 705.94, "tax_jurisdiction": "NY"}
    -- GST example: {"cgst_rate": 9.0, "sgst_rate": 9.0, "cgst_amount": 715.50, "sgst_amount": 715.50, "gstin": "29ABCDE1234F1Z5"}
    approvals       JSONB NOT NULL DEFAULT '[]',
    notes           TEXT,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, po_number)
);

CREATE INDEX idx_po_tenant_status ON purchase_order(tenant_id, status);
CREATE INDEX idx_po_supplier ON purchase_order(supplier_id);
CREATE INDEX idx_po_order_date ON purchase_order(tenant_id, order_date);
CREATE INDEX idx_po_transmission ON purchase_order USING GIN (transmission_data);

-- ============================================================
-- PURCHASE ORDER LINES (relational — needed for three-way matching JOINs)
-- ============================================================
-- Lines are kept as a separate relational table (not JSONB) because
-- three-way matching requires line-level JOINs between PO lines,
-- GRN lines, and invoice lines. JSONB-to-JSONB joins across tables
-- are too slow for this critical path.

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
    tax_rate        NUMERIC(8,4) DEFAULT 0,
    line_total      NUMERIC(18,2) NOT NULL,
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    gl_account_id   UUID REFERENCES gl_account(id),
    classification_data JSONB,
    -- AI classification context per line
    custom_fields   JSONB NOT NULL DEFAULT '{}',
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
    -- Example: {
    --   "delivery_note_number": "DN-2026-001",
    --   "carrier": "DHL",
    --   "tracking_number": "1Z999AA10123456784",
    --   "sscc_codes": ["00340012345678901234"],
    --   "delivery_photos": ["s3://bucket/photo1.jpg"]
    -- }
    notes           TEXT,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, grn_number)
);

CREATE INDEX idx_grn_po ON goods_receipt(purchase_order_id);

-- Lines are relational for three-way matching JOINs
CREATE TABLE goods_receipt_line (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    goods_receipt_id UUID NOT NULL REFERENCES goods_receipt(id) ON DELETE CASCADE,
    line_number     SMALLINT NOT NULL,
    po_line_id      UUID NOT NULL REFERENCES purchase_order_line(id),
    quantity_received NUMERIC(18,4) NOT NULL,
    quantity_accepted NUMERIC(18,4) NOT NULL,
    quantity_rejected NUMERIC(18,4) NOT NULL DEFAULT 0,
    rejection_reason TEXT,
    condition       TEXT NOT NULL DEFAULT 'good',
    inspection_data JSONB,
    -- Example: {
    --   "gtin_scanned": "00012345678905",
    --   "batch_number": "BATCH-2026-A",
    --   "serial_numbers": ["SN001", "SN002"],
    --   "expiry_date": "2028-12-31",
    --   "temperature_log": {"min": 2.1, "max": 4.8, "unit": "celsius"}
    -- }
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
    invoice_number  TEXT NOT NULL,
    internal_ref    TEXT NOT NULL,
    supplier_id     UUID NOT NULL REFERENCES supplier(id),
    purchase_order_id UUID REFERENCES purchase_order(id),
    status          TEXT NOT NULL DEFAULT 'received',
    match_status    TEXT NOT NULL DEFAULT 'pending',
    invoice_date    DATE NOT NULL,
    due_date        DATE NOT NULL,
    received_date   DATE NOT NULL DEFAULT CURRENT_DATE,
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    subtotal        NUMERIC(18,2) NOT NULL,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_amount    NUMERIC(18,2) NOT NULL,
    match_score     NUMERIC(5,4),
    -- Source document metadata
    source_data     JSONB NOT NULL DEFAULT '{}',
    -- Example (OCR source): {
    --   "type": "ocr",
    --   "document_url": "s3://invoices/inv-8847.pdf",
    --   "ocr_confidence": 0.9650,
    --   "ocr_model": "extract-v3",
    --   "extracted_fields": {"invoice_number": 0.99, "total": 0.97, "line_items": 0.94}
    -- }
    -- Example (PEPPOL source): {
    --   "type": "peppol",
    --   "transmission_id": "peppol-msg-uuid",
    --   "ubl_invoice_id": "ubl-uuid",
    --   "sender_endpoint": "0192:987654321",
    --   "en16931_valid": true,
    --   "validation_report": {"errors": 0, "warnings": 2}
    -- }
    -- Example (EDI source): {
    --   "type": "edi_x12",
    --   "transaction_set": "810",
    --   "interchange_control": "000005678",
    --   "raw_segments": "ISA*00*...\nGS*IN*..."
    -- }
    tax_details     JSONB,
    -- Same jurisdiction-variable structure as on purchase_order
    payment_data    JSONB,
    -- Example: {
    --   "status": "paid",
    --   "paid_amount": 9460.50,
    --   "paid_at": "2026-06-10T08:00:00Z",
    --   "payment_reference": "PAY-2026-0042",
    --   "payment_method": "bank_transfer",
    --   "iso20022_msg_id": "pain001-uuid"
    -- }
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, supplier_id, invoice_number)
);

CREATE INDEX idx_inv_tenant_status ON invoice(tenant_id, status);
CREATE INDEX idx_inv_match ON invoice(tenant_id, match_status);
CREATE INDEX idx_inv_supplier ON invoice(supplier_id);
CREATE INDEX idx_inv_po ON invoice(purchase_order_id);
CREATE INDEX idx_inv_due ON invoice(tenant_id, due_date);
CREATE INDEX idx_inv_source ON invoice USING GIN (source_data);

-- Lines are relational for three-way matching
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
    tax_rate        NUMERIC(8,4) DEFAULT 0,
    tax_amount      NUMERIC(18,2) NOT NULL DEFAULT 0,
    line_total      NUMERIC(18,2) NOT NULL,
    unspsc_code     CHAR(8) REFERENCES unspsc_commodity(code),
    gl_account_id   UUID REFERENCES gl_account(id),
    cost_centre_id  UUID REFERENCES cost_centre(id),
    -- Match results per line
    match_result    JSONB,
    -- Example: {
    --   "status": "matched",
    --   "price_variance": 0.00,
    --   "quantity_variance": 0,
    --   "variance_pct": 0.0,
    --   "tolerance_rule": "default-2pct",
    --   "auto_approved": true
    -- }
    -- OR:
    -- {
    --   "status": "exception",
    --   "price_variance": 15.00,
    --   "quantity_variance": -2,
    --   "variance_pct": 9.4,
    --   "exception_type": "price_over_tolerance",
    --   "ai_resolution": "approve",
    --   "ai_confidence": 0.87,
    --   "ai_reasoning": "Supplier typically adjusts price for rush orders; within historical variance",
    --   "resolved_by": null,
    --   "resolved_at": null
    -- }
    classification_data JSONB,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(invoice_id, line_number)
);

CREATE INDEX idx_inv_line_invoice ON invoice_line(invoice_id);
CREATE INDEX idx_inv_line_po_line ON invoice_line(po_line_id);
CREATE INDEX idx_inv_line_match ON invoice_line USING GIN (match_result);
```

---

## Sourcing

```sql
-- ============================================================
-- SOURCING EVENTS (RFQ/RFP)
-- ============================================================

CREATE TABLE sourcing_event (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    event_number    TEXT NOT NULL,
    event_type      TEXT NOT NULL,                    -- 'rfq','rfp','rfi','reverse_auction'
    title           TEXT NOT NULL,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'draft',
    published_at    TIMESTAMPTZ,
    response_deadline TIMESTAMPTZ,
    created_by      UUID NOT NULL REFERENCES app_user(id),
    -- Lines, invitations, and responses as JSONB
    -- (sourcing events are less query-intensive than PO/invoice matching)
    lines           JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"line_number": 1, "description": "Ergonomic chairs", "quantity": 50, "target_price": 150.00}
    -- ]
    invitations     JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {"supplier_id": "s1-...", "status": "invited", "invited_at": "2026-05-01T10:00:00Z"},
    --   {"supplier_id": "s2-...", "status": "responded", "responded_at": "2026-05-05T14:00:00Z"}
    -- ]
    responses       JSONB NOT NULL DEFAULT '[]',
    -- Example: [
    --   {
    --     "supplier_id": "s1-...",
    --     "total_amount": 7500.00,
    --     "currency": "EUR",
    --     "lead_time_days": 14,
    --     "submitted_at": "2026-05-05T14:00:00Z",
    --     "lines": [
    --       {"line_number": 1, "unit_price": 150.00, "quantity_offered": 50}
    --     ]
    --   }
    -- ]
    evaluation_criteria JSONB,
    award_data      JSONB,
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(tenant_id, event_number)
);

CREATE INDEX idx_sourcing_tenant ON sourcing_event(tenant_id, status);
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
    -- Example: {
    --   "old": {"status": "draft", "total_amount": 7950.00},
    --   "new": {"status": "approved", "total_amount": 7950.00},
    --   "ip_address": "192.168.1.42",
    --   "user_agent": "Mozilla/5.0...",
    --   "channel": "web"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at);
```

---

## JSONB Query Examples

```sql
-- 1. Find suppliers with a PEPPOL ID in a specific scheme
SELECT id, name, identifiers->>'peppol_id' AS peppol_id
FROM supplier
WHERE tenant_id = $1
  AND identifiers ? 'peppol_id';

-- 2. Find invoices received via PEPPOL with EN 16931 validation errors
SELECT id, invoice_number, source_data->'validation_report'->'errors' AS errors
FROM invoice
WHERE tenant_id = $1
  AND source_data->>'type' = 'peppol'
  AND (source_data->'validation_report'->>'errors')::INT > 0;

-- 3. Find all invoice lines with unresolved match exceptions
SELECT
    i.invoice_number,
    il.line_number,
    il.description,
    il.match_result->>'exception_type' AS exception_type,
    il.match_result->>'ai_resolution' AS ai_suggestion,
    (il.match_result->>'ai_confidence')::NUMERIC AS ai_confidence
FROM invoice i
JOIN invoice_line il ON il.invoice_id = i.id
WHERE i.tenant_id = $1
  AND il.match_result->>'status' = 'exception'
  AND il.match_result->>'resolved_by' IS NULL
ORDER BY (il.match_result->>'ai_confidence')::NUMERIC DESC;

-- 4. Budget utilization with monthly breakdown
SELECT
    cc.code, cc.name,
    b.amount,
    b.committed,
    b.spent,
    ROUND((b.spent / b.amount * 100), 2) AS utilization_pct,
    b.periods
FROM budget b
JOIN cost_centre cc ON cc.id = b.cost_centre_id
WHERE b.tenant_id = $1
  AND b.fiscal_year = 2026;

-- 5. Search suppliers by contact email (JSONB array search)
SELECT id, name
FROM supplier
WHERE tenant_id = $1
  AND contacts @> '[{"email": "hans@ergo.de"}]';

-- 6. Three-way matching join (relational lines, JSONB match result)
SELECT
    i.invoice_number,
    il.line_number,
    il.description,
    il.quantity AS invoiced_qty,
    pol.quantity_ordered,
    grl.quantity_accepted AS received_qty,
    il.unit_price AS invoiced_price,
    pol.unit_price AS po_price,
    il.match_result
FROM invoice i
JOIN invoice_line il ON il.invoice_id = i.id
LEFT JOIN purchase_order_line pol ON pol.id = il.po_line_id
LEFT JOIN goods_receipt_line grl ON grl.id = il.grn_line_id
WHERE i.tenant_id = $1
  AND i.match_status = 'exception';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 3 | tenant, organization, app_user (roles in JSONB) |
| Supplier Management | 1 | Single supplier table with JSONB for contacts, addresses, bank accounts |
| Reference Data | 2 | unspsc_commodity (flattened), item |
| Budgets | 3 | cost_centre, budget, gl_account |
| Purchase Requisitions | 1 | Lines and approvals in JSONB |
| Purchase Orders | 2 | PO header + relational lines (for matching) |
| Goods Receipt | 2 | GRN header + relational lines (for matching) |
| Invoices | 2 | Invoice header + relational lines (for matching) |
| Sourcing | 1 | Single table with lines, invitations, responses in JSONB |
| Audit | 1 | audit_log with JSONB changes |
| **Total** | **~18** | Dramatically fewer than normalized (~43) |

---

## Key Design Decisions

1. **JSONB for variable data, relational for join-critical data** — the critical boundary is three-way matching. PO lines, GRN lines, and invoice lines are relational tables because matching requires cross-table JOINs on `po_line_id` and `grn_line_id`. Everything else that varies by tenant, jurisdiction, or integration channel goes into JSONB.

2. **Requisition lines in JSONB, PO/GRN/invoice lines relational** — requisition lines are only ever read as a batch (display the full requisition). They are never joined to other tables. So JSONB is fine. PO lines, on the other hand, are the anchoring point for three-way matching and must be relational.

3. **Supplier as a single table** — instead of 6 tables (supplier + contacts + addresses + bank_accounts + risk_signals + performance), a single supplier table holds everything. Contacts, addresses, and bank accounts are arrays of objects in JSONB. This reduces JOIN count for the most common query pattern: "show me the supplier with all their details."

4. **Jurisdiction-variable tax details in JSONB** — EU VAT, US sales tax, Indian GST, and Saudi ZATCA all have different tax field structures. Rather than creating tax_eu, tax_us, tax_in, tax_sa tables, a single `tax_details JSONB` column holds whatever structure applies. Application-layer validation ensures the correct schema per jurisdiction.

5. **E-invoicing integration data in JSONB** — PEPPOL, EDI X12, EDIFACT, and cXML all have different metadata structures. `transmission_data` and `source_data` JSONB columns store channel-specific payloads without requiring a separate table per integration type.

6. **Sourcing event as a single table** — RFQ/RFP events are relatively low-volume and read-heavy. Lines, invitations, and responses are stored as JSONB arrays within the sourcing event. This dramatically simplifies the sourcing module at the cost of not being able to query individual response lines across events (acceptable for this use case).

7. **Match results inline on invoice lines** — instead of a separate match_exception table, match results (including AI resolution suggestions) are stored as JSONB on the invoice_line. This keeps the match context co-located with the line item, simplifying the most common query: "show me this invoice with all its match results."

8. **Tenant config as JSONB** — approval rules, tolerance thresholds, PEPPOL configuration, custom field schemas, and feature flags are all stored in `tenant.config`. This avoids a proliferation of configuration tables and makes it trivial to add new tenant-scoped settings.

9. **GIN indexes on critical JSONB columns** — PostgreSQL GIN indexes on JSONB enable fast containment queries (`@>`, `?`, `?|`). Applied selectively to JSONB columns that are actually queried (supplier identifiers, match results, source data), not universally.

10. **Custom fields as a JSONB escape hatch** — every major entity has a `custom_fields JSONB` column. The tenant's `config.custom_fields_schema` defines what fields are valid per entity type. This provides Salesforce-style custom fields without EAV complexity. Validation happens at the application layer using JSON Schema.
