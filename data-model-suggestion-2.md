# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Early Childhood Education Platform (428)
> Approach: Event Sourcing with Command Query Responsibility Segregation
> Generated: 2026-05-25

---

## Summary

An event-sourced architecture where every state change is captured as an immutable event in an append-only event store. The system separates command handling (writes) from query handling (reads) using the CQRS pattern. Read-optimized projections are materialized from the event stream for each query use case: classroom dashboards, parent activity feeds, billing summaries, and developmental portfolios. This approach provides a complete audit trail of every action taken on a child's record -- a significant advantage for regulatory compliance (COPPA, FERPA, licensing audits) and for reconstructing the developmental history of a child over their entire enrollment.

---

## Architecture Overview

```
                     ┌──────────────┐
                     │   Commands   │
                     │ (Write Side) │
                     └──────┬───────┘
                            │
                     ┌──────▼───────┐
                     │   Command    │
                     │   Handlers   │
                     │  (validate,  │
                     │   enforce    │
                     │   rules)     │
                     └──────┬───────┘
                            │
                     ┌──────▼───────┐
                     │  Event Store │
                     │ (append-only │
                     │  immutable)  │
                     └──────┬───────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
       ┌──────▼──────┐ ┌───▼────┐ ┌──────▼──────┐
       │  Classroom  │ │Billing │ │ Portfolio   │
       │  Dashboard  │ │Summary │ │ Timeline    │
       │ Projection  │ │Proj.   │ │ Projection  │
       └─────────────┘ └────────┘ └─────────────┘
              │             │             │
       ┌──────▼─────────────▼─────────────▼──────┐
       │           Query Side (Reads)            │
       │        Optimized Read Models            │
       └─────────────────────────────────────────┘
```

---

## Event Store Schema

The event store is the single source of truth. All events are immutable and append-only.

```sql
-- Core event store table (PostgreSQL)
CREATE TABLE events (
    id              BIGSERIAL PRIMARY KEY,
    event_id        UUID NOT NULL DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    event_type      TEXT NOT NULL,
    event_version   INTEGER NOT NULL,
    payload         JSONB NOT NULL,
    metadata        JSONB NOT NULL DEFAULT '{}',
    causation_id    UUID,           -- which command caused this
    correlation_id  UUID,           -- groups related events
    actor_id        UUID NOT NULL,  -- who performed the action
    actor_type      TEXT NOT NULL,  -- 'staff','guardian','system','ai'
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Optimistic concurrency control
    UNIQUE (aggregate_type, aggregate_id, event_version)
);

CREATE INDEX idx_events_aggregate ON events (aggregate_type, aggregate_id, event_version);
CREATE INDEX idx_events_org_type ON events (organization_id, event_type, occurred_at);
CREATE INDEX idx_events_correlation ON events (correlation_id);
CREATE INDEX idx_events_occurred ON events (occurred_at);

-- Partition by month for performance
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

### Snapshot Store (for aggregate rehydration optimization)

```sql
CREATE TABLE snapshots (
    aggregate_type  TEXT NOT NULL,
    aggregate_id    UUID NOT NULL,
    version         INTEGER NOT NULL,
    state           JSONB NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);
```

---

## Aggregate Roots and Event Types

### Child Aggregate

The Child aggregate is the central domain object, encompassing enrollment, health records, and classroom assignments.

```
Aggregate: Child
Stream: child-{childId}

Events:
  ChildRegistered           { firstName, lastName, dateOfBirth, familyId, gender }
  ChildEnrolled             { siteId, classroomId, enrollmentDate, scheduleType }
  ChildClassroomChanged     { fromClassroomId, toClassroomId, effectiveDate, reason }
  ChildWithdrawn            { withdrawalDate, reason }
  ChildGraduated            { graduationDate, nextProgram }
  ChildAllergyAdded         { allergen, severity, actionPlan }
  ChildAllergyRemoved       { allergen, reason }
  ChildMedicalRecordAdded   { recordType, description, effectiveDate, documentUrl }
  ChildPhotoConsentUpdated  { consentGranted, consentedBy, consentedAt }
  ChildProfileUpdated       { fieldsChanged }
