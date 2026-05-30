# Early Childhood Education Platform — Phased Development Plan

> Project: 428-early-childhood-education-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.
>
> This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` documents. The data design follows **Suggestion 3 (Hybrid Relational + JSONB)** — a PostgreSQL core for transactionally-critical operational/billing data, with JSONB columns for the high-variability domains (developmental framework definitions, subsidy rule parameters, state reporting payloads) where pure normalization is cumbersome (see Suggestion 1 "Cons" §1, §3, §6).

---

## Product Summary

An AI-native, open-source platform unifying child development tracking, parent communication, attendance, and multi-source billing for daycare centers, preschools, Head Start sites, and family childcare programs. It targets the structural gaps no incumbent fills: an open, documented REST API; configurable alignment to arbitrary developmental frameworks (NAEYC, HSELOF, EYFS, state standards) from one codebase; a multi-state subsidy billing rules engine; offline-first mobile capture; and embedded AI (observation narrative generation, vision-AI domain tagging, billing anomaly detection, ratio forecasting).

**Primary personas**
- **Teacher / classroom staff** — captures observations, photos, daily activities, check-in/out; works on tablets, often offline.
- **Director / site admin** — manages enrollment, billing, staff scheduling, ratio compliance, licensing documentation.
- **Multi-site operator admin** — consolidated billing and cross-site reporting.
- **Parent / guardian** — daily updates, messaging, payments, portfolio viewing via mobile app.
- **Integrator** — third-party developer consuming the open API.

**Deployment model** — Hybrid: SaaS-hosted by default; fully self-hostable via Docker Compose for Head Start / district data-residency requirements. Single codebase serves both.

**MVP value path** — Phases 1–6 deliver the core operational loop (auth/tenancy → child & enrollment records → attendance & check-in → daily activities & parent feed → billing & payments → developmental observations). AI augmentation, subsidy engine, staff/compliance, and the open API follow.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | **TypeScript** (Node.js 22 LTS) | One language across API, web admin, and React Native mobile maximizes code/type sharing (DTOs, validation schemas) for a high-turnover-staff product where shipping velocity matters. The domain is API/frontend-heavy, not ML-training-heavy; LLM work is API calls, well-served by TS SDKs. |
| API framework | **NestJS** (Fastify adapter) | Modular DI architecture maps cleanly onto the phased, multi-module design (billing, attendance, observations as isolated modules). First-class OpenAPI 3.1 generation via decorators satisfies the standards.md API-gap differentiator. Fastify adapter for throughput. |
| Database | **PostgreSQL 16** | Per data-model Suggestion 3. ACID for billing/subsidy reconciliation; RLS for tenant isolation; JSONB for framework/subsidy-rule flexibility; native full-text search for observations; mature Ed-Fi/CEDS mapping. |
| ORM / migrations | **Prisma** + raw SQL for RLS policies & partitioning | Prisma gives typed queries and migration tooling. RLS policies, partition DDL, and GIN indexes on JSONB are applied via raw SQL migration steps Prisma can't express. |
| Validation / DTO | **Zod** (shared package) | Single source of truth for request validation, OpenAPI schema derivation (`zod-to-openapi`), and client-side form validation reused in web + mobile. |
| Task queue | **BullMQ** on **Redis 7** | Async workloads: invoice generation, recurring billing runs, subsidy claim assembly, AI narrative generation, translation, push notifications, webhook delivery. Redis also backs real-time roster cache and rate limiting. |
| Real-time | **Socket.IO** (Redis adapter) | Live classroom roster, ratio alerts, parent activity feed, messaging delivery across horizontally-scaled API nodes. |
| Web admin frontend | **Next.js 16** (App Router) + **shadcn/ui** + Tailwind | Director/operator dashboard: server components for reporting reads, client components for interactive scheduling/billing. shadcn for accessible (WCAG 2.2) primitives. |
| Mobile (parent + staff) | **React Native (Expo)** + **WatermelonDB** | Offline-first underserved-area differentiator. WatermelonDB provides a local SQLite-backed observable store with a sync protocol for background reconciliation; battery-efficient for all-day tablet use. Shares Zod schemas with backend. |
| Object storage | **S3-compatible** (AWS S3 / Cloudflare R2 / MinIO self-hosted) | Photo/video/audio observation media and documents. Signed URLs; MinIO for self-host parity. |
| Auth / identity | **OAuth 2.0 + OIDC**, JWT (RFC 7519), self-issued via `@nestjs/passport` + optional external IdP | Parent/staff/admin auth; SSO (Azure AD / Google Workspace) for enterprise operators per standards.md OIDC. Parental-consent workflow layered on top for COPPA. |
| Payments | **Stripe** (Connect + ACH) | PCI DSS scope reduction to SAQ A via Stripe Elements / hosted fields; supports ACH next-day, cards, payment portal, autopay. Abstracted behind a `PaymentProvider` interface for self-host alternatives. |
| LLM provider | **Vercel AI SDK** over OpenAI/Anthropic, provider-abstracted | Observation narratives, lesson planning, anomaly explanation, translation. Provider abstraction + documented data-processor boundary for COPPA child-PII handling. Vision model for domain auto-tagging. |
| Translation | LLM-backed `TranslationProvider` + cache | Multilingual parent comms (underserved-area differentiator). Cached per (source, target, hash) to control cost. |
| Containerisation | **Docker** + **Docker Compose** | Self-host parity (api, web, postgres, redis, minio, worker). Per-service Dockerfiles; compose for one-command local + self-hosted deploy. |
| Testing | **Vitest** (unit/integration) + **Supertest** (HTTP) + **Playwright** (web e2e) + **Testcontainers** (real Postgres/Redis) | Fast unit runs; Testcontainers spin real Postgres for RLS/migration/partition tests that mocks can't validate. |
| Code quality | **ESLint** (typescript-eslint) + **Prettier** + **tsc** strict | Standard TS toolchain; `tsc --noEmit` in CI gate. |
| Package manager / monorepo | **pnpm workspaces** + **Turborepo** | Monorepo: `apps/*` and `packages/*` share Zod schemas, generated OpenAPI client, and domain types. Turborepo caches builds/tests. |
| API spec | **OpenAPI 3.1** (generated) | standards.md: the documented public API is a primary differentiator. Spec drives a generated TS client consumed by web + mobile. |
| Standards data formats | Ed-Fi v6.0 / CEDS mapping, JSON Schema 2020-12, FHIR R5 Immunization (later phase) | State reporting interoperability and health-record integration pathway. |

### Project Structure

```
ece-platform/
├── package.json                      # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml                # api, web, worker, postgres, redis, minio
├── docker-compose.selfhost.yml       # production self-host overlay
├── .env.example
├── tsconfig.base.json
├── apps/
│   ├── api/                          # NestJS REST + WebSocket gateway
│   │   ├── Dockerfile
│   │   ├── src/
│   │   │   ├── main.ts               # bootstrap, OpenAPI doc, RLS interceptor
│   │   │   ├── common/               # guards, interceptors (tenant, audit), filters
│   │   │   │   ├── tenant/           # RLS context + AsyncLocalStorage
│   │   │   │   ├── auth/             # JWT, OIDC, consent guard
│   │   │   │   └── audit/            # audit-log interceptor (COPPA/FERPA)
│   │   │   ├── modules/
│   │   │   │   ├── organizations/    # orgs, sites, classrooms
│   │   │   │   ├── children/         # children, families, guardians, medical
│   │   │   │   ├── enrollment/       # applications, waitlist, e-sign, docs
│   │   │   │   ├── attendance/       # check-in/out, roster, ratio
│   │   │   │   ├── activities/       # daily activities + parent feed
│   │   │   │   ├── observations/     # observations, milestones, media, AI
│   │   │   │   ├── frameworks/       # developmental framework engine
│   │   │   │   ├── billing/          # plans, invoices, payments
│   │   │   │   ├── subsidy/          # authorizations, claims, rules engine
│   │   │   │   ├── staff/            # staff, credentials, schedule, timeclock
│   │   │   │   ├── compliance/       # incidents, medication, licensing, CACFP
│   │   │   │   ├── messaging/        # messages, broadcasts, translation
│   │   │   │   ├── ai/               # LLM provider, prompts, vision tagging
│   │   │   │   ├── sync/             # offline sync endpoints (pull/push)
│   │   │   │   ├── webhooks/         # outbound webhook delivery
│   │   │   │   └── reporting/        # CACFP, licensing, Ed-Fi/CEDS export
│   │   │   └── infra/                # prisma client, redis, storage, queue
│   │   └── test/
│   ├── worker/                       # BullMQ processors (shares modules via DI)
│   │   ├── Dockerfile
│   │   └── src/processors/           # billing-run, subsidy-claim, ai-narrative, notify
│   ├── web/                          # Next.js 16 admin dashboard
│   │   ├── Dockerfile
│   │   └── src/app/
│   └── mobile/                       # Expo React Native (parent + staff)
│       └── src/
│           ├── db/                   # WatermelonDB models + sync
│           ├── parent/               # parent app screens
│           └── staff/                # staff capture screens
├── packages/
│   ├── schemas/                      # Zod schemas (shared DTOs + validation)
│   ├── api-client/                   # generated OpenAPI TS client
│   ├── domain/                       # shared enums, value objects, calculators
│   ├── framework-defs/               # bundled framework JSON (NAEYC, HSELOF, EYFS…)
│   └── config/                       # shared eslint/tsconfig/prettier
├── prisma/
│   ├── schema.prisma
│   └── migrations/                   # incl. raw-SQL RLS/partition steps
└── tests/
    └── fixtures/                     # sample orgs, frameworks, diffs, subsidy rules
```

The structure is additive: each phase adds a module under `apps/api/src/modules/`, optionally a worker processor, web routes, and mobile screens — no restructuring required between phases.

---

## Phase 1: Foundation — Monorepo, Tenancy, Auth, Data Core

### Purpose
Establish the monorepo, the PostgreSQL schema foundation with RLS-based multi-tenancy, authentication/authorization, the shared Zod/OpenAPI contract pipeline, and the audit/consent scaffolding that COPPA and FERPA require. After this phase, an authenticated user scoped to a tenant can be created and every subsequent module inherits tenant isolation, auditing, and typed contracts for free.

### Tasks

#### 1.1 — Monorepo & toolchain bootstrap

**What**: Stand up the pnpm/Turborepo workspace, shared config packages, Docker Compose dev stack, and CI gates.

**Design**:
- `pnpm-workspace.yaml` globs `apps/*` and `packages/*`.
- `packages/config` exports shared `eslint.config.js`, `tsconfig.base.json` (`strict: true`, `noUncheckedIndexedAccess: true`), `prettier` config.
- `turbo.json` pipelines: `build` (depends on `^build`), `lint`, `typecheck`, `test`.
- `docker-compose.yml` services: `postgres:16`, `redis:7`, `minio`, plus `api` and `worker` with hot-reload mounts. Healthchecks on postgres/redis.
- `.env.example` enumerates: `DATABASE_URL`, `REDIS_URL`, `S3_ENDPOINT/KEY/SECRET/BUCKET`, `JWT_SECRET`, `OIDC_*`, `STRIPE_*`, `AI_PROVIDER/AI_API_KEY`, `APP_BASE_URL`.
- CI (GitHub Actions): `pnpm install --frozen-lockfile` → `turbo lint typecheck test build`.

**Testing**:
- `Unit: turbo build` → all packages compile, exit 0.
- `Integration: docker compose up` → postgres, redis, minio report healthy within 60s.
- `CI: lint with an intentional unused var` → fails with the var name in output (proves gate works), then removed.

#### 1.2 — Prisma schema + RLS multi-tenancy

**What**: Define the foundational tables (`organizations`, `sites`, `classrooms`, `users`, `audit_logs`) and enforce `organization_id` isolation via PostgreSQL RLS.

**Design**:
- Adopt the DDL from data-model Suggestion 1 for `organizations`, `sites`, `classrooms` verbatim (TEXT enums become Prisma enums where stable; JSONB used for `sites.licensing_metadata` and `classrooms.ratio_rules` per Suggestion 3 hybrid stance).
- `users` table is the auth anchor referenced by `guardians.user_account_id` and `staff.user_account_id`:

```prisma
model User {
  id             String   @id @default(uuid()) @db.Uuid
  organizationId String   @map("organization_id") @db.Uuid
  email          String
  passwordHash   String?  @map("password_hash")   // null when OIDC-only
  oidcSubject    String?  @map("oidc_subject")
  role           UserRole
  status         UserStatus @default(active)
  createdAt      DateTime @default(now()) @map("created_at")
  @@unique([organizationId, email])
  @@map("users")
}
enum UserRole { super_admin org_admin site_director teacher assistant guardian integrator }
enum UserStatus { active invited suspended }
```

- RLS applied via a raw-SQL migration step for every tenant-scoped table:

```sql
ALTER TABLE children ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON children
  USING (organization_id = current_setting('app.current_tenant', true)::uuid);
```

- A `TenantContextInterceptor` reads the JWT `org` claim, opens a Prisma transaction, and issues `SELECT set_config('app.current_tenant', $1, true)` before handing the scoped client to the request via `AsyncLocalStorage`. `super_admin` may set a `bypass_rls` role for cross-tenant admin ops.
- Partition DDL stubs (raw SQL) for `attendance_records`, `daily_activities`, `observations`, `messages` declared `PARTITION BY RANGE (created_at)` with a monthly partition-creation helper function.

**Testing**:
- `Integration (Testcontainers Postgres): two orgs, query children as org A` → returns only org A rows.
- `Integration: set_config to org B then query` → returns only org B rows; zero leakage across the boundary.
- `Integration: super_admin bypass role` → sees both orgs.
- `Integration: missing app.current_tenant` → RLS returns 0 rows (fail-closed), asserted explicitly.
- `Unit: monthly partition helper for 2026-07` → creates `attendance_records_2026_07` with correct bounds.

#### 1.3 — Auth: JWT + OIDC + RBAC + consent guard

**What**: Email/password and OIDC login issuing JWTs, role-based guards, and a COPPA parental-consent gate.

**Design**:
- `POST /auth/login` → `{ accessToken, refreshToken }`. Access JWT claims: `sub`, `org`, `role`, `sites: string[]`, `consentRequired: boolean`. 15-min access, 30-day rotating refresh.
- `POST /auth/oidc/callback` → exchanges IdP code, upserts `User` by `oidcSubject`, issues platform JWT.
- `@Roles(...)` decorator + `RolesGuard`; `@RequireConsent()` decorator + `ConsentGuard` that blocks child-PII-writing routes until the guardian's consent record is `granted`.
- `consents` table:

```prisma
model Consent {
  id             String   @id @default(uuid()) @db.Uuid
  organizationId String   @db.Uuid @map("organization_id")
  guardianId     String   @db.Uuid @map("guardian_id")
  childId        String?  @db.Uuid @map("child_id")
  scope          String   // 'data_processing','photo','ai_processing','third_party_share'
  status         ConsentStatus
  grantedAt      DateTime? @map("granted_at")
  revokedAt      DateTime? @map("revoked_at")
  policyVersion  String   @map("policy_version")
  @@map("consents")
}
enum ConsentStatus { pending granted revoked }
```

**Testing**:
- `Integration: login valid creds` → 200 + tokens; `invalid` → 401, generic message (no user-enumeration).
- `Integration: teacher hits org-admin route` → 403.
- `Integration: write observation without photo consent on @RequireConsent route` → 403 with scope name; with consent → 200.
- `Unit: expired access token` → 401; `valid refresh` → new access token, refresh rotated.

#### 1.4 — Shared contract pipeline (Zod → OpenAPI → client)

**What**: Establish `packages/schemas`, OpenAPI 3.1 generation, and the generated typed API client.

**Design**:
- All DTOs defined once as Zod schemas in `packages/schemas`, decorated via `zod-to-openapi`.
- `apps/api` builds the OpenAPI 3.1 doc at boot, served at `/openapi.json` and Swagger UI at `/docs`.
- A build step runs `openapi-typescript` (or `orval`) to emit `packages/api-client`.
- `audit_logs` interceptor writes `{ actor, action, entity, entityId, before, after, ip }` for every mutating request (FERPA access/modification trail).

**Testing**:
- `Unit: Zod CreateChild schema, missing dateOfBirth` → parse error naming the field.
- `Integration: GET /openapi.json` → valid OpenAPI 3.1 (validated with `@apidevtools/swagger-parser`).
- `Integration: any mutating request` → corresponding `audit_logs` row with before/after.

---

## Phase 2: Children, Families & Enrollment

### Purpose
Model the central entity (the child) and everything that orbits it: families, guardians, medical/allergy records, and the full digital enrollment funnel (application → waitlist → e-signature → enrolled). After this phase the platform holds a real roster of children with verified guardians, the prerequisite for attendance, billing, and observations.

### Tasks

#### 2.1 — Children, families, guardians, medical records

**What**: CRUD for families, guardians, children, allergies, and medical records with consent enforcement.

**Design**: Adopt Suggestion 1 DDL for `families`, `guardians`, `children`, `child_allergies`, `child_medical_records`. Endpoints:
- `POST /families`, `POST /families/:id/guardians`, `POST /children`, `GET /children?status=&classroomId=&q=`.
- `children.notes` and emergency/custody flags; `photo_consent` mirrored into a `Consent` row.
- Age computed from `date_of_birth` as a derived field in responses (months + band).
- `child_allergies.severity = 'life_threatening'` surfaces an `allergyAlerts[]` array on every child read used downstream by activities/meals.

**Testing**:
- `Integration: create child under family` → 201, linked, RLS-scoped.
- `Integration: list children q=partial-name` → full-text match.
- `Unit: age-band calculator for DOB` → correct months and band ('infant'/'toddler'/'preschool'/'pre_k').
- `Integration: severe allergy` → appears in child read `allergyAlerts`.

#### 2.2 — Enrollment applications, waitlist, documents, e-signature

**What**: Public enrollment application intake, waitlist ordering, document collection, and e-signature on the enrollment agreement.

**Design**:
- `enrollment_applications` (JSONB `form_data` for site-configurable fields per Suggestion 3) with `status` state machine: `submitted → waitlisted → offered → accepted → enrolled | declined | withdrawn`.
- `POST /public/applications` (unauthenticated, rate-limited, captcha-gated) creates application + provisional family/guardian.
- Waitlist position = ordered by `(priority, submitted_at)`; `GET /waitlist?classroomId=`.
- Document upload via signed S3 PUT URL; `enrollment_documents` rows track `{ type, status, expiresAt }` (immunization, custody, emergency-contact).
- E-signature: `POST /applications/:id/sign` stores signer identity, timestamp, IP, document hash, and user-agent (tamper-evident envelope).
- On `accepted`, a worker job promotes the application to `children` + `families` + `guardians` and creates the consent records.

**Testing**:
- `Unit: state machine, offered→enrolled` valid; `submitted→enrolled` rejected.
- `Integration: public application` → 201, provisional records, no auth required.
- `Integration: sign agreement` → signature envelope persisted with hash; re-sign blocked.
- `E2E: submit → waitlist → offer → accept` → child appears in roster with consents seeded.

---

## Phase 3: Attendance, Check-in/out & Live Roster

### Purpose
Deliver the daily operational heartbeat: contactless check-in/out, authorized-pickup verification, the real-time classroom roster, and live staff-to-child ratio computation. This is table-stakes for every center and feeds attendance data into both subsidy billing (Phase 8) and CACFP (Phase 9).

### Tasks

#### 3.1 — Check-in / check-out & attendance records

**What**: QR/PIN/manual check-in and check-out with authorized-pickup verification.

**Design**: Adopt Suggestion 1 `attendance_records` DDL (partitioned by month per 1.2).
- `POST /attendance/check-in` `{ childId, method, guardianId?, pin? }` → validates guardian `can_pickup`; on PIN method verifies hashed PIN; rejects if `enrollment_status != 'enrolled'`.
- `POST /attendance/check-out` requires an authorized-pickup guardian (`can_pickup = true`) or a per-event authorization token.
- Idempotent per `(child_id, date)`; double check-in updates, never duplicates.

**Testing**:
- `Integration: check-in QR` → record with `check_in_time`, method.
- `Integration: check-out by non-authorized guardian` → 403.
- `Integration: check-in withdrawn child` → 409.
- `Unit: PIN hash verify` correct/incorrect → true/false.

#### 3.2 — Live roster & ratio computation (WebSocket)

**What**: Real-time per-classroom present-children list and staff:child ratio status pushed over Socket.IO.

**Design**:
- Redis hash `roster:{classroomId}` of present child IDs, updated on every check-in/out event.
- `RatioCalculator.evaluate(classroomId)` → `{ presentChildren, presentStaff, requiredStaff, status: 'ok'|'warning'|'violation' }` using `classrooms.min_staff_ratio` and age-band rules (JSONB `ratio_rules`). `requiredStaff = ceil(presentChildren / ratio)`.
- Socket room per classroom; emits `roster:update` and `ratio:alert` (alert only on `violation`, debounced 30s). Staff present count comes from Phase 7 time-clock when available; until then from active staff schedule.

**Testing**:
- `Unit: ratio calc, 9 children @ 1:4, 2 staff` → requiredStaff 3, status 'violation'.
- `Unit: 8 children @ 1:4, 2 staff` → 'ok'.
- `Integration: check-in pushing over ratio` → `ratio:alert` emitted once (debounce holds the second within 30s).
- `Integration: two API nodes, check-in on node A` → subscriber on node B receives `roster:update` (Redis adapter).

---

## Phase 4: Daily Activities & Parent Feed

### Purpose
Capture the moment-to-moment record of a child's day (meals, naps, diapers, mood, activities) and surface it to families as a real-time feed — the single most-used feature by parents and the primary daily teacher workflow. Establishes the media-upload pipeline reused by observations and messaging.

### Tasks

#### 4.1 — Daily activity logging

**What**: Typed daily-activity capture with bulk (whole-classroom) entry.

**Design**: Adopt Suggestion 1 `daily_activities` DDL (partitioned). Discriminated Zod union per `activity_type` (meal/nap/diaper/potty/activity/mood/sunscreen/medication) so each type validates its own fields. `POST /activities/bulk` records one activity type across a selected child set in a single transaction (e.g., "all napping 1:00–2:30"). Meal entries with a flagged allergen against the child's `child_allergies` raise a 409 warning requiring an `override` flag.

**Testing**:
- `Unit: meal activity missing meal_type` → validation error; `nap` without meal_type → valid.
- `Integration: bulk nap for 10 children` → 10 rows, one transaction.
- `Integration: meal containing a logged allergen without override` → 409 listing allergen.

#### 4.2 — Media pipeline & parent feed

**What**: Photo/video upload pipeline and the family-facing real-time activity feed.

**Design**:
- Direct-to-S3 signed-PUT upload; worker generates thumbnails + transcodes video; `media` rows store `{ type, storageUrl, thumbnailUrl, size, durationMs }`.
- Media inherits `photo_consent`: if absent, the child is excluded from any shared/broadcast media and the upload is flagged private.
- `GET /feed?childId=&cursor=` returns a merged, paginated (RFC 8288 `Link` header) timeline of activities + shared observations + media for the guardian's children only.
- `feed:new` Socket.IO event to subscribed guardians.

**Testing**:
- `Integration: signed-PUT then confirm` → media row, thumbnail job enqueued.
- `Integration: feed for guardian` → only their children's items; other families' excluded.
- `Integration: shared media for no-consent child` → omitted from feed.
- `Unit: cursor pagination` → stable ordering, correct `Link: rel="next"`.

---

## Phase 5: Billing, Invoicing & Payments

### Purpose
Generate recurring tuition invoices, process ACH/card payments through a parent portal, and automate late fees — the revenue engine. Built before subsidy (Phase 8) because subsidy credits apply against invoices defined here. Transactional integrity is paramount (Suggestion 1 "Pros" §4).

### Tasks

#### 5.1 — Billing plans & invoice generation

**What**: Define billing plans, assign them to children, and generate invoices on a schedule via a worker job.

**Design**: Adopt Suggestion 1 DDL for `billing_plans`, `child_billing_assignments`, `invoices`, `invoice_line_items`.
- `BillingRunProcessor` (BullMQ, cron per org timezone) computes, per family, line items from active assignments for the period, applies discounts, and creates a `draft` invoice atomically (one transaction per family).
- `InvoiceCalculator.compute(assignments, period)` is a pure function (unit-testable, no DB): handles flat/daily/hourly rate types, proration on mid-period enrollment, and per-child discounts.
- Invoice status machine: `draft → sent → (paid | partial | overdue) → void`. `due_date` from plan; overdue transition via daily sweep job.

**Testing**:
- `Unit: compute, monthly flat $1200, full period` → $1200; `enrolled on day 15 of 30, daily proration` → $600.
- `Unit: two children, sibling 10% discount on second` → correct subtotal/discount split.
- `Integration: billing run for org` → one draft invoice per family, correct line items, single transaction (partial failure rolls back).
- `Integration: overdue sweep past due_date unpaid` → status `overdue`.

#### 5.2 — Payments, portal & late fees

**What**: Stripe-backed payments, autopay, the parent payment portal, and automated late fees.

**Design**: Adopt Suggestion 1 `payments` DDL.
- `PaymentProvider` interface (`createPaymentIntent`, `chargeSaved`, `setupAutopay`, `refund`); `StripePaymentProvider` is the default impl. PCI scope held to SAQ A via Stripe Elements (card data never touches our servers).
- `POST /payments` records intent; **Stripe webhook** (`payment_intent.succeeded`/`failed`, signature-verified) is the source of truth that transitions `payments.status` and reconciles `invoices.status` + `paid_date`. No client-trusted success.
- Autopay: stored payment method charged by worker on `due_date`.
- Late-fee job: invoices past `due_date + late_fee_grace_days` get a `late_fee` line item and recompute `total_due` (idempotent — never double-charges).

**Testing**:
- `Integration (mocked Stripe): signed webhook payment_intent.succeeded` → payment `completed`, invoice `paid`. `Invalid signature` → 400, no state change.
- `Integration: partial payment` → invoice `partial`, balance correct.
- `Unit: late-fee idempotency, run twice` → exactly one late-fee line item.
- `Integration: refund` → `payments.status = refunded`, invoice reopened.

---

## Phase 6: Developmental Frameworks & Observations

### Purpose
Deliver the pedagogical core and the key differentiator: a configurable framework engine that ingests arbitrary developmental frameworks (NAEYC, HSELOF, EYFS, state standards) as data, and an observation workflow that tags photos/notes to milestones and builds shareable portfolios. This is where the platform out-positions operations-only incumbents.

### Tasks

#### 6.1 — Configurable framework engine

**What**: A pluggable framework library storing hierarchical domains/milestones as data, seeded from bundled JSON definitions.

**Design**: Per Suggestion 3 hybrid: keep `developmental_frameworks` relational, but store the domain/milestone hierarchy as a validated JSONB `structure` tree on the framework (avoiding deep recursive-CTE traversal flagged in Suggestion 1 "Cons" §6) while ALSO denormalizing leaf milestones into a flat `developmental_milestones` table for tagging/indexing.

```prisma
model DevelopmentalFramework {
  id          String  @id @default(uuid()) @db.Uuid
  name        String          // 'NAEYC','Head Start ELOF','EYFS 2025'
  version     String
  countryCode String? @map("country_code")
  stateCode   String? @map("state_code")
  structure   Json            // hierarchical domains→subdomains→milestones
  isActive    Boolean @default(true)
  @@map("developmental_frameworks")
}
```

- `framework-defs` package bundles JSON for NAEYC, HSELOF, and EYFS 2025 (public-domain frameworks per features.md Legal §). A `FrameworkLoader` validates each against a JSON Schema (2020-12) and flattens milestones into `developmental_milestones` on import.
- An org selects an active framework per classroom/age band; this drives which milestones the observation UI surfaces (TeachKloud-style framework-driven prompts).
- `POST /frameworks/import` accepts a custom framework JSON for the long-tail "500+ frameworks" goal.

**Testing**:
- `Unit: FrameworkLoader on bundled NAEYC JSON` → validates, flattens N milestones.
- `Unit: malformed framework (missing milestone id)` → JSON Schema validation error naming the path.
- `Integration: import custom framework` → queryable; milestones tag-able.

#### 6.2 — Observations, milestone tagging, media & portfolio

**What**: Observation capture linked to children, milestones, and media; portfolio timeline generation.

**Design**: Adopt Suggestion 1 DDL for `observations`, `observation_milestones`, `observation_media`.
- `POST /observations` `{ childId, narrative, context, milestoneIds[], proficiency, mediaIds[], shareWithFamily }`.
- `GET /children/:id/portfolio?from=&to=` returns a chronological developmental timeline (observations + tagged milestones + media + proficiency progression), backed by a denormalized read to avoid the multi-join cost flagged in Suggestion 1 "Cons" §2.
- Proficiency progression per milestone computed across observations (`emerging → developing → proficient → exceeding`).
- Shared observations flow into the parent feed (Phase 4) honoring photo consent.

**Testing**:
- `Integration: create observation tagging 3 milestones` → join rows + media linked.
- `Integration: portfolio over date range` → chronological, includes proficiency trend.
- `Integration: tag milestone from inactive framework` → 422.
- `Integration: share observation, child without photo consent on a photo` → text shared, photo withheld.

---

## Phase 7: Staff Management, Scheduling & Time-Clock

### Purpose
Manage the workforce: staff records, credential/certification expiry tracking, scheduling, and a time-clock that feeds real (not scheduled) present-staff counts into the Phase 3 ratio calculator — closing the loop on real-time compliance.

### Tasks

#### 7.1 — Staff, credentials & expiry alerts

**What**: Staff CRUD and credential tracking with proactive expiry alerting.

**Design**: Adopt Suggestion 1 DDL for `staff`, `staff_credentials`. Daily worker job flags credentials expiring within a configurable window (default 30 days) and emits notifications + a `complianceWarnings` feed for directors. Background-check and CPR/first-aid status gate scheduling eligibility.

**Testing**:
- `Integration: credential expiring in 20 days` → appears in warnings; `in 90 days` → does not.
- `Unit: scheduling eligibility, expired background check` → ineligible.

#### 7.2 — Scheduling & time-clock

**What**: Shift scheduling and PIN/QR/geolocation clock-in feeding live ratios.

**Design**: Adopt Suggestion 1 DDL for `staff_schedules`, `staff_time_entries`. Clock-in writes present-staff to Redis `present-staff:{classroomId}`, consumed by `RatioCalculator` (replacing the schedule-based estimate from 3.2). Geolocation clock-in validates within a configurable site radius.

**Testing**:
- `Integration: clock-in updates ratio present-staff` → recomputed ratio reflects real staff.
- `Integration: geolocation outside radius` → rejected.
- `Unit: overlapping shift insert` → unique-constraint conflict surfaced as 409.

---

## Phase 8: Subsidy Billing Rules Engine

### Purpose
Build the headline market differentiator: a configurable, multi-state subsidy rules engine (CCDF/TANF/pre-K vouchers) that computes co-pays, assembles attendance-based claims, and reconciles reimbursements against invoices. Depends on attendance (Phase 3) and billing (Phase 5).

### Tasks

#### 8.1 — Subsidy authorizations & rules engine

**What**: Store authorizations and a data-driven, per-state rules engine computing billable amounts and co-pays.

**Design**: Adopt Suggestion 1 DDL for `subsidy_authorizations`, adding a JSONB `rule_params`. Per Suggestion 1 "Cons" §3, model state rules as data, not tables:

```typescript
interface SubsidyRule {
  state: string;                          // 'CA','TX','NY','FL','IL'
  program: 'CCDF' | 'TANF' | 'pre_k_voucher' | 'head_start';
  billingBasis: 'attendance' | 'enrollment' | 'certificate';
  unit: 'daily' | 'hourly' | 'weekly' | 'monthly';
  absenceRule: { paidAbsenceDaysPerMonth: number };
  copay: { basis: 'family_income' | 'flat'; frequency: 'weekly'|'monthly' };
  attendanceThreshold?: number;           // min % attendance for full reimbursement
}
```

- `SubsidyRuleEngine.computeClaim(auth, attendance, rule)` → `{ billableUnits, claimedAmount, copayDue }` — a pure function tested per state. Bundle the top-5 states (CA, TX, NY, FL, IL) from features.md v1.1 scope.
- Co-pay is applied as an `invoice_line_item` credit/charge against the family's invoice (Phase 5 integration).

**Testing**:
- `Unit: CA CCDF, 20 attended + 3 absent (cap 5)` → 23 billable days.
- `Unit: TX with 10% attendance shortfall below threshold` → prorated reimbursement.
- `Unit: monthly co-pay applied to invoice` → invoice `subsidy_credit` and co-pay line correct.
- `Unit: unknown state` → explicit `UnsupportedStateError`.

#### 8.2 — Claim assembly, submission & reconciliation

**What**: Assemble claims per period, export submission files, and reconcile reimbursements.

**Design**: Adopt Suggestion 1 `subsidy_claims` DDL. Claim status machine `draft → submitted → (approved|denied) → paid | appealed`. Since no national submission standard exists (standards.md), export is a per-state adapter producing the state's file format (CSV/XML/Ed-Fi-aligned JSON) behind a `SubsidySubmissionAdapter` interface; manual upload to state portals plus recorded confirmation. Reconciliation diffs `reimbursed_amount` vs `claimed_amount` and flags shortfalls.

**Testing**:
- `Unit: claim assembly from attendance + rule` → correct days/hours/amount.
- `Integration: CA adapter export` → file matches CA fixture format.
- `Integration: reimbursement < claimed` → shortfall flagged for review.

---

## Phase 9: Compliance, Incidents, Medication & CACFP

### Purpose
Satisfy the regulatory surface licensing inspectors check: incident reports, medication administration records, illness logs, CACFP meal tracking with claim generation, and audit-ready licensing reports. Largely independent of Phase 8, so parallelizable.

### Tasks

#### 9.1 — Incidents, medication & health logs

**What**: Incident reports with parent acknowledgement and medication administration records.

**Design**: Adopt Suggestion 1 DDL for `incident_reports`, `medication_logs`. Incident creation triggers a parent notification; guardian e-acknowledgement captured (signature URL + timestamp). Medication administration requires a referenced parent authorization; "five rights" validation (right child/med/dose/time/route) enforced in the DTO.

**Testing**:
- `Integration: serious incident` → parent notified, awaits acknowledgement; acknowledgement recorded.
- `Integration: medication without authorization ref` → 422.
- `Unit: dose outside authorized range` → validation error.

#### 9.2 — CACFP meal tracking & claims; licensing reports

**What**: CACFP-compliant meal recording, monthly claim generation, and audit-ready licensing reports.

**Design**: Adopt Suggestion 1 DDL for `meal_records`, `meal_attendance`. Meal recording cross-references attendance (only present children claimable). `CacfpClaimBuilder` aggregates monthly meal counts by reimbursement tier per USDA rules; records retained 3 years (retention policy job). Licensing report generator produces ratio-history, incident-summary, and credential-status PDFs.

**Testing**:
- `Unit: monthly CACFP claim from meal_attendance` → counts by meal type match.
- `Integration: meal claimed for absent child` → excluded.
- `Integration: licensing report for date range` → PDF with ratio history + incidents.

---

## Phase 10: AI Augmentation

### Purpose
Layer the AI-native advantages on top of the now-complete data substrate: observation narrative generation, vision-AI developmental-domain tagging (absent from every incumbent), lesson planning, billing/attendance anomaly detection, and ratio forecasting. Each is an additive worker job over existing data.

### Tasks

#### 10.1 — Observation narrative generation & vision domain tagging

**What**: Generate/enhance observation narratives from teacher bullet notes, and auto-suggest developmental-domain tags from photos.

**Design**:
- `AiProvider` abstraction (Vercel AI SDK) with documented child-PII data-processor boundary; `ai_processing` consent required (Phase 1 ConsentGuard).
- Narrative prompt (structure):
  - *System*: "You are an early-childhood educator writing a developmental observation aligned to {frameworkName}. Write 2–4 sentences, objective, strengths-based, no diagnosis."
  - *User*: "{teacherBulletNotes} | Child age band: {band} | Candidate milestones: {milestones}."
  - Output stored in `observations.ai_narrative`; teacher edits/approves before sharing (never auto-published).
- Vision tagging: photo → vision model → suggested `milestoneIds[]` (within the classroom's active framework) returned as suggestions only, never auto-applied.

**Testing**:
- `Integration (mocked AI): bullet notes` → `ai_narrative` populated; original `narrative` preserved.
- `Integration: vision tagging` → suggestions restricted to active-framework milestones.
- `Integration: AI route without ai_processing consent` → 403.
- `Unit: prompt builder` → includes framework name + age band.

#### 10.2 — Anomaly detection, lesson planning & ratio forecasting

**What**: Flag billing/attendance anomalies, generate lesson plans, and forecast ratio violations.

**Design**:
- Billing anomaly detector (worker): statistical baseline per family/child (e.g., sudden attendance drop, recurring payment failures, subsidy claim outliers) → director `anomalies` feed with an AI-generated explanation.
- Lesson planner: given observed child interests (aggregated from observation context) + age band + framework, generates an activity plan referencing target milestones.
- Ratio forecasting: time-series over historical `attendance_records` predicts hours likely to breach ratio; surfaces a staffing-suggestion alert ahead of time.

**Testing**:
- `Unit: anomaly detector, 3 consecutive payment failures` → flagged.
- `Unit: attendance forecast on seasonal fixture` → predicts known afternoon dip.
- `Integration (mocked AI): lesson plan request` → plan references in-framework milestones for the age band.

---

## Phase 11: Open API, Webhooks & State Reporting Interop

### Purpose
Ship the structural differentiator no major incumbent (bar Illumine) offers: a documented public REST API with OAuth2 client-credentials, outbound webhooks, and Ed-Fi/CEDS-aligned state reporting export. Turns the platform into an integration hub.

### Tasks

#### 11.1 — Public API, OAuth2 clients & webhooks

**What**: API-key/OAuth2 client-credentials access for integrators, rate limiting, and signed outbound webhooks.

**Design**:
- `api_clients` (per-org) with OAuth 2.0 client-credentials flow (RFC 6749) issuing scoped JWTs; scopes map to read/write per module.
- The same generated OpenAPI 3.1 doc (Phase 1.4) is the public contract; published `/docs`.
- Outbound webhooks: `webhook_subscriptions` `{ url, events[], secret }`; `WebhookDeliveryProcessor` (BullMQ) signs payloads (HMAC-SHA256), retries with exponential backoff, records delivery attempts. Events: `child.enrolled`, `invoice.paid`, `attendance.checked_in`, `observation.created`, etc.
- Redis-backed token-bucket rate limiting per client.

**Testing**:
- `Integration: client-credentials token, scoped read` → allowed; out-of-scope write → 403.
- `Integration: invoice.paid event` → signed webhook POSTed; HMAC verifies; failing endpoint retried per backoff.
- `Integration: exceed rate limit` → 429 with `Retry-After`.

#### 11.2 — Ed-Fi / CEDS export & FHIR immunization pathway

**What**: State-reporting export mapping platform entities to Ed-Fi v6.0 / CEDS, and a FHIR R5 Immunization ingest pathway.

**Design**:
- `EdFiMapper` maps `children`/`attendance_records`/`developmental_milestones` to Ed-Fi v6.0 resources; `GET /reporting/edfi?from=&to=` emits Ed-Fi-conformant JSON validated against the published Ed-Fi OpenAPI schema.
- CEDS element mapping table for state-specific reports.
- FHIR pathway: `POST /health/fhir/immunization` accepts FHIR R5 `Immunization` resources, stores into `child_medical_records`, and flags incomplete immunization at enrollment (the standards.md emerging opportunity).

**Testing**:
- `Integration: Ed-Fi export` → validates against Ed-Fi v6.0 schema.
- `Unit: child → CEDS element mapping` → correct element IDs.
- `Integration: FHIR Immunization ingest` → medical record created; incomplete schedule flags enrollment.

---

## Phase 12: Offline-First Mobile & Sync

### Purpose
Deliver the offline-first mobile architecture that no incumbent has — the staff capture app and parent app that work through unreliable classroom Wi-Fi with background sync. This is sequenced last because it depends on stable API contracts from all prior phases, but the sync endpoints it consumes can be built once the contracts settle.

### Tasks

#### 12.1 — Sync protocol (server)

**What**: Pull/push sync endpoints with conflict resolution for offline capture.

**Design**:
- WatermelonDB-compatible sync: `GET /sync/pull?lastPulledAt=` returns `{ changes: { created, updated, deleted }, timestamp }` per syncable table (attendance, daily_activities, observations, media metadata). `POST /sync/push` applies client changes with last-write-wins on server timestamp, except billing/payment entities which are server-authoritative and never client-writable.
- Media uploaded separately via signed URLs with client-generated UUIDs for offline-created references (no server round-trip needed to create the local record).
- Idempotency keys on push to tolerate retried syncs.

**Testing**:
- `Integration: pull since timestamp` → only changed rows.
- `Integration: push offline-created observations` → persisted, dedup by idempotency key on retry.
- `Integration: conflicting concurrent edits` → last-write-wins by server timestamp; conflict logged.

#### 12.2 — Mobile apps (staff capture + parent)

**What**: Expo React Native apps with local WatermelonDB store and background sync.

**Design**:
- Shared Zod schemas + generated API client from `packages/*`.
- Staff app: offline observation/activity/check-in capture, camera-first photo flow, queued media upload on reconnect, battery-conscious sync interval.
- Parent app: feed, messaging, payments (Stripe RN SDK), portfolio, consent management.
- Background sync via Expo `BackgroundTask`; optimistic UI from the local store.

**Testing**:
- `E2E (Detox/Maestro): airplane-mode capture 3 observations → reconnect` → all sync, media uploads, appear server-side.
- `Integration: parent payment via RN Stripe` → payment intent + webhook reconciliation.
- `Unit: local-store optimistic write then sync failure` → retained, retried, no data loss.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (tenancy, auth, contracts, audit)   ─── required by everything
    │
Phase 2: Children, Families & Enrollment                ─── requires P1
    │
    ├── Phase 3: Attendance & Live Roster                ─── requires P2
    │      ├── Phase 4: Daily Activities & Parent Feed   ─── requires P3 (media pipeline)
    │      └── Phase 7: Staff, Scheduling & Time-Clock   ─── requires P3 (feeds ratio); ∥ with P4
    │
    ├── Phase 5: Billing, Invoicing & Payments           ─── requires P2; ∥ with P3/P4
    │      └── Phase 8: Subsidy Rules Engine              ─── requires P3 (attendance) + P5
    │
    └── Phase 6: Frameworks & Observations               ─── requires P2 + P4 (feed/media); ∥ with P5
           └── Phase 10: AI Augmentation                 ─── requires P4, P5, P6 (data substrate)

Phase 9: Compliance, Incidents, Medication & CACFP       ─── requires P3 (attendance/meals); ∥ with P5–P8
Phase 11: Open API, Webhooks & State Reporting Interop   ─── requires stable contracts (after P6); ∥ with P9/P10
Phase 12: Offline-First Mobile & Sync                    ─── requires settled contracts (P3,P4,P6 minimum)
```

**Parallelism opportunities**
- After Phase 2: Phases **3, 5, and 6** can proceed concurrently (distinct modules; 6 needs 4's media pipeline so couple 4 with 6, or stub media).
- After Phase 3: Phases **4 and 7** in parallel.
- Phase **9** can be built any time after Phase 3, parallel to the 5→8 billing track.
- Phases **10, 11** can be developed in parallel once their data dependencies land.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass; new code paths covered (happy-path + edge cases listed in each task).
3. Integration tests touching the DB run against a **real Postgres via Testcontainers** (RLS, migrations, partitions, and constraints are not mocked).
4. ESLint and Prettier pass; `tsc --noEmit` (strict) passes across the workspace.
5. New/changed Prisma migrations created, including any raw-SQL RLS/partition steps, and apply cleanly from an empty database.
6. New API endpoints appear in the generated `/openapi.json` (OpenAPI 3.1, schema-validated) and the typed `api-client` regenerates without errors.
7. Every mutating endpoint writes an `audit_logs` entry (FERPA/COPPA trail); routes touching child PII enforce the appropriate consent scope.
8. `docker compose up` builds and runs the full stack (api, worker, web, postgres, redis, minio) healthily.
9. Feature works end-to-end (HTTP via Supertest; web flows via Playwright; mobile flows via Detox/Maestro where applicable).
10. New configuration/environment variables documented in `.env.example`.
11. Tenant isolation verified: no new table or query path can return cross-organization data (negative RLS test included).
```
