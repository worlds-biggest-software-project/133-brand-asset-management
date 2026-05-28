# Brand Asset Management — Phased Development Plan

> Project: 133-brand-asset-management · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (Backend) | TypeScript (Node.js) | API-heavy project with a full web UI; the metadata JSONB queries, webhook handling, and real-time events suit the async Node.js model. TypeScript provides type safety across the API boundary, shared types between server and client, and strong tooling for OpenAPI generation. |
| Language (Frontend) | TypeScript (React) | DAM UIs are interaction-heavy (drag-drop upload, grid/list views, image crop, inline metadata editing). React's component model and ecosystem (react-dropzone, react-image-crop) are purpose-built for this. Next.js provides SSR for the brand guidelines portal. |
| API Framework | Fastify | High-performance Node.js framework with first-class TypeScript support, built-in JSON schema validation, and automatic OpenAPI 3.1 spec generation via @fastify/swagger. Preferred over Express for its schema-driven request/response validation. |
| Frontend Framework | Next.js 15 (App Router) | SSR for the public brand guidelines portal (SEO, fast initial load); CSR for the internal DAM dashboard. API routes colocated for the BFF pattern. |
| Database | PostgreSQL 16 + pgvector | Relational core for asset metadata, RBAC, and licensing; JSONB for flexible metadata (IPTC, EXIF, Dublin Core); pgvector extension for semantic search embeddings. Single database avoids operational complexity of a separate vector DB. |
| Object Storage | S3-compatible (MinIO self-hosted / AWS S3) | Industry-standard blob storage for asset files. MinIO enables self-hosted deployments; S3 for cloud. Abstracted behind a storage service interface. |
| Task Queue | BullMQ (Redis) | Async job processing for: AI tagging on ingest, thumbnail generation, video transcoding, license expiry checks, embedding generation. BullMQ provides retries, priorities, rate limiting, and a dashboard. |
| Search | PostgreSQL tsvector + pgvector | Full-text search via tsvector for keyword/metadata queries. Semantic search via pgvector cosine similarity. Avoids Elasticsearch dependency for MVP; can be added later behind the search service interface. |
| Cache | Redis 7 | Session store, BullMQ backend, CDN cache invalidation signals, and rate limiting. Single Redis instance serves multiple concerns. |
| AI / LLM | OpenAI API (gpt-4o-mini for tagging, text-embedding-3-large for embeddings, gpt-4o for contract extraction) | Multi-modal vision model for auto-tagging; embedding model for semantic search; large model for contract term extraction. Abstracted behind an AI service interface to allow model swaps. |
| Image Processing | Sharp | High-performance Node.js image processing for thumbnail generation, format conversion, and resize operations. libvips-based; handles JPEG, PNG, WebP, AVIF, SVG, TIFF. |
| Video Processing | FFmpeg (via fluent-ffmpeg) | Industry-standard transcoding, thumbnail extraction, and metadata parsing for video assets. |
| Authentication | Lucia Auth + Arctic (OAuth/OIDC) | Lightweight session-based auth with OIDC/SAML support for enterprise SSO (Okta, Azure AD). JWT for API clients. Satisfies OAuth 2.0/OIDC standards from standards.md. |
| ORM / Query Builder | Drizzle ORM | Type-safe SQL builder with zero runtime overhead, first-class PostgreSQL JSONB and pgvector support, and migration generation. Preferred over Prisma for JSONB and raw SQL flexibility. |
| Testing | Vitest (unit/integration) + Playwright (E2E) | Vitest is fast, TypeScript-native, and compatible with the Node.js ecosystem. Playwright for browser-based E2E testing of the DAM UI. |
| Linting & Formatting | ESLint + Prettier + oxlint | oxlint for fast pre-commit checks; ESLint for comprehensive rules; Prettier for formatting. |
| Type Checking | TypeScript strict mode (tsc --noEmit) | Full strict mode with no implicit any, strict null checks, and exhaustive switch. |
| Package Manager | pnpm | Faster than npm, strict dependency resolution, workspace support for monorepo. |
| Containerisation | Docker + Docker Compose | Multi-container setup: app server, PostgreSQL, Redis, MinIO. docker-compose.yml for local dev; Dockerfile for production. |
| Metadata Parsing | ExifTool (via exiftool-vendored) + c2pa-node | ExifTool for IPTC/EXIF/XMP extraction (supports IPTC 2025.1 properties). c2pa-node for C2PA manifest reading/writing. |
| API Documentation | OpenAPI 3.1 (auto-generated from Fastify schemas) | Machine-readable API spec generated from route schemas. Serves as contract for SDK generation and API portal. |

### Project Structure

```
brand-asset-management/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── drizzle.config.ts
├── packages/
│   └── shared/                          # Shared types & constants
│       ├── src/
│       │   ├── types/
│       │   │   ├── asset.ts
│       │   │   ├── brand.ts
│       │   │   ├── license.ts
│       │   │   ├── metadata.ts
│       │   │   ├── user.ts
│       │   │   └── workflow.ts
│       │   ├── constants/
│       │   │   ├── asset-statuses.ts
│       │   │   ├── license-types.ts
│       │   │   └── permissions.ts
│       │   └── schemas/
│       │       ├── iptc.ts              # IPTC 2025.1 JSON schema
│       │       ├── dublin-core.ts
│       │       ├── plus.ts
│       │       └── c2pa.ts
│       └── package.json
├── apps/
│   ├── api/                             # Fastify API server
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── config.ts
│   │   │   ├── db/
│   │   │   │   ├── schema.ts            # Drizzle schema
│   │   │   │   ├── migrations/
│   │   │   │   └── seed.ts
│   │   │   ├── routes/
│   │   │   │   ├── assets.ts
│   │   │   │   ├── brands.ts
│   │   │   │   ├── collections.ts
│   │   │   │   ├── licenses.ts
│   │   │   │   ├── search.ts
│   │   │   │   ├── users.ts
│   │   │   │   ├── workflows.ts
│   │   │   │   ├── auth.ts
│   │   │   │   └── webhooks.ts
│   │   │   ├── services/
│   │   │   │   ├── asset.service.ts
│   │   │   │   ├── storage.service.ts
│   │   │   │   ├── metadata.service.ts
│   │   │   │   ├── search.service.ts
│   │   │   │   ├── ai.service.ts
│   │   │   │   ├── license.service.ts
│   │   │   │   ├── brand.service.ts
│   │   │   │   ├── compliance.service.ts
│   │   │   │   ├── provenance.service.ts
│   │   │   │   └── distribution.service.ts
│   │   │   ├── jobs/
│   │   │   │   ├── queue.ts
│   │   │   │   ├── thumbnail.job.ts
│   │   │   │   ├── ai-tag.job.ts
│   │   │   │   ├── embedding.job.ts
│   │   │   │   ├── video-transcode.job.ts
│   │   │   │   ├── license-expiry.job.ts
│   │   │   │   └── compliance-scan.job.ts
│   │   │   ├── middleware/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── tenant.ts
│   │   │   │   ├── rbac.ts
│   │   │   │   └── rate-limit.ts
│   │   │   └── lib/
│   │   │       ├── exiftool.ts
│   │   │       ├── c2pa.ts
│   │   │       ├── sharp-utils.ts
│   │   │       └── ffmpeg-utils.ts
│   │   ├── tests/
│   │   │   ├── unit/
│   │   │   ├── integration/
│   │   │   └── fixtures/
│   │   └── package.json
│   └── web/                             # Next.js frontend
│       ├── src/
│       │   ├── app/
│       │   │   ├── (dashboard)/         # Authenticated DAM UI
│       │   │   │   ├── assets/
│       │   │   │   ├── brands/
│       │   │   │   ├── collections/
│       │   │   │   ├── licenses/
│       │   │   │   └── settings/
│       │   │   ├── (portal)/            # Public brand guidelines portal
│       │   │   │   └── [brand-slug]/
│       │   │   └── api/                 # BFF routes
│       │   ├── components/
│       │   │   ├── asset-grid/
│       │   │   ├── asset-detail/
│       │   │   ├── upload/
│       │   │   ├── metadata-editor/
│       │   │   ├── search/
│       │   │   └── brand-guidelines/
│       │   ├── hooks/
│       │   ├── lib/
│       │   │   └── api-client.ts
│       │   └── styles/
│       ├── tests/
│       │   ├── e2e/
│       │   └── component/
│       └── package.json
└── scripts/
    ├── dev-setup.sh
    └── seed-demo-data.ts
```

---

## Phase 1: Foundation and Project Setup

### Purpose

Establish the monorepo structure, database schema, configuration system, and development tooling. After this phase, a developer can clone the repo, run `docker compose up`, and have a working (but empty) API server connected to PostgreSQL, Redis, and MinIO. No business logic yet — just the scaffold.

### Tasks

#### 1.1 — Monorepo Initialisation

**What**: Create the pnpm workspace monorepo with shared package, API app, and web app stubs.

**Design**:

Root `package.json`:
```json
{
  "name": "brand-asset-management",
  "private": true,
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "typecheck": "turbo run typecheck",
    "db:migrate": "pnpm --filter api db:migrate",
    "db:seed": "pnpm --filter api db:seed"
  }
}
```

`pnpm-workspace.yaml`:
```yaml
packages:
  - "packages/*"
  - "apps/*"
```

`turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] }
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors → all workspace packages resolve`
- `Unit: turbo run build succeeds → dist/ produced for each package`
- `Unit: turbo run typecheck succeeds → no TypeScript errors`

#### 1.2 — Docker Compose Development Environment

**What**: Docker Compose stack with PostgreSQL 16 (pgvector), Redis 7, MinIO, and the API server.

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: brand_dam
      POSTGRES_USER: dam_user
      POSTGRES_PASSWORD: dam_local_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: development
    environment:
      DATABASE_URL: postgresql://dam_user:dam_local_password@postgres:5432/brand_dam
      REDIS_URL: redis://redis:6379
      S3_ENDPOINT: http://minio:9000
      S3_ACCESS_KEY: minioadmin
      S3_SECRET_KEY: minioadmin
      S3_BUCKET: brand-assets
    ports: ["3001:3001"]
    volumes: ["./apps/api/src:/app/apps/api/src"]
    depends_on: [postgres, redis, minio]

volumes:
  pgdata:
  miniodata:
```

**Testing**:
- `Integration: docker compose up --wait → all 4 services healthy within 60 seconds`
- `Integration: psql query against postgres → pgvector extension available (SELECT 'test'::vector(3))`
- `Integration: MinIO healthcheck → bucket creation API responds 200`
- `Integration: Redis PING → PONG response`

#### 1.3 — Configuration System

**What**: Centralised config loading from environment variables with validation and typed defaults.

**Design**:

```typescript
// apps/api/src/config.ts
import { z } from "zod";

