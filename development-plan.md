# Procurement Automation — Phased Development Plan

> Project: 056-procurement-automation
> Created: 2026-05-25
> Status: Planning

---

## Technology Decisions

### Language & Framework

| Decision | Choice | Rationale |
|----------|--------|-----------|
| **Backend language** | TypeScript (Node.js) | Fastest path to MVP for a full-stack team; strong async I/O for webhook/event processing; same language front and back reduces context switching. Python considered for AI layer but TypeScript's LLM SDK ecosystem (Anthropic SDK, OpenAI SDK, LangChain.js) is now mature. |
| **Backend framework** | Fastify | Faster than Express with built-in schema validation (Ajv/JSON Schema), OpenAPI generation, and plugin architecture. Matches the need for strict API contracts (OAS 3.1) identified in standards research. |
| **Frontend framework** | Next.js 15 (App Router) | Server components for data-heavy procurement dashboards; built-in API routes for BFF pattern; React Server Components reduce client bundle for approval-heavy UIs. |
| **UI component library** | shadcn/ui + Tailwind CSS | Accessible, composable components; no vendor lock-in (copy-paste, not npm dependency); matches enterprise UX expectations identified in feature survey. |
| **Database** | PostgreSQL 16 | All four data model suggestions target PostgreSQL. Supports JSONB with GIN indexing, recursive CTEs for approval hierarchies, partitioning for audit logs, and pgvector for AI embeddings. |
| **ORM / Query builder** | Drizzle ORM | Type-safe SQL with zero runtime overhead; supports PostgreSQL-native features (JSONB operators, CTEs, array types) that Prisma abstracts away. Critical for three-way matching queries. |
| **AI/LLM integration** | Anthropic Claude API (primary), OpenAI-compatible fallback | Claude for spend classification, conversational intake, and match exception reasoning. Abstraction layer allows model swapping. Research identifies AI as the primary differentiator. |
| **Message queue** | BullMQ (Redis-backed) | For async job processing: spend classification batches, supplier risk signal ingestion, GRN/invoice matching, notification dispatch. Lightweight; no Kafka overhead for MVP. |
| **Authentication** | Auth.js (NextAuth) v5 + OIDC | Supports OAuth 2.0 / OpenID Connect for enterprise SSO (Azure AD, Okta, Google Workspace). Standards research mandates OIDC compliance. |
| **API documentation** | OpenAPI 3.1 auto-generated from Fastify schemas | Standards research identifies OAS 3.1 as the vendor-neutral API contract standard; Fastify generates this natively. |
| **Deployment** | Docker Compose (self-hosted primary), Kubernetes Helm chart (enterprise) | README positions self-hosted as primary target. Docker Compose for SMB; Helm chart for enterprise Kubernetes deployments. |
| **Licence** | MIT | Research recommends MIT or Apache 2.0. MIT chosen for maximum permissiveness and alignment with Frappe/ERPNext ecosystem. |

### Data Model Decision

**Selected: Hybrid Relational + JSONB (Data Model Suggestion 3)** with selective elements from Suggestion 1 (normalized relational) and Suggestion 2 (event-sourced audit trail).

**Rationale:**
- Suggestion 3's hybrid approach provides the fastest path to MVP (~18 tables vs ~43 for fully normalized) while keeping relational lines for three-way matching JOINs.
- JSONB columns absorb jurisdiction-variable tax structures (EU VAT, US sales tax, Indian GST), e-invoicing metadata (PEPPOL, EDI, cXML), and custom fields without schema migrations.
- From Suggestion 1: adopt the normalized UNSPSC taxonomy (4 tables) rather than the flattened single table, because roll-up spend analytics queries benefit from the hierarchical FK structure.
- From Suggestion 2: adopt the append-only audit_log pattern with JSONB diffs and the outbox pattern for reliable event publication to the job queue. Not full event sourcing (too complex for MVP), but the audit trail is immutable and captures AI actor actions.
- Graph layer from Suggestion 4 deferred to Phase 10 as an analytics enhancement; not needed for core procurement workflows.

### Project Structure

```
procurement-automation/
  apps/
    web/                          # Next.js 15 frontend
      src/
        app/                      # App Router pages
          (auth)/                  # Login, SSO callback
          (dashboard)/             # Main authenticated shell
            requisitions/
            purchase-orders/
            goods-receipts/
            invoices/
            matching/
            suppliers/
            spend-analytics/
            sourcing/
            settings/
        components/
          ui/                     # shadcn/ui components
          procurement/            # Domain-specific components
        lib/
          api-client.ts           # Typed API client
          auth.ts                 # Auth.js config
    api/                          # Fastify API server
      src/
        routes/
          v1/
            requisitions/
            purchase-orders/
            goods-receipts/
            invoices/
            matching/
            suppliers/
            budgets/
            spend/
            sourcing/
            approvals/
            webhooks/
        services/                 # Business logic layer
        ai/                       # AI integration layer
          classifier.ts           # UNSPSC spend classification
          matcher.ts              # Three-way match AI resolution
          intake.ts               # Conversational intake
          risk-monitor.ts         # Supplier risk signal processing
        jobs/                     # BullMQ job processors
        db/
          schema/                 # Drizzle schema definitions
          migrations/             # SQL migration files
          seeds/                  # Reference data (UNSPSC, currencies)
        integrations/
          peppol/                 # PEPPOL Access Point client
          edi/                    # EDI X12/EDIFACT parser/generator
          ubl/                    # UBL 2.1 document builder/parser
          erp/                    # ERP connector abstraction
        middleware/
        utils/
  packages/
    shared/                       # Shared types, constants, validation schemas
    eslint-config/
    tsconfig/
  docker/
    docker-compose.yml
    docker-compose.prod.yml
    Dockerfile.api
    Dockerfile.web
  docs/
    api/                          # Generated OpenAPI docs
    architecture/
  tests/
    e2e/                          # Playwright E2E tests
    integration/                  # API integration tests
    unit/                         # Unit tests
  turbo.json                      # Turborepo config
  package.json
```

---

## Phase Dependency Graph

```
Phase 1: Foundation
    |
    v
Phase 2: Requisitions & Approvals
    |
    v
Phase 3: Purchase Orders --------+
    |                             |
    v                             v
Phase 4: Goods Receipt      Phase 5: AI Spend Classification
    |                             |
    +-------- + -----------------+
              |
              v
Phase 6: Invoices & Three-Way Matching
              |
              v
Phase 7: AI Match Resolution & Spend Analytics
              |
              v
Phase 8: Supplier Risk & Management ----+
              |                          |
              v                          v
Phase 9: Sourcing (RFQ/RFP)   Phase 10: E-Invoicing & EDI Compliance
              |                          |
              +-------- + --------------+
                        |
                        v
Phase 11: ERP Integrations & Chat Approvals
                        |
                        v
Phase 12: Production Hardening & Deployment
```

**Critical path:** Phases 1 -> 2 -> 3 -> 4 -> 6 -> 7 (requisition-to-payment core)
**AI track:** Phases 5, 7, 8 (can partially overlap with the critical path once Phase 3 is complete)
**Compliance track:** Phase 10 (can begin once Phase 6 is complete)

---

## Phase 1: Foundation & Infrastructure

**Goal:** Establish the monorepo, database schema, authentication, multi-tenancy, and API skeleton so that all subsequent phases build on a stable base.

**Duration estimate:** 3-4 weeks

### Task 1.1: Monorepo & Build Tooling

**What:** Initialize the Turborepo monorepo with `apps/web` (Next.js 15), `apps/api` (Fastify), and `packages/shared`. Configure TypeScript, ESLint, Prettier, and Husky pre-commit hooks.

**Design:**
```bash
# Root package.json with Turborepo
npx create-turbo@latest procurement-automation --example with-tailwind
```

```json
// turbo.json
{
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": [".next/**", "dist/**"] },
    "dev": { "persistent": true, "cache": false },
    "lint": {},
    "test": {},
    "db:migrate": { "cache": false }
  }
}
```

```typescript
// packages/shared/src/index.ts
export * from './types';
export * from './constants';
export * from './validation';
```

**Testing:**
- `turbo build` completes without errors for all workspaces
- `turbo dev` starts both Next.js (port 3000) and Fastify (port 4000) concurrently
- `turbo lint` passes with zero warnings
- `packages/shared` types are importable from both `apps/web` and `apps/api`
- Pre-commit hook runs lint and type-check on staged files

### Task 1.2: Database Schema & Migrations

**What:** Set up PostgreSQL via Docker, configure Drizzle ORM, and create the foundational migration covering tenant, organization, app_user, UNSPSC taxonomy tables (4-table hierarchy from Data Model 1), cost_centre, budget, gl_account, and audit_log.

**Design:**
```yaml
# docker/docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: procurement
      POSTGRES_USER: procurement
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

```typescript
// apps/api/src/db/schema/tenant.ts
import { pgTable, uuid, text, timestamptz, jsonb } from 'drizzle-orm/pg-core';

export const tenant = pgTable('tenant', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  subscriptionTier: text('subscription_tier').notNull().default('free'),
  config: jsonb('config').notNull().default({}),
  createdAt: timestamptz('created_at').notNull().defaultNow(),
  updatedAt: timestamptz('updated_at').notNull().defaultNow(),
});
```

```sql
-- Seed: UNSPSC reference data (subset for development)
INSERT INTO unspsc_segment (code, name) VALUES
  ('43', 'Information Technology Broadcasting and Telecommunications'),
  ('44', 'Office Equipment and Accessories and Supplies'),
  ('56', 'Furniture and Furnishings');
