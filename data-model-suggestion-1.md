# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

## Overview

This model uses a fully normalized relational schema in PostgreSQL, leveraging its mature ecosystem for ACID compliance, row-level security for multi-tenancy, declarative partitioning for archival data at scale, and temporal tables for built-in change tracking. Every entity is represented as a first-class table with explicit foreign key relationships, making the schema self-documenting and enforceable at the database level.

This approach aligns with the OAIS reference model's separation of concerns (Ingest, Archival Storage, Data Management, Access) and the EDRM framework's nine-stage lifecycle by giving each concept -- policies, holds, custodians, disposition workflows, audit events -- its own well-defined table with clear relational integrity.

---

## Core Schema

### 1. Multi-Tenancy and Organisation

```sql
-- ============================================================
-- ORGANISATION & MULTI-TENANCY
-- ============================================================

CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    slug                TEXT NOT NULL UNIQUE,
    billing_plan        TEXT NOT NULL DEFAULT 'standard',
    data_residency      TEXT NOT NULL DEFAULT 'us-east-1',  -- AWS region or Azure region
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMPTZ  -- soft delete
);

CREATE TABLE business_units (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                TEXT NOT NULL,
    parent_unit_id      UUID REFERENCES business_units(id),  -- hierarchy
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, name)
);

CREATE TABLE users (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    email               TEXT NOT NULL,
    full_name           TEXT NOT NULL,
    role                TEXT NOT NULL CHECK (role IN (
                            'super_admin', 'compliance_admin', 'legal_admin',
                            'it_operator', 'records_manager', 'auditor', 'end_user'
                        )),
    department          TEXT,
    business_unit_id    UUID REFERENCES business_units(id),
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

-- Row-Level Security for multi-tenant isolation
ALTER TABLE organisations ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY org_isolation ON organisations
    USING (id = current_setting('app.current_org_id')::UUID);

CREATE POLICY user_isolation ON users
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

### 2. Data Sources and Connectors

```sql
-- ============================================================
-- DATA SOURCES & CONNECTORS
-- ============================================================

CREATE TABLE data_source_types (
    id                  TEXT PRIMARY KEY,  -- 'microsoft_365', 'google_workspace', 'slack', etc.
    display_name        TEXT NOT NULL,
    category            TEXT NOT NULL CHECK (category IN (
                            'email', 'collaboration', 'file_storage',
                            'database', 'social_media', 'erp', 'custom'
                        )),
    connector_version   TEXT NOT NULL,
    supported_actions   TEXT[] NOT NULL DEFAULT '{}',  -- {'ingest', 'hold', 'delete', 'export'}
    is_active           BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE data_sources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    source_type_id      TEXT NOT NULL REFERENCES data_source_types(id),
    display_name        TEXT NOT NULL,
    connection_config   TEXT NOT NULL,  -- encrypted JSON, stored via pgcrypto
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'connected', 'syncing', 'error', 'disabled'
                        )),
    last_sync_at        TIMESTAMPTZ,
    next_sync_at        TIMESTAMPTZ,
    sync_frequency_min  INTEGER NOT NULL DEFAULT 60,
    total_items_synced  BIGINT NOT NULL DEFAULT 0,
    total_bytes_synced  BIGINT NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_org ON data_sources(organisation_id);
ALTER TABLE data_sources ENABLE ROW LEVEL SECURITY;
CREATE POLICY ds_isolation ON data_sources
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

### 3. Archived Records (the core content store)

