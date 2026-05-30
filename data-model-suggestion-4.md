# Data Model Suggestion 4: Multi-Engine Architecture with Apache Iceberg Data Lakehouse

## Overview

This model uses a polyglot persistence strategy centred on an Apache Iceberg data lakehouse as the primary archive store, with PostgreSQL for operational metadata, Neo4j for data lineage and relationship graphs, and TimescaleDB for time-series compliance metrics. The core insight is that a data retention platform at enterprise scale is fundamentally an archival system managing petabytes of heterogeneous data with complex lifecycle rules -- a workload that aligns more naturally with lakehouse architecture than traditional OLTP databases.

Apache Iceberg provides ACID transactions, schema evolution, time-travel queries, partition evolution, and hidden partitioning over object storage (S3/MinIO). This means archived records can be stored directly in open-format Parquet files on immutable object storage (satisfying WORM requirements for SEC 17a-4), while still being queryable via standard SQL through engines like Trino, Spark, or DuckDB. The lakehouse becomes the compliance-grade archive, PostgreSQL handles the operational workflow, and Neo4j maps the complex web of relationships between records, policies, holds, custodians, and matters.

This approach is purpose-built for organisations at the higher end of the scale spectrum -- those with hundreds of millions to billions of archived records across dozens of data sources, where a single relational database would struggle with both the volume and the analytical query patterns required by compliance reporting and eDiscovery.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                      API Layer                            │
└────────┬──────────────┬──────────────┬───────────────────┘
         │              │              │
┌────────▼────────┐  ┌──▼──────────┐  ┌▼──────────────────┐
│  PostgreSQL     │  │  Apache     │  │  Neo4j            │
│  (Operational)  │  │  Iceberg    │  │  (Lineage &       │
│                 │  │  Lakehouse  │  │   Relationships)  │
│  - Users        │  │             │  │                   │
│  - Policies     │  │  - Archived │  │  - Record->Policy │
│  - Legal Holds  │  │    Records  │  │  - Record->Hold   │
│  - Dispositions │  │  - Audit    │  │  - Custodian->     │
│  - Workflows    │  │    Events   │  │    DataSource     │
│  - Connectors   │  │  - Content  │  │  - Matter->Hold-> │
│                 │  │    Index    │  │    Record chain   │
└────────┬────────┘  └──────┬──────┘  │  - Policy conflict│
         │                  │         │    graph          │
         │          ┌───────▼──────┐  └───────────────────┘
         │          │  Object      │
         │          │  Storage     │         ┌─────────────┐
         │          │  (S3/MinIO)  │         │ TimescaleDB │
         │          │  with WORM   │         │ (Metrics &  │
         │          └──────────────┘         │  Time-Series│
         │                                  │  Analytics) │
         └──────────────────────────────────┘
```

---

## Component 1: PostgreSQL -- Operational Metadata

PostgreSQL handles the transactional, workflow-oriented data: user management, policy definitions, legal holds, disposition workflows, and connector configurations. These are low-volume, high-consistency entities that benefit from relational integrity and ACID transactions.

```sql
-- ============================================================
-- POSTGRESQL: OPERATIONAL METADATA
-- ============================================================

-- Organisation & Users (same as relational model)
CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    slug                TEXT NOT NULL UNIQUE,
    billing_plan        TEXT NOT NULL DEFAULT 'standard',
    data_residency      TEXT NOT NULL DEFAULT 'us-east-1',
    settings            JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    email               TEXT NOT NULL,
    full_name           TEXT NOT NULL,
    role                TEXT NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

-- Data Sources
CREATE TABLE data_sources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    source_type         TEXT NOT NULL,
    display_name        TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'pending',
    connection_config   JSONB NOT NULL DEFAULT '{}',
    sync_state          JSONB NOT NULL DEFAULT '{}',
    last_sync_at        TIMESTAMPTZ,
    total_items_synced  BIGINT NOT NULL DEFAULT 0,
    total_bytes_synced  BIGINT NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Retention Policies
CREATE TABLE retention_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL DEFAULT 'draft',
    retention_period_days INTEGER NOT NULL,
    retention_trigger   TEXT NOT NULL DEFAULT 'creation_date',
    disposition_action  TEXT NOT NULL DEFAULT 'delete',
    scope_conditions    JSONB NOT NULL DEFAULT '{}',
    approval_workflow   JSONB NOT NULL DEFAULT '{}',
    regulatory_framework TEXT,
    regulatory_citation TEXT,
    priority            INTEGER NOT NULL DEFAULT 100,
    created_by          UUID REFERENCES users(id),
    effective_from      TIMESTAMPTZ,
    effective_until     TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Legal Matters
CREATE TABLE legal_matters (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_number       TEXT NOT NULL,
    name                TEXT NOT NULL,
    matter_type         TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'active',
    matter_details      JSONB NOT NULL DEFAULT '{}',
    lead_attorney_id    UUID REFERENCES users(id),
    opened_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at           TIMESTAMPTZ,
    created_by          UUID NOT NULL REFERENCES users(id),
    UNIQUE (organisation_id, matter_number)
);