```

### Attendance Aggregate

```
Aggregate: Attendance
Stream: attendance-{childId}-{date}

Events:
  ChildCheckedIn            { childId, siteId, checkInTime, method, checkedInBy }
  ChildCheckedOut           { childId, checkOutTime, method, checkedOutBy }
  AttendanceMarkedAbsent    { childId, date, reason, notifiedBy }
  AttendanceCorrected       { childId, date, correctedField, oldValue, newValue, correctedBy }
  UnauthorizedPickupAttempted { childId, attemptedBy, time, actionTaken }
```

### Daily Activity Aggregate

```
Aggregate: DailyActivity
Stream: daily-activity-{childId}-{date}

Events:
  MealRecorded              { mealType, foodItems, portionEaten, recordedBy }
  NapStarted                { startTime, recordedBy }
  NapEnded                  { endTime, duration, recordedBy }
  DiaperChanged             { status, recordedBy }
  PottyUsed                 { success, recordedBy }
  ActivityLogged            { description, recordedBy }
  MoodRecorded              { mood, notes, recordedBy }
  MedicationAdministered    { medicationName, dosage, administeredBy, parentAuthRef }
  SunscreenApplied          { recordedBy }
```

### Observation Aggregate

```
Aggregate: Observation
Stream: observation-{observationId}

Events:
  ObservationCreated        { childId, observerId, narrative, context, observedAt }
  ObservationNarrativeEnhanced { aiNarrative, modelVersion, originalNarrative }
  ObservationMilestoneLinked { milestoneId, frameworkId, domainId, proficiency }
  ObservationMilestoneUnlinked { milestoneId, reason }
  ObservationMediaAttached  { mediaType, storageUrl, thumbnailUrl, caption }
  ObservationSharedWithFamily { sharedAt, sharedBy }
  ObservationFamilyCommentAdded { guardianId, comment }
  ObservationRetracted      { reason, retractedBy }
```

### Billing Aggregate

```
Aggregate: Invoice
Stream: invoice-{invoiceId}

Events:
  InvoiceGenerated          { familyId, periodStart, periodEnd, lineItems, subtotal, totalDue, dueDate }
  InvoiceLineItemAdded      { childId, description, quantity, unitPrice, category }
  InvoiceSubsidyCreditApplied { authorizationId, creditAmount, program }
  InvoiceLateFeeApplied     { amount, daysLate }
  InvoiceDiscountApplied    { amount, reason }
  InvoiceSent               { sentTo, sentVia, sentAt }
  InvoicePaymentReceived    { paymentId, amount, method, gatewayRef }
  InvoicePartialPaymentReceived { paymentId, amount, remainingBalance }
  InvoiceMarkedOverdue      { daysOverdue }
  InvoiceVoided             { reason, voidedBy }
  InvoiceRefundIssued       { amount, reason, refundMethod }
```

### Subsidy Aggregate

```
Aggregate: SubsidyAuthorization
Stream: subsidy-{authorizationId}

Events:
  SubsidyAuthorizationCreated { childId, programType, stateCode, authNumber, rates, copay, startDate, endDate }
  SubsidyAuthorizationRenewed { newEndDate, newRates }
  SubsidyAuthorizationSuspended { reason, suspendedAt }
  SubsidyAuthorizationTerminated { reason, terminatedAt }
  SubsidyClaimSubmitted     { claimPeriod, attendanceDays, attendanceHours, claimedAmount }
  SubsidyClaimApproved      { approvedAmount }
  SubsidyClaimDenied        { reason, deniedAt }
  SubsidyClaimPaid          { paidAmount, paidAt }
  SubsidyClaimAppealed      { reason, appealedAt }
```

### Staff Aggregate

```
Aggregate: Staff
Stream: staff-{staffId}

