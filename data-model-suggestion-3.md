# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Brand Asset Management · Created: 2026-05-19

## Philosophy

This approach uses strongly-typed relational columns for core identity, relationships, and frequently-queried fields, while delegating variable, format-specific, and jurisdiction-dependent metadata to JSONB columns. The result is a significantly smaller table count than the fully normalised model, with comparable query power for the fields that matter most.

The design is inspired by how modern SaaS DAM platforms like Brandfolder and Canto actually work in practice: a small number of core tables with flexible metadata that varies per asset type, per tenant configuration, and per industry vertical. Rather than creating separate metadata tables for IPTC, Dublin Core, EXIF, and XMP, all metadata lives in a structured JSONB column on the asset, with well-defined JSON schemas validated at the application layer.

This is the fastest approach to iterate on during early development. New metadata fields, new compliance requirements, or new AI output formats can be added without DDL changes. The trade-off is that JSONB fields are harder to enforce at the database level and require GIN indexes for performant querying.

**Best for:** Teams prioritising development velocity, multi-format asset support with varying metadata schemas, and rapid feature iteration in an AI-native DAM MVP.

**Trade-offs:**
- Pro: Far fewer tables (~20 vs ~43); simpler schema to understand and maintain
- Pro: New metadata fields added without migrations — just update the JSON schema
- Pro: Format-specific metadata (EXIF for photos, codec info for video) naturally accommodated
- Pro: Tenant-specific custom fields trivially supported via JSONB
- Con: No database-level enforcement of JSONB structure — validation shifts to application code
- Con: JSONB queries are slower than column queries for complex predicates
- Con: Reporting and analytics require JSONB extraction, which can be verbose
- Con: Schema evolution in JSONB requires careful versioning to avoid ambiguity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IPTC Photo Metadata 2025.1 | Stored in `assets.metadata` under the `iptc` key; validated against JSON schema |
| XMP (ISO 16684) | Raw XMP packet preserved in `asset_files.embedded_metadata`; parsed fields merged into `assets.metadata` |
| Dublin Core (ISO 15836) | Mapped to top-level `assets.metadata` keys (`dc_title`, `dc_creator`, etc.) |
| EXIF 2.2 | Extracted to `assets.metadata.exif` sub-object on ingest |
| PLUS License Data Format | License terms stored in `asset_licenses.terms` JSONB with PLUS vocabulary keys |
| C2PA 2.4 | Manifest data in `asset_provenance.c2pa_manifest` JSONB; raw manifest in binary column |
| ISO 15489 | Retention rules in `tenants.settings`; audit log with JSONB `details` |
| ISO 3166 | Jurisdiction codes in `assets.metadata.iptc.country_code` |

---