-- Legal Holds
CREATE TABLE legal_holds (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_id           UUID NOT NULL REFERENCES legal_matters(id),
    hold_name           TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL DEFAULT 'draft',
    scope_definition    JSONB NOT NULL DEFAULT '{}',
    ai_scope_suggestions JSONB NOT NULL DEFAULT '{}',
    issued_at           TIMESTAMPTZ,
    issued_by           UUID REFERENCES users(id),
    released_at         TIMESTAMPTZ,
    released_by         UUID REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Custodians
CREATE TABLE custodians (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID REFERENCES users(id),
    external_name       TEXT,
    external_email      TEXT,
    department          TEXT,
    employment_status   TEXT DEFAULT 'active',
    data_source_access  JSONB NOT NULL DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE hold_custodian_assignments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hold_id             UUID NOT NULL REFERENCES legal_holds(id),
    custodian_id        UUID NOT NULL REFERENCES custodians(id),
    status              TEXT NOT NULL DEFAULT 'pending',
    notified_at         TIMESTAMPTZ,
    acknowledged_at     TIMESTAMPTZ,
    reminder_count      INTEGER NOT NULL DEFAULT 0,
    notification_history JSONB NOT NULL DEFAULT '[]',
    UNIQUE (hold_id, custodian_id)
);

-- Disposition Workflows
CREATE TABLE disposition_batches (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    batch_number        TEXT NOT NULL,
    policy_id           UUID NOT NULL REFERENCES retention_policies(id),
    status              TEXT NOT NULL DEFAULT 'pending_review',
    total_records       INTEGER NOT NULL DEFAULT 0,
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    records_deleted     INTEGER NOT NULL DEFAULT 0,
    records_excepted    INTEGER NOT NULL DEFAULT 0,
    batch_summary       JSONB NOT NULL DEFAULT '{}',
    execution_log       JSONB NOT NULL DEFAULT '[]',
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at        TIMESTAMPTZ,
    UNIQUE (organisation_id, batch_number)
);

CREATE TABLE disposition_approvals (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id            UUID NOT NULL REFERENCES disposition_batches(id),
    approver_id         UUID NOT NULL REFERENCES users(id),
    approval_order      INTEGER NOT NULL,
    decision            TEXT,
    comments            TEXT,
    decided_at          TIMESTAMPTZ,
    UNIQUE (batch_id, approver_id)
);

CREATE TABLE deletion_certificates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    batch_id            UUID NOT NULL REFERENCES disposition_batches(id),
    certificate_number  TEXT NOT NULL UNIQUE,
    records_count       INTEGER NOT NULL,
    total_bytes         BIGINT NOT NULL,
    content_hash        BYTEA NOT NULL,
    previous_cert_hash  BYTEA,
    chain_sequence      BIGINT NOT NULL,
    certificate_content JSONB NOT NULL,
    issued_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    issued_by           UUID NOT NULL REFERENCES users(id)
);

-- eDiscovery Collections & Productions
CREATE TABLE collections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_id           UUID NOT NULL REFERENCES legal_matters(id),
    name                TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'collecting',
    total_items         INTEGER NOT NULL DEFAULT 0,
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    query_spec          JSONB NOT NULL DEFAULT '{}',
    collected_by        UUID NOT NULL REFERENCES users(id),
    collected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE productions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    collection_id       UUID NOT NULL REFERENCES collections(id),
    matter_id           UUID NOT NULL REFERENCES legal_matters(id),
    production_number   TEXT NOT NULL,
    format              TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'preparing',
    total_items         INTEGER NOT NULL DEFAULT 0,
    production_config   JSONB NOT NULL DEFAULT '{}',
    created_by          UUID NOT NULL REFERENCES users(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    delivered_at        TIMESTAMPTZ,
    UNIQUE (organisation_id, production_number)
);

-- Record-level operational state (lightweight reference to Iceberg records)
CREATE TABLE record_operations (
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    lifecycle_status    TEXT NOT NULL DEFAULT 'active',
    hold_count          INTEGER NOT NULL DEFAULT 0,
    governing_policy_id UUID REFERENCES retention_policies(id),
    retention_expiry    TIMESTAMPTZ,
    data_class          TEXT,
    classification_verified BOOLEAN NOT NULL DEFAULT false,
    last_action_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (record_id, organisation_id)
);

CREATE INDEX idx_record_ops_expiry ON record_operations(organisation_id, retention_expiry)
    WHERE lifecycle_status = 'active' AND hold_count = 0;
CREATE INDEX idx_record_ops_holds ON record_operations(organisation_id)
    WHERE hold_count > 0;
CREATE INDEX idx_record_ops_status ON record_operations(organisation_id, lifecycle_status);

-- Hold-to-record junction (operational)
CREATE TABLE hold_records (
    hold_id             UUID NOT NULL REFERENCES legal_holds(id),
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    placed_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    released_at         TIMESTAMPTZ,
    PRIMARY KEY (hold_id, record_id)
);
```

---

## Component 2: Apache Iceberg Data Lakehouse -- Archive Store

The Iceberg lakehouse stores the actual archived record metadata and content references at scale. This is the compliance-grade archive that satisfies WORM requirements via S3 Object Lock, provides time-travel for point-in-time queries, and supports schema evolution as new connector types are added.

### Table Definitions (Iceberg SQL via Trino/Spark)

```sql
-- ============================================================
-- ICEBERG: ARCHIVED RECORDS TABLE
-- Stored as Parquet files on S3 with Object Lock for WORM compliance.
-- Partitioned by organisation, content type, and ingestion month.
-- ============================================================

