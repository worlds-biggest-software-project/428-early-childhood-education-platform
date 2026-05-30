# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Early Childhood Education Platform (428)
> Approach: Fully normalized relational schema with PostgreSQL
> Generated: 2026-05-25

---

## Summary

A traditional normalized relational database design using PostgreSQL, with shared-schema multi-tenancy enforced via Row-Level Security (RLS). Every entity is stored in its own table with foreign key relationships, conforming to Third Normal Form (3NF). This approach prioritizes referential integrity, transactional consistency, and compatibility with education data standards (Ed-Fi, CEDS).

---

## Multi-Tenancy Strategy

Every major table includes an `organization_id` column referencing the top-level childcare organization (operator). PostgreSQL Row-Level Security policies filter all queries by `organization_id`, set at the connection level via `SET app.current_tenant`. This provides strong data isolation without the operational overhead of schema-per-tenant.

```sql
-- RLS policy applied to every tenant-scoped table
ALTER TABLE children ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON children
  USING (organization_id = current_setting('app.current_tenant')::uuid);
```

---

## Core Entities and Schema

### Organizations and Sites

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL CHECK (type IN ('single_site','multi_site','head_start','district')),
    tax_id          TEXT,
    address_line1   TEXT,
    address_line2   TEXT,
    city            TEXT,
    state_code      CHAR(2),
    postal_code     TEXT,
    country_code    CHAR(2) DEFAULT 'US',
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE sites (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    license_number  TEXT,
    license_expiry  DATE,
    capacity        INTEGER,
    address_line1   TEXT,
    city            TEXT,
    state_code      CHAR(2),
    postal_code     TEXT,
    phone           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE classrooms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    name            TEXT NOT NULL,
    age_group       TEXT, -- e.g. 'infant','toddler','preschool','pre-k'
    max_capacity    INTEGER NOT NULL,
    min_staff_ratio NUMERIC(3,1) NOT NULL, -- e.g. 4.0 means 1:4
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Children and Families

```sql
CREATE TABLE families (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    primary_language TEXT DEFAULT 'en',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE guardians (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT,
    phone           TEXT,
    relationship    TEXT NOT NULL, -- 'mother','father','guardian','grandparent'
    is_primary      BOOLEAN DEFAULT false,
    is_emergency    BOOLEAN DEFAULT false,
    can_pickup      BOOLEAN DEFAULT true,
    user_account_id UUID, -- links to auth system
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE children (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    date_of_birth   DATE NOT NULL,
    gender          TEXT,
    enrollment_status TEXT NOT NULL DEFAULT 'waitlisted'
        CHECK (enrollment_status IN ('waitlisted','enrolled','withdrawn','graduated')),
    enrollment_date DATE,
    withdrawal_date DATE,
    photo_consent   BOOLEAN DEFAULT false,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE child_classroom_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    classroom_id    UUID NOT NULL REFERENCES classrooms(id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    schedule_type   TEXT DEFAULT 'full_time', -- 'full_time','part_time','drop_in'
    UNIQUE (child_id, classroom_id, start_date)
);

CREATE TABLE child_allergies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    allergen        TEXT NOT NULL,
    severity        TEXT CHECK (severity IN ('mild','moderate','severe','life_threatening')),
    action_plan     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE child_medical_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    record_type     TEXT NOT NULL, -- 'immunization','condition','medication','physician'
    description     TEXT NOT NULL,
    effective_date  DATE,
    expiry_date     DATE,
    document_url    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Attendance and Daily Activities

```sql
CREATE TABLE attendance_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    date            DATE NOT NULL,
    check_in_time   TIMESTAMPTZ,
    check_out_time  TIMESTAMPTZ,
    check_in_method TEXT, -- 'qr_code','pin','manual','contactless'
    checked_in_by   UUID REFERENCES guardians(id),
    checked_out_by  UUID REFERENCES guardians(id),
    absence_reason  TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (child_id, date)
);

CREATE TABLE daily_activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    recorded_by     UUID NOT NULL, -- staff user id
    activity_type   TEXT NOT NULL
        CHECK (activity_type IN ('meal','nap','diaper','potty','activity','mood','sunscreen','medication')),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    details         TEXT,
    -- meal-specific
    meal_type       TEXT, -- 'breakfast','am_snack','lunch','pm_snack','dinner'
    food_items      TEXT,
    portion_eaten   TEXT, -- 'none','some','most','all'
    -- nap-specific
    nap_start       TIMESTAMPTZ,
    nap_end         TIMESTAMPTZ,
    -- diaper-specific
    diaper_status   TEXT, -- 'wet','soiled','dry'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Developmental Frameworks and Observations

