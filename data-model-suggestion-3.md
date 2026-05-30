# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL with JSONB)

## Overview

This model uses PostgreSQL as a single database engine but strategically mixes normalised relational tables for stable, well-understood entities with JSONB columns for data that varies by connector type, jurisdiction, or tenant configuration. The key insight is that a data retention platform spans dozens of data source types (email, chat, documents, database records, social media), each with wildly different metadata schemas, while the core compliance concepts (policies, holds, dispositions, audit entries) are universal and well-defined.

Rather than creating a separate table for every source-specific metadata variant (which explodes the table count and requires migrations for each new connector), or going fully document-oriented (which sacrifices referential integrity and query optimisation for stable fields), this hybrid approach puts stable relational columns where they matter most and JSONB where flexibility is essential.

PostgreSQL's JSONB support includes GIN indexing, JSON path queries, partial indexing on JSONB fields, and the pg_jsonschema extension for schema validation -- making it a pragmatic middle ground that avoids the operational burden of running both a relational database and a separate document store.

---

## Design Principles

1. **Relational columns for queryable, filterable, joinable data** -- organisation IDs, lifecycle status, timestamps, foreign keys, and fields used in WHERE clauses, JOINs, or aggregate functions.
2. **JSONB columns for variable-shape data** -- connector-specific metadata, per-jurisdiction policy parameters, custom classification taxonomies, flexible report configurations, and enrichment data that evolves without migrations.
3. **JSON Schema validation on JSONB columns** -- using pg_jsonschema to enforce structure at the database level, preventing garbage data while retaining flexibility.
4. **GIN indexes on JSONB paths** -- for the specific JSONB fields that appear in frequent queries (e.g., `source_metadata->>'message_id'` for email records).

---

## Core Schema

### 1. Organisation and Users (Fully Relational)

```sql
-- ============================================================
-- ORGANISATION & USERS
-- These are stable entities with fixed schemas; fully relational.
-- ============================================================

CREATE TABLE organisations (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                TEXT NOT NULL,
    slug                TEXT NOT NULL UNIQUE,
    
    -- Relational: stable fields
    billing_plan        TEXT NOT NULL DEFAULT 'standard',
    data_residency      TEXT NOT NULL DEFAULT 'us-east-1',
    primary_jurisdiction TEXT NOT NULL DEFAULT 'US',
    
    -- JSONB: variable tenant-specific configuration
    settings            JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example settings:
    -- {
    --   "default_retention_days": 2555,
    --   "auto_classify": true,
    --   "classification_model": "v2.1",
    --   "notification_channels": ["email", "slack"],
    --   "branding": { "logo_url": "...", "primary_color": "#2563eb" },
    --   "feature_flags": { "ai_hold_scoping": true, "advanced_search": false },
    --   "custom_data_classes": [
    --     { "code": "trade_secret", "name": "Trade Secret", "sensitivity": 5 }
    --   ]
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Validate settings structure
ALTER TABLE organisations ADD CONSTRAINT check_settings
    CHECK (jsonb_typeof(settings) = 'object');

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
    is_active           BOOLEAN NOT NULL DEFAULT true,
    
    -- JSONB: user preferences that may grow arbitrarily
    preferences         JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example: { "timezone": "America/New_York", "digest_frequency": "daily",
    --            "dashboard_layout": {...}, "saved_filters": [...] }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

-- RLS
ALTER TABLE organisations ENABLE ROW LEVEL SECURITY;
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY org_isolation ON organisations
    USING (id = current_setting('app.current_org_id')::UUID);
CREATE POLICY user_isolation ON users
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

### 2. Data Sources (Relational + JSONB for Connector Config)

```sql
-- ============================================================
-- DATA SOURCES
-- Core fields are relational; connector-specific config is JSONB.
-- ============================================================

CREATE TABLE data_sources (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    source_type         TEXT NOT NULL,                  -- 'microsoft_365', 'google_workspace', 'slack', etc.
    display_name        TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
                            'pending', 'connected', 'syncing', 'error', 'disabled'
                        )),
    
    -- Relational: universal sync metadata
    last_sync_at        TIMESTAMPTZ,
    next_sync_at        TIMESTAMPTZ,
    sync_frequency_min  INTEGER NOT NULL DEFAULT 60,
    total_items_synced  BIGINT NOT NULL DEFAULT 0,
    total_bytes_synced  BIGINT NOT NULL DEFAULT 0,
    
    -- JSONB: connector-specific configuration (encrypted at application layer)
    connection_config   JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Microsoft 365 example:
    -- {
    --   "tenant_id": "abc-123",
    --   "client_id": "def-456",
    --   "client_secret_ref": "vault://secrets/m365/client_secret",
    --   "scopes": ["Mail.Read", "Files.Read.All", "Chat.Read"],
    --   "target_mailboxes": ["*"],
    --   "excluded_folders": ["Junk Email", "Drafts"]
    -- }
    --
    -- Slack example:
    -- {
    --   "workspace_id": "T01234567",
    --   "bot_token_ref": "vault://secrets/slack/bot_token",
    --   "channels": ["*"],
    --   "include_threads": true,
    --   "include_reactions": true,
    --   "include_files": true
    -- }
    
    -- JSONB: connector-specific sync state
    sync_state          JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example: { "last_delta_token": "abc123", "cursor": "xyz789",
    --            "pages_processed": 147, "estimated_remaining": 53 }
    
    -- JSONB: connector-specific capabilities and constraints
    capabilities        JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example: { "supports_hold": true, "supports_incremental_sync": true,
    --            "max_items_per_request": 1000, "rate_limit_per_minute": 600 }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_data_sources_org ON data_sources(organisation_id, status);