CREATE TABLE iceberg.retention.archived_records (
    -- Identity
    record_id           VARCHAR NOT NULL,           -- UUID as string
    organisation_id     VARCHAR NOT NULL,
    data_source_id      VARCHAR NOT NULL,
    source_type         VARCHAR NOT NULL,
    original_id         VARCHAR NOT NULL,
    
    -- Content metadata
    content_type        VARCHAR NOT NULL,           -- email, chat_message, document, etc.
    subject             VARCHAR,
    body_text           VARCHAR,                    -- extracted plaintext for search indexing
    body_hash           VARBINARY NOT NULL,         -- SHA-256
    file_size_bytes     BIGINT NOT NULL,
    original_format     VARCHAR NOT NULL,
    
    -- Participants
    sender              VARCHAR,
    recipients          ARRAY(VARCHAR),
    participants        ARRAY(VARCHAR),
    
    -- Dates
    source_created_at   TIMESTAMP(6) WITH TIME ZONE,
    source_modified_at  TIMESTAMP(6) WITH TIME ZONE,
    ingested_at         TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    
    -- Storage
    content_storage_key VARCHAR NOT NULL,           -- S3 key to original content
    content_bucket      VARCHAR NOT NULL,
    content_region      VARCHAR NOT NULL,
    storage_tier        VARCHAR NOT NULL,           -- hot, warm, cold, glacier
    
    -- Classification
    data_class          VARCHAR,
    classification_method VARCHAR,                  -- ai, manual, rule
    classification_confidence DOUBLE,
    
    -- Source-specific metadata (schema-on-read flexibility)
    source_metadata     VARCHAR,                    -- JSON string; parsed by query engine
    
    -- AI enrichment
    enrichment_data     VARCHAR,                    -- JSON string
    
    -- Tags
    custom_tags         ARRAY(VARCHAR),
    
    -- Partitioning columns
    ingestion_year      INTEGER NOT NULL,
    ingestion_month     INTEGER NOT NULL
)
WITH (
    format = 'PARQUET',
    partitioning = ARRAY[
        'organisation_id',
        'content_type',
        'ingestion_year',
        'ingestion_month'
    ],
    sorted_by = ARRAY['ingested_at'],
    -- WORM compliance via S3 Object Lock
    location = 's3://retention-archive/records/',
    -- Snapshot retention for time-travel (regulatory: keep 7 years)
    'history.expire.max-snapshot-age-ms' = '220752000000',  -- ~7 years
    -- Write properties
    'write.parquet.compression-codec' = 'ZSTD',
    'write.metadata.compression-codec' = 'gzip',
    'write.distribution-mode' = 'hash',
    'write.target-file-size-bytes' = '536870912'  -- 512 MB target file size
);

-- ============================================================
-- ICEBERG: AUDIT EVENT LOG
-- Immutable, append-only audit trail stored in lakehouse.
-- ============================================================

CREATE TABLE iceberg.retention.audit_events (
    event_id            VARCHAR NOT NULL,
    global_sequence     BIGINT NOT NULL,
    organisation_id     VARCHAR NOT NULL,
    
    -- Event identification
    event_type          VARCHAR NOT NULL,
    event_category      VARCHAR NOT NULL,
    severity            VARCHAR NOT NULL,
    
    -- Actor
    actor_id            VARCHAR,
    actor_type          VARCHAR NOT NULL,
    actor_email         VARCHAR,
    actor_ip            VARCHAR,
    
    -- Target
    target_type         VARCHAR,
    target_id           VARCHAR,
    
    -- Content
    summary             VARCHAR NOT NULL,
    event_details       VARCHAR,                    -- JSON string
    
    -- Tamper-evidence
    previous_hash       VARBINARY,
    event_hash          VARBINARY NOT NULL,
    
    -- Timestamps
    occurred_at         TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    
    -- Partitioning
    event_year          INTEGER NOT NULL,
    event_month         INTEGER NOT NULL
)
WITH (
    format = 'PARQUET',
    partitioning = ARRAY[
        'organisation_id',
        'event_year',
        'event_month'
    ],
    sorted_by = ARRAY['global_sequence'],
    location = 's3://retention-archive/audit/',
    'write.parquet.compression-codec' = 'ZSTD',
    'write.wap.enabled' = 'true'  -- Write-Audit-Publish for staging writes
);

-- ============================================================
-- ICEBERG: CONTENT INDEX TABLE
-- Deduplicated content references for cross-record deduplication.
-- ============================================================

CREATE TABLE iceberg.retention.content_index (
    content_hash        VARBINARY NOT NULL,         -- SHA-256 of content
    organisation_id     VARCHAR NOT NULL,
    
    -- Content location
    storage_bucket      VARCHAR NOT NULL,
    storage_key         VARCHAR NOT NULL,
    storage_region      VARCHAR NOT NULL,
    content_size_bytes  BIGINT NOT NULL,
    original_format     VARCHAR NOT NULL,
    
    -- Deduplication
    reference_count     INTEGER NOT NULL DEFAULT 1,  -- how many records reference this content
    first_seen_at       TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    
    -- Immutability metadata
    worm_lock_until     TIMESTAMP(6) WITH TIME ZONE, -- S3 Object Lock retention date
    worm_mode           VARCHAR,                      -- GOVERNANCE or COMPLIANCE
    
    -- Content extraction status
    text_extracted      BOOLEAN NOT NULL DEFAULT FALSE,
    ocr_processed       BOOLEAN NOT NULL DEFAULT FALSE,
    
    -- Partitioning
    first_seen_year     INTEGER NOT NULL
)
WITH (
    format = 'PARQUET',
    partitioning = ARRAY['organisation_id', 'first_seen_year'],
    location = 's3://retention-archive/content-index/'
);