-- ... family, class, commodity tables populated similarly
```

**Testing:**
- `drizzle-kit push` applies schema to a clean database without errors
- `drizzle-kit generate` produces deterministic migration SQL files
- All foreign key relationships are enforced (inserting an organization with a non-existent tenant_id fails)
- UNSPSC seed data loads correctly; querying `unspsc_commodity` with joins to segment/family/class returns full taxonomy path
- Audit log INSERT succeeds with JSONB changes column
- `UNIQUE(tenant_id, email)` constraint on app_user prevents duplicate emails within a tenant

### Task 1.3: Authentication & Multi-Tenancy

**What:** Implement Auth.js v5 with credential-based login (MVP) and OIDC provider support (Azure AD, Google). Add tenant-scoped middleware that extracts tenant context from the authenticated session and injects it into all database queries.

**Design:**
```typescript
// apps/web/src/lib/auth.ts
import NextAuth from 'next-auth';
import Credentials from 'next-auth/providers/credentials';
import AzureAD from 'next-auth/providers/azure-ad';

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [
    Credentials({
      credentials: { email: {}, password: {} },
      authorize: async (credentials) => {
        // Verify against app_user table with bcrypt
      },
    }),
    AzureAD({
      clientId: process.env.AZURE_AD_CLIENT_ID,
      clientSecret: process.env.AZURE_AD_CLIENT_SECRET,
      tenantId: process.env.AZURE_AD_TENANT_ID,
    }),
  ],
  callbacks: {
    jwt({ token, user }) {
      if (user) {
        token.tenantId = user.tenantId;
        token.roles = user.roles;
      }
      return token;
    },
  },
});
```

```typescript
// apps/api/src/middleware/tenant.ts
import { FastifyRequest } from 'fastify';

export async function tenantMiddleware(req: FastifyRequest) {
  const tenantId = req.user.tenantId;
  if (!tenantId) throw new UnauthorizedError('No tenant context');
  req.tenantId = tenantId;
  // All subsequent DB queries MUST filter by req.tenantId
}
```

**Testing:**
- Login with valid credentials returns JWT with tenantId and roles
- Login with invalid credentials returns 401
- API requests without valid JWT return 401
- API requests with valid JWT but missing tenant context return 403
- User from tenant A cannot access data belonging to tenant B (cross-tenant isolation test)
- Azure AD OIDC flow completes and creates/links app_user record
- Session refresh works before JWT expiry

### Task 1.4: API Skeleton & OpenAPI

**What:** Configure Fastify with JSON Schema validation, CORS, rate limiting, request logging, and automatic OpenAPI 3.1 spec generation. Create health check and tenant info endpoints.

**Design:**
```typescript
// apps/api/src/server.ts
import Fastify from 'fastify';
import fastifySwagger from '@fastify/swagger';
import fastifySwaggerUI from '@fastify/swagger-ui';
import fastifyCors from '@fastify/cors';
import fastifyRateLimit from '@fastify/rate-limit';

const app = Fastify({ logger: true });

await app.register(fastifySwagger, {
  openapi: {
    info: { title: 'Procurement Automation API', version: '1.0.0' },
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
      },
    },
  },
});

await app.register(fastifyRateLimit, { max: 100, timeWindow: '1 minute' });

app.get('/health', { schema: { response: { 200: { type: 'object', properties: { status: { type: 'string' } } } } } },
  async () => ({ status: 'ok', timestamp: new Date().toISOString() })
);
```

**Testing:**
- `GET /health` returns `{ "status": "ok" }` with 200
- `GET /docs` serves Swagger UI with the OpenAPI spec
- Requests exceeding rate limit return 429
- CORS headers are present for allowed origins
- Invalid request bodies fail schema validation with 400 and descriptive error messages
- Request/response logging captures correlation IDs

### Task 1.5: Audit Log Service

**What:** Implement an append-only audit logging service that records all entity changes with actor, action, old/new values (JSONB), and timestamp. Integrate with Drizzle query hooks so audit entries are created automatically on INSERT/UPDATE/DELETE.

**Design:**
```typescript
// apps/api/src/services/audit.service.ts
interface AuditEntry {
  tenantId: string;
  entityType: 'purchase_requisition' | 'purchase_order' | 'invoice' | 'supplier' | 'payment';
  entityId: string;
  action: 'created' | 'updated' | 'deleted' | 'status_changed' | 'approved' | 'rejected' | 'matched';
  actorId: string | null;
  actorType: 'user' | 'system' | 'ai' | 'api';
  oldValues: Record<string, unknown> | null;
  newValues: Record<string, unknown> | null;
}

export class AuditService {
  async log(entry: AuditEntry): Promise<void> {
    await db.insert(auditLog).values({
      tenantId: entry.tenantId,
      entityType: entry.entityType,
      entityId: entry.entityId,
      action: entry.action,
      actorId: entry.actorId,
      actorType: entry.actorType,
      changes: { old: entry.oldValues, new: entry.newValues },
    });
  }

  async getHistory(entityType: string, entityId: string): Promise<AuditEntry[]> {
    return db.select().from(auditLog)
      .where(and(eq(auditLog.entityType, entityType), eq(auditLog.entityId, entityId)))
      .orderBy(desc(auditLog.createdAt));
  }
}
```

**Testing:**
- Creating a new entity produces an audit log entry with action='created' and newValues populated
- Updating an entity produces an audit log entry with action='updated', oldValues and newValues showing the diff
- Audit log entries cannot be updated or deleted (append-only enforced at service layer)
- `getHistory` returns entries in reverse chronological order
- AI-initiated actions are logged with actorType='ai' and null actorId
- Audit log query performance is acceptable with 100K+ entries (index on entity_type, entity_id)

### Definition of Done — Phase 1

- [ ] Monorepo builds and runs with `turbo dev`
- [ ] PostgreSQL schema applied with all foundational tables
- [ ] UNSPSC reference data seeded (at least 100 commodity codes for development)
- [ ] Authentication works with credentials and at least one OIDC provider
- [ ] Multi-tenant data isolation verified with automated tests
- [ ] OpenAPI spec generated and accessible at `/docs`
- [ ] Audit log captures all entity mutations
- [ ] Docker Compose starts the full stack (Postgres, Redis, API, Web) with one command
- [ ] CI pipeline runs lint, type-check, and tests on every PR
- [ ] Minimum 80% unit test coverage on service layer

---

## Phase 2: Purchase Requisitions & Approval Workflows

**Goal:** Implement the intake-to-approval flow: users create purchase requisitions (including natural-language intake), configure approval policies, and route requisitions through multi-level approval.

**Duration estimate:** 3-4 weeks

### Task 2.1: Approval Policy Configuration

**What:** Build CRUD API and settings UI for approval policies: define rules based on spend amount, category, cost centre, and supplier risk that determine the approval chain.

**Design:**
```typescript
// apps/api/src/db/schema/approval-policy.ts
export const approvalPolicy = pgTable('approval_policy', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  name: text('name').notNull(),
  description: text('description'),
  isActive: boolean('is_active').notNull().default(true),
  priority: integer('priority').notNull().default(0),
  rules: jsonb('rules').notNull().default([]),
  // rules example: [
  //   { "condition": { "field": "total_estimated", "operator": "gte", "value": 10000 },
  //     "approverRole": "cfo", "isMandatory": true, "stepOrder": 2 },
  //   { "condition": { "field": "total_estimated", "operator": "gte", "value": 1000 },
  //     "approverUserId": "uuid-...", "isMandatory": true, "stepOrder": 1 }
  // ]
  createdAt: timestamptz('created_at').notNull().defaultNow(),
  updatedAt: timestamptz('updated_at').notNull().defaultNow(),
});

// Approval engine
export class ApprovalEngine {
  async evaluatePolicy(requisition: PurchaseRequisition): Promise<ApprovalStep[]> {
    const policies = await this.getActivePolicies(requisition.tenantId);
    const steps: ApprovalStep[] = [];
    for (const policy of policies) {
      for (const rule of policy.rules) {
        if (this.evaluateCondition(rule.condition, requisition)) {
          steps.push({
            policyId: policy.id,
            stepOrder: rule.stepOrder,
            approverId: rule.approverUserId ?? await this.resolveRoleToUser(rule.approverRole, requisition),
            isMandatory: rule.isMandatory,
          });
        }
      }
    }
    return steps.sort((a, b) => a.stepOrder - b.stepOrder);
  }
}
```

**Testing:**
- CRUD operations on approval policies (create, read, update, deactivate)
- Policy with condition `total_estimated >= 10000` matches requisition with total 15000 and does not match total 5000
- Multiple policies evaluated in priority order; first match wins for each step
- Role-based approver resolution finds the correct user for the given role and organization
- Deactivated policies are not evaluated
- Invalid policy rules (unknown field, missing operator) rejected at creation time with 400

### Task 2.2: Purchase Requisition CRUD

**What:** Implement purchase_requisition and purchase_requisition_line tables, API routes, and form UI for creating, editing, and viewing requisitions. Include the `original_text` field for conversational intake (AI integration deferred to Phase 5 for classification, but the field is stored now).

**Design:**
```typescript
// apps/api/src/routes/v1/requisitions/create.ts
export const createRequisitionSchema = {
  body: {
    type: 'object',
    required: ['title', 'lines'],
    properties: {
      title: { type: 'string', minLength: 1, maxLength: 200 },
      description: { type: 'string' },
      originalText: { type: 'string' }, // natural language input
      costCentreId: { type: 'string', format: 'uuid' },
      neededByDate: { type: 'string', format: 'date' },
      priority: { type: 'string', enum: ['low', 'normal', 'high', 'urgent'] },
      lines: {
        type: 'array', minItems: 1,
        items: {
          type: 'object',
          required: ['description', 'quantity', 'unitOfMeasure'],
          properties: {
            description: { type: 'string' },
            quantity: { type: 'number', minimum: 0.0001 },
            unitOfMeasure: { type: 'string' },
            estimatedUnitPrice: { type: 'number', minimum: 0 },
            suggestedSupplierId: { type: 'string', format: 'uuid' },
          },
        },
      },
    },
  },
};
```

```tsx
// apps/web/src/app/(dashboard)/requisitions/new/page.tsx
// Multi-line form with dynamic line addition, cost centre picker,
// date picker, priority selector, and "Submit for Approval" button.
```

**Testing:**
- Create requisition with valid data returns 201 with requisition_number auto-generated (e.g., PR-2026-00001)
- Requisition number is unique per tenant and auto-increments
- Creating requisition with zero lines returns 400
- Lines with quantity <= 0 rejected with validation error
- `total_estimated` computed as sum of (quantity * estimatedUnitPrice) for all lines
- Currency defaults to organization's default_currency
- Requisition created with status='draft'
- GET /requisitions returns paginated list filtered by tenant_id
- GET /requisitions/:id returns full requisition with lines
- PATCH /requisitions/:id updates allowed fields only while status='draft'
- DELETE /requisitions/:id soft-deletes (status='cancelled') only while status='draft'
- Audit log entry created for each mutation

### Task 2.3: Approval Workflow Execution

**What:** When a requisition is submitted for approval, the approval engine evaluates applicable policies, creates approval_request records for each step, and notifies approvers. Approvers can approve, reject, or delegate. Status transitions: draft -> pending_approval -> approved | rejected.

**Design:**
```typescript
// apps/api/src/services/approval.service.ts
export class ApprovalService {
  async submitForApproval(requisitionId: string, submitterId: string): Promise<void> {
    const requisition = await this.getRequisition(requisitionId);
    if (requisition.status !== 'draft') throw new ConflictError('Only draft requisitions can be submitted');

    const steps = await this.approvalEngine.evaluatePolicy(requisition);
    if (steps.length === 0) {
      // No approval required — auto-approve
      await this.updateStatus(requisitionId, 'approved');
      return;
    }

    await db.transaction(async (tx) => {
      await tx.update(purchaseRequisition)
        .set({ status: 'pending_approval' })
        .where(eq(purchaseRequisition.id, requisitionId));

      for (const step of steps) {
        await tx.insert(approvalRequest).values({
          tenantId: requisition.tenantId,
          entityType: 'purchase_requisition',
          entityId: requisitionId,
          policyId: step.policyId,
          stepOrder: step.stepOrder,
          approverId: step.approverId,
          status: step.stepOrder === 1 ? 'pending' : 'waiting',
          isMandatory: step.isMandatory,
        });
      }
    });

    // Notify first-step approver(s)
    await this.notificationService.notifyApprover(steps[0].approverId, requisition);
  }