## Core Tables

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings example:
    -- {
    --   "retention_days": 365,
    --   "allowed_file_types": ["image/*", "video/*", "application/pdf"],
    --   "max_file_size_mb": 500,
    --   "custom_metadata_schema": { ... },
    --   "brand_compliance_rules": { ... },
    --   "gdpr_biometric_enabled": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',  -- 'admin', 'manager', 'editor', 'viewer', 'external'
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- permissions example:
    -- ["asset.upload", "asset.approve", "brand.manage", "collection.create"]
    auth_config     JSONB NOT NULL DEFAULT '{}',
    -- auth_config example:
    -- {
    --   "provider": "oidc",
    --   "subject": "auth0|abc123",
    --   "mfa_enabled": true
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role ON users(role);
```

---

## Assets (Central Table with JSONB Metadata)

```sql
CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    brand_id        UUID REFERENCES brands(id),
    asset_type      VARCHAR(50) NOT NULL,          -- 'image', 'video', 'document', 'audio', '3d'
    title           VARCHAR(512) NOT NULL,
    description     TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',

    -- Current version info (denormalised for fast reads)
    current_version INTEGER NOT NULL DEFAULT 1,
    file_name       VARCHAR(512) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    mime_type       VARCHAR(255) NOT NULL,
    storage_path    TEXT NOT NULL,
    checksum_sha256 CHAR(64) NOT NULL,

    -- Technical properties (type-dependent)
    dimensions      JSONB,
    -- For images: {"width": 3840, "height": 2160, "dpi": 300, "colour_space": "sRGB"}
    -- For video:  {"width": 1920, "height": 1080, "duration_secs": 125.4, "fps": 30, "codec": "h264"}
    -- For audio:  {"duration_secs": 245.0, "sample_rate": 44100, "channels": 2, "bitrate": 320000}
    -- For documents: {"page_count": 12, "word_count": 5400}

    -- All metadata (IPTC, Dublin Core, EXIF, custom) in one JSONB column
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "iptc": {
    --     "headline": "Summer Campaign Hero Shot",
    --     "caption": "Model wearing...",
    --     "keywords": ["summer", "fashion", "outdoor"],
    --     "creator": "Jane Doe",
    --     "copyright": "(c) 2026 Acme Corp",
    --     "country_code": "USA",
    --     "model_release_status": "MR-LPR",
    --     "ai_system_used": "DALL-E 3",
    --     "ai_prompt_info": "Generate a summer fashion..."
    --   },
    --   "dublin_core": {
    --     "creator": "Jane Doe",
    --     "type": "Image",
    --     "language": "en"
    --   },
    --   "exif": {
    --     "camera_make": "Canon",
    --     "camera_model": "EOS R5",
    --     "focal_length": 85.0,
    --     "aperture": "f/1.4",
    --     "iso": 200,
    --     "gps": {"lat": 40.7128, "lng": -74.0060}
    --   },
    --   "custom": {
    --     "campaign": "Summer 2026",
    --     "region": "North America",
    --     "product_line": "Activewear"
    --   }
    -- }

    -- AI processing state
    ai_processing   JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "auto_tagged": true,
    --   "tagged_at": "2026-05-19T10:30:00Z",
    --   "model": "clip-vit-large-patch14",
    --   "is_ai_generated": false,
    --   "faces_detected": 2,
    --   "embedding_generated": true,
    --   "compliance_checked": true,
    --   "last_compliance_check": "2026-05-19T11:00:00Z"
    -- }

    -- Ownership
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,

    -- Full-text search vector
    search_vector   tsvector,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core indexes
CREATE INDEX idx_assets_tenant ON assets(tenant_id);
CREATE INDEX idx_assets_brand ON assets(brand_id);
CREATE INDEX idx_assets_status ON assets(status);
CREATE INDEX idx_assets_type ON assets(asset_type);
CREATE INDEX idx_assets_uploaded_by ON assets(uploaded_by);

-- JSONB indexes for metadata querying
CREATE INDEX idx_assets_metadata ON assets USING GIN(metadata);
CREATE INDEX idx_assets_iptc_keywords ON assets USING GIN((metadata->'iptc'->'keywords'));
CREATE INDEX idx_assets_iptc_country ON assets((metadata->'iptc'->>'country_code'));
CREATE INDEX idx_assets_ai ON assets USING GIN(ai_processing);

-- Full-text search
CREATE INDEX idx_assets_search ON assets USING GIN(search_vector);

-- Trigger to update search vector
-- CREATE TRIGGER assets_search_update BEFORE INSERT OR UPDATE ON assets
-- FOR EACH ROW EXECUTE FUNCTION
-- tsvector_update_trigger(search_vector, 'pg_catalog.english',
--   title, description);
```

---

## Asset Versions

```sql
CREATE TABLE asset_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    file_name       VARCHAR(512) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    mime_type       VARCHAR(255) NOT NULL,
    storage_path    TEXT NOT NULL,
    checksum_sha256 CHAR(64) NOT NULL,
    dimensions      JSONB,
    embedded_metadata JSONB,                       -- raw extracted XMP/IPTC/EXIF before merging
    change_notes    TEXT,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (asset_id, version_number)
);