-- ============================================================
-- ICEBERG: DISPOSITION RECORDS (tombstones for deleted records)
-- When records are deleted, their metadata is preserved here
-- as proof of defensible deletion.
-- ============================================================

CREATE TABLE iceberg.retention.disposition_tombstones (
    record_id           VARCHAR NOT NULL,
    organisation_id     VARCHAR NOT NULL,
    
    -- What was deleted
    content_type        VARCHAR NOT NULL,
    data_class          VARCHAR,
    file_size_bytes     BIGINT NOT NULL,
    source_type         VARCHAR NOT NULL,
    source_created_at   TIMESTAMP(6) WITH TIME ZONE,
    ingested_at         TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    
    -- Why it was deleted
    policy_name         VARCHAR NOT NULL,
    policy_id           VARCHAR NOT NULL,
    regulatory_citation VARCHAR,
    batch_id            VARCHAR NOT NULL,
    certificate_number  VARCHAR NOT NULL,
    
    -- When and by whom
    deleted_at          TIMESTAMP(6) WITH TIME ZONE NOT NULL,
    approved_by         ARRAY(VARCHAR) NOT NULL,
    
    -- Verification
    content_hash        VARBINARY NOT NULL,
    content_verified_deleted BOOLEAN NOT NULL,
    storage_keys_removed ARRAY(VARCHAR) NOT NULL,
    
    -- Narrative
    disposition_narrative VARCHAR NOT NULL,
    
    -- Partitioning
    deletion_year       INTEGER NOT NULL,
    deletion_month      INTEGER NOT NULL
)
WITH (
    format = 'PARQUET',
    partitioning = ARRAY['organisation_id', 'deletion_year'],
    location = 's3://retention-archive/tombstones/',
    'write.parquet.compression-codec' = 'ZSTD'
);
```

### Iceberg Time-Travel Queries

```sql
-- Query records as they existed on a specific date (point-in-time compliance query)
SELECT *
FROM iceberg.retention.archived_records
FOR TIMESTAMP AS OF TIMESTAMP '2025-03-15 00:00:00 UTC'
WHERE organisation_id = 'org-uuid'
  AND content_type = 'email'
  AND data_class = 'pii';

-- Query a specific snapshot (by snapshot ID from Iceberg metadata)
SELECT *
FROM iceberg.retention.archived_records
FOR VERSION AS OF 8234567890123;

-- View snapshot history for audit purposes
SELECT snapshot_id, committed_at, operation, summary
FROM iceberg.retention."archived_records$snapshots"
ORDER BY committed_at DESC;

-- View file-level manifests for storage audit
SELECT file_path, record_count, file_size_in_bytes, partition
FROM iceberg.retention."archived_records$files"
WHERE partition.organisation_id = 'org-uuid';
```

---

## Component 3: Neo4j -- Relationship Graph

Neo4j models the complex web of relationships that are central to retention management: which policies govern which records, which holds freeze which records, which custodians are connected to which data sources, how matters relate to holds and collections, and where policy conflicts arise. Graph queries answer questions that would require multiple expensive JOINs in a relational database.

### Graph Schema (Cypher DDL)

```cypher
// ============================================================
// NEO4J: NODE TYPES
// ============================================================

// Record node (lightweight reference -- full data in Iceberg)
CREATE CONSTRAINT record_id IF NOT EXISTS
FOR (r:Record) REQUIRE r.record_id IS UNIQUE;

// Properties: record_id, organisation_id, content_type, data_class,
//             lifecycle_status, ingested_at, retention_expiry

// Policy node
CREATE CONSTRAINT policy_id IF NOT EXISTS
FOR (p:Policy) REQUIRE p.policy_id IS UNIQUE;

// Properties: policy_id, organisation_id, name, status, retention_period_days,
//             disposition_action, regulatory_framework, priority

// Legal Hold node
CREATE CONSTRAINT hold_id IF NOT EXISTS
FOR (h:LegalHold) REQUIRE h.hold_id IS UNIQUE;

// Properties: hold_id, organisation_id, hold_name, status, matter_id

// Matter node
CREATE CONSTRAINT matter_id IF NOT EXISTS
FOR (m:Matter) REQUIRE m.matter_id IS UNIQUE;

// Properties: matter_id, organisation_id, matter_number, name, status, matter_type

// Custodian node
CREATE CONSTRAINT custodian_id IF NOT EXISTS
FOR (c:Custodian) REQUIRE c.custodian_id IS UNIQUE;

// Properties: custodian_id, organisation_id, name, email, department

// DataSource node
CREATE CONSTRAINT datasource_id IF NOT EXISTS
FOR (ds:DataSource) REQUIRE ds.source_id IS UNIQUE;

// Properties: source_id, organisation_id, source_type, display_name, status

// Organisation node
CREATE CONSTRAINT org_id IF NOT EXISTS
FOR (o:Organisation) REQUIRE o.org_id IS UNIQUE;

// Jurisdiction node
CREATE CONSTRAINT jurisdiction_code IF NOT EXISTS
FOR (j:Jurisdiction) REQUIRE j.code IS UNIQUE;

// DataClass node
CREATE CONSTRAINT dataclass_code IF NOT EXISTS
FOR (dc:DataClass) REQUIRE (dc.organisation_id, dc.code) IS UNIQUE;

