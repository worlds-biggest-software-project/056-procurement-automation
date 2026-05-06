# Standards & API Reference

> Project: Procurement Automation · Generated: 2026-05-06

## Industry Standards & Specifications

### ISO Standards

**ISO 20400:2017 — Sustainable Procurement — Guidance**
- URL: https://www.iso.org/standard/63026.html
- Provides guidance on integrating environmental, social, and governance (ESG) sustainability considerations across the full procurement life cycle, from policy development and supplier selection through to contract performance monitoring. Not a certifiable standard, but increasingly required in enterprise RFPs.

**ISO 9001:2015 — Quality Management Systems — Requirements**
- URL: https://www.iso.org/standard/62085.html
- Clause 8.4 ("Control of externally provided processes, products and services") defines requirements for evaluating, selecting, and monitoring suppliers. A procurement automation platform that surfaces supplier scorecards and audit trails directly supports ISO 9001 compliance workflows.

**ISO/IEC 19845:2015 — Universal Business Language (UBL) 2.1**
- URL: https://docs.oasis-open.org/ubl/UBL-2.1.html
- The international standard for XML-based electronic business documents covering purchase orders, invoices, despatch advice, and 62 other transaction types. Mandated for EU public-sector e-procurement and widely adopted in private-sector B2B commerce.

**ISO/IEC 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- Clauses 5.19–5.23 define controls for supplier information security. Procurement platforms that handle supplier personal data and payment information must align with ISMS controls including vendor risk assessment, data processing agreements, and ongoing monitoring obligations.

---

### W3C & IETF Standards

**IETF RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- De facto standard for securing API access across all major procurement platforms (SAP Ariba, Coupa, Zip, Tipalti). Defines four grant types; the Client Credentials flow (two-legged OAuth) is most common for system-to-system procurement integrations where no human user interaction is needed.

**IETF RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Widely used alongside OAuth 2.0 for bearer token representation in procurement API calls. SAP Ariba, Coupa, and other platforms issue JWT access tokens that encode scope, expiry, and tenant context.

**IETF RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://datatracker.ietf.org/doc/html/rfc7231
- Foundational HTTP standard governing REST API method semantics (GET, POST, PUT, PATCH, DELETE) and status codes; all REST-based procurement APIs conform to this specification.

**IETF RFC 8288 — Web Linking**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Defines Link header semantics used for pagination in procurement REST APIs (e.g. `next`, `prev` relations for navigating large PO or invoice result sets).

---

### Data Model & Message Standards

**OASIS UBL 2.1 / UBL 2.2 Specification**
- URL: https://docs.oasis-open.org/ubl/UBL-2.1.html / https://docs.oasis-open.org/ubl/UBL-2.2.html
- Open XML library of 65 standardised business document schemas including Order, Invoice, DespatchAdvice, OrderResponse, and CreditNote. The definitive schema for interoperable procurement document exchange; also the encoding syntax for the EU's EN 16931 standard and PEPPOL network.

**EN 16931:2017 (updated 2026) — European E-Invoicing Semantic Data Model**
- URL: https://ec.europa.eu/digital-building-blocks/sites/spaces/DIGITAL/pages/obtaining-a-copy
- CEN standard defining the semantic core of electronic invoices for EU business-to-government and business-to-business invoicing. Mandatory for public procurement across all EU member states since 2019. The 2026 revision (EN 16931-1:2026) aligns with the VAT in the Digital Age (ViDA) directive. Has two official syntax bindings: UBL 2.1 and UN/CEFACT CII.

**UN/EDIFACT — ORDERS, INVOIC, ORDRSP, DESADV Message Types**
- URL: https://unece.org/trade/uncefact/introducing-unedifact
- International EDI standard (United Nations/Electronic Data Interchange for Administration, Commerce and Transport) still dominant in manufacturing, retail, and logistics supply chains globally. The ORDERS message maps to purchase orders; INVOIC maps to invoices; DESADV to advance ship notices. Requires translation from modern JSON/XML formats for legacy supplier integration.

