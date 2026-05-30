# Data Model Suggestion 2: Event-Sourced / CQRS Model

## Overview

This model treats every state change in the data retention and archiving platform as an immutable domain event stored in an append-only event store. The system never updates or deletes rows in the event store; instead, it appends new events that describe what happened. Read models (projections) are built from the event stream to serve queries efficiently.

This approach is a natural fit for a data retention platform because the domain inherently demands immutable audit trails, defensible proof of every action taken on every record, and the ability to reconstruct the exact state of any entity at any point in time. The event store itself becomes the definitive audit log -- not a secondary concern bolted on after the fact, but the foundational data structure from which all other views are derived.

The architecture separates the write side (command handling and event persistence) from the read side (projections optimised for specific query patterns), following the CQRS (Command Query Responsibility Segregation) pattern.

---

## Architecture Overview

```
                    ┌─────────────────┐
                    │   API Gateway   │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────▼────────┐          ┌─────────▼─────────┐
     │  Command Side   │          │   Query Side       │
     │  (Write Model)  │          │   (Read Models)    │
     └────────┬────────┘          └─────────▲─────────┘
              │                             │
              │  append events              │  project events
              │                             │
     ┌────────▼─────────────────────────────┤
     │         EVENT STORE                  │
     │    (append-only, immutable)          │
     └──────────────────────────────────────┘
              │
              │  event bus (async)
              ├──────────────────┐
              │                  │
     ┌────────▼────────┐  ┌─────▼──────────┐
     │  Projection     │  │  External       │
     │  Processors     │  │  Integrations   │
     └────────┬────────┘  └────────────────┘
              │
     ┌────────▼────────┐
     │  Read Databases  │
     │  (PostgreSQL,    │
     │   OpenSearch,    │
     │   Redis)         │
     └─────────────────┘
```

---

## Event Store Schema

The event store is the single source of truth. It uses PostgreSQL for durability and transactional guarantees, but the table is strictly append-only.

