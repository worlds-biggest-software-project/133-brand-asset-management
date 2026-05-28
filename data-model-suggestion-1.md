# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Brand Asset Management · Created: 2026-05-19

## Philosophy

This approach models every domain concept as a dedicated table with strict foreign key relationships, normalised to Third Normal Form (3NF). Metadata standards (IPTC, XMP, PLUS, Dublin Core) are mapped to explicit columns rather than embedded in flexible structures, ensuring compile-time schema validation and strong referential integrity.

The design follows the principles used by enterprise DAM platforms like Bynder and IntelligenceBank, where assets, collections, permissions, and rights are first-class entities with well-defined relationships. Every field that matters for search, filtering, or compliance has its own indexed column.

This is the most traditional approach and the easiest to reason about for teams familiar with relational databases. It produces the largest number of tables but offers the strongest guarantees about data consistency and query predictability.

**Best for:** Teams prioritising data integrity, complex cross-entity reporting, and regulatory compliance in a single-tenant or small-multi-tenant deployment.

**Trade-offs:**
- Pro: Strong referential integrity; no orphaned data; standard SQL tooling works perfectly
- Pro: Every field is queryable and indexable without special operators
- Pro: Schema changes are explicit and reviewable via migrations
- Con: High table count increases schema complexity and migration burden
- Con: Adding new metadata fields requires DDL changes and redeployment
- Con: Jurisdiction-specific or format-specific fields lead to sparse columns or excessive subtables
- Con: Slower iteration velocity compared to JSONB-based approaches

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| IPTC Photo Metadata 2025.1 | Core and Extension fields mapped to columns in `asset_metadata_iptc` |
| XMP (ISO 16684) | XMP namespace fields stored in `asset_metadata_xmp`; raw XMP preserved in `asset_files` |
| Dublin Core (ISO 15836) | 15 Dublin Core elements mapped to `asset_metadata_dublin_core` |
| PLUS License Data Format | License fields (`plus_license_id`, `licensor_name`, `usage_type`) in `asset_licenses` |
| C2PA 2.4 | Manifest, claim, and assertion data in `asset_c2pa_manifests` and `asset_c2pa_assertions` |
| ISO 15489 | Retention policies and disposal schedules in `retention_policies`; audit trail in `audit_log` |
| ISO 3166-1/2 | Jurisdiction codes in `jurisdictions` reference table |
| OAuth 2.0 / OIDC | API credentials and identity provider config in `oauth_clients` and `identity_providers` |

---

## Core Identity & Multi-Tenancy

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    plan            VARCHAR(50) NOT NULL DEFAULT 'free',
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    auth_provider   VARCHAR(50) NOT NULL DEFAULT 'local',  -- 'local', 'oidc', 'saml'
    auth_subject    VARCHAR(512),                           -- external IdP subject ID
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system       BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(100) NOT NULL UNIQUE,  -- e.g. 'asset.upload', 'asset.delete', 'brand.manage'
    description     TEXT
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