**ANSI ASC X12 — Transaction Sets 850 (Purchase Order) and 810 (Invoice)**
- URL: https://www.x12.org/
- North American EDI standard governed by the Accredited Standards Committee X12. The 850 transaction set is the electronic equivalent of a purchase order; the 810 is the invoice response. Still the default EDI format for major US retailers, healthcare procurement (HIPAA), and automotive supply chains.

**cXML (Commerce XML) — Ariba Protocol**
- URL: https://cxml.org/
- Open XML protocol originally created by Ariba in 1999 for communication between procurement applications and suppliers. Covers PunchOut (catalogue browsing), purchase orders, order confirmations, ship notices, and invoices. Widely used in SAP Ariba-connected supplier networks; freely published with no licensing restrictions.

**OAGIS (Open Applications Group Integration Specification)**
- URL: https://www.oagi.org/
- XML-based business document standard from the Open Applications Group covering procurement, order management, and supply chain messages. Versions 9 and 10 are in active use, particularly in manufacturing ERP integrations (SAP, Oracle, Infor). Provides canonical data models for PurchaseOrder, ReceiveDelivery, and SupplierInvoice messages.

**OpenAPI Specification v3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0.html
- The vendor-neutral standard for describing REST APIs using JSON or YAML. Used by all modern procurement platforms to publish machine-readable API contracts. Enables automatic SDK generation, API gateway configuration, and interactive documentation tooling (Swagger UI, Redoc).

**PEPPOL (Pan-European Public Procurement Online)**
- URL: https://peppol.org/ / https://docs.peppol.eu/
- A federated network and set of interoperability specifications maintained by OpenPeppol enabling cross-border e-procurement document exchange. Uses Peppol-UBL (a constrained profile of UBL 2.1) for orders, invoices, shipping notices, and catalogues. As of early 2026, over 2.5 million organisations in 111 countries are registered participants. Mandatory for all EU public-sector institutions to receive Peppol invoices since April 2020.

**GS1 Standards — GTIN, SSCC, GS1 XML**
- URL: https://www.gs1.org/standards
- Barcode and product identification standards (Global Trade Item Number, Serial Shipping Container Code) essential for goods receipt and three-way matching in physical goods procurement. GS1 XML provides structured data models for purchase orders and advance ship notices aligned with UN/EDIFACT and UBL.

**ISO 20022 — Universal Financial Industry Message Scheme**
- URL: https://www.iso20022.org/
- Financial messaging standard underpinning global payment rails (SWIFT, SEPA, Fedwire). Directly relevant to the payment leg of source-to-pay procurement cycles. Defines XML message schemas for payment initiation, bank-to-customer statements, and remittance advice that connect to AP automation workflows.

**UNSPSC — United Nations Standard Products and Services Code**
- URL: https://www.unspsc.org/
- Hierarchical taxonomy for classifying products and services, widely used in spend analytics and category management. Mandatory for public-sector contracts in many jurisdictions and used by SAP Ariba, Coupa, and JAGGAER for spend cube construction and AI-driven categorisation.

---

### Security & Authentication Standards

**OWASP API Security Top 10 — 2023 Edition**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- Defines the ten most critical API security risks including Broken Object Level Authorization (API1), Broken Authentication (API2), and Unsafe Consumption of Third-Party APIs (API10). Procurement platforms handling supplier financial data and PO approval workflows are high-value targets and must address all ten risks.

**OpenID Connect (OIDC) 1.0**
- URL: https://openid.net/connect/
- Identity layer built on top of OAuth 2.0; used by Coupa and other platforms for SSO and delegated user authentication. Enables procurement systems to integrate with enterprise identity providers (Azure AD, Okta, Ping).

**GDPR — General Data Protection Regulation (EU) 2016/679**
- URL: https://gdpr-info.eu/
- Applies to procurement systems that process personal data of EU-based supplier contacts and employees. Requires Data Processing Agreements (Article 28) for all processors handling personal data, lawful basis for processing, breach notification within 72 hours, and rights of access/erasure for data subjects. Fines of up to 4% of global annual revenue for non-compliance.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic Open Standard**
- URL: https://modelcontextprotocol.io/specification/2025-11-25
- Open protocol (announced November 2024) for connecting AI assistants to external data sources and tools via a standardised client-server architecture. Directly applicable to procurement AI agents that need to query live PO status, supplier databases, spend analytics, or approval queues. Enterprise adoption growing rapidly; as of 2026, MCP compliance is increasingly an enterprise procurement requirement when evaluating AI vendors and agentic systems.