```sql
-- ============================================================
-- ARCHIVED RECORDS
-- Partitioned by organisation_id for tenant isolation at storage level,
-- and sub-partitioned by ingestion month for lifecycle management.
-- ============================================================

CREATE TABLE archived_records (
    id                  UUID NOT NULL DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    data_source_id      UUID NOT NULL,
    
    -- OAIS Content Information
    original_id         TEXT NOT NULL,           -- ID in source system
    content_type        TEXT NOT NULL,           -- 'email', 'document', 'chat_message', 'file', etc.
    subject             TEXT,
    body_text           TEXT,                    -- extracted plaintext for search
    body_hash           BYTEA NOT NULL,          -- SHA-256 of original content
    original_format     TEXT NOT NULL,           -- 'eml', 'docx', 'pdf', 'json', etc.
    file_size_bytes     BIGINT NOT NULL DEFAULT 0,
    
    -- OAIS Preservation Description Information
    sender              TEXT,
    recipients          TEXT[],
    participants        TEXT[],
    source_created_at   TIMESTAMPTZ,             -- when created in source system
    source_modified_at  TIMESTAMPTZ,
    ingested_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Storage references (content stored in immutable object storage)
    storage_tier        TEXT NOT NULL DEFAULT 'hot' CHECK (storage_tier IN ('hot', 'warm', 'cold', 'glacier')),
    storage_bucket      TEXT NOT NULL,
    storage_key         TEXT NOT NULL,
    storage_region      TEXT NOT NULL,
    
    -- Retention & lifecycle state
    retention_class_id  UUID,                    -- FK set after classification
    retention_expiry    TIMESTAMPTZ,             -- computed from policy
    lifecycle_status    TEXT NOT NULL DEFAULT 'active' CHECK (lifecycle_status IN (
                            'active', 'archived', 'hold', 'pending_disposition',
                            'approved_for_deletion', 'deleted', 'exported'
                        )),
    hold_count          INTEGER NOT NULL DEFAULT 0,  -- denormalized: number of active holds
    
    -- Classification
    data_class          TEXT,                    -- 'pii', 'phi', 'financial', 'confidential', 'general'
    ai_classification   TEXT,                    -- AI-suggested class
    ai_confidence       REAL,                   -- 0.0 - 1.0
    classification_verified BOOLEAN NOT NULL DEFAULT false,
    
    -- Partitioning key
    ingested_month      DATE NOT NULL DEFAULT date_trunc('month', now()),
    
    PRIMARY KEY (id, organisation_id, ingested_month),
    FOREIGN KEY (organisation_id) REFERENCES organisations(id),
    FOREIGN KEY (data_source_id) REFERENCES data_sources(id)
) PARTITION BY LIST (organisation_id);

-- Each tenant gets a sub-partitioned table (created dynamically)
-- Example for a specific tenant:
-- CREATE TABLE archived_records_tenant_abc PARTITION OF archived_records
--     FOR VALUES IN ('abc-uuid-here')
--     PARTITION BY RANGE (ingested_month);

-- Indexes for common query patterns
CREATE INDEX idx_records_content_type ON archived_records(organisation_id, content_type);
CREATE INDEX idx_records_lifecycle ON archived_records(organisation_id, lifecycle_status);
CREATE INDEX idx_records_retention_expiry ON archived_records(organisation_id, retention_expiry)
    WHERE lifecycle_status = 'active';
CREATE INDEX idx_records_hold ON archived_records(organisation_id)
    WHERE hold_count > 0;
CREATE INDEX idx_records_source ON archived_records(data_source_id);

-- Full-text search index
CREATE INDEX idx_records_fts ON archived_records
    USING GIN (to_tsvector('english', coalesce(subject, '') || ' ' || coalesce(body_text, '')));
```

### 4. Retention Policies and Data Classification