ALTER TABLE data_sources ENABLE ROW LEVEL SECURITY;
CREATE POLICY ds_isolation ON data_sources
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

### 3. Archived Records (The Core Hybrid Table)

This is where the hybrid approach delivers the most value. Every archived record shares a common set of relational columns, but the source-specific metadata varies enormously between email, chat messages, documents, calendar events, and database records.

```sql
-- ============================================================
-- ARCHIVED RECORDS
-- Relational columns for universal fields; JSONB for source-specific metadata.
-- Partitioned by organisation_id for tenant isolation.
-- ============================================================

CREATE TABLE archived_records (
    id                  UUID NOT NULL DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL,
    data_source_id      UUID NOT NULL,
    
    -- ========== RELATIONAL: Universal fields ==========
    -- These columns exist on every record regardless of source type.
    -- They are indexed and used in WHERE, JOIN, ORDER BY, and GROUP BY.
    
    source_type         TEXT NOT NULL,                  -- denormalized for fast filtering
    original_id         TEXT NOT NULL,
    content_type        TEXT NOT NULL,                  -- 'email', 'chat_message', 'document', etc.
    subject             TEXT,
    body_text           TEXT,                           -- extracted plaintext
    body_hash           BYTEA NOT NULL,                -- SHA-256 of original content
    file_size_bytes     BIGINT NOT NULL DEFAULT 0,
    original_format     TEXT NOT NULL,
    
    -- Dates
    source_created_at   TIMESTAMPTZ,
    source_modified_at  TIMESTAMPTZ,
    ingested_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Storage
    storage_tier        TEXT NOT NULL DEFAULT 'hot',
    storage_bucket      TEXT NOT NULL,
    storage_key         TEXT NOT NULL,
    storage_region      TEXT NOT NULL,
    
    -- Classification
    data_class          TEXT,
    ai_confidence       REAL,
    classification_verified BOOLEAN NOT NULL DEFAULT false,
    
    -- Retention
    governing_policy_id UUID,
    retention_expiry    TIMESTAMPTZ,
    lifecycle_status    TEXT NOT NULL DEFAULT 'active' CHECK (lifecycle_status IN (
                            'active', 'archived', 'hold', 'pending_disposition',
                            'approved_for_deletion', 'deleted', 'exported'
                        )),
    hold_count          INTEGER NOT NULL DEFAULT 0,
    
    -- ========== JSONB: Source-specific metadata ==========
    -- This varies per source_type and can be extended without migrations.
    
    source_metadata     JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- EMAIL example:
    -- {
    --   "message_id": "<abc@example.com>",
    --   "in_reply_to": "<def@example.com>",
    --   "thread_id": "thread_123",
    --   "from": { "name": "Alice", "email": "alice@example.com" },
    --   "to": [{ "name": "Bob", "email": "bob@example.com" }],
    --   "cc": [],
    --   "bcc": [],
    --   "headers": { "X-Mailer": "Outlook 16.0", "DKIM-Signature": "..." },
    --   "importance": "high",
    --   "has_attachments": true,
    --   "attachment_count": 3,
    --   "attachment_names": ["report.pdf", "data.xlsx", "photo.jpg"],
    --   "folder": "Inbox",
    --   "is_read": true,
    --   "is_draft": false,
    --   "conversation_index": "..."
    -- }
    --
    -- SLACK MESSAGE example:
    -- {
    --   "channel_id": "C01234567",
    --   "channel_name": "#engineering",
    --   "channel_type": "public",
    --   "thread_ts": "1672531200.000000",
    --   "reply_count": 5,
    --   "reaction_count": 12,
    --   "reactions": [{ "name": "thumbsup", "count": 8 }, ...],
    --   "is_pinned": false,
    --   "has_files": true,
    --   "file_ids": ["F01", "F02"],
    --   "mentioned_users": ["U01", "U02"],
    --   "blocks": [...],  -- Slack Block Kit structure
    --   "edited_at": "2024-01-15T10:30:00Z"
    -- }
    --
    -- DOCUMENT (SharePoint/Drive) example:
    -- {
    --   "drive_id": "b!abc123",
    --   "site_name": "Engineering",
    --   "library": "Shared Documents",
    --   "path": "/projects/q4-review/",
    --   "version": "3.0",
    --   "version_history": [
    --     { "version": "1.0", "modified_by": "alice@example.com", "date": "..." },
    --     ...
    --   ],
    --   "permissions": ["alice@example.com", "engineering-team"],
    --   "content_type_id": "0x0101",
    --   "custom_columns": { "Project": "Q4 Review", "Status": "Final" },
    --   "last_accessed_by": "bob@example.com"
    -- }
    --
    -- DATABASE RECORD (ERP/CRM archival) example:
    -- {
    --   "source_table": "sales_orders",
    --   "source_database": "erp_production",
    --   "primary_key": { "order_id": 12345 },
    --   "column_values": { "customer_id": 678, "total": 15000.00, "currency": "USD" },
    --   "column_types": { "order_id": "integer", "total": "decimal(10,2)" },
    --   "related_records": [
    --     { "table": "order_items", "count": 5, "archive_ids": [...] }
    --   ]
    -- }
    
    -- JSONB: AI enrichment data (grows over time as AI models improve)
    enrichment_data     JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "entities": ["Acme Corp", "John Smith", "Project Alpha"],
    --   "pii_detected": { "email_addresses": 2, "phone_numbers": 1, "ssn": 0 },
    --   "sentiment": "neutral",
    --   "language": "en",
    --   "topics": ["contract", "negotiation", "pricing"],
    --   "summary": "Discussion about Q4 contract renewal terms with Acme Corp",
    --   "classification_reasoning": "Contains financial terms and contract language"
    -- }
    
    -- JSONB: custom tags and labels (tenant-defined)
    custom_tags         JSONB NOT NULL DEFAULT '[]'::JSONB,
    -- Example: [
    --   { "key": "project", "value": "alpha", "applied_by": "user-uuid", "at": "..." },
    --   { "key": "review_priority", "value": "high", "applied_by": "ai", "at": "..." }
    -- ]
    
    -- Partitioning
    ingested_month      DATE NOT NULL DEFAULT date_trunc('month', now()),
    
    PRIMARY KEY (id, organisation_id, ingested_month)
) PARTITION BY LIST (organisation_id);

-- ========== INDEXES ==========

-- Relational column indexes (standard B-tree)
CREATE INDEX idx_records_lifecycle ON archived_records(organisation_id, lifecycle_status);
CREATE INDEX idx_records_expiry ON archived_records(organisation_id, retention_expiry)
    WHERE lifecycle_status = 'active' AND hold_count = 0;
CREATE INDEX idx_records_content_type ON archived_records(organisation_id, content_type);
CREATE INDEX idx_records_source ON archived_records(data_source_id);
CREATE INDEX idx_records_class ON archived_records(organisation_id, data_class)
    WHERE data_class IS NOT NULL;

-- JSONB GIN indexes for source-specific queries
CREATE INDEX idx_records_source_meta ON archived_records USING GIN (source_metadata jsonb_path_ops);

-- Partial GIN indexes for frequently queried JSONB paths per source type
CREATE INDEX idx_records_email_from ON archived_records
    USING GIN ((source_metadata->'from'))
    WHERE content_type = 'email';

CREATE INDEX idx_records_email_thread ON archived_records
    ((source_metadata->>'thread_id'))
    WHERE content_type = 'email' AND source_metadata ? 'thread_id';

CREATE INDEX idx_records_slack_channel ON archived_records
    ((source_metadata->>'channel_id'))
    WHERE content_type = 'chat_message' AND source_type = 'slack';

CREATE INDEX idx_records_enrichment ON archived_records USING GIN (enrichment_data jsonb_path_ops);
CREATE INDEX idx_records_tags ON archived_records USING GIN (custom_tags jsonb_path_ops);

-- Full-text search
CREATE INDEX idx_records_fts ON archived_records
    USING GIN (to_tsvector('english', coalesce(subject, '') || ' ' || coalesce(body_text, '')));
```

