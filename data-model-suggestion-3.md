# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Early Childhood Education Platform (428)
> Approach: PostgreSQL with strategic JSONB columns for flexible domains
> Generated: 2026-05-25

---

## Summary

A hybrid architecture that uses normalized relational tables for entities with stable, well-defined schemas (organizations, children, families, staff, invoices, payments) and PostgreSQL JSONB columns for domains with high variability and tenant-specific customization (developmental frameworks, observation data, subsidy rules, daily activity details, curriculum configurations). This approach captures the integrity benefits of relational modeling where it matters most (billing, attendance, compliance) while providing schema-flexible storage for the domains that vary across states, frameworks, and individual center preferences.

This is the "pragmatic middle ground" -- it avoids the rigidity of pure normalization for inherently variable data and avoids the complexity of full event sourcing, while still using a single database engine (PostgreSQL) that the team already knows.

---

## Design Principles

1. **Relational for money and compliance.** Billing, payments, subsidies, attendance, and staff time entries use fully normalized tables with foreign keys and constraints. These domains have regulatory and financial consequences for data errors.

2. **JSONB for variability.** Developmental frameworks, observation details, daily activity specifics, subsidy rule configurations, and center-specific form fields use JSONB columns. These domains change frequently, vary by tenant, and benefit from schema flexibility.

3. **GIN indexes on JSONB.** Every JSONB column used in queries gets a GIN index, ensuring that document-style queries remain performant.

4. **Immutable observation log.** Observations and daily activities are append-only, providing a natural timeline without the full complexity of event sourcing.

5. **Multi-tenancy via RLS.** Shared-schema with `organization_id` and PostgreSQL Row-Level Security, identical to the normalized approach.

---

## Core Schema

### Organizations, Sites, and Classrooms

These entities are stable and well-defined. They use standard relational tables with a JSONB `settings` column for per-tenant configuration.