```sql
-- ============================================================
-- REGULATORY FRAMEWORKS & JURISDICTIONS
-- ============================================================

CREATE TABLE jurisdictions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                TEXT NOT NULL UNIQUE,     -- 'US', 'EU', 'UK', 'AU', 'SG', etc.
    name                TEXT NOT NULL,
    region              TEXT NOT NULL             -- 'North America', 'Europe', 'Asia Pacific'
);

CREATE TABLE regulatory_frameworks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                TEXT NOT NULL UNIQUE,     -- 'GDPR', 'HIPAA', 'SOX', 'PCI_DSS', 'FINRA', 'SEC_17A4'
    name                TEXT NOT NULL,
    description         TEXT,
    jurisdiction_id     UUID NOT NULL REFERENCES jurisdictions(id),
    version             TEXT,
    effective_date      DATE,
    url                 TEXT                      -- link to regulation text
);

-- ============================================================
-- DATA CLASSIFICATION & RETENTION RULES
-- ============================================================

CREATE TABLE data_classes (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    code                TEXT NOT NULL,            -- 'pii', 'phi', 'financial_record', etc.
    name                TEXT NOT NULL,
    description         TEXT,
    sensitivity_level   INTEGER NOT NULL DEFAULT 1 CHECK (sensitivity_level BETWEEN 1 AND 5),
    is_system_defined   BOOLEAN NOT NULL DEFAULT false,  -- true for built-in classes
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE retention_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
                            'draft', 'pending_review', 'active', 'suspended', 'retired'
                        )),
    
    -- Scope: what does this policy apply to?
    applies_to_data_classes UUID[] NOT NULL DEFAULT '{}',
    applies_to_source_types TEXT[] NOT NULL DEFAULT '{}',
    applies_to_business_units UUID[] NOT NULL DEFAULT '{}',
    applies_to_jurisdictions UUID[] NOT NULL DEFAULT '{}',
    
    -- Retention schedule
    retention_period_days   INTEGER NOT NULL,          -- e.g., 2555 for 7 years
    retention_trigger       TEXT NOT NULL DEFAULT 'creation_date' CHECK (retention_trigger IN (
                                'creation_date', 'last_modified_date', 'ingestion_date',
                                'employee_termination', 'contract_end', 'custom_event'
                            )),
    
    -- Disposition action
    disposition_action  TEXT NOT NULL DEFAULT 'delete' CHECK (disposition_action IN (
                            'delete', 'archive_cold', 'migrate', 'review'
                        )),
    requires_approval   BOOLEAN NOT NULL DEFAULT true,
    approval_chain      UUID[] NOT NULL DEFAULT '{}',  -- ordered list of user IDs
    
    -- Regulatory basis
    regulatory_framework_id UUID REFERENCES regulatory_frameworks(id),
    regulatory_citation TEXT,                          -- e.g., "HIPAA 45 CFR 164.530(j)"
    
    -- Priority for conflict resolution
    priority            INTEGER NOT NULL DEFAULT 100,  -- lower = higher priority
    
    -- Template info
    is_template         BOOLEAN NOT NULL DEFAULT false,
    template_source     TEXT,                          -- 'system_gdpr', 'system_hipaa', etc.
    
    created_by          UUID REFERENCES users(id),
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    effective_from      TIMESTAMPTZ,
    effective_until     TIMESTAMPTZ
);

CREATE INDEX idx_policies_org ON retention_policies(organisation_id);
CREATE INDEX idx_policies_status ON retention_policies(organisation_id, status);

-- Many-to-many: which policies apply to which records?
CREATE TABLE record_policy_assignments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    policy_id           UUID NOT NULL REFERENCES retention_policies(id),
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    computed_expiry     TIMESTAMPTZ NOT NULL,          -- when this record expires under this policy
    is_longest          BOOLEAN NOT NULL DEFAULT false, -- true if this is the governing policy
    UNIQUE (record_id, policy_id)
);

-- Policy conflict detection
CREATE TABLE policy_conflicts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    policy_a_id         UUID NOT NULL REFERENCES retention_policies(id),
    policy_b_id         UUID NOT NULL REFERENCES retention_policies(id),
    conflict_type       TEXT NOT NULL CHECK (conflict_type IN (
                            'retention_overlap', 'jurisdiction_conflict',
                            'erasure_vs_retention', 'action_conflict'
                        )),
    description         TEXT NOT NULL,
    resolution          TEXT,                          -- how it was resolved
    resolved_by         UUID REFERENCES users(id),
    resolved_at         TIMESTAMPTZ,
    status              TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'resolved', 'acknowledged')),
    detected_at         TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 5. Legal Hold Management

```sql
-- ============================================================
-- LEGAL HOLDS & CUSTODIANS
-- ============================================================