CREATE INDEX idx_asset_versions_asset ON asset_versions(asset_id);
```

---

## Tags & Collections

```sql
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    tag_type        VARCHAR(50) NOT NULL DEFAULT 'keyword',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name, tag_type)
);

CREATE TABLE asset_tags (
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    source          VARCHAR(30) NOT NULL DEFAULT 'manual',
    confidence      NUMERIC(4,3),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (asset_id, tag_id)
);

CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    collection_type VARCHAR(30) NOT NULL DEFAULT 'manual',
    smart_filter    JSONB,
    -- Smart filter example:
    -- {
    --   "asset_type": ["image"],
    --   "metadata.iptc.country_code": "USA",
    --   "tags": ["summer", "outdoor"],
    --   "status": "approved",
    --   "uploaded_after": "2026-01-01"
    -- }
    cover_asset_id  UUID REFERENCES assets(id),
    created_by      UUID NOT NULL REFERENCES users(id),
    is_public       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE collection_assets (
    collection_id   UUID NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    added_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (collection_id, asset_id)
);

CREATE INDEX idx_tags_tenant ON tags(tenant_id);
CREATE INDEX idx_collections_tenant ON collections(tenant_id);
```

---

## Brands

```sql
CREATE TABLE brands (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    parent_brand_id UUID REFERENCES brands(id),
    is_active       BOOLEAN NOT NULL DEFAULT true,

    -- Brand identity as JSONB (flexible per-brand)
    guidelines      JSONB NOT NULL DEFAULT '{}',
    -- {
    --   "colours": [
    --     {"name": "Primary Blue", "hex": "#0066CC", "pantone": "286 C", "context": "primary"},
    --     {"name": "Accent Orange", "hex": "#FF6600", "context": "accent"}
    --   ],
    --   "fonts": [
    --     {"family": "Inter", "weight": "400", "context": "body"},
    --     {"family": "Inter", "weight": "700", "context": "heading"}
    --   ],
    --   "logos": [
    --     {"variant": "full", "asset_id": "uuid-here", "min_size_px": 120, "clear_space_px": 20},
    --     {"variant": "icon", "asset_id": "uuid-here", "min_size_px": 32}
    --   ],
    --   "tone_of_voice": "Professional but approachable...",
    --   "dos_and_donts": ["Always use approved colour palette", "Never stretch the logo"]
    -- }

    -- Published guideline pages
    guideline_pages JSONB NOT NULL DEFAULT '[]',
    -- [
    --   {"title": "Logo Usage", "content": "...", "order": 1, "published": true},
    --   {"title": "Typography", "content": "...", "order": 2, "published": true}
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE INDEX idx_brands_tenant ON brands(tenant_id);
```

---

## Rights & Licensing

```sql
CREATE TABLE asset_licenses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    license_type    VARCHAR(50) NOT NULL,          -- 'royalty_free', 'rights_managed', 'creative_commons', 'custom'
    status          VARCHAR(30) NOT NULL DEFAULT 'active',

    -- All license terms in JSONB (accommodates PLUS, CC, custom)
    terms           JSONB NOT NULL DEFAULT '{}',
    -- PLUS example:
    -- {
    --   "plus_license_id": "PLUS-12345",
    --   "licensor": {"name": "Getty Images", "id": "getty-001"},
    --   "licensee": {"name": "Acme Corp"},
    --   "usage_type": "Advertising",
    --   "usage_media": "Digital",
    --   "territory": "WW",
    --   "exclusivity": false,
    --   "max_impressions": 1000000
    -- }
    -- Creative Commons example:
    -- {
    --   "cc_code": "CC-BY-SA-4.0",
    --   "cc_url": "https://creativecommons.org/licenses/by-sa/4.0/",
    --   "attribution_text": "Photo by Jane Doe"
    -- }

    license_start   DATE,
    license_expiry  DATE,
    is_perpetual    BOOLEAN NOT NULL DEFAULT false,
    max_uses        INTEGER,
    current_uses    INTEGER NOT NULL DEFAULT 0,
    alert_days      INTEGER DEFAULT 30,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE releases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    release_type    VARCHAR(30) NOT NULL,          -- 'model', 'property'
    subject_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(30) NOT NULL,          -- 'obtained', 'pending', 'not_required', 'denied'
    release_file_path TEXT,
    details         JSONB NOT NULL DEFAULT '{}',
    -- {"signed_date": "2026-01-15", "expiry_date": "2027-01-15", "notes": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_licenses_asset ON asset_licenses(asset_id);
CREATE INDEX idx_licenses_expiry ON asset_licenses(license_expiry);
CREATE INDEX idx_licenses_status ON asset_licenses(status);
CREATE INDEX idx_releases_asset ON releases(asset_id);
```

---

## Content Provenance

```sql
CREATE TABLE asset_provenance (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    provenance_type VARCHAR(30) NOT NULL,          -- 'c2pa', 'watermark', 'blockchain'

    -- C2PA manifest data
    c2pa_manifest   JSONB,
    -- {
    --   "manifest_label": "urn:uuid:abc-123",
    --   "claim_generator": "BrandDAM/1.0",
    --   "signature_algorithm": "ES256",
    --   "is_valid": true,
    --   "validated_at": "2026-05-19T10:00:00Z",
    --   "assertions": [
    --     {"label": "c2pa.actions", "data": {"actions": [{"action": "c2pa.created"}]}},
    --     {"label": "stds.exif", "data": {"EXIF:Make": "Canon"}},
    --     {"label": "c2pa.ai_training", "data": {"use": "notAllowed"}}
    --   ]
    -- }

    raw_manifest    BYTEA,                         -- full CBOR binary
    signing_cert    TEXT,                          -- PEM certificate chain

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_provenance_asset ON asset_provenance(asset_id);
CREATE INDEX idx_provenance_type ON asset_provenance(provenance_type);
```

---

## AI & Search

```sql
CREATE TABLE asset_embeddings (
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    model_name      VARCHAR(100) NOT NULL,
    embedding_type  VARCHAR(30) NOT NULL,
    embedding       vector(1536),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (asset_id, model_name, embedding_type)
);

CREATE TABLE detected_faces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    person_id       UUID,
    person_name     VARCHAR(255),
    bounding_box    JSONB NOT NULL,
    confidence      NUMERIC(4,3) NOT NULL,
    face_encoding   vector(128),
    consent_status  VARCHAR(30) NOT NULL DEFAULT 'pending',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE compliance_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    check_type      VARCHAR(50) NOT NULL,
    severity        VARCHAR(20) NOT NULL,
    message         TEXT NOT NULL,
    details         JSONB,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_vector ON asset_embeddings
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_faces_asset ON detected_faces(asset_id);
CREATE INDEX idx_compliance_asset ON compliance_checks(asset_id);
```

---

## Workflows

```sql
CREATE TABLE workflows (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    workflow_type   VARCHAR(50) NOT NULL,          -- 'approval', 'review', 'publish'
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',
    config          JSONB NOT NULL DEFAULT '{}',   -- workflow template / step definitions
    current_step    INTEGER NOT NULL DEFAULT 0,
    initiated_by    UUID NOT NULL REFERENCES users(id),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workflow_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id     UUID NOT NULL REFERENCES workflows(id) ON DELETE CASCADE,
    step_number     INTEGER NOT NULL,
    action          VARCHAR(30) NOT NULL,
    actor_id        UUID NOT NULL REFERENCES users(id),
    comment         TEXT,
    acted_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_workflows_tenant ON workflows(tenant_id);
CREATE INDEX idx_workflows_asset ON workflows(asset_id);
CREATE INDEX idx_workflows_status ON workflows(status);
```

---

## Distribution & Sharing

```sql
CREATE TABLE share_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    token           VARCHAR(64) NOT NULL UNIQUE,
    target_type     VARCHAR(30) NOT NULL,          -- 'asset', 'collection'
    target_id       UUID NOT NULL,
    config          JSONB NOT NULL DEFAULT '{}',
    -- {"password_protected": true, "max_downloads": 10, "allowed_formats": ["jpg", "png"]}
    created_by      UUID NOT NULL REFERENCES users(id),
    download_count  INTEGER NOT NULL DEFAULT 0,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_share_links_token ON share_links(token);
```

---

## Audit Log

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    actor_type      VARCHAR(30) NOT NULL DEFAULT 'user',
    action          VARCHAR(50) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB,
    -- {"old_status": "draft", "new_status": "approved", "ip": "192.168.1.1"}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_action ON audit_log(action);
```

---

## Example: Querying JSONB Metadata

```sql
-- Find all approved images from USA with "summer" keyword
SELECT id, title, metadata->'iptc'->>'headline' AS headline
FROM assets
WHERE tenant_id = 'tenant-uuid'
  AND status = 'approved'
  AND asset_type = 'image'
  AND metadata->'iptc'->>'country_code' = 'USA'
  AND metadata->'iptc'->'keywords' ? 'summer';

-- Find all AI-generated assets
SELECT id, title, metadata->'iptc'->>'ai_system_used' AS ai_system
FROM assets
WHERE tenant_id = 'tenant-uuid'
  AND (ai_processing->>'is_ai_generated')::boolean = true;

-- Custom metadata query (tenant-specific fields)
SELECT id, title, metadata->'custom'->>'campaign' AS campaign
FROM assets
WHERE tenant_id = 'tenant-uuid'
  AND metadata->'custom'->>'campaign' = 'Summer 2026';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity | 2 | tenants, users |
| Assets | 2 | assets, asset_versions |
| Brands | 1 | brands (guidelines in JSONB) |
| Taxonomy | 4 | tags, asset_tags, collections, collection_assets |
| Rights & Licensing | 2 | asset_licenses, releases |
| Content Provenance | 1 | asset_provenance |
| AI & Search | 3 | asset_embeddings, detected_faces, compliance_checks |
| Workflows | 2 | workflows, workflow_actions |
| Distribution | 1 | share_links |
| Audit | 1 | audit_log |
| **Total** | **~19** | |

---

## Key Design Decisions

1. **Single `metadata` JSONB column** on assets replaces 4 separate metadata tables (IPTC, Dublin Core, EXIF, XMP). This cuts the table count dramatically and allows format-specific metadata to coexist without schema changes.

2. **GIN indexes on JSONB paths** for the most-queried metadata fields (IPTC keywords, country code). Not every nested field needs an index — only the ones used in filters and faceted search.

3. **Brand guidelines as JSONB** rather than separate `brand_colours`, `brand_fonts`, `brand_guidelines` tables. Brand identity varies widely between organisations; JSONB accommodates this without DDL changes.

4. **Unified `releases` table** for both model and property releases, using a `release_type` discriminator. This avoids two nearly-identical tables with the same structure.

5. **License `terms` as JSONB** accommodating PLUS vocabulary, Creative Commons fields, and custom license terms in one flexible column. The relational fields (`license_start`, `license_expiry`, `status`) remain typed columns for efficient querying.

6. **`dimensions` JSONB on assets** instead of nullable `width_px`, `height_px`, `duration_secs` columns. Different asset types have different technical properties; JSONB handles this naturally.

7. **`ai_processing` JSONB** tracks AI pipeline state (tagged, embedded, compliance-checked) without a separate processing queue table. This is a pragmatic simplification for MVP; a dedicated job queue may be added later.

8. **Tenant `settings` JSONB** stores configuration like retention policies, file type restrictions, and custom metadata schemas, avoiding a separate settings table per configuration type.

9. **C2PA manifest stored as JSONB + raw CBOR binary.** The parsed JSONB enables queries on assertion content; the raw binary enables cryptographic re-verification.

10. **~19 tables vs ~43 in the normalised model** — roughly half the schema complexity, with the trade-off that JSONB validation must happen in application code rather than database constraints.
