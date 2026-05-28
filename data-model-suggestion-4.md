# Data Model Suggestion 4: Graph-Relational

> Project: Brand Asset Management · Created: 2026-05-19

## Philosophy

This approach layers a property graph on top of a relational core, using a generic `graph_nodes` / `graph_edges` pattern for relationship-rich queries while maintaining relational tables for operational CRUD and transactional integrity. The graph layer excels at the kinds of queries that are awkward in pure relational models: "find all assets connected to this person across all brands", "trace the provenance chain of this derivative work", "show me everything related to this campaign across 3 sub-brands".

Brand asset management is inherently graph-shaped. Assets relate to brands, people, campaigns, channels, licenses, and each other (derivatives, variants, translations). People relate to assets via creation, approval, facial appearance, model releases, and talent rights. Brands form hierarchies. Provenance chains (C2PA) form directed acyclic graphs. A graph layer makes these multi-hop, multi-type relationship traversals natural rather than requiring complex recursive CTEs or 6-way JOINs.

The design is inspired by knowledge graph approaches used in media companies (where Orange Logic CORTEX and similar AI-first platforms use semantic graphs for asset discovery) and by graph-enhanced relational patterns used in compliance systems (conflict-of-interest detection, ownership chain analysis).

**Best for:** Organisations managing complex brand hierarchies, derivative asset networks, talent relationship tracking, and provenance chains — especially media companies, agencies, and multi-brand enterprises.

**Trade-offs:**
- Pro: Multi-hop relationship queries (provenance chains, talent networks) are natural and fast
- Pro: New relationship types added by inserting edges, not by creating junction tables
- Pro: Semantic discovery powered by graph traversal + embeddings
- Pro: Conflict-of-interest and rights-chain analysis that would require recursive CTEs in relational models
- Con: Dual-model complexity — developers must understand both relational and graph paradigms
- Con: Graph consistency harder to enforce than relational foreign keys
- Con: Generic `graph_nodes`/`graph_edges` tables lack column-level type safety
- Con: Graph query optimisation requires different expertise than SQL tuning

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| C2PA 2.4 | Provenance chains modelled as directed graph edges between asset nodes, naturally representing C2PA's "history of actions" model |
| IPTC Photo Metadata 2025.1 | Stored in relational `assets` table with JSONB metadata; graph edges connect assets to persons, locations, and concepts referenced in IPTC fields |
| PLUS License Data Format | License relationships modelled as graph edges (Asset --licensed_to--> Organisation, Asset --licensed_by--> Licensor) |
| Schema.org | Graph node/edge types aligned with Schema.org vocabulary (CreativeWork, Person, Organization, Place) |
| Dublin Core | Dublin Core `relation` field maps to graph edges between related assets |
| ISO 3166 | Geographic nodes in the graph enable jurisdiction-based traversal |
| GDPR | Person nodes carry consent metadata; consent relationships tracked as graph edges |

---

## Relational Core (Operational Tables)

```sql
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    settings        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(320) NOT NULL,
    display_name    VARCHAR(255) NOT NULL,
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    permissions     JSONB NOT NULL DEFAULT '[]',
    is_active       BOOLEAN NOT NULL DEFAULT true,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

CREATE TABLE assets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    asset_type      VARCHAR(50) NOT NULL,
    title           VARCHAR(512) NOT NULL,
    description     TEXT,
    status          VARCHAR(30) NOT NULL DEFAULT 'draft',
    current_version INTEGER NOT NULL DEFAULT 1,
    file_name       VARCHAR(512) NOT NULL,
    file_size_bytes BIGINT NOT NULL,
    mime_type       VARCHAR(255) NOT NULL,
    storage_path    TEXT NOT NULL,
    checksum_sha256 CHAR(64) NOT NULL,
    dimensions      JSONB,
    metadata        JSONB NOT NULL DEFAULT '{}',
    ai_processing   JSONB NOT NULL DEFAULT '{}',
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    search_vector   tsvector,
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
    storage_path    TEXT NOT NULL,
    checksum_sha256 CHAR(64) NOT NULL,
    dimensions      JSONB,
    change_notes    TEXT,
    uploaded_by     UUID NOT NULL REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (asset_id, version_number)
);

CREATE INDEX idx_assets_tenant ON assets(tenant_id);
CREATE INDEX idx_assets_status ON assets(status);
CREATE INDEX idx_assets_type ON assets(asset_type);
CREATE INDEX idx_assets_metadata ON assets USING GIN(metadata);
CREATE INDEX idx_assets_search ON assets USING GIN(search_vector);
```