CREATE TABLE legal_matters (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_number       TEXT NOT NULL,
    name                TEXT NOT NULL,
    description         TEXT,
    matter_type         TEXT NOT NULL CHECK (matter_type IN (
                            'litigation', 'investigation', 'regulatory_inquiry',
                            'internal_audit', 'subpoena', 'foia_request'
                        )),
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'active', 'on_hold', 'closed', 'archived'
                        )),
    outside_counsel     TEXT,
    lead_attorney_id    UUID REFERENCES users(id),
    opened_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at           TIMESTAMPTZ,
    created_by          UUID NOT NULL REFERENCES users(id),
    UNIQUE (organisation_id, matter_number)
);

CREATE TABLE custodians (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID REFERENCES users(id),      -- NULL if external custodian
    external_name       TEXT,                            -- for non-employee custodians
    external_email      TEXT,
    department          TEXT,
    title               TEXT,
    employment_status   TEXT DEFAULT 'active' CHECK (employment_status IN (
                            'active', 'terminated', 'contractor', 'external'
                        )),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE legal_holds (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_id           UUID NOT NULL REFERENCES legal_matters(id),
    hold_name           TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
                            'draft', 'active', 'released', 'expired'
                        )),
    
    -- Scope definition
    scope_data_sources  UUID[] NOT NULL DEFAULT '{}',
    scope_date_from     TIMESTAMPTZ,
    scope_date_to       TIMESTAMPTZ,
    scope_keywords      TEXT[],
    scope_custodian_ids UUID[] NOT NULL DEFAULT '{}',
    
    -- Lifecycle
    issued_at           TIMESTAMPTZ,
    issued_by           UUID REFERENCES users(id),
    released_at         TIMESTAMPTZ,
    released_by         UUID REFERENCES users(id),
    release_reason      TEXT,
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE hold_custodian_assignments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hold_id             UUID NOT NULL REFERENCES legal_holds(id),
    custodian_id        UUID NOT NULL REFERENCES custodians(id),
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'notified', 'acknowledged', 'released', 'escalated'
                        )),
    notified_at         TIMESTAMPTZ,
    acknowledged_at     TIMESTAMPTZ,
    reminder_count      INTEGER NOT NULL DEFAULT 0,
    last_reminder_at    TIMESTAMPTZ,
    released_at         TIMESTAMPTZ,
    UNIQUE (hold_id, custodian_id)
);

-- Records placed under hold (prevents disposition)
CREATE TABLE hold_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hold_id             UUID NOT NULL REFERENCES legal_holds(id),
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    placed_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    released_at         TIMESTAMPTZ,
    UNIQUE (hold_id, record_id)
);

-- Notification templates and tracking
CREATE TABLE hold_notifications (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hold_id             UUID NOT NULL REFERENCES legal_holds(id),
    custodian_id        UUID NOT NULL REFERENCES custodians(id),
    notification_type   TEXT NOT NULL CHECK (notification_type IN (
                            'initial_notice', 'reminder', 'acknowledgement_request',
                            'release_notice', 'escalation'
                        )),
    channel             TEXT NOT NULL DEFAULT 'email' CHECK (channel IN ('email', 'in_app', 'both')),
    subject             TEXT NOT NULL,
    body                TEXT NOT NULL,
    sent_at             TIMESTAMPTZ NOT NULL DEFAULT now(),
    delivered_at        TIMESTAMPTZ,
    read_at             TIMESTAMPTZ,
    response            TEXT
);
```

### 6. Disposition and Defensible Deletion

```sql
-- ============================================================
-- DISPOSITION WORKFLOWS
-- ============================================================

CREATE TABLE disposition_batches (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    batch_number        TEXT NOT NULL,
    policy_id           UUID NOT NULL REFERENCES retention_policies(id),
    status              TEXT NOT NULL DEFAULT 'pending_review' CHECK (status IN (
                            'pending_review', 'under_review', 'approved',
                            'partially_approved', 'rejected', 'executing',
                            'completed', 'failed', 'cancelled'
                        )),
    total_records       INTEGER NOT NULL DEFAULT 0,
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    
    -- Review tracking
    review_started_at   TIMESTAMPTZ,
    review_completed_at TIMESTAMPTZ,
    
    -- Execution tracking
    execution_started_at TIMESTAMPTZ,
    execution_completed_at TIMESTAMPTZ,
    records_deleted      INTEGER NOT NULL DEFAULT 0,
    records_excepted     INTEGER NOT NULL DEFAULT 0,  -- held back due to holds or exceptions
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL REFERENCES users(id),
    UNIQUE (organisation_id, batch_number)
);

