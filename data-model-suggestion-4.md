# Data Model Suggestion 4: Developmental Knowledge Graph + Relational Operational Core

> Project: Early Childhood Education Platform (428)
> Approach: Graph database for developmental/curriculum data + PostgreSQL for operations
> Generated: 2026-05-25

---

## Summary

A polyglot persistence architecture that uses a graph database (Neo4j or Apache AGE -- a PostgreSQL graph extension) for the developmental, curriculum, and learning documentation domain, paired with PostgreSQL for operational data (billing, attendance, staff, enrollment). The key insight is that child development tracking is fundamentally a graph problem: children connect to observations, observations connect to milestones, milestones nest within domains, domains belong to frameworks, and the relationships between these entities carry meaning (proficiency levels, temporal ordering, frequency patterns). A graph model makes traversals like "show all milestones this child has demonstrated across any framework" or "find all children who are emerging in social-emotional development" natural and performant, while relational tables continue to handle the transactional, compliance, and financial operations where they excel.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                      Application Layer                       │
│   (Next.js / React Native -- API Gateway / GraphQL)          │
└───────────┬────────────────────────────┬─────────────────────┘
            │                            │
    ┌───────▼────────┐          ┌────────▼──────────┐
    │  Graph Database │          │    PostgreSQL      │
    │  (Neo4j / AGE)  │          │  (Operational DB)  │
    │                 │          │                    │
    │ • Frameworks    │          │ • Organizations    │
    │ • Domains       │   sync   │ • Children (core)  │
    │ • Milestones    │◄────────►│ • Families         │
    │ • Observations  │          │ • Attendance       │
    │ • Child nodes   │          │ • Billing          │
    │ • Learning      │          │ • Payments         │
    │   pathways      │          │ • Staff            │
    │ • Curriculum    │          │ • Subsidies        │
    │   plans         │          │ • Incidents        │
    └────────────────┘          │ • CACFP            │
                                │ • Messaging        │
                                └───────────────────┘
```

---

## Graph Database Schema (Neo4j / Cypher)

### Node Types

```cypher
// Developmental frameworks as a hierarchy
(:Framework {id, name, version, country, state, ageRangeMonths, isActive})
(:Domain {id, name, description, sortOrder})
(:Subdomain {id, name, description, sortOrder})
(:Indicator {id, name, description, ageRangeMin, ageRangeMax})
(:Milestone {id, name, description, ageRangeMin, ageRangeMax, proficiencyLevels})

// Child and observation nodes (synced from PostgreSQL)
(:Child {id, firstName, lastName, dateOfBirth, organizationId, enrollmentStatus})
(:Observation {id, childId, observerId, observedAt, narrative, aiNarrative, context})
(:Media {id, mediaType, storageUrl, thumbnailUrl, caption})

// Curriculum and learning
(:LessonPlan {id, title, description, ageGroup, createdBy, createdAt})
(:Activity {id, name, description, materialsNeeded, duration})
(:LearningArea {id, name, description})

// AI-generated
(:Interest {id, name, frequency, firstObserved, lastObserved})
(:DevelopmentalProfile {id, childId, generatedAt, summary})
```

### Relationship Types

```cypher
// Framework hierarchy
(:Framework)-[:HAS_DOMAIN]->(:Domain)
(:Domain)-[:HAS_SUBDOMAIN]->(:Subdomain)
(:Subdomain)-[:HAS_INDICATOR]->(:Indicator)
(:Indicator)-[:HAS_MILESTONE]->(:Milestone)

// Cross-framework alignment (powerful for multi-framework centers)
(:Milestone)-[:ALIGNS_WITH {confidence, mappedBy}]->(:Milestone)
// e.g., NAEYC milestone "Recognizes own emotions" ALIGNS_WITH EYFS "Managing feelings and behaviour"

// Child developmental progress
(:Child)-[:HAS_OBSERVATION {observedAt}]->(:Observation)
(:Observation)-[:DEMONSTRATES {proficiency, assessedBy, assessedAt}]->(:Milestone)
(:Observation)-[:HAS_MEDIA]->(:Media)
(:Observation)-[:TAGGED_WITH]->(:LearningArea)