---

## Graph Layer

```sql
-- Graph nodes represent entities that participate in relationships
CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    node_type       VARCHAR(50) NOT NULL,
    -- Node types (aligned with Schema.org where applicable):
    -- 'Asset'          -- digital asset (linked to assets table)
    -- 'Person'         -- human (photographer, model, approver)
    -- 'Organization'   -- company, agency, licensor
    -- 'Brand'          -- brand or sub-brand
    -- 'Campaign'       -- marketing campaign
    -- 'Collection'     -- asset collection
    -- 'Place'          -- geographic location
    -- 'Concept'        -- abstract tag/topic for semantic linking
    -- 'License'        -- license agreement
    -- 'C2paManifest'   -- content provenance manifest

    -- Reference to the relational entity (if one exists)
    entity_table    VARCHAR(50),                   -- 'assets', 'users', 'brands', null for graph-only nodes
    entity_id       UUID,                          -- FK to the relational table

    -- Node properties
    label           VARCHAR(512) NOT NULL,         -- display name
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by node_type:
    -- Person:       {"role": "photographer", "consent_status": "consented", "consent_date": "2026-01-15"}
    -- Organization: {"type": "agency", "country": "US", "lei": "5493001KJTIIGC8Y1R12"}
    -- Brand:        {"tier": "primary", "active": true}
    -- Campaign:     {"start_date": "2026-06-01", "end_date": "2026-08-31", "budget": 50000}
    -- Place:        {"iso_3166": "US-CA", "lat": 34.0522, "lng": -118.2437}
    -- Concept:      {"category": "season", "value": "summer"}
    -- License:      {"type": "rights_managed", "expiry": "2027-01-01", "territory": "WW"}
    -- C2paManifest: {"claim_generator": "BrandDAM/1.0", "is_valid": true}

    embedding       vector(1536),                  -- semantic embedding for similarity search
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Graph edges represent typed, directed relationships between nodes
CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    -- Asset relationships:
    --   'belongs_to_brand'     -- Asset -> Brand
    --   'in_collection'        -- Asset -> Collection
    --   'in_campaign'          -- Asset -> Campaign
    --   'derivative_of'        -- Asset -> Asset (cropped, resized, translated version)
    --   'variant_of'           -- Asset -> Asset (different format of same content)
    --   'related_to'           -- Asset -> Asset (editorial relationship)
    --   'located_at'           -- Asset -> Place
    --   'tagged_with'          -- Asset -> Concept
    --
    -- Person relationships:
    --   'created_by'           -- Asset -> Person (photographer, designer)
    --   'approved_by'          -- Asset -> Person (approver)
    --   'features'             -- Asset -> Person (person appears in asset)
    --   'has_model_release'    -- Person -> Asset (release document)
    --   'employed_by'          -- Person -> Organization
    --
    -- Rights relationships:
    --   'licensed_by'          -- Asset -> Organization (licensor)
    --   'licensed_to'          -- Asset -> Organization (licensee)
    --   'governed_by'          -- Asset -> License
    --
    -- Provenance relationships:
    --   'has_manifest'         -- Asset -> C2paManifest
    --   'derived_from'         -- C2paManifest -> C2paManifest (provenance chain)
    --   'generated_by'         -- Asset -> Concept (AI system node)
    --
    -- Brand relationships:
    --   'sub_brand_of'         -- Brand -> Brand
    --   'manages'              -- Organization -> Brand

    -- Edge properties
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by edge_type:
    -- 'features':         {"bounding_box": {...}, "confidence": 0.95, "detected_by": "model-v2"}
    -- 'licensed_to':      {"start": "2026-01-01", "expiry": "2027-01-01", "territory": "US"}
    -- 'derivative_of':    {"transformation": "crop_resize", "created_at": "2026-05-19"}
    -- 'tagged_with':      {"confidence": 0.92, "source": "ai_vision"}

    weight          NUMERIC(5,3) DEFAULT 1.0,      -- edge weight for ranking/scoring
    valid_from      TIMESTAMPTZ,                   -- temporal validity
    valid_to        TIMESTAMPTZ,                   -- null = still valid
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Core graph indexes
CREATE INDEX idx_graph_nodes_tenant ON graph_nodes(tenant_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes(node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_table, entity_id);
CREATE INDEX idx_graph_nodes_label ON graph_nodes(label);
CREATE INDEX idx_graph_nodes_properties ON graph_nodes USING GIN(properties);
CREATE INDEX idx_graph_nodes_embedding ON graph_nodes
    USING ivfflat (embedding vector_cosine_ops) WITH (lists = 100);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_id);
CREATE INDEX idx_graph_edges_type ON graph_edges(edge_type);
CREATE INDEX idx_graph_edges_tenant ON graph_edges(tenant_id);
CREATE INDEX idx_graph_edges_temporal ON graph_edges(valid_from, valid_to)
    WHERE valid_from IS NOT NULL;
CREATE INDEX idx_graph_edges_source_type ON graph_edges(source_id, edge_type);
CREATE INDEX idx_graph_edges_target_type ON graph_edges(target_id, edge_type);
```