```sql
CREATE TABLE developmental_frameworks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL, -- 'NAEYC','EYFS','Head Start ELOF','HighScope COR'
    version         TEXT,
    country_code    CHAR(2),
    state_code      CHAR(2),
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE developmental_domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework_id    UUID NOT NULL REFERENCES developmental_frameworks(id),
    parent_id       UUID REFERENCES developmental_domains(id), -- hierarchy
    name            TEXT NOT NULL,
    description     TEXT,
    sort_order      INTEGER DEFAULT 0
);

CREATE TABLE developmental_milestones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES developmental_domains(id),
    name            TEXT NOT NULL,
    description     TEXT,
    age_range_start INTEGER, -- months
    age_range_end   INTEGER, -- months
    sort_order      INTEGER DEFAULT 0
);

CREATE TABLE observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    observer_id     UUID NOT NULL, -- staff user id
    observed_at     TIMESTAMPTZ NOT NULL,
    narrative       TEXT, -- teacher's written observation
    ai_narrative    TEXT, -- AI-enhanced version
    context         TEXT, -- where/during what activity
    is_shared_with_family BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE observation_milestones (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    observation_id  UUID NOT NULL REFERENCES observations(id),
    milestone_id    UUID NOT NULL REFERENCES developmental_milestones(id),
    proficiency     TEXT CHECK (proficiency IN ('emerging','developing','proficient','exceeding')),
    UNIQUE (observation_id, milestone_id)
);

CREATE TABLE observation_media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    observation_id  UUID NOT NULL REFERENCES observations(id),
    media_type      TEXT NOT NULL CHECK (media_type IN ('photo','video','audio')),
    storage_url     TEXT NOT NULL,
    thumbnail_url   TEXT,
    file_size_bytes BIGINT,
    caption         TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Billing, Payments, and Subsidies

```sql
CREATE TABLE billing_plans (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    name            TEXT NOT NULL,
    billing_cycle   TEXT NOT NULL CHECK (billing_cycle IN ('weekly','biweekly','monthly')),
    base_rate       NUMERIC(10,2) NOT NULL,
    rate_type       TEXT NOT NULL CHECK (rate_type IN ('flat','hourly','daily')),
    late_fee_amount NUMERIC(10,2) DEFAULT 0,
    late_fee_grace_days INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE child_billing_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    billing_plan_id UUID NOT NULL REFERENCES billing_plans(id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    discount_pct    NUMERIC(5,2) DEFAULT 0,
    notes           TEXT
);

CREATE TABLE invoices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    invoice_number  TEXT NOT NULL,
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    subtotal        NUMERIC(10,2) NOT NULL,
    discount        NUMERIC(10,2) DEFAULT 0,
    subsidy_credit  NUMERIC(10,2) DEFAULT 0,
    late_fee        NUMERIC(10,2) DEFAULT 0,
    total_due       NUMERIC(10,2) NOT NULL,
    status          TEXT NOT NULL DEFAULT 'draft'
        CHECK (status IN ('draft','sent','paid','partial','overdue','void')),
    due_date        DATE NOT NULL,
    paid_date       DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE invoice_line_items (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    invoice_id      UUID NOT NULL REFERENCES invoices(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    description     TEXT NOT NULL,
    quantity        NUMERIC(10,2) DEFAULT 1,
    unit_price      NUMERIC(10,2) NOT NULL,
    total           NUMERIC(10,2) NOT NULL,
    category        TEXT -- 'tuition','registration','supplies','field_trip','late_fee'
);

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    invoice_id      UUID REFERENCES invoices(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    amount          NUMERIC(10,2) NOT NULL,
    payment_method  TEXT NOT NULL CHECK (payment_method IN ('ach','credit_card','check','cash','subsidy')),
    payment_gateway_ref TEXT,
    status          TEXT NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending','completed','failed','refunded')),
    processed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subsidy_authorizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    program_type    TEXT NOT NULL, -- 'CCDF','TANF','pre_k_voucher','head_start','employer'
    state_code      CHAR(2),
    authorization_number TEXT,
    authorized_hours_per_week NUMERIC(5,1),
    daily_rate      NUMERIC(10,2),
    weekly_rate     NUMERIC(10,2),
    monthly_rate    NUMERIC(10,2),
    copay_amount    NUMERIC(10,2),
    copay_frequency TEXT, -- 'weekly','monthly'
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT DEFAULT 'active'
        CHECK (status IN ('pending','active','suspended','expired','terminated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE subsidy_claims (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    authorization_id UUID NOT NULL REFERENCES subsidy_authorizations(id),
    claim_period_start DATE NOT NULL,
    claim_period_end   DATE NOT NULL,
    attendance_days    INTEGER,
    attendance_hours   NUMERIC(6,1),
    claimed_amount     NUMERIC(10,2) NOT NULL,
    reimbursed_amount  NUMERIC(10,2),
    status          TEXT DEFAULT 'draft'
        CHECK (status IN ('draft','submitted','approved','denied','paid','appealed')),
    submitted_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Staff Management

```sql
CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    site_id         UUID REFERENCES sites(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT NOT NULL,
    phone           TEXT,
    role            TEXT NOT NULL, -- 'teacher','assistant','director','admin','cook','driver'
    hire_date       DATE,
    termination_date DATE,
    hourly_rate     NUMERIC(8,2),
    user_account_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff_credentials (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    staff_id        UUID NOT NULL REFERENCES staff(id),
    credential_type TEXT NOT NULL, -- 'cpr','first_aid','cda','state_training','background_check'
    credential_name TEXT NOT NULL,
    issued_date     DATE,
    expiry_date     DATE,
    document_url    TEXT,
    hours           NUMERIC(6,1),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE staff_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    staff_id        UUID NOT NULL REFERENCES staff(id),
    classroom_id    UUID REFERENCES classrooms(id),
    date            DATE NOT NULL,
    shift_start     TIME NOT NULL,
    shift_end       TIME NOT NULL,
    status          TEXT DEFAULT 'scheduled'
        CHECK (status IN ('scheduled','confirmed','completed','absent','swapped')),
    UNIQUE (staff_id, date, shift_start)
);

CREATE TABLE staff_time_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    staff_id        UUID NOT NULL REFERENCES staff(id),
    clock_in        TIMESTAMPTZ NOT NULL,
    clock_out       TIMESTAMPTZ,
    clock_in_method TEXT, -- 'pin','qr_code','geolocation','manual'
    break_minutes   INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Incidents and Compliance

```sql
CREATE TABLE incident_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    reported_by     UUID NOT NULL REFERENCES staff(id),
    incident_type   TEXT NOT NULL, -- 'injury','illness','behavioral','medication_error','allergy_reaction'
    occurred_at     TIMESTAMPTZ NOT NULL,
    description     TEXT NOT NULL,
    action_taken    TEXT,
    parent_notified BOOLEAN DEFAULT false,
    parent_notified_at TIMESTAMPTZ,
    guardian_signature_url TEXT,
    severity        TEXT CHECK (severity IN ('minor','moderate','serious')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE medication_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    administered_by UUID NOT NULL REFERENCES staff(id),
    medication_name TEXT NOT NULL,
    dosage          TEXT NOT NULL,
    administered_at TIMESTAMPTZ NOT NULL,
    parent_authorization_ref TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### CACFP Meal Tracking

```sql
CREATE TABLE meal_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    date            DATE NOT NULL,
    meal_type       TEXT NOT NULL CHECK (meal_type IN ('breakfast','am_snack','lunch','pm_snack','dinner')),
    menu_items      TEXT NOT NULL,
    children_served INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE meal_attendance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meal_record_id  UUID NOT NULL REFERENCES meal_records(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    served          BOOLEAN DEFAULT true,
    portion_eaten   TEXT, -- 'none','some','most','all'
    UNIQUE (meal_record_id, child_id)
);
```

### Messaging and Communication

```sql
CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    thread_id       UUID,
    sender_type     TEXT NOT NULL CHECK (sender_type IN ('staff','guardian')),
    sender_id       UUID NOT NULL,
    recipient_type  TEXT NOT NULL CHECK (recipient_type IN ('family','staff','classroom','site','broadcast')),
    recipient_id    UUID, -- null for broadcast
    subject         TEXT,
    body            TEXT NOT NULL,
    is_translated   BOOLEAN DEFAULT false,
    original_language TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE message_attachments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_id      UUID NOT NULL REFERENCES messages(id),
    media_type      TEXT NOT NULL,
    storage_url     TEXT NOT NULL,
    file_name       TEXT,
    file_size_bytes BIGINT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Key Indexes

```sql
-- Core query patterns
CREATE INDEX idx_children_org_status ON children (organization_id, enrollment_status);
CREATE INDEX idx_children_family ON children (family_id);
CREATE INDEX idx_attendance_child_date ON attendance_records (child_id, date DESC);
CREATE INDEX idx_attendance_site_date ON attendance_records (site_id, date);
CREATE INDEX idx_observations_child ON observations (child_id, observed_at DESC);
CREATE INDEX idx_invoices_family_status ON invoices (family_id, status);
CREATE INDEX idx_daily_activities_child ON daily_activities (child_id, recorded_at DESC);
CREATE INDEX idx_staff_schedules_date ON staff_schedules (date, classroom_id);
CREATE INDEX idx_subsidy_auth_child ON subsidy_authorizations (child_id, status);
CREATE INDEX idx_messages_recipient ON messages (recipient_type, recipient_id, created_at DESC);
```

---

## Pros

1. **Referential integrity everywhere.** Foreign keys enforce that every observation links to a real child, every invoice to a real family, and every subsidy claim to a valid authorization. This prevents orphaned records, which is critical when dealing with financial and regulatory data.

2. **Standards alignment.** The normalized entity structure maps cleanly to Ed-Fi and CEDS data elements, making state reporting and interoperability straightforward. Entities like `children`, `attendance_records`, and `developmental_milestones` correspond directly to CEDS Early Learning data elements.

3. **Mature tooling and ecosystem.** PostgreSQL has the broadest ecosystem of ORMs, migration tools, monitoring, backup solutions, and hosting options. Every framework (Rails, Django, Next.js/Prisma, Spring Boot) has first-class PostgreSQL support.

4. **ACID transactions for billing.** Financial operations (invoice generation, payment recording, subsidy reconciliation) benefit from strict transactional guarantees. A partially applied invoice or orphaned payment is a serious business problem.

5. **Row-Level Security for multi-tenancy.** PostgreSQL RLS provides database-enforced tenant isolation without the complexity of separate schemas or databases per center. A single shared schema simplifies migrations and deployments.

6. **Well-understood scaling path.** Read replicas for reporting queries, connection pooling (PgBouncer), table partitioning (e.g., partitioning `attendance_records` by month), and eventual migration to Citus for horizontal sharding are all well-documented paths.

---

## Cons

1. **Schema rigidity.** Adding a new field to an entity requires a migration. In a domain where developmental frameworks vary by state and regulatory requirements change frequently, this creates ongoing migration overhead. Supporting 500+ curriculum frameworks (as TeachKloud does) with purely relational tables is cumbersome.

2. **Complex queries for developmental timelines.** Generating a child's developmental portfolio -- spanning observations, milestones, media, and daily activities over months -- requires joining many tables. These queries can become expensive at scale without careful denormalization.

3. **Subsidy rules engine awkwardness.** State-specific subsidy billing rules (50+ variations) are difficult to model purely relationally. The rules themselves are procedural (if/then logic varying by state), and storing them as relational data leads to an explosion of configuration tables.

4. **Media metadata bloat.** Photos and videos generate high-volume metadata rows. The `observation_media` and `daily_activities` tables can grow extremely large in a center that captures 50-100 photos per day per classroom.

5. **Migration burden across hundreds of tenants.** Schema migrations must be applied atomically across all tenants in a shared-schema model. Large ALTER TABLE operations on tables with millions of rows (e.g., `attendance_records`, `daily_activities`) can cause downtime.

6. **No native support for hierarchical framework traversal.** Developmental frameworks have deep hierarchies (framework > domain > subdomain > indicator > milestone). Recursive CTE queries work but are not as performant as graph or document stores for deep traversals.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ |
| Multi-tenancy | Row-Level Security with `organization_id` |
| ORM | Prisma (TypeScript) or SQLAlchemy (Python) |
| Migrations | Prisma Migrate, Flyway, or Alembic |
| Connection pooling | PgBouncer or Supavisor |
| Search | PostgreSQL full-text search for observations/messages; Meilisearch for parent-facing search |
| File storage | S3-compatible object storage (AWS S3, Cloudflare R2) with signed URLs |
| Caching | Redis for session data and real-time classroom rosters |

---

## Migration and Scaling Considerations

- **Partitioning strategy:** Partition `attendance_records`, `daily_activities`, `observations`, and `messages` by month using PostgreSQL declarative partitioning. This keeps query performance stable as data grows over years.
- **Archive policy:** COPPA and FERPA require defined retention periods. Implement automated archival of child records after withdrawal + retention period, moving data to cold storage (S3 + Parquet) while maintaining referential integrity stubs.
- **Read replicas:** Route reporting and analytics queries (CACFP reports, developmental progress summaries, billing analytics) to read replicas to avoid impacting operational workloads.
- **Sharding readiness:** The `organization_id` on every table is a natural shard key. If scaling beyond a single PostgreSQL instance, Citus or similar distributed PostgreSQL can shard by organization without schema changes.
- **Zero-downtime migrations:** Use `pg_repack` or online DDL tools to avoid locking large tables during schema changes. For column additions, use `ALTER TABLE ... ADD COLUMN ... DEFAULT` which is non-blocking in PostgreSQL 11+.