  async processDecision(approvalRequestId: string, decision: 'approved' | 'rejected', note: string): Promise<void> {
    // Update current step, advance to next step or finalize
  }
}
```

**Testing:**
- Submitting a draft requisition changes status to 'pending_approval' and creates approval_request records
- Submitting a non-draft requisition returns 409 Conflict
- Requisition below all policy thresholds is auto-approved (no approval steps created)
- Approving step 1 advances the workflow to step 2 (step 2 status changes from 'waiting' to 'pending')
- Approving the final step changes requisition status to 'approved'
- Rejecting any mandatory step changes requisition status to 'rejected' and cancels remaining steps
- Delegation creates a new approval_request for the delegate and marks the original as 'delegated'
- Concurrent approval of the same step by two approvers is handled idempotently (only first is recorded)
- Notification is sent to each approver when their step becomes active
- Audit log records each approval decision with approver ID and note
- GET /approvals/pending?approverId=X returns all pending approvals for a user across entity types

### Task 2.4: Requisition Dashboard UI

**What:** Build the requisition list view (filterable by status, date range, requester) and detail view (showing lines, approval timeline, and action buttons). Include the approval inbox for approvers.

**Design:**
```tsx
// apps/web/src/app/(dashboard)/requisitions/page.tsx
// Server component that fetches requisitions with status filters
// Columns: Number, Title, Requester, Total, Status, Date, Priority

// apps/web/src/components/procurement/ApprovalTimeline.tsx
// Visual timeline showing each approval step with status badges:
// [Step 1: Jane Doe - Approved ✓] -> [Step 2: CFO - Pending ⏳] -> [Step 3: VP - Waiting]
```

**Testing:**
- Requisition list loads with correct data for the authenticated tenant
- Status filter works (draft, pending_approval, approved, rejected)
- Pagination works with 100+ requisitions
- Detail view shows all lines with calculated totals
- Approval timeline accurately reflects current step status
- "Approve" and "Reject" buttons visible only to the current-step approver
- Approve/Reject actions update the UI in real-time (optimistic update with rollback on error)
- Mobile-responsive layout for approval inbox (approvers on mobile)

### Definition of Done — Phase 2

- [ ] Approval policies configurable via settings UI
- [ ] Requisitions created, submitted, approved, and rejected through the full lifecycle
- [ ] Multi-step approval workflow executes correctly with step ordering
- [ ] Approval delegation works
- [ ] Notification stubs in place (email integration deferred)
- [ ] Requisition dashboard with filtering and pagination
- [ ] Approval inbox shows pending approvals across entity types
- [ ] All status transitions audited
- [ ] API tests cover all status transition edge cases
- [ ] E2E test: create requisition -> submit -> approve all steps -> status is approved

---

## Phase 3: Purchase Orders

**Goal:** Convert approved requisitions into purchase orders, manage PO lifecycle (draft -> sent -> acknowledged), and deliver POs to suppliers via email.

**Duration estimate:** 2-3 weeks

### Task 3.1: Purchase Order Schema & CRUD

**What:** Implement purchase_order and purchase_order_line tables with all fields from Data Model 3 (including transmission_data and tax_details JSONB columns). Build API routes for creating POs from approved requisitions and for manual PO creation.

**Design:**
```typescript
// apps/api/src/services/purchase-order.service.ts
export class PurchaseOrderService {
  async createFromRequisition(requisitionId: string, supplierId: string, userId: string): Promise<PurchaseOrder> {
    const requisition = await this.getApprovedRequisition(requisitionId);
    const supplier = await this.getActiveSupplier(supplierId);

    const poNumber = await this.generatePoNumber(requisition.tenantId);

    return db.transaction(async (tx) => {
      const [po] = await tx.insert(purchaseOrder).values({
        tenantId: requisition.tenantId,
        organizationId: requisition.organizationId,
        poNumber,
        requisitionId,
        supplierId,
        status: 'draft',
        currency: requisition.currency,
        paymentTermsDays: supplier.paymentTermsDays,
        costCentreId: requisition.costCentreId,
        createdBy: userId,
      }).returning();

      // Copy requisition lines to PO lines
      for (const line of requisition.lines) {
        await tx.insert(purchaseOrderLine).values({
          purchaseOrderId: po.id,
          lineNumber: line.lineNumber,
          requisitionLineId: line.id,
          itemId: line.itemId,
          description: line.description,
          quantityOrdered: line.quantity,
          unitOfMeasure: line.unitOfMeasure,
          unitPrice: line.estimatedUnitPrice,
          lineTotal: line.quantity * line.estimatedUnitPrice,
          unspscCode: line.unspscCode,
          costCentreId: line.costCentreId ?? requisition.costCentreId,
        });
      }

      // Update PO totals
      await this.recalculateTotals(tx, po.id);

      // Update requisition status
      await tx.update(purchaseRequisition)
        .set({ status: 'converted' })
        .where(eq(purchaseRequisition.id, requisitionId));

      // Commit budget (increase committed amount on cost centre budget)
      await this.budgetService.commitSpend(tx, po.costCentreId, po.totalAmount, po.currency);

      return po;
    });
  }
}
```

**Testing:**
- PO created from approved requisition with correct line mapping
- PO number auto-generated in format PO-{year}-{sequence} (e.g., PO-2026-00001)
- Requisition status changes to 'converted' after PO creation
- Creating PO from non-approved requisition returns 400
- Budget committed amount increases by PO total
- Budget threshold alert triggered when committed + spent exceeds alert_threshold_pct
- Manual PO creation (no requisition) works with all required fields
- PO totals (subtotal, tax, total) recalculated correctly on line changes
- GET /purchase-orders returns paginated list with supplier name
- GET /purchase-orders/:id returns full PO with lines
- PATCH updates allowed while status='draft'

### Task 3.2: PO Email Delivery

**What:** Generate PDF purchase orders and send them to the supplier's primary contact email. Track sent/acknowledged status.

**Design:**
```typescript
// apps/api/src/services/po-delivery.service.ts
export class PODeliveryService {
  async sendToSupplier(poId: string): Promise<void> {
    const po = await this.getPOWithDetails(poId);
    if (po.status !== 'approved' && po.status !== 'draft') {
      throw new ConflictError('PO must be approved before sending');
    }

    const pdfBuffer = await this.pdfGenerator.generatePO(po);
    const supplierContact = po.supplier.contacts.find(c => c.isPrimary);

    await this.emailService.send({
      to: supplierContact.email,
      subject: `Purchase Order ${po.poNumber} from ${po.organization.name}`,
      template: 'purchase-order',
      context: { po, acknowledgeUrl: `${this.baseUrl}/supplier/acknowledge/${po.id}` },
      attachments: [{ filename: `${po.poNumber}.pdf`, content: pdfBuffer }],
    });

    await db.update(purchaseOrder).set({
      status: 'sent',
      sentAt: new Date(),
      transmissionData: { channel: 'email', sentTo: supplierContact.email },
    }).where(eq(purchaseOrder.id, poId));
  }
}
```

**Testing:**
- PDF generation produces a valid PDF with correct PO data (number, lines, totals, supplier info)
- Email sent to supplier's primary contact with PDF attachment
- PO status changes to 'sent' and sentAt is populated
- Acknowledge URL in email links to a public page where supplier can confirm receipt
- Supplier acknowledgement updates PO status to 'acknowledged' and acknowledgedAt is populated
- Sending a PO that has already been sent returns 409
- Email delivery failure is retried via job queue (BullMQ)
- Audit log records the send event with transmission details

### Task 3.3: Purchase Order List & Detail UI

**What:** Build PO list view with status badges, supplier filter, date range filter, and detail view showing lines, delivery status, and budget impact.

**Design:**
```tsx
// apps/web/src/app/(dashboard)/purchase-orders/page.tsx
// Table with columns: PO#, Supplier, Total, Status, Order Date, Delivery Status
// Action buttons: "Send to Supplier", "Print PDF", "Cancel"