Events:
  StaffHired                { firstName, lastName, email, role, siteId, hireDate }
  StaffCredentialAdded      { credentialType, name, issuedDate, expiryDate }
  StaffCredentialExpired    { credentialType, name, expiredAt }
  StaffScheduleAssigned     { classroomId, date, shiftStart, shiftEnd }
  StaffClockedIn            { clockInTime, method }
  StaffClockedOut           { clockOutTime, hoursWorked, breakMinutes }
  StaffTerminated           { terminationDate, reason }
```

### Messaging Aggregate

```
Aggregate: Conversation
Stream: conversation-{conversationId}

Events:
  ConversationStarted       { senderType, senderId, recipientType, recipientId, subject }
  MessageSent               { body, senderType, senderId }
  MessageTranslated         { originalLanguage, targetLanguage, translatedBody }
  MediaAttached             { mediaType, storageUrl, fileName }
  BroadcastSent             { siteId, subject, body, sentBy }
  MessageRead               { readBy, readAt }
```

---

## Read Model Projections

Projections subscribe to the event stream and maintain denormalized read models optimized for specific query patterns.

### Classroom Dashboard Projection

```sql
-- Materialized from Attendance, DailyActivity, and Staff events
CREATE TABLE classroom_dashboard (
    classroom_id    UUID NOT NULL,
    organization_id UUID NOT NULL,
    date            DATE NOT NULL,
    children_present INTEGER DEFAULT 0,
    children_capacity INTEGER,
    staff_present   INTEGER DEFAULT 0,
    current_ratio   NUMERIC(4,1),
    ratio_compliant BOOLEAN DEFAULT true,
    meals_served    JSONB DEFAULT '{}',  -- {"breakfast": 12, "lunch": 14, ...}
    last_updated    TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (classroom_id, date)
);
```

### Parent Activity Feed Projection

```sql
-- Materialized from DailyActivity, Observation, Attendance, and Message events
CREATE TABLE parent_activity_feed (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    family_id       UUID NOT NULL,
    child_id        UUID NOT NULL,
    event_type      TEXT NOT NULL, -- 'check_in','meal','nap','observation','message'
    summary         TEXT NOT NULL,
    details         JSONB,
    media_urls      TEXT[],
    occurred_at     TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_parent_feed ON parent_activity_feed (family_id, occurred_at DESC);
```

### Developmental Portfolio Projection

```sql
-- Materialized from Observation and milestone events
CREATE TABLE child_portfolio (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    child_id        UUID NOT NULL,
    framework_id    UUID NOT NULL,
    domain_name     TEXT NOT NULL,
    milestone_name  TEXT NOT NULL,
    proficiency     TEXT,
    observation_count INTEGER DEFAULT 0,
    first_observed  TIMESTAMPTZ,
    last_observed   TIMESTAMPTZ,
    latest_narrative TEXT,
    media_urls      TEXT[],
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_portfolio ON child_portfolio (child_id, framework_id);
```

### Billing Summary Projection

```sql
-- Materialized from Invoice and Payment events
CREATE TABLE billing_summary (
    organization_id UUID NOT NULL,
    family_id       UUID NOT NULL,
    total_invoiced  NUMERIC(12,2) DEFAULT 0,
    total_paid      NUMERIC(12,2) DEFAULT 0,
    total_subsidized NUMERIC(12,2) DEFAULT 0,
    outstanding_balance NUMERIC(12,2) DEFAULT 0,
    last_payment_date DATE,
    overdue_count   INTEGER DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organization_id, family_id)
);
```

### Ratio Compliance Projection

```sql
-- Materialized from Attendance and Staff schedule/clock events
CREATE TABLE ratio_snapshots (
    id              BIGSERIAL PRIMARY KEY,
    organization_id UUID NOT NULL,
    classroom_id    UUID NOT NULL,
    snapshot_time   TIMESTAMPTZ NOT NULL,
    children_count  INTEGER NOT NULL,
    staff_count     INTEGER NOT NULL,
    actual_ratio    NUMERIC(4,1) NOT NULL,
    required_ratio  NUMERIC(4,1) NOT NULL,
    is_compliant    BOOLEAN NOT NULL,
    alert_sent      BOOLEAN DEFAULT false
);

CREATE INDEX idx_ratio_compliance ON ratio_snapshots (classroom_id, snapshot_time DESC);
```

### CACFP Reporting Projection

```sql
-- Materialized from MealRecorded and Attendance events
CREATE TABLE cacfp_monthly_report (
    organization_id UUID NOT NULL,
    site_id         UUID NOT NULL,
    year_month      TEXT NOT NULL, -- '2026-05'
    meal_type       TEXT NOT NULL,
    total_servings  INTEGER DEFAULT 0,
    children_served INTEGER DEFAULT 0,
    menu_items      JSONB DEFAULT '[]',
    updated_at      TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organization_id, site_id, year_month, meal_type)
);
```

---

## Event Processing Pipeline

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────────┐
│  Event Store │────▶│  Event Bus   │────▶│  Projection Workers │
│ (PostgreSQL) │     │ (NATS/Kafka) │     │  (per read model)   │
└─────────────┘     └──────┬───────┘     └─────────────────────┘
                           │
                    ┌──────▼───────┐
                    │  Side-Effect │
                    │  Processors  │
                    │ (push notif, │
                    │  email, AI,  │
                    │  translation)│
                    └──────────────┘
```