---

## Similar Products — Developer Documentation & APIs

### SAP Ariba

- **Description:** Enterprise source-to-pay platform with supplier network of 5M+ organisations. Covers sourcing, contracting, purchasing, invoicing, and payment.
- **API Documentation:** https://developer.ariba.com/ (SAP Ariba Developer Portal)
- **Additional Docs:** https://help.sap.com/docs/ariba-apis (SAP Help Portal — Ariba APIs)
- **SDKs/Libraries:** No official SDKs; community wrappers available via SAP API Business Hub
- **Developer Guide:** https://help.sap.com/docs/ariba-apis/help-for-sap-ariba-developer-portal/use-of-api-gateway-and-oauth-to-authenticate-applications
- **Standards:** REST/JSON, cXML for supplier transactions, OpenAPI definitions available on SAP API Business Hub
- **Authentication:** OAuth 2.0 Client Credentials (two-legged); API Gateway key required alongside access token
- **Key APIs:** Operational Reporting for Procurement (synchronous and asynchronous), Document Approval API, Supplier Data API, Network Catalog Management API, Asset Management API

### Coupa

- **Description:** Cloud-native Business Spend Management platform covering procurement, invoicing, expenses, pay, and treasury. Known for strong UX and AI-driven spend visibility.
- **API Documentation:** https://compass.coupa.com/en-us/products/product-documentation/integration-technical-documentation/the-coupa-core-api
- **Purchase Orders API:** https://compass.coupa.com/en-us/products/product-documentation/integration-technical-documentation/the-coupa-core-api/resources/transactional-resources/purchase-orders-api-(purchase_orders)
- **Invoices API:** https://compass.coupa.com/en-us/products/product-documentation/integration-technical-documentation/the-coupa-core-api/resources/transactional-resources/invoices-api-(invoices)
- **GraphQL:** https://compass.coupa.com/en-us/products/product-documentation/integration-technical-documentation/the-coupa-core-api/get-started-with-the-api/introducing-graphql
- **SDKs/Libraries:** No official SDKs; REST and GraphQL APIs callable from any HTTP client
- **Standards:** REST/JSON, XML (optional), GraphQL, cXML for supplier portal, OpenID Connect (OIDC) for SSO
- **Authentication:** OAuth 2.0 (OpenID Connect); OAuth 2.0 Getting Started: https://compass.coupa.com/en-us/products/total-spend-management-platform/integration-playbooks-and-resources/integration-knowledge-articles/oauth-2.0-getting-started-with-coupa-api
- **Notes:** Strict 50-record offset pagination; rate limits not fully documented; responses available in JSON or XML

### Zip (ZipHQ)

- **Description:** Intake-to-pay orchestration platform that sits above ERPs to unify purchase requests, multi-stakeholder approvals, and supplier onboarding. Strong in mid-market and enterprise.
- **API Documentation:** https://docs.us.zip.co/docs/api-integration
- **Developer Docs:** https://docs.ziphq.com/hc/en-us
- **SDKs/Libraries:** No official SDKs; REST API with JSON
- **Developer Guide:** https://docs.us.zip.co/docs/api-implementation
- **Standards:** REST/JSON; supports REST/SOAP-based endpoint connectors via low-code integration platform
- **Authentication:** OAuth 2.0 Client Credentials

### Tipalti Procurement

- **Description:** AP automation and procurement platform targeting mid-market and high-growth companies. Strong on global payments, supplier onboarding, and purchase-to-pay workflows.
- **API Documentation:** https://help.tipalti.com/hc/en-us/articles/30718248220823-Procurement-REST-API-documentation
- **Developer Hub:** https://developer.tipalti.com/
- **SDKs/Libraries:** Available via API tracker; Elixir library available on Hex (hexdocs.pm/tipalti)
- **Standards:** REST/JSON; ISO 8601 date format; RESTful resource-oriented URLs
- **Authentication:** OAuth 2.0
- **Key Endpoints:** GET and UPDATE Purchase Orders, GET Purchase Requests