```sql
-- ============================================================
-- EVENT STORE (append-only, immutable)
-- ============================================================

CREATE TABLE event_store (
    -- Global ordering
    global_sequence     BIGSERIAL NOT NULL,
    
    -- Aggregate identity
    aggregate_type      TEXT NOT NULL,              -- 'Record', 'Policy', 'LegalHold', 'Matter', etc.
    aggregate_id        UUID NOT NULL,
    aggregate_version   INTEGER NOT NULL,           -- version within this aggregate (optimistic concurrency)
    
    -- Tenant isolation
    organisation_id     UUID NOT NULL,
    
    -- Event data
    event_type          TEXT NOT NULL,              -- fully qualified event name
    event_data          JSONB NOT NULL,             -- the event payload
    event_metadata      JSONB NOT NULL DEFAULT '{}', -- correlation IDs, causation, actor info
    
    -- Timestamps
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Tamper-evidence
    previous_hash       BYTEA,                     -- hash of previous event in this aggregate
    event_hash          BYTEA NOT NULL,            -- HMAC-SHA256 of this event
    
    PRIMARY KEY (global_sequence),
    UNIQUE (aggregate_type, aggregate_id, aggregate_version)
) PARTITION BY RANGE (global_sequence);

-- Partitions by sequence range (e.g., 10M events per partition)
CREATE TABLE event_store_part_001 PARTITION OF event_store
    FOR VALUES FROM (1) TO (10000001);
CREATE TABLE event_store_part_002 PARTITION OF event_store
    FOR VALUES FROM (10000001) TO (20000001);

-- Indexes for common access patterns
CREATE INDEX idx_events_aggregate ON event_store(aggregate_type, aggregate_id, aggregate_version);
CREATE INDEX idx_events_org ON event_store(organisation_id, occurred_at);
CREATE INDEX idx_events_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_events_correlation ON event_store USING GIN ((event_metadata->'correlation_id'));

-- Strict append-only enforcement
CREATE OR REPLACE FUNCTION prevent_event_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Event store is append-only. Events cannot be modified or deleted.';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_update_events BEFORE UPDATE ON event_store
    FOR EACH ROW EXECUTE FUNCTION prevent_event_modification();
CREATE TRIGGER no_delete_events BEFORE DELETE ON event_store
    FOR EACH ROW EXECUTE FUNCTION prevent_event_modification();

-- ============================================================
-- SNAPSHOT STORE (for aggregate rehydration optimisation)
-- ============================================================

CREATE TABLE aggregate_snapshots (
    aggregate_type      TEXT NOT NULL,
    aggregate_id        UUID NOT NULL,
    aggregate_version   INTEGER NOT NULL,
    organisation_id     UUID NOT NULL,
    snapshot_data       JSONB NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, aggregate_version)
);

-- Keep only the latest N snapshots per aggregate
CREATE INDEX idx_snapshots_lookup ON aggregate_snapshots(aggregate_type, aggregate_id, aggregate_version DESC);

-- ============================================================
-- STREAM SUBSCRIPTIONS (for projection checkpointing)
-- ============================================================

CREATE TABLE stream_subscriptions (
    subscription_id     TEXT PRIMARY KEY,           -- e.g., 'projection:records_read_model'
    last_processed_seq  BIGINT NOT NULL DEFAULT 0,
    last_processed_at   TIMESTAMPTZ,
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'paused', 'rebuilding')),
    error_message       TEXT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Domain Event Types

### Record Lifecycle Events

```
Record Aggregate Events:
├── RecordIngested
│   { record_id, source_id, source_type, original_id, content_type, subject,
│     body_hash, file_size, original_format, sender, recipients,
│     source_created_at, storage_bucket, storage_key, storage_region }
│
├── RecordClassified
│   { record_id, data_class, confidence, classified_by, method: 'ai'|'manual' }
│
├── RecordClassificationOverridden
│   { record_id, previous_class, new_class, overridden_by, reason }
│
├── RecordRetentionAssigned
│   { record_id, policy_id, policy_name, computed_expiry, retention_trigger }
│
├── RecordRetentionExtended
│   { record_id, policy_id, previous_expiry, new_expiry, reason }
│
├── RecordPlacedOnHold
│   { record_id, hold_id, matter_id, placed_by }
│
├── RecordReleasedFromHold
│   { record_id, hold_id, released_by, reason }
│
├── RecordMarkedForDisposition
│   { record_id, batch_id, policy_id, expiry_date }
│
├── RecordDispositionApproved
│   { record_id, batch_id, approved_by }
│
├── RecordDispositionExcepted
│   { record_id, batch_id, reason, excepted_by }
│
├── RecordDeleted
│   { record_id, batch_id, certificate_id, deletion_method,
│     storage_keys_removed, body_hash_verified }
│
├── RecordStorageTierChanged
│   { record_id, previous_tier, new_tier, reason }
│
├── RecordExportedForProduction
│   { record_id, production_id, collection_id, export_format }
│
└── RecordMetadataEnriched
    { record_id, enrichment_type, new_metadata }
```

### Retention Policy Events

```
Policy Aggregate Events:
├── RetentionPolicyCreated
│   { policy_id, name, description, retention_period_days, disposition_action,
│     applies_to_data_classes, applies_to_source_types, applies_to_jurisdictions,
│     regulatory_framework, regulatory_citation, created_by }
│
├── RetentionPolicyUpdated
│   { policy_id, changes: { field: { old, new } }, updated_by, reason }
│
├── RetentionPolicyActivated
│   { policy_id, activated_by, effective_from }
│
├── RetentionPolicySuspended
│   { policy_id, suspended_by, reason }
│
├── RetentionPolicyRetired
│   { policy_id, retired_by, reason, replacement_policy_id }
│
└── PolicyConflictDetected
    { policy_a_id, policy_b_id, conflict_type, description, affected_record_count }
```

### Legal Hold Events

```
LegalHold Aggregate Events:
├── LegalHoldCreated
│   { hold_id, matter_id, name, description, scope_data_sources,
│     scope_date_from, scope_date_to, scope_keywords, created_by }
│
├── LegalHoldActivated
│   { hold_id, activated_by, custodian_count, record_count_estimated }
│
├── CustodianAddedToHold
│   { hold_id, custodian_id, added_by }
│
├── CustodianRemovedFromHold
│   { hold_id, custodian_id, removed_by, reason }
│
├── CustodianNotified
│   { hold_id, custodian_id, notification_type, channel, message_id }
│
├── CustodianAcknowledgedHold
│   { hold_id, custodian_id, acknowledged_at }
│
├── CustodianEscalated
│   { hold_id, custodian_id, escalation_reason, escalated_to }
│
├── LegalHoldReleased
│   { hold_id, released_by, reason, records_released_count }
│
└── LegalHoldExpired
    { hold_id, expiry_reason }