CREATE INDEX idx_users_tenant ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_roles_tenant ON roles(tenant_id);
```

---

## Brand Management

```sql
CREATE TABLE brands (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    description     TEXT,
    parent_brand_id UUID REFERENCES brands(id),  -- sub-brand hierarchy
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE TABLE brand_guidelines (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_id        UUID NOT NULL REFERENCES brands(id) ON DELETE CASCADE,
    title           VARCHAR(255) NOT NULL,
    content         TEXT,                          -- rich text / markdown
    section_order   INTEGER NOT NULL DEFAULT 0,
    is_published    BOOLEAN NOT NULL DEFAULT false,
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE brand_colours (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_id        UUID NOT NULL REFERENCES brands(id) ON DELETE CASCADE,
    name            VARCHAR(100) NOT NULL,
    hex_value       CHAR(7) NOT NULL,              -- e.g. '#FF5733'
    rgb_r           SMALLINT,
    rgb_g           SMALLINT,
    rgb_b           SMALLINT,
    cmyk_c          SMALLINT,
    cmyk_m          SMALLINT,
    cmyk_y          SMALLINT,
    cmyk_k          SMALLINT,
    pantone_code    VARCHAR(20),
    usage_context   VARCHAR(100),                  -- 'primary', 'secondary', 'accent'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE brand_fonts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_id        UUID NOT NULL REFERENCES brands(id) ON DELETE CASCADE,
    font_family     VARCHAR(255) NOT NULL,
    font_weight     VARCHAR(50),
    usage_context   VARCHAR(100),                  -- 'heading', 'body', 'caption'
    font_file_url   TEXT,
    license_info    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE brand_logos (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    brand_id        UUID NOT NULL REFERENCES brands(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    variant         VARCHAR(100),                  -- 'full', 'icon', 'monochrome', 'reversed'
    asset_id        UUID,                          -- references assets(id), added after assets table
    min_clear_space INTEGER,                       -- pixels
    min_size_px     INTEGER,
    usage_notes     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_brands_tenant ON brands(tenant_id);
CREATE INDEX idx_brand_guidelines_brand ON brand_guidelines(brand_id);
```

---

## Asset Management

```sql
CREATE TABLE asset_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,   -- 'image', 'video', 'document', '3d', 'audio'
    label           VARCHAR(100) NOT NULL,
    allowed_extensions TEXT[]                       -- '{jpg,jpeg,png,gif,webp,svg}'
);

CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    brand_id        UUID REFERENCES brands(id),
    asset_type_id   UUID NOT NULL REFERENCES asset_types(id),
    title           VARCHAR(512) NOT NULL,
    description     TEXT,
    slug            VARCHAR(255),
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',  -- 'draft','pending_review','approved','archived','expired'
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    archived_at     TIMESTAMPTZ,
    is_ai_generated BOOLEAN NOT NULL DEFAULT false,
    ai_system_used  VARCHAR(255),                  -- IPTC 2025.1: AI System Used
    ai_prompt       TEXT,                          -- IPTC 2025.1: AI Prompt Information
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE asset_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    version_number  INTEGER NOT NULL,
    file_name       VARCHAR(512) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    mime_type       VARCHAR(255) NOT NULL,
    storage_path    TEXT NOT NULL,                  -- S3/GCS path
    storage_bucket  VARCHAR(255) NOT NULL,
    checksum_sha256 CHAR(64) NOT NULL,
    width_px        INTEGER,
    height_px       INTEGER,
    duration_secs   NUMERIC(10,2),                 -- for video/audio
    colour_space    VARCHAR(50),                   -- 'sRGB', 'Adobe RGB', 'CMYK'
    dpi             INTEGER,
    is_current      BOOLEAN NOT NULL DEFAULT true,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    change_notes    TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (asset_id, version_number)
);

CREATE TABLE asset_thumbnails (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_version_id UUID NOT NULL REFERENCES asset_versions(id) ON DELETE CASCADE,
    size_label      VARCHAR(20) NOT NULL,          -- 'small', 'medium', 'large', 'preview'
    width_px        INTEGER NOT NULL,
    height_px       INTEGER NOT NULL,
    storage_path    TEXT NOT NULL,
    mime_type       VARCHAR(100) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Add FK from brand_logos to assets
ALTER TABLE brand_logos ADD CONSTRAINT fk_brand_logos_asset
    FOREIGN KEY (asset_id) REFERENCES assets(id);

CREATE INDEX idx_assets_tenant ON assets(tenant_id);
CREATE INDEX idx_assets_brand ON assets(brand_id);
CREATE INDEX idx_assets_status ON assets(status);
CREATE INDEX idx_assets_type ON assets(asset_type_id);
CREATE INDEX idx_assets_uploaded_by ON assets(uploaded_by);
CREATE INDEX idx_asset_versions_asset ON asset_versions(asset_id);
CREATE INDEX idx_asset_versions_current ON asset_versions(asset_id) WHERE is_current = true;
```

---

## Metadata (Standards-Aligned)

```sql
-- IPTC Core + Extension metadata (explicit columns per IPTC 2025.1)
CREATE TABLE asset_metadata_iptc (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id            UUID NOT NULL UNIQUE REFERENCES assets(id) ON DELETE CASCADE,
    -- IPTC Core
    headline            VARCHAR(256),
    caption_abstract    TEXT,
    keywords            TEXT[],
    creator_name        VARCHAR(255),
    creator_job_title   VARCHAR(255),
    credit_line         VARCHAR(255),
    source              VARCHAR(255),
    copyright_notice    VARCHAR(500),
    rights_usage_terms  TEXT,
    date_created        DATE,
    city                VARCHAR(255),
    province_state      VARCHAR(255),
    country_name        VARCHAR(255),
    country_code        CHAR(3),                   -- ISO 3166-1 alpha-3
    -- IPTC Extension
    location_shown      TEXT[],
    model_age           INTEGER[],
    model_release_status VARCHAR(50),               -- 'MR-NON', 'MR-NAP', 'MR-UPR', 'MR-LPR'
    property_release_status VARCHAR(50),
    -- IPTC 2025.1 AI fields
    ai_prompt_info      TEXT,
    ai_prompt_writer    VARCHAR(255),
    ai_system_used      VARCHAR(255),
    ai_system_version   VARCHAR(100),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Dublin Core metadata (ISO 15836)
CREATE TABLE asset_metadata_dublin_core (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL UNIQUE REFERENCES assets(id) ON DELETE CASCADE,
    dc_title        VARCHAR(512),
    dc_creator      VARCHAR(255),
    dc_subject      TEXT[],
    dc_description  TEXT,
    dc_publisher    VARCHAR(255),
    dc_contributor  VARCHAR(255),
    dc_date         DATE,
    dc_type         VARCHAR(100),                  -- 'Image', 'MovingImage', 'Text', 'Sound'
    dc_format       VARCHAR(100),                  -- MIME type
    dc_identifier   VARCHAR(500),                  -- URI or unique ID
    dc_source       VARCHAR(500),
    dc_language     VARCHAR(10),                   -- ISO 639-1
    dc_relation     VARCHAR(500),
    dc_coverage     VARCHAR(255),
    dc_rights       TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- XMP raw embedded metadata
CREATE TABLE asset_metadata_xmp (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL UNIQUE REFERENCES assets(id) ON DELETE CASCADE,
    xmp_namespace   VARCHAR(255) NOT NULL,
    raw_xmp_xml     TEXT,                          -- full XMP packet as extracted
    extracted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- EXIF technical metadata
CREATE TABLE asset_metadata_exif (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL UNIQUE REFERENCES assets(id) ON DELETE CASCADE,
    camera_make     VARCHAR(100),
    camera_model    VARCHAR(100),
    lens_model      VARCHAR(200),
    focal_length_mm NUMERIC(6,1),
    aperture        VARCHAR(20),
    shutter_speed   VARCHAR(20),
    iso_speed       INTEGER,
    flash_fired     BOOLEAN,
    orientation     SMALLINT,
    gps_latitude    NUMERIC(10,7),
    gps_longitude   NUMERIC(10,7),
    gps_altitude    NUMERIC(8,2),
    capture_date    TIMESTAMPTZ,
    software        VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_iptc_keywords ON asset_metadata_iptc USING GIN(keywords);
CREATE INDEX idx_iptc_country ON asset_metadata_iptc(country_code);
CREATE INDEX idx_dc_subject ON asset_metadata_dublin_core USING GIN(dc_subject);
```

---

## Taxonomy & Organisation

```sql
CREATE TABLE categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    parent_id       UUID REFERENCES categories(id),
    depth           INTEGER NOT NULL DEFAULT 0,
    sort_order      INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    tag_type        VARCHAR(50) NOT NULL DEFAULT 'keyword',  -- 'keyword', 'campaign', 'season', 'ai_generated'
    confidence      NUMERIC(4,3),                  -- AI confidence score (0.000-1.000)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

CREATE TABLE asset_categories (
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    category_id     UUID NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (asset_id, category_id)
);

CREATE TABLE asset_tags (
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    tag_id          UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    source          VARCHAR(30) NOT NULL DEFAULT 'manual',  -- 'manual', 'ai_vision', 'ai_nlp', 'imported'
    confidence      NUMERIC(4,3),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (asset_id, tag_id)
);

CREATE TABLE collections (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    collection_type VARCHAR(30) NOT NULL DEFAULT 'manual',  -- 'manual', 'smart'
    smart_filter    JSONB,                         -- for smart collections: filter criteria
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

CREATE INDEX idx_categories_tenant ON categories(tenant_id);
CREATE INDEX idx_categories_parent ON categories(parent_id);
CREATE INDEX idx_tags_tenant ON tags(tenant_id);
CREATE INDEX idx_tags_type ON tags(tag_type);
CREATE INDEX idx_collections_tenant ON collections(tenant_id);
```

---

## Rights & Licensing

```sql
CREATE TABLE license_types (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    code            VARCHAR(50) NOT NULL UNIQUE,   -- 'royalty_free', 'rights_managed', 'creative_commons', 'editorial', 'custom'
    label           VARCHAR(100) NOT NULL,
    description     TEXT
);

CREATE TABLE asset_licenses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    license_type_id UUID NOT NULL REFERENCES license_types(id),
    -- PLUS License Data Format fields
    plus_license_id VARCHAR(255),                  -- PLUS registry ID
    licensor_name   VARCHAR(255),
    licensor_id     VARCHAR(255),
    licensee_name   VARCHAR(255),
    usage_type      VARCHAR(100),                  -- PLUS usage type vocabulary
    usage_media     VARCHAR(100),                  -- PLUS media type
    usage_territory VARCHAR(10),                   -- ISO 3166-1 alpha-2 or 'WW' for worldwide
    -- Dates
    license_start   DATE,
    license_expiry  DATE,
    is_perpetual    BOOLEAN NOT NULL DEFAULT false,
    -- Creative Commons
    cc_license_url  TEXT,                          -- e.g. 'https://creativecommons.org/licenses/by/4.0/'
    cc_license_code VARCHAR(20),                   -- 'CC-BY-4.0', 'CC-BY-SA-4.0'
    -- Tracking
    max_uses        INTEGER,                       -- null = unlimited
    current_uses    INTEGER NOT NULL DEFAULT 0,
    alert_days_before_expiry INTEGER DEFAULT 30,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE model_releases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    person_name     VARCHAR(255) NOT NULL,
    release_status  VARCHAR(30) NOT NULL,          -- 'obtained', 'pending', 'not_required', 'denied'
    release_file_id UUID,                          -- reference to uploaded release document
    signed_date     DATE,
    expiry_date     DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE property_releases (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    property_name   VARCHAR(255) NOT NULL,
    release_status  VARCHAR(30) NOT NULL,
    release_file_id UUID,
    signed_date     DATE,
    expiry_date     DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_asset_licenses_asset ON asset_licenses(asset_id);
CREATE INDEX idx_asset_licenses_expiry ON asset_licenses(license_expiry);
CREATE INDEX idx_model_releases_asset ON model_releases(asset_id);
CREATE INDEX idx_model_releases_status ON model_releases(release_status);
```

---

## Content Provenance (C2PA)

```sql
CREATE TABLE asset_c2pa_manifests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    manifest_label  VARCHAR(255) NOT NULL,         -- C2PA manifest label (URN)
    claim_generator VARCHAR(255),                  -- tool that created the claim
    claim_signature BYTEA,                         -- digital signature bytes
    signing_cert    TEXT,                          -- X.509 certificate chain (PEM)
    signature_alg   VARCHAR(50),                   -- 'ES256', 'ES384', 'PS256'
    is_valid        BOOLEAN,
    validated_at    TIMESTAMPTZ,
    raw_manifest    BYTEA,                         -- full CBOR-encoded manifest
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE asset_c2pa_assertions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    manifest_id     UUID NOT NULL REFERENCES asset_c2pa_manifests(id) ON DELETE CASCADE,
    assertion_label VARCHAR(255) NOT NULL,         -- e.g. 'c2pa.actions', 'stds.exif'
    assertion_kind  VARCHAR(50) NOT NULL,          -- 'cbor', 'json', 'binary'
    assertion_data  JSONB,                         -- parsed assertion content
    hash_alg        VARCHAR(20),                   -- 'sha256', 'sha384'
    hash_value      VARCHAR(128),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_c2pa_manifests_asset ON asset_c2pa_manifests(asset_id);
CREATE INDEX idx_c2pa_assertions_manifest ON asset_c2pa_assertions(manifest_id);
```

---

## Workflows & Approvals

```sql
CREATE TABLE workflow_templates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    steps           JSONB NOT NULL,                -- ordered list of approval steps
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workflow_instances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    template_id     UUID NOT NULL REFERENCES workflow_templates(id),
    asset_id        UUID NOT NULL REFERENCES assets(id),
    status          VARCHAR(30) NOT NULL DEFAULT 'pending',  -- 'pending', 'in_progress', 'approved', 'rejected', 'cancelled'
    current_step    INTEGER NOT NULL DEFAULT 0,
    initiated_by    UUID NOT NULL REFERENCES users(id),
    completed_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workflow_step_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instance_id     UUID NOT NULL REFERENCES workflow_instances(id) ON DELETE CASCADE,
    step_number     INTEGER NOT NULL,
    action          VARCHAR(30) NOT NULL,          -- 'approve', 'reject', 'request_changes', 'skip'
    actor_id        UUID NOT NULL REFERENCES users(id),
    comment         TEXT,
    acted_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_workflow_instances_asset ON workflow_instances(asset_id);
CREATE INDEX idx_workflow_instances_status ON workflow_instances(status);
```

---

## Search & AI

```sql
-- AI-generated embeddings for semantic search
CREATE TABLE asset_embeddings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    model_name      VARCHAR(100) NOT NULL,         -- e.g. 'clip-vit-large', 'text-embedding-3-large'
    model_version   VARCHAR(50),
    embedding_type  VARCHAR(30) NOT NULL,          -- 'visual', 'textual', 'combined'
    embedding       vector(1536),                  -- pgvector; dimension depends on model
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (asset_id, model_name, embedding_type)
);

-- Facial recognition results
CREATE TABLE detected_faces (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    person_id       UUID,                          -- references known_persons if identified
    bounding_box    JSONB NOT NULL,                -- {x, y, width, height} normalised 0-1
    confidence      NUMERIC(4,3) NOT NULL,
    face_encoding   vector(128),                   -- face embedding for matching
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE known_persons (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    reference_face  vector(128),                   -- canonical face embedding
    consent_status  VARCHAR(30) NOT NULL DEFAULT 'pending',  -- GDPR: 'consented', 'pending', 'withdrawn'
    consent_date    DATE,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Brand compliance scan results
CREATE TABLE compliance_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    check_type      VARCHAR(50) NOT NULL,          -- 'colour', 'typography', 'logo', 'content', 'rights_expiry'
    severity        VARCHAR(20) NOT NULL,          -- 'info', 'warning', 'violation'
    message         TEXT NOT NULL,
    details         JSONB,
    is_resolved     BOOLEAN NOT NULL DEFAULT false,
    resolved_by     UUID REFERENCES users(id),
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_embeddings_asset ON asset_embeddings(asset_id);
CREATE INDEX idx_embeddings_vector ON asset_embeddings USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);
CREATE INDEX idx_detected_faces_asset ON detected_faces(asset_id);
CREATE INDEX idx_detected_faces_person ON detected_faces(person_id);
CREATE INDEX idx_known_persons_tenant ON known_persons(tenant_id);
CREATE INDEX idx_compliance_checks_asset ON compliance_checks(asset_id);
CREATE INDEX idx_compliance_checks_type ON compliance_checks(check_type);
```

---

## Distribution & CDN

```sql
CREATE TABLE distribution_channels (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    channel_type    VARCHAR(50) NOT NULL,          -- 'cdn', 'portal', 'api', 'social'
    config          JSONB NOT NULL DEFAULT '{}',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE asset_distributions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    asset_version_id UUID NOT NULL REFERENCES asset_versions(id),
    channel_id      UUID NOT NULL REFERENCES distribution_channels(id),
    public_url      TEXT,
    cdn_cache_key   VARCHAR(255),
    variant_spec    JSONB,                         -- {width, height, format, quality}
    distributed_by  UUID NOT NULL REFERENCES users(id),
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE share_links (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    token           VARCHAR(64) NOT NULL UNIQUE,
    link_type       VARCHAR(30) NOT NULL,          -- 'asset', 'collection', 'portal'
    target_id       UUID NOT NULL,                 -- asset_id or collection_id
    created_by      UUID NOT NULL REFERENCES users(id),
    password_hash   VARCHAR(255),
    max_downloads   INTEGER,
    download_count  INTEGER NOT NULL DEFAULT 0,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_distributions_asset ON asset_distributions(asset_version_id);
CREATE INDEX idx_share_links_token ON share_links(token);
```

---

## Audit & Retention

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    actor_id        UUID REFERENCES users(id),
    actor_type      VARCHAR(30) NOT NULL DEFAULT 'user',  -- 'user', 'system', 'api_client'
    action          VARCHAR(50) NOT NULL,          -- 'asset.created', 'asset.downloaded', 'license.expired'
    resource_type   VARCHAR(50) NOT NULL,          -- 'asset', 'collection', 'user', 'brand'
    resource_id     UUID NOT NULL,
    details         JSONB,                         -- action-specific context
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE retention_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(255) NOT NULL,
    applies_to      VARCHAR(50) NOT NULL,          -- 'asset', 'audit_log', 'share_link'
    retention_days  INTEGER NOT NULL,
    disposal_action VARCHAR(30) NOT NULL,          -- 'archive', 'delete', 'anonymise'
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Partition audit_log by month for performance
CREATE INDEX idx_audit_log_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_log_resource ON audit_log(resource_type, resource_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
```

---

## External Integrations

```sql
CREATE TABLE oauth_clients (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    client_name     VARCHAR(255) NOT NULL,
    client_id       VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash VARCHAR(255) NOT NULL,
    redirect_uris   TEXT[],
    scopes          TEXT[],
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE webhooks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             TEXT NOT NULL,
    events          TEXT[] NOT NULL,               -- '{asset.created,asset.approved,license.expiring}'
    secret_hash     VARCHAR(255) NOT NULL,
    is_active       BOOLEAN NOT NULL DEFAULT true,
    last_triggered  TIMESTAMPTZ,
    failure_count   INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE identity_providers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    provider_type   VARCHAR(30) NOT NULL,          -- 'oidc', 'saml'
    name            VARCHAR(255) NOT NULL,
    issuer_url      TEXT,
    client_id       VARCHAR(255),
    client_secret_encrypted BYTEA,
    metadata_url    TEXT,                          -- SAML metadata endpoint
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Identity & Multi-Tenancy | 6 | tenants, users, roles, permissions, role_permissions, user_roles |
| Brand Management | 4 | brands, brand_guidelines, brand_colours, brand_fonts, brand_logos |
| Asset Management | 3 | assets, asset_versions, asset_thumbnails |
| Metadata (Standards) | 4 | asset_metadata_iptc, asset_metadata_dublin_core, asset_metadata_xmp, asset_metadata_exif |
| Taxonomy & Organisation | 5 | categories, tags, asset_categories, asset_tags, collections, collection_assets |
| Rights & Licensing | 4 | license_types, asset_licenses, model_releases, property_releases |
| Content Provenance | 2 | asset_c2pa_manifests, asset_c2pa_assertions |
| Workflows | 3 | workflow_templates, workflow_instances, workflow_step_actions |
| Search & AI | 4 | asset_embeddings, detected_faces, known_persons, compliance_checks |
| Distribution | 3 | distribution_channels, asset_distributions, share_links |
| Audit & Retention | 2 | audit_log, retention_policies |
| External Integrations | 3 | oauth_clients, webhooks, identity_providers |
| **Total** | **~43** | |

---

## Key Design Decisions

1. **Separate metadata tables per standard (IPTC, Dublin Core, EXIF, XMP)** rather than a single wide metadata table. This allows each standard's fields to be independently queryable and avoids a monolithic table with 100+ columns.

2. **UUID primary keys throughout** for globally unique identifiers, safe for distributed systems and API exposure without sequential ID enumeration.

3. **Row-level tenant isolation** via `tenant_id` foreign keys on all user-facing tables, supporting multi-tenant deployment with a single database.

4. **PLUS license fields as explicit columns** rather than JSONB, enabling direct SQL queries on license expiry, usage territory, and licensor without JSON path operators.

5. **C2PA manifests stored as both raw CBOR and parsed assertions** to support both cryptographic verification (raw bytes) and queryable provenance data (parsed JSONB assertions).

6. **pgvector for semantic search embeddings** enabling cosine similarity search across asset embeddings without an external vector database.

7. **Facial recognition with GDPR consent tracking** via `known_persons.consent_status`, ensuring biometric data compliance is enforced at the data layer.

8. **Brand hierarchy via self-referential `parent_brand_id`** supporting sub-brand relationships without a separate junction table.

9. **Smart collections via `smart_filter` JSONB** on the collections table — a pragmatic concession to JSONB in an otherwise fully normalised schema, since filter criteria are inherently variable.

10. **Audit log designed for time-based partitioning** with a composite index on `(tenant_id, created_at)` to support high-volume append and range queries per ISO 15489 retention requirements.
