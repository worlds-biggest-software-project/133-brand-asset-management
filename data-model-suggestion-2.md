# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Brand Asset Management · Created: 2026-05-19

## Philosophy

This approach treats every state change as an immutable event appended to a single event store. The event store is the authoritative source of truth; all queryable state (asset listings, metadata views, license dashboards) is derived by projecting events into optimised read models. This is the CQRS (Command Query Responsibility Segregation) pattern applied to digital asset management.

The design is directly inspired by compliance-heavy systems used in financial services and healthcare, where audit trails are not an afterthought but the foundation of the data model. IntelligenceBank's compliance-first approach hints at this philosophy, but no current DAM platform fully implements event sourcing. For a brand asset management platform that must satisfy ISO 15489 records management, GDPR biometric consent tracking, and C2PA content provenance, an immutable event log provides a naturally auditable, temporally queryable, and legally defensible data layer.

Write operations (commands) produce events; read operations (queries) hit materialised views that are rebuilt from events. This separation allows read models to be independently tuned, cached, or replaced without touching the event store.

**Best for:** Regulated industries (pharma, finance, media) requiring complete audit trails, temporal queries ("what was the license status on date X?"), and provenance chains for AI-generated content.

**Trade-offs:**
- Pro: Complete, immutable audit trail by construction — no separate audit_log table needed
- Pro: Temporal queries are natural — replay events to any point in time
- Pro: Event replay enables new read models without data migration
- Pro: Natural fit for C2PA provenance chains (each manifest operation is an event)
- Con: Higher write latency (event + projection update)
- Con: Eventual consistency between event store and read models
- Con: More complex to implement and debug than direct CRUD
- Con: Read model rebuilds can be slow for large event histories
- Con: Developers must learn event-sourcing patterns; steeper onboarding

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 15489 | The event store itself IS the records management system — every action is an immutable record with timestamp, actor, and context |
| IPTC Photo Metadata 2025.1 | Metadata changes recorded as `MetadataUpdated` events; read model materialises current IPTC fields |
| PLUS License Data Format | License lifecycle modelled as events: `LicenseGranted`, `LicenseExpired`, `LicenseRenewed` |
| C2PA 2.4 | C2PA manifests and assertions stored as events in the provenance chain, naturally mirroring the C2PA "history of actions" model |
| GDPR | Consent events (`BiometricConsentGranted`, `BiometricConsentWithdrawn`) create an auditable consent timeline |
| EU AI Act | AI processing events (`AiTaggingPerformed`, `AiContentGenerated`) with model metadata satisfy GPAI documentation requirements |
| Dublin Core | Metadata projections expose Dublin Core fields; changes traced through events |

---

## Event Store (Source of Truth)

```sql
-- The single append-only event store
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                 -- aggregate ID (asset_id, user_id, brand_id, etc.)
    stream_type     VARCHAR(50) NOT NULL,          -- 'Asset', 'License', 'Brand', 'User', 'Collection'
    event_type      VARCHAR(100) NOT NULL,         -- 'AssetUploaded', 'MetadataUpdated', 'LicenseExpired'
    event_version   INTEGER NOT NULL,              -- per-stream sequence number for ordering
    tenant_id       UUID NOT NULL,
    actor_id        UUID,                          -- user or system that caused the event
    actor_type      VARCHAR(30) NOT NULL DEFAULT 'user',  -- 'user', 'system', 'api_client', 'ai_agent'
    payload         JSONB NOT NULL,                -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',   -- correlation_id, causation_id, ip_address, user_agent
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)              -- optimistic concurrency control
);

-- Partition by month for performance
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_events_stream ON events(stream_id, event_version);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_tenant_time ON events(tenant_id, created_at DESC);
CREATE INDEX idx_events_actor ON events(actor_id);
CREATE INDEX idx_events_payload ON events USING GIN(payload);
```

### Event Type Taxonomy