```

### Matter and eDiscovery Events

```
Matter Aggregate Events:
├── MatterOpened
│   { matter_id, matter_number, name, matter_type, lead_attorney_id, opened_by }
│
├── MatterUpdated
│   { matter_id, changes, updated_by }
│
├── MatterClosed
│   { matter_id, closed_by, reason, holds_released }
│
└── MatterArchived
    { matter_id, archived_by }

Collection Aggregate Events:
├── CollectionCreated
│   { collection_id, matter_id, name, source_query, created_by }
│
├── CollectionItemAdded
│   { collection_id, record_id, added_at }
│
├── CollectionItemReviewed
│   { collection_id, record_id, review_status, reviewed_by, notes }
│
└── CollectionCompleted
    { collection_id, total_items, completed_by }

Production Aggregate Events:
├── ProductionCreated
│   { production_id, collection_id, matter_id, format, created_by }
│
├── ProductionCompleted
│   { production_id, total_items, total_bytes, export_path }
│
└── ProductionDelivered
    { production_id, delivered_to, delivered_at, acknowledgement }
```

### Disposition Events

```
DispositionBatch Aggregate Events:
├── DispositionBatchCreated
│   { batch_id, policy_id, record_count, total_bytes, created_by }
│
├── DispositionBatchSubmittedForApproval
│   { batch_id, approval_chain, submitted_by }
│
├── DispositionApprovalReceived
│   { batch_id, approver_id, decision, comments, approval_order }
│
├── DispositionBatchApproved
│   { batch_id, final_approver_id }
│
├── DispositionBatchRejected
│   { batch_id, rejected_by, reason }
│
├── DispositionExecutionStarted
│   { batch_id, started_at, estimated_duration }
│
├── DispositionExecutionCompleted
│   { batch_id, records_deleted, records_excepted, duration }
│
└── DeletionCertificateIssued
    { certificate_id, batch_id, certificate_number, records_count,
      total_bytes, content_hash, chain_hash, disposition_narrative }
```

### Data Source and Connector Events

```
DataSource Aggregate Events:
├── DataSourceRegistered
│   { source_id, source_type, display_name, registered_by }
│
├── DataSourceConnected
│   { source_id, connected_at }
│
├── DataSourceSyncStarted
│   { source_id, sync_id, estimated_items }
│
├── DataSourceSyncCompleted
│   { source_id, sync_id, items_synced, bytes_synced, duration }
│
├── DataSourceSyncFailed
│   { source_id, sync_id, error_message, items_partial }
│
└── DataSourceDisabled
    { source_id, disabled_by, reason }