const configSchema = z.object({
  // Server
  PORT: z.coerce.number().default(3001),
  HOST: z.string().default("0.0.0.0"),
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  LOG_LEVEL: z.enum(["fatal", "error", "warn", "info", "debug", "trace"]).default("info"),

  // Database
  DATABASE_URL: z.string().url(),
  DATABASE_POOL_SIZE: z.coerce.number().default(10),

  // Redis
  REDIS_URL: z.string().url(),

  // Object Storage (S3-compatible)
  S3_ENDPOINT: z.string().url(),
  S3_ACCESS_KEY: z.string().min(1),
  S3_SECRET_KEY: z.string().min(1),
  S3_BUCKET: z.string().default("brand-assets"),
  S3_REGION: z.string().default("us-east-1"),

  // AI / LLM
  OPENAI_API_KEY: z.string().optional(),
  AI_TAGGING_MODEL: z.string().default("gpt-4o-mini"),
  AI_EMBEDDING_MODEL: z.string().default("text-embedding-3-large"),
  AI_EMBEDDING_DIMENSIONS: z.coerce.number().default(1536),

  // Auth
  SESSION_SECRET: z.string().min(32),
  OIDC_ISSUER: z.string().url().optional(),
  OIDC_CLIENT_ID: z.string().optional(),
  OIDC_CLIENT_SECRET: z.string().optional(),

  // Features
  ENABLE_FACIAL_RECOGNITION: z.coerce.boolean().default(false),
  ENABLE_VIDEO_TRANSCODING: z.coerce.boolean().default(false),
  MAX_UPLOAD_SIZE_MB: z.coerce.number().default(500),
});

export type Config = z.infer<typeof configSchema>;

export function loadConfig(): Config {
  const result = configSchema.safeParse(process.env);
  if (!result.success) {
    const formatted = result.error.issues
      .map((i) => `  ${i.path.join(".")}: ${i.message}`)
      .join("\n");
    throw new Error(`Configuration validation failed:\n${formatted}`);
  }
  return result.data;
}
```

**Testing**:
- `Unit: all env vars set → Config object with correct types`
- `Unit: DATABASE_URL missing → error message includes "DATABASE_URL"`
- `Unit: PORT="abc" → error about expected number`
- `Unit: NODE_ENV="staging" → error listing valid enum values`
- `Unit: defaults applied → ENABLE_FACIAL_RECOGNITION is false when not set`

#### 1.4 — Database Schema and Migrations

**What**: Drizzle ORM schema implementing the Hybrid Relational + JSONB data model (data-model-suggestion-3) with targeted normalised tables for licensing and compliance.

**Design**:

The schema adopts data-model-suggestion-3 (Hybrid Relational + JSONB, ~19 tables) as the primary data model. It provides the best balance between development velocity and query capability for MVP. Normalised metadata columns from data-model-suggestion-1 are used selectively for the most-queried IPTC fields. The audit log design from data-model-suggestion-2 (event-sourced) informs the audit table structure.

```typescript
// apps/api/src/db/schema.ts
import { pgTable, uuid, varchar, text, boolean, integer,
         bigint, timestamp, jsonb, index, uniqueIndex,
         date, numeric, char } from "drizzle-orm/pg-core";
import { vector } from "drizzle-orm/pg-core"; // pgvector support

// -- Core Identity & Multi-Tenancy --

export const tenants = pgTable("tenants", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
  plan: varchar("plan", { length: 50 }).notNull().default("free"),
  settings: jsonb("settings").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  email: varchar("email", { length: 320 }).notNull(),
  displayName: varchar("display_name", { length: 255 }).notNull(),
  avatarUrl: text("avatar_url"),
  role: varchar("role", { length: 50 }).notNull().default("viewer"),
  permissions: jsonb("permissions").notNull().default([]),
  authConfig: jsonb("auth_config").notNull().default({}),
  isActive: boolean("is_active").notNull().default(true),
  lastLoginAt: timestamp("last_login_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("users_tenant_email").on(table.tenantId, table.email),
  index("idx_users_tenant").on(table.tenantId),
]);

// -- Assets --