### 4. Retention Policies (Relational + JSONB for Jurisdiction-Specific Parameters)

```sql
-- ============================================================
-- RETENTION POLICIES
-- Core policy fields are relational; jurisdiction-specific parameters
-- and custom rule conditions are JSONB.
-- ============================================================

CREATE TABLE regulatory_frameworks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code                TEXT NOT NULL UNIQUE,
    name                TEXT NOT NULL,
    jurisdiction        TEXT NOT NULL,
    
    -- JSONB: framework-specific requirements that vary by regulation
    requirements        JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- GDPR example:
    -- {
    --   "right_to_erasure": true,
    --   "data_portability": true,
    --   "breach_notification_hours": 72,
    --   "dpo_required": true,
    --   "cross_border_transfer_rules": "adequacy_decision_or_sccs",
    --   "legal_bases": ["consent", "contract", "legal_obligation", "legitimate_interest"]
    -- }
    --
    -- SEC 17a-4 example:
    -- {
    --   "worm_required": true,
    --   "audit_trail_alternative": true,
    --   "min_retention_years": { "communications": 3, "financial": 6 },
    --   "immediate_access_years": 2,
    --   "third_party_access_undertaking": true,
    --   "designated_examining_authority": "FINRA"
    -- }
    
    version             TEXT,
    effective_date      DATE
);

CREATE TABLE retention_policies (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    name                TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
                            'draft', 'pending_review', 'active', 'suspended', 'retired'
                        )),
    
    -- Relational: core retention parameters
    retention_period_days   INTEGER NOT NULL,
    retention_trigger       TEXT NOT NULL DEFAULT 'creation_date',
    disposition_action      TEXT NOT NULL DEFAULT 'delete',
    requires_approval       BOOLEAN NOT NULL DEFAULT true,
    priority                INTEGER NOT NULL DEFAULT 100,
    regulatory_framework_id UUID REFERENCES regulatory_frameworks(id),
    regulatory_citation     TEXT,
    
    -- JSONB: flexible scope conditions (replaces rigid FK arrays)
    scope_conditions    JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "data_classes": ["pii", "phi"],
    --   "source_types": ["microsoft_365", "google_workspace"],
    --   "business_units": ["uuid-1", "uuid-2"],
    --   "jurisdictions": ["EU", "UK"],
    --   "content_types": ["email", "document"],
    --   "custom_conditions": [
    --     { "field": "source_metadata.folder", "operator": "equals", "value": "Inbox" },
    --     { "field": "enrichment_data.pii_detected.ssn", "operator": "greater_than", "value": 0 }
    --   ]
    -- }
    
    -- JSONB: approval workflow configuration
    approval_workflow   JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "stages": [
    --     { "role": "records_manager", "required": true, "timeout_days": 5 },
    --     { "role": "compliance_admin", "required": true, "timeout_days": 3 },
    --     { "role": "legal_admin", "required": false, "timeout_days": 7 }
    --   ],
    --   "escalation_policy": "auto_approve_after_timeout",
    --   "minimum_approvals": 2,
    --   "require_comments": true
    -- }
    
    -- JSONB: jurisdiction-specific retention overrides
    jurisdiction_overrides JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "EU": {
    --     "retention_period_days": 1825,
    --     "erasure_request_handling": "prioritise_erasure",
    --     "legal_basis_required": true
    --   },
    --   "US_FINRA": {
    --     "retention_period_days": 2190,
    --     "worm_storage_required": true,
    --     "immediate_access_period_days": 730
    --   }
    -- }
    
    created_by          UUID REFERENCES users(id),
    approved_by         UUID REFERENCES users(id),
    approved_at         TIMESTAMPTZ,
    effective_from      TIMESTAMPTZ,
    effective_until     TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policies_org ON retention_policies(organisation_id, status);
CREATE INDEX idx_policies_scope ON retention_policies USING GIN (scope_conditions jsonb_path_ops);

-- Policy-to-record assignments
CREATE TABLE record_policy_assignments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    policy_id           UUID NOT NULL REFERENCES retention_policies(id),
    computed_expiry     TIMESTAMPTZ NOT NULL,
    is_governing        BOOLEAN NOT NULL DEFAULT false,
    assigned_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- JSONB: assignment reasoning (for defensibility)
    assignment_details  JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "matched_conditions": ["data_class:pii", "jurisdiction:EU"],
    --   "trigger_date": "2024-01-15T10:30:00Z",
    --   "trigger_type": "creation_date",
    --   "classification_method": "ai",
    --   "classification_confidence": 0.94
    -- }
    
    UNIQUE (record_id, policy_id)
);
```