// Child developmental state (latest assessed proficiency per milestone)
(:Child)-[:CURRENT_LEVEL {proficiency, lastAssessedAt, observationCount}]->(:Milestone)

// Emerging interests detected by AI
(:Child)-[:SHOWS_INTEREST {strength, observationCount}]->(:Interest)
(:Interest)-[:RELATES_TO]->(:LearningArea)

// Curriculum connections
(:LessonPlan)-[:TARGETS]->(:Milestone)
(:LessonPlan)-[:INCLUDES]->(:Activity)
(:Activity)-[:DEVELOPS]->(:LearningArea)
(:Activity)-[:APPROPRIATE_FOR {ageMin, ageMax}]->(:Domain)

// Learning pathway (AI-recommended sequence)
(:Milestone)-[:PREREQUISITE_FOR {typical, sequence}]->(:Milestone)
(:Child)-[:NEXT_RECOMMENDED {reason, generatedAt}]->(:Activity)
```

### Example Graph Queries

**Child's complete developmental profile across all frameworks:**

```cypher
MATCH (c:Child {id: $childId})-[r:CURRENT_LEVEL]->(m:Milestone)
      <-[:HAS_MILESTONE]-(i:Indicator)<-[:HAS_INDICATOR]-(sd:Subdomain)
      <-[:HAS_SUBDOMAIN]-(d:Domain)<-[:HAS_DOMAIN]-(f:Framework)
RETURN f.name AS framework,
       d.name AS domain,
       sd.name AS subdomain,
       m.name AS milestone,
       r.proficiency AS currentLevel,
       r.lastAssessedAt AS lastAssessed,
       r.observationCount AS evidenceCount
ORDER BY f.name, d.sortOrder, sd.sortOrder
```

**Find children emerging in a specific domain for targeted lesson planning:**

```cypher
MATCH (c:Child {organizationId: $orgId, enrollmentStatus: 'enrolled'})
      -[r:CURRENT_LEVEL {proficiency: 'emerging'}]->(m:Milestone)
      <-[:HAS_MILESTONE]-(i:Indicator)<-[:HAS_INDICATOR]-(sd:Subdomain)
      <-[:HAS_SUBDOMAIN]-(d:Domain {name: 'Social-Emotional Development'})
RETURN c.firstName, c.lastName, m.name AS milestone, r.lastAssessedAt
ORDER BY r.lastAssessedAt DESC
```

**Cross-framework milestone alignment:**

```cypher
MATCH (m1:Milestone)<-[:HAS_MILESTONE]-(:Indicator)
      <-[:HAS_INDICATOR]-(:Subdomain)<-[:HAS_SUBDOMAIN]-(:Domain)
      <-[:HAS_DOMAIN]-(f1:Framework {name: 'NAEYC'})
MATCH (m1)-[a:ALIGNS_WITH]->(m2:Milestone)<-[:HAS_MILESTONE]-(:Indicator)
      <-[:HAS_INDICATOR]-(:Subdomain)<-[:HAS_SUBDOMAIN]-(:Domain)
      <-[:HAS_DOMAIN]-(f2:Framework {name: 'EYFS 2025'})
WHERE a.confidence > 0.8
RETURN m1.name AS naeyc_milestone, m2.name AS eyfs_milestone, a.confidence
ORDER BY a.confidence DESC
```

**Suggest next activities for a child based on current levels and interests:**

```cypher
MATCH (c:Child {id: $childId})-[:CURRENT_LEVEL {proficiency: 'emerging'}]->(m:Milestone)
MATCH (m)<-[:TARGETS]-(lp:LessonPlan)-[:INCLUDES]->(a:Activity)
OPTIONAL MATCH (c)-[si:SHOWS_INTEREST]->(int:Interest)-[:RELATES_TO]->(la:LearningArea)
                <-[:TAGGED_WITH]-(obs:Observation)<-[:HAS_OBSERVATION]-(c)