### Event Processing Examples

```typescript
// Projection handler: update parent feed when a meal is recorded
async function handleMealRecorded(event: MealRecordedEvent) {
  await db.parentActivityFeed.insert({
    organizationId: event.metadata.organizationId,
    familyId: await lookupFamilyId(event.payload.childId),
    childId: event.payload.childId,
    eventType: 'meal',
    summary: `${event.payload.mealType}: ate ${event.payload.portionEaten}`,
    details: { foodItems: event.payload.foodItems },
    occurredAt: event.occurredAt,
  });
}

// Side-effect: trigger AI narrative enhancement when observation is created
async function handleObservationCreated(event: ObservationCreatedEvent) {
  await commandBus.dispatch({
    type: 'EnhanceObservationNarrative',
    observationId: event.aggregateId,
    originalNarrative: event.payload.narrative,
    childAge: await calculateChildAge(event.payload.childId),
    framework: await getActiveFramework(event.metadata.organizationId),
  });
}

// Side-effect: send ratio compliance alert
async function handleChildCheckedIn(event: ChildCheckedInEvent) {
  const snapshot = await ratioProjection.getCurrentSnapshot(event.payload.siteId);
  if (!snapshot.isCompliant) {
    await notificationService.sendRatioAlert({
      siteId: event.payload.siteId,
      currentRatio: snapshot.actualRatio,
      requiredRatio: snapshot.requiredRatio,
    });
  }
}
```

---

## Pros

1. **Complete audit trail by design.** Every action on every child record, every billing change, every attendance event is permanently recorded with who did what and when. This is invaluable for COPPA/FERPA compliance, licensing audits, and resolving disputes (e.g., "who picked up the child?", "was the subsidy rate changed?"). No data is ever lost or overwritten.

2. **Temporal queries are natural.** "What was this child's developmental profile six months ago?" is trivially answered by replaying events up to a point in time. Traditional databases require complex versioning schemes to answer such questions. This is especially valuable for generating developmental progress reports over time.

3. **Decoupled read models.** Each read model (parent feed, billing summary, portfolio) is independently optimized for its query pattern. The parent activity feed can be a simple time-ordered table, while the developmental portfolio can be a denormalized milestone matrix. New read models can be added without modifying the write side.

4. **Natural fit for real-time features.** The event bus enables push notifications to parent apps (child checked in, meal recorded, observation shared) as a natural side effect of event publication. No polling or separate notification system needed.

5. **Debugging and support.** When a parent disputes a billing charge or a licensing inspector questions an attendance record, the event stream provides an indisputable, time-ordered record of exactly what happened. Events include causation and correlation IDs for tracing.

6. **AI pipeline integration.** AI observation enhancement, translation, and anomaly detection fit naturally as event subscribers. An `ObservationCreated` event triggers AI narrative generation, which publishes `ObservationNarrativeEnhanced` -- clean, decoupled, and retryable.