### 5. Legal Holds (Relational + JSONB for Scope and AI Suggestions)

```sql
-- ============================================================
-- LEGAL HOLDS
-- ============================================================

CREATE TABLE legal_matters (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_number       TEXT NOT NULL,
    name                TEXT NOT NULL,
    matter_type         TEXT NOT NULL,
    status              TEXT NOT NULL DEFAULT 'active',
    lead_attorney_id    UUID REFERENCES users(id),
    
    -- JSONB: matter-specific details that vary by type
    matter_details      JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Litigation example:
    -- {
    --   "case_number": "2024-CV-12345",
    --   "court": "US District Court, Southern District of New York",
    --   "judge": "Hon. Smith",
    --   "opposing_counsel": "Jones & Associates",
    --   "complaint_filed": "2024-06-01",
    --   "estimated_exposure": 5000000,
    --   "insurance_notified": true
    -- }
    --
    -- Regulatory inquiry example:
    -- {
    --   "agency": "SEC",
    --   "inquiry_number": "INQ-2024-789",
    --   "subject": "Trading activity review Q3 2024",
    --   "response_deadline": "2024-09-15",
    --   "assigned_examiner": "Jane Doe"
    -- }
    
    opened_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    closed_at           TIMESTAMPTZ,
    created_by          UUID NOT NULL REFERENCES users(id),
    UNIQUE (organisation_id, matter_number)
);

CREATE TABLE custodians (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    user_id             UUID REFERENCES users(id),
    external_name       TEXT,
    external_email      TEXT,
    department          TEXT,
    title               TEXT,
    employment_status   TEXT DEFAULT 'active',
    
    -- JSONB: custodian data source access map
    data_source_access  JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "email": { "source_id": "uuid", "mailbox": "alice@example.com" },
    --   "onedrive": { "source_id": "uuid", "drive_id": "b!abc123" },
    --   "slack": { "source_id": "uuid", "user_id": "U01234567" },
    --   "teams": { "source_id": "uuid", "user_id": "alice@example.com" }
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE legal_holds (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_id           UUID NOT NULL REFERENCES legal_matters(id),
    hold_name           TEXT NOT NULL,
    description         TEXT,
    status              TEXT NOT NULL DEFAULT 'draft',
    
    -- Relational: core hold parameters
    issued_at           TIMESTAMPTZ,
    issued_by           UUID REFERENCES users(id),
    released_at         TIMESTAMPTZ,
    released_by         UUID REFERENCES users(id),
    
    -- JSONB: flexible scope definition
    scope_definition    JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "data_sources": ["uuid-1", "uuid-2"],
    --   "date_range": { "from": "2023-01-01", "to": "2024-06-30" },
    --   "keywords": ["project alpha", "merger", "acquisition"],
    --   "boolean_query": "(merger OR acquisition) AND NOT (draft OR template)",
    --   "content_types": ["email", "chat_message", "document"],
    --   "custodian_departments": ["legal", "finance", "executive"],
    --   "exclude_patterns": ["newsletter@*", "noreply@*"]
    -- }
    
    -- JSONB: AI-suggested scope (for hold scoping assistance)
    ai_scope_suggestions JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "suggested_custodians": [
    --     { "custodian_id": "uuid", "name": "Alice", "relevance_score": 0.95,
    --       "reason": "Sent 47 emails containing keywords during the date range" }
    --   ],
    --   "suggested_data_sources": [
    --     { "source_id": "uuid", "name": "Slack #deals", "relevance_score": 0.88,
    --       "reason": "Channel contains 23 messages matching hold keywords" }
    --   ],
    --   "estimated_record_count": 15000,
    --   "estimated_storage_gb": 4.2,
    --   "generated_at": "2024-07-01T10:00:00Z",
    --   "model_version": "hold-scope-v2.1"
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE hold_custodian_assignments (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hold_id             UUID NOT NULL REFERENCES legal_holds(id),
    custodian_id        UUID NOT NULL REFERENCES custodians(id),
    status              TEXT NOT NULL DEFAULT 'pending',
    notified_at         TIMESTAMPTZ,
    acknowledged_at     TIMESTAMPTZ,
    reminder_count      INTEGER NOT NULL DEFAULT 0,
    released_at         TIMESTAMPTZ,
    
    -- JSONB: notification history
    notification_history JSONB NOT NULL DEFAULT '[]'::JSONB,
    -- Example:
    -- [
    --   { "type": "initial_notice", "sent_at": "...", "channel": "email",
    --     "template": "hold_notice_v2", "delivered": true },
    --   { "type": "reminder", "sent_at": "...", "channel": "email",
    --     "escalation_level": 1 },
    --   { "type": "acknowledgement", "received_at": "...", "ip": "10.0.0.1" }
    -- ]
    
    UNIQUE (hold_id, custodian_id)
);

CREATE TABLE hold_records (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    hold_id             UUID NOT NULL REFERENCES legal_holds(id),
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    placed_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    released_at         TIMESTAMPTZ,
    UNIQUE (hold_id, record_id)
);
```