WHERE a.ageMin <= duration.between(c.dateOfBirth, date()).months
  AND a.ageMax >= duration.between(c.dateOfBirth, date()).months
RETURN a.name AS suggestedActivity,
       a.description,
       collect(DISTINCT m.name) AS targetMilestones,
       collect(DISTINCT int.name) AS relatedInterests,
       count(DISTINCT obs) AS interestEvidence
ORDER BY interestEvidence DESC, size(targetMilestones) DESC
LIMIT 10
```

**Developmental timeline for a child (observation history):**

```cypher
MATCH (c:Child {id: $childId})-[:HAS_OBSERVATION]->(o:Observation)
OPTIONAL MATCH (o)-[d:DEMONSTRATES]->(m:Milestone)
      <-[:HAS_MILESTONE]-(i:Indicator)<-[:HAS_INDICATOR]-(sd:Subdomain)
      <-[:HAS_SUBDOMAIN]-(dom:Domain)
OPTIONAL MATCH (o)-[:HAS_MEDIA]->(media:Media)
RETURN o.observedAt AS date,
       o.narrative,
       collect(DISTINCT {
         domain: dom.name,
         milestone: m.name,
         proficiency: d.proficiency
       }) AS milestones,
       collect(DISTINCT media.thumbnailUrl) AS photos
ORDER BY o.observedAt DESC
```

---

## PostgreSQL Schema (Operational Core)

The PostgreSQL schema handles all transactional, financial, and compliance-critical data. It is identical in structure to the normalized relational model (Suggestion 1) for these domains. Key tables:

```sql
-- Same as Suggestion 1 for:
-- organizations, sites, classrooms
-- families, guardians, children (core fields only; health_info can be JSONB)
-- attendance_records (partitioned by date)
-- billing_plans, invoices, invoice_line_items, payments
-- subsidy_authorizations, subsidy_claims, subsidy_rule_sets
-- staff, staff_credentials, staff_schedules, staff_time_entries
-- incident_reports, medication_logs
-- meal_records, meal_attendance
-- messages, message_attachments