// apps/web/src/app/(dashboard)/purchase-orders/[id]/page.tsx
// Header: PO number, status badge, supplier info, dates
// Lines table: item, qty ordered/received/invoiced, unit price, total
// Budget section: cost centre, committed amount, remaining budget
// Timeline: created -> approved -> sent -> acknowledged -> received -> invoiced -> closed
```

**Testing:**
- PO list renders with correct status badges (color-coded)
- Supplier filter dropdown populated from vendor master
- "Send to Supplier" button visible only for approved/draft POs
- Detail view shows quantity received and invoiced columns (zero at this phase)
- PDF preview in modal before sending
- Budget impact section shows committed vs remaining accurately
- Timeline component shows completed and upcoming steps

### Definition of Done — Phase 3

- [ ] POs created from approved requisitions and manually
- [ ] PO numbers auto-generated and unique per tenant
- [ ] Budget commitment tracking works
- [ ] PDF generation produces professional purchase order documents
- [ ] Email delivery to suppliers with PDF attachment
- [ ] Supplier acknowledgement via public URL
- [ ] PO lifecycle: draft -> approved -> sent -> acknowledged
- [ ] PO list and detail views with filtering
- [ ] E2E test: create requisition -> approve -> create PO -> send -> supplier acknowledges

---

## Phase 4: Goods Receipt

**Goal:** Record the physical receipt of goods against purchase orders, tracking quantities received, accepted, and rejected per line. Update PO line quantities for three-way matching readiness.

**Duration estimate:** 2 weeks

### Task 4.1: Goods Receipt Schema & CRUD

**What:** Implement goods_receipt and goods_receipt_line tables. Build API routes for creating GRNs linked to POs, recording line-level quantities, and confirming receipts.

**Design:**
```typescript
// apps/api/src/services/goods-receipt.service.ts
export class GoodsReceiptService {
  async create(input: CreateGRNInput, userId: string): Promise<GoodsReceipt> {
    const po = await this.getPO(input.purchaseOrderId);
    if (!['sent', 'acknowledged', 'partially_received'].includes(po.status)) {
      throw new ConflictError('PO is not in a receivable state');
    }

    return db.transaction(async (tx) => {
      const grnNumber = await this.generateGrnNumber(po.tenantId);
      const [grn] = await tx.insert(goodsReceipt).values({
        tenantId: po.tenantId,
        grnNumber,
        purchaseOrderId: po.id,
        supplierId: po.supplierId,
        receivedBy: userId,
        status: 'confirmed',
        deliveryData: input.deliveryData ?? {},
      }).returning();

      for (const line of input.lines) {
        const poLine = await this.getPOLine(line.poLineId);
        const remaining = poLine.quantityOrdered - poLine.quantityReceived;
        if (line.quantityReceived > remaining) {
          throw new ValidationError(`Line ${poLine.lineNumber}: receiving ${line.quantityReceived} but only ${remaining} remaining`);
        }

        await tx.insert(goodsReceiptLine).values({
          goodsReceiptId: grn.id,
          lineNumber: line.lineNumber,
          poLineId: line.poLineId,
          quantityReceived: line.quantityReceived,
          quantityAccepted: line.quantityAccepted,
          quantityRejected: line.quantityRejected,
          rejectionReason: line.rejectionReason,
          condition: line.condition ?? 'good',
          inspectionData: line.inspectionData,
        });

        // Update PO line denormalized counter
        await tx.update(purchaseOrderLine)
          .set({ quantityReceived: sql`quantity_received + ${line.quantityAccepted}` })
          .where(eq(purchaseOrderLine.id, line.poLineId));
      }

      // Update PO status based on receipt completeness
      await this.updatePOReceiptStatus(tx, po.id);

      return grn;
    });
  }
}
```

**Testing:**
- GRN created with valid data and linked to PO
- GRN number auto-generated in format GRN-{year}-{sequence}
- Receiving more than the remaining quantity on a PO line returns 400
- PO line quantity_received updated correctly after GRN confirmation
- PO status changes to 'partially_received' after partial receipt
- PO status changes to 'received' when all lines are fully received
- Multiple GRNs against the same PO (partial receipts) accumulate correctly
- Rejected quantity tracked separately from accepted
- Delivery data (carrier, tracking number, SSCC codes) stored in JSONB
- Audit log records GRN creation and PO status changes
- GET /goods-receipts?purchaseOrderId=X returns all GRNs for a PO

### Task 4.2: Goods Receipt UI

**What:** Build the GRN creation form (pre-populated from PO lines with remaining quantities), GRN list view, and GRN detail view.

**Design:**
```tsx
// apps/web/src/app/(dashboard)/goods-receipts/new/page.tsx
// Step 1: Select PO (dropdown of POs in receivable state)
// Step 2: Pre-populated lines showing ordered qty, already received, and remaining
//         User enters received qty, accepted qty, rejected qty per line
// Step 3: Optional delivery details (carrier, tracking, notes)
// Step 4: Review and confirm

// Visual indicator showing PO fulfillment progress (e.g., 30/50 units received)
```

**Testing:**
- PO selector shows only POs in receivable state (sent, acknowledged, partially_received)
- Lines pre-populated with correct remaining quantities
- User cannot enter received qty exceeding remaining
- Rejected qty auto-calculates as received - accepted
- Rejection reason required when quantityRejected > 0
- Submit creates GRN and updates PO status in the UI immediately
- GRN detail view shows received quantities alongside PO ordered quantities

### Definition of Done — Phase 4

- [ ] Goods receipts recorded against purchase orders
- [ ] Line-level quantity tracking (received, accepted, rejected)
- [ ] PO status transitions: partially_received, received
- [ ] PO line denormalized counters updated atomically
- [ ] Over-receipt prevented at validation layer
- [ ] GRN list and detail views functional
- [ ] E2E test: create PO -> partial GRN -> second GRN -> PO fully received

---

## Phase 5: AI Spend Classification

**Goal:** Implement LLM-based automatic classification of line-item descriptions against the UNSPSC taxonomy. This is the first AI-native feature and a key differentiator identified in the research.

**Duration estimate:** 2-3 weeks

### Task 5.1: UNSPSC Classification Service

**What:** Build a service that takes a line-item text description and returns the best-matching UNSPSC commodity code with confidence score and alternatives. Uses the Anthropic Claude API with a curated system prompt and few-shot examples. Includes a feedback loop for human corrections.

**Design:**
```typescript
// apps/api/src/ai/classifier.ts
import Anthropic from '@anthropic-ai/sdk';

export class SpendClassifier {
  private client: Anthropic;

  async classify(description: string, tenantId: string): Promise<ClassificationResult> {
    // Fetch recent correction feedback for this tenant to improve accuracy
    const corrections = await this.getRecentCorrections(tenantId, 20);

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 500,
      system: `You are a procurement spend classification expert. Given a line-item description, classify it against the UNSPSC (United Nations Standard Products and Services Code) taxonomy. Return the most likely 8-digit UNSPSC commodity code, the full taxonomy path (segment > family > class > commodity), a confidence score between 0 and 1, and up to 3 alternative codes with their confidence scores.

Respond in JSON format only:
{
  "code": "56101504",
  "path": "Furniture > Office Furniture > Seating > Task and operator seating",
  "confidence": 0.95,
  "alternatives": [
    {"code": "56101505", "path": "... > Executive seating", "confidence": 0.03},
    {"code": "56101503", "path": "... > Side chairs", "confidence": 0.01}
  ]
}`,
      messages: [
        // Include few-shot examples from tenant corrections
        ...corrections.map(c => ([
          { role: 'user' as const, content: c.descriptionText },
          { role: 'assistant' as const, content: JSON.stringify({ code: c.correctedUnspsc, path: c.path, confidence: 1.0, alternatives: [] }) },
        ])).flat(),
        { role: 'user', content: description },
      ],
    });

    return this.parseClassificationResponse(response);
  }

  async classifyBatch(items: { id: string; description: string }[], tenantId: string): Promise<Map<string, ClassificationResult>> {
    // Process in parallel with rate limiting
    const results = new Map();
    const chunks = this.chunk(items, 10);
    for (const chunk of chunks) {
      const promises = chunk.map(item => this.classify(item.description, tenantId));
      const classified = await Promise.allSettled(promises);
      classified.forEach((result, i) => {
        if (result.status === 'fulfilled') results.set(chunk[i].id, result.value);
      });
    }
    return results;
  }
}
```

```typescript
// apps/api/src/services/classification-feedback.service.ts
export class ClassificationFeedbackService {
  async recordCorrection(input: {
    tenantId: string;
    entityType: 'purchase_requisition_line' | 'invoice_line' | 'item';
    entityId: string;
    originalUnspsc: string;
    correctedUnspsc: string;
    correctedBy: string;
    descriptionText: string;
  }): Promise<void> {
    await db.insert(spendClassificationFeedback).values(input);
    // Update the entity's UNSPSC code
    // This correction will be used as few-shot example in future classifications
  }
}
```

**Testing:**
- Classification of "Ergonomic office chair, adjustable height" returns UNSPSC code in the 56101504 family with confidence > 0.8
- Classification of "Dell Latitude 7440 laptop" returns UNSPSC code in the 43211500 family
- Classification of ambiguous text "supplies for the office" returns lower confidence (<0.7) with multiple alternatives
- Batch classification processes 50 items without exceeding API rate limits
- Human correction stored in feedback table and influences subsequent classifications for the same tenant
- Classification result stored as classificationData JSONB on the line item
- Job queue processes classification asynchronously for newly created requisition lines
- Fallback to lower confidence threshold when API is unavailable (graceful degradation)
- API cost tracking: each classification call logged with token count for cost monitoring

### Task 5.2: Classification Integration & UI

**What:** Integrate the classifier into the requisition and PO creation flows. Add UI elements showing classification results with confidence indicators and a correction mechanism.

**Design:**
```tsx
// apps/web/src/components/procurement/UnspscBadge.tsx
// Shows classified category with confidence color:
//   Green (>0.9): "Office Furniture > Seating > Task seating (98%)"
//   Yellow (0.7-0.9): "IT Equipment > Laptops (82%) [Review]"
//   Red (<0.7): "Uncategorized [Classify]"
// Clicking opens a correction modal with UNSPSC tree browser
```

**Testing:**
- New requisition lines are auto-classified after creation (async job)
- Classification badge shows on requisition line items within 5 seconds of creation
- Confidence indicator color-coded correctly (green/yellow/red)
- UNSPSC correction modal shows searchable taxonomy tree
- Submitting a correction updates the line item and creates a feedback record
- Spend analytics dashboard can group by UNSPSC segment/family (preparatory for Phase 7)

### Definition of Done — Phase 5

- [ ] UNSPSC classification service returns accurate results for common procurement items
- [ ] Batch classification processes requisition lines asynchronously
- [ ] Human correction feedback loop implemented and used in future classifications
- [ ] Classification results visible in requisition and PO detail views
- [ ] API cost tracking for LLM calls in place
- [ ] Graceful degradation when AI service is unavailable
- [ ] Classification accuracy > 80% on a test set of 100 common procurement items

---

## Phase 6: Invoices & Three-Way Matching

**Goal:** Receive vendor invoices, perform three-way matching against PO and GRN data, and manage the exception queue. This is the core AP automation feature.

**Duration estimate:** 3-4 weeks

### Task 6.1: Invoice Schema & Ingestion

**What:** Implement invoice and invoice_line tables. Build API routes for manual invoice entry and email-based invoice ingestion (PDF parsing deferred to Phase 7 for AI-powered OCR).

**Design:**
```typescript
// apps/api/src/db/schema/invoice.ts
// As per Data Model 3: invoice with source_data JSONB, tax_details JSONB,
// payment_data JSONB, and relational invoice_line for three-way matching.