---

## Licensing (Graph-Native)

```sql
-- Licenses are both graph nodes and have a relational table for transactional queries
CREATE TABLE licenses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    asset_id        UUID NOT NULL REFERENCES assets(id) ON DELETE CASCADE,
    license_type    VARCHAR(50) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'active',
    terms           JSONB NOT NULL DEFAULT '{}',
    license_start   DATE,
    license_expiry  DATE,
    is_perpetual    BOOLEAN NOT NULL DEFAULT false,
    graph_node_id   UUID REFERENCES graph_nodes(id),  -- link to graph representation
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_licenses_asset ON licenses(asset_id);
CREATE INDEX idx_licenses_expiry ON licenses(license_expiry);
CREATE INDEX idx_licenses_status ON licenses(status);
```

---

## Audit & Compliance

```sql
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_id        UUID,
    action          VARCHAR(50) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID NOT NULL,
    details         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_tenant_time ON audit_log(tenant_id, created_at DESC);
CREATE INDEX idx_audit_resource ON audit_log(resource_type, resource_id);
```

---

## Example: Graph Traversal Queries

### Find all assets featuring a specific person (talent rights check)

```sql
-- "Show me every asset that features person X, across all brands"
SELECT a.id, a.title, a.status,
       e.properties->>'confidence' AS detection_confidence,
       pn.properties->>'consent_status' AS consent_status
FROM graph_nodes pn
JOIN graph_edges e ON e.target_id = pn.id AND e.edge_type = 'features'
JOIN graph_nodes an ON an.id = e.source_id AND an.node_type = 'Asset'
JOIN assets a ON a.id = an.entity_id
WHERE pn.node_type = 'Person'
  AND pn.label = 'Jane Smith'
  AND pn.tenant_id = 'tenant-uuid';
```

### Trace provenance chain of a derivative asset

```sql
-- "Show the full creation lineage of this asset"
WITH RECURSIVE provenance AS (
    -- Start with the target asset
    SELECT n.id AS node_id, n.label, n.properties, 0 AS depth
    FROM graph_nodes n
    WHERE n.entity_table = 'assets' AND n.entity_id = 'asset-uuid'

    UNION ALL

    -- Follow 'derivative_of' edges backwards
    SELECT parent.id, parent.label, parent.properties, p.depth + 1
    FROM provenance p
    JOIN graph_edges e ON e.source_id = p.node_id AND e.edge_type = 'derivative_of'
    JOIN graph_nodes parent ON parent.id = e.target_id
    WHERE p.depth < 10  -- prevent infinite loops
)
SELECT * FROM provenance ORDER BY depth ASC;
```

### Find all assets in a campaign with expiring licenses

```sql
-- "Which assets in the Summer 2026 campaign have licenses expiring within 30 days?"
SELECT a.id, a.title, l.license_expiry, l.license_type
FROM graph_nodes cn
JOIN graph_edges ce ON ce.target_id = cn.id AND ce.edge_type = 'in_campaign'
JOIN graph_nodes an ON an.id = ce.source_id AND an.node_type = 'Asset'
JOIN assets a ON a.id = an.entity_id
JOIN licenses l ON l.asset_id = a.id
WHERE cn.node_type = 'Campaign'
  AND cn.label = 'Summer 2026'
  AND l.license_expiry BETWEEN now() AND now() + INTERVAL '30 days'
  AND l.status = 'active';
```

### Find brand hierarchy and all associated assets