```

---

## Read Model Projections

Each projection is a materialised view of the event stream, optimised for a specific query pattern. Projections are rebuilt from events and can be safely dropped and reconstructed at any time.

### Projection 1: Records Read Model (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: Current state of all archived records
-- Materialised by the RecordProjection processor
-- ============================================================

CREATE TABLE rm_records (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    data_source_id      UUID NOT NULL,
    source_type         TEXT NOT NULL,
    original_id         TEXT NOT NULL,
    content_type        TEXT NOT NULL,
    subject             TEXT,
    body_text           TEXT,
    body_hash           BYTEA NOT NULL,
    original_format     TEXT NOT NULL,
    file_size_bytes     BIGINT NOT NULL,
    sender              TEXT,
    recipients          TEXT[],
    source_created_at   TIMESTAMPTZ,
    ingested_at         TIMESTAMPTZ NOT NULL,
    
    -- Storage
    storage_tier        TEXT NOT NULL DEFAULT 'hot',
    storage_bucket      TEXT NOT NULL,
    storage_key         TEXT NOT NULL,
    
    -- Classification (latest)
    data_class          TEXT,
    classification_confidence REAL,
    classification_method TEXT,
    classification_verified BOOLEAN NOT NULL DEFAULT false,
    
    -- Retention (governing policy -- the one with longest expiry)
    governing_policy_id UUID,
    governing_policy_name TEXT,
    retention_expiry    TIMESTAMPTZ,
    
    -- Hold status
    active_hold_count   INTEGER NOT NULL DEFAULT 0,
    hold_ids            UUID[] NOT NULL DEFAULT '{}',
    
    -- Lifecycle
    lifecycle_status    TEXT NOT NULL DEFAULT 'active',
    
    -- Projection metadata
    last_event_seq      BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_records_org ON rm_records(organisation_id);
CREATE INDEX idx_rm_records_lifecycle ON rm_records(organisation_id, lifecycle_status);
CREATE INDEX idx_rm_records_expiry ON rm_records(organisation_id, retention_expiry)
    WHERE lifecycle_status = 'active' AND active_hold_count = 0;
CREATE INDEX idx_rm_records_holds ON rm_records(organisation_id)
    WHERE active_hold_count > 0;
CREATE INDEX idx_rm_records_fts ON rm_records
    USING GIN (to_tsvector('english', coalesce(subject, '') || ' ' || coalesce(body_text, '')));
```

### Projection 2: Policy Dashboard (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: Retention policies with computed statistics
-- ============================================================