-- Children table (simplified -- developmental data lives in graph)
CREATE TABLE children (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    family_id       UUID NOT NULL REFERENCES families(id),
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    date_of_birth   DATE NOT NULL,
    gender          TEXT,
    enrollment_status TEXT NOT NULL DEFAULT 'waitlisted',
    enrollment_date DATE,
    withdrawal_date DATE,
    photo_consent   BOOLEAN DEFAULT false,
    health_info     JSONB NOT NULL DEFAULT '{}',
    graph_node_id   TEXT, -- reference to Neo4j node ID for sync
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Daily activities remain in PostgreSQL (high-volume, time-series)
-- but observations are synced to the graph for relationship queries
CREATE TABLE daily_activities (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL REFERENCES organizations(id),
    child_id        UUID NOT NULL REFERENCES children(id),
    recorded_by     UUID NOT NULL,
    activity_type   TEXT NOT NULL,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    details         JSONB NOT NULL,
    shared_with_family BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Observations stored in both PostgreSQL (source of truth) and graph (relationships)
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
    graph_synced    BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Synchronization Strategy

Data flows between PostgreSQL and the graph database via a sync service:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  PostgreSQL  │────▶│  Sync Worker │────▶│    Neo4j     │
│  (writes)    │     │  (CDC/queue) │     │  (graph)     │
└──────────────┘     └──────────────┘     └──────────────┘

Sync triggers:
1. Child enrolled/updated → create/update Child node in graph
2. Observation created → create Observation node + DEMONSTRATES relationships
3. Milestone assessment added → create/update CURRENT_LEVEL relationship
4. Framework added/updated → rebuild framework subgraph
```

### Sync Implementation Options

**Option A: Change Data Capture (CDC) with Debezium**

PostgreSQL WAL changes are captured by Debezium and published to a message queue. A consumer service processes child and observation events and writes them to Neo4j.

**Option B: Application-level dual write with outbox pattern**

The application writes to PostgreSQL and publishes an outbox event in the same transaction. A background worker reads the outbox and writes to Neo4j. This ensures at-least-once delivery without CDC infrastructure.

```sql
CREATE TABLE sync_outbox (
    id              BIGSERIAL PRIMARY KEY,
    aggregate_type  TEXT NOT NULL, -- 'child', 'observation', 'framework'
    aggregate_id    UUID NOT NULL,
    event_type      TEXT NOT NULL, -- 'created', 'updated', 'deleted'
    payload         JSONB NOT NULL,
    processed       BOOLEAN DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Option C: Apache AGE (graph extension for PostgreSQL)**

If operational simplicity is paramount, use Apache AGE, which adds graph query capabilities directly within PostgreSQL. This eliminates the sync problem entirely -- graph and relational data live in the same database.

```sql
-- Apache AGE: graph queries inside PostgreSQL
SELECT * FROM cypher('development_graph', $$
    MATCH (c:Child {id: 'child-uuid'})-[r:CURRENT_LEVEL]->(m:Milestone)
          <-[:HAS_MILESTONE]-(i:Indicator)<-[:HAS_INDICATOR]-(sd:Subdomain)
    RETURN sd.name, m.name, r.proficiency
$$) AS (subdomain agtype, milestone agtype, proficiency agtype);
```

---

## Specialized Graph Features for ECE

### 1. Cross-Framework Alignment Engine

One of the platform's key differentiators is supporting arbitrary frameworks. The graph makes it natural to define alignment relationships between milestones across frameworks:

```cypher
// An administrator or AI maps equivalent milestones across frameworks
CREATE (naeyc_se1)-[:ALIGNS_WITH {confidence: 0.92, mappedBy: 'ai', verifiedBy: 'admin'}]->(eyfs_psed1)

// A center using both NAEYC and state standards can see unified progress
MATCH (c:Child {id: $childId})-[cl:CURRENT_LEVEL]->(m:Milestone)
OPTIONAL MATCH (m)-[:ALIGNS_WITH]-(aligned:Milestone)
      <-[:HAS_MILESTONE]-(:Indicator)<-[:HAS_INDICATOR]-(:Subdomain)
      <-[:HAS_SUBDOMAIN]-(:Domain)<-[:HAS_DOMAIN]-(f:Framework)
RETURN m.name, cl.proficiency,
       collect({framework: f.name, alignedMilestone: aligned.name}) AS crossFramework
```

### 2. Learning Pathway Recommendations

The graph naturally represents prerequisite relationships between milestones and can power AI-driven learning pathway recommendations:

```cypher
// Define prerequisite chain
CREATE (m1:Milestone {name: "Grasps objects"})-[:PREREQUISITE_FOR]->(m2:Milestone {name: "Transfers objects hand to hand"})
CREATE (m2)-[:PREREQUISITE_FOR]->(m3:Milestone {name: "Uses pincer grasp"})

// Find unmet prerequisites for a child's emerging milestones
MATCH (c:Child {id: $childId})-[:CURRENT_LEVEL {proficiency: 'emerging'}]->(target:Milestone)
MATCH (prereq:Milestone)-[:PREREQUISITE_FOR*1..3]->(target)
WHERE NOT EXISTS {
    MATCH (c)-[:CURRENT_LEVEL {proficiency: 'proficient'}]->(prereq)
}
RETURN target.name AS targetMilestone, prereq.name AS unmetPrerequisite
```

### 3. Cohort Developmental Analytics

Graph queries can efficiently compare a child's developmental profile against classroom or age-group cohorts:

```cypher
// Compare a child's profile to their classroom peers
MATCH (c:Child {id: $childId})-[cl:CURRENT_LEVEL]->(m:Milestone)
WITH c, collect({milestone: m.id, proficiency: cl.proficiency}) AS childProfile
MATCH (peer:Child {organizationId: c.organizationId, enrollmentStatus: 'enrolled'})
WHERE peer.id <> c.id
  AND duration.between(peer.dateOfBirth, date()).months
     BETWEEN duration.between(c.dateOfBirth, date()).months - 3
     AND duration.between(c.dateOfBirth, date()).months + 3
MATCH (peer)-[pl:CURRENT_LEVEL]->(m:Milestone)
RETURN m.name,
       count(CASE WHEN pl.proficiency = 'proficient' THEN 1 END) AS peersAtProficient,
       count(CASE WHEN pl.proficiency = 'developing' THEN 1 END) AS peersAtDeveloping,
       count(CASE WHEN pl.proficiency = 'emerging' THEN 1 END) AS peersAtEmerging,
       count(peer) AS totalPeers
```

### 4. Interest Detection and Curriculum Matching

```cypher
// AI populates interest nodes from observation analysis
// Then match interests to available curriculum activities
MATCH (c:Child {id: $childId})-[si:SHOWS_INTEREST]->(int:Interest)
WHERE si.strength > 0.7
MATCH (int)-[:RELATES_TO]->(la:LearningArea)<-[:DEVELOPS]-(act:Activity)
      <-[:INCLUDES]-(lp:LessonPlan)-[:TARGETS]->(m:Milestone)
      <-[:CURRENT_LEVEL {proficiency: 'emerging'}]-(c)
RETURN lp.title, act.name,
       collect(DISTINCT int.name) AS matchedInterests,
       collect(DISTINCT m.name) AS targetedMilestones,
       avg(si.strength) AS interestRelevance
ORDER BY interestRelevance DESC
LIMIT 5
```

---

## Pros

1. **Natural modeling of developmental hierarchies.** Frameworks, domains, subdomains, indicators, and milestones form deep hierarchies with cross-cutting relationships. Graph databases handle arbitrary-depth traversals without recursive CTEs or self-joins. Querying "all milestones a child has demonstrated across all frameworks" is a single, readable Cypher query.

2. **Cross-framework alignment is a first-class feature.** The `ALIGNS_WITH` relationship between milestones across frameworks is something no competitor offers. It enables centers that use multiple frameworks (common for NAEYC-accredited programs also reporting to state standards) to see unified developmental profiles. This is extremely difficult to model relationally.

3. **AI-powered learning pathways.** Prerequisite relationships between milestones, combined with interest detection, enable AI-driven curriculum recommendations that are grounded in the graph structure. This is the "AI-native" differentiator described in the README -- it becomes architecturally natural rather than bolted on.

4. **Rich developmental analytics.** Cohort comparisons, developmental trend detection, and gap analysis are natural graph queries. "Which milestones in social-emotional development do children in the Butterfly classroom typically achieve before transitioning to pre-K?" is a traversal, not a complex multi-join query.

5. **Relational integrity where it matters.** Billing, attendance, subsidies, and compliance data stay in PostgreSQL with full ACID guarantees, foreign keys, and RLS. The graph handles the domain where relationships and traversals matter more than transactions.

6. **Visualization-ready data model.** Graph data naturally maps to visual representations: developmental timelines, milestone maps, learning pathway diagrams, and interest webs. Parent-facing portfolio views and educator dashboards can render directly from graph query results.

---

## Cons

1. **Polyglot persistence complexity.** Running two database engines (PostgreSQL + Neo4j) doubles operational overhead: separate backups, monitoring, scaling, security hardening, and capacity planning. The team must be proficient in both SQL and Cypher.

2. **Data synchronization is non-trivial.** Keeping child records, observations, and milestone assessments consistent between PostgreSQL and Neo4j requires a reliable sync mechanism. Sync failures or lag can cause data inconsistencies visible to users (e.g., a teacher creates an observation in PostgreSQL, but the portfolio view reads from the graph and does not yet see it).

3. **Neo4j licensing and cost.** Neo4j Community Edition is open source but limited (no clustering, no role-based access). Neo4j Enterprise requires a commercial license. Apache AGE avoids this but is less mature and has a smaller ecosystem.

4. **Graph database scaling patterns are less familiar.** Neo4j scales differently from PostgreSQL. Sharding a graph database is inherently harder because relationship-heavy queries may span shard boundaries. For a single-tenant childcare center this is not an issue, but for a multi-tenant SaaS serving thousands of centers it requires careful capacity planning.

5. **Smaller hiring pool.** Developers with graph database experience are less common than PostgreSQL/relational developers. In an industry (ECE SaaS) that already competes for engineering talent, this is a meaningful constraint.

6. **Apache AGE trade-off.** If using Apache AGE to avoid polyglot persistence, the graph query capabilities are more limited than Neo4j, the community is smaller, and advanced features (graph algorithms, GDS library) are not available. AGE is suitable for the framework hierarchy use case but may not scale to the full graph vision.

7. **Over-engineering risk for MVP.** The graph database delivers its strongest benefits for cross-framework alignment, AI learning pathways, and cohort analytics -- features that are in the "nice-to-have" backlog rather than the MVP feature set. For an MVP, a simpler approach (Suggestion 3) may reach market faster.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Graph Database | Neo4j 5.x (AuraDB managed) or Apache AGE 1.5+ (PostgreSQL extension) |
| Operational Database | PostgreSQL 16+ |
| Graph query language | Cypher (Neo4j) or openCypher (AGE) |
| Sync mechanism | Outbox pattern with async worker, or Debezium CDC |
| API layer | GraphQL (Apollo Server) mapping to both Cypher and SQL resolvers |
| ORM | Prisma for PostgreSQL; Neo4j JavaScript driver or neo4j-graphql for graph |
| Multi-tenancy | RLS in PostgreSQL; property-based filtering in Neo4j (`organizationId` on all nodes) |
| File storage | S3-compatible object storage |
| Search | Neo4j full-text indexes for observation narrative search |

### Apache AGE Alternative (Single Database)

If operational simplicity is the priority, Apache AGE provides graph capabilities within PostgreSQL:

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with Apache AGE extension |
| Graph queries | openCypher via AGE SQL functions |
| ORM | Prisma for relational; raw SQL/AGE for graph queries |
| Advantage | Single database, single backup, no sync needed |
| Trade-off | Less mature graph features, no graph algorithms library |

---

## Migration and Scaling Considerations

- **Start with AGE, graduate to Neo4j.** Begin with Apache AGE to validate the graph model within PostgreSQL. If graph query volume and complexity justify it, migrate the graph data to a dedicated Neo4j instance. The Cypher query language is compatible between AGE and Neo4j, minimizing application changes.

- **Graph data is derived, not authoritative.** PostgreSQL remains the source of truth for all data. The graph is a derived, queryable projection. If the graph becomes corrupted or needs rebuilding, it can be fully reconstructed from PostgreSQL data. This makes the graph operationally safer.

- **Phased graph adoption:** 
  - Phase 1 (MVP): Store frameworks as JSONB in PostgreSQL (Suggestion 3 approach). Build the observation and milestone assessment tables relationally.
  - Phase 2 (v1.1): Add Apache AGE. Load framework hierarchies and milestone relationships into the graph. Implement portfolio queries via Cypher.
  - Phase 3 (v2.0): If query patterns demand it, migrate to Neo4j AuraDB. Add cross-framework alignment, learning pathways, and interest detection.

- **Multi-tenancy in the graph.** Every node carries an `organizationId` property. Graph queries always filter by `organizationId`. For Neo4j, use database-per-tenant for large operators or property-based filtering for smaller ones.

- **COPPA compliance.** When deleting a child's data, remove all graph nodes and relationships associated with that child. With derived graph data, this means deleting from PostgreSQL (source of truth) and then removing the corresponding graph subgraph.

- **Backup strategy.** PostgreSQL: standard pg_dump or continuous archiving (WAL-G). Neo4j: neo4j-admin dump for full backups, or AuraDB managed backups. AGE: backed up as part of the PostgreSQL backup.

- **Performance monitoring.** Monitor graph query performance separately from SQL query performance. Neo4j provides query profiling via `PROFILE` and `EXPLAIN`. Set up alerting on slow graph queries, especially cohort analytics that scan many nodes.