```sql
-- "Show brand hierarchy and asset counts"
WITH RECURSIVE brand_tree AS (
    SELECT n.id, n.label, n.properties, 0 AS depth
    FROM graph_nodes n
    WHERE n.node_type = 'Brand'
      AND n.tenant_id = 'tenant-uuid'
      AND NOT EXISTS (
          SELECT 1 FROM graph_edges e
          WHERE e.source_id = n.id AND e.edge_type = 'sub_brand_of'
      )

    UNION ALL

    SELECT child.id, child.label, child.properties, bt.depth + 1
    FROM brand_tree bt
    JOIN graph_edges e ON e.target_id = bt.id AND e.edge_type = 'sub_brand_of'
    JOIN graph_nodes child ON child.id = e.source_id
)
SELECT bt.label AS brand_name, bt.depth,
       COUNT(DISTINCT ae.source_id) AS asset_count
FROM brand_tree bt
LEFT JOIN graph_edges ae ON ae.target_id = bt.id AND ae.edge_type = 'belongs_to_brand'
GROUP BY bt.id, bt.label, bt.depth
ORDER BY bt.depth, bt.label;
```

### Semantic similarity via graph embeddings

```sql
-- "Find assets semantically similar to this one, weighted by relationship proximity"
WITH target_embedding AS (
    SELECT embedding FROM graph_nodes
    WHERE entity_table = 'assets' AND entity_id = 'asset-uuid'
)
SELECT a.id, a.title,
       1 - (gn.embedding <=> te.embedding) AS similarity,
       COUNT(DISTINCT shared_edge.id) AS shared_connections
FROM graph_nodes gn
CROSS JOIN target_embedding te
JOIN assets a ON a.id = gn.entity_id
LEFT JOIN graph_edges shared_edge ON (
    shared_edge.source_id = gn.id
    AND shared_edge.target_id IN (
        SELECT target_id FROM graph_edges WHERE source_id = (
            SELECT id FROM graph_nodes WHERE entity_table = 'assets' AND entity_id = 'asset-uuid'
        )
    )
)
WHERE gn.node_type = 'Asset'
  AND gn.entity_id != 'asset-uuid'
  AND gn.tenant_id = 'tenant-uuid'
GROUP BY a.id, a.title, gn.embedding, te.embedding
ORDER BY similarity DESC, shared_connections DESC
LIMIT 20;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Identity | 2 | tenants, users |
| Assets (Relational) | 2 | assets, asset_versions |
| Graph Layer | 2 | graph_nodes, graph_edges |
| Licensing | 1 | licenses (also represented as graph nodes) |
| Audit | 1 | audit_log |
| **Total** | **~8** | Graph layer absorbs brands, collections, persons, campaigns, provenance, and relationships |

---

## Key Design Decisions

1. **Generic graph layer (`graph_nodes` + `graph_edges`)** rather than a dedicated graph database like Neo4j. This keeps the entire system in PostgreSQL, avoiding operational complexity of a second database, while providing graph traversal via recursive CTEs.

2. **Dual representation: relational + graph.** Assets and licenses exist in both relational tables (for fast CRUD and transactional queries) and as graph nodes (for relationship traversal). The `entity_table` / `entity_id` columns link graph nodes to their relational counterparts.

3. **Brands, persons, campaigns, and collections exist only as graph nodes** rather than separate relational tables. This dramatically reduces table count and makes adding new entity types a data operation (insert a node) rather than a schema operation (create a table).

4. **Edge types encode relationship semantics.** Rather than junction tables like `asset_tags`, `asset_categories`, `collection_assets`, the graph uses typed edges: `tagged_with`, `in_collection`, `belongs_to_brand`. This unifies all relationships into a single queryable pattern.

5. **Temporal edges via `valid_from` / `valid_to`** enable point-in-time queries: "which brand did this asset belong to on date X?" This is crucial for license validity windows and campaign date ranges.

6. **Embeddings on graph nodes** enable semantic similarity search that respects graph structure. Two assets can be similar both by visual/textual content (embedding distance) and by graph proximity (shared edges to the same concepts, brands, or persons).

7. **C2PA provenance chains as graph edges** (`derived_from` between manifest nodes) make provenance a first-class traversable structure rather than a flat table. This mirrors C2PA's own directed acyclic graph model.

8. **Fewest tables of any approach (~8)** because the graph absorbs what would otherwise be 15+ tables (brands, brand_guidelines, categories, collections, persons, campaigns, etc.). The trade-off is that graph node `properties` are JSONB and lack column-level constraints.

9. **Schema.org vocabulary alignment** for node types (`Person`, `Organization`, `CreativeWork`, `Place`) ensures semantic interoperability and makes the graph interpretable by external systems and AI agents.

10. **Composite indexes on `(source_id, edge_type)` and `(target_id, edge_type)`** optimise the most common graph traversal pattern: "find all edges of type X from/to node Y". This is critical for performant recursive CTEs in PostgreSQL.