// apps/api/src/services/invoice.service.ts
export class InvoiceService {
  async create(input: CreateInvoiceInput): Promise<Invoice> {
    // Validate supplier exists and is active
    // Auto-match to PO if invoice references a PO number
    // Generate internal_ref (INV-{year}-{sequence})
    // Create invoice and lines
    // Queue for three-way matching
  }
}
```

**Testing:**
- Invoice created with valid data and internal reference generated
- Invoice linked to PO via supplier + PO number reference
- Invoice without PO reference created with match_status='no_po'
- Duplicate invoice detection: same tenant + supplier + invoice_number returns 409
- Invoice lines created with FK to PO lines when PO is referenced
- Invoice totals validated against sum of line totals
- GET /invoices returns paginated list with match_status filter
- GET /invoices/:id returns full invoice with lines and match results

### Task 6.2: Three-Way Matching Engine

**What:** Implement the matching engine that compares each invoice line against the corresponding PO line (price, quantity) and GRN line (quantity received). Apply configurable tolerance rules and classify each line as matched, variance, or exception.

**Design:**
```typescript
// apps/api/src/services/matching.service.ts
export class MatchingService {
  async matchInvoice(invoiceId: string): Promise<MatchResult> {
    const invoice = await this.getInvoiceWithLines(invoiceId);
    if (!invoice.purchaseOrderId) {
      await this.updateMatchStatus(invoiceId, 'no_po');
      return { status: 'no_po', exceptions: [] };
    }

    const po = await this.getPOWithLines(invoice.purchaseOrderId);
    const grns = await this.getGRNLinesForPO(invoice.purchaseOrderId);
    const tolerances = await this.getToleranceRules(invoice.tenantId);

    const exceptions: MatchException[] = [];
    let allMatched = true;

    for (const invLine of invoice.lines) {
      const poLine = po.lines.find(l => l.id === invLine.poLineId);
      if (!poLine) {
        exceptions.push({ type: 'no_po_line', lineNumber: invLine.lineNumber, severity: 'high' });
        allMatched = false;
        continue;
      }

      const grnLine = grns.find(g => g.poLineId === poLine.id);
      if (!grnLine) {
        exceptions.push({ type: 'no_grn', lineNumber: invLine.lineNumber, severity: 'high' });
        allMatched = false;
        continue;
      }

      // Price variance check
      const priceVariance = invLine.unitPrice - poLine.unitPrice;
      const priceVariancePct = Math.abs(priceVariance / poLine.unitPrice) * 100;
      const priceTolerance = this.findApplicableTolerance(tolerances, 'price', invLine.unspscCode);

      // Quantity variance check
      const qtyVariance = invLine.quantity - grnLine.quantityAccepted;

      let lineMatchStatus: string;
      const matchResult: MatchResultData = {
        priceVariance,
        quantityVariance: qtyVariance,
        variancePct: priceVariancePct,
        toleranceRule: priceTolerance?.name,
      };

      if (priceVariancePct <= (priceTolerance?.toleranceValue ?? 0) && qtyVariance === 0) {
        lineMatchStatus = 'matched';
        matchResult.status = 'matched';
        matchResult.autoApproved = priceTolerance?.autoApprove ?? true;
      } else {
        lineMatchStatus = priceVariancePct > (priceTolerance?.toleranceValue ?? 0) ? 'price_variance' : 'quantity_variance';
        matchResult.status = 'exception';
        matchResult.exceptionType = lineMatchStatus;
        exceptions.push({
          type: lineMatchStatus,
          lineNumber: invLine.lineNumber,
          severity: priceVariancePct > 10 ? 'high' : 'medium',
          priceVariance,
          quantityVariance: qtyVariance,
        });
        allMatched = false;
      }

      // Update invoice line match_result JSONB
      await db.update(invoiceLine).set({ matchResult }).where(eq(invoiceLine.id, invLine.id));

      // Update PO line denormalized counter
      await db.update(purchaseOrderLine)
        .set({ quantityInvoiced: sql`quantity_invoiced + ${invLine.quantity}` })
        .where(eq(purchaseOrderLine.id, poLine.id));
    }

    const overallStatus = allMatched ? 'auto_matched' : 'exception';
    const matchScore = 1 - (exceptions.length / invoice.lines.length);
    await db.update(invoiceTable).set({
      matchStatus: overallStatus,
      matchScore,
      matchedAt: allMatched ? new Date() : null,
      status: allMatched ? 'matched' : 'exception',
    }).where(eq(invoiceTable.id, invoiceId));

    return { status: overallStatus, matchScore, exceptions };
  }
}
```

**Testing:**
- Perfect match (PO qty = GRN qty = invoice qty, PO price = invoice price) results in match_status='auto_matched'
- Price variance within tolerance (e.g., 2%) results in matched with auto_approve=true
- Price variance exceeding tolerance creates an exception with severity proportional to variance
- Quantity mismatch (invoiced qty != received qty) creates a quantity_variance exception
- Invoice line with no corresponding PO line creates a no_po_line exception
- Invoice line with no GRN (goods not yet received) creates a no_grn exception
- Match score calculated as (matched_lines / total_lines)
- PO line quantity_invoiced updated after matching
- Multiple invoices against the same PO (partial invoicing) accumulate correctly
- Tolerance rules scoped by UNSPSC category apply correctly (e.g., tighter tolerance for IT equipment)
- Default tolerance rule applied when no category-specific rule exists

### Task 6.3: Match Tolerance Configuration UI

**What:** Build a settings page for configuring match tolerance rules: price tolerance (percentage or absolute), quantity tolerance, auto-approve thresholds, and category-specific overrides.

**Design:**
```tsx
// apps/web/src/app/(dashboard)/settings/matching/page.tsx
// Table of tolerance rules with columns: Name, Type (price/quantity/total),
// Tolerance (2% or $5.00), Auto-Approve, Category, Priority
// Add/Edit/Delete actions
// Preview panel: "With these rules, a $100 PO line would auto-approve invoices between $98-$102"
```

**Testing:**
- CRUD operations on tolerance rules
- Rules ordered by priority (highest priority evaluated first)
- Category-specific rules override default rules
- Preview calculation shows correct tolerance range
- Deleting the last rule leaves the default (0% tolerance, no auto-approve) in effect

### Task 6.4: Exception Queue UI

**What:** Build the exception management dashboard showing invoices with unresolved match exceptions. Each exception shows the variance details, PO/GRN reference data, and action buttons (approve, reject, adjust, escalate).

**Design:**
```tsx
// apps/web/src/app/(dashboard)/matching/page.tsx
// Exception queue table:
// - Invoice number, supplier, total, exception count, oldest exception date
// - Expandable row showing individual line exceptions with:
//   - PO price vs invoice price, PO qty vs received qty vs invoiced qty
//   - Variance amount and percentage
//   - Action buttons: Approve (accept variance), Reject (reject invoice line),
//     Adjust (edit invoice line amount), Escalate (route to senior AP)

// Summary cards at top: Total exceptions, Avg resolution time, Auto-resolved %
```

**Testing:**
- Exception queue shows only invoices with unresolved exceptions
- Exception severity badges (low/medium/high/critical) display correctly
- Approve action resolves the exception and updates match_status
- Reject action creates a credit note request (stub for now)
- Approving all exceptions on an invoice changes invoice status to 'matched'
- Summary cards show accurate metrics
- Pagination and sorting by severity, date, amount

### Definition of Done — Phase 6

- [ ] Invoices created manually with validation
- [ ] Three-way matching engine processes invoice against PO and GRN
- [ ] Tolerance rules configurable per tenant with category overrides
- [ ] Exception queue shows unresolved mismatches
- [ ] Manual exception resolution (approve, reject, adjust) works
- [ ] PO line quantity_invoiced tracking accurate
- [ ] Invoice lifecycle: received -> matched/exception -> approved
- [ ] Duplicate invoice detection prevents double-payment
- [ ] E2E test: create PO -> GRN -> invoice -> auto-match (within tolerance)
- [ ] E2E test: create PO -> GRN -> invoice with price over tolerance -> exception -> manual resolve

---

## Phase 7: AI Match Resolution & Spend Analytics

**Goal:** Add AI-powered autonomous resolution of match exceptions and build the spend analytics dashboard. These are the two headline AI features.

**Duration estimate:** 3-4 weeks

### Task 7.1: AI Match Exception Resolution

**What:** When a three-way match exception is raised, an AI agent analyzes the PO, GRN, invoice, supplier history, and tolerance context to propose a resolution with confidence and reasoning. High-confidence resolutions (>0.9) can be auto-applied; lower-confidence ones queue for human review.

**Design:**
```typescript
// apps/api/src/ai/matcher.ts
export class AIMatchResolver {
  async resolveException(exception: MatchException): Promise<AIResolution> {
    const context = await this.gatherContext(exception);
    // context includes: PO details, GRN details, invoice details,
    // supplier's historical variance patterns, tolerance rules,
    // previous exception resolutions for this supplier

    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1000,
      system: `You are an accounts payable expert analyzing a three-way match exception in a procurement system. Given the context of a purchase order, goods receipt, and vendor invoice, determine whether the variance should be approved, rejected, or adjusted.

Consider:
1. Is the variance within the supplier's historical pattern?
2. Is there a reasonable business explanation (rush delivery, market price change)?
3. What is the financial risk of approving vs rejecting?

Respond in JSON:
{
  "resolution": "approve" | "reject" | "adjust",
  "confidence": 0.0-1.0,
  "reasoning": "Explanation of the decision",
  "adjustedAmount": null | number (only if resolution is "adjust")
}`,
      messages: [{ role: 'user', content: JSON.stringify(context) }],
    });

    const result = this.parseResolution(response);

    // Store AI resolution on the match result
    await this.updateMatchResult(exception.invoiceLineId, {
      aiResolution: result.resolution,
      aiConfidence: result.confidence,
      aiReasoning: result.reasoning,
    });

    // Auto-apply if confidence exceeds threshold
    if (result.confidence >= 0.9 && result.resolution === 'approve') {
      await this.autoApplyResolution(exception, result);
      await this.auditService.log({
        entityType: 'invoice',
        entityId: exception.invoiceId,
        action: 'matched',
        actorType: 'ai',
        newValues: { resolution: result.resolution, confidence: result.confidence },
      });
    }

    return result;
  }
}
```

**Testing:**
- AI resolver called automatically when a match exception is created (via job queue)
- Resolution with confidence >= 0.9 and resolution='approve' is auto-applied
- Resolution with confidence < 0.9 added to exception queue for human review with AI suggestion displayed
- AI reasoning text is stored and displayed in the exception detail view
- Auto-applied resolutions audited with actorType='ai'
- Historical supplier variance patterns improve resolution accuracy over time
- Timeout handling: if AI call takes >30s, exception queued for human review without AI suggestion
- Metrics tracked: auto-resolution rate, accuracy of auto-resolutions (measured by human override rate)

### Task 7.2: Spend Analytics Dashboard

**What:** Build the spend analytics dashboard with interactive visualizations: spend by supplier, by category (UNSPSC), by cost centre, by time period, with budget-vs-actual comparison.

**Design:**
```typescript
// apps/api/src/routes/v1/spend/analytics.ts
export const spendAnalyticsSchema = {
  querystring: {
    type: 'object',
    properties: {
      fiscalYear: { type: 'integer' },
      groupBy: { type: 'string', enum: ['supplier', 'unspsc_segment', 'unspsc_family', 'cost_centre', 'month'] },
      supplierId: { type: 'string', format: 'uuid' },
      unspscSegment: { type: 'string', minLength: 2, maxLength: 2 },
      costCentreId: { type: 'string', format: 'uuid' },
    },
  },
};