// DispositionBatch node
CREATE CONSTRAINT batch_id IF NOT EXISTS
FOR (db:DispositionBatch) REQUIRE db.batch_id IS UNIQUE;

// Collection node
CREATE CONSTRAINT collection_id IF NOT EXISTS
FOR (col:Collection) REQUIRE col.collection_id IS UNIQUE;

// ============================================================
// NEO4J: RELATIONSHIP TYPES
// ============================================================

// Record relationships
// (:Record)-[:GOVERNED_BY {computed_expiry, is_governing, assigned_at}]->(:Policy)
// (:Record)-[:HELD_BY {placed_at, released_at}]->(:LegalHold)
// (:Record)-[:INGESTED_FROM {original_id, ingested_at}]->(:DataSource)
// (:Record)-[:CLASSIFIED_AS {confidence, method, verified}]->(:DataClass)
// (:Record)-[:IN_COLLECTION {added_at, review_status}]->(:Collection)
// (:Record)-[:DISPOSED_IN {deleted_at}]->(:DispositionBatch)

// Policy relationships
// (:Policy)-[:APPLIES_TO_CLASS]->(:DataClass)
// (:Policy)-[:APPLIES_IN]->(:Jurisdiction)
// (:Policy)-[:APPLIES_TO_SOURCE_TYPE]->(:DataSource)
// (:Policy)-[:CONFLICTS_WITH {conflict_type, description, status}]->(:Policy)

// Legal Hold relationships
// (:LegalHold)-[:BELONGS_TO]->(:Matter)
// (:LegalHold)-[:PRESERVES {placed_at}]->(:Record)
// (:Custodian)-[:ASSIGNED_TO {status, notified_at, acknowledged_at}]->(:LegalHold)

// Custodian relationships
// (:Custodian)-[:HAS_ACCESS_TO {mailbox, drive_id}]->(:DataSource)
// (:Custodian)-[:WORKS_IN]->(:Organisation)

// Matter relationships
// (:Matter)-[:HAS_HOLD]->(:LegalHold)
// (:Matter)-[:HAS_COLLECTION]->(:Collection)

// Collection relationships
// (:Collection)-[:PRODUCED_AS]->(:Production)

// Organisation relationships
// (:Organisation)-[:OPERATES_IN]->(:Jurisdiction)
// (:Organisation)-[:OWNS]->(:DataSource)
```

### Example Graph Queries

```cypher
// 1. Find all records that cannot be deleted because they are under hold,
//    even though their retention has expired
MATCH (r:Record)-[:HELD_BY]->(h:LegalHold {status: 'active'})
WHERE r.organisation_id = $org_id
  AND r.retention_expiry < datetime()
  AND r.lifecycle_status = 'active'
RETURN r.record_id, r.content_type, r.retention_expiry,
       collect(h.hold_name) AS active_holds,
       count(h) AS hold_count
ORDER BY r.retention_expiry ASC;

// 2. Impact analysis: if we release a legal hold, which records become
//    eligible for disposition?
MATCH (r:Record)-[:HELD_BY]->(h:LegalHold {hold_id: $hold_id})
WHERE NOT EXISTS {
    MATCH (r)-[:HELD_BY]->(other:LegalHold {status: 'active'})
    WHERE other.hold_id <> $hold_id
}
AND r.retention_expiry < datetime()
RETURN count(r) AS records_eligible_for_disposition,
       sum(r.file_size_bytes) AS total_bytes;

// 3. Custodian data map: all data sources and record counts for a custodian
MATCH (c:Custodian {custodian_id: $custodian_id})-[:HAS_ACCESS_TO]->(ds:DataSource)
OPTIONAL MATCH (r:Record)-[:INGESTED_FROM]->(ds)
WHERE r.organisation_id = c.organisation_id
RETURN ds.display_name, ds.source_type,
       count(r) AS record_count,
       sum(r.file_size_bytes) AS total_bytes;

// 4. Policy conflict analysis: find all policy pairs that apply to the
//    same records with conflicting retention periods
MATCH (r:Record)-[:GOVERNED_BY]->(p1:Policy),
      (r)-[:GOVERNED_BY]->(p2:Policy)
WHERE p1.policy_id < p2.policy_id  // avoid duplicates
  AND p1.organisation_id = $org_id
  AND abs(p1.retention_period_days - p2.retention_period_days) > 365
RETURN p1.name AS policy_1, p2.name AS policy_2,
       p1.retention_period_days AS retention_1,
       p2.retention_period_days AS retention_2,
       count(r) AS affected_records
ORDER BY affected_records DESC;

// 5. Chain of custody: trace a record's full lifecycle through the system
MATCH path = (r:Record {record_id: $record_id})-[*1..5]-(related)
RETURN path;

// 6. Coverage gap detection: find data sources with records not governed by any policy
MATCH (ds:DataSource {organisation_id: $org_id})<-[:INGESTED_FROM]-(r:Record)
WHERE NOT EXISTS {
    MATCH (r)-[:GOVERNED_BY]->(:Policy)
}
RETURN ds.display_name, ds.source_type,
       count(r) AS uncovered_records,
       sum(r.file_size_bytes) AS uncovered_bytes
ORDER BY uncovered_records DESC;

// 7. GDPR erasure impact: find all records linked to a specific person
//    across all data sources (right to erasure request)
MATCH (c:Custodian {external_email: $email})-[:HAS_ACCESS_TO]->(ds:DataSource)
      <-[:INGESTED_FROM]-(r:Record)