CREATE TABLE disposition_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id            UUID NOT NULL REFERENCES disposition_batches(id),
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'approved', 'excepted', 'deleted', 'failed'
                        )),
    exception_reason    TEXT,
    deleted_at          TIMESTAMPTZ,
    deletion_verified   BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE disposition_approvals (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    batch_id            UUID NOT NULL REFERENCES disposition_batches(id),
    approver_id         UUID NOT NULL REFERENCES users(id),
    approval_order      INTEGER NOT NULL,              -- sequence in approval chain
    decision            TEXT CHECK (decision IN ('approved', 'rejected', 'deferred')),
    comments            TEXT,
    decided_at          TIMESTAMPTZ,
    required            BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (batch_id, approver_id)
);

-- Deletion certificates (tamper-evident)
CREATE TABLE deletion_certificates (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    batch_id            UUID NOT NULL REFERENCES disposition_batches(id),
    certificate_number  TEXT NOT NULL UNIQUE,
    
    -- What was deleted
    records_count       INTEGER NOT NULL,
    total_bytes         BIGINT NOT NULL,
    data_sources        TEXT[] NOT NULL,
    date_range_from     TIMESTAMPTZ,
    date_range_to       TIMESTAMPTZ,
    
    -- Regulatory basis
    policy_name         TEXT NOT NULL,
    regulatory_citation TEXT,
    
    -- Integrity
    content_hash        BYTEA NOT NULL,                -- SHA-256 of certificate content
    previous_cert_hash  BYTEA,                         -- hash chain to previous certificate
    chain_sequence      BIGINT NOT NULL,
    
    -- Narrative
    disposition_narrative TEXT NOT NULL,                -- human-readable explanation
    
    issued_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    issued_by           UUID NOT NULL REFERENCES users(id)
);

CREATE INDEX idx_certs_org ON deletion_certificates(organisation_id);
CREATE INDEX idx_certs_chain ON deletion_certificates(chain_sequence);
```

### 7. Search and eDiscovery

```sql
-- ============================================================
-- SEARCH & eDISCOVERY
-- ============================================================

CREATE TABLE search_queries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_id           UUID REFERENCES legal_matters(id),
    name                TEXT,
    query_text          TEXT NOT NULL,                   -- Boolean search expression
    filters             JSONB NOT NULL DEFAULT '{}',     -- date range, source, custodian, etc.
    result_count        INTEGER,
    executed_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    executed_by         UUID NOT NULL REFERENCES users(id),
    is_saved            BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE collections (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_id           UUID NOT NULL REFERENCES legal_matters(id),
    name                TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL DEFAULT 'collecting' CHECK (status IN (
                            'collecting', 'processing', 'ready_for_review',
                            'under_review', 'reviewed', 'produced'
                        )),
    source_query_id     UUID REFERENCES search_queries(id),
    total_items         INTEGER NOT NULL DEFAULT 0,
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    collected_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    collected_by        UUID NOT NULL REFERENCES users(id)
);

CREATE TABLE collection_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id       UUID NOT NULL REFERENCES collections(id),
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    review_status       TEXT NOT NULL DEFAULT 'pending' CHECK (review_status IN (
                            'pending', 'responsive', 'non_responsive',
                            'privileged', 'partially_privileged', 'redacted'
                        )),
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    reviewer_notes      TEXT,
    tags                TEXT[] NOT NULL DEFAULT '{}'
);