### JAGGAER

- **Description:** Enterprise procurement platform specialising in direct materials sourcing, supplier collaboration, and category management. Strong in manufacturing, pharma, and higher education verticals.
- **API Documentation:** https://www.jaggaer.com/solutions/integrations
- **Standards:** REST/JSON; cXML for catalogue and PO exchange; SAP and Oracle ERP connectors
- **Authentication:** OAuth 2.0; supports SAML 2.0 for SSO
- **Notes:** JAGGAER One integration framework provides prebuilt ERP connectors; REST APIs documented per vertical module (sourcing, invoicing, inventory)

### Levelpath

- **Description:** AI-native procurement orchestration platform (recognised in 2025 Gartner Procurement Orchestration Platforms report). Designed for agentic AI workflows across sourcing, purchasing, and supplier management.
- **Website:** https://www.levelpath.com/platform
- **Developer Resources:** Levelpath provides OAuth 2.0 client credential authentication, sandbox environments, and developer documentation with code examples
- **Standards:** REST/JSON; OAuth 2.0 Client Credentials; webhook-based event delivery; third-party API integration support
- **Authentication:** OAuth 2.0 Client Credentials

### Fairmarkit

- **Description:** AI-driven sourcing and supplier selection platform focused on tail spend and spot sourcing automation. Integrates with major procurement suites.
- **API Documentation:** https://docs.fairmarkit.com/buyers/integrations/sap-ariba-integration/setup-in-sap-ariba/sap-ariba-apis
- **Standards:** REST/JSON; uses SAP Ariba Operational Reporting APIs (synchronous and async variants) for integration
- **Authentication:** OAuth 2.0 (separate credentials per API)

### ERPNext / Frappe (Open Source)

- **Description:** Open-source ERP platform including a full Buying module covering supplier quotations, purchase orders, goods receipt notes (GRN), three-way matching, and payment terms.
- **API Documentation:** https://frappeframework.com/docs/user/en/api
- **GitHub:** https://github.com/frappe/erpnext
- **SDKs/Libraries:** Python (frappe-client), JavaScript; full REST API auto-generated from DocType definitions
- **Standards:** REST/JSON; OpenAPI-compatible; OAuth 2.0 and API key/token authentication
- **Authentication:** API key + secret or OAuth 2.0 Bearer Token

### Open Procurement API (Ukraine/Open Government)

- **Description:** Open-source REST API for public tender and procurement databases; reference implementation of transparent government procurement. Used as a model for open e-procurement architecture.
- **GitHub & Docs:** https://github.com/openprocurement/openprocurement.api
- **Standards:** REST/JSON; predictable resource-oriented URLs; ISO 8601 dates; open data principles
- **Authentication:** Public read access; authenticated write access for registered contracting authorities

---

## Notes

**Evolving Landscape:** The procurement API landscape is undergoing rapid consolidation around OAuth 2.0 and REST/JSON as the baseline, with GraphQL adoption growing (Coupa is a leader here). Legacy EDI formats (X12, EDIFACT) persist in large enterprise and retail supply chains and will require translation layers for the foreseeable future.

**PEPPOL Expansion:** PEPPOL is expanding rapidly beyond Europe — Australia, New Zealand, Singapore, Japan, and the US (OpenPEPPOL) are adopting it for public-sector e-invoicing. Any procurement platform targeting global or government markets should plan PEPPOL Access Point connectivity.

**AI Agent Integration:** MCP is emerging as the standard integration layer for agentic AI in enterprise procurement. Platforms that expose MCP-compliant tool servers will have a significant advantage as AI-assisted procurement workflows mature through 2026 and beyond.

**EN 16931 Update:** The 2026 revision (EN 16931-1:2026) is aligned with the EU's ViDA (VAT in the Digital Age) directive. This will mandate structured e-invoicing for all B2B transactions within the EU, dramatically expanding the compliance surface for any platform handling EU supplier invoices.

**Rate Limiting & Pagination:** Most procurement APIs (notably Coupa) have undocumented or poorly documented rate limits. Integration implementations should implement exponential back-off and respect `Retry-After` headers. Coupa's hard 50-record offset pagination is a known friction point for bulk data extraction use cases.