```sql
CREATE TABLE organizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    type            TEXT NOT NULL CHECK (type IN ('single_site','multi_site','head_start','district')),
    tax_id          TEXT,
    timezone        TEXT NOT NULL DEFAULT 'America/New_York',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "billing_defaults": {"cycle": "monthly", "late_fee": 25.00, "grace_days": 5},
    --   "attendance": {"check_in_methods": ["qr_code","pin"], "require_checkout": true},
    --   "communication": {"default_language": "en", "auto_translate": true},
    --   "branding": {"logo_url": "...", "primary_color": "#4A90D9"}
    -- }
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
    address         JSONB NOT NULL DEFAULT '{}',
    -- address: {"line1": "...", "line2": "...", "city": "...", "state": "TX", "postal": "78701", "country": "US"}
    phone           TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- site-level overrides for organization defaults
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE classrooms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    name            TEXT NOT NULL,
    age_group       TEXT,
    max_capacity    INTEGER NOT NULL,
    ratio_requirement JSONB NOT NULL,
    -- ratio_requirement: {"min_ratio": 4.0, "state_code": "TX", "age_range": "12-24m"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Children and Families (Relational Core + JSONB Extensions)

```sql
CREATE TABLE families (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    primary_language TEXT DEFAULT 'en',
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- custom_fields: per-center custom enrollment fields
    -- e.g. {"employer": "Acme Corp", "referred_by": "Google", "custody_notes": "..."}
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
    relationship    TEXT NOT NULL,
    is_primary      BOOLEAN DEFAULT false,
    is_emergency    BOOLEAN DEFAULT false,
    can_pickup      BOOLEAN DEFAULT true,
    user_account_id UUID,
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
    -- Health and medical data as JSONB (varies by state requirements)
    health_info     JSONB NOT NULL DEFAULT '{}',
    -- health_info: {
    --   "allergies": [{"allergen": "peanuts", "severity": "severe", "action_plan": "EpiPen in office"}],
    --   "conditions": [{"name": "asthma", "medications": ["albuterol inhaler"]}],
    --   "immunizations": [{"vaccine": "DTaP", "date": "2024-03-15", "verified": true}],
    --   "physician": {"name": "Dr. Smith", "phone": "512-555-0100"},
    --   "dietary_restrictions": ["dairy-free"]
    -- }
    custom_fields   JSONB NOT NULL DEFAULT '{}',
    -- custom_fields: per-center enrollment fields
    -- e.g. {"t_shirt_size": "4T", "swim_level": "beginner", "transportation": "bus_route_3"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_children_health_allergies ON children
    USING GIN ((health_info -> 'allergies'));

CREATE TABLE child_classroom_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    classroom_id    UUID NOT NULL REFERENCES classrooms(id),
    start_date      DATE NOT NULL,
    end_date        DATE,
    schedule        JSONB NOT NULL DEFAULT '{}',
    -- schedule: {
    --   "type": "part_time",
    --   "days": ["mon","wed","fri"],
    --   "drop_off_window": "07:30-09:00",
    --   "pickup_window": "15:00-17:30"
    -- }
    UNIQUE (child_id, classroom_id, start_date)
);
```

### Attendance (Fully Relational -- Compliance Critical)

```sql
CREATE TABLE attendance_records (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    classroom_id    UUID REFERENCES classrooms(id),
    date            DATE NOT NULL,
    check_in_time   TIMESTAMPTZ,
    check_out_time  TIMESTAMPTZ,
    check_in_method TEXT,
    checked_in_by   UUID REFERENCES guardians(id),
    checked_out_by  UUID REFERENCES guardians(id),
    absence_reason  TEXT,
    total_hours     NUMERIC(4,1) GENERATED ALWAYS AS (
        EXTRACT(EPOCH FROM (check_out_time - check_in_time)) / 3600.0
    ) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (child_id, date)
) PARTITION BY RANGE (date);

-- Create monthly partitions
-- CREATE TABLE attendance_2026_05 PARTITION OF attendance_records
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Daily Activities (JSONB Details with Relational Skeleton)

Daily activities have a common skeleton (who, when, what type) but highly variable details depending on the activity type. JSONB for the details avoids a wide table with mostly-null columns.

```sql
CREATE TABLE daily_activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    recorded_by     UUID NOT NULL,
    activity_type   TEXT NOT NULL
        CHECK (activity_type IN ('meal','nap','diaper','potty','activity','mood','medication','sunscreen','note')),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    details         JSONB NOT NULL,
    -- details varies by activity_type:
    --
    -- meal:       {"meal_type": "lunch", "food_items": ["chicken","rice","broccoli"],
    --              "portion_eaten": "most", "drink": "milk"}
    -- nap:        {"start_time": "12:30", "end_time": "14:15", "duration_minutes": 105,
    --              "quality": "restful"}
    -- diaper:     {"status": "wet", "rash_noted": false}
    -- potty:      {"success": true, "prompted": false}
    -- medication: {"name": "Tylenol", "dosage": "5ml", "parent_auth_ref": "MED-2026-001",
    --              "time_given": "10:30"}
    -- activity:   {"description": "Painted with watercolors", "learning_areas": ["creative","fine_motor"]}
    -- mood:       {"mood": "happy", "notes": "Excited about art project"}
    --
    shared_with_family BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_daily_act_child_date ON daily_activities (child_id, recorded_at DESC);
CREATE INDEX idx_daily_act_type ON daily_activities (activity_type, recorded_at DESC);
CREATE INDEX idx_daily_act_details ON daily_activities USING GIN (details);
```

### Developmental Frameworks (JSONB-Heavy -- High Variability)

Developmental frameworks are the most variable domain. Supporting NAEYC, EYFS, HighScope COR, Head Start ELOF, 50 state standards, Montessori, and custom center frameworks requires a schema-flexible approach.

```sql
CREATE TABLE developmental_frameworks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            TEXT NOT NULL UNIQUE, -- 'naeyc-2024', 'eyfs-2025', 'tx-prek-guidelines'
    name            TEXT NOT NULL,
    version         TEXT,
    region          JSONB, -- {"country": "US", "state": "TX"} or {"country": "GB"}
    age_range       JSONB, -- {"min_months": 0, "max_months": 60}
    is_active       BOOLEAN DEFAULT true,
    -- The entire framework hierarchy as a JSONB tree
    structure       JSONB NOT NULL,
    -- structure: [
    --   {
    --     "id": "domain-1",
    --     "name": "Social-Emotional Development",
    --     "description": "...",
    --     "subdomains": [
    --       {
    --         "id": "subdomain-1a",
    --         "name": "Self-Awareness",
    --         "indicators": [
    --           {
    --             "id": "ind-1a-1",
    --             "name": "Recognizes own emotions",
    --             "age_range": {"min": 24, "max": 48},
    --             "proficiency_levels": ["emerging","developing","proficient"]
    --           }
    --         ]
    --       }
    --     ]
    --   }
    -- ]
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata: {"source_url": "...", "publisher": "NAEYC", "adopted_states": ["CA","TX","NY"]}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_framework_structure ON developmental_frameworks USING GIN (structure);

-- Which frameworks each organization uses
CREATE TABLE organization_frameworks (
    organization_id UUID NOT NULL REFERENCES organizations(id),
    framework_id    UUID NOT NULL REFERENCES developmental_frameworks(id),
    is_primary      BOOLEAN DEFAULT false,
    custom_overrides JSONB NOT NULL DEFAULT '{}',
    -- custom_overrides: org-specific additions or label changes to the base framework
    PRIMARY KEY (organization_id, framework_id)
);
```

### Observations (Relational Core + JSONB Flexibility)

```sql
CREATE TABLE observations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    observer_id     UUID NOT NULL,
    observed_at     TIMESTAMPTZ NOT NULL,
    narrative       TEXT,
    ai_narrative    TEXT,
    context         TEXT,
    is_shared_with_family BOOLEAN DEFAULT false,

    -- Framework-linked milestone assessments as JSONB
    milestone_assessments JSONB NOT NULL DEFAULT '[]',
    -- milestone_assessments: [
    --   {
    --     "framework_id": "uuid",
    --     "framework_slug": "naeyc-2024",
    --     "domain": "Social-Emotional Development",
    --     "subdomain": "Self-Awareness",
    --     "indicator_id": "ind-1a-1",
    --     "indicator_name": "Recognizes own emotions",
    --     "proficiency": "developing",
    --     "notes": "Named feeling as frustrated when tower fell"
    --   }
    -- ]

    -- AI-generated tags and analysis
    ai_analysis     JSONB NOT NULL DEFAULT '{}',
    -- ai_analysis: {
    --   "detected_domains": ["social_emotional", "cognitive"],
    --   "suggested_milestones": ["ind-1a-1", "ind-2b-3"],
    --   "sentiment": "positive",
    --   "model_version": "gpt-4o-2026-05",
    --   "confidence": 0.87
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_observations_child ON observations (child_id, observed_at DESC);
CREATE INDEX idx_observations_milestones ON observations USING GIN (milestone_assessments);
CREATE INDEX idx_observations_ai ON observations USING GIN (ai_analysis);

CREATE TABLE observation_media (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    observation_id  UUID NOT NULL REFERENCES observations(id),
    media_type      TEXT NOT NULL CHECK (media_type IN ('photo','video','audio')),
    storage_url     TEXT NOT NULL,
    thumbnail_url   TEXT,
    file_size_bytes BIGINT,
    caption         TEXT,
    ai_tags         JSONB NOT NULL DEFAULT '[]',
    -- ai_tags: ["outdoor_play", "group_activity", "fine_motor", "painting"]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Billing and Payments (Fully Relational)

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
    rate_modifiers  JSONB NOT NULL DEFAULT '[]',
    -- rate_modifiers: [
    --   {"type": "sibling_discount", "pct": 10, "applies_to": "second_child"},
    --   {"type": "part_time_rate", "multiplier": 0.6, "max_days": 3},
    --   {"type": "drop_in_surcharge", "flat_amount": 15.00}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
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
    line_items      JSONB NOT NULL DEFAULT '[]',
    -- line_items: [
    --   {"child_id": "uuid", "child_name": "Emma", "description": "Weekly tuition",
    --    "quantity": 4, "unit_price": 275.00, "total": 1100.00, "category": "tuition"},
    --   {"child_id": "uuid", "child_name": "Emma", "description": "Late pickup fee 5/12",
    --    "quantity": 1, "unit_price": 15.00, "total": 15.00, "category": "fee"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
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
```

### Subsidy Management (Relational + JSONB Rules Engine)

```sql
CREATE TABLE subsidy_authorizations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    program_type    TEXT NOT NULL,
    state_code      CHAR(2),
    authorization_number TEXT,
    rates           JSONB NOT NULL,
    -- rates: {
    --   "daily_rate": 45.00,
    --   "weekly_rate": 200.00,
    --   "monthly_rate": 850.00,
    --   "hourly_rate": null,
    --   "rate_tier": "infant_full_time",
    --   "authorized_hours_per_week": 40
    -- }
    copay           JSONB NOT NULL DEFAULT '{}',
    -- copay: {"amount": 35.00, "frequency": "weekly", "due_day": "monday"}
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          TEXT DEFAULT 'active'
        CHECK (status IN ('pending','active','suspended','expired','terminated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- State-specific subsidy billing rules (the rules engine)
CREATE TABLE subsidy_rule_sets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    state_code      CHAR(2) NOT NULL,
    program_type    TEXT NOT NULL,
    effective_date  DATE NOT NULL,
    expiry_date     DATE,
    rules           JSONB NOT NULL,
    -- rules: {
    --   "billing_unit": "daily",               -- or "hourly", "weekly", "monthly"
    --   "attendance_verification": "sign_in_out", -- or "headcount", "scheduled"
    --   "absence_policy": {
    --     "max_absent_days_per_month": 5,
    --     "absent_days_billable": true,
    --     "holiday_billable": true
    --   },
    --   "rate_tiers": [
    --     {"age_range": "0-12m", "care_type": "full_time", "rate": 55.00},
    --     {"age_range": "0-12m", "care_type": "part_time", "rate": 35.00},
    --     {"age_range": "13-24m", "care_type": "full_time", "rate": 48.00}
    --   ],
    --   "submission": {
    --     "portal_url": "https://tcc.force.com/childcare",
    --     "format": "csv",
    --     "fields": ["auth_number","child_name","attendance_days","amount"],
    --     "deadline_day_of_month": 10
    --   }
    -- }
    metadata        JSONB NOT NULL DEFAULT '{}',
    UNIQUE (state_code, program_type, effective_date)
);

CREATE TABLE subsidy_claims (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    authorization_id UUID NOT NULL REFERENCES subsidy_authorizations(id),
    claim_period_start DATE NOT NULL,
    claim_period_end   DATE NOT NULL,
    claim_data      JSONB NOT NULL,
    -- claim_data: {
    --   "attendance_days": 22,
    --   "attendance_hours": 176.5,
    --   "absent_days_claimed": 2,
    --   "daily_breakdown": [
    --     {"date": "2026-05-01", "hours": 8.5, "billable": true},
    --     {"date": "2026-05-02", "hours": 0, "absent": true, "billable": true}
    --   ]
    -- }
    claimed_amount  NUMERIC(10,2) NOT NULL,
    reimbursed_amount NUMERIC(10,2),
    status          TEXT DEFAULT 'draft'
        CHECK (status IN ('draft','submitted','approved','denied','paid','appealed')),
    submitted_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Staff Management (Relational + JSONB Credentials)

```sql
CREATE TABLE staff (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    site_id         UUID REFERENCES sites(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT NOT NULL,
    phone           TEXT,
    role            TEXT NOT NULL,
    hire_date       DATE,
    termination_date DATE,
    hourly_rate     NUMERIC(8,2),
    credentials     JSONB NOT NULL DEFAULT '[]',
    -- credentials: [
    --   {"type": "cpr", "name": "CPR/First Aid", "issued": "2025-06-01", "expires": "2027-06-01", "doc_url": "..."},
    --   {"type": "cda", "name": "Child Development Associate", "issued": "2024-01-15", "expires": null},
    --   {"type": "state_training", "name": "TX 30-hour pre-service", "hours": 30, "completed": "2025-08-20"}
    -- ]
    user_account_id UUID,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_staff_credentials ON staff USING GIN (credentials);

CREATE TABLE staff_schedules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    staff_id        UUID NOT NULL REFERENCES staff(id),
    classroom_id    UUID REFERENCES classrooms(id),
    date            DATE NOT NULL,
    shift_start     TIME NOT NULL,
    shift_end       TIME NOT NULL,
    status          TEXT DEFAULT 'scheduled',
    UNIQUE (staff_id, date, shift_start)
);

CREATE TABLE staff_time_entries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    staff_id        UUID NOT NULL REFERENCES staff(id),
    clock_in        TIMESTAMPTZ NOT NULL,
    clock_out       TIMESTAMPTZ,
    clock_in_method TEXT,
    break_minutes   INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Incidents and Health Logs

```sql
CREATE TABLE incident_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    site_id         UUID NOT NULL REFERENCES sites(id),
    reported_by     UUID NOT NULL REFERENCES staff(id),
    incident_type   TEXT NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL,
    severity        TEXT CHECK (severity IN ('minor','moderate','serious')),
    report_data     JSONB NOT NULL,
    -- report_data: {
    --   "description": "Fell from climbing structure",
    --   "body_part": "left knee",
    --   "first_aid": "Cleaned and applied bandage",
    --   "witnesses": ["staff-uuid-1"],
    --   "parent_notified": true,
    --   "parent_notified_at": "2026-05-25T10:30:00Z",
    --   "parent_signature_url": "...",
    --   "follow_up_needed": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Messaging

```sql
CREATE TABLE conversations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    conversation_type TEXT NOT NULL CHECK (conversation_type IN ('direct','classroom','broadcast')),
    subject         TEXT,
    participants    JSONB NOT NULL DEFAULT '[]',
    -- participants: [
    --   {"type": "staff", "id": "uuid", "name": "Ms. Johnson"},
    --   {"type": "guardian", "id": "uuid", "name": "Sarah Chen", "family_id": "uuid"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE messages (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    conversation_id UUID NOT NULL REFERENCES conversations(id),
    sender_type     TEXT NOT NULL CHECK (sender_type IN ('staff','guardian','system')),
    sender_id       UUID NOT NULL,
    body            TEXT NOT NULL,
    translation     JSONB,
    -- translation: {"original_language": "es", "translated_body": "...", "target_language": "en"}
    attachments     JSONB NOT NULL DEFAULT '[]',
    -- attachments: [{"type": "photo", "url": "...", "name": "IMG_1234.jpg", "size": 245000}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_messages_conversation ON messages (conversation_id, created_at DESC);
```

---

## Querying JSONB Data -- Examples

### Find all children with severe allergies

```sql
SELECT c.id, c.first_name, c.last_name, allergy
FROM children c,
     jsonb_array_elements(c.health_info -> 'allergies') AS allergy
WHERE allergy ->> 'severity' = 'severe'
  AND c.enrollment_status = 'enrolled';
```

### Generate a developmental milestone progress report

```sql
SELECT
    o.child_id,
    o.observed_at,
    ma ->> 'domain' AS domain,
    ma ->> 'indicator_name' AS indicator,
    ma ->> 'proficiency' AS proficiency
FROM observations o,
     jsonb_array_elements(o.milestone_assessments) AS ma
WHERE o.child_id = $1
  AND ma ->> 'framework_slug' = 'naeyc-2024'
ORDER BY o.observed_at;
```

### Retrieve subsidy billing rules for a state

```sql
SELECT rules
FROM subsidy_rule_sets
WHERE state_code = 'TX'
  AND program_type = 'CCDF'
  AND effective_date <= CURRENT_DATE
  AND (expiry_date IS NULL OR expiry_date > CURRENT_DATE)
ORDER BY effective_date DESC
LIMIT 1;
```

### Count meals by type for CACFP reporting

```sql
SELECT
    details ->> 'meal_type' AS meal_type,
    COUNT(*) AS servings,
    COUNT(DISTINCT child_id) AS children_served
FROM daily_activities
WHERE activity_type = 'meal'
  AND recorded_at >= '2026-05-01'
  AND recorded_at < '2026-06-01'
  AND organization_id = $1
GROUP BY details ->> 'meal_type';
```

### Search observations by AI-detected domain

```sql
SELECT o.id, o.narrative, o.observed_at
FROM observations o
WHERE o.ai_analysis @> '{"detected_domains": ["social_emotional"]}'
  AND o.child_id = $1
ORDER BY o.observed_at DESC;
```

---

## Pros

1. **Best of both worlds for this domain.** Financial and compliance data gets full relational integrity. Variable-schema domains (frameworks, daily activity details, health records, subsidy rules) get the flexibility they need without migration overhead. This directly addresses the challenge of supporting 500+ developmental frameworks and 50+ state subsidy rule variations.

2. **Single database engine.** Unlike polyglot persistence (e.g., PostgreSQL + MongoDB), this uses only PostgreSQL. The team manages one database, one backup strategy, one connection pool, one set of migrations. PostgreSQL's JSONB is well-tested and production-proven for this exact use case.

3. **No migration needed for new framework support.** Adding a new developmental framework (e.g., a state publishes new early learning standards) means inserting a row into `developmental_frameworks` with the framework structure as JSONB. No schema migration, no downtime, no deployment.

4. **Subsidy rules engine in data.** State-specific subsidy rules stored as JSONB in `subsidy_rule_sets` can be updated by business analysts without developer involvement. The application reads the rules and applies them programmatically, enabling faster multi-state rollout.

5. **JSONB indexing keeps queries fast.** PostgreSQL GIN indexes on JSONB columns support efficient containment queries (`@>`), existence checks (`?`), and path-based queries (`->`). The observation milestone search, allergy lookup, and AI tag queries shown above all use indexed paths.

6. **Append-only daily activities.** The `daily_activities` and `observations` tables are naturally append-only (teachers add records throughout the day; they never update past records). This provides an implicit timeline and audit trail without event sourcing complexity.

7. **Progressive enhancement path.** Start with this hybrid model and layer on event sourcing for specific high-value streams (e.g., billing events for audit) later if needed. The JSONB approach does not foreclose future architectural evolution.

---

## Cons

1. **JSONB is not type-safe at the database level.** The database cannot enforce that a `meal` activity has a `meal_type` field or that a credential has an `expires` date. Type validation must be enforced at the application layer (e.g., JSON Schema validation in the API). This increases the risk of inconsistent data if validation is bypassed.

2. **JSONB queries are less intuitive.** Developers must learn PostgreSQL's JSONB operators (`->`, `->>`, `@>`, `?`, `jsonb_array_elements`). Joins on JSONB fields are less performant than joins on indexed foreign keys. Complex analytical queries on JSONB data can be harder to write and optimize.

3. **Reporting on JSONB data requires extraction.** Standard reporting tools (Metabase, Looker, Tableau) work best with flat relational tables. Generating reports from JSONB columns requires views or materialized views that extract the relevant fields.

4. **Schema evolution within JSONB is implicit.** When the structure of a JSONB column changes (e.g., adding a new field to the `health_info` schema), old records retain the old structure. The application must handle both old and new formats gracefully -- a form of schema versioning that is easy to overlook.

5. **Line items as JSONB in invoices.** Storing invoice line items as JSONB (rather than a separate table) makes aggregate financial queries (total revenue by category, average tuition per child) more complex. A materialized view or ETL pipeline may be needed for financial reporting.

6. **GIN index storage overhead.** GIN indexes on large JSONB columns can be significantly larger than B-tree indexes on scalar columns, increasing storage requirements and write amplification.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ |
| Multi-tenancy | Row-Level Security with `organization_id` |
| ORM | Prisma (with `Json` type for JSONB columns) or Drizzle ORM |
| Schema validation | Zod or AJV (JSON Schema) at the API layer for JSONB payloads |
| Migrations | Prisma Migrate for relational schema; data migrations for JSONB structure changes |
| Search | PostgreSQL full-text + GIN for observation search |
| File storage | S3-compatible object storage |
| Caching | Redis for session data and real-time classroom state |
| Reporting | Materialized views for JSONB extraction; Metabase for dashboards |

---

## Migration and Scaling Considerations

- **JSONB validation pipeline:** Implement JSON Schema validation (e.g., AJV) in the API middleware layer. Define schemas for each JSONB column's expected structure and validate on write. Publish schemas alongside the OpenAPI spec.

- **Materialized views for reporting:** Create materialized views that flatten JSONB data into relational columns for reporting. Refresh these views on a schedule (e.g., hourly for operational dashboards, nightly for financial reports).

  ```sql
  CREATE MATERIALIZED VIEW mv_allergy_report AS
  SELECT c.id AS child_id, c.first_name, c.last_name,
         allergy ->> 'allergen' AS allergen,
         allergy ->> 'severity' AS severity,
         allergy ->> 'action_plan' AS action_plan
  FROM children c,
       jsonb_array_elements(c.health_info -> 'allergies') AS allergy
  WHERE c.enrollment_status = 'enrolled';
  ```

- **Partitioning for time-series tables:** Partition `attendance_records`, `daily_activities`, and `messages` by month. Partition `observations` by quarter (lower volume, longer retention).

- **JSONB column size monitoring:** Monitor the average size of JSONB columns (especially `developmental_frameworks.structure` and `subsidy_claims.claim_data`). If individual JSONB documents grow very large (>1MB), consider normalization or TOAST tuning.

- **Data portability:** For child data export (a key differentiator per the README), implement a JSON export format that maps to CEDS Early Learning data elements. The JSONB-stored developmental data can be exported directly without transformation.

- **COPPA data deletion:** JSONB fields containing child PII can be individually nulled or redacted without deleting the entire row, enabling granular compliance with COPPA deletion requirements while retaining anonymized statistical data.