CREATE TABLE productions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    collection_id       UUID NOT NULL REFERENCES collections(id),
    matter_id           UUID NOT NULL REFERENCES legal_matters(id),
    production_number   TEXT NOT NULL,
    format              TEXT NOT NULL CHECK (format IN ('native', 'pdf', 'tiff', 'eml', 'pst', 'load_file')),
    status              TEXT NOT NULL DEFAULT 'preparing' CHECK (status IN (
                            'preparing', 'ready', 'delivered', 'acknowledged'
                        )),
    total_items         INTEGER NOT NULL DEFAULT 0,
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    export_path         TEXT,
    delivered_at        TIMESTAMPTZ,
    delivered_to        TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL REFERENCES users(id),
    UNIQUE (organisation_id, production_number)
);

-- Chain of custody for eDiscovery
CREATE TABLE chain_of_custody (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    record_id           UUID,
    collection_id       UUID REFERENCES collections(id),
    production_id       UUID REFERENCES productions(id),
    action              TEXT NOT NULL CHECK (action IN (
                            'collected', 'processed', 'reviewed', 'tagged',
                            'redacted', 'produced', 'exported', 'delivered'
                        )),
    performed_by        UUID NOT NULL REFERENCES users(id),
    performed_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    details             TEXT,
    content_hash        BYTEA                           -- hash at this point in custody
);
```

### 8. Immutable Audit Log

```sql
-- ============================================================
-- TAMPER-EVIDENT AUDIT LOG
-- Uses HMAC-SHA256 hash chaining for immutability verification.
-- This table is append-only; no UPDATE or DELETE is permitted.
-- ============================================================

CREATE TABLE audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    
    -- What happened
    event_type          TEXT NOT NULL,                  -- 'record.ingested', 'hold.placed', 'policy.created', etc.
    event_category      TEXT NOT NULL CHECK (event_category IN (
                            'record', 'policy', 'hold', 'disposition',
                            'search', 'collection', 'production',
                            'user', 'system', 'connector'
                        )),
    severity            TEXT NOT NULL DEFAULT 'info' CHECK (severity IN (
                            'info', 'warning', 'critical'
                        )),
    
    -- Who did it
    actor_id            UUID,                           -- NULL for system events
    actor_type          TEXT NOT NULL DEFAULT 'user' CHECK (actor_type IN (
                            'user', 'system', 'api_key', 'connector'
                        )),
    actor_ip            INET,
    
    -- What was affected
    target_type         TEXT,                           -- 'record', 'policy', 'hold', etc.
    target_id           UUID,
    
    -- Details
    summary             TEXT NOT NULL,                  -- human-readable description
    details             JSONB,                          -- structured detail payload
    
    -- Tamper-evidence hash chain
    previous_hash       BYTEA,                          -- hash of previous entry
    entry_hash          BYTEA NOT NULL,                 -- HMAC-SHA256 of this entry
    
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);

-- Create monthly partitions (automated via pg_partman or cron)
CREATE TABLE audit_log_2026_01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE audit_log_2026_02 PARTITION OF audit_log
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');
-- ... additional partitions created automatically

CREATE INDEX idx_audit_org ON audit_log(organisation_id, occurred_at);
CREATE INDEX idx_audit_event ON audit_log(event_type, occurred_at);
CREATE INDEX idx_audit_target ON audit_log(target_type, target_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id) WHERE actor_id IS NOT NULL;

-- Prevent updates and deletes on audit_log
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Audit log entries cannot be modified or deleted';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_update_audit BEFORE UPDATE ON audit_log
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();
CREATE TRIGGER no_delete_audit BEFORE DELETE ON audit_log
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();
```

### 9. Compliance Reporting and Dashboards

```sql
-- ============================================================
-- COMPLIANCE REPORTING
-- ============================================================

CREATE TABLE compliance_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    report_type         TEXT NOT NULL CHECK (report_type IN (
                            'policy_adherence', 'hold_status', 'disposition_summary',
                            'coverage_gap', 'audit_readiness', 'regulatory_filing',
                            'storage_analytics', 'custom'
                        )),
    name                TEXT NOT NULL,
    parameters          JSONB NOT NULL DEFAULT '{}',
    generated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    generated_by        UUID NOT NULL REFERENCES users(id),
    data_snapshot       JSONB NOT NULL,                 -- report data at generation time
    export_format       TEXT DEFAULT 'pdf',
    export_path         TEXT,
    schedule_cron       TEXT,                           -- NULL if one-off
    is_scheduled        BOOLEAN NOT NULL DEFAULT false
);