### 6. Disposition and Deletion Certificates

```sql
-- ============================================================
-- DISPOSITION WORKFLOWS
-- ============================================================

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
    
    -- JSONB: batch summary statistics
    batch_summary       JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "by_content_type": { "email": 5000, "document": 1200, "chat_message": 800 },
    --   "by_data_source": { "Microsoft 365": 4500, "Slack": 1500, "Google Workspace": 1000 },
    --   "by_data_class": { "pii": 2000, "financial": 1500, "general": 3500 },
    --   "date_range": { "oldest": "2017-01-15", "newest": "2017-12-31" },
    --   "avg_record_age_days": 2920,
    --   "estimated_cost_savings_monthly_cents": 15000
    -- }
    
    -- JSONB: execution details
    execution_log       JSONB NOT NULL DEFAULT '[]'::JSONB,
    -- Example:
    -- [
    --   { "step": "validation", "started": "...", "completed": "...",
    --     "result": "passed", "checks": ["no_active_holds", "all_approvals_received"] },
    --   { "step": "storage_deletion", "started": "...", "completed": "...",
    --     "objects_deleted": 7000, "bytes_freed": 15000000000 },
    --   { "step": "index_removal", "started": "...", "completed": "...",
    --     "records_removed_from_search": 7000 },
    --   { "step": "metadata_tombstone", "started": "...", "completed": "...",
    --     "tombstones_created": 7000 }
    -- ]
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL REFERENCES users(id),
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
    
    -- Relational: integrity chain
    content_hash        BYTEA NOT NULL,
    previous_cert_hash  BYTEA,
    chain_sequence      BIGINT NOT NULL,
    
    -- JSONB: rich certificate content
    certificate_content JSONB NOT NULL,
    -- Example:
    -- {
    --   "organisation_name": "Acme Corp",
    --   "policy_name": "GDPR Personal Data - 5 Year Retention",
    --   "regulatory_basis": "GDPR Article 17 / Article 5(1)(e)",
    --   "disposition_narrative": "7,000 records archived between January 2017 and December 2017
    --     were deleted following completion of the 5-year retention period mandated by GDPR.
    --     No active legal holds applied. All records were classified as personal data (PII)
    --     originating from Microsoft 365 and Slack. Two-stage approval was completed by
    --     Records Manager Jane Smith on 2024-07-10 and Compliance Officer John Doe on 2024-07-12.",
    --   "approval_chain": [
    --     { "name": "Jane Smith", "role": "Records Manager", "approved_at": "2024-07-10T14:30:00Z" },
    --     { "name": "John Doe", "role": "Compliance Officer", "approved_at": "2024-07-12T09:15:00Z" }
    --   ],
    --   "data_summary": {
    --     "by_type": { "email": 5000, "chat_message": 1500, "document": 500 },
    --     "date_range": { "from": "2017-01-01", "to": "2017-12-31" },
    --     "data_sources": ["Microsoft 365 - Acme Tenant", "Slack - Acme Workspace"]
    --   },
    --   "storage_verification": {
    --     "objects_verified_deleted": 7000,
    --     "search_index_purged": true,
    --     "backup_retention_note": "Backups containing these records will expire per 90-day rotation"
    --   }
    -- }
    
    issued_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    issued_by           UUID NOT NULL REFERENCES users(id)
);
```