CREATE TABLE rm_policies (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    name                TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL,
    retention_period_days INTEGER NOT NULL,
    disposition_action  TEXT NOT NULL,
    regulatory_framework TEXT,
    regulatory_citation TEXT,
    
    -- Computed stats (updated by projection)
    assigned_record_count BIGINT NOT NULL DEFAULT 0,
    assigned_total_bytes  BIGINT NOT NULL DEFAULT 0,
    records_expiring_30d  INTEGER NOT NULL DEFAULT 0,
    records_expiring_90d  INTEGER NOT NULL DEFAULT 0,
    active_conflict_count INTEGER NOT NULL DEFAULT 0,
    
    -- Lifecycle
    created_at          TIMESTAMPTZ NOT NULL,
    activated_at        TIMESTAMPTZ,
    created_by_name     TEXT,
    
    last_event_seq      BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_policy_conflicts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    policy_a_id         UUID NOT NULL,
    policy_a_name       TEXT NOT NULL,
    policy_b_id         UUID NOT NULL,
    policy_b_name       TEXT NOT NULL,
    conflict_type       TEXT NOT NULL,
    description         TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'open',
    detected_at         TIMESTAMPTZ NOT NULL,
    resolved_at         TIMESTAMPTZ,
    
    last_event_seq      BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Projection 3: Legal Hold Status (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: Legal holds with custodian and record counts
-- ============================================================

CREATE TABLE rm_legal_holds (
    id                  UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    matter_id           UUID NOT NULL,
    matter_name         TEXT NOT NULL,
    matter_number       TEXT NOT NULL,
    hold_name           TEXT NOT NULL,
    status              TEXT NOT NULL,
    
    -- Scope
    scope_data_sources  TEXT[] NOT NULL DEFAULT '{}',
    scope_date_from     TIMESTAMPTZ,
    scope_date_to       TIMESTAMPTZ,
    scope_keywords      TEXT[],
    
    -- Custodian stats
    total_custodians    INTEGER NOT NULL DEFAULT 0,
    acknowledged_count  INTEGER NOT NULL DEFAULT 0,
    pending_count       INTEGER NOT NULL DEFAULT 0,
    escalated_count     INTEGER NOT NULL DEFAULT 0,
    
    -- Record stats
    records_held        BIGINT NOT NULL DEFAULT 0,
    bytes_held          BIGINT NOT NULL DEFAULT 0,
    
    -- Lifecycle
    issued_at           TIMESTAMPTZ,
    released_at         TIMESTAMPTZ,
    
    last_event_seq      BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE rm_custodian_holds (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    hold_id             UUID NOT NULL,
    hold_name           TEXT NOT NULL,
    custodian_id        UUID NOT NULL,
    custodian_name      TEXT NOT NULL,
    custodian_email     TEXT,
    status              TEXT NOT NULL,
    notified_at         TIMESTAMPTZ,
    acknowledged_at     TIMESTAMPTZ,
    reminder_count      INTEGER NOT NULL DEFAULT 0,
    
    last_event_seq      BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Projection 4: Disposition Pipeline (PostgreSQL)

```sql
-- ============================================================
-- READ MODEL: Disposition batches and approval status
-- ============================================================

CREATE TABLE rm_disposition_pipeline (
    batch_id            UUID PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    batch_number        TEXT NOT NULL,
    policy_name         TEXT NOT NULL,
    status              TEXT NOT NULL,
    total_records       INTEGER NOT NULL,
    total_bytes         BIGINT NOT NULL,
    
    -- Approval progress
    approvals_required  INTEGER NOT NULL DEFAULT 0,
    approvals_received  INTEGER NOT NULL DEFAULT 0,
    current_approver    TEXT,
    
    -- Execution progress
    records_deleted     INTEGER NOT NULL DEFAULT 0,
    records_excepted    INTEGER NOT NULL DEFAULT 0,
    
    -- Certificate
    certificate_number  TEXT,
    certificate_issued  BOOLEAN NOT NULL DEFAULT false,
    
    created_at          TIMESTAMPTZ NOT NULL,
    completed_at        TIMESTAMPTZ,
    
    last_event_seq      BIGINT NOT NULL,
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_disposition_org ON rm_disposition_pipeline(organisation_id, status);
```

### Projection 5: Full-Text Search (OpenSearch)

```
OpenSearch Index: archived_records
{
    "mappings": {
        "properties": {
            "record_id":          { "type": "keyword" },
            "organisation_id":    { "type": "keyword" },
            "content_type":       { "type": "keyword" },
            "subject":            { "type": "text", "analyzer": "standard" },
            "body_text":          { "type": "text", "analyzer": "standard" },
            "sender":             { "type": "keyword" },
            "recipients":         { "type": "keyword" },
            "participants":       { "type": "keyword" },
            "data_class":         { "type": "keyword" },
            "source_created_at":  { "type": "date" },
            "ingested_at":        { "type": "date" },
            "lifecycle_status":   { "type": "keyword" },
            "hold_ids":           { "type": "keyword" },
            "data_source_type":   { "type": "keyword" },
            "storage_tier":       { "type": "keyword" },
            "tags":               { "type": "keyword" },
            "file_size_bytes":    { "type": "long" },
            "original_format":    { "type": "keyword" }
        }
    },
    "settings": {
        "number_of_shards": 5,
        "number_of_replicas": 1,
        "index.lifecycle.name": "retention_ilm_policy"
    }
}
```

### Projection 6: Compliance Dashboard (Redis)

```
Redis keys (precomputed counters refreshed by projection):

org:{org_id}:records:total              -> integer
org:{org_id}:records:by_status:{status} -> integer
org:{org_id}:records:by_class:{class}   -> integer
org:{org_id}:records:under_hold         -> integer
org:{org_id}:records:expiring:30d       -> integer
org:{org_id}:records:expiring:90d       -> integer
org:{org_id}:policies:active            -> integer
org:{org_id}:policies:with_conflicts    -> integer
org:{org_id}:holds:active               -> integer
org:{org_id}:holds:pending_ack          -> integer
org:{org_id}:disposition:pending        -> integer
org:{org_id}:storage:total_bytes        -> integer
org:{org_id}:coverage_gaps              -> integer
```

### Projection 7: Audit Timeline (PostgreSQL, for regulatory access)

```sql
-- ============================================================
-- READ MODEL: Denormalized audit timeline for compliance officers
-- Built from ALL events, provides human-readable timeline
-- ============================================================

CREATE TABLE rm_audit_timeline (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    occurred_at         TIMESTAMPTZ NOT NULL,
    event_type          TEXT NOT NULL,
    category            TEXT NOT NULL,
    severity            TEXT NOT NULL DEFAULT 'info',
    
    -- Human-readable
    summary             TEXT NOT NULL,
    
    -- Actor
    actor_name          TEXT,
    actor_email         TEXT,
    actor_role          TEXT,
    
    -- Target
    target_type         TEXT,
    target_id           UUID,
    target_description  TEXT,
    
    -- Related entities
    matter_id           UUID,
    hold_id             UUID,
    policy_id           UUID,
    batch_id            UUID,
    
    -- Source event reference
    source_event_seq    BIGINT NOT NULL,
    
    projected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

CREATE INDEX idx_rm_audit_org ON rm_audit_timeline(organisation_id, occurred_at DESC);
CREATE INDEX idx_rm_audit_target ON rm_audit_timeline(target_type, target_id);
CREATE INDEX idx_rm_audit_matter ON rm_audit_timeline(matter_id) WHERE matter_id IS NOT NULL;
```

---

## Command Handlers and Business Rules

### Example: Place Record on Hold

```python
# Pseudocode for command handler

class PlaceRecordOnHoldCommand:
    record_id: UUID
    hold_id: UUID
    matter_id: UUID
    placed_by: UUID

class RecordCommandHandler:
    def handle_place_on_hold(self, cmd: PlaceRecordOnHoldCommand):
        # 1. Load aggregate from event store
        record = self.event_store.load_aggregate('Record', cmd.record_id)
        
        # 2. Business rule validation
        if record.lifecycle_status == 'deleted':
            raise BusinessRuleViolation("Cannot hold a deleted record")
        if cmd.hold_id in record.active_holds:
            raise BusinessRuleViolation("Record already under this hold")
        
        # 3. Emit event (optimistic concurrency via aggregate_version)
        event = RecordPlacedOnHold(
            record_id=cmd.record_id,
            hold_id=cmd.hold_id,
            matter_id=cmd.matter_id,
            placed_by=cmd.placed_by
        )
        self.event_store.append(
            aggregate_type='Record',
            aggregate_id=cmd.record_id,
            expected_version=record.version,
            event=event
        )
        
        # 4. Event published to bus for projections and side effects
```

### Example: Approve Disposition Batch

```python
class ApproveDispositionCommand:
    batch_id: UUID
    approver_id: UUID
    decision: str  # 'approved' | 'rejected'
    comments: str

class DispositionCommandHandler:
    def handle_approval(self, cmd: ApproveDispositionCommand):
        batch = self.event_store.load_aggregate('DispositionBatch', cmd.batch_id)
        
        # Validate approver is in the chain and it is their turn
        if cmd.approver_id not in batch.approval_chain:
            raise BusinessRuleViolation("Not an authorized approver")
        if batch.next_approver_id != cmd.approver_id:
            raise BusinessRuleViolation("Not your turn in the approval chain")
        
        # Check no records are under hold
        for record_id in batch.record_ids:
            record = self.event_store.load_aggregate('Record', record_id)
            if record.active_hold_count > 0:
                raise BusinessRuleViolation(
                    f"Record {record_id} is under legal hold; cannot approve disposition"
                )
        
        event = DispositionApprovalReceived(
            batch_id=cmd.batch_id,
            approver_id=cmd.approver_id,
            decision=cmd.decision,
            comments=cmd.comments,
            approval_order=batch.current_approval_step
        )
        self.event_store.append(
            aggregate_type='DispositionBatch',
            aggregate_id=cmd.batch_id,
            expected_version=batch.version,
            event=event
        )
        
        # If all approvals received, emit batch-level approval
        if batch.current_approval_step + 1 >= len(batch.approval_chain):
            self.event_store.append(
                aggregate_type='DispositionBatch',
                aggregate_id=cmd.batch_id,
                expected_version=batch.version + 1,
                event=DispositionBatchApproved(
                    batch_id=cmd.batch_id,
                    final_approver_id=cmd.approver_id
                )
            )
```

---

## Event Store Maintenance

### Snapshotting Strategy

```sql
-- Snapshot every 100 events per aggregate to avoid long replay chains
-- Triggered by the command handler after appending an event

INSERT INTO aggregate_snapshots (aggregate_type, aggregate_id, aggregate_version, organisation_id, snapshot_data)
SELECT 'Record', $1, $2, $3, $4
WHERE $2 % 100 = 0  -- snapshot every 100 versions
ON CONFLICT (aggregate_type, aggregate_id, aggregate_version) DO NOTHING;

-- Aggregate rehydration: load snapshot + replay subsequent events
SELECT snapshot_data, aggregate_version
FROM aggregate_snapshots
WHERE aggregate_type = $1 AND aggregate_id = $2
ORDER BY aggregate_version DESC
LIMIT 1;

-- Then replay events after the snapshot version
SELECT event_type, event_data, aggregate_version
FROM event_store
WHERE aggregate_type = $1 AND aggregate_id = $2 AND aggregate_version > $3
ORDER BY aggregate_version;
```

### Projection Rebuilding

```sql
-- To rebuild a projection from scratch:
-- 1. Reset the subscription checkpoint
UPDATE stream_subscriptions
SET last_processed_seq = 0, status = 'rebuilding'
WHERE subscription_id = 'projection:records_read_model';

-- 2. Truncate the read model table
TRUNCATE TABLE rm_records;

-- 3. The projection processor replays all events from sequence 0
-- 4. Once caught up, status returns to 'active'
```

### Archival of Old Events

Events are never deleted, but old partitions can be moved to cold storage:

```sql
-- Detach old partition and export to Parquet on S3
ALTER TABLE event_store DETACH PARTITION event_store_part_001;

-- Export via pg_dump or COPY to S3-backed foreign table
-- The partition remains queryable via postgres_fdw pointing to S3/Parquet

-- Create foreign table for archived events
CREATE FOREIGN TABLE event_store_archive_001 (
    global_sequence BIGINT,
    aggregate_type TEXT,
    aggregate_id UUID,
    aggregate_version INTEGER,
    organisation_id UUID,
    event_type TEXT,
    event_data JSONB,
    event_metadata JSONB,
    occurred_at TIMESTAMPTZ,
    recorded_at TIMESTAMPTZ,
    previous_hash BYTEA,
    event_hash BYTEA
) SERVER s3_server
OPTIONS (filename 's3://archive-bucket/events/part_001.parquet');
```

---

## Pros and Cons

### Pros

1. **Immutable audit trail by design.** The event store IS the audit trail. There is no separate audit log that could drift out of sync with the actual data. Every state change -- from record ingestion to deletion certificate issuance -- is captured as a permanent, hash-chained event. This is the strongest possible compliance posture for SEC 17a-4, FINRA, and GDPR accountability.

2. **Perfect temporal queries.** "What was the state of this record on March 15th?" is answered by replaying events up to that date. "Which records were under hold when the matter was closed?" is a straightforward event stream query. This is invaluable for regulatory investigations that require point-in-time reconstruction.

3. **Decoupled read models.** The compliance dashboard, search index, disposition pipeline, and audit timeline are all independent projections. Each can be optimised for its specific access pattern without compromising the write model. If the search index needs restructuring, it is rebuilt from events with zero risk to the source of truth.

4. **Natural fit for disposition workflows.** The multi-stage disposition approval process maps perfectly to an event sequence: BatchCreated -> SubmittedForApproval -> ApprovalReceived (x N) -> BatchApproved -> ExecutionStarted -> ExecutionCompleted -> CertificateIssued. Each step is an immutable fact that cannot be retroactively altered.

5. **Resilient to schema evolution.** New event types can be added without migrating existing data. Old events remain valid forever. A new connector type simply introduces new event types (e.g., TeamsMessageIngested) that existing projections can handle or ignore.

6. **Debugging and forensics.** When a compliance officer asks "why was this record deleted?", the entire causal chain is in the event stream: PolicyCreated -> PolicyActivated -> RecordRetentionAssigned -> RecordMarkedForDisposition -> DispositionApproved -> RecordDeleted -> CertificateIssued.

7. **Regulatory-grade evidence.** Events are digitally hash-chained (each event includes the hash of the previous event in its aggregate stream). Any tampering is detectable by verifying the hash chain, providing cryptographic proof of log integrity.

### Cons

1. **Complexity.** Event sourcing with CQRS is significantly more complex than traditional CRUD. The team must understand aggregates, event handlers, projections, eventual consistency, and idempotent event processing. This is a non-trivial architectural commitment.

2. **Eventual consistency of read models.** There is always a lag between an event being written and the projections being updated. This means a user might place a hold and not immediately see it reflected in the dashboard. The system must handle this with appropriate UX (optimistic UI updates, loading states) and careful SLA management for projection lag.

3. **Event schema evolution.** While adding new event types is trivial, changing the shape of existing event types is complex. An event upcasting/versioning strategy is required from day one. If RecordIngested v1 has different fields than v2, all projections must handle both versions.

4. **Aggregate size risk.** A record that has been reclassified, held, released, and held again dozens of times over many years will accumulate hundreds of events. Without snapshotting, rehydrating this aggregate becomes slow. The snapshot strategy must be implemented and maintained.

5. **Storage growth.** Events are never deleted, so the event store grows monotonically. A platform ingesting millions of records per day, each generating 3-5 lifecycle events, will produce tens of millions of events per day. Partition management and cold-tier archival are essential.

6. **Projection rebuilds are expensive.** If a projection has a bug and needs rebuilding, replaying billions of events can take hours or days. Incremental rebuild strategies and parallel processing are needed but add further complexity.

7. **Testing complexity.** Testing event-sourced systems requires verifying both the events produced by commands and the read model state produced by those events. This doubles the test surface compared to CRUD.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Event store** | PostgreSQL 16+ (append-only table) | ACID guarantees, partitioning, mature tooling |
| **Alternative event store** | EventStoreDB | Purpose-built for event sourcing; built-in subscriptions and projections |
| **Read model database** | PostgreSQL 16+ | Shared infrastructure with event store; separate instance for isolation |
| **Search projection** | OpenSearch 2.x | Full-text search, faceted filtering, ILM for index rotation |
| **Dashboard counters** | Redis 7+ | Sub-millisecond reads for compliance dashboard metrics |
| **Event bus** | Apache Kafka or NATS JetStream | Durable, ordered event distribution to projection processors |
| **Projection framework** | Custom processors or Marten (.NET) / Axon (Java) | Framework support for event replay, checkpointing, error handling |
| **Object storage** | MinIO with Object Lock or S3 | Immutable content storage (WORM compliance) |
| **Serialisation** | JSON with schema registry | Event payload validation; Avro or Protobuf for high-throughput |
| **Monitoring** | OpenTelemetry + Grafana | Event throughput, projection lag, aggregate replay times |

---

## Migration and Scaling Considerations

### Initial Deployment

- Single PostgreSQL instance for event store + read models
- Kafka (or NATS) single-broker for event distribution
- Single projection processor per read model
- Redis standalone for dashboard counters
- Expected throughput: ~1,000 events/second

### Growth Phase

- Separate PostgreSQL instances for event store and read models
- Kafka cluster (3+ brokers) with topic-per-aggregate-type partitioning
- Multiple projection processor instances (consumer groups)
- OpenSearch cluster for search projection
- Snapshot frequency tuning based on aggregate event counts
- Expected throughput: ~10,000 events/second

### Enterprise Scale

- Partitioned event store across multiple PostgreSQL instances (by organisation_id hash)
- Kafka cluster with cross-region mirroring for data residency compliance
- Dedicated projection processor clusters per region
- Event archival pipeline: old partitions -> Parquet on S3 -> queryable via federation
- Event store compaction: for aggregates exceeding 10,000 events, create "compacted snapshots" that combine snapshot + tombstone of pre-snapshot events (events remain in archive but are not replayed)
- Expected throughput: ~100,000+ events/second

### Migration from CRUD to Event Sourcing

If starting from an existing CRUD system:

1. **Dual-write phase**: New writes produce both CRUD updates and events. Read from CRUD.
2. **Shadow projection phase**: Build projections from events; compare with CRUD state for validation.
3. **Cutover phase**: Switch reads to projections; CRUD tables become legacy.
4. **Cleanup phase**: Remove CRUD write path; event store is the sole source of truth.

### Disaster Recovery

- **Event store RPO**: 0 (synchronous replication to standby)
- **Event store RTO**: < 5 minutes (streaming replication failover)
- **Projection recovery**: Projections are rebuilt from the event store; no separate backup needed
- **Cross-region**: Kafka MirrorMaker or NATS super-cluster for geographic distribution
- **Hash chain verification**: Periodic automated verification of event hash chains; alerts on any detected tampering