-- Coverage gap tracking: which data sources lack policies?
CREATE TABLE coverage_gaps (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    data_source_id      UUID NOT NULL REFERENCES data_sources(id),
    gap_type            TEXT NOT NULL CHECK (gap_type IN (
                            'no_policy', 'partial_coverage', 'expired_policy',
                            'conflicting_policies', 'no_classification'
                        )),
    description         TEXT NOT NULL,
    detected_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at         TIMESTAMPTZ,
    resolved_by         UUID REFERENCES users(id)
);

-- Storage tier tracking for cost analytics
CREATE TABLE storage_metrics (
    id                  BIGSERIAL PRIMARY KEY,
    organisation_id     UUID NOT NULL,
    measured_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    storage_tier        TEXT NOT NULL,
    record_count        BIGINT NOT NULL,
    total_bytes         BIGINT NOT NULL,
    monthly_cost_cents  INTEGER                         -- estimated cost in cents
) PARTITION BY RANGE (measured_at);
```

---

## Entity Relationship Summary

```
organisations
  |-- business_units (hierarchy)
  |-- users
  |-- data_sources --> data_source_types
  |-- retention_policies --> regulatory_frameworks --> jurisdictions
  |     |-- record_policy_assignments --> archived_records
  |     |-- policy_conflicts
  |-- archived_records (partitioned by org + month)
  |     |-- hold_records --> legal_holds
  |     |-- collection_items --> collections
  |     |-- chain_of_custody
  |-- legal_matters
  |     |-- legal_holds
  |     |     |-- hold_custodian_assignments --> custodians
  |     |     |-- hold_notifications
  |     |     |-- hold_records
  |     |-- collections --> search_queries
  |     |-- productions
  |-- disposition_batches
  |     |-- disposition_items
  |     |-- disposition_approvals
  |     |-- deletion_certificates
  |-- audit_log (append-only, hash-chained)
  |-- compliance_reports
  |-- coverage_gaps
  |-- storage_metrics