```
-- Asset lifecycle
AssetUploaded           -- {file_name, file_size, mime_type, storage_path, checksum_sha256}
AssetVersionAdded       -- {version_number, file_name, file_size, storage_path, change_notes}
AssetMetadataUpdated    -- {field_name, old_value, new_value, source: 'manual'|'ai'|'import'}
AssetStatusChanged      -- {old_status, new_status, reason}
AssetTagged             -- {tag_name, tag_type, source, confidence}
AssetUntagged           -- {tag_name, reason}
AssetCategorised        -- {category_id, category_name}
AssetDownloaded         -- {user_id, format, resolution, channel}
AssetArchived           -- {reason, retention_policy_id}
AssetDeleted            -- {reason, deleted_by}

-- Metadata standards
IptcMetadataSet         -- {headline, caption, keywords[], creator, copyright_notice, ...}
DublinCoreMetadataSet   -- {dc_title, dc_creator, dc_subject[], ...}
ExifMetadataExtracted   -- {camera_make, camera_model, gps_latitude, gps_longitude, ...}
XmpPacketExtracted      -- {namespace, raw_xmp_xml}

-- Rights and licensing
LicenseGranted          -- {license_type, licensor, licensee, start_date, expiry_date, territory, plus_fields}
LicenseExpired          -- {license_id, expiry_date}
LicenseRenewed          -- {license_id, new_expiry_date, terms_changed}
LicenseRevoked          -- {license_id, reason}
ModelReleaseObtained    -- {person_name, signed_date, expiry_date}
ModelReleaseExpired     -- {release_id, person_name}
PropertyReleaseObtained -- {property_name, signed_date}

-- AI processing
AiTaggingPerformed      -- {model_name, model_version, tags_generated[], processing_time_ms}
AiEmbeddingGenerated    -- {model_name, embedding_type, dimension}
FaceDetected            -- {bounding_box, confidence, face_encoding_ref}
FaceIdentified          -- {detected_face_id, person_id, person_name, confidence}
ComplianceCheckRun      -- {check_type, severity, message, details}
AiContentGenerated      -- {ai_system, ai_version, prompt, output_asset_id}

-- C2PA provenance
C2paManifestAttached    -- {manifest_label, claim_generator, signature_alg, assertions[]}
C2paManifestValidated   -- {manifest_id, is_valid, validation_errors[]}

-- Brand governance
BrandCreated            -- {name, slug, parent_brand_id}
BrandGuidelinePublished -- {guideline_id, title, section_order}
BrandColourDefined      -- {name, hex_value, pantone_code, usage_context}
BrandComplianceViolation -- {asset_id, violation_type, severity, details}

-- GDPR / consent
BiometricConsentGranted -- {person_id, consent_type, legal_basis}
BiometricConsentWithdrawn -- {person_id, withdrawal_reason}

-- Workflow
WorkflowStarted         -- {template_id, asset_id, initiator_id}
WorkflowStepCompleted   -- {step_number, action, actor_id, comment}
WorkflowCompleted       -- {outcome: 'approved'|'rejected', completed_by}

-- Distribution
AssetDistributed        -- {channel_id, channel_type, public_url, variant_spec}
ShareLinkCreated        -- {token, target_type, target_id, expires_at}
ShareLinkAccessed       -- {token, accessor_ip, user_agent}
```

---

## Read Models (Materialised Projections)

These tables are rebuilt from events. They can be dropped and recreated at any time.

### Asset Read Model

```sql
CREATE TABLE rm_assets (
    asset_id        UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    brand_id        UUID,
    asset_type      VARCHAR(50) NOT NULL,
    title           VARCHAR(512) NOT NULL,
    description     TEXT,
    status          VARCHAR(30) NOT NULL,
    current_version INTEGER NOT NULL DEFAULT 1,

    -- Denormalised current file info
    file_name       VARCHAR(512),
    file_size_bytes BIGINT,
    mime_type       VARCHAR(255),
    storage_path    TEXT,
    width_px        INTEGER,
    height_px       INTEGER,
    duration_secs   NUMERIC(10,2),

    -- Denormalised metadata (most-queried fields)
    iptc_headline   VARCHAR(256),
    iptc_keywords   TEXT[],
    iptc_copyright  VARCHAR(500),
    iptc_country_code CHAR(3),
    dc_creator      VARCHAR(255),
    dc_type         VARCHAR(100),

    -- AI flags
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    ai_system_used  VARCHAR(255),
    tag_count       INTEGER NOT NULL DEFAULT 0,
    face_count      INTEGER NOT NULL DEFAULT 0,

    -- Compliance summary
    has_valid_license BOOLEAN NOT NULL DEFAULT false,
    license_expiry  DATE,
    compliance_violations INTEGER NOT NULL DEFAULT 0,

    -- Timestamps
    uploaded_by     UUID NOT NULL,
    uploaded_at     TIMESTAMPTZ NOT NULL,
    last_modified_by UUID,
    last_modified_at TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    last_event_version INTEGER NOT NULL,           -- tracks projection progress

    -- Full-text search
    search_vector   tsvector
);

CREATE INDEX idx_rm_assets_tenant ON rm_assets(tenant_id);
CREATE INDEX idx_rm_assets_brand ON rm_assets(brand_id);
CREATE INDEX idx_rm_assets_status ON rm_assets(status);
CREATE INDEX idx_rm_assets_keywords ON rm_assets USING GIN(iptc_keywords);
CREATE INDEX idx_rm_assets_search ON rm_assets USING GIN(search_vector);
CREATE INDEX idx_rm_assets_license_expiry ON rm_assets(license_expiry) WHERE license_expiry IS NOT NULL;
```