---

## Cons

1. **Significant implementation complexity.** Event sourcing requires building and maintaining an event store, command handlers, event handlers, projection workers, and a projection rebuild pipeline. This is substantially more complex than a standard CRUD application and requires the team to have deep experience with the pattern.

2. **Eventual consistency in read models.** Projections are asynchronously updated. After a teacher records a meal, the parent activity feed may not reflect it for a few hundred milliseconds (or longer under load). For most ECE use cases this is acceptable, but billing and attendance projections may need stronger consistency guarantees.

3. **Projection rebuild cost.** If a projection has a bug and needs to be rebuilt, it must replay all relevant events from the beginning. For a platform with millions of daily activity events per year, this can take hours. Snapshots mitigate but add complexity.

4. **Event schema evolution.** As the domain model evolves, event schemas change. Old events in the store must remain readable. This requires upcasting (transforming old event formats to new ones at read time) and careful versioning -- an ongoing maintenance burden.

5. **Operational complexity.** Running an event store, message bus (NATS/Kafka), and multiple projection workers adds operational surface area compared to a single PostgreSQL database. Monitoring, alerting, and deployment are more complex.

6. **Developer onboarding difficulty.** ECE is a high-turnover industry, and the development team may also experience turnover. Event sourcing is a less common pattern than CRUD, making it harder to hire for and onboard new developers.

7. **Overkill for simple CRUD entities.** Some entities (developmental frameworks, billing plan configurations, site settings) are simple configuration data that does not benefit from event sourcing. A hybrid approach (event-sourced aggregates for child/attendance/billing, CRUD for configuration) adds further architectural complexity.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event Store | PostgreSQL with append-only events table (or EventStoreDB for dedicated event store) |
| Event Bus | NATS JetStream (lightweight, ECE-scale) or Apache Kafka (if enterprise scale) |
| Command/Query framework | Custom (TypeScript/Node.js) or Axon Framework (Java/Kotlin) |
| Projection Store | PostgreSQL (for relational projections), Redis (for real-time dashboards) |
| Snapshot Store | PostgreSQL JSONB |
| Search | Elasticsearch or Meilisearch (projected from observation events) |
| File Storage | S3-compatible object storage |
| Real-time push | WebSocket (Socket.io) or Server-Sent Events, driven by event bus |

---

## Migration and Scaling Considerations

- **Event store partitioning:** Partition the events table by month. Events older than the retention period can be moved to cold storage (S3 + Parquet) while remaining queryable for audit purposes.

- **Projection scaling:** Each projection is an independent consumer. Projections that serve different use cases (parent app, director dashboard, state reporting) can run on separate infrastructure and scale independently.

- **Multi-region deployment:** The event store can be replicated across regions. Projections in each region consume from a local replica, providing low-latency reads for parent apps while maintaining a single authoritative event stream.

- **Gradual adoption path:** Start with event sourcing for the highest-value aggregates (Child, Attendance, Observation, Invoice) and use standard CRUD for configuration entities (frameworks, billing plans, sites). This reduces initial complexity while delivering the core benefits.

- **Event archival for COPPA compliance:** The append-only nature of the event store creates a tension with COPPA/GDPR right-to-deletion requirements. Implement crypto-shredding: encrypt event payloads with per-child keys, and delete the key when the child's data must be erased. Events remain in the store (maintaining referential integrity) but are unreadable.

- **Projection versioning:** Use numbered projection versions (e.g., `parent_activity_feed_v2`). Build new projections alongside old ones, switch traffic when the new projection catches up, then drop the old version. This enables zero-downtime projection schema changes.

- **Snapshotting strategy:** Take snapshots of frequently accessed aggregates (Child, Invoice) every N events (e.g., every 50 events). This bounds rehydration time and keeps command processing fast.

- **Idempotency:** All event handlers must be idempotent. Use the event's `event_id` as an idempotency key in projections to safely handle replays and redeliveries.