// Spend query using UNSPSC hierarchy (normalized 4-table model pays off here)
// SELECT
//   s.code AS segment_code, s.name AS segment_name,
//   SUM(il.line_total) AS total_spend,
//   COUNT(DISTINCT i.id) AS invoice_count
// FROM invoice_line il
// JOIN invoice i ON i.id = il.invoice_id
// JOIN unspsc_commodity c ON c.code = il.unspsc_code
// JOIN unspsc_class cl ON cl.code = c.class_code
// JOIN unspsc_family f ON f.code = cl.family_code
// JOIN unspsc_segment s ON s.code = f.segment_code
// WHERE i.tenant_id = $1 AND i.status = 'matched'
// GROUP BY s.code, s.name
// ORDER BY total_spend DESC;
```

```tsx
// apps/web/src/app/(dashboard)/spend-analytics/page.tsx
// Dashboard layout:
// Row 1: KPI cards — Total Spend YTD, Budget Utilization %, Top Supplier, Auto-Match Rate
// Row 2: Spend by category (treemap/sunburst chart using UNSPSC hierarchy)
// Row 3: Spend over time (bar chart by month), Budget vs Actual (line chart)
// Row 4: Top 10 suppliers by spend (horizontal bar), Supplier concentration risk
// Drill-down: click any segment to filter dashboard to that category
```

**Testing:**
- Analytics endpoint returns correct aggregations for each groupBy option
- Spend totals match sum of matched invoice line totals (consistency check)
- UNSPSC hierarchy drill-down: segment -> family -> class -> commodity
- Budget vs actual comparison shows committed (PO) and spent (invoice) amounts against budget
- Date range filter works correctly across fiscal year boundaries
- Spend by supplier identifies concentration risk (single supplier > 30% of category spend)
- Dashboard renders within 2 seconds with 10K+ invoices (query performance test)
- Charts interactive with hover tooltips showing amounts and percentages
- Export to CSV for all analytics views

### Task 7.3: Duplicate Invoice Detection

**What:** Implement AI-powered duplicate invoice detection that flags invoices with similar amounts, dates, and suppliers, going beyond the exact-match detection in Phase 6.

**Design:**
```typescript
// apps/api/src/services/duplicate-detection.service.ts
export class DuplicateDetectionService {
  async checkForDuplicates(invoice: Invoice): Promise<DuplicateCandidate[]> {
    // Step 1: Exact match (already handled by unique constraint)
    // Step 2: Fuzzy match — same supplier, similar amount (within 1%), similar date (within 7 days)
    const candidates = await db.select().from(invoiceTable)
      .where(and(
        eq(invoiceTable.tenantId, invoice.tenantId),
        eq(invoiceTable.supplierId, invoice.supplierId),
        ne(invoiceTable.id, invoice.id),
        between(invoiceTable.totalAmount,
          invoice.totalAmount * 0.99, invoice.totalAmount * 1.01),
        between(invoiceTable.invoiceDate,
          subDays(invoice.invoiceDate, 7), addDays(invoice.invoiceDate, 7)),
      ));

    return candidates.map(c => ({
      invoiceId: c.id,
      invoiceNumber: c.invoiceNumber,
      similarity: this.calculateSimilarity(invoice, c),
      reason: this.explainMatch(invoice, c),
    }));
  }
}
```

**Testing:**
- Exact duplicate (same supplier + invoice number) prevented at DB level (409)
- Fuzzy duplicate detected: same supplier, amount within 1%, date within 7 days
- Non-duplicates with same supplier but different amounts not flagged
- Duplicate flag displayed in invoice detail view with link to potential duplicates
- User can dismiss false-positive duplicate flags

### Definition of Done — Phase 7

- [ ] AI match resolution proposes resolutions with confidence and reasoning
- [ ] Auto-resolution for high-confidence approvals (configurable threshold)
- [ ] AI suggestions displayed in exception queue for human review
- [ ] Spend analytics dashboard with drill-down by supplier, category, cost centre, time
- [ ] Budget vs actual comparison visualization
- [ ] Duplicate invoice detection (fuzzy matching)
- [ ] Auto-resolution rate metric tracked and displayed
- [ ] E2E test: invoice with price variance -> AI proposes approve with 0.95 confidence -> auto-resolved
- [ ] Dashboard renders within 2 seconds with production-scale data (10K+ invoices)

---

## Phase 8: Supplier Risk & Management

**Goal:** Build the supplier management module with vendor master CRUD, performance tracking, and continuous risk monitoring from external signals. Integrate risk scores into the PO approval workflow.

**Duration estimate:** 2-3 weeks

### Task 8.1: Vendor Master Management

**What:** Build the supplier CRUD with contacts, addresses, and bank accounts stored as JSONB arrays (per Data Model 3). Include supplier onboarding workflow with approval.

**Design:**
```typescript
// apps/api/src/routes/v1/suppliers/create.ts
// Validates: name, legal_name, tax_id, default_currency, payment_terms_days
// JSONB arrays: contacts (at least one primary), addresses (at least one billing), bank_accounts
// Status lifecycle: pending_approval -> active -> inactive | blocked

// apps/web/src/app/(dashboard)/suppliers/[id]/page.tsx
// Tabbed detail view:
// - Overview: name, status, risk score, total spend
// - Contacts: JSONB array editor with role picker
// - Addresses: JSONB array editor with type and country
// - Bank Accounts: JSONB array editor with IBAN/SWIFT validation (masked display)
// - Purchase History: recent POs and invoices
// - Performance: on-time delivery %, quality score, response time
// - Risk Signals: timeline of risk events
```

**Testing:**
- Supplier created with all required fields
- At least one primary contact required (validation)
- Bank account numbers masked in API responses (only last 4 digits visible)
- Supplier search by name, tax_id, peppol_id
- Supplier status changes audited
- Supplier deactivation prevents new PO creation against that supplier
- JSONB contact/address arrays validate structure at application layer

### Task 8.2: Supplier Risk Monitoring

**What:** Implement a background job that periodically checks external signals (news, financial data) for active suppliers and updates their risk scores. Flag high-risk suppliers in the PO approval flow.

**Design:**
```typescript
// apps/api/src/ai/risk-monitor.ts
export class SupplierRiskMonitor {
  async assessRisk(supplierId: string): Promise<RiskAssessment> {
    const supplier = await this.getSupplier(supplierId);

    // Use AI to analyze publicly available signals
    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 1000,
      system: `You are a supplier risk analyst. Given a company name and available identifiers, assess the current risk level based on publicly known information. Consider financial stability, regulatory actions, ESG concerns, cybersecurity incidents, and geopolitical factors.

Return JSON:
{
  "riskScore": 0-100 (higher = riskier),
  "signals": [
    {"type": "financial|cyber|esg|geopolitical|news", "severity": "low|medium|high|critical", "title": "...", "description": "..."}
  ],
  "summary": "One-paragraph risk assessment"
}`,
      messages: [{ role: 'user', content: `Company: ${supplier.name}\nLegal Name: ${supplier.legalName}\nCountry: ${supplier.addresses?.[0]?.country_code}\nTax ID: ${supplier.taxId}` }],
    });

    const assessment = this.parseAssessment(response);

    // Update supplier risk score and signals
    await db.update(supplierTable).set({
      riskScore: assessment.riskScore,
      riskUpdatedAt: new Date(),
    }).where(eq(supplierTable.id, supplierId));

    return assessment;
  }
}

// Integration with PO approval: if supplier risk_score > 70,
// inject an additional mandatory approval step requiring CFO sign-off
```

**Testing:**
- Risk assessment job runs weekly for all active suppliers (configurable frequency)
- Risk score updated on supplier record after assessment
- Risk signals stored in supplier risk_signals JSONB
- PO creation against a high-risk supplier (score > 70) triggers additional approval step
- Risk score badge visible on supplier list and detail views
- Risk signal timeline shows history of assessments
- Manual risk assessment trigger available for ad-hoc checks

### Task 8.3: Supplier Performance Tracking

**What:** Automatically calculate supplier performance metrics (on-time delivery, quality score, price competitiveness) from PO, GRN, and invoice data.

**Design:**
```typescript
// apps/api/src/services/supplier-performance.service.ts
export class SupplierPerformanceService {
  async calculatePerformance(supplierId: string, periodStart: Date, periodEnd: Date): Promise<Performance> {
    // On-time delivery: % of GRNs received by PO expected_delivery_date
    // Quality score: % of items accepted vs received
    // Price competitiveness: average variance of invoice price vs PO price
    // Responsiveness: average time from PO sent to acknowledged
  }
}
```

**Testing:**
- On-time delivery calculated correctly from GRN dates vs PO expected delivery dates
- Quality score reflects accepted vs total received quantities
- Performance metrics update monthly via scheduled job
- Supplier detail view shows performance trend over last 12 months
- Performance data exportable as CSV

### Definition of Done — Phase 8

- [ ] Vendor master CRUD with contacts, addresses, bank accounts
- [ ] Supplier search and filtering
- [ ] Risk score assessment and continuous monitoring
- [ ] High-risk supplier flag in PO approval workflow
- [ ] Supplier performance metrics calculated from transaction data
- [ ] Risk signal timeline on supplier detail view
- [ ] Bank account data masked in all API responses and UI
- [ ] E2E test: supplier risk score > 70 -> PO requires additional CFO approval

---

## Phase 9: Strategic Sourcing (RFQ/RFP)

**Goal:** Build the sourcing module for creating and managing RFQ/RFP events, inviting suppliers to bid, evaluating responses, and awarding contracts that flow into PO creation.

**Duration estimate:** 2-3 weeks

### Task 9.1: Sourcing Event Management

**What:** Implement the sourcing_event table (with lines, invitations, and responses as JSONB per Data Model 3) and API routes for the full sourcing lifecycle: draft -> published -> evaluation -> awarded -> closed.

**Design:**
```typescript
// apps/api/src/services/sourcing.service.ts
export class SourcingService {
  async createEvent(input: CreateSourcingEventInput): Promise<SourcingEvent> {
    // Create event with lines, set status='draft'
  }