OPTIONAL MATCH (r)-[:HELD_BY]->(h:LegalHold {status: 'active'})
RETURN r.record_id, r.content_type, ds.display_name,
       r.lifecycle_status,
       CASE WHEN h IS NOT NULL THEN true ELSE false END AS blocked_by_hold,
       h.hold_name AS blocking_hold
ORDER BY r.ingested_at;
```

---

## Component 4: TimescaleDB -- Time-Series Compliance Metrics

TimescaleDB (a PostgreSQL extension) captures time-series data for compliance dashboards, storage analytics, SLA monitoring, and trend analysis. This data is inherently temporal and benefits from TimescaleDB's hypertables, continuous aggregates, and native retention policies.

```sql
-- ============================================================
-- TIMESCALEDB: COMPLIANCE METRICS
-- ============================================================

-- Create hypertable for record ingestion metrics
CREATE TABLE metrics_ingestion (
    time                TIMESTAMPTZ NOT NULL,
    organisation_id     UUID NOT NULL,
    data_source_id      UUID NOT NULL,
    source_type         TEXT NOT NULL,
    content_type        TEXT NOT NULL,
    records_ingested    INTEGER NOT NULL DEFAULT 0,
    bytes_ingested      BIGINT NOT NULL DEFAULT 0,
    records_classified  INTEGER NOT NULL DEFAULT 0,
    records_failed      INTEGER NOT NULL DEFAULT 0,
    avg_classification_confidence DOUBLE PRECISION,
    p95_ingest_latency_ms INTEGER
);