### Asset Tags Read Model

```sql
CREATE TABLE rm_asset_tags (
    asset_id        UUID NOT NULL,
    tag_name        VARCHAR(100) NOT NULL,
    tag_type        VARCHAR(50) NOT NULL,
    source          VARCHAR(30) NOT NULL,
    confidence      NUMERIC(4,3),
    tagged_at       TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (asset_id, tag_name)
);

CREATE INDEX idx_rm_asset_tags_name ON rm_asset_tags(tag_name);
CREATE INDEX idx_rm_asset_tags_type ON rm_asset_tags(tag_type);
```

### License Timeline Read Model

```sql
CREATE TABLE rm_license_timeline (
    license_id      UUID PRIMARY KEY,
    asset_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    license_type    VARCHAR(50) NOT NULL,
    licensor_name   VARCHAR(255),
    licensee_name   VARCHAR(255),
    usage_territory VARCHAR(10),
    license_start   DATE,
    license_expiry  DATE,
    is_perpetual    BOOLEAN NOT NULL DEFAULT false,
    current_status  VARCHAR(30) NOT NULL,          -- 'active', 'expired', 'revoked', 'renewed'
    plus_license_id VARCHAR(255),
    max_uses        INTEGER,
    current_uses    INTEGER NOT NULL DEFAULT 0,
    events_count    INTEGER NOT NULL DEFAULT 0,    -- how many lifecycle events
    last_event_at   TIMESTAMPTZ
);

CREATE INDEX idx_rm_license_asset ON rm_license_timeline(asset_id);
CREATE INDEX idx_rm_license_expiry ON rm_license_timeline(license_expiry);
CREATE INDEX idx_rm_license_status ON rm_license_timeline(current_status);
```

### Compliance Dashboard Read Model

```sql
CREATE TABLE rm_compliance_summary (
    asset_id            UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    total_checks        INTEGER NOT NULL DEFAULT 0,
    violations          INTEGER NOT NULL DEFAULT 0,
    warnings            INTEGER NOT NULL DEFAULT 0,
    unresolved_count    INTEGER NOT NULL DEFAULT 0,
    last_check_at       TIMESTAMPTZ,
    license_status      VARCHAR(30),               -- 'valid', 'expiring_soon', 'expired', 'none'
    model_release_status VARCHAR(30),               -- 'all_obtained', 'pending', 'missing'
    brand_compliance    VARCHAR(30),                -- 'compliant', 'violations', 'unchecked'
    c2pa_status         VARCHAR(30),                -- 'verified', 'unverified', 'no_manifest'
    consent_status      VARCHAR(30),                -- 'all_consented', 'pending', 'withdrawn'
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_compliance_tenant ON rm_compliance_summary(tenant_id);
CREATE INDEX idx_rm_compliance_violations ON rm_compliance_summary(violations) WHERE violations > 0;
```

### Semantic Search Read Model

```sql
CREATE TABLE rm_asset_embeddings (
    asset_id        UUID NOT NULL,
    model_name      VARCHAR(100) NOT NULL,
    embedding_type  VARCHAR(30) NOT NULL,
    embedding       vector(1536),
    generated_at    TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (asset_id, model_name, embedding_type)
);

CREATE INDEX idx_rm_embeddings_vector ON rm_asset_embeddings
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
```

### Face Detection Read Model

```sql
CREATE TABLE rm_detected_faces (
    face_id         UUID PRIMARY KEY,
    asset_id        UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    person_id       UUID,
    person_name     VARCHAR(255),
    bounding_box    JSONB NOT NULL,
    confidence      NUMERIC(4,3) NOT NULL,
    consent_status  VARCHAR(30),                   -- denormalised from consent events
    detected_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_faces_asset ON rm_detected_faces(asset_id);
CREATE INDEX idx_rm_faces_person ON rm_detected_faces(person_id);
```

### Activity Feed Read Model

```sql
CREATE TABLE rm_activity_feed (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_name      VARCHAR(255),
    action_summary  TEXT NOT NULL,                 -- human-readable: "Alice uploaded banner-v3.png"
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    resource_title  VARCHAR(512),
    event_type      VARCHAR(100) NOT NULL,
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_rm_activity_tenant ON rm_activity_feed(tenant_id, occurred_at DESC);
CREATE INDEX idx_rm_activity_resource ON rm_activity_feed(resource_type, resource_id);
```

---

## Projection Infrastructure

```sql
-- Tracks which read models are up to date
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,      -- 'rm_assets', 'rm_license_timeline', etc.
    last_event_id   UUID NOT NULL,
    last_event_at   TIMESTAMPTZ NOT NULL,
    events_processed BIGINT NOT NULL DEFAULT 0,
    is_rebuilding   BOOLEAN NOT NULL DEFAULT false,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dead letter queue for failed projections
CREATE TABLE projection_failures (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    projection_name VARCHAR(100) NOT NULL,
    event_id        UUID NOT NULL REFERENCES events(event_id),
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    last_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_projection_failures_name ON projection_failures(projection_name);
```