export const assets = pgTable("assets", {
  id: uuid("id").primaryKey().defaultRandom(),
  tenantId: uuid("tenant_id").notNull().references(() => tenants.id),
  brandId: uuid("brand_id").references(() => brands.id),
  assetType: varchar("asset_type", { length: 50 }).notNull(),
  title: varchar("title", { length: 512 }).notNull(),
  description: text("description"),
  status: varchar("status", { length: 30 }).notNull().default("draft"),
  currentVersion: integer("current_version").notNull().default(1),
  fileName: varchar("file_name", { length: 512 }).notNull(),
  fileSizeBytes: bigint("file_size_bytes", { mode: "number" }).notNull(),
  mimeType: varchar("mime_type", { length: 255 }).notNull(),
  storagePath: text("storage_path").notNull(),
  checksumSha256: char("checksum_sha256", { length: 64 }).notNull(),
  dimensions: jsonb("dimensions"),
  metadata: jsonb("metadata").notNull().default({}),
  aiProcessing: jsonb("ai_processing").notNull().default({}),
  uploadedBy: uuid("uploaded_by").notNull().references(() => users.id),
  approvedBy: uuid("approved_by").references(() => users.id),
  approvedAt: timestamp("approved_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  index("idx_assets_tenant").on(table.tenantId),
  index("idx_assets_brand").on(table.brandId),
  index("idx_assets_status").on(table.status),
  index("idx_assets_type").on(table.assetType),
]);

export const assetVersions = pgTable("asset_versions", {
  id: uuid("id").primaryKey().defaultRandom(),
  assetId: uuid("asset_id").notNull().references(() => assets.id, { onDelete: "cascade" }),
  versionNumber: integer("version_number").notNull(),
  fileName: varchar("file_name", { length: 512 }).notNull(),
  fileSizeBytes: bigint("file_size_bytes", { mode: "number" }).notNull(),
  mimeType: varchar("mime_type", { length: 255 }).notNull(),
  storagePath: text("storage_path").notNull(),
  checksumSha256: char("checksum_sha256", { length: 64 }).notNull(),
  dimensions: jsonb("dimensions"),
  embeddedMetadata: jsonb("embedded_metadata"),
  changeNotes: text("change_notes"),
  uploadedBy: uuid("uploaded_by").notNull().references(() => users.id),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex("asset_versions_unique").on(table.assetId, table.versionNumber),
]);

// Remaining tables defined similarly following the Hybrid JSONB model:
// brands, tags, asset_tags, collections, collection_assets,
// asset_licenses, releases, asset_provenance, asset_embeddings,
// detected_faces, compliance_checks, workflows, workflow_actions,
// share_links, audit_log
```

Full DDL for all 19 tables follows the patterns established in data-model-suggestion-3, with these additions from data-model-suggestion-1:
- `asset_embeddings` uses `vector(1536)` type for pgvector semantic search
- `detected_faces` includes `face_encoding vector(128)` for face matching

**Testing**:
- `Integration: drizzle-kit push against empty database → all 19 tables created`
- `Integration: drizzle-kit generate → migration SQL file produced`
- `Unit: schema types compile → no TypeScript errors`
- `Integration: insert tenant + user + asset → all foreign keys resolve`
- `Integration: JSONB metadata insert → GIN index created and queryable`
- `Integration: vector column insert → pgvector cosine similarity query works`

#### 1.5 — Fastify Server Bootstrap

**What**: Minimal Fastify server with health check, CORS, request logging, error handling, and OpenAPI spec generation.

**Design**:

```typescript
// apps/api/src/server.ts
import Fastify from "fastify";
import cors from "@fastify/cors";
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";
import { loadConfig } from "./config.js";

export async function buildServer() {
  const config = loadConfig();

  const app = Fastify({
    logger: {
      level: config.LOG_LEVEL,
      transport: config.NODE_ENV === "development"
        ? { target: "pino-pretty" }
        : undefined,
    },
    ajv: {
      customOptions: { removeAdditional: "all", coerceTypes: true },
    },
  });

  // Plugins
  await app.register(cors, { origin: true, credentials: true });
  await app.register(swagger, {
    openapi: {
      info: {
        title: "Brand Asset Management API",
        version: "1.0.0",
        description: "AI-native digital asset management platform",
      },
      servers: [{ url: `http://${config.HOST}:${config.PORT}` }],
    },
  });
  await app.register(swaggerUi, { routePrefix: "/docs" });

  // Health check
  app.get("/health", {
    schema: {
      response: {
        200: {
          type: "object",
          properties: {
            status: { type: "string", enum: ["ok"] },
            timestamp: { type: "string", format: "date-time" },
            version: { type: "string" },
          },
        },
      },
    },
  }, async () => ({
    status: "ok" as const,
    timestamp: new Date().toISOString(),
    version: "1.0.0",
  }));

  // Global error handler
  app.setErrorHandler((error, request, reply) => {
    const statusCode = error.statusCode ?? 500;
    app.log.error({ err: error, requestId: request.id }, "Request error");
    reply.status(statusCode).send({
      error: error.name,
      message: statusCode < 500 ? error.message : "Internal Server Error",
      statusCode,
    });
  });

  return app;
}
```

**Testing**:
- `Integration: GET /health → 200 with { status: "ok", timestamp, version }`
- `Integration: GET /docs → Swagger UI HTML page`
- `Integration: GET /docs/json → valid OpenAPI 3.1 JSON spec`
- `Unit: invalid route → 404 with JSON error body`
- `Unit: thrown error in handler → 500 with sanitised message (no stack trace in production)`

---

## Phase 2: Storage and Asset Ingest Pipeline

### Purpose

Build the core asset upload and storage pipeline. After this phase, users can upload files (images, video, documents) via the API, files are stored in S3-compatible storage, metadata is extracted (EXIF, IPTC, XMP), thumbnails are generated, and versions are tracked. This is the foundation that every subsequent feature builds upon.

### Tasks

#### 2.1 — Storage Service Abstraction

**What**: S3-compatible storage service with upload, download, delete, and presigned URL operations.

**Design**:

```typescript
// apps/api/src/services/storage.service.ts
import { S3Client, PutObjectCommand, GetObjectCommand,
         DeleteObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

export interface StorageService {
  upload(key: string, body: Buffer | Readable, contentType: string): Promise<StorageResult>;
  download(key: string): Promise<Readable>;
  delete(key: string): Promise<void>;
  getPresignedUrl(key: string, expiresInSeconds?: number): Promise<string>;
  getPresignedUploadUrl(key: string, contentType: string, maxSizeBytes: number): Promise<string>;
}

export interface StorageResult {
  key: string;
  bucket: string;
  size: number;
  etag: string;
}

// Storage key format: {tenant_id}/{asset_id}/{version}/{filename}
export function buildStorageKey(
  tenantId: string,
  assetId: string,
  version: number,
  filename: string
): string {
  return `${tenantId}/${assetId}/v${version}/${filename}`;
}

// Thumbnail key format: {tenant_id}/{asset_id}/thumbs/{size}-{filename}.webp
export function buildThumbnailKey(
  tenantId: string,
  assetId: string,
  size: "sm" | "md" | "lg" | "preview",
  filename: string
): string {
  return `${tenantId}/${assetId}/thumbs/${size}-${filename}.webp`;
}
```

**Testing**:
- `Integration (MinIO): upload 1MB file → StorageResult with correct key, size, etag`
- `Integration (MinIO): download uploaded file → content matches original`
- `Integration (MinIO): delete file → subsequent download returns NoSuchKey`
- `Integration (MinIO): presigned URL → HTTP GET returns file content with correct Content-Type`
- `Unit: buildStorageKey → correct path format "tenant/asset/v1/file.jpg"`
- `Unit: buildThumbnailKey → correct path format "tenant/asset/thumbs/sm-file.webp"`

#### 2.2 — Metadata Extraction Service

**What**: Extract EXIF, IPTC, and XMP metadata from uploaded files using ExifTool.

**Design**:

```typescript
// apps/api/src/services/metadata.service.ts
import { exiftool } from "exiftool-vendored";

export interface ExtractedMetadata {
  iptc: IptcMetadata;
  exif: ExifMetadata;
  dublinCore: DublinCoreMetadata;
  xmpRaw?: string;
  technical: TechnicalMetadata;
}

export interface IptcMetadata {
  headline?: string;
  caption?: string;
  keywords?: string[];
  creator?: string;
  copyrightNotice?: string;
  creditLine?: string;
  source?: string;
  countryCode?: string;       // ISO 3166-1 alpha-3
  city?: string;
  provinceState?: string;
  modelReleaseStatus?: "MR-NON" | "MR-NAP" | "MR-UPR" | "MR-LPR";
  // IPTC 2025.1 AI fields
  aiPromptInfo?: string;
  aiPromptWriter?: string;
  aiSystemUsed?: string;
  aiSystemVersion?: string;
}

export interface ExifMetadata {
  cameraMake?: string;
  cameraModel?: string;
  focalLength?: number;
  aperture?: string;
  shutterSpeed?: string;
  iso?: number;
  flashFired?: boolean;
  gpsLatitude?: number;
  gpsLongitude?: number;
  captureDate?: string;
}

export interface TechnicalMetadata {
  width?: number;
  height?: number;
  dpi?: number;
  colourSpace?: string;
  durationSecs?: number;
  codec?: string;
  bitrate?: number;
  pageCount?: number;
}

export async function extractMetadata(filePath: string): Promise<ExtractedMetadata> {
  const tags = await exiftool.read(filePath);
  return {
    iptc: mapIptcTags(tags),
    exif: mapExifTags(tags),
    dublinCore: mapDublinCoreTags(tags),
    xmpRaw: tags.XMP,
    technical: mapTechnicalTags(tags),
  };
}
```

**Testing**:
- `Unit (fixture): JPEG with IPTC data → iptc.headline, iptc.keywords extracted correctly`
- `Unit (fixture): JPEG with EXIF GPS → exif.gpsLatitude, exif.gpsLongitude populated`
- `Unit (fixture): JPEG with IPTC 2025.1 AI fields → aiSystemUsed and aiPromptInfo extracted`
- `Unit (fixture): PNG with no metadata → all fields undefined, no error thrown`
- `Unit (fixture): PDF document → technical.pageCount populated`
- `Unit (fixture): MP4 video → technical.durationSecs, technical.codec populated`
- `Unit: invalid file path → descriptive error with path in message`

#### 2.3 — Thumbnail Generation

**What**: Generate multiple thumbnail sizes from uploaded images and video keyframes.

**Design**:

```typescript
// apps/api/src/services/thumbnail.service.ts
import sharp from "sharp";

export interface ThumbnailSpec {
  size: "sm" | "md" | "lg" | "preview";
  maxWidth: number;
  maxHeight: number;
  quality: number;
}

export const THUMBNAIL_SPECS: ThumbnailSpec[] = [
  { size: "sm", maxWidth: 150, maxHeight: 150, quality: 80 },
  { size: "md", maxWidth: 400, maxHeight: 400, quality: 80 },
  { size: "lg", maxWidth: 800, maxHeight: 800, quality: 85 },
  { size: "preview", maxWidth: 1920, maxHeight: 1920, quality: 90 },
];

export interface GeneratedThumbnail {
  size: "sm" | "md" | "lg" | "preview";
  buffer: Buffer;
  width: number;
  height: number;
  format: "webp";
}

export async function generateThumbnails(
  inputBuffer: Buffer,
  mimeType: string,
): Promise<GeneratedThumbnail[]> {
  if (!mimeType.startsWith("image/")) {
    // For video: extract keyframe with ffmpeg first, then process
    // For documents: convert first page to image, then process
    return [];
  }

  return Promise.all(
    THUMBNAIL_SPECS.map(async (spec) => {
      const result = await sharp(inputBuffer)
        .resize(spec.maxWidth, spec.maxHeight, { fit: "inside", withoutEnlargement: true })
        .webp({ quality: spec.quality })
        .toBuffer({ resolveWithObject: true });

      return {
        size: spec.size,
        buffer: result.data,
        width: result.info.width,
        height: result.info.height,
        format: "webp" as const,
      };
    })
  );
}
```

**Testing**:
- `Unit (fixture): 4000x3000 JPEG → 4 thumbnails at correct max dimensions`
- `Unit (fixture): 100x100 image → thumbnails not enlarged (withoutEnlargement)`
- `Unit (fixture): PNG with alpha → WebP output preserves transparency`
- `Unit: non-image MIME type → empty array returned`
- `Unit: corrupted image buffer → sharp error caught and rethrown with context`

#### 2.4 — Asset Upload API Endpoint

**What**: Multipart upload endpoint that stores the file, extracts metadata, generates thumbnails, and creates the asset record.

**Design**:

```typescript
// POST /api/v1/assets
// Content-Type: multipart/form-data

// Request fields:
//   file: binary (required)
//   title: string (optional; defaults to filename)
//   description: string (optional)
//   brandId: UUID (optional)
//   tags: string[] (optional; JSON array)

// Response 201:
interface AssetUploadResponse {
  id: string;                    // UUID
  title: string;
  fileName: string;
  fileSize: number;
  mimeType: string;
  assetType: "image" | "video" | "document" | "audio" | "3d";
  status: "draft";
  version: 1;
  metadata: Record<string, unknown>;   // extracted IPTC/EXIF/Dublin Core
  dimensions: Record<string, unknown>; // width, height, etc.
  thumbnails: {
    sm: string;                  // presigned URL
    md: string;
    lg: string;
    preview: string;
  };
  createdAt: string;
}

// Upload pipeline (sequential):
// 1. Validate file size and MIME type against tenant config
// 2. Compute SHA-256 checksum
// 3. Determine asset type from MIME type
// 4. Store file in S3 at tenant/asset/v1/filename
// 5. Extract metadata via ExifTool
// 6. Generate thumbnails and store in S3
// 7. Insert asset record + asset_version record in a transaction
// 8. Enqueue AI tagging job (if OPENAI_API_KEY configured)
// 9. Return asset with presigned thumbnail URLs
```

State machine for asset `status`:
```
draft → pending_review → approved → archived
  │         │                         ↑
  └─────────┴── (can also archive) ───┘
  draft → expired (automatic, via license expiry job)
```

**Testing**:
- `Integration: upload valid JPEG → 201 with asset record, thumbnails URLs resolve`
- `Integration: upload 500MB file (exceeds default limit) → 413 Payload Too Large`
- `Integration: upload unsupported MIME type (e.g., .exe) → 415 Unsupported Media Type`
- `Integration: upload with brandId → asset.brandId set correctly`
- `Integration: upload with tags → asset_tags junction records created`
- `Integration: upload duplicate file (same checksum) → new asset created (not deduplicated in MVP)`
- `Integration: upload concurrent requests → no race conditions on version numbering`
- `Unit: MIME type → asset type mapping (image/jpeg → "image", video/mp4 → "video", application/pdf → "document")`

#### 2.5 — Asset Version Upload

**What**: Upload a new version of an existing asset, preserving all previous versions.

**Design**:

```typescript
// POST /api/v1/assets/:assetId/versions
// Content-Type: multipart/form-data

// Request fields:
//   file: binary (required)
//   changeNotes: string (optional)

// Response 201:
interface AssetVersionResponse {
  id: string;
  assetId: string;
  versionNumber: number;
  fileName: string;
  fileSize: number;
  mimeType: string;
  changeNotes: string | null;
  createdAt: string;
}

// Pipeline:
// 1. Verify asset exists and user has upload permission
// 2. Increment version_number (SELECT MAX(version_number) + 1)
// 3. Store file at tenant/asset/vN/filename
// 4. Extract metadata
// 5. Generate thumbnails (replace current set)
// 6. Insert asset_version record
// 7. Update assets.current_version, assets.file_name, etc.
// 8. Log audit event
```

**Testing**:
- `Integration: upload version to existing asset → version_number incremented`
- `Integration: GET asset after version upload → current_version reflects new version`
- `Integration: list asset versions → returns all versions in order`
- `Integration: upload version to nonexistent asset → 404`
- `Integration: version upload preserves previous version's S3 objects`

#### 2.6 — Asset Listing and Detail API

**What**: Paginated asset listing with filtering, and single asset detail endpoint.

**Design**:

```typescript
// GET /api/v1/assets
// Query params:
//   page: number (default 1)
//   limit: number (default 20, max 100)
//   status: string (filter by status)
//   assetType: string (filter by type)
//   brandId: UUID (filter by brand)
//   q: string (full-text search)
//   sort: "created_at" | "updated_at" | "title" | "file_size" (default "created_at")
//   order: "asc" | "desc" (default "desc")

interface AssetListResponse {
  data: AssetSummary[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

interface AssetSummary {
  id: string;
  title: string;
  assetType: string;
  status: string;
  fileName: string;
  fileSize: number;
  mimeType: string;
  thumbnailUrl: string;         // presigned sm thumbnail
  brandId: string | null;
  tagCount: number;
  createdAt: string;
  updatedAt: string;
}

// GET /api/v1/assets/:id
interface AssetDetailResponse {
  id: string;
  title: string;
  description: string | null;
  assetType: string;
  status: string;
  currentVersion: number;
  fileName: string;
  fileSize: number;
  mimeType: string;
  dimensions: Record<string, unknown>;
  metadata: Record<string, unknown>;
  aiProcessing: Record<string, unknown>;
  brand: { id: string; name: string } | null;
  tags: Array<{ id: string; name: string; source: string; confidence: number | null }>;
  thumbnails: Record<string, string>;   // size → presigned URL
  uploadedBy: { id: string; displayName: string };
  versions: AssetVersionSummary[];
  createdAt: string;
  updatedAt: string;
}
```

**Testing**:
- `Integration: list with no assets → empty data array, total 0`
- `Integration: list with 25 assets, page 1 limit 20 → 20 items, totalPages 2`
- `Integration: filter by status="approved" → only approved assets returned`
- `Integration: filter by assetType="image" → only images returned`
- `Integration: full-text search q="summer campaign" → matching assets returned`
- `Integration: sort by file_size desc → largest first`
- `Integration: get asset detail → includes metadata, tags, versions, thumbnails`
- `Integration: get nonexistent asset → 404`
- `Integration: cross-tenant isolation → tenant A cannot see tenant B's assets`

---

## Phase 3: Authentication, RBAC, and Tenant Isolation

### Purpose

Implement user authentication (local + OIDC SSO), role-based access control, and strict tenant data isolation. After this phase, every API endpoint enforces authentication and authorisation. Multi-tenant deployment is secure by default.

### Tasks

#### 3.1 — Session-Based Authentication

**What**: Local email/password authentication with session management using Lucia Auth.

**Design**:

```typescript
// apps/api/src/services/auth.service.ts
import { Lucia } from "lucia";
import { DrizzlePostgreSQLAdapter } from "@lucia-auth/adapter-drizzle";

// Session stored in database (sessions table)
// Cookie: brand_dam_session (httpOnly, secure, sameSite: lax)

// Endpoints:
// POST /api/v1/auth/register
//   Body: { email, password, displayName, tenantSlug }
//   Returns: { user, session }

// POST /api/v1/auth/login
//   Body: { email, password }
//   Returns: { user, session }

// POST /api/v1/auth/logout
//   Deletes session
//   Returns: 204

// GET /api/v1/auth/me
//   Returns: { user } from session

export interface SessionUser {
  id: string;
  tenantId: string;
  email: string;
  displayName: string;
  role: UserRole;
  permissions: string[];
}

export type UserRole = "admin" | "manager" | "editor" | "viewer" | "external";
```

Password hashing: Argon2id via `@node-rs/argon2`.

**Testing**:
- `Integration: register → 201 with user and session cookie set`
- `Integration: login with correct credentials → 200 with session cookie`
- `Integration: login with wrong password → 401`
- `Integration: access protected route without session → 401`
- `Integration: access protected route with valid session → 200`
- `Integration: logout → session cookie cleared, subsequent requests return 401`
- `Unit: password hashed with Argon2id → verify succeeds with correct password`

#### 3.2 — OIDC/SSO Integration

**What**: Enterprise SSO via OpenID Connect (Okta, Azure AD, Google Workspace).

**Design**:

```typescript
// Endpoints:
// GET /api/v1/auth/oidc/authorize
//   Redirects to IdP authorization URL (per RFC 6749)
//   Query: { provider: "okta" | "azure_ad" | "google" }

// GET /api/v1/auth/oidc/callback
//   Handles IdP callback, creates/updates user, creates session
//   Query: { code, state }

// Per-tenant OIDC configuration stored in tenants.settings:
// {
//   "oidc": {
//     "issuer": "https://dev-12345.okta.com",
//     "clientId": "0oa...",
//     "clientSecret": "encrypted...",
//     "scopes": ["openid", "profile", "email"],
//     "autoProvision": true
//   }
// }

// On OIDC callback:
// 1. Exchange code for tokens (RFC 6749 Section 4.1.3)
// 2. Validate ID token (OIDC Core Section 3.1.3.7)
// 3. Extract user info (sub, email, name)
// 4. Find or create user in tenant (if autoProvision enabled)
// 5. Create session
// 6. Redirect to frontend
```

**Testing**:
- `Integration (mocked IdP): OIDC authorize → redirect to IdP URL with correct params`
- `Integration (mocked IdP): OIDC callback with valid code → user created, session set`
- `Integration (mocked IdP): OIDC callback for existing user → user updated, session set`
- `Integration (mocked IdP): OIDC callback with invalid state → 400`
- `Integration (mocked IdP): OIDC with autoProvision=false, unknown user → 403`

#### 3.3 — Role-Based Access Control Middleware

**What**: Permission-checking middleware that enforces RBAC on every route.

**Design**:

```typescript
// Permission hierarchy:
// admin     → all permissions
// manager   → asset.*, brand.*, collection.*, license.*, workflow.*
// editor    → asset.upload, asset.edit, asset.tag, collection.create, collection.edit
// viewer    → asset.view, asset.download, collection.view
// external  → asset.view, asset.download (only for shared/portal assets)

export const PERMISSIONS = {
  ASSET_VIEW: "asset.view",
  ASSET_UPLOAD: "asset.upload",
  ASSET_EDIT: "asset.edit",
  ASSET_DELETE: "asset.delete",
  ASSET_APPROVE: "asset.approve",
  ASSET_TAG: "asset.tag",
  ASSET_DOWNLOAD: "asset.download",
  BRAND_VIEW: "brand.view",
  BRAND_MANAGE: "brand.manage",
  COLLECTION_VIEW: "collection.view",
  COLLECTION_CREATE: "collection.create",
  COLLECTION_EDIT: "collection.edit",
  LICENSE_VIEW: "license.view",
  LICENSE_MANAGE: "license.manage",
  WORKFLOW_VIEW: "workflow.view",
  WORKFLOW_MANAGE: "workflow.manage",
  USER_MANAGE: "user.manage",
  TENANT_ADMIN: "tenant.admin",
} as const;

// Middleware factory:
export function requirePermission(...permissions: string[]) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const user = request.sessionUser;
    if (!user) return reply.status(401).send({ error: "Unauthorized" });
    if (user.role === "admin") return; // admin bypasses
    const hasPermission = permissions.every(p => user.permissions.includes(p));
    if (!hasPermission) return reply.status(403).send({ error: "Forbidden" });
  };
}

// Tenant isolation middleware:
// Automatically adds WHERE tenant_id = ? to all database queries
// via Drizzle's query builder context or manual injection.
export function requireTenant() {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    const user = request.sessionUser;
    if (!user?.tenantId) return reply.status(401).send({ error: "Unauthorized" });
    request.tenantId = user.tenantId;
  };
}
```

**Testing**:
- `Unit: admin user → all permission checks pass`
- `Unit: viewer user → asset.upload check fails with 403`
- `Unit: editor user → asset.upload passes, asset.approve fails`
- `Unit: custom permissions override role defaults`
- `Integration: tenant A user → cannot access tenant B's assets`
- `Integration: unauthenticated request to protected route → 401`
- `Integration: cross-tenant asset access via direct ID → 404 (not 403, to prevent enumeration)`

#### 3.4 — Audit Log Service

**What**: Append-only audit log capturing all state-changing operations for ISO 15489 compliance.

**Design**:

```typescript
// apps/api/src/services/audit.service.ts

export interface AuditEntry {
  tenantId: string;
  actorId: string | null;
  actorType: "user" | "system" | "api_client";
  action: string;              // e.g., "asset.created", "asset.downloaded", "license.expired"
  resourceType: string;        // e.g., "asset", "collection", "brand"
  resourceId: string;
  details: Record<string, unknown>;
  ipAddress?: string;
  userAgent?: string;
}

export async function logAuditEvent(entry: AuditEntry): Promise<void> {
  await db.insert(auditLog).values({
    ...entry,
    createdAt: new Date(),
  });
}

// Audit log query API:
// GET /api/v1/audit
//   Query: resourceType, resourceId, action, actorId, dateFrom, dateTo, page, limit
//   Requires: tenant.admin permission
```

**Testing**:
- `Integration: asset upload → audit entry with action "asset.created"`
- `Integration: asset download → audit entry with action "asset.downloaded" including IP`
- `Integration: query audit log by resourceId → returns matching entries`
- `Integration: query audit log by date range → correct filtering`
- `Integration: non-admin user → 403 on audit log query`

---

## Phase 4: AI-Powered Tagging and Semantic Search

### Purpose

Implement the AI-native core: automatic metadata generation on upload, semantic search across the asset library, and AI-generated descriptions and alt text. This phase delivers the primary differentiator over existing open-source DAM solutions (Piwigo, ResourceSpace).

### Tasks

#### 4.1 — AI Tagging Job

**What**: Background job that sends uploaded images to a multi-modal AI model for automatic tag generation.

**Design**:

```typescript
// apps/api/src/jobs/ai-tag.job.ts
import { Queue, Worker } from "bullmq";

export interface AiTagJobData {
  assetId: string;
  tenantId: string;
  storagePath: string;
  mimeType: string;
}

// AI tagging prompt:
const TAGGING_SYSTEM_PROMPT = `You are a professional digital asset manager. Analyze the image
and return structured metadata in JSON format:
{
  "tags": ["keyword1", "keyword2", ...],       // 10-30 descriptive keywords
  "description": "...",                         // 2-3 sentence description
  "altText": "...",                             // accessibility alt text
  "dominantColours": ["#hex1", "#hex2", ...],   // top 5 dominant colours
  "category": "...",                            // one of: people, product, landscape, abstract, food, architecture, event, document
  "sentiment": "positive|neutral|negative",
  "isAiGenerated": true|false,                  // detect if image appears AI-generated
  "suggestedTitle": "...",                       // concise title if none provided
  "contentWarnings": ["..."]                    // any content that may need review
}
Respond with valid JSON only.`;

// Worker processes:
// 1. Download image from S3
// 2. Resize to max 2048px (reduce API cost)
// 3. Send to OpenAI gpt-4o-mini with vision
// 4. Parse JSON response
// 5. Create tag records (source="ai_vision", with confidence)
// 6. Update asset metadata JSONB (merge AI-generated fields)
// 7. Update asset.aiProcessing state
// 8. Log audit event
```

**Testing**:
- `Integration (mocked OpenAI): upload image → AI tag job enqueued`
- `Integration (mocked OpenAI): AI job processes → tags created with source="ai_vision"`
- `Integration (mocked OpenAI): AI response with 15 tags → 15 asset_tag records`
- `Integration (mocked OpenAI): metadata.description populated from AI response`
- `Unit: invalid AI response (non-JSON) → job retried up to 3 times, then failed`
- `Unit: asset with no OPENAI_API_KEY → job skipped gracefully`
- `Integration: AI tagging on video → keyframe extracted, then tagged`

#### 4.2 — Embedding Generation Job

**What**: Generate vector embeddings for semantic search using text and/or image content.

**Design**:

```typescript
// apps/api/src/jobs/embedding.job.ts

export interface EmbeddingJobData {
  assetId: string;
  tenantId: string;
}

// Pipeline:
// 1. Build text representation: title + description + tags + IPTC caption
// 2. Call OpenAI text-embedding-3-large (1536 dimensions)
// 3. Store in asset_embeddings table (model_name, embedding_type="textual")
// 4. For images: also generate visual embedding from image content
//    (via CLIP model or OpenAI with image input)
// 5. Update asset.aiProcessing.embeddingGenerated = true

// Embedding is regenerated when:
// - Asset tags are modified (manual or AI)
// - Asset description is updated
// - New version uploaded
```

**Testing**:
- `Integration (mocked OpenAI): embedding job → vector(1536) stored in asset_embeddings`
- `Integration: re-tag asset → new embedding generated (replaces old)`
- `Unit: text representation builder → concatenates title, description, tags, caption`
- `Unit: empty asset (no title/description/tags) → still generates embedding from filename`

#### 4.3 — Semantic Search API

**What**: Natural-language search endpoint that combines full-text search (tsvector) with semantic similarity (pgvector cosine distance).

**Design**:

```typescript
// GET /api/v1/search
// Query params:
//   q: string (required; natural language query)
//   type: string[] (optional; filter by asset type)
//   brand: UUID (optional; filter by brand)
//   status: string (optional; filter by status)
//   page: number (default 1)
//   limit: number (default 20)

interface SearchResponse {
  data: SearchResult[];
  pagination: { page: number; limit: number; total: number };
  searchType: "semantic" | "fulltext" | "hybrid";
}

interface SearchResult {
  asset: AssetSummary;
  score: number;              // 0-1 relevance score
  matchType: "semantic" | "fulltext" | "both";
  highlights: string[];       // matched text fragments
}

// Search pipeline:
// 1. Generate embedding for query text
// 2. Execute hybrid query:
//    a. Full-text: ts_rank(search_vector, plainto_tsquery(q))
//    b. Semantic:  1 - (embedding <=> query_embedding)
//    c. Combined score = 0.3 * fulltext_score + 0.7 * semantic_score
// 3. Apply filters (tenant_id, type, brand, status)
// 4. Return ranked results with scores

// SQL (simplified):
// WITH semantic AS (
//   SELECT ae.asset_id,
//          1 - (ae.embedding <=> $query_embedding::vector) AS score
//   FROM asset_embeddings ae
//   JOIN assets a ON a.id = ae.asset_id
//   WHERE a.tenant_id = $tenant_id
//   ORDER BY ae.embedding <=> $query_embedding::vector
//   LIMIT 100
// ),
// fulltext AS (
//   SELECT a.id AS asset_id,
//          ts_rank(a.search_vector, plainto_tsquery($q)) AS score
//   FROM assets a
//   WHERE a.tenant_id = $tenant_id
//     AND a.search_vector @@ plainto_tsquery($q)
// )
// SELECT asset_id,
//        COALESCE(s.score, 0) * 0.7 + COALESCE(f.score, 0) * 0.3 AS combined
// FROM semantic s
// FULL OUTER JOIN fulltext f USING (asset_id)
// ORDER BY combined DESC
// LIMIT $limit OFFSET $offset
```

**Testing**:
- `Integration: search "blue sky landscape" → returns landscape images ranked by relevance`
- `Integration: search with type filter → only matching types returned`
- `Integration: search with no embeddings → falls back to full-text only`
- `Integration: search empty query → 400 Bad Request`
- `Integration: tenant isolation → search only returns current tenant's assets`
- `Unit: score combination → 0.3*fulltext + 0.7*semantic correctly calculated`
- `E2E: upload image of a cat, search "feline" → AI-tagged asset found via semantic similarity`

#### 4.4 — Manual Tag Management API

**What**: CRUD endpoints for manually adding, removing, and editing tags on assets.

**Design**:

```typescript
// POST /api/v1/assets/:id/tags
//   Body: { tags: [{ name: string }] }
//   Creates tags with source="manual"

// DELETE /api/v1/assets/:id/tags/:tagId
//   Removes tag association

// PUT /api/v1/assets/:id/metadata
//   Body: { iptc?: Partial<IptcMetadata>, custom?: Record<string, unknown> }
//   Merges provided fields into asset.metadata JSONB
//   Triggers embedding regeneration job

// GET /api/v1/tags?q=&limit=
//   Autocomplete endpoint for tag names within tenant
```

**Testing**:
- `Integration: add manual tag → asset_tag created with source="manual"`
- `Integration: remove tag → junction record deleted, asset.metadata unchanged`
- `Integration: update IPTC metadata → metadata JSONB merged correctly`
- `Integration: metadata update → embedding regeneration job enqueued`
- `Integration: tag autocomplete "sum" → returns "summer", "summit", etc.`
- `Unit: JSONB merge → nested objects merged, not replaced`

---

## Phase 5: Brand Management and Guidelines

### Purpose

Build the brand management module: brand hierarchy, brand guidelines (colours, fonts, logos, tone of voice), and the public brand guidelines portal. This enables the platform's brand governance value proposition, differentiating it from pure DAM tools.

### Tasks

#### 5.1 — Brand CRUD API

**What**: Create, read, update, and archive brands with parent-child hierarchy support.

**Design**:

```typescript
// Brand entity (from schema):
interface Brand {
  id: string;
  tenantId: string;
  name: string;
  slug: string;
  parentBrandId: string | null;   // sub-brand hierarchy
  isActive: boolean;
  guidelines: BrandGuidelines;    // JSONB
  guidelinePages: GuidelinePage[]; // JSONB array
  createdAt: string;
  updatedAt: string;
}

interface BrandGuidelines {
  colours: BrandColour[];
  fonts: BrandFont[];
  logos: BrandLogo[];
  toneOfVoice?: string;
  dosAndDonts?: string[];
}

interface BrandColour {
  name: string;
  hex: string;
  pantone?: string;
  context: "primary" | "secondary" | "accent" | "neutral";
}

interface BrandFont {
  family: string;
  weight: string;
  context: "heading" | "body" | "caption" | "code";
  fontFileAssetId?: string;
}

interface BrandLogo {
  variant: "full" | "icon" | "monochrome" | "reversed" | "wordmark";
  assetId: string;
  minSizePx: number;
  clearSpacePx: number;
  usageNotes?: string;
}

interface GuidelinePage {
  title: string;
  content: string;           // Markdown
  order: number;
  published: boolean;
}

// Endpoints:
// POST   /api/v1/brands              → create brand
// GET    /api/v1/brands              → list brands (tree structure)
// GET    /api/v1/brands/:id          → brand detail with guidelines
// PUT    /api/v1/brands/:id          → update brand
// DELETE /api/v1/brands/:id          → archive brand (soft delete)
// PUT    /api/v1/brands/:id/guidelines → update guidelines JSONB
// PUT    /api/v1/brands/:id/pages     → update guideline pages
```

**Testing**:
- `Integration: create brand → 201 with slug auto-generated from name`
- `Integration: create sub-brand with parentBrandId → hierarchy established`
- `Integration: list brands → tree structure with nested children`
- `Integration: update guidelines → colours, fonts, logos persisted in JSONB`
- `Integration: duplicate slug within tenant → 409 Conflict`
- `Integration: archive brand → isActive=false, assets still accessible`
- `Unit: slug generation → "Acme Corp" → "acme-corp"`

#### 5.2 — Brand Guidelines Portal (Public)

**What**: Server-rendered public portal displaying brand guidelines, accessible without authentication via a shareable URL.

**Design**:

```typescript
// URL pattern: /portal/:brand-slug
// Rendered with Next.js SSR (App Router)

// Page structure:
// - Brand header (name, logo, description)
// - Sidebar navigation (guideline pages)
// - Colour palette (interactive swatches with hex/RGB/Pantone)
// - Typography specimens
// - Logo usage guide with download buttons
// - Custom guideline pages (markdown rendered)

// Components:
// - ColourSwatch: displays colour with copy-to-clipboard for hex/RGB
// - FontSpecimen: renders sample text in the brand's fonts
// - LogoCard: shows logo variant with min size, clear space overlay
// - GuidelinePageRenderer: markdown to HTML with syntax highlighting

// Data fetching:
// - getStaticParams() generates paths for all active brands
// - ISR with 5-minute revalidation
// - Public route — no authentication required
// - 404 if brand not found or not active
```

**Testing**:
- `E2E: visit /portal/acme-corp → brand guidelines page renders`
- `E2E: colour swatches display correct hex values`
- `E2E: click colour swatch → hex copied to clipboard`
- `E2E: logo download button → file downloaded`
- `E2E: nonexistent brand slug → 404 page`
- `E2E: inactive brand → 404 page`
- `Unit: markdown renderer → headings, lists, images, code blocks rendered correctly`

#### 5.3 — Asset-Brand Association and Filtering

**What**: Associate assets with brands and filter the asset library by brand.

**Design**:

```typescript
// PUT /api/v1/assets/:id
//   Body: { brandId: UUID | null }
//   Associates or disassociates an asset from a brand

// GET /api/v1/assets?brandId=UUID
//   Filters asset list by brand

// GET /api/v1/brands/:id/assets
//   Lists all assets for a brand (paginated)
//   Includes sub-brand assets if ?includeSubBrands=true

// Sub-brand asset query:
// WITH RECURSIVE brand_tree AS (
//   SELECT id FROM brands WHERE id = $brandId
//   UNION ALL
//   SELECT b.id FROM brands b
//   JOIN brand_tree bt ON b.parent_brand_id = bt.id
// )
// SELECT a.* FROM assets a
// WHERE a.brand_id IN (SELECT id FROM brand_tree)
```

**Testing**:
- `Integration: associate asset with brand → brandId set on asset`
- `Integration: list assets by brand → only matching assets returned`
- `Integration: includeSubBrands=true → includes sub-brand assets`
- `Integration: disassociate → brandId set to null`
- `Integration: delete brand with assets → assets retain brandId (soft delete)`

---

## Phase 6: Rights Management and Licensing

### Purpose

Implement digital rights management: license tracking, expiry monitoring, model/property releases, and automated expiry alerts. This addresses one of the most underserved areas identified in the feature survey (features.md) — most DAMs treat rights as an afterthought.

### Tasks

#### 6.1 — License CRUD API

**What**: Create, read, update, and track licenses associated with assets, supporting PLUS vocabulary and Creative Commons.

**Design**:

```typescript
// License entity (from schema):
interface AssetLicense {
  id: string;
  assetId: string;
  licenseType: "royalty_free" | "rights_managed" | "creative_commons" | "editorial" | "custom";
  status: "active" | "expired" | "revoked" | "pending";
  terms: LicenseTerms;          // JSONB — flexible per license type
  licenseStart: string | null;   // ISO date
  licenseExpiry: string | null;  // ISO date
  isPerpetual: boolean;
  maxUses: number | null;
  currentUses: number;
  alertDays: number;             // days before expiry to send alert
  notes: string | null;
  createdAt: string;
  updatedAt: string;
}

// PLUS-aligned terms (JSONB):
interface PlusTerms {
  plusLicenseId: string;
  licensor: { name: string; id?: string };
  licensee: { name: string };
  usageType: string;          // PLUS usage vocabulary
  usageMedia: string;         // PLUS media type
  territory: string;          // ISO 3166-1 or "WW"
  exclusivity: boolean;
}

// Creative Commons terms:
interface CreativeCommonsTerms {
  ccCode: string;             // "CC-BY-4.0", "CC-BY-SA-4.0"
  ccUrl: string;
  attributionText: string;
}

// Endpoints:
// POST   /api/v1/assets/:assetId/licenses   → create license
// GET    /api/v1/assets/:assetId/licenses   → list licenses for asset
// GET    /api/v1/licenses                    → list all licenses (dashboard)
// GET    /api/v1/licenses/:id               → license detail
// PUT    /api/v1/licenses/:id               → update license
// DELETE /api/v1/licenses/:id               → revoke license

// GET /api/v1/licenses?status=expiring_soon
//   Returns licenses expiring within alertDays
```

**Testing**:
- `Integration: create PLUS license → terms stored as JSONB with PLUS fields`
- `Integration: create CC license → ccCode and ccUrl stored`
- `Integration: list expiring licenses → returns those within alertDays of expiry`
- `Integration: revoke license → status set to "revoked", audit event logged`
- `Integration: increment currentUses on download → counter updated`
- `Integration: maxUses reached → asset status changed to "expired"`
- `Unit: PLUS terms validation → missing licensor rejected`

#### 6.2 — Model and Property Release Tracking

**What**: Track model and property releases linked to assets, with status tracking and file upload.

**Design**:

```typescript
// Release entity (unified model + property):
interface Release {
  id: string;
  assetId: string;
  releaseType: "model" | "property";
  subjectName: string;          // person name or property name
  status: "obtained" | "pending" | "not_required" | "denied" | "expired";
  releaseFilePath: string | null; // S3 path to signed release document
  details: {
    signedDate?: string;
    expiryDate?: string;
    notes?: string;
    territory?: string;
  };
  createdAt: string;
  updatedAt: string;
}

// Endpoints:
// POST   /api/v1/assets/:assetId/releases
// GET    /api/v1/assets/:assetId/releases
// PUT    /api/v1/releases/:id
// DELETE /api/v1/releases/:id

// POST /api/v1/releases/:id/upload
//   Upload signed release document (PDF/image)
```

**Testing**:
- `Integration: create model release → record created with status "pending"`
- `Integration: upload release document → file stored, releaseFilePath set`
- `Integration: release with expiryDate in past → status auto-set to "expired"`
- `Integration: list releases for asset → grouped by releaseType`
- `Unit: release status transitions validated (cannot go from "denied" to "obtained" directly)`

#### 6.3 — License Expiry Monitoring Job

**What**: Scheduled job that scans for expiring/expired licenses and sends notifications.

**Design**:

```typescript
// apps/api/src/jobs/license-expiry.job.ts
// Runs via BullMQ repeatable job: every 6 hours

// Pipeline:
// 1. Query licenses WHERE license_expiry <= now() + alert_days AND status = 'active'
// 2. For each expiring license:
//    a. If expiry <= now(): set status = 'expired', set asset.status = 'expired'
//    b. If expiry within alert_days: log warning, create compliance_check record
// 3. Trigger webhook events: 'license.expiring', 'license.expired'
// 4. Log audit events

// Also checks model/property releases for expiry
```

**Testing**:
- `Integration: license expired yesterday → status set to "expired", asset status updated`
- `Integration: license expiring in 15 days (alert_days=30) → compliance_check created`
- `Integration: perpetual license → never flagged as expiring`
- `Integration: already-expired license → not re-processed`
- `Unit: job handles empty results gracefully`

---

## Phase 7: Collections, Workflows, and Collaboration

### Purpose

Build asset organisation (collections, smart collections), approval workflows, and sharing. After this phase, teams can collaborate on asset curation, run approval processes, and share curated asset sets with external partners.

### Tasks

#### 7.1 — Collections API

**What**: Manual and smart (filter-based) collections for organising assets.

**Design**:

```typescript
interface Collection {
  id: string;
  tenantId: string;
  name: string;
  description: string | null;
  collectionType: "manual" | "smart";
  smartFilter: SmartFilter | null;  // JSONB — only for smart collections
  coverAssetId: string | null;
  isPublic: boolean;
  createdBy: string;
  createdAt: string;
  updatedAt: string;
}

interface SmartFilter {
  assetType?: string[];
  status?: string[];
  brandId?: string;
  tags?: string[];                // match any
  metadataMatch?: Record<string, string>; // JSONB path → value
  uploadedAfter?: string;
  uploadedBefore?: string;
}

// Endpoints:
// POST   /api/v1/collections
// GET    /api/v1/collections
// GET    /api/v1/collections/:id
// PUT    /api/v1/collections/:id
// DELETE /api/v1/collections/:id
// POST   /api/v1/collections/:id/assets    → add assets (manual only)
// DELETE /api/v1/collections/:id/assets/:assetId → remove asset
// GET    /api/v1/collections/:id/assets    → list assets in collection

// Smart collection: assets resolved at query time by applying smartFilter
// Manual collection: assets stored in collection_assets junction table
```

**Testing**:
- `Integration: create manual collection → add 3 assets → list returns 3`
- `Integration: create smart collection with tag filter → assets matching tag returned`
- `Integration: add asset matching smart filter tag → auto-included in results`
- `Integration: remove asset from manual collection → no longer in list`
- `Integration: cannot add assets to smart collection → 400`
- `Integration: public collection → accessible without auth via share link`

#### 7.2 — Approval Workflow Engine

**What**: Configurable multi-step approval workflows for assets.

**Design**:

```typescript
// Workflow entity:
interface Workflow {
  id: string;
  tenantId: string;
  assetId: string;
  workflowType: "approval" | "review" | "publish";
  status: "pending" | "in_progress" | "approved" | "rejected" | "cancelled";
  config: WorkflowConfig;       // JSONB — step definitions
  currentStep: number;
  initiatedBy: string;
  completedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

interface WorkflowConfig {
  steps: WorkflowStep[];
}

interface WorkflowStep {
  name: string;
  approverRoles: string[];       // roles that can act on this step
  approverUserIds?: string[];    // specific users (optional)
  requiredApprovals: number;     // e.g., 2 of 3 approvers
  autoApproveAfterDays?: number; // auto-approve if no action
}

// State machine:
// pending → in_progress (first step begins)
// in_progress → approved (all steps approved)
// in_progress → rejected (any step rejected)
// in_progress → cancelled (initiator cancels)
// approved → asset.status set to "approved"
// rejected → asset.status remains "pending_review"

// Endpoints:
// POST   /api/v1/assets/:assetId/workflows    → start workflow
// GET    /api/v1/workflows                     → list workflows (dashboard)
// GET    /api/v1/workflows/:id                → workflow detail with history
// POST   /api/v1/workflows/:id/actions        → approve/reject/request_changes
//   Body: { action: "approve" | "reject" | "request_changes", comment?: string }
// POST   /api/v1/workflows/:id/cancel         → cancel workflow
```

**Testing**:
- `Integration: start approval workflow → status "in_progress", currentStep 0`
- `Integration: approve step 1 → currentStep advances to 1`
- `Integration: approve final step → workflow status "approved", asset status "approved"`
- `Integration: reject any step → workflow status "rejected"`
- `Integration: non-approver tries to approve → 403`
- `Integration: cancel workflow → status "cancelled"`
- `Unit: requiredApprovals=2 → step not complete until 2 approvals received`
- `Unit: auto-approve after 3 days → step approved if no action`

#### 7.3 — Share Links

**What**: Generate time-limited, optionally password-protected share links for assets and collections.

**Design**:

```typescript
// POST /api/v1/share-links
//   Body: {
//     targetType: "asset" | "collection",
//     targetId: UUID,
//     expiresInDays?: number,        // default: 7
//     maxDownloads?: number,
//     passwordProtected?: boolean,
//     password?: string,
//     allowedFormats?: string[]       // restrict download formats
//   }
//   Returns: { token, url, expiresAt }

// GET /share/:token
//   Public route — no auth required
//   If password protected → prompt for password
//   Returns asset/collection detail with download links

// Share link URL format: https://dam.example.com/share/{token}
```

**Testing**:
- `Integration: create share link → token generated, URL returned`
- `Integration: access share link → asset details returned`
- `Integration: expired share link → 410 Gone`
- `Integration: max downloads reached → 410 Gone`
- `Integration: password-protected link without password → 401`
- `Integration: password-protected link with correct password → access granted`
- `Integration: share link for collection → all collection assets accessible`

---

## Phase 8: Compliance and Brand Governance

### Purpose

Implement automated brand compliance scanning, off-brand detection, and compliance dashboards. This delivers on the governance differentiator identified in the research — continuous AI-powered brand compliance that most DAMs lack.

### Tasks

#### 8.1 — Brand Compliance Scanner

**What**: Background job that scans assets for brand guideline violations (colour, typography, logo usage).

**Design**:

```typescript
// apps/api/src/jobs/compliance-scan.job.ts

export interface ComplianceScanJobData {
  assetId: string;
  tenantId: string;
  brandId: string;
  checkTypes: ("colour" | "typography" | "logo" | "content")[];
}

// Checks performed:
// 1. COLOUR CHECK:
//    - Extract dominant colours from image
//    - Compare against brand colour palette
//    - Flag colours that don't match any brand colour within deltaE threshold
//    - Severity: "warning" if close match, "violation" if no match

// 2. LOGO CHECK:
//    - Detect brand logos in image (via AI vision)
//    - Check if logo is below minimum size
//    - Check if clear space is violated
//    - Flag outdated logo versions
//    - Severity: "violation" for wrong logo version

// 3. CONTENT CHECK (AI-powered):
//    - Send image to AI with brand tone of voice guidelines
//    - Ask: "Does this asset align with the brand guidelines?"
//    - Flag off-brand content with explanation
//    - Severity: "warning" for questionable, "violation" for clearly off-brand

// Results stored in compliance_checks table:
interface ComplianceCheck {
  id: string;
  assetId: string;
  checkType: "colour" | "typography" | "logo" | "content" | "rights_expiry";
  severity: "info" | "warning" | "violation";
  message: string;
  details: Record<string, unknown>;  // check-specific data
  isResolved: boolean;
  resolvedBy: string | null;
  resolvedAt: string | null;
  createdAt: string;
}
```

**Testing**:
- `Integration (mocked AI): image with off-brand colours → compliance_check with check_type="colour"`
- `Integration (mocked AI): image with undersized logo → violation created`
- `Integration: resolve compliance check → isResolved=true, resolvedBy set`
- `Integration: rescan asset → new checks created, old unresolved remain`
- `Unit: deltaE colour comparison → correct threshold applied`
- `Unit: asset without brand → compliance scan skipped`

#### 8.2 — Compliance Dashboard API

**What**: API endpoints for viewing compliance status across the asset library.

**Design**:

```typescript
// GET /api/v1/compliance/summary
//   Returns: {
//     totalAssets: number,
//     compliant: number,
//     withWarnings: number,
//     withViolations: number,
//     unchecked: number,
//     byCheckType: { colour: {...}, typography: {...}, logo: {...}, content: {...} }
//   }

// GET /api/v1/compliance/violations
//   Query: checkType, severity, isResolved, brandId, page, limit
//   Returns paginated compliance checks

// GET /api/v1/assets/:id/compliance
//   Returns all compliance checks for an asset

// POST /api/v1/compliance/:id/resolve
//   Body: { notes?: string }
//   Marks compliance check as resolved
```

**Testing**:
- `Integration: summary with mix of compliant/violated assets → correct counts`
- `Integration: filter violations by checkType → only matching type returned`
- `Integration: resolve violation → reflected in summary counts`
- `Integration: brand-level filtering → only violations for that brand's assets`

#### 8.3 — Content Provenance (C2PA) Integration

**What**: Read and write C2PA content credentials manifests for asset provenance tracking.

**Design**:

```typescript
// apps/api/src/services/provenance.service.ts
import { createC2pa } from "c2pa-node";

export interface ProvenanceInfo {
  hasManifest: boolean;
  isValid: boolean;
  claimGenerator: string;
  assertions: C2paAssertion[];
  signatureInfo: {
    algorithm: string;
    issuer: string;
    signedAt: string;
  };
}

// On asset upload:
// 1. Check if uploaded file contains C2PA manifest
// 2. If yes: validate manifest, extract assertions, store in asset_provenance
// 3. Display provenance info in asset detail

// On asset export (when configured):
// 1. Create new C2PA manifest
// 2. Add assertions (c2pa.actions, stds.iptc, stds.exif)
// 3. Sign with platform certificate
// 4. Embed manifest in exported file

// Endpoints:
// GET /api/v1/assets/:id/provenance
//   Returns provenance chain for asset

// POST /api/v1/assets/:id/provenance/verify
//   Re-validates C2PA manifest
```

**Testing**:
- `Integration (fixture): upload image with C2PA manifest → provenance extracted and stored`
- `Integration (fixture): verify valid manifest → is_valid=true`
- `Integration (fixture): verify tampered image → is_valid=false`
- `Integration: asset without manifest → hasManifest=false`
- `Unit: C2PA assertion parsing → correct extraction of action types`

---

## Phase 9: Distribution and CDN

### Purpose

Build asset distribution capabilities: format conversion, CDN delivery, and partner/external portals. This phase turns the DAM from an internal storage system into a content supply chain hub.

### Tasks

#### 9.1 — Asset Transformation and Download API

**What**: On-demand asset format conversion and resizing for download.

**Design**:

```typescript
// GET /api/v1/assets/:id/download
// Query params:
//   format: "original" | "jpeg" | "png" | "webp" | "avif" | "pdf"
//   width: number (optional; max dimension)
//   height: number (optional; max dimension)
//   quality: number (optional; 1-100, default 85)
//   dpi: number (optional; for print)
//
// Response: binary file stream with appropriate Content-Type and
//   Content-Disposition: attachment; filename="asset-name.format"
//
// Pipeline:
// 1. Check download permission (RBAC + license usage check)
// 2. Increment license currentUses (if tracked)
// 3. If format=original: stream from S3
// 4. If transformation needed:
//    a. Check transform cache (Redis key: transform:{assetId}:{params_hash})
//    b. If cached: stream from S3 cache path
//    c. If not cached: transform via Sharp, store in cache path, stream
// 5. Log audit event (asset.downloaded)

// Transformation cache key:
// {tenant_id}/{asset_id}/transforms/{width}x{height}_{quality}_{dpi}.{format}
```

**Testing**:
- `Integration: download original → file matches uploaded content`
- `Integration: download as WebP → correct Content-Type, valid WebP file`
- `Integration: download with width=800 → image resized correctly`
- `Integration: download increments license currentUses`
- `Integration: second identical transform → served from cache`
- `Integration: download without permission → 403`
- `Integration: download expired asset → 403 with descriptive message`
- `Unit: params hash → deterministic for same parameters`

#### 9.2 — Webhook System

**What**: Outbound webhook notifications for asset lifecycle events.

**Design**:

```typescript
// Webhook entity:
interface Webhook {
  id: string;
  tenantId: string;
  url: string;
  events: string[];              // ["asset.created", "asset.approved", "license.expiring"]
  secretHash: string;            // HMAC signing secret
  isActive: boolean;
  lastTriggered: string | null;
  failureCount: number;
}

// Webhook payload:
interface WebhookPayload {
  event: string;
  timestamp: string;
  tenantId: string;
  data: Record<string, unknown>;  // event-specific
}

// Delivery:
// 1. Compute HMAC-SHA256 signature of payload using webhook secret
// 2. POST to webhook URL with headers:
//    X-BrandDAM-Signature: sha256={signature}
//    X-BrandDAM-Event: {event_type}
//    Content-Type: application/json
// 3. Retry with exponential backoff: 10s, 30s, 90s, 270s (max 4 retries)
// 4. After 5 consecutive failures: deactivate webhook, notify admin

// Endpoints:
// POST   /api/v1/webhooks
// GET    /api/v1/webhooks
// PUT    /api/v1/webhooks/:id
// DELETE /api/v1/webhooks/:id
// POST   /api/v1/webhooks/:id/test   → sends test event

// Events:
// asset.created, asset.updated, asset.approved, asset.archived,
// asset.downloaded, license.expiring, license.expired,
// compliance.violation, workflow.completed
```

**Testing**:
- `Integration: create webhook → stored with hashed secret`
- `Integration: asset created → webhook POST sent with correct signature`
- `Integration: webhook endpoint returns 500 → retry scheduled`
- `Integration: 5 consecutive failures → webhook deactivated`
- `Integration: test webhook → test event delivered`
- `Unit: HMAC signature → matches expected value for known payload+secret`
- `Unit: exponential backoff → correct delay sequence`

#### 9.3 — External Partner Portal

**What**: Authenticated portal for external partners (agencies, freelancers) with restricted access to designated collections.

**Design**:

```typescript
// Partner portal uses the same Next.js app with "external" role
// External users can:
// - View collections shared with them
// - Download assets from shared collections
// - Upload assets to designated upload collections
// - Cannot: manage brands, approve workflows, view audit logs

// Endpoints for portal management:
// POST /api/v1/portals
//   Body: {
//     name: string,
//     collections: UUID[],         // which collections to expose
//     allowUpload: boolean,
//     uploadCollectionId?: UUID,
//     branding: { logoUrl, primaryColour, welcomeMessage }
//   }
//   Returns portal config

// POST /api/v1/portals/:id/invite
//   Body: { email: string, displayName: string }
//   Creates external user with portal access
//   Sends invitation email
```

**Testing**:
- `Integration: create portal with 2 collections → portal config stored`
- `Integration: external user login → can only see portal collections`
- `Integration: external user download → asset downloaded, audit logged`
- `Integration: external user tries to access admin → 403`
- `Integration: external user upload (if allowed) → asset in upload collection`
- `E2E: partner portal renders with custom branding`

---

## Phase 10: Video and Document Support

### Purpose

Extend the asset pipeline to handle video (transcoding, thumbnail extraction, transcription) and documents (PDF preview, page extraction). This broadens the platform from image-centric to truly multi-format, matching the capabilities of MediaValet and Brandfolder.

### Tasks

#### 10.1 — Video Transcoding Pipeline

**What**: Transcode uploaded videos to web-friendly formats and generate preview thumbnails.

**Design**:

```typescript
// apps/api/src/jobs/video-transcode.job.ts

export interface VideoTranscodeJobData {
  assetId: string;
  tenantId: string;
  storagePath: string;
}

// Pipeline:
// 1. Download video from S3
// 2. Extract technical metadata (duration, codec, resolution, fps, bitrate)
// 3. Generate thumbnails: keyframe at 25%, 50%, 75% of duration
// 4. Transcode to:
//    - HLS adaptive streaming (for preview): 720p, 480p, 360p
//    - MP4 web preview: 720p H.264 AAC
// 5. Upload transcoded files to S3
// 6. Update asset.dimensions with video metadata
// 7. Enqueue AI tagging job with keyframe image

// ffmpeg command templates:
// Keyframe: ffmpeg -i input.mp4 -vf "select=eq(n\,{frame})" -frames:v 1 thumb.jpg
// Web preview: ffmpeg -i input.mp4 -c:v libx264 -crf 23 -preset medium
//              -c:a aac -b:a 128k -movflags +faststart -vf scale=-2:720 output.mp4
```

**Testing**:
- `Integration (fixture): upload MP4 → transcoded to 720p, thumbnails generated`
- `Integration: video metadata extracted → duration, codec, resolution stored`
- `Integration: keyframe thumbnails → 3 images at correct timestamps`
- `Unit: unsupported video codec → error logged, original preserved`
- `Unit: video shorter than 3s → single thumbnail at midpoint`

#### 10.2 — Document Preview Generation

**What**: Generate image previews of document pages (PDF, DOCX) for in-browser viewing.

**Design**:

```typescript
// Pipeline (PDF):
// 1. Use pdf-lib or pdf2pic to render each page as an image
// 2. Store page images in S3: {tenant}/{asset}/pages/page-{N}.webp
// 3. Update asset.dimensions with pageCount
// 4. First page becomes the asset thumbnail

// Pipeline (DOCX/PPTX):
// 1. Convert to PDF via LibreOffice (soffice --headless --convert-to pdf)
// 2. Follow PDF pipeline

// GET /api/v1/assets/:id/pages
//   Returns: { pageCount: number, pages: [{ number, thumbnailUrl, previewUrl }] }
```

**Testing**:
- `Integration (fixture): upload 5-page PDF → 5 page previews generated`
- `Integration: GET pages → correct pageCount and URLs`
- `Integration: first page → used as asset thumbnail`
- `Unit: encrypted PDF → error with descriptive message`

---

## Phase 11: Facial Recognition and Talent Management

### Purpose

Add facial recognition capabilities for automatically identifying people in images and linking them to model releases and consent records. This addresses GDPR requirements for biometric data and the talent management use case critical for media companies. Only activated when `ENABLE_FACIAL_RECOGNITION=true`.

### Tasks

#### 11.1 — Face Detection and Encoding

**What**: Detect faces in images and generate face encodings for matching.

**Design**:

```typescript
// apps/api/src/services/face-detection.service.ts

// Uses a face detection model (face-api.js with TensorFlow.js, or
// external API like Amazon Rekognition abstracted behind interface)

export interface FaceDetectionResult {
  faces: DetectedFace[];
}

export interface DetectedFace {
  boundingBox: { x: number; y: number; width: number; height: number }; // normalised 0-1
  confidence: number;
  encoding: number[];           // 128-dimensional face encoding
  matchedPersonId?: string;     // if matched to known person
  matchedPersonName?: string;
}

// Pipeline (triggered after AI tagging):
// 1. Check ENABLE_FACIAL_RECOGNITION flag
// 2. Download image from S3
// 3. Run face detection model
// 4. For each detected face:
//    a. Generate 128-dim face encoding
//    b. Compare against known_persons encodings (cosine similarity > 0.85)
//    c. If match found: link to known person
//    d. Store in detected_faces table
// 5. Update asset.aiProcessing.facesDetected count
```

**Testing**:
- `Integration (fixture): photo with 2 faces → 2 detected_face records`
- `Integration: face matches known person → matchedPersonId set`
- `Integration: unknown face → matchedPersonId null`
- `Integration: ENABLE_FACIAL_RECOGNITION=false → job skipped`
- `Unit: cosine similarity threshold → 0.85 correctly discriminates`
- `Unit: image with no faces → empty results, no error`

#### 11.2 — Known Persons and Consent Management

**What**: Manage known persons with reference face encodings and GDPR consent tracking.

**Design**:

```typescript
// GDPR consent states:
// pending → consented (person gives consent for biometric processing)
// consented → withdrawn (person withdraws consent)
// withdrawn: all face encodings for this person must be deleted

// Endpoints:
// POST   /api/v1/persons                  → register known person
// GET    /api/v1/persons                  → list known persons
// GET    /api/v1/persons/:id              → person detail with consent status
// PUT    /api/v1/persons/:id              → update person info
// POST   /api/v1/persons/:id/consent      → record consent
// POST   /api/v1/persons/:id/withdraw     → withdraw consent (triggers data deletion)

// GET /api/v1/persons/:id/assets
//   Returns all assets featuring this person (via detected_faces)
```

**Testing**:
- `Integration: register person with reference face → encoding stored`
- `Integration: new upload with matching face → auto-linked to person`
- `Integration: withdraw consent → face encodings deleted, detected_faces updated`
- `Integration: list assets for person → correct assets returned`
- `Integration: consent status transitions → only valid transitions allowed`

---

## Phase 12: Generative Variants and Advanced AI

### Purpose

Deliver the advanced AI capabilities that differentiate this platform: generative asset variant creation, LLM-powered contract extraction, AI brand assistant, and unknown asset discovery. These are the features identified as underserved opportunities in the competitive landscape.

### Tasks

#### 12.1 — Generative Asset Variant Creation

**What**: AI-powered generation of channel-optimised asset variants (resize, reformat, copy adaptation).

**Design**:

```typescript
// POST /api/v1/assets/:id/variants
//   Body: {
//     channels: Array<{
//       name: string,               // "Instagram Story", "LinkedIn Post", "Email Header"
//       width: number,
//       height: number,
//       format: "jpeg" | "png" | "webp",
//       cropStrategy: "smart" | "center" | "face",    // "smart" uses AI for composition
//       overlayText?: string,        // optional text overlay
//     }>
//   }
//   Returns: {
//     variants: Array<{
//       channel: string,
//       assetId: string,             // new asset linked as derivative
//       thumbnailUrl: string,
//     }>
//   }

// Pipeline:
// 1. Download original asset
// 2. For each channel:
//    a. Apply smart crop (AI-guided focal point detection)
//    b. Resize to target dimensions
//    c. Apply text overlay if specified
//    d. Store as new asset with derivative relationship
//    e. Copy parent's metadata, adjust dimensions
//    f. Link via collection or tag
// 3. All variants share parent's license
```

**Testing**:
- `Integration: generate Instagram variant → new asset at 1080x1920`
- `Integration: smart crop with face → face kept in frame`
- `Integration: variant linked as derivative of parent`
- `Integration: variant inherits parent license`
- `Unit: crop strategy "center" → correct crop rectangle`

#### 12.2 — LLM-Powered License Contract Extraction

**What**: Extract licensing terms from uploaded contract documents using LLM.

**Design**:

```typescript
// POST /api/v1/licenses/extract
//   Content-Type: multipart/form-data
//   Fields: contractFile (PDF or image)
//   Returns: {
//     extractedTerms: {
//       licenseType: string,
//       licensor: string,
//       licensee: string,
//       territory: string,
//       startDate: string,
//       expiryDate: string,
//       usageType: string,
//       maxUses: number | null,
//       restrictions: string[],
//       confidence: number,          // 0-1 extraction confidence
//     },
//     rawText: string,              // OCR'd text from contract
//   }

// Pipeline:
// 1. If PDF: extract text via pdf-parse
// 2. If image: OCR via Tesseract.js or AI vision
// 3. Send extracted text to GPT-4o with structured output
// 4. Return extracted terms for user review before saving
```

**Testing**:
- `Integration (mocked AI): upload sample license PDF → terms extracted correctly`
- `Integration: extracted terms applied to asset → license record created`
- `Unit: confidence below 0.5 → warning flag on result`
- `Unit: non-contract document → error "Could not identify license terms"`

#### 12.3 — Unknown Asset Discovery

**What**: Surface assets that exist in the library but have never been downloaded, searched for, or included in collections.

**Design**:

```typescript
// GET /api/v1/assets/discover
// Query:
//   strategy: "unused" | "similar_to" | "trending" | "expiring"
//   similarTo?: UUID (for "similar_to" strategy)
//   limit: number (default 20)

// "unused" strategy:
// SELECT a.* FROM assets a
// LEFT JOIN audit_log al ON al.resource_id = a.id
//   AND al.action = 'asset.downloaded'
// WHERE al.id IS NULL
//   AND a.status = 'approved'
//   AND a.created_at < now() - interval '30 days'
// ORDER BY a.created_at ASC
// LIMIT $limit

// "similar_to" strategy:
// Use pgvector similarity to find related but overlooked assets

// "trending" strategy:
// Assets with highest download count in last 7 days
```

**Testing**:
- `Integration: unused strategy → returns assets never downloaded`
- `Integration: similar_to strategy → returns semantically similar assets`
- `Integration: trending strategy → returns most-downloaded assets`
- `Integration: empty library → empty results`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Storage & Ingest             ─── requires Phase 1
    │
Phase 3: Auth, RBAC, Tenancy          ─── requires Phase 1 (can parallel Phase 2 after 1.4)
    │
    ├─── Phase 4: AI Tagging & Search  ─── requires Phase 2 + Phase 3
    │
    ├─── Phase 5: Brand Management     ─── requires Phase 2 + Phase 3
    │        │
    │        └─── Phase 8: Compliance  ─── requires Phase 5 + Phase 4
    │
    ├─── Phase 6: Rights Management    ─── requires Phase 2 + Phase 3
    │        │
    │        └─── Phase 11: Facial Recognition ─── requires Phase 4 + Phase 6
    │
    ├─── Phase 7: Collections & Workflows ─── requires Phase 2 + Phase 3
    │        │
    │        └─── Phase 9: Distribution & CDN  ─── requires Phase 7 + Phase 6
    │
    └─── Phase 10: Video & Documents   ─── requires Phase 2 (can parallel Phases 4-7)
              │
              └─── Phase 12: Advanced AI ─── requires Phase 4 + Phase 6 + Phase 8

Parallelism opportunities:
  - Phases 4, 5, 6, 7 can be developed concurrently after Phase 3
  - Phase 10 can be developed concurrently with Phases 4-7 (only needs Phase 2)
  - Phases 8, 9, 11 depend on their prerequisites but are independent of each other
  - Phase 12 is the final convergence point
```

---

## Definition of Done (per phase)

1. All tasks implemented and code compiles without errors.
2. All unit tests pass (`pnpm test --filter api`).
3. All integration tests pass (requires Docker Compose services running).
4. ESLint passes with zero warnings (`pnpm lint`).
5. Prettier formatting applied (`pnpm format:check` passes).
6. TypeScript strict mode passes (`pnpm typecheck`).
7. Docker build succeeds (`docker build -t brand-dam .`).
8. All new API endpoints appear in the auto-generated OpenAPI spec at `/docs/json`.
9. Database migrations generated and applied cleanly (`pnpm db:migrate`).
10. New environment variables documented in `.env.example`.
11. Manual smoke test confirms feature works end-to-end in Docker Compose environment.
12. Audit log entries generated for all state-changing operations.
13. Tenant isolation verified — no cross-tenant data leakage in new endpoints.