SELECT create_hypertable('metrics_ingestion', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Storage tier metrics
CREATE TABLE metrics_storage (
    time                TIMESTAMPTZ NOT NULL,
    organisation_id     UUID NOT NULL,
    storage_tier        TEXT NOT NULL,
    content_type        TEXT NOT NULL,
    record_count        BIGINT NOT NULL,
    total_bytes         BIGINT NOT NULL,
    estimated_monthly_cost_cents INTEGER
);

SELECT create_hypertable('metrics_storage', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Policy compliance metrics
CREATE TABLE metrics_compliance (
    time                TIMESTAMPTZ NOT NULL,
    organisation_id     UUID NOT NULL,
    
    -- Policy coverage
    total_records       BIGINT NOT NULL,
    records_with_policy BIGINT NOT NULL,
    records_without_policy BIGINT NOT NULL,
    coverage_percentage DOUBLE PRECISION NOT NULL,
    
    -- Hold status
    active_holds        INTEGER NOT NULL,
    records_under_hold  BIGINT NOT NULL,
    pending_acknowledgements INTEGER NOT NULL,
    
    -- Disposition pipeline
    records_pending_disposition BIGINT NOT NULL,
    records_overdue_disposition BIGINT NOT NULL,  -- expired but not yet in batch
    batches_awaiting_approval INTEGER NOT NULL,
    
    -- Conflicts
    active_policy_conflicts INTEGER NOT NULL,
    
    -- Classification
    records_unclassified BIGINT NOT NULL,
    avg_classification_confidence DOUBLE PRECISION
);

SELECT create_hypertable('metrics_compliance', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- Connector health metrics
CREATE TABLE metrics_connector_health (
    time                TIMESTAMPTZ NOT NULL,
    organisation_id     UUID NOT NULL,
    data_source_id      UUID NOT NULL,
    source_type         TEXT NOT NULL,
    sync_status         TEXT NOT NULL,
    sync_duration_ms    INTEGER,
    items_synced        INTEGER NOT NULL DEFAULT 0,
    errors_count        INTEGER NOT NULL DEFAULT 0,
    api_calls_made      INTEGER NOT NULL DEFAULT 0,
    rate_limit_hits     INTEGER NOT NULL DEFAULT 0
);

SELECT create_hypertable('metrics_connector_health', 'time',
    chunk_time_interval => INTERVAL '1 day');

-- ============================================================
-- CONTINUOUS AGGREGATES (materialized views with automatic refresh)
-- ============================================================

-- Hourly ingestion summary
CREATE MATERIALIZED VIEW metrics_ingestion_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    organisation_id,
    source_type,
    content_type,
    sum(records_ingested) AS total_records,
    sum(bytes_ingested) AS total_bytes,
    avg(avg_classification_confidence) AS avg_confidence
FROM metrics_ingestion
GROUP BY bucket, organisation_id, source_type, content_type;

-- Daily compliance summary
CREATE MATERIALIZED VIEW metrics_compliance_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    organisation_id,
    avg(coverage_percentage) AS avg_coverage,
    max(active_holds) AS max_active_holds,
    max(records_overdue_disposition) AS max_overdue,
    max(active_policy_conflicts) AS max_conflicts
FROM metrics_compliance
GROUP BY bucket, organisation_id;

-- Weekly storage trend
CREATE MATERIALIZED VIEW metrics_storage_weekly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('7 days', time) AS bucket,
    organisation_id,
    storage_tier,
    avg(record_count)::BIGINT AS avg_records,
    avg(total_bytes)::BIGINT AS avg_bytes,
    sum(estimated_monthly_cost_cents) / 7 AS avg_daily_cost_cents
FROM metrics_storage
GROUP BY bucket, organisation_id, storage_tier;

-- ============================================================
-- RETENTION POLICIES (for metrics data itself)
-- ============================================================

-- Keep raw metrics for 90 days
SELECT add_retention_policy('metrics_ingestion', INTERVAL '90 days');
SELECT add_retention_policy('metrics_connector_health', INTERVAL '90 days');

-- Keep compliance metrics for 3 years (regulatory requirement)
SELECT add_retention_policy('metrics_compliance', INTERVAL '3 years');

-- Keep storage metrics for 2 years
SELECT add_retention_policy('metrics_storage', INTERVAL '2 years');

-- Continuous aggregates live longer than raw data
-- Hourly aggregates: 1 year
-- Daily aggregates: 5 years
-- Weekly aggregates: 10 years

-- Compression for older chunks
SELECT add_compression_policy('metrics_ingestion', INTERVAL '7 days');
SELECT add_compression_policy('metrics_storage', INTERVAL '7 days');
SELECT add_compression_policy('metrics_compliance', INTERVAL '30 days');
SELECT add_compression_policy('metrics_connector_health', INTERVAL '7 days');
```

---

## Data Flow and Synchronisation

### Ingestion Pipeline

```
1. Connector pulls records from source (M365, Slack, etc.)
2. Content stored in S3 with Object Lock (WORM)
3. Record metadata written to Iceberg archived_records table
4. Record operational state inserted into PostgreSQL record_operations
5. Neo4j nodes and relationships created:
   - (:Record) node
   - (:Record)-[:INGESTED_FROM]->(:DataSource) relationship
6. Metrics emitted to TimescaleDB
7. OpenSearch index updated for full-text search
```

### Policy Assignment Pipeline

```
1. Policy engine evaluates new/updated records against active policies
2. PostgreSQL record_operations updated with governing policy
3. Neo4j (:Record)-[:GOVERNED_BY]->(:Policy) relationships created
4. Iceberg record updated with retention_expiry (via merge)
5. Conflict detection runs in Neo4j (graph query for overlapping policies)
6. Metrics updated in TimescaleDB
```

### Disposition Pipeline

```
1. Cron job queries PostgreSQL for records past retention_expiry with hold_count=0
2. Disposition batch created in PostgreSQL
3. Neo4j queried to verify no active holds (graph traversal)
4. Approval workflow runs in PostgreSQL
5. Upon approval:
   a. Content deleted from S3 (Object Lock must have expired)
   b. Iceberg record marked as deleted (soft delete via merge)
   c. Tombstone written to Iceberg disposition_tombstones table
   d. PostgreSQL record_operations updated
   e. Neo4j Record node updated, DISPOSED_IN relationship created
   f. Deletion certificate created in PostgreSQL
   g. Metrics updated in TimescaleDB
```

### Cross-Engine Consistency

```
Eventual consistency model with saga pattern:
- PostgreSQL is the source of truth for operational state
- Iceberg is the source of truth for archived content metadata
- Neo4j is a derived view (rebuildable from PostgreSQL + Iceberg)
- TimescaleDB is a derived view (rebuildable from event history)

Consistency guarantees:
- PostgreSQL -> Iceberg: transactional outbox pattern
- PostgreSQL -> Neo4j: CDC via Debezium
- Iceberg -> OpenSearch: Iceberg table scan + incremental sync
- All -> TimescaleDB: async metric emission with at-least-once delivery
```

---

## Pros and Cons

### Pros

1. **Purpose-built storage per workload.** Archived records live in columnar Parquet on object storage (cost-effective at petabyte scale), operational workflows use ACID-compliant PostgreSQL, relationship queries leverage Neo4j's native graph engine, and time-series metrics use TimescaleDB's hypertables. Each engine operates in its sweet spot.

2. **WORM compliance native to the architecture.** S3 Object Lock with Iceberg provides SEC 17a-4 compliant immutable storage without bolting WORM onto a database. The archive format is an open standard (Parquet), eliminating vendor lock-in for long-term preservation.

3. **Iceberg time-travel for regulatory queries.** "Show me the state of all archived records as of the date of the regulatory inquiry" is a first-class Iceberg feature, not a custom implementation. Time-travel works across schema evolution -- queries against a 2024 snapshot work even after 2026 schema changes.

4. **Graph queries for impact analysis.** "If we release this hold, which records become eligible for deletion?" and "which policies conflict and how many records are affected?" are natural graph traversals that would require complex multi-table JOINs in a relational model. Neo4j answers these in milliseconds.

5. **Cost-effective at scale.** Storing billions of records in Parquet on S3 costs a fraction of equivalent PostgreSQL storage. ZSTD compression typically achieves 5-10x compression on text-heavy archival metadata. Cold data in S3 Glacier costs $0.004/GB/month versus $0.10+/GB/month for database storage.

6. **Schema evolution without migration.** Iceberg supports adding, renaming, and reordering columns without rewriting data files. A new connector type that adds new metadata fields does not require a migration of existing Parquet files.

7. **Decoupled scaling.** PostgreSQL, Iceberg (query engine), Neo4j, and TimescaleDB scale independently. If eDiscovery search load spikes, add Trino workers. If hold relationship queries slow down, scale Neo4j. Storage scales automatically with S3.

8. **Open standards throughout.** Iceberg (Apache), Parquet (Apache), S3 API (de facto standard), and Cypher (openCypher) are all open. No proprietary lock-in at any layer.

### Cons

1. **Operational complexity.** Four distinct database engines plus an object storage layer means four sets of operational procedures, monitoring, backup, patching, and expertise requirements. This is significantly more complex than a single-engine approach.

2. **Cross-engine consistency is hard.** Maintaining consistency between PostgreSQL, Iceberg, Neo4j, and TimescaleDB requires saga patterns, outbox tables, CDC pipelines (Debezium), and careful idempotency. A failure in the Debezium connector can cause Neo4j to drift from PostgreSQL.

3. **Latency for cross-engine queries.** Answering "show me all email records under hold with their policy details and custodian acknowledgement status" requires data from Iceberg (record content), PostgreSQL (hold status, custodian acknowledgement), and Neo4j (relationship traversal). The application must orchestrate these queries and merge results.

4. **Team expertise requirements.** The team needs proficiency in PostgreSQL, Apache Iceberg (including Trino/Spark), Neo4j (Cypher query language), TimescaleDB, and the integration plumbing (Debezium, Kafka). Hiring and retaining engineers with this breadth of expertise is challenging.

5. **Neo4j is a derived view.** Neo4j must be kept in sync with PostgreSQL and Iceberg. If it becomes inconsistent, it must be rebuilt from the source systems. This rebuild process for billions of relationships can take hours.

6. **Iceberg query latency.** While Iceberg excels at analytical queries over large datasets, point lookups of individual records are slower than PostgreSQL (seconds vs. milliseconds). The `record_operations` table in PostgreSQL mitigates this for operational queries, but adds a synchronisation burden.

7. **Licensing costs.** Neo4j Enterprise (required for production clustering, RLS, and performance) is commercially licensed. The open-source Community Edition lacks key features like causal clustering and role-based access. This partially offsets the cost savings from S3-based storage.

8. **Cold start for eDiscovery.** Full-text search over Iceberg tables requires an external search engine (OpenSearch). The search index must be built and maintained separately, adding another component to the architecture.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Operational DB** | PostgreSQL 16+ | ACID transactions for workflows, RLS for multi-tenancy |
| **Archive store** | Apache Iceberg on S3/MinIO | Time-travel, schema evolution, WORM via Object Lock |
| **Query engine** | Trino (self-hosted) or AWS Athena | SQL over Iceberg tables; Trino for self-hosted, Athena for managed |
| **Graph database** | Neo4j 5.x Enterprise | Native graph for relationship queries and impact analysis |
| **Time-series** | TimescaleDB (PostgreSQL extension) | Runs on same PostgreSQL infra; hypertables, continuous aggregates |
| **Object storage** | MinIO (self-hosted) or AWS S3 | Object Lock for WORM; lifecycle policies for tiering |
| **Search engine** | OpenSearch 2.x | Full-text search over archived content |
| **CDC/sync** | Debezium + Kafka Connect | PostgreSQL CDC to Neo4j and OpenSearch |
| **Event streaming** | Apache Kafka | Event bus between components; durable, ordered delivery |
| **Orchestration** | Temporal or Apache Airflow | Saga orchestration for cross-engine workflows |
| **Monitoring** | Prometheus + Grafana | Unified metrics across all engines |
| **Data format** | Apache Parquet with ZSTD | Columnar, compressed, open standard |

---

## Migration and Scaling Considerations

### Initial Deployment (Proof of Concept)

For an initial deployment, simplify the architecture:

- PostgreSQL handles operational data (same as suggestion 1)
- Iceberg on MinIO for archive storage (single Trino coordinator + 2 workers)
- Skip Neo4j initially; use PostgreSQL for relationship queries
- TimescaleDB as a PostgreSQL extension on the same instance
- OpenSearch single node for search

This reduces the architecture to two engines (PostgreSQL + Trino/Iceberg) while preserving the ability to add Neo4j and scale TimescaleDB later.

### Growth Phase

- Add Neo4j when relationship queries become too expensive in PostgreSQL (typically when record counts exceed 100M and policy-to-record relationships exceed 500M)
- Scale Trino to 5-10 workers for parallel Iceberg queries
- Deploy Debezium for PostgreSQL-to-Neo4j CDC
- Separate TimescaleDB to dedicated instance
- Deploy Kafka cluster for event distribution

### Enterprise Scale

- Multi-region Iceberg with separate S3 buckets per data residency zone
- Neo4j causal cluster (3+ cores) per region
- Trino cluster per region with shared Iceberg catalog (Hive Metastore or Nessie)
- PostgreSQL with Citus for horizontal sharding of operational data
- TimescaleDB multi-node for distributed hypertables
- Cross-region Kafka (MirrorMaker 2) for event replication

### Data Migration Strategy

1. **From single-engine PostgreSQL**: Export archived_records to Parquet via COPY, register as Iceberg table
2. **Neo4j population**: Batch import from PostgreSQL using neo4j-admin import tool
3. **TimescaleDB backfill**: Generate historical metrics from audit log events
4. **Rollback plan**: PostgreSQL remains the source of truth during migration; Iceberg, Neo4j, and TimescaleDB are derived views that can be rebuilt

### Disaster Recovery

- **PostgreSQL**: Streaming replication + pgBackRest (RPO < 1 min)
- **Iceberg/S3**: Cross-region replication with Object Lock; data is immutable by design
- **Neo4j**: Causal cluster with read replicas; full rebuild from PostgreSQL if cluster lost
- **TimescaleDB**: PostgreSQL streaming replication; continuous aggregates rebuild from raw data
- **Kafka**: Multi-broker cluster with replication factor 3; MirrorMaker for cross-region