### 7. Search and eDiscovery

```sql
-- ============================================================
-- eDISCOVERY
-- ============================================================

CREATE TABLE search_queries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    matter_id           UUID REFERENCES legal_matters(id),
    name                TEXT,
    
    -- JSONB: rich query specification
    query_spec          JSONB NOT NULL,
    -- Example:
    -- {
    --   "boolean_query": "(merger OR acquisition) AND confidential",
    --   "date_range": { "from": "2023-01-01", "to": "2024-06-30" },
    --   "content_types": ["email", "document"],
    --   "data_sources": ["uuid-1", "uuid-2"],
    --   "custodians": ["uuid-a", "uuid-b"],
    --   "data_classes": ["financial", "pii"],
    --   "source_metadata_filters": [
    --     { "path": "source_metadata.folder", "operator": "equals", "value": "Inbox" },
    --     { "path": "source_metadata.has_attachments", "operator": "equals", "value": true }
    --   ],
    --   "exclude_keywords": ["newsletter", "automated"],
    --   "file_size_range": { "min_bytes": 0, "max_bytes": 52428800 },
    --   "sort": { "field": "source_created_at", "order": "desc" }
    -- }
    
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
    status              TEXT NOT NULL DEFAULT 'collecting',
    source_query_id     UUID REFERENCES search_queries(id),
    total_items         INTEGER NOT NULL DEFAULT 0,
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    collected_by        UUID NOT NULL REFERENCES users(id),
    collected_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE collection_items (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    collection_id       UUID NOT NULL REFERENCES collections(id),
    record_id           UUID NOT NULL,
    organisation_id     UUID NOT NULL,
    review_status       TEXT NOT NULL DEFAULT 'pending',
    reviewed_by         UUID REFERENCES users(id),
    reviewed_at         TIMESTAMPTZ,
    
    -- JSONB: review annotations
    review_annotations  JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "tags": ["key_document", "hot_doc"],
    --   "privilege_type": "attorney_client",
    --   "redaction_notes": "Redact paragraph 3 - contains privileged legal advice",
    --   "relevance_score": 0.92,
    --   "reviewer_notes": "Critical evidence of knowledge prior to filing date",
    --   "coding_decisions": { "responsive": true, "privileged": false, "confidential": true }
    -- }
    
    UNIQUE (collection_id, record_id)
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
    total_bytes         BIGINT NOT NULL DEFAULT 0,
    
    -- JSONB: production configuration and delivery details
    production_config   JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "format_options": {
    --     "native_with_text": true, "pdf_ocr": false,
    --     "include_metadata_load_file": true, "load_file_format": "concordance"
    --   },
    --   "numbering": { "prefix": "ACME", "start": 1, "zero_pad": 7 },
    --   "delivery": {
    --     "method": "secure_transfer",
    --     "recipient": "opposing_counsel@law.com",
    --     "encryption": "AES-256",
    --     "transfer_url": "https://secure.example.com/prod/ACME-001"
    --   },
    --   "bates_range": { "start": "ACME0000001", "end": "ACME0005000" }
    -- }
    
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by          UUID NOT NULL REFERENCES users(id),
    delivered_at        TIMESTAMPTZ,
    UNIQUE (organisation_id, production_number)
);
```

### 8. Audit Log (Relational + JSONB Details)

```sql
-- ============================================================
-- AUDIT LOG (append-only, hash-chained)
-- Core fields relational for fast filtering; details in JSONB.
-- ============================================================

CREATE TABLE audit_log (
    id                  BIGSERIAL NOT NULL,
    organisation_id     UUID NOT NULL,
    
    -- Relational: indexed filter columns
    event_type          TEXT NOT NULL,
    event_category      TEXT NOT NULL,
    severity            TEXT NOT NULL DEFAULT 'info',
    actor_id            UUID,
    actor_type          TEXT NOT NULL DEFAULT 'user',
    target_type         TEXT,
    target_id           UUID,
    occurred_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Relational: human-readable summary
    summary             TEXT NOT NULL,
    
    -- JSONB: variable event details
    event_details       JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Examples by event type:
    --
    -- policy.created:
    -- { "policy_name": "GDPR 5yr", "retention_days": 1825, "scope": {...} }
    --
    -- record.classified:
    -- { "data_class": "pii", "method": "ai", "confidence": 0.94,
    --    "previous_class": null, "model_version": "cls-v2.1" }
    --
    -- hold.custodian_notified:
    -- { "hold_name": "SEC Inquiry 2024", "custodian_email": "alice@example.com",
    --    "notification_type": "initial_notice", "channel": "email" }
    --
    -- disposition.certificate_issued:
    -- { "certificate_number": "DEL-2024-0042", "records_count": 7000,
    --    "total_bytes": 15000000000, "policy_name": "GDPR 5yr" }
    
    -- Tamper-evidence
    previous_hash       BYTEA,
    entry_hash          BYTEA NOT NULL,
    
    PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);

-- Monthly partitions
CREATE TABLE audit_log_y2026m01 PARTITION OF audit_log
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
-- ... auto-created by pg_partman

CREATE INDEX idx_audit_org_time ON audit_log(organisation_id, occurred_at DESC);
CREATE INDEX idx_audit_event ON audit_log(event_type);
CREATE INDEX idx_audit_target ON audit_log(target_type, target_id);
CREATE INDEX idx_audit_details ON audit_log USING GIN (event_details jsonb_path_ops);

-- Append-only enforcement
CREATE OR REPLACE FUNCTION prevent_audit_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Audit log is append-only';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_update_audit BEFORE UPDATE ON audit_log
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();
CREATE TRIGGER no_delete_audit BEFORE DELETE ON audit_log
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_modification();
```