```

---

## Pros and Cons

### Pros

1. **Referential integrity everywhere.** Foreign keys enforce that a legal hold cannot reference a non-existent matter, a disposition cannot bypass an approval chain, and every record traces back to a verified data source. This is critical for defensible compliance.

2. **Mature tooling.** PostgreSQL has decades of production hardening, extensive monitoring (pg_stat_statements, pgBouncer), backup tooling (pg_dump, pgBackRest, Barman), and replication (streaming, logical). The operations team does not need to learn a new technology.

3. **Row-Level Security for multi-tenancy.** PostgreSQL RLS provides database-level tenant isolation without application-layer enforcement bugs. A misconfigured API endpoint cannot leak cross-tenant data because the database itself prevents it.

4. **Declarative partitioning for lifecycle management.** Partitioning archived_records by organisation and ingestion month means that dropping a partition is an O(1) operation -- vastly more efficient than DELETE FROM with millions of rows. Cold partitions can be moved to cheaper tablespaces.

5. **Full-text search built in.** PostgreSQL's tsvector/GIN indexes handle Boolean search over archived content without requiring a separate search cluster for moderate volumes (up to tens of millions of records).

6. **Standards alignment.** The schema maps directly to OAIS concepts (Content Information, Preservation Description Information) and EDRM stages (identification through production), making compliance audits straightforward.

7. **Tamper-evident audit trail.** The hash-chained audit_log with trigger-enforced append-only semantics provides cryptographic proof that logs have not been modified, satisfying SEC 17a-4 audit-trail requirements.

### Cons

1. **Full-text search scalability ceiling.** PostgreSQL FTS is adequate for millions of records but will struggle at petabyte scale. Organisations with hundreds of millions of archived items will need an external search engine (OpenSearch/Elasticsearch) synchronized via logical replication or CDC.

2. **Schema rigidity.** Adding a new metadata field for a new connector type (e.g., a "reaction_count" field specific to Slack messages) requires a schema migration. With dozens of data source types, the archived_records table either becomes very wide or requires supplementary tables for each source type.

3. **Partition management overhead.** Dynamic partition creation for each new tenant and each new month requires automation (pg_partman or custom tooling). Mismanaged partitions lead to query planning overhead when hundreds of partitions exist.

4. **Single-node write bottleneck.** PostgreSQL's write throughput is limited by a single primary node. High-ingestion scenarios (millions of records per hour across many connectors) may require write sharding (Citus) or a queue-based buffering layer.

5. **BLOB storage is external.** Actual content (emails, documents) must be stored in object storage (S3, MinIO), not in PostgreSQL. The database holds only metadata and text extracts, which means the application must coordinate between two storage systems and maintain consistency between them.

6. **Cross-shard joins with Citus.** If horizontal sharding is adopted for scale, some analytics queries (e.g., "how many records across all tenants are under hold?") become expensive cross-shard operations.

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Primary database** | PostgreSQL 16+ | Native partitioning, RLS, JSONB, FTS |
| **Connection pooling** | PgBouncer or Supavisor | Essential for multi-tenant connection management |
| **Partition management** | pg_partman extension | Automates monthly partition creation and detachment |
| **Full-text search (scale)** | OpenSearch 2.x | When PostgreSQL FTS exceeds capacity; use logical replication for sync |
| **Object storage** | MinIO (self-hosted) or S3 with Object Lock | Immutable storage for WORM compliance |
| **Encryption** | pgcrypto + application-layer envelope encryption | Connection configs encrypted at rest; per-tenant key management |
| **Backup** | pgBackRest with S3 targets | Point-in-time recovery, parallel backup/restore |
| **Horizontal scaling** | Citus (if needed) | Distributes archived_records across nodes by organisation_id |
| **Monitoring** | pg_stat_statements + Prometheus + Grafana | Query performance, partition health, replication lag |
| **Migration tool** | Flyway or golang-migrate | Versioned, repeatable schema migrations |

---

## Migration and Scaling Considerations

### Initial Deployment (< 10M records)

- Single PostgreSQL instance with streaming replica for reads
- Monthly partitions on archived_records and audit_log
- PostgreSQL FTS for search
- MinIO single-node for object storage
- Estimated storage: ~500 GB database, ~5 TB object storage

### Growth Phase (10M - 500M records)

- Add read replicas (up to 3-5) for search and reporting queries
- Deploy OpenSearch cluster for full-text search, fed by PostgreSQL logical replication
- Increase partition granularity (weekly for high-volume tenants)
- Implement storage tiering: move cold partitions to cheaper tablespaces
- Add PgBouncer for connection management
- Estimated storage: ~5 TB database, ~100 TB object storage

### Enterprise Scale (500M+ records)

- Deploy Citus for horizontal sharding by organisation_id
- Distributed partitioning: each shard manages its own monthly partitions
- Dedicated OpenSearch cluster per data residency region
- Multi-region PostgreSQL with logical replication for cross-border compliance
- Object storage with cross-region replication and lifecycle policies
- Dedicated audit_log cluster with WORM-backed object storage for tamper evidence
- Estimated storage: ~50 TB+ database, ~1 PB+ object storage

### Data Migration Strategy

1. **Schema versioning**: All migrations tracked in a `schema_migrations` table via Flyway
2. **Zero-downtime migrations**: Use `ALTER TABLE ... ADD COLUMN` with defaults (instant in PostgreSQL 11+), backfill asynchronously
3. **Partition detachment for archival**: Old partitions detached and exported to Parquet on S3 for long-term retention at minimal cost
4. **Tenant onboarding**: Automated partition creation triggered by new organisation registration
5. **Tenant offboarding**: Partition drop + object storage lifecycle deletion with deletion certificate generation

### Backup and Disaster Recovery

- **RPO**: < 1 minute via streaming replication
- **RTO**: < 15 minutes via pgBackRest delta restore
- **Audit log durability**: Hash-chained entries replicated to separate storage; periodic Merkle root anchoring to external timestamping service
- **Cross-region**: Logical replication to secondary region for data residency compliance; active-passive failover
