# Procurement Automation

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source source-to-pay platform that automates PO creation, approval workflows, three-way matching, and spend analytics.

Procurement Automation is a candidate project to build a modern, open-source procurement system that closes the gap between workflow-only OSS tools (ERPNext, Odoo) and expensive AI-enabled commercial suites (SAP Ariba, Coupa, GEP SMART). It targets mid-market and SMB buyers who cannot justify $100K–$500K/yr commercial licences but still need autonomous matching, AI spend classification, and conversational intake.

---

## Why Procurement Automation?

- **Enterprise suites are prohibitively expensive.** SAP Ariba, Coupa, and GEP SMART list at $150K–$500K+/yr before implementation, with year-one professional services adding another 50–100%.
- **Implementation timelines are punishing.** Full-suite Ariba deployments routinely run 12–24 months; even Coupa typically takes 6–12 months for enterprise rollout.
- **Existing OSS lacks an AI layer.** ERPNext (MIT) and Odoo Community (LGPLv3) cover the workflow layer competently but offer no spend classification, no AI exception resolution, and no supplier risk monitoring.
- **AI capabilities are gated behind paid add-ons.** Continuous supplier risk monitoring (e.g., Coupa Risk Aware) and advanced spend analytics require expensive modules even for customers already paying enterprise licences.
- **Taxonomy mapping blocks SMB adoption.** Mapping spend to UNSPSC or custom hierarchies in incumbent tools is a months-long professional services engagement; an LLM-based classifier removes that barrier entirely.

---

## Key Features

### Intake and Requisitioning

- Natural language purchase intake: employees describe needs in plain English (e.g. "50 ergonomic chairs for the Berlin office by June, budget ~€8K") and the system generates a compliant requisition
- Configurable multi-level approval workflows with conditional routing based on spend category, amount, and vendor risk
- Conversational approval interface via Slack or Teams so approvers never need to log in to a portal
- Preferred-vendor suggestion and policy-aware routing to reduce maverick spend
- PO generation, supplier email delivery, and supplier acknowledgement tracking

### Three-Way Matching and AP Automation

- Three-way matching across PO, goods receipt note (GRN), and vendor invoice with configurable tolerance rules
- Autonomous exception resolution: AI reasons across document context, supplier history, and tolerance rules to auto-resolve common discrepancy patterns
- Exception queue for human review of cases the model cannot confidently resolve
- Quantity and rate variance tracking; automatic approval for matched invoices
- Audit trail of all procurement transactions for SOX and internal audit compliance

### AI Spend Analytics

- Zero-configuration spend classification: LLM-based categorisation of raw line-item descriptions against UNSPSC or custom taxonomies, self-improving via feedback loops
- Spend reporting by supplier, category, business unit, and time period with budget-vs-actual comparison
- Budget tracking by department, project, and cost centre with real-time alerts on approaching limits
- Duplicate-invoice detection and out-of-policy spend flags

### Supplier Management and Risk

- Vendor master with contact, banking, tax, and payment-term information
- Continuous supplier risk monitoring from external signals (financial filings, news, cyber incidents, ESG data)
- Risk scores integrated into the PO approval flow so high-risk suppliers are flagged before a PO is issued
- Supplier performance history tracking

### Sourcing and Compliance

- Strategic sourcing module: RFQ/RFP creation, multi-supplier comparison, and award workflow
- PEPPOL and UBL 2.x e-invoicing compliance for European and APAC regulatory requirements (EN 16931)
- Support for EDI X12 850 and EDIFACT ORDERS for retail and manufacturing supply chains
- UNSPSC classification and ISO 20022 alignment for payment messaging

---

## AI-Native Advantage

Incumbent procurement suites bolt AI onto legacy workflow engines; this project is designed AI-first. Autonomous three-way matching aims to auto-resolve 80%+ of invoice exceptions by reasoning across PO, GRN, invoice history, and supplier behaviour — a step beyond the exception-flagging that current tools stop at. LLM-based spend classification eliminates the months-long taxonomy-mapping engagement that blocks SMB and mid-market adoption of spend analytics in incumbent platforms. Conversational intake and approval flows replace form-based requisitioning, addressing the "shadow IT" problem of employees bypassing procurement systems.

---

## Tech Stack & Deployment

The project is positioned to be built on the Frappe Framework (MIT) or from scratch under MIT/Apache 2.0, both of which avoid IP entanglement with commercial incumbents. Self-hosted deployment is the primary target, with managed-cloud hosting as a secondary option. Open standards are first-class: PEPPOL and UBL 2.x for e-invoicing, EN 16931 for EU compliance, UNSPSC for spend taxonomy, EDI X12 850 / EDIFACT ORDERS for legacy supplier integration, and ISO 20022 for payment messaging. Integration is exposed via REST API plus event webhooks, with native connectors planned for common ERPs (NetSuite, SAP, QuickBooks, Xero, Workday) and chat-based approvals via Slack and Teams.

---

## Market Context

The global procurement-automation software market is estimated at $9B–$12B in 2026 with an 11–13% CAGR through 2031, sitting above $50T+ of indirect spend under management globally. Enterprise pricing runs $150K–$500K+/yr (SAP Ariba, Coupa, GEP SMART), mid-market $30K–$150K/yr (Zip, Jaggaer, Precoro), and SMB $6K–$20K/yr. Primary buyers are Chief Procurement Officers (strategic ROI and risk reduction), CFOs and VP Finance (spend visibility and budget control), AP managers (matching accuracy), and IT/digital-transformation teams (integration and data governance). Industry surveys cited in the research (EY, Hackett Group, Icertis) report 80–90% of procurement leaders planning AI-agent adoption within three years.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

**Candidate metadata** (from `candidate-projects.md`): Complexity 7/10 · Domain availability Medium · Demand High · Category: Business & Enterprise Operations.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. Research notes recommend MIT or Apache 2.0 to avoid IP entanglement with commercial incumbents and to permit downstream proprietary use, with Frappe Framework (MIT) identified as a candidate foundation.