### 9. Compliance Reporting

```sql
-- ============================================================
-- COMPLIANCE REPORTING
-- ============================================================

CREATE TABLE compliance_reports (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    report_type         TEXT NOT NULL,
    name                TEXT NOT NULL,
    
    -- JSONB: report parameters and snapshot
    parameters          JSONB NOT NULL DEFAULT '{}'::JSONB,
    data_snapshot       JSONB NOT NULL DEFAULT '{}'::JSONB,
    
    -- JSONB: schedule configuration
    schedule_config     JSONB,
    -- Example:
    -- {
    --   "cron": "0 8 * * MON",
    --   "timezone": "America/New_York",
    --   "recipients": ["compliance@example.com", "legal@example.com"],
    --   "format": "pdf",
    --   "include_drill_down": true
    -- }
    
    generated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    generated_by        UUID NOT NULL REFERENCES users(id)
);

CREATE TABLE coverage_gaps (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id     UUID NOT NULL REFERENCES organisations(id),
    data_source_id      UUID NOT NULL REFERENCES data_sources(id),
    gap_type            TEXT NOT NULL,
    description         TEXT NOT NULL,
    
    -- JSONB: gap analysis details
    gap_details         JSONB NOT NULL DEFAULT '{}'::JSONB,
    -- Example:
    -- {
    --   "uncovered_record_count": 15000,
    --   "uncovered_content_types": ["chat_message"],
    --   "suggested_policies": ["uuid-1", "uuid-2"],
    --   "risk_level": "high",
    --   "regulatory_exposure": ["FINRA 4511", "SEC 17a-4"]
    -- }
    
    detected_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    resolved_at         TIMESTAMPTZ
);
```

---

## JSON Schema Validation

Use the `pg_jsonschema` extension to enforce structure on critical JSONB columns without losing flexibility.

```sql
-- Install the extension
CREATE EXTENSION IF NOT EXISTS pg_jsonschema;

-- Validate source_metadata for email records
ALTER TABLE archived_records ADD CONSTRAINT check_email_metadata
    CHECK (
        content_type != 'email' OR
        jsonb_matches_schema(source_metadata, '{
            "type": "object",
            "required": ["message_id", "from"],
            "properties": {
                "message_id": { "type": "string" },
                "from": {
                    "type": "object",
                    "required": ["email"],
                    "properties": {
                        "name": { "type": "string" },
                        "email": { "type": "string", "format": "email" }
                    }
                },
                "to": { "type": "array" },
                "cc": { "type": "array" },
                "has_attachments": { "type": "boolean" },
                "attachment_count": { "type": "integer", "minimum": 0 }
            }
        }')
    );

-- Validate scope_conditions for retention policies
ALTER TABLE retention_policies ADD CONSTRAINT check_scope_conditions
    CHECK (
        jsonb_matches_schema(scope_conditions, '{
            "type": "object",
            "properties": {
                "data_classes": { "type": "array", "items": { "type": "string" } },
                "source_types": { "type": "array", "items": { "type": "string" } },
                "jurisdictions": { "type": "array", "items": { "type": "string" } },
                "content_types": { "type": "array", "items": { "type": "string" } },
                "custom_conditions": { "type": "array" }
            }
        }')
    );
```

---

## Pros and Cons

### Pros

1. **Best of both worlds.** Stable compliance fields (lifecycle_status, retention_expiry, hold_count) are relational with full index, constraint, and join support. Variable data (email headers, Slack reactions, AI enrichment) lives in JSONB where it can evolve independently per connector.

2. **Single database engine.** One PostgreSQL deployment handles both relational and document workloads. No need to synchronise between PostgreSQL and MongoDB, no dual-write consistency headaches, and one operational runbook.

3. **Zero-migration connector expansion.** Adding a new data source type (e.g., Microsoft Teams or Salesforce) requires only a new connector adapter in the application. The database schema does not change -- source-specific metadata goes into the existing `source_metadata` JSONB column. The connector defines its own JSON schema for validation.

4. **GIN indexes for JSONB querying.** PostgreSQL's `jsonb_path_ops` GIN indexes provide fast containment queries on JSONB fields. Searching for all emails from a specific sender (`source_metadata @> '{"from": {"email": "alice@example.com"}}'`) uses the index efficiently.

5. **JSON Schema validation.** The `pg_jsonschema` extension enforces structural requirements on JSONB columns at the database level, preventing garbage data while retaining flexibility. Each connector defines its own schema constraint.

6. **Rich eDiscovery annotations.** Review annotations, coding decisions, and privilege tags vary enormously per collection and per reviewer workflow. JSONB captures this without a proliferation of narrow tables.