---

## Command-Side Validation Tables

```sql
-- Minimal state needed for command validation (not for querying)
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    UNIQUE (tenant_id, email)
);

-- Optimistic concurrency: latest version per stream
CREATE TABLE stream_versions (
    stream_id       UUID PRIMARY KEY,
    stream_type     VARCHAR(50) NOT NULL,
    current_version INTEGER NOT NULL DEFAULT 0,
    tenant_id       UUID NOT NULL REFERENCES tenants(id)
);
```

---

## Example: Temporal Query

Reconstruct an asset's license status on a specific date:

```sql
-- "What was the license status for asset X on 2026-01-15?"
SELECT
    event_type,
    payload->>'license_type' AS license_type,
    payload->>'expiry_date' AS expiry_date,
    payload->>'current_status' AS status,
    created_at
FROM events
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'  -- asset ID
  AND event_type IN ('LicenseGranted', 'LicenseExpired', 'LicenseRenewed', 'LicenseRevoked')
  AND created_at <= '2026-01-15T23:59:59Z'
ORDER BY event_version DESC
LIMIT 1;
```

## Example: Event Replay for Read Model Rebuild

```sql
-- Rebuild rm_assets for a single asset
SELECT event_type, payload, metadata, created_at
FROM events
WHERE stream_id = '550e8400-e29b-41d4-a716-446655440000'
ORDER BY event_version ASC;

-- Application code replays each event through the projection handler:
-- AssetUploaded     -> INSERT INTO rm_assets
-- AssetMetadataUpdated -> UPDATE rm_assets SET iptc_headline = ...
-- AssetTagged       -> INSERT INTO rm_asset_tags; UPDATE rm_assets SET tag_count = tag_count + 1
-- AssetStatusChanged -> UPDATE rm_assets SET status = ...
```

## Example: Compliance Audit Report

```sql
-- "Show all actions taken on asset X in Q1 2026, for regulatory audit"
SELECT
    event_type,
    actor_id,
    u.display_name AS actor_name,
    payload,
    metadata->>'ip_address' AS ip_address,
    created_at
FROM events e
LEFT JOIN users u ON u.id = e.actor_id
WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.created_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY e.event_version ASC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | events (partitioned by month) |
| Read Models — Assets | 3 | rm_assets, rm_asset_tags, rm_asset_embeddings |
| Read Models — Rights | 1 | rm_license_timeline |
| Read Models — Compliance | 1 | rm_compliance_summary |
| Read Models — Faces | 1 | rm_detected_faces |
| Read Models — Activity | 1 | rm_activity_feed |
| Projection Infrastructure | 2 | projection_checkpoints, projection_failures |
| Command-Side Validation | 3 | tenants, users, stream_versions |
| **Total** | **~13** | Plus additional read models as needed |

---

## Key Design Decisions

1. **Single event store table** with a `stream_type` discriminator, rather than separate event tables per entity. This simplifies cross-entity queries (e.g., "all events for tenant X in the last hour") and enables unified audit reporting.

2. **Optimistic concurrency via `(stream_id, event_version)` unique constraint.** Two concurrent writes to the same aggregate will fail at the database level, preventing lost updates without distributed locking.

3. **Events carry full payload snapshots** for the changed fields, not just deltas. This makes each event self-describing and enables projection rebuilds without needing the prior event.

4. **Read models are disposable.** They carry no data that cannot be reconstructed from events. This allows schema changes to read models without data migration — just rebuild from the event store.

5. **Compliance is structural, not bolted on.** The event store naturally satisfies ISO 15489 records management requirements: every action has a timestamp, actor, and immutable record. No separate audit log is needed.

6. **C2PA provenance maps naturally to events.** The C2PA specification's "history of actions" model (assertions within claims within manifests) directly maps to the event stream's chronological record, creating a unified provenance chain.

7. **GDPR consent tracking via events** creates an auditable consent timeline: `BiometricConsentGranted` followed by `BiometricConsentWithdrawn` events, with the read model reflecting the current state.

8. **Projection checkpoints enable exactly-once processing** and allow read models to be rebuilt from any point. The dead letter queue captures failed projections for investigation without blocking the main event stream.

9. **Event type taxonomy uses domain language** (not CRUD verbs). `LicenseExpired` communicates business meaning; `UPDATE licenses SET status = 'expired'` does not.

10. **Minimal command-side state.** The `tenants`, `users`, and `stream_versions` tables exist only for command validation (does this user exist? is this the correct version?). All queryable state lives in read models.