  async publish(eventId: string): Promise<void> {
    // Validate event has lines and invitations
    // Set status='published', record published_at
    // Send invitation emails to all invited suppliers
  }

  async submitResponse(eventId: string, supplierId: string, response: SupplierResponse): Promise<void> {
    // Validate supplier is invited and deadline not passed
    // Append to responses JSONB array
  }

  async evaluateAndAward(eventId: string, supplierId: string): Promise<void> {
    // Set winning response status='awarded'
    // Create PO from awarded response lines
    // Set event status='awarded'
  }
}
```

**Testing:**
- Sourcing event created with lines and status='draft'
- Publishing sends invitation emails to all invited suppliers
- Supplier response submission rejected after deadline
- Multiple supplier responses stored and comparable
- Side-by-side comparison view of responses (price, lead time, terms)
- Award creates PO from winning response
- Event lifecycle: draft -> published -> evaluation -> awarded -> closed

### Task 9.2: Sourcing UI

**What:** Build the sourcing dashboard, event creation wizard, supplier invitation interface, response comparison view, and award workflow.

**Design:**
```tsx
// Event creation wizard:
// Step 1: Event type (RFQ, RFP, RFI) and basic details
// Step 2: Add line items with descriptions, quantities, target prices
// Step 3: Select and invite suppliers from vendor master
// Step 4: Set response deadline and evaluation criteria
// Step 5: Review and publish

// Response comparison matrix:
// Columns: one per supplier response
// Rows: line items showing unit price, qty offered, lead time
// Highlight: lowest price in green, highest in red
// Totals row with grand total comparison
```

**Testing:**
- Wizard completes sourcing event creation in 5 steps
- Invited suppliers appear in the invitation list with status
- Response comparison matrix displays all responses side-by-side
- Best-price highlighting works correctly across all lines
- Award button creates PO and shows confirmation

### Definition of Done — Phase 9

- [ ] Sourcing events created with RFQ/RFP/RFI types
- [ ] Supplier invitations sent via email
- [ ] Supplier response submission and deadline enforcement
- [ ] Multi-supplier response comparison
- [ ] Award workflow creates PO from winning response
- [ ] Sourcing dashboard shows all events by status
- [ ] E2E test: create RFQ -> invite 3 suppliers -> receive responses -> award -> PO created

---

## Phase 10: E-Invoicing & EDI Compliance

**Goal:** Implement PEPPOL e-invoicing (UBL 2.1/EN 16931), EDI X12 850/810 support, and cXML integration. This addresses the growing regulatory requirements identified in the standards research.

**Duration estimate:** 3-4 weeks

### Task 10.1: UBL Document Builder/Parser

**What:** Build a library that generates and parses UBL 2.1 XML documents (Order, Invoice, CreditNote, DespatchAdvice) compliant with EN 16931 for e-invoicing.

**Design:**
```typescript
// apps/api/src/integrations/ubl/builder.ts
export class UBLInvoiceBuilder {
  build(invoice: Invoice): string {
    // Generate UBL 2.1 Invoice XML compliant with EN 16931
    // Include all mandatory fields: BT-1 (invoice number), BT-2 (issue date),
    // BT-5 (currency), BT-6 (VAT accounting currency), BT-9 (due date),
    // BT-24 (specification identifier), BT-44-BT-63 (seller/buyer party)
    // Validate against EN 16931 schematron rules before returning
  }
}

// apps/api/src/integrations/ubl/parser.ts
export class UBLInvoiceParser {
  parse(xml: string): ParsedInvoice {
    // Parse UBL 2.1 Invoice XML
    // Extract all fields to CreateInvoiceInput format
    // Validate EN 16931 compliance
    // Return parsed data with validation report
  }
}
```

**Testing:**
- Generated UBL Invoice XML validates against UBL 2.1 XSD schema
- Generated XML passes EN 16931 schematron validation rules
- Round-trip: build -> parse -> build produces identical XML
- All mandatory EN 16931 fields present in generated documents
- Parser handles UBL invoices from different EU implementations (German XRechnung, Italian FatturaPA wrapper)
- Invalid UBL documents produce clear validation error reports

### Task 10.2: PEPPOL Access Point Integration

**What:** Integrate with a PEPPOL Access Point (AP) to send and receive business documents (purchase orders, invoices) over the PEPPOL network.

**Design:**
```typescript
// apps/api/src/integrations/peppol/client.ts
export class PeppolClient {
  async sendDocument(document: UBLDocument, recipientId: string): Promise<TransmissionResult> {
    // Send UBL document to PEPPOL Access Point via AS4 or REST API
    // AP handles SMP lookup and routing
    // Returns transmission ID for tracking
  }

  async receiveDocuments(): Promise<ReceivedDocument[]> {
    // Poll Access Point for incoming documents
    // Parse UBL and create invoices/order responses
  }
}

// Webhook handler for real-time document receipt from AP
// POST /webhooks/peppol/receive
```

**Testing:**
- PO sent via PEPPOL to a test supplier endpoint
- Invoice received via PEPPOL parsed and created as invoice record
- Transmission data (PEPPOL message ID, sender/receiver endpoints) stored in transmission_data JSONB
- Failed transmissions retried via job queue
- PEPPOL participant ID validated during supplier setup

### Task 10.3: EDI X12/EDIFACT Support

**What:** Build parsers and generators for EDI X12 850 (purchase order) and 810 (invoice), and EDIFACT ORDERS and INVOIC messages.

**Design:**
```typescript
// apps/api/src/integrations/edi/x12-generator.ts
export class X12Generator {
  generatePO850(po: PurchaseOrder): string {
    // Generate X12 850 transaction set
    // ISA/GS/ST/BEG/DTM/N1/PO1/CTT/SE/GE/IEA segments
  }
}

// apps/api/src/integrations/edi/x12-parser.ts
export class X12Parser {
  parseInvoice810(edi: string): ParsedInvoice {
    // Parse X12 810 transaction set
    // Extract invoice data from IT1/TDS/CTT segments
  }
}
```

**Testing:**
- Generated X12 850 validates against X12 grammar rules
- Parsed X12 810 extracts correct invoice lines and totals
- EDIFACT ORDERS generates valid EDIFACT syntax
- EDI interchange control numbers tracked for duplicate detection
- Round-trip test: generate 850 -> parse 850 -> verify data integrity

### Definition of Done — Phase 10

- [ ] UBL 2.1 Invoice and Order documents generated and validated
- [ ] EN 16931 compliance validation passes for all mandatory fields
- [ ] PEPPOL document sending and receiving functional with test Access Point
- [ ] EDI X12 850/810 generation and parsing works
- [ ] EDIFACT ORDERS/INVOIC generation and parsing works
- [ ] E-invoicing source_data JSONB populated with compliance metadata
- [ ] Integration tests against sample UBL/EDI documents from real-world sources

---

## Phase 11: ERP Integrations & Chat Approvals

**Goal:** Build native connectors for common ERPs and chat-based approval workflows via Slack and Microsoft Teams, as identified in the feature survey.

**Duration estimate:** 3-4 weeks

### Task 11.1: ERP Connector Abstraction

**What:** Design an integration abstraction layer and build connectors for QuickBooks, Xero, and NetSuite. Connectors synchronize suppliers, chart of accounts, and payment status.

**Design:**
```typescript
// apps/api/src/integrations/erp/connector.ts
export interface ERPConnector {
  syncSuppliers(): Promise<SyncResult>;
  syncChartOfAccounts(): Promise<SyncResult>;
  pushPurchaseOrder(po: PurchaseOrder): Promise<PushResult>;
  pushInvoice(invoice: Invoice): Promise<PushResult>;
  getPaymentStatus(invoiceRef: string): Promise<PaymentStatus>;
}

// apps/api/src/integrations/erp/quickbooks.connector.ts
export class QuickBooksConnector implements ERPConnector { /* ... */ }

// apps/api/src/integrations/erp/xero.connector.ts
export class XeroConnector implements ERPConnector { /* ... */ }
```

**Testing:**
- QuickBooks connector syncs vendors and chart of accounts
- Xero connector pushes invoices and reads payment status
- Connector failure retried with exponential backoff
- Sync conflicts resolved with "most recent wins" strategy
- Credentials stored encrypted, never logged

### Task 11.2: Slack & Teams Approval Bot

**What:** Build a bot that sends approval requests to Slack channels or Teams chats with inline approve/reject buttons. Approvers can complete approvals without logging into the procurement portal.

**Design:**
```typescript
// apps/api/src/integrations/slack/approval-bot.ts
export class SlackApprovalBot {
  async sendApprovalRequest(approval: ApprovalRequest, requisition: PurchaseRequisition): Promise<void> {
    await this.slack.chat.postMessage({
      channel: approver.slackChannelId,
      blocks: [
        { type: 'header', text: { type: 'plain_text', text: `Approval Required: ${requisition.title}` } },
        { type: 'section', text: { type: 'mrkdwn', text: `*Requester:* ${requisition.requesterName}\n*Amount:* ${requisition.totalEstimated} ${requisition.currency}\n*Cost Centre:* ${requisition.costCentreName}` } },
        { type: 'actions', elements: [
          { type: 'button', text: { type: 'plain_text', text: 'Approve' }, style: 'primary', action_id: `approve_${approval.id}` },
          { type: 'button', text: { type: 'plain_text', text: 'Reject' }, style: 'danger', action_id: `reject_${approval.id}` },
          { type: 'button', text: { type: 'plain_text', text: 'View Details' }, url: `${this.baseUrl}/requisitions/${requisition.id}` },
        ]},
      ],
    });
  }