7. **Tenant-specific customisation.** Organisation settings, custom data classes, feature flags, and branding all live in JSONB. Tenants can be configured differently without schema changes.

8. **AI-friendly.** Enrichment data, AI classification reasoning, and AI-suggested hold scoping all have unpredictable structures that evolve as models improve. JSONB columns accommodate this without migrations.

### Cons

1. **JSONB queries are slower than relational queries.** While GIN indexes help, JSONB path queries are fundamentally slower than B-tree lookups on indexed columns. Queries like "find all records where `source_metadata->'from'->>'email' = 'alice@example.com'`" are slower than the equivalent relational column query.

2. **No foreign key enforcement in JSONB.** The `scope_conditions.data_classes` array in retention_policies cannot reference a `data_classes` table via foreign key. The application must enforce referential integrity for JSONB-embedded references.

3. **Schema drift risk.** Even with pg_jsonschema validation, JSONB columns can accumulate inconsistent data over time as connector implementations evolve. A disciplined schema registry is needed to track which fields each connector version writes.

4. **Reporting complexity.** Aggregating across JSONB fields (e.g., "count of records by email folder across all tenants") requires JSONB extraction in GROUP BY clauses, which is verbose and can be slow without targeted indexes.

5. **Backup size.** JSONB data is stored as binary and does not compress as well as normalised relational data. The `source_metadata` column on billions of records can grow very large, especially for content types with rich metadata (email headers, Slack blocks).

6. **Migration complexity for JSONB evolution.** While adding new JSONB fields requires no schema migration, renaming or restructuring existing JSONB fields requires data migration scripts that parse and update JSON for potentially billions of rows.

7. **Developer discipline required.** The boundary between "this should be a relational column" and "this should be JSONB" is a judgment call. Without clear guidelines, teams will inconsistently use JSONB for data that should be relational (e.g., putting `lifecycle_status` in JSONB, which defeats indexing).

---

## Technology Recommendations

| Component | Recommendation | Rationale |
|-----------|---------------|-----------|
| **Database** | PostgreSQL 16+ | JSONB, GIN indexes, partitioning, RLS, pg_jsonschema |
| **JSONB validation** | pg_jsonschema extension | Database-level structural validation of JSONB columns |
| **Connection pool** | PgBouncer | Multi-tenant connection management |
| **Full-text search** | PostgreSQL FTS (small); OpenSearch (scale) | Graduated approach based on data volume |
| **Object storage** | MinIO / S3 with Object Lock | WORM compliance for archived content |
| **JSON Schema registry** | Application-level schema registry | Track JSONB schemas per connector version |
| **Backup** | pgBackRest | JSONB-aware backup with parallel restore |
| **Monitoring** | pg_stat_statements + JSONB query analysis | Track slow JSONB path queries |
| **Migration** | Flyway with JSONB migration scripts | Versioned schema + data migrations for JSONB restructuring |
| **Partition management** | pg_partman | Automated monthly partitions for records and audit log |

---

## Migration and Scaling Considerations

### Initial Deployment

- Single PostgreSQL instance with RLS for multi-tenancy
- `source_metadata` GIN index covers most JSONB query patterns
- pg_jsonschema validates core connector metadata schemas
- Expected capacity: ~50M records, ~500 GB database

### Growth Phase (50M - 1B records)

- Read replicas for search and reporting queries
- OpenSearch for full-text search (synced via logical replication)
- Selective extraction of hot JSONB paths into generated columns:

```sql
-- For frequently queried JSONB paths, create generated columns
ALTER TABLE archived_records 
    ADD COLUMN email_from TEXT GENERATED ALWAYS AS (source_metadata->>'from') STORED;
    
ALTER TABLE archived_records
    ADD COLUMN email_thread_id TEXT GENERATED ALWAYS AS (source_metadata->>'thread_id') STORED;

-- Now these can use standard B-tree indexes
CREATE INDEX idx_records_email_from_gen ON archived_records(email_from)
    WHERE email_from IS NOT NULL;
```

- Partition large tenants into dedicated sub-partitions
- JSONB compression with TOAST tuning

### Enterprise Scale (1B+ records)

- Citus for horizontal sharding by organisation_id
- Per-region PostgreSQL clusters for data residency
- JSONB to Parquet ETL pipeline for analytics (extract JSONB to columnar format)
- Materialised views for cross-tenant reporting:

```sql
CREATE MATERIALIZED VIEW mv_records_by_class_monthly AS
SELECT 
    organisation_id,
    data_class,
    date_trunc('month', ingested_at) AS month,
    count(*) AS record_count,
    sum(file_size_bytes) AS total_bytes
FROM archived_records
WHERE lifecycle_status != 'deleted'
GROUP BY organisation_id, data_class, date_trunc('month', ingested_at);
```

### JSONB Schema Evolution Strategy

1. **Version field**: Every JSONB column includes a `"_schema_version": 1` field
2. **Backward compatibility**: New connector versions only add fields; never remove or rename
3. **Migration scripts**: When restructuring is needed, use batch UPDATE with `jsonb_set()` during low-traffic windows
4. **Read-time upcasting**: Application reads handle both old and new JSONB structures via adapter pattern
5. **Generated columns**: High-query JSONB paths promoted to generated columns when query patterns stabilise