  async handleInteraction(payload: SlackInteractionPayload): Promise<void> {
    // Process approve/reject from Slack button click
    // Update approval_request and requisition status
    // Update the Slack message to show the decision
  }
}
```

**Testing:**
- Approval request posted to correct Slack channel with formatted message
- Approve button click resolves the approval step and updates requisition status
- Reject button click prompts for rejection reason (Slack modal) and rejects
- Slack message updated after decision (e.g., "Approved by Jane Doe at 2:30 PM")
- Invalid or expired approval links return appropriate error
- Teams integration mirrors Slack functionality via Adaptive Cards

### Task 11.3: Webhook Event System

**What:** Build a configurable outgoing webhook system that notifies external systems when procurement events occur (PO created, invoice matched, payment initiated).

**Design:**
```typescript
// apps/api/src/services/webhook.service.ts
// Tenant configures webhook URLs for specific event types
// Outbox pattern: events queued in event_outbox table, delivered with retry
// Payload follows CloudEvents 1.0 specification
// HMAC signature for webhook verification (X-Signature-256 header)
```

**Testing:**
- Webhook delivered within 30 seconds of event
- Failed webhooks retried with exponential backoff (max 5 retries)
- HMAC signature validated by receiving system
- Webhook payload conforms to CloudEvents 1.0 spec
- Webhook delivery log shows success/failure history

### Definition of Done — Phase 11

- [ ] QuickBooks and Xero connectors functional (sync and push)
- [ ] NetSuite connector functional
- [ ] Slack approval bot sends requests and processes decisions
- [ ] Teams approval bot sends requests and processes decisions
- [ ] Webhook system delivers events to configured URLs
- [ ] ERP connector configuration UI in tenant settings
- [ ] E2E test: requisition submitted -> Slack notification -> approve in Slack -> PO created

---

## Phase 12: Production Hardening & Deployment

**Goal:** Prepare the system for production use: security hardening, performance optimization, monitoring, documentation, and deployment automation.

**Duration estimate:** 3-4 weeks

### Task 12.1: Security Hardening

**What:** Implement OWASP API Security Top 10 mitigations, row-level security for multi-tenancy, encrypted secrets management, and penetration testing.

**Design:**
```typescript
// Row-Level Security (RLS) in PostgreSQL
// ALTER TABLE purchase_order ENABLE ROW LEVEL SECURITY;
// CREATE POLICY tenant_isolation ON purchase_order
//   USING (tenant_id = current_setting('app.current_tenant')::UUID);

// apps/api/src/middleware/security.ts
// - Rate limiting per tenant (not just per IP)
// - Request body size limits (10MB for file uploads, 1MB for API calls)
// - SQL injection prevention via parameterized queries (Drizzle handles this)
// - CSRF protection for web forms
// - Content Security Policy headers
// - Input sanitization for stored XSS prevention
```

**Testing:**
- RLS prevents cross-tenant data access even with direct SQL (escape-hatch proof)
- API rate limits enforced per tenant (429 returned when exceeded)
- OWASP ZAP scan produces no high or critical findings
- Encrypted credentials (bank accounts, API keys) not visible in logs, error messages, or API responses
- Authentication bypasses tested and prevented
- BOLA (Broken Object Level Authorization) tests: user cannot access other users' requisitions by ID guessing

### Task 12.2: Performance Optimization

**What:** Query optimization for the critical paths (three-way matching, spend analytics), database connection pooling, Redis caching for hot paths, and audit log partitioning.

**Design:**
```sql
-- Audit log partitioning by month
ALTER TABLE audit_log RENAME TO audit_log_template;
CREATE TABLE audit_log (LIKE audit_log_template INCLUDING ALL) PARTITION BY RANGE (created_at);
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... automated partition creation via pg_partman

-- Connection pooling via PgBouncer
-- Redis caching for: tenant config, approval policies, tolerance rules, UNSPSC taxonomy
```

**Testing:**
- Three-way matching for a single invoice completes in <500ms with 100K+ PO lines
- Spend analytics query (group by supplier, last 12 months) returns in <2s with 50K+ invoices
- Audit log query by entity_type + entity_id returns in <100ms with 10M+ rows (partitioned)
- System handles 100 concurrent API requests without degradation (load test)
- Redis cache invalidation works correctly when configuration changes
- Database connection pool handles 50 concurrent connections

### Task 12.3: Monitoring & Observability

**What:** Structured logging, application metrics (Prometheus), distributed tracing (OpenTelemetry), health checks, and alerting.

**Design:**
```typescript
// apps/api/src/observability/metrics.ts
// Metrics: request_duration_seconds, db_query_duration_seconds,
// ai_classification_duration_seconds, ai_classification_cost_usd,
// match_auto_resolved_total, match_exception_total, active_tenants,
// peppol_documents_sent_total, edi_documents_processed_total

// Health check endpoint: GET /health
// Deep health check: GET /health/ready (checks DB, Redis, AI service connectivity)
```

**Testing:**
- Prometheus /metrics endpoint exposes all defined metrics
- Structured JSON logs include correlation ID, tenant ID, and request path
- OpenTelemetry traces capture end-to-end request flow including DB queries
- Alert rules fire for: error rate > 5%, P99 latency > 5s, AI service unreachable
- Health check returns 503 when database is unreachable

### Task 12.4: Docker & Deployment

**What:** Production Docker images, Docker Compose for self-hosted deployment, Kubernetes Helm chart for enterprise, CI/CD pipeline, and database migration automation.

**Design:**
```yaml
# docker/docker-compose.prod.yml
services:
  api:
    build: { context: ., dockerfile: docker/Dockerfile.api }
    environment:
      DATABASE_URL: postgresql://procurement:${DB_PASSWORD}@postgres:5432/procurement
      REDIS_URL: redis://redis:6379
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
    depends_on: [postgres, redis]
    restart: unless-stopped
  web:
    build: { context: ., dockerfile: docker/Dockerfile.web }
    ports: ["3000:3000"]
    depends_on: [api]
    restart: unless-stopped
  postgres:
    image: postgres:16-alpine
    volumes: [pgdata:/var/lib/postgresql/data]
    restart: unless-stopped
  redis:
    image: redis:7-alpine
    restart: unless-stopped
  worker:
    build: { context: ., dockerfile: docker/Dockerfile.api }
    command: ["node", "dist/worker.js"]
    depends_on: [postgres, redis]
    restart: unless-stopped
```

**Testing:**
- `docker compose up` starts the full stack from scratch with database migration
- `docker compose down && docker compose up` preserves data (volumes persist)
- Multi-stage Docker build produces images < 200MB
- Health checks pass within 30 seconds of container start
- Database migrations run automatically on API startup (idempotent)
- Helm chart deploys to Kubernetes with `helm install procurement ./charts/procurement`
- CI/CD pipeline: lint -> test -> build -> push images -> deploy to staging

### Task 12.5: Documentation & Developer Experience

**What:** API documentation, architecture decision records, contribution guide, and developer setup guide.

**Design:**
- OpenAPI spec auto-generated and hosted at `/docs`
- Architecture Decision Records (ADRs) for key decisions (data model, AI model selection, auth strategy)
- `CONTRIBUTING.md` with development setup, coding standards, PR template
- `docker compose up` starts the full dev environment in one command

**Testing:**
- New developer can clone, run `docker compose up`, and have a working system in < 5 minutes
- API documentation covers all endpoints with request/response examples
- ADRs exist for the top 10 architectural decisions

### Definition of Done — Phase 12

- [ ] OWASP API Security Top 10 mitigations implemented
- [ ] Row-level security enforced in PostgreSQL
- [ ] Performance benchmarks met (matching <500ms, analytics <2s)
- [ ] Prometheus metrics and alerting configured
- [ ] Docker Compose self-hosted deployment works end-to-end
- [ ] Helm chart deploys to Kubernetes
- [ ] CI/CD pipeline automates build, test, and deploy
- [ ] API documentation complete and auto-generated
- [ ] New developer setup time < 5 minutes
- [ ] Load test passes: 100 concurrent users, <5% error rate, P99 <5s
- [ ] Security scan (OWASP ZAP) shows zero high/critical findings

---

## Summary

| Phase | Name | Duration | Key Deliverables |
|-------|------|----------|------------------|
| 1 | Foundation & Infrastructure | 3-4 weeks | Monorepo, DB schema, auth, multi-tenancy, audit log |
| 2 | Requisitions & Approvals | 3-4 weeks | Purchase requisitions, approval workflows, approval inbox |
| 3 | Purchase Orders | 2-3 weeks | PO creation, PDF generation, email delivery, budget commitment |
| 4 | Goods Receipt | 2 weeks | GRN recording, quantity tracking, PO status updates |
| 5 | AI Spend Classification | 2-3 weeks | UNSPSC classification, feedback loop, confidence scoring |
| 6 | Invoices & Three-Way Matching | 3-4 weeks | Invoice ingestion, matching engine, tolerance rules, exception queue |
| 7 | AI Match Resolution & Analytics | 3-4 weeks | Autonomous exception resolution, spend dashboards, duplicate detection |
| 8 | Supplier Risk & Management | 2-3 weeks | Vendor master, risk monitoring, performance tracking |
| 9 | Strategic Sourcing | 2-3 weeks | RFQ/RFP events, supplier bidding, award workflow |
| 10 | E-Invoicing & EDI | 3-4 weeks | UBL 2.1, PEPPOL, EN 16931, EDI X12/EDIFACT |
| 11 | ERP Integrations & Chat | 3-4 weeks | QuickBooks/Xero/NetSuite, Slack/Teams approvals, webhooks |
| 12 | Production Hardening | 3-4 weeks | Security, performance, monitoring, deployment, documentation |
| **Total** | | **32-43 weeks** | **Full AI-native procurement platform** |

**MVP (Phases 1-4 + 6):** ~14-17 weeks to a working procure-to-pay system with three-way matching.
**AI-differentiated MVP (+ Phase 5 + 7):** ~19-24 weeks with AI spend classification and autonomous match resolution.
**Full product (all phases):** ~32-43 weeks for the complete platform with compliance, integrations, and production hardening.
