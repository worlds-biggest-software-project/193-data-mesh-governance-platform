# Data Mesh Governance Platform — Phased Development Plan

> Project: 193-data-mesh-governance-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (backend) | TypeScript (Node.js 22+) | Strong typing for large API surfaces; excellent OpenAPI codegen tooling; async I/O for concurrent lineage ingestion and webhook processing; same language for API and potential frontend reduces context switching |
| API framework | Fastify + @fastify/swagger | Fastest Node.js HTTP framework; native OpenAPI 3.1 schema generation from route schemas; plugin architecture matches the modular domain design; first-class TypeScript support |
| Database | PostgreSQL 16 | Required for JSONB (hybrid model), GIN indexes, partitioning, recursive CTEs for lineage, and `tsvector` full-text search; mature, battle-tested for governance audit workloads |
| ORM / Query builder | Drizzle ORM | Type-safe SQL with zero runtime overhead; native PostgreSQL JSON operators; migration tooling; avoids the abstraction leaks of Prisma for complex queries |
| Task queue | BullMQ (Redis-backed) | Handles async workloads: OpenLineage event ingestion, quality check scheduling, AI contract generation, webhook delivery; delayed jobs for SLA monitoring; rate limiting for LLM calls |
| Cache | Redis 7 | Session cache, API response cache, rate limiting, and BullMQ backing store in a single service |
| Search | PostgreSQL tsvector + pg_trgm | Avoids Elasticsearch operational overhead for MVP; GIN indexes on tsvector columns provide adequate full-text search; trigram indexes for fuzzy matching; upgrade path to Elasticsearch later |
| Frontend | Next.js 15 (App Router) | React Server Components for dashboard pages; API route handlers for BFF pattern; TailwindCSS + shadcn/ui for rapid UI development; same TypeScript ecosystem |
| AI / LLM | Anthropic Claude API (via @anthropic-ai/sdk) | Contract generation, NLP policy authoring, semantic search embeddings; prompt caching for repeated schema analysis; tool use for structured output |
| Embeddings | Anthropic Voyager 3 | Semantic search over data products and glossary terms; stored in PostgreSQL with pgvector extension |
| Authentication | OIDC via next-auth (Auth.js v5) | Supports enterprise SSO (Okta, Azure AD, Google Workspace); JWT session tokens; API key fallback for service-to-service |
| Containerisation | Docker + docker-compose | Self-hosted deployment target; single `docker compose up` for dev; production-ready multi-service orchestration |
| Testing | Vitest + Supertest + Testcontainers | Vitest for unit/integration (fast, ESM-native); Supertest for HTTP assertions; Testcontainers for real PostgreSQL/Redis in CI |
| Code quality | ESLint 9 (flat config) + Prettier + typescript-eslint | Consistent style enforcement; strict TypeScript checking |
| Package manager | pnpm | Workspace support for monorepo (api, web, shared); faster installs; strict dependency resolution |
| API documentation | OpenAPI 3.1 (auto-generated) | @fastify/swagger generates spec from route schemas; Scalar UI for interactive docs |
| Data contracts | ODCS v3.1.0 (YAML/JSON) | Industry standard for data contract definition; validated against JSON Schema at ingestion |
| Lineage format | OpenLineage 1.x (JSON) | LF AI & Data standard; direct ingestion from Airflow, Spark, dbt without transformation |
| MCP server | @modelcontextprotocol/sdk | Expose catalog to AI coding assistants; standard protocol for LLM tool integration |

### Project Structure

```
data-mesh-governance-platform/
├── pnpm-workspace.yaml
├── package.json
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── drizzle.config.ts
├── turbo.json
│
├── packages/
│   └── shared/                        # Shared types, schemas, utilities
│       ├── package.json
│       └── src/
│           ├── types/                  # Domain types (DataProduct, Contract, etc.)
│           ├── schemas/                # Zod schemas for validation
│           ├── constants/              # Enums, status codes, dimensions
│           └── utils/                  # Date, UUID, hashing utilities
│
├── apps/
│   ├── api/                           # Fastify REST API
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── server.ts              # Fastify app bootstrap
│   │   │   ├── config.ts              # Environment config with defaults
│   │   │   ├── plugins/               # Fastify plugins (auth, swagger, error handler)
│   │   │   ├── routes/                # Route handlers grouped by domain
│   │   │   │   ├── domains/
│   │   │   │   ├── data-products/
│   │   │   │   ├── contracts/
│   │   │   │   ├── lineage/
│   │   │   │   ├── policies/
│   │   │   │   ├── glossary/
│   │   │   │   ├── quality/
│   │   │   │   ├── access/
│   │   │   │   └── search/
│   │   │   ├── services/              # Business logic layer
│   │   │   ├── repositories/          # Database access layer
│   │   │   ├── workers/               # BullMQ job processors
│   │   │   ├── ai/                    # LLM integration (contract gen, policy authoring)
│   │   │   ├── ingestion/             # OpenLineage event processing
│   │   │   └── mcp/                   # MCP server implementation
│   │   └── tests/
│   │       ├── unit/
│   │       ├── integration/
│   │       └── fixtures/
│   │
│   └── web/                           # Next.js frontend
│       ├── package.json
│       ├── src/
│       │   ├── app/                   # App Router pages
│       │   │   ├── (auth)/            # Login, callback
│       │   │   ├── (dashboard)/       # Main authenticated layout
│       │   │   │   ├── domains/
│       │   │   │   ├── products/
│       │   │   │   ├── contracts/
│       │   │   │   ├── lineage/
│       │   │   │   ├── quality/
│       │   │   │   └── settings/
│       │   │   └── api/               # BFF API routes
│       │   ├── components/            # React components
│       │   │   ├── ui/                # shadcn/ui primitives
│       │   │   ├── domains/
│       │   │   ├── products/
│       │   │   ├── contracts/
│       │   │   ├── lineage/
│       │   │   └── quality/
│       │   ├── lib/                   # Client utilities, API client
│       │   └── hooks/                 # React hooks
│       └── tests/
│
├── db/
│   ├── migrations/                    # Drizzle migration files
│   ├── schema/                        # Drizzle schema definitions
│   │   ├── domains.ts
│   │   ├── data-products.ts
│   │   ├── contracts.ts
│   │   ├── lineage.ts
│   │   ├── policies.ts
│   │   ├── glossary.ts
│   │   ├── quality.ts
│   │   ├── access.ts
│   │   ├── audit.ts
│   │   └── search.ts
│   └── seed/                          # Development seed data
│
└── docs/
    ├── api-guide.md
    ├── deployment.md
    └── data-contracts.md
```

---

## Phase 1: Foundation & Project Scaffold

### Purpose

Establish the monorepo structure, configure tooling (TypeScript, ESLint, Docker, CI), create the database schema, and produce a running but minimal API server that responds to health checks. After this phase, any developer can clone, `docker compose up`, and have a working development environment with database migrations applied.

### Tasks

#### 1.1 — Monorepo Scaffold and Tooling

**What**: Create the pnpm workspace with `api`, `web`, and `shared` packages, configure TypeScript, ESLint, Prettier, and Turbo.

**Design**:

```typescript
// pnpm-workspace.yaml
packages:
  - 'packages/*'
  - 'apps/*'

// tsconfig.base.json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "outDir": "./dist",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "resolveJsonModule": true
  }
}

// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": {}
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors`
- `Unit: pnpm run build succeeds for all packages`
- `Unit: pnpm run lint passes with zero warnings`
- `Unit: pnpm run typecheck passes`

#### 1.2 — Docker Compose Development Environment

**What**: Create docker-compose.yml with PostgreSQL 16, Redis 7, and the API service with hot reload.

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: datamesh
      POSTGRES_USER: datamesh
      POSTGRES_PASSWORD: dev_password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U datamesh"]
      interval: 5s
      timeout: 3s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  api:
    build:
      context: .
      target: development
    ports:
      - "3001:3001"
    environment:
      DATABASE_URL: postgres://datamesh:dev_password@postgres:5432/datamesh
      REDIS_URL: redis://redis:6379
      NODE_ENV: development
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./apps/api/src:/app/apps/api/src
      - ./packages:/app/packages

volumes:
  pgdata:
```

```typescript
// apps/api/src/config.ts
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3001),
  HOST: z.string().default('0.0.0.0'),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),
  JWT_SECRET: z.string().min(32).default('dev-secret-change-in-production-min-32-chars'),
  ANTHROPIC_API_KEY: z.string().optional(),
});

export type Config = z.infer<typeof envSchema>;
export const config = envSchema.parse(process.env);
```

**Testing**:
- `Integration: docker compose up starts all services healthy within 30s`
- `Integration: PostgreSQL accepts connections on port 5432`
- `Integration: Redis responds to PING on port 6379`
- `Unit: config.ts throws ZodError for missing DATABASE_URL`
- `Unit: config.ts applies defaults for optional fields`

#### 1.3 — Database Schema (Hybrid Relational + JSONB)

**What**: Implement the database schema using Drizzle ORM, based on Data Model Suggestion 3 (Hybrid Relational + JSONB) for its balance of query efficiency, standards fidelity, and rapid development.

**Design**:

```typescript
// db/schema/domains.ts
import { pgTable, uuid, varchar, text, timestamp, jsonb, index } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  externalId: varchar('external_id', { length: 255 }).unique(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  status: varchar('status', { length: 50 }).notNull().default('active'),
  profile: jsonb('profile').notNull().default({}),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const teams = pgTable('teams', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).notNull().unique(),
  contact: jsonb('contact').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const teamMembers = pgTable('team_members', {
  teamId: uuid('team_id').notNull().references(() => teams.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 50 }).notNull().default('member'),
  joinedAt: timestamp('joined_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  pk: { columns: [t.teamId, t.userId] },
}));

export const domains = pgTable('domains', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).notNull().unique(),
  description: text('description'),
  parentId: uuid('parent_id').references((): any => domains.id),
  ownerTeamId: uuid('owner_team_id').notNull().references(() => teams.id),
  status: varchar('status', { length: 50 }).notNull().default('active'),
  config: jsonb('config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  parentIdx: index('idx_domains_parent').on(t.parentId),
  slugIdx: index('idx_domains_slug').on(t.slug),
}));
```

```typescript
// db/schema/data-products.ts
import { pgTable, uuid, varchar, text, timestamp, jsonb, smallint, index, unique } from 'drizzle-orm/pg-core';
import { domains, teams } from './domains';

export const dataProducts = pgTable('data_products', {
  id: uuid('id').primaryKey().defaultRandom(),
  domainId: uuid('domain_id').notNull().references(() => domains.id),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).notNull(),
  description: text('description'),
  version: varchar('version', { length: 50 }).notNull().default('1.0.0'),
  status: varchar('status', { length: 50 }).notNull().default('draft'),
  visibility: varchar('visibility', { length: 50 }).notNull().default('domain'),
  ownerTeamId: uuid('owner_team_id').notNull().references(() => teams.id),
  dcatMetadata: jsonb('dcat_metadata').notNull().default({}),
  fairScores: jsonb('fair_scores').notNull().default({}),
  jurisdictions: jsonb('jurisdictions').notNull().default([]),
  distributions: jsonb('distributions').notNull().default([]),
  customMetadata: jsonb('custom_metadata').notNull().default({}),
  publishedAt: timestamp('published_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  domainSlugUnique: unique().on(t.domainId, t.slug),
  domainIdx: index('idx_products_domain').on(t.domainId),
  statusIdx: index('idx_products_status').on(t.status),
  ownerIdx: index('idx_products_owner').on(t.ownerTeamId),
}));
```

```typescript
// db/schema/contracts.ts
import { pgTable, uuid, varchar, text, date, timestamp, jsonb, integer, index, unique } from 'drizzle-orm/pg-core';
import { dataProducts } from './data-products';
import { teams, users } from './domains';

export const dataContracts = pgTable('data_contracts', {
  id: uuid('id').primaryKey().defaultRandom(),
  dataProductId: uuid('data_product_id').notNull().references(() => dataProducts.id),
  version: varchar('version', { length: 50 }).notNull(),
  status: varchar('status', { length: 50 }).notNull().default('draft'),
  ownerTeamId: uuid('owner_team_id').notNull().references(() => teams.id),
  contractBody: jsonb('contract_body').notNull(),  // Full ODCS v3.1.0 contract
  effectiveFrom: date('effective_from'),
  effectiveTo: date('effective_to'),
  schemaHash: varchar('schema_hash', { length: 64 }),
  qualityRuleCount: integer('quality_rule_count').default(0),
  slaCount: integer('sla_count').default(0),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  productVersionUnique: unique().on(t.dataProductId, t.version),
  productIdx: index('idx_contracts_product').on(t.dataProductId),
  statusIdx: index('idx_contracts_status').on(t.status),
}));

export const contractSubscriptions = pgTable('contract_subscriptions', {
  id: uuid('id').primaryKey().defaultRandom(),
  contractId: uuid('contract_id').notNull().references(() => dataContracts.id),
  consumerTeamId: uuid('consumer_team_id').notNull().references(() => teams.id),
  purpose: text('purpose'),
  status: varchar('status', { length: 50 }).notNull().default('pending'),
  approvedBy: uuid('approved_by').references(() => users.id),
  approvedAt: timestamp('approved_at', { withTimezone: true }),
  accessConfig: jsonb('access_config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => ({
  contractIdx: index('idx_subscriptions_contract').on(t.contractId),
  consumerIdx: index('idx_subscriptions_consumer').on(t.consumerTeamId),
}));
```

**Testing**:
- `Integration: drizzle-kit push applies schema without errors`
- `Integration: all tables created with correct columns and constraints`
- `Unit: inserting a data_product with invalid status throws constraint violation`
- `Unit: unique constraint on (domain_id, slug) prevents duplicates`
- `Integration: JSONB GIN indexes created and usable for containment queries`

#### 1.4 — Minimal Fastify API Server

**What**: Bootstrap Fastify with health check, CORS, error handling, request logging, and OpenAPI documentation generation.

**Design**:

```typescript
// apps/api/src/server.ts
import Fastify from 'fastify';
import cors from '@fastify/cors';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';
import { config } from './config';

export async function buildApp() {
  const app = Fastify({
    logger: {
      level: config.LOG_LEVEL,
      transport: config.NODE_ENV === 'development'
        ? { target: 'pino-pretty' }
        : undefined,
    },
  });

  await app.register(cors, { origin: true });

  await app.register(swagger, {
    openapi: {
      info: {
        title: 'Data Mesh Governance Platform API',
        version: '0.1.0',
        description: 'API for managing data products, contracts, lineage, and governance policies',
      },
      servers: [{ url: `http://localhost:${config.PORT}` }],
      components: {
        securitySchemes: {
          bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
          apiKey: { type: 'apiKey', in: 'header', name: 'X-API-Key' },
        },
      },
    },
  });

  await app.register(swaggerUi, { routePrefix: '/docs' });

  // Health check
  app.get('/health', {
    schema: {
      response: {
        200: {
          type: 'object',
          properties: {
            status: { type: 'string', enum: ['ok'] },
            timestamp: { type: 'string', format: 'date-time' },
            version: { type: 'string' },
          },
        },
      },
    },
  }, async () => ({
    status: 'ok',
    timestamp: new Date().toISOString(),
    version: '0.1.0',
  }));

  return app;
}
```

**Testing**:
- `Integration: GET /health returns 200 with { status: "ok" }`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: GET /docs/json returns OpenAPI 3.1 spec`
- `Unit: buildApp() registers all plugins without throwing`
- `Unit: unknown routes return 404 with structured error body`
- `Unit: malformed JSON body returns 400 with validation message`

---

## Phase 2: Domain & Team Management

### Purpose

Implement the foundational domain and team management — creating, updating, and querying domains and teams. This is prerequisite for all subsequent features since every data product, contract, and policy belongs to a domain and is owned by a team.

### Tasks

#### 2.1 — Domain CRUD API

**What**: REST endpoints for creating, listing, updating, and archiving domains with hierarchical parent-child relationships.

**Design**:

```typescript
// packages/shared/src/types/domain.ts
export interface Domain {
  id: string;
  name: string;
  slug: string;
  description: string | null;
  parentId: string | null;
  ownerTeamId: string;
  status: 'active' | 'archived' | 'draft';
  config: DomainConfig;
  createdAt: string;
  updatedAt: string;
}

export interface DomainConfig {
  governanceLevel?: 'strict' | 'standard' | 'light';
  defaultClassification?: 'public' | 'internal' | 'confidential' | 'restricted';
  requiredContractSections?: ('schema' | 'quality' | 'sla')[];
  approvalWorkflow?: 'single_person' | 'two_person' | 'committee';
}

export interface CreateDomainRequest {
  name: string;
  slug?: string;      // Auto-generated from name if omitted
  description?: string;
  parentId?: string;
  ownerTeamId: string;
  config?: DomainConfig;
}

export interface UpdateDomainRequest {
  name?: string;
  description?: string;
  parentId?: string | null;
  ownerTeamId?: string;
  status?: 'active' | 'archived' | 'draft';
  config?: Partial<DomainConfig>;
}

// API Routes:
// POST   /api/v1/domains         → Create domain
// GET    /api/v1/domains         → List domains (with pagination, filtering)
// GET    /api/v1/domains/:id     → Get domain by ID
// PATCH  /api/v1/domains/:id     → Update domain
// GET    /api/v1/domains/:id/children → List child domains
// GET    /api/v1/domains/:id/tree     → Full subtree (recursive)
```

**Testing**:
- `Unit: createDomain generates slug from name ("Finance Data" → "finance-data")`
- `Unit: createDomain rejects duplicate slug`
- `Unit: updateDomain with parentId creating circular reference → 400 error`
- `Integration: POST /api/v1/domains creates domain and returns 201 with Location header`
- `Integration: GET /api/v1/domains returns paginated list with total count`
- `Integration: GET /api/v1/domains?status=active filters correctly`
- `Integration: GET /api/v1/domains/:id/tree returns nested hierarchy`
- `Integration: PATCH /api/v1/domains/:id with status=archived cascades to child domains`

#### 2.2 — Team & User Management

**What**: CRUD for teams, team membership, and user profiles. Users are created on first SSO login; teams are created manually by admins.

**Design**:

```typescript
// packages/shared/src/types/team.ts
export interface Team {
  id: string;
  name: string;
  slug: string;
  contact: TeamContact;
  memberCount?: number;
  createdAt: string;
  updatedAt: string;
}

export interface TeamContact {
  email?: string;
  slack?: string;
  pagerduty?: string;
  msTeams?: string;
}

export interface TeamMember {
  userId: string;
  teamId: string;
  role: 'owner' | 'maintainer' | 'member' | 'viewer';
  user?: UserSummary;
  joinedAt: string;
}

export interface UserSummary {
  id: string;
  email: string;
  displayName: string;
  avatarUrl?: string;
}

// API Routes:
// POST   /api/v1/teams              → Create team
// GET    /api/v1/teams              → List teams
// GET    /api/v1/teams/:id          → Get team with members
// PATCH  /api/v1/teams/:id          → Update team
// POST   /api/v1/teams/:id/members  → Add member
// DELETE /api/v1/teams/:id/members/:userId → Remove member
// PATCH  /api/v1/teams/:id/members/:userId → Change role
// GET    /api/v1/users/me           → Current user profile
```

**Testing**:
- `Unit: createTeam generates slug from name`
- `Unit: addMember with non-existent userId returns 404`
- `Unit: removeMember when user is sole owner returns 400 (must have at least one owner)`
- `Integration: POST /api/v1/teams creates team, returns 201`
- `Integration: POST /api/v1/teams/:id/members adds user with default role "member"`
- `Integration: GET /api/v1/teams/:id includes memberCount`
- `Integration: PATCH role from member to owner succeeds`

#### 2.3 — Audit Log Foundation

**What**: Implement a cross-cutting audit log that records all write operations with actor, action, target, and change details.

**Design**:

```typescript
// apps/api/src/services/audit.service.ts
export interface AuditEntry {
  actorId: string | null;
  actorType: 'user' | 'system' | 'api_key' | 'service';
  action: string;          // 'domain.created', 'team.member_added', etc.
  targetType: string;      // 'domain', 'team', 'data_product', etc.
  targetId: string;
  details: Record<string, unknown>;  // { before: {...}, after: {...} }
  ipAddress?: string;
}

export class AuditService {
  async log(entry: AuditEntry): Promise<void>;
  async query(params: {
    targetType?: string;
    targetId?: string;
    actorId?: string;
    action?: string;
    from?: Date;
    to?: Date;
    limit?: number;
    offset?: number;
  }): Promise<{ entries: AuditEntry[]; total: number }>;
}

// Fastify plugin that auto-audits route handlers via onResponse hook
// Routes opt-in by setting `config: { audit: { action: 'domain.created', targetType: 'domain' } }`
```

**Testing**:
- `Unit: AuditService.log writes entry to audit_log table`
- `Unit: AuditService.query filters by targetType and date range`
- `Integration: creating a domain auto-generates audit entry with action "domain.created"`
- `Integration: updating a domain records before/after in details JSONB`
- `Integration: audit entries include requester IP from X-Forwarded-For`

---

## Phase 3: Data Product Registry

### Purpose

Implement the core data product lifecycle — registration, versioning, publication, deprecation, and discovery. This is the heart of the data mesh platform: domain teams publish data products, and consumers discover and understand them.

### Tasks

#### 3.1 — Data Product CRUD

**What**: Full lifecycle management for data products including creation, versioning, publication, and deprecation.

**Design**:

```typescript
// packages/shared/src/types/data-product.ts
export interface DataProduct {
  id: string;
  domainId: string;
  name: string;
  slug: string;
  description: string | null;
  version: string;
  status: 'draft' | 'published' | 'deprecated' | 'archived';
  visibility: 'private' | 'domain' | 'organisation' | 'public';
  ownerTeamId: string;
  dcatMetadata: DcatMetadata;
  fairScores: FairScores;
  jurisdictions: string[];    // ISO 3166-1 alpha-2 codes
  distributions: Distribution[];
  customMetadata: Record<string, unknown>;
  publishedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface DcatMetadata {
  '@type'?: string;
  'dcat:theme'?: string[];
  'dcat:keyword'?: string[];
  'dcat:landingPage'?: string;
  'dcterms:accrualPeriodicity'?: string;
  'dcterms:spatial'?: string;
  'dcterms:temporal'?: { 'dcat:startDate'?: string; 'dcat:endDate'?: string };
}

export interface FairScores {
  findable?: number;    // 0-100
  accessible?: number;
  interoperable?: number;
  reusable?: number;
}

export interface Distribution {
  name: string;
  type: 'api' | 'file' | 'stream' | 'database' | 'delta_sharing';
  format?: string;
  url?: string;
  mediaType?: string;
}

export interface CreateDataProductRequest {
  domainId: string;
  name: string;
  slug?: string;
  description?: string;
  ownerTeamId: string;
  visibility?: DataProduct['visibility'];
  dcatMetadata?: DcatMetadata;
  jurisdictions?: string[];
  distributions?: Distribution[];
  customMetadata?: Record<string, unknown>;
}

// API Routes:
// POST   /api/v1/data-products                → Register new data product
// GET    /api/v1/data-products                → List (paginated, filtered)
// GET    /api/v1/data-products/:id            → Get by ID
// PATCH  /api/v1/data-products/:id            → Update
// POST   /api/v1/data-products/:id/publish    → Transition to published
// POST   /api/v1/data-products/:id/deprecate  → Transition to deprecated
// GET    /api/v1/domains/:id/data-products    → List products in domain
```

**Testing**:
- `Unit: createDataProduct validates jurisdictions are ISO 3166-1 alpha-2 codes`
- `Unit: publish transition from "draft" → "published" succeeds`
- `Unit: publish transition from "archived" → "published" fails (invalid transition)`
- `Unit: deprecate requires successor_product_id or reason`
- `Integration: POST /api/v1/data-products creates product with generated slug, returns 201`
- `Integration: GET /api/v1/data-products?domainId=X returns only products in that domain`
- `Integration: GET /api/v1/data-products?jurisdictions=DE returns products with JSONB containment`
- `Integration: POST /api/v1/data-products/:id/publish sets publishedAt timestamp`
- `Fixture: seed 20 data products across 4 domains for pagination testing`

#### 3.2 — Data Product Versioning

**What**: Track version history with changelog and breaking change detection.

**Design**:

```typescript
// packages/shared/src/types/data-product-version.ts
export interface DataProductVersion {
  id: string;
  dataProductId: string;
  version: string;           // semver
  changelog: string | null;
  isBreaking: boolean;
  publishedBy: string | null;
  publishedAt: string;
}

// Service logic:
// - When a data product is published, a version record is created
// - Breaking changes detected by comparing schema_hash of active contract
// - Version number must be valid semver and greater than the current version
// - Breaking version increments major; non-breaking increments minor/patch

// API Routes:
// GET    /api/v1/data-products/:id/versions   → List all versions
// POST   /api/v1/data-products/:id/versions   → Create new version (triggers publish)
```

**Testing**:
- `Unit: version "2.0.0" after "1.5.0" succeeds`
- `Unit: version "1.0.0" after "1.5.0" fails (must be greater)`
- `Unit: version "not-semver" fails validation`
- `Unit: breaking change detected when schema fields are removed`
- `Integration: creating a version updates the data_product.version field`
- `Integration: GET /api/v1/data-products/:id/versions returns ordered list`

#### 3.3 — Data Product Search

**What**: Unified keyword and semantic search across all data products using PostgreSQL full-text search with tsvector.

**Design**:

```typescript
// db/schema/search.ts
export const searchIndex = pgTable('search_index', {
  id: uuid('id').primaryKey().defaultRandom(),
  entityType: varchar('entity_type', { length: 100 }).notNull(),
  entityId: uuid('entity_id').notNull(),
  domainId: uuid('domain_id'),
  title: varchar('title', { length: 500 }).notNull(),
  description: text('description'),
  tags: text('tags').array(),
  ownerTeam: varchar('owner_team', { length: 255 }),
  status: varchar('status', { length: 50 }),
  searchVector: /* tsvector column - raw SQL */,
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Trigger to auto-update search_vector:
// CREATE FUNCTION update_search_vector() RETURNS trigger AS $$
// BEGIN
//   NEW.search_vector :=
//     setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
//     setweight(to_tsvector('english', COALESCE(NEW.description, '')), 'B') ||
//     setweight(to_tsvector('english', COALESCE(array_to_string(NEW.tags, ' '), '')), 'C');
//   RETURN NEW;
// END;
// $$ LANGUAGE plpgsql;

// API:
// GET /api/v1/search?q=revenue&type=data_product&domain=finance
export interface SearchRequest {
  q: string;                    // Search query
  type?: string[];              // Filter by entity type
  domain?: string;              // Filter by domain
  status?: string[];            // Filter by status
  tags?: string[];              // Filter by tags
  limit?: number;               // Default 20, max 100
  offset?: number;
}

export interface SearchResult {
  entityType: string;
  entityId: string;
  title: string;
  description: string | null;
  domainId: string | null;
  ownerTeam: string | null;
  status: string | null;
  tags: string[];
  rank: number;                 // ts_rank score
}
```

**Testing**:
- `Unit: search for "revenue" matches product titled "Quarterly Revenue Report"`
- `Unit: search for "revnue" (typo) matches via pg_trgm similarity`
- `Unit: search with type=data_product filters out other entity types`
- `Unit: search with domain filter restricts to that domain`
- `Integration: creating a data product auto-inserts into search_index`
- `Integration: updating a data product's name updates the search_index entry`
- `Integration: search results ordered by ts_rank descending`
- `Fixture: seed 50 products with varied names and descriptions for relevance testing`

---

## Phase 4: Data Contract Management

### Purpose

Implement the full data contract lifecycle using ODCS v3.1.0: authoring, validation, activation, versioning, and consumer subscriptions. After this phase, producers can define contracts for their data products, and consumers can subscribe with purpose-bound access.

### Tasks

#### 4.1 — Contract Authoring and Validation

**What**: Create and validate data contracts conforming to ODCS v3.1.0 specification.

**Design**:

```typescript
// packages/shared/src/types/contract.ts
export interface DataContract {
  id: string;
  dataProductId: string;
  version: string;
  status: 'draft' | 'active' | 'deprecated' | 'violated';
  ownerTeamId: string;
  contractBody: OdcsContract;    // Full ODCS v3.1.0 structure
  effectiveFrom: string | null;
  effectiveTo: string | null;
  schemaHash: string | null;
  qualityRuleCount: number;
  slaCount: number;
  createdAt: string;
  updatedAt: string;
}

// ODCS v3.1.0 contract structure (subset - key fields)
export interface OdcsContract {
  apiVersion: string;            // "v3.1.0"
  kind: 'DataContract';
  id: string;                    // URN
  info: {
    title: string;
    version: string;
    description?: string;
    owner: { team: string; email?: string };
  };
  terms?: {
    usage?: string;
    limitations?: string;
    billing?: string;
  };
  schema: OdcsSchemaObject[];
  quality?: OdcsQualityRule[];
  sla?: OdcsSla[];
}

export interface OdcsSchemaObject {
  name: string;
  type: 'table' | 'topic' | 'file' | 'api' | 'view';
  description?: string;
  properties: OdcsSchemaProperty[];
}

export interface OdcsSchemaProperty {
  name: string;
  type: string;              // logical type
  physicalType?: string;
  required?: boolean;
  primaryKey?: boolean;
  pii?: boolean;
  classification?: 'public' | 'internal' | 'confidential' | 'restricted';
  description?: string;
}

export interface OdcsQualityRule {
  dimension: string;          // ISO 25012: accuracy, completeness, consistency, freshness
  rule: string;
  threshold?: number;
  unit?: 'percent' | 'count' | 'minutes' | 'hours';
  severity?: 'info' | 'warning' | 'error' | 'critical';
}

export interface OdcsSla {
  metric: string;             // availability, freshness, latency, completeness
  target: number;
  unit: string;
  window: 'hourly' | 'daily' | 'weekly' | 'monthly';
}

// Validation service validates contractBody against ODCS JSON Schema
// apps/api/src/services/contract-validation.service.ts
export class ContractValidationService {
  validate(contract: unknown): { valid: boolean; errors: ValidationError[] };
  computeSchemaHash(schema: OdcsSchemaObject[]): string;  // SHA-256 of canonical JSON
  detectBreakingChanges(oldSchema: OdcsSchemaObject[], newSchema: OdcsSchemaObject[]): BreakingChange[];
}

export interface BreakingChange {
  type: 'field_removed' | 'type_changed' | 'required_added' | 'object_removed';
  path: string;
  description: string;
}

// API Routes:
// POST   /api/v1/contracts                    → Create contract
// GET    /api/v1/contracts/:id                → Get contract
// PATCH  /api/v1/contracts/:id                → Update draft contract
// POST   /api/v1/contracts/:id/activate       → Activate contract
// POST   /api/v1/contracts/:id/deprecate      → Deprecate contract
// POST   /api/v1/contracts/validate           → Validate contract body without saving
// GET    /api/v1/data-products/:id/contracts  → List contracts for a product
```

**Testing**:
- `Unit: validate rejects contract missing "apiVersion" field`
- `Unit: validate rejects contract with unknown quality dimension`
- `Unit: validate accepts well-formed ODCS v3.1.0 contract`
- `Unit: computeSchemaHash is deterministic for same input`
- `Unit: detectBreakingChanges identifies removed field as breaking`
- `Unit: detectBreakingChanges identifies type change as breaking`
- `Unit: detectBreakingChanges allows adding optional field as non-breaking`
- `Integration: POST /api/v1/contracts with valid body returns 201`
- `Integration: POST /api/v1/contracts with invalid body returns 400 with validation errors`
- `Integration: POST /api/v1/contracts/:id/activate sets status=active, effectiveFrom=today`
- `Fixture: 3 valid ODCS contracts and 2 invalid ones in test fixtures`

#### 4.2 — Contract Subscriptions and Consumer Management

**What**: Allow consumer teams to subscribe to contracts, with approval workflows and purpose-bound access tracking.

**Design**:

```typescript
// packages/shared/src/types/subscription.ts
export interface ContractSubscription {
  id: string;
  contractId: string;
  consumerTeamId: string;
  purpose: string | null;
  status: 'pending' | 'approved' | 'rejected' | 'revoked';
  approvedBy: string | null;
  approvedAt: string | null;
  accessConfig: AccessConfig;
  createdAt: string;
  updatedAt: string;
}

export interface AccessConfig {
  level?: 'read' | 'write';
  expiresAt?: string;
  ipWhitelist?: string[];
  rateLimitPerMinute?: number;
}

export interface CreateSubscriptionRequest {
  contractId: string;
  consumerTeamId: string;
  purpose: string;
  accessConfig?: AccessConfig;
}

// Approval flow:
// 1. Consumer creates subscription → status = 'pending'
// 2. Producer team (contract owner) receives notification
// 3. Producer approves/rejects → status = 'approved'/'rejected'
// 4. If domain config requires two_person approval, two distinct approvers needed

// API Routes:
// POST   /api/v1/subscriptions                     → Request subscription
// GET    /api/v1/subscriptions                     → List (filter by team, contract, status)
// POST   /api/v1/subscriptions/:id/approve         → Approve
// POST   /api/v1/subscriptions/:id/reject          → Reject (with reason)
// POST   /api/v1/subscriptions/:id/revoke          → Revoke access
// GET    /api/v1/contracts/:id/subscribers          → List subscribers for contract
// GET    /api/v1/teams/:id/subscriptions            → List subscriptions for team
```

**Testing**:
- `Unit: createSubscription sets status to "pending"`
- `Unit: approve by non-owner-team member returns 403`
- `Unit: approve sets approvedAt to current timestamp`
- `Unit: reject requires reason field`
- `Unit: revoke transitions from approved to revoked`
- `Integration: POST subscription triggers notification event (verified via event listener)`
- `Integration: GET /api/v1/contracts/:id/subscribers returns approved subscribers`
- `Integration: expired subscriptions auto-transition to revoked via scheduled job`

#### 4.3 — Contract Breach Detection

**What**: Monitor active contracts for SLA violations and emit breach events when quality or freshness targets are missed.

**Design**:

```typescript
// apps/api/src/services/breach-detection.service.ts
export class BreachDetectionService {
  // Called by quality check worker after each check run
  async evaluateQualityResult(
    contractId: string,
    results: QualityCheckResult
  ): Promise<BreachEvent[]>;

  // Checks all active SLAs for time-based violations (freshness, availability)
  async evaluateTimeSlas(): Promise<BreachEvent[]>;

  // Notify subscribed consumers of breaches
  async notifyConsumers(breachEvent: BreachEvent): Promise<void>;
}

export interface QualityCheckResult {
  contractId: string;
  checkTime: string;
  results: {
    totalRules: number;
    passed: number;
    failed: number;
    details: RuleResult[];
  };
  overallPassed: boolean;
  runDurationMs: number;
}

export interface RuleResult {
  dimension: string;
  rule: string;
  passed: boolean;
  actualValue: number;
  threshold?: number;
  unit?: string;
}

export interface BreachEvent {
  contractId: string;
  slaMetric: string;
  actualValue: number;
  targetValue: number;
  breachTime: string;
  severity: 'warning' | 'error' | 'critical';
  affectedConsumers: string[];  // team IDs
}

// BullMQ scheduled job runs every 5 minutes for SLA evaluation
// Quality check results flow in via POST /api/v1/quality/results
```

**Testing**:
- `Unit: evaluateQualityResult with all rules passing → no breach events`
- `Unit: evaluateQualityResult with freshness > threshold → breach event with correct severity`
- `Unit: evaluateTimeSlas detects contract with no quality check in > SLA window`
- `Unit: notifyConsumers sends to all approved subscribers`
- `Integration: quality result submission triggers breach detection asynchronously`
- `Integration: breach event changes contract status to "violated"`
- `Integration: breach notification recorded in audit log`

---

## Phase 5: OpenLineage Ingestion & Lineage Graph

### Purpose

Implement ingestion of OpenLineage events, build the lineage graph (datasets, jobs, column-level dependencies), and expose lineage query APIs. After this phase, organisations can connect their data pipelines (Airflow, Spark, dbt) and get automated lineage.

### Tasks

#### 5.1 — OpenLineage Event Ingestion Endpoint

**What**: HTTP endpoint that accepts OpenLineage RunEvent payloads, validates them, and stores them with extracted indexes.

**Design**:

```typescript
// apps/api/src/ingestion/openlineage.handler.ts
// POST /api/v1/lineage/events (OpenLineage-compatible endpoint)

export interface OpenLineageRunEvent {
  eventType: 'START' | 'RUNNING' | 'COMPLETE' | 'ABORT' | 'FAIL' | 'OTHER';
  eventTime: string;
  run: {
    runId: string;
    facets?: Record<string, unknown>;
  };
  job: {
    namespace: string;
    name: string;
    facets?: Record<string, unknown>;
  };
  inputs?: OpenLineageDataset[];
  outputs?: OpenLineageDataset[];
}

export interface OpenLineageDataset {
  namespace: string;
  name: string;
  facets?: {
    schema?: { fields: { name: string; type: string }[] };
    columnLineage?: {
      fields: Record<string, { inputFields: { namespace: string; name: string; field: string }[] }>;
    };
    dataQualityMetrics?: Record<string, unknown>;
  };
}

// Ingestion pipeline:
// 1. Validate event against OpenLineage JSON Schema
// 2. Extract job_namespace, job_name, run_id, event_type for relational indexing
// 3. Extract input_datasets and output_datasets as JSONB arrays
// 4. Store full event_body as JSONB (preserving all facets)
// 5. Upsert into datasets table for known dataset tracking
// 6. If column lineage facet present, extract to column_lineage records
// 7. Enqueue async job to link datasets to known data products

// Rate limiting: 1000 events/second per namespace
// Batch endpoint: POST /api/v1/lineage/events/batch (array of events)
```

**Testing**:
- `Unit: valid OpenLineage COMPLETE event → stored with correct indexes`
- `Unit: event missing required "eventType" field → 400 validation error`
- `Unit: event with column lineage facet → column_lineage records created`
- `Unit: duplicate run_id + event_type → upsert (idempotent)`
- `Integration: POST /api/v1/lineage/events returns 202 Accepted`
- `Integration: batch endpoint accepts array of up to 100 events`
- `Integration: datasets table auto-populated from input/output references`
- `Fixture: sample Airflow OpenLineage event, sample dbt OpenLineage event, sample Spark event`

#### 5.2 — Lineage Query API

**What**: APIs for traversing upstream/downstream lineage from any dataset or data product, including column-level lineage.

**Design**:

```typescript
// packages/shared/src/types/lineage.ts
export interface LineageNode {
  id: string;
  type: 'dataset' | 'job' | 'data_product';
  namespace: string;
  name: string;
  dataProductId?: string;
  sourceType?: string;
}

export interface LineageEdge {
  sourceId: string;
  targetId: string;
  relationship: 'input_of' | 'output_of';
  lastRunId?: string;
  lastRunAt?: string;
}

export interface LineageGraph {
  nodes: LineageNode[];
  edges: LineageEdge[];
  depth: number;
}

export interface ColumnLineageEntry {
  sourceDataset: { namespace: string; name: string };
  sourceColumn: string;
  targetDataset: { namespace: string; name: string };
  targetColumn: string;
  transformation?: string;
  lastRunAt: string;
}

// API Routes:
// GET /api/v1/lineage/upstream?dataset=ns:name&depth=5    → Upstream graph
// GET /api/v1/lineage/downstream?dataset=ns:name&depth=5  → Downstream graph
// GET /api/v1/lineage/impact?dataset=ns:name&column=col   → Column-level impact
// GET /api/v1/lineage/datasets                            → List known datasets
// GET /api/v1/lineage/datasets/:id                        → Dataset details with latest schema
// GET /api/v1/data-products/:id/lineage                   → Lineage for a data product
```

**Testing**:
- `Unit: upstream query with depth=3 returns 3 levels of ancestors`
- `Unit: downstream query stops at specified depth (no infinite loops)`
- `Unit: circular lineage graph handled without stack overflow`
- `Unit: column-level impact for column "amount" returns all derived columns`
- `Integration: after ingesting 5 connected events, GET /upstream returns complete path`
- `Integration: GET /api/v1/data-products/:id/lineage links datasets to product`
- `Fixture: pre-built lineage graph with 10 datasets, 5 jobs, column lineage between 3 datasets`

#### 5.3 — Dataset-to-Product Linking

**What**: Automatically and manually link lineage datasets to registered data products so lineage flows into the governance graph.

**Design**:

```typescript
// apps/api/src/services/dataset-linking.service.ts
export class DatasetLinkingService {
  // Auto-link: match dataset namespace+name patterns to data product distributions
  async autoLink(): Promise<LinkResult[]>;

  // Manual link: API for operators to explicitly link a dataset to a product
  async manualLink(datasetId: string, dataProductId: string): Promise<void>;

  // Unlink
  async unlink(datasetId: string): Promise<void>;

  // Suggest links based on name similarity
  async suggestLinks(datasetId: string): Promise<LinkSuggestion[]>;
}

export interface LinkResult {
  datasetId: string;
  dataProductId: string;
  confidence: 'high' | 'medium' | 'low';
  matchReason: string;
}

export interface LinkSuggestion {
  dataProductId: string;
  dataProductName: string;
  confidence: number;  // 0-1
  reason: string;
}

// API Routes:
// POST   /api/v1/lineage/datasets/:id/link     → Manual link
// DELETE /api/v1/lineage/datasets/:id/link     → Unlink
// GET    /api/v1/lineage/datasets/:id/suggest  → Suggest links
// POST   /api/v1/lineage/auto-link             → Trigger auto-link job
```

**Testing**:
- `Unit: autoLink matches "snowflake://acme:analytics.revenue" to product with matching distribution URL`
- `Unit: suggestLinks returns products with similar names scored by trigram similarity`
- `Unit: manualLink updates dataset.data_product_id`
- `Integration: after auto-link, lineage queries include product context`
- `Integration: manual link is audited`

---

## Phase 6: Governance Policies & Access Control

### Purpose

Implement the policy engine — defining governance policies, assigning them to domains/products, enforcing them at contract creation time, and managing access requests. This phase makes the platform useful for compliance teams.

### Tasks

#### 6.1 — Policy Definition and Assignment

**What**: CRUD for governance policies with JSONB rule definitions, and assignment of policies to domains or data products.

**Design**:

```typescript
// packages/shared/src/types/policy.ts
export interface Policy {
  id: string;
  name: string;
  slug: string;
  category: PolicyCategory;
  scope: 'global' | 'organisation' | 'domain' | 'data_product';
  enforcement: 'mandatory' | 'advisory' | 'automated';
  status: 'draft' | 'active' | 'retired';
  rules: PolicyRule[];
  createdBy: string | null;
  createdAt: string;
  updatedAt: string;
}

// ISO/IEC 38505 aligned categories plus data-governance-specific additions
export type PolicyCategory =
  | 'responsibility' | 'strategy' | 'acquisition'
  | 'performance' | 'conformance' | 'human_behaviour'
  | 'access_control' | 'data_quality' | 'retention'
  | 'classification' | 'privacy';

export interface PolicyRule {
  type: string;                      // 'classification_required', 'contract_required', etc.
  appliesTo?: string;                // 'data_product', 'contract', 'domain'
  condition?: Record<string, unknown>;  // Rule-specific parameters
}

// Policy enforcement:
// When a data product is published or a contract is activated,
// the PolicyEnforcementService checks all applicable policies
// and returns pass/fail with details.

export class PolicyEnforcementService {
  async evaluate(
    targetType: 'data_product' | 'contract',
    targetId: string,
    action: 'publish' | 'activate'
  ): Promise<PolicyEvaluationResult>;
}

export interface PolicyEvaluationResult {
  passed: boolean;
  results: {
    policyId: string;
    policyName: string;
    enforcement: string;
    passed: boolean;
    violations: string[];
  }[];
}

// API Routes:
// POST   /api/v1/policies                      → Create policy
// GET    /api/v1/policies                      → List policies
// GET    /api/v1/policies/:id                  → Get policy
// PATCH  /api/v1/policies/:id                  → Update policy
// POST   /api/v1/policies/:id/activate         → Activate policy
// POST   /api/v1/policy-assignments            → Assign policy to target
// DELETE /api/v1/policy-assignments/:id        → Remove assignment
// POST   /api/v1/policies/evaluate             → Evaluate policies for target
```

**Testing**:
- `Unit: evaluate with "classification_required" policy on product missing classification → fails`
- `Unit: evaluate with "contract_required" policy on published product without active contract → fails`
- `Unit: advisory policy failure does not block publication`
- `Unit: mandatory policy failure blocks publication with error`
- `Integration: assigning a policy to a domain applies to all products in that domain`
- `Integration: policy evaluation includes inherited domain policies`
- `Integration: retired policies are excluded from evaluation`

#### 6.2 — Access Request Workflow

**What**: End-to-end access request and approval flow for data products, integrated with team ownership.

**Design**:

```typescript
// packages/shared/src/types/access-request.ts
export interface AccessRequest {
  id: string;
  dataProductId: string;
  requesterId: string;
  requesterTeamId: string;
  purpose: string;
  requestConfig: {
    accessLevel: 'read' | 'write';
    durationDays?: number;
    useCase?: string;
  };
  status: 'pending' | 'approved' | 'rejected' | 'expired' | 'revoked';
  reviewedBy: string | null;
  reviewedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

// Workflow:
// 1. User submits access request
// 2. Owning team notified (BullMQ notification job)
// 3. Owner approves or rejects
// 4. If approved with durationDays, scheduled job revokes at expiry
// 5. Access can be manually revoked at any time

// API Routes:
// POST   /api/v1/access-requests               → Submit request
// GET    /api/v1/access-requests               → List (filter by product, team, status)
// POST   /api/v1/access-requests/:id/approve   → Approve
// POST   /api/v1/access-requests/:id/reject    → Reject (with reason)
// POST   /api/v1/access-requests/:id/revoke    → Revoke
// GET    /api/v1/data-products/:id/access      → List access grants for product
```

**Testing**:
- `Unit: approve sets reviewedBy and reviewedAt`
- `Unit: reject requires reason in body`
- `Unit: approve by non-owner returns 403`
- `Unit: request with durationDays=90 schedules expiry job`
- `Integration: POST access request triggers notification event`
- `Integration: expired access auto-revoked by scheduled worker`
- `Integration: access request history visible in audit log`

#### 6.3 — Business Glossary

**What**: Manage business terms with definitions, synonyms, and domain scoping for consistent data vocabulary across the organisation.

**Design**:

```typescript
// packages/shared/src/types/glossary.ts
export interface GlossaryTerm {
  id: string;
  term: string;
  definition: string;
  domainId: string | null;
  status: 'draft' | 'approved' | 'deprecated';
  synonyms: string[];
  relatedTerms: { id: string; relationship: 'broader' | 'narrower' | 'related' }[];
  approvedBy: string | null;
  createdAt: string;
  updatedAt: string;
}

// API Routes:
// POST   /api/v1/glossary                      → Create term
// GET    /api/v1/glossary                      → List (search, filter by domain/status)
// GET    /api/v1/glossary/:id                  → Get term with relations
// PATCH  /api/v1/glossary/:id                  → Update
// POST   /api/v1/glossary/:id/approve          → Approve term
// GET    /api/v1/glossary/search?q=revenue     → Search terms
```

**Testing**:
- `Unit: creating a term adds it to search index`
- `Unit: synonyms searchable (searching "income" finds term "Revenue")`
- `Unit: approve sets approvedBy and changes status`
- `Integration: GET /api/v1/glossary/search returns ranked results`
- `Integration: glossary terms appear in unified search`

---

## Phase 7: Quality Monitoring & SLA Dashboard

### Purpose

Implement quality check result storage, SLA monitoring, historical trend tracking, and the quality dashboard API. After this phase, organisations can monitor data product health over time and detect degradation before consumers are impacted.

### Tasks

#### 7.1 — Quality Check Result Ingestion

**What**: Accept quality check results (from external runners like Soda, dbt tests, or the platform's own scheduler), store them with time partitioning, and trigger breach detection.

**Design**:

```typescript
// packages/shared/src/types/quality.ts
export interface QualityCheckSubmission {
  contractId: string;
  checkTime?: string;          // Defaults to now
  results: {
    totalRules: number;
    passed: number;
    failed: number;
    details: {
      dimension: string;
      rule: string;
      expression?: string;
      passed: boolean;
      actualValue?: number;
      threshold?: number;
      unit?: string;
    }[];
  };
  runDurationMs?: number;
}

export interface QualityDashboardEntry {
  contractId: string;
  dataProductId: string;
  domainId: string;
  checkDate: string;
  totalChecks: number;
  passedChecks: number;
  failedChecks: number;
  passRate: number;
  activeBreach: boolean;
}

// API Routes:
// POST   /api/v1/quality/results                → Submit quality check results
// GET    /api/v1/quality/dashboard              → Quality dashboard (aggregated)
// GET    /api/v1/quality/contracts/:id/history  → Quality trend for contract
// GET    /api/v1/quality/domains/:id/summary    → Domain-level quality summary
```

**Testing**:
- `Unit: submitting results triggers breach detection service`
- `Unit: results stored in partitioned table with correct partition`
- `Unit: dashboard aggregates daily pass rates correctly`
- `Integration: POST results with failed rule → contract status set to "violated"`
- `Integration: GET history returns time-series data for last 30 days`
- `Integration: domain summary aggregates across all products in domain`
- `Fixture: 90 days of quality check results for trend testing`

#### 7.2 — SLA Monitoring Scheduler

**What**: BullMQ scheduled job that evaluates time-based SLAs (freshness, availability) for all active contracts and generates breach events when targets are missed.

**Design**:

```typescript
// apps/api/src/workers/sla-monitor.worker.ts
export class SlaMonitorWorker {
  // Runs every 5 minutes via BullMQ repeatable job
  async process(): Promise<SlaMonitorResult>;
}

export interface SlaMonitorResult {
  contractsChecked: number;
  breachesDetected: number;
  breachesResolved: number;
}

// Logic:
// 1. Query all active contracts with SLA definitions
// 2. For each freshness SLA: check if latest quality check is within window
// 3. For each availability SLA: calculate uptime from quality check pass rate over window
// 4. If target missed and not already breached → create breach event
// 5. If previously breached but now within target → resolve breach
// 6. Notify affected consumers of new breaches

// BullMQ job configuration:
// Queue: 'sla-monitor'
// Repeat: { every: 300000 } (5 minutes)
// Attempts: 3
// Backoff: { type: 'exponential', delay: 10000 }
```

**Testing**:
- `Unit: freshness SLA with 60-minute window and last check 120 minutes ago → breach`
- `Unit: freshness SLA with 60-minute window and last check 30 minutes ago → no breach`
- `Unit: availability SLA at 99.5% with actual 98% → breach`
- `Unit: previously breached contract now passing → breach resolved`
- `Integration: scheduled job runs and processes all active contracts`
- `Integration: breach events persisted and consumers notified`

#### 7.3 — Notification Service

**What**: Send notifications to teams on contract breaches, access request actions, and policy violations via webhook and in-app channels.

**Design**:

```typescript
// apps/api/src/services/notification.service.ts
export class NotificationService {
  async send(notification: Notification): Promise<void>;
  async getForUser(userId: string, unreadOnly?: boolean): Promise<Notification[]>;
  async markRead(notificationId: string): Promise<void>;
}

export interface Notification {
  id?: string;
  recipientType: 'user' | 'team';
  recipientId: string;
  channel: 'in_app' | 'webhook' | 'email';
  type: 'breach' | 'access_request' | 'access_approved' | 'policy_violation' | 'contract_updated';
  title: string;
  body: string;
  metadata: Record<string, unknown>;
  readAt?: string;
  createdAt?: string;
}

// Webhook delivery via BullMQ with retry:
// - Teams can configure webhook URLs in team settings
// - Failed webhooks retried 3 times with exponential backoff
// - Dead letter after 3 failures

// API Routes:
// GET    /api/v1/notifications                 → List notifications for current user
// PATCH  /api/v1/notifications/:id/read        → Mark as read
// POST   /api/v1/teams/:id/webhooks            → Configure webhook URL
```

**Testing**:
- `Unit: breach notification sent to all subscriber teams`
- `Unit: webhook delivery retries on 5xx response`
- `Unit: webhook delivery gives up after 3 attempts and moves to dead letter`
- `Integration: access approval triggers notification to requester`
- `Integration: GET /api/v1/notifications returns unread notifications sorted by date`

---

## Phase 8: Authentication & Authorization

### Purpose

Implement OIDC-based authentication, API key management, and role-based access control. This phase secures the platform for multi-tenant, multi-domain deployments where teams should only see and manage their own resources.

### Tasks

#### 8.1 — OIDC Authentication (Auth.js)

**What**: Implement OIDC login flow for the web frontend and JWT token validation for API requests.

**Design**:

```typescript
// apps/api/src/plugins/auth.plugin.ts
export interface AuthContext {
  userId: string;
  email: string;
  displayName: string;
  teams: { teamId: string; role: string }[];
}

// Fastify plugin:
// - Validates Bearer JWT tokens on protected routes
// - Extracts user context into request.auth
// - Routes marked with { auth: false } skip validation
// - API key alternative: X-API-Key header lookup against api_keys table

// JWT payload structure:
export interface JwtPayload {
  sub: string;           // user.id
  email: string;
  name: string;
  iat: number;
  exp: number;
}

// API key table:
// CREATE TABLE api_keys (
//   id UUID PRIMARY KEY,
//   name VARCHAR(255) NOT NULL,
//   key_hash VARCHAR(64) NOT NULL UNIQUE,  -- SHA-256 of the key
//   team_id UUID REFERENCES teams(id),
//   scopes TEXT[] NOT NULL DEFAULT '{}',
//   expires_at TIMESTAMPTZ,
//   last_used_at TIMESTAMPTZ,
//   created_at TIMESTAMPTZ NOT NULL DEFAULT now()
// );

// API Routes:
// POST   /api/v1/auth/login          → Initiate OIDC flow
// GET    /api/v1/auth/callback        → OIDC callback
// POST   /api/v1/auth/refresh         → Refresh token
// POST   /api/v1/api-keys             → Create API key
// GET    /api/v1/api-keys             → List API keys for team
// DELETE /api/v1/api-keys/:id         → Revoke API key
```

**Testing**:
- `Unit: valid JWT → request.auth populated with correct user context`
- `Unit: expired JWT → 401 Unauthorized`
- `Unit: invalid JWT signature → 401 Unauthorized`
- `Unit: valid API key → request.auth populated with team context`
- `Unit: revoked API key → 401 Unauthorized`
- `Integration: OIDC login flow with mock provider creates user on first login`
- `Integration: API key creation returns key once (never stored in plaintext)`

#### 8.2 — Role-Based Access Control (RBAC)

**What**: Enforce permissions based on team membership and role: owners can manage, maintainers can write, members can read within their domain, viewers read-only.

**Design**:

```typescript
// apps/api/src/plugins/rbac.plugin.ts
export type Permission =
  | 'domain:read' | 'domain:write' | 'domain:admin'
  | 'product:read' | 'product:write' | 'product:publish'
  | 'contract:read' | 'contract:write' | 'contract:activate'
  | 'policy:read' | 'policy:write' | 'policy:enforce'
  | 'access:approve' | 'access:revoke'
  | 'admin:*';

export const rolePermissions: Record<string, Permission[]> = {
  owner: ['domain:admin', 'product:publish', 'contract:activate', 'policy:enforce', 'access:approve', 'access:revoke'],
  maintainer: ['domain:write', 'product:write', 'contract:write', 'access:approve'],
  member: ['domain:read', 'product:read', 'product:write', 'contract:read'],
  viewer: ['domain:read', 'product:read', 'contract:read'],
};

// Route-level permission decorator:
// { config: { permissions: ['product:write'] } }
// The RBAC plugin checks:
// 1. User's teams and roles
// 2. Whether the target resource belongs to a domain owned by one of those teams
// 3. Whether the role has the required permission

export class RbacService {
  async canAccess(
    userId: string,
    resource: { type: string; id: string },
    permission: Permission
  ): Promise<boolean>;

  async getAccessibleDomains(userId: string): Promise<string[]>;
}
```

**Testing**:
- `Unit: owner of domain A can publish product in domain A`
- `Unit: member of domain A cannot publish product in domain A`
- `Unit: viewer of domain A cannot write to products in domain A`
- `Unit: user with no team membership gets empty domain access`
- `Integration: API returns 403 when user lacks permission`
- `Integration: list endpoints filter results to accessible resources`

---

## Phase 9: AI-Native Features

### Purpose

Implement the AI-powered capabilities that differentiate this platform from incumbent tools: automated contract generation from schema analysis, natural-language policy authoring, and intelligent data product recommendations. Uses Anthropic Claude API with prompt caching.

### Tasks

#### 9.1 — AI Contract Generation

**What**: Given a data product's schema (from lineage events or manual input), generate a draft ODCS v3.1.0 data contract including quality rules and SLA suggestions.

**Design**:

```typescript
// apps/api/src/ai/contract-generator.ts
import Anthropic from '@anthropic-ai/sdk';

export class AiContractGenerator {
  constructor(private client: Anthropic) {}

  async generateContract(input: ContractGenerationInput): Promise<OdcsContract> {
    // Uses tool_use for structured output
    // Prompt caching enabled for the system prompt + ODCS schema reference
    const response = await this.client.messages.create({
      model: 'claude-sonnet-4-20250514',
      max_tokens: 4096,
      system: [
        {
          type: 'text',
          text: SYSTEM_PROMPT,
          cache_control: { type: 'ephemeral' },  // Cache the system prompt
        },
        {
          type: 'text',
          text: ODCS_SCHEMA_REFERENCE,
          cache_control: { type: 'ephemeral' },  // Cache the ODCS schema
        },
      ],
      messages: [{ role: 'user', content: this.buildUserPrompt(input) }],
      tools: [ODCS_CONTRACT_TOOL],
    });

    return this.extractContract(response);
  }
}

export interface ContractGenerationInput {
  dataProductId: string;
  schema?: OdcsSchemaObject[];          // From lineage or manual
  queryPatterns?: string[];              // Common queries against this product
  existingContracts?: OdcsContract[];    // Contracts from similar products
  domainConfig?: DomainConfig;           // Domain governance requirements
}

// System prompt instructs Claude to:
// 1. Analyse the schema and infer appropriate quality rules
// 2. Suggest SLAs based on query patterns and domain governance level
// 3. Classify columns for PII sensitivity
// 4. Output a complete ODCS v3.1.0 contract

// API Routes:
// POST /api/v1/ai/generate-contract    → Generate contract from schema
// POST /api/v1/ai/suggest-quality-rules → Suggest quality rules for existing contract
```

**Testing**:
- `Unit (mocked LLM): schema with email column → PII classification in generated contract`
- `Unit (mocked LLM): finance domain with strict governance → higher SLA targets`
- `Unit (mocked LLM): generated contract validates against ODCS v3.1.0 schema`
- `Unit: handles LLM rate limit with exponential backoff`
- `Unit: handles LLM timeout gracefully with user-friendly error`
- `Integration (mocked LLM): POST /api/v1/ai/generate-contract returns valid ODCS contract`
- `Integration (mocked LLM): generated contract can be directly saved via POST /api/v1/contracts`

#### 9.2 — Natural Language Policy Authoring

**What**: Convert plain-English policy descriptions into structured policy rules (JSONB) that the policy engine can enforce.

**Design**:

```typescript
// apps/api/src/ai/policy-author.ts
export class AiPolicyAuthor {
  async generatePolicyRules(
    description: string,
    context: PolicyContext
  ): Promise<PolicyRule[]>;

  async explainPolicy(policy: Policy): Promise<string>;  // Human-readable explanation
}

export interface PolicyContext {
  existingPolicies: Policy[];      // Avoid duplicates
  domainContext?: string;          // Domain description
  complianceRequirements?: string[];  // GDPR, HIPAA, SOX
}

// Example:
// Input: "All data products containing PII must have a freshness SLA of under 24 hours
//         and require classification before publishing"
// Output: [
//   { type: "classification_required", appliesTo: "data_product", condition: { hasPii: true } },
//   { type: "sla_required", appliesTo: "contract", condition: { metric: "freshness", maxHours: 24, whenPii: true } }
// ]

// API Routes:
// POST /api/v1/ai/author-policy     → Generate policy rules from description
// POST /api/v1/ai/explain-policy    → Explain a policy in plain English
```

**Testing**:
- `Unit (mocked LLM): PII-related description → rules with PII conditions`
- `Unit (mocked LLM): GDPR compliance description → classification + retention rules`
- `Unit (mocked LLM): ambiguous input returns clarifying questions instead of rules`
- `Unit: generated rules conform to PolicyRule schema`
- `Integration (mocked LLM): generated rules enforceable by PolicyEnforcementService`

#### 9.3 — Semantic Search with Embeddings

**What**: Enhance the search system with vector embeddings for semantic queries ("find tables related to customer purchase behavior").

**Design**:

```typescript
// apps/api/src/ai/embeddings.service.ts
export class EmbeddingsService {
  // Generate embedding for a text using Anthropic Voyager 3
  async embed(text: string): Promise<number[]>;

  // Batch embed for indexing
  async batchEmbed(texts: string[]): Promise<number[][]>;

  // Search by semantic similarity
  async semanticSearch(
    query: string,
    options: { limit?: number; threshold?: number; entityTypes?: string[] }
  ): Promise<SemanticSearchResult[]>;
}

export interface SemanticSearchResult {
  entityType: string;
  entityId: string;
  title: string;
  similarity: number;  // cosine similarity 0-1
}

// pgvector extension for storing and querying embeddings:
// ALTER TABLE search_index ADD COLUMN embedding vector(1024);
// CREATE INDEX idx_search_embedding ON search_index USING ivfflat (embedding vector_cosine_ops);

// Hybrid search: combine tsvector rank (keyword) + cosine similarity (semantic)
// Final score = 0.3 * keyword_rank + 0.7 * cosine_similarity

// API Routes:
// GET /api/v1/search?q=...&mode=semantic   → Semantic search
// GET /api/v1/search?q=...&mode=hybrid     → Hybrid (default)
// GET /api/v1/search?q=...&mode=keyword    → Keyword only (tsvector)
```

**Testing**:
- `Unit (mocked embeddings): "customer purchases" → similar to "buyer transactions"`
- `Unit: hybrid search combines keyword and semantic scores correctly`
- `Unit: threshold=0.7 filters out low-similarity results`
- `Integration (mocked): new data product indexed with embedding within 5s`
- `Integration (mocked): semantic search returns relevant results not matching keyword`

---

## Phase 10: MCP Server & External Integrations

### Purpose

Expose the governance platform to AI agents via Model Context Protocol (MCP), and integrate with external tools (dbt, Airflow) for automated lineage and contract enforcement in CI/CD pipelines.

### Tasks

#### 10.1 — MCP Server Implementation

**What**: Implement an MCP server that exposes data product catalog, contract status, and lineage to LLM agents and AI coding assistants.

**Design**:

```typescript
// apps/api/src/mcp/server.ts
import { McpServer, ResourceTemplate, Tool } from '@modelcontextprotocol/sdk/server';

export function createMcpServer(): McpServer {
  const server = new McpServer({
    name: 'data-mesh-governance',
    version: '1.0.0',
  });

  // Resources: expose catalog data as browseable resources
  server.resource('data-products', new ResourceTemplate(
    'data-product://{domain}/{slug}',
    { list: listDataProducts }
  ));

  server.resource('contracts', new ResourceTemplate(
    'contract://{productSlug}/{version}',
    { list: listContracts }
  ));

  // Tools: enable agents to query and interact
  server.tool('search_data_products', {
    description: 'Search for data products by name, domain, or keyword',
    inputSchema: { type: 'object', properties: { query: { type: 'string' } }, required: ['query'] },
    handler: searchDataProductsTool,
  });

  server.tool('get_contract', {
    description: 'Get the active data contract for a data product',
    inputSchema: { type: 'object', properties: { productId: { type: 'string' } }, required: ['productId'] },
    handler: getContractTool,
  });

  server.tool('check_lineage', {
    description: 'Get upstream or downstream lineage for a dataset',
    inputSchema: {
      type: 'object',
      properties: {
        dataset: { type: 'string' },
        direction: { type: 'string', enum: ['upstream', 'downstream'] },
        depth: { type: 'number', default: 3 },
      },
      required: ['dataset', 'direction'],
    },
    handler: checkLineageTool,
  });

  server.tool('validate_contract', {
    description: 'Validate a data contract against ODCS v3.1.0 specification',
    inputSchema: { type: 'object', properties: { contract: { type: 'object' } }, required: ['contract'] },
    handler: validateContractTool,
  });

  return server;
}
```

**Testing**:
- `Unit: search_data_products tool returns matching products`
- `Unit: get_contract tool returns active contract for product`
- `Unit: check_lineage tool returns graph within depth limit`
- `Unit: validate_contract tool returns validation result`
- `Integration: MCP server responds to tool/list request`
- `Integration: MCP server responds to resource/list request`
- `Integration: MCP server handles concurrent requests`

#### 10.2 — CI/CD Contract Validation CLI

**What**: A CLI tool (packaged as npm binary) that validates data contracts in CI/CD pipelines — similar to Data Contract CLI but integrated with the platform.

**Design**:

```typescript
// packages/cli/src/commands/validate.ts
// Usage: npx @datamesh/cli validate --contract ./contracts/revenue.yaml --api-url https://governance.example.com

export interface CliConfig {
  apiUrl: string;
  apiKey: string;
  contractPath?: string;
  contractGlob?: string;  // e.g., './contracts/**/*.yaml'
}

// Commands:
// validate    - Validate contract YAML against ODCS v3.1.0 and platform policies
// publish     - Push contract to platform (creates or updates)
// check       - Run quality checks defined in contract against live data source
// status      - Check contract status and active breaches

// Exit codes:
// 0 - all validations pass
// 1 - validation errors found
// 2 - connection error to platform

// Output format: JSON for CI parsing, human-readable for terminal
```

**Testing**:
- `Unit: validate with valid YAML → exit 0, no errors`
- `Unit: validate with schema errors → exit 1, errors listed`
- `Unit: validate with --format=json outputs parseable JSON`
- `Integration: publish pushes contract to running API`
- `Integration: status retrieves current breach status`

#### 10.3 — Webhook Event System

**What**: Configurable outbound webhooks that fire on platform events (contract breach, access approved, product published), enabling integration with Slack, PagerDuty, and custom systems.

**Design**:

```typescript
// packages/shared/src/types/webhook.ts
export interface WebhookConfig {
  id: string;
  teamId: string;
  url: string;
  secret: string;               // For HMAC signature verification
  events: WebhookEvent[];       // Events to subscribe to
  active: boolean;
  createdAt: string;
}

export type WebhookEvent =
  | 'contract.activated' | 'contract.violated' | 'contract.deprecated'
  | 'product.published' | 'product.deprecated'
  | 'access.requested' | 'access.approved' | 'access.rejected'
  | 'quality.check_failed' | 'sla.breached';

export interface WebhookPayload {
  event: WebhookEvent;
  timestamp: string;
  data: Record<string, unknown>;
  signature: string;            // HMAC-SHA256(secret, body)
}

// Delivery via BullMQ:
// - Max 3 retries with exponential backoff (10s, 60s, 300s)
// - Timeout: 10 seconds per delivery
// - Dead letter after 3 failures
// - Delivery log for debugging

// API Routes:
// POST   /api/v1/webhooks               → Create webhook config
// GET    /api/v1/webhooks               → List webhooks for team
// PATCH  /api/v1/webhooks/:id           → Update webhook
// DELETE /api/v1/webhooks/:id           → Delete webhook
// GET    /api/v1/webhooks/:id/deliveries → Delivery log
// POST   /api/v1/webhooks/:id/test      → Send test event
```

**Testing**:
- `Unit: webhook payload includes HMAC-SHA256 signature`
- `Unit: failed delivery retries with exponential backoff`
- `Unit: dead letter after 3 failures`
- `Unit: inactive webhook does not receive events`
- `Integration: contract.violated event triggers webhook delivery to configured URL`
- `Integration: delivery log records response code and timing`
- `Integration: test endpoint sends sample event to configured URL`

---

## Phase 11: Web Frontend

### Purpose

Build the Next.js web application providing the user-facing dashboard: domain browser, data product catalog, contract viewer, lineage visualisation, quality dashboard, and access management UI.

### Tasks

#### 11.1 — Layout, Navigation, and Authentication UI

**What**: App shell with sidebar navigation, authentication flow (login/logout), and responsive layout.

**Design**:

```typescript
// apps/web/src/app/(dashboard)/layout.tsx
// Server component with:
// - Sidebar: domains, products, contracts, lineage, quality, policies, settings
// - Top bar: search, notifications, user menu
// - Auth: next-auth session check, redirect to /login if unauthenticated

// Navigation structure:
// /domains           - Domain tree browser
// /products          - Data product catalog
// /products/:id      - Data product detail
// /contracts         - Contract list
// /contracts/:id     - Contract detail with schema + quality + SLA
// /lineage           - Lineage explorer (graph visualisation)
// /quality           - Quality dashboard
// /policies          - Policy management
// /settings          - Team settings, webhook config, API keys
```

**Testing**:
- `E2E: unauthenticated user redirected to /login`
- `E2E: authenticated user sees sidebar with navigation items`
- `E2E: responsive layout collapses sidebar on mobile`

#### 11.2 — Data Product Catalog UI

**What**: Browse, search, filter, and view data products with detail pages showing distributions, contracts, and lineage summary.

**Design**:

Components:
- `ProductList` — filterable, paginated grid/list view
- `ProductCard` — summary card with status badge, domain, FAIR scores
- `ProductDetail` — tabbed detail view (Overview, Schema, Contracts, Lineage, Access)
- `ProductPublishDialog` — confirmation dialog for publishing

**Testing**:
- `E2E: search for "revenue" shows matching products`
- `E2E: filter by domain shows only products in that domain`
- `E2E: product detail page shows active contract and distributions`
- `E2E: publish button transitions product to published state`

#### 11.3 — Lineage Visualisation

**What**: Interactive directed graph visualisation showing data flow between datasets, jobs, and data products with zoom, pan, and click-to-inspect.

**Design**:

```typescript
// Library: @xyflow/react (React Flow) for DAG rendering
// Layout: dagre for automatic hierarchical layout

// Component: LineageGraph
// - Fetches lineage from /api/v1/lineage/upstream or /downstream
// - Renders nodes (datasets as rectangles, jobs as diamonds, products as rounded)
// - Colour-coded edges by data freshness (green = recent, yellow = stale, red = violated)
// - Click node to see metadata panel
// - Supports column-level lineage toggle (expand node to show columns)
```

**Testing**:
- `E2E: lineage page renders graph for a data product`
- `E2E: clicking a node opens metadata sidebar`
- `E2E: toggling column-level lineage shows column nodes`
- `E2E: graph handles 50+ nodes without performance degradation`

#### 11.4 — Quality Dashboard UI

**What**: Time-series quality metrics dashboard with breach alerts, pass rate trends, and drill-down to individual rules.

**Design**:

```typescript
// Library: recharts for time-series charts
// Components:
// - QualityOverview — org-wide pass rate, active breaches count, worst-performing products
// - ContractQualityDetail — per-contract trend chart with rule-level breakdown
// - BreachAlert — banner/notification for active breaches with link to affected contract
// - SlaStatusTable — table of all SLAs with current status indicators
```

**Testing**:
- `E2E: quality dashboard shows pass rate chart for last 30 days`
- `E2E: clicking a breach alert navigates to affected contract`
- `E2E: domain filter updates all dashboard widgets`

---

## Phase 12: Federation & Production Hardening

### Purpose

Enable multi-domain federated deployments (domain teams running their own instances syncing to a central graph), add production security hardening, and implement performance optimisations for scale.

### Tasks

#### 12.1 — Federated Domain Sync

**What**: Allow domain-owned instances to sync metadata to a central governance platform via event-based replication.

**Design**:

```typescript
// Federation protocol:
// Each domain instance has its own PostgreSQL + API
// Central instance aggregates metadata from all domains
//
// Sync mechanism:
// 1. Domain instance publishes events to central via authenticated webhook
// 2. Central validates event, checks domain ownership, stores in central DB
// 3. Central search index updated with federated data
// 4. Lineage graph spans multiple domains

export interface FederationConfig {
  mode: 'central' | 'domain';
  centralUrl?: string;          // Required for domain instances
  domainId: string;
  syncApiKey: string;
  syncIntervalMs: number;       // Default 30000 (30s)
}

// Sync endpoint on central:
// POST /api/v1/federation/sync
// Body: { domainId, events: FederationEvent[] }
// Auth: federation API key

export interface FederationEvent {
  type: string;
  entityType: string;
  entityId: string;
  payload: Record<string, unknown>;
  occurredAt: string;
}
```

**Testing**:
- `Unit: domain instance buffers events during central downtime`
- `Unit: central rejects events from unknown domain`
- `Unit: central deduplicates events by ID`
- `Integration: product published on domain instance appears in central search within 60s`
- `Integration: lineage crossing domain boundaries renders correctly`

#### 12.2 — Rate Limiting and Security Hardening

**What**: Implement rate limiting, request size limits, OWASP API security controls, and production-grade error handling.

**Design**:

```typescript
// Security measures (OWASP API Security Top 10 2023):
// 1. Rate limiting: @fastify/rate-limit (100 req/min default, 1000/min for lineage ingestion)
// 2. Request size limit: 1MB default, 10MB for lineage batch endpoint
// 3. Input validation: Zod schemas on all route inputs (already in place)
// 4. Authentication on all non-health endpoints (Phase 8)
// 5. CORS strict origin whitelist in production
// 6. Security headers: helmet plugin
// 7. SQL injection: parameterised queries via Drizzle ORM (already in place)
// 8. Audit logging on all write operations (Phase 2)
// 9. API key rotation support
// 10. HTTPS enforcement in production (reverse proxy)

// Rate limit configuration:
export interface RateLimitConfig {
  global: { max: 100; timeWindow: '1 minute' };
  lineageIngestion: { max: 1000; timeWindow: '1 minute' };
  aiEndpoints: { max: 20; timeWindow: '1 minute' };
  search: { max: 60; timeWindow: '1 minute' };
}
```

**Testing**:
- `Unit: 101st request within 1 minute returns 429 Too Many Requests`
- `Unit: request exceeding 1MB returns 413 Payload Too Large`
- `Unit: response includes security headers (X-Content-Type-Options, etc.)`
- `Integration: rate limit state persisted in Redis (distributed)`
- `Integration: AI endpoints rate limited more aggressively`

#### 12.3 — Performance Optimisation

**What**: Query optimisation, connection pooling, response caching, and database partitioning for production scale.

**Design**:

```typescript
// Optimisations:
// 1. Connection pooling: pg-pool with max 20 connections
// 2. Response caching: Redis cache for:
//    - Search results (TTL 60s)
//    - Lineage graphs (TTL 300s, invalidated on new lineage event)
//    - Data product detail (TTL 30s)
// 3. Database partitioning:
//    - lineage_events: by event_time (quarterly)
//    - quality_checks: by check_time (quarterly)
//    - audit_log: by created_at (monthly)
// 4. Materialised views for quality dashboard aggregation
// 5. Pagination on all list endpoints (cursor-based for large result sets)

// Cache invalidation strategy:
// - Write-through for data products and contracts (update cache on write)
// - TTL-based for search and lineage (eventual consistency acceptable)
// - Event-based invalidation for lineage cache (new event → purge affected datasets)
```

**Testing**:
- `Unit: cached search result returns within 5ms`
- `Unit: cache invalidated on data product update`
- `Integration: 1000 concurrent lineage queries complete within 2s (with caching)`
- `Integration: partition pruning confirmed for time-range queries`
- `Load: 100 concurrent users browsing catalog without degradation`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Project Scaffold ─── required by everything
    │
Phase 2: Domain & Team Management ─── requires Phase 1
    │
    ├── Phase 3: Data Product Registry ─── requires Phase 2
    │       │
    │       ├── Phase 4: Data Contract Management ─── requires Phase 3
    │       │       │
    │       │       └── Phase 7: Quality Monitoring & SLA Dashboard ─── requires Phase 4
    │       │
    │       └── Phase 5: OpenLineage Ingestion & Lineage Graph ─── requires Phase 3
    │
    ├── Phase 6: Governance Policies & Access Control ─── requires Phase 2
    │       (can be developed in parallel with Phases 4-5 after Phase 3)
    │
    └── Phase 8: Authentication & Authorization ─── requires Phase 2
            (can be developed in parallel with Phases 3-7; integrate before Phase 11)

Phase 9: AI-Native Features ─── requires Phases 4, 5
    (can parallel with Phase 10 after Phase 5 complete)

Phase 10: MCP Server & External Integrations ─── requires Phases 4, 5
    (can parallel with Phase 9)

Phase 11: Web Frontend ─── requires Phases 3, 4, 5, 6, 7, 8
    (starts after core API surface is stable)

Phase 12: Federation & Production Hardening ─── requires Phases 1-11
    (final phase, production readiness)
```

### Parallelism Opportunities

- **Phases 4, 5, 6** can be developed concurrently after Phase 3 is complete.
- **Phase 8** can be developed in parallel with Phases 3-7 (stubbed authentication initially, then integrated).
- **Phases 9, 10** can be developed concurrently after Phases 4 and 5.
- **Phase 11** frontend work can start incrementally as each backend phase completes.

---

## Definition of Done (per phase)

1. All tasks for the phase are implemented with full functionality.
2. All unit tests pass (`pnpm run test:unit`).
3. All integration tests pass (`pnpm run test:integration`).
4. ESLint passes with zero errors (`pnpm run lint`).
5. TypeScript type checking passes with zero errors (`pnpm run typecheck`).
6. Docker build succeeds (`docker build .`).
7. All new API endpoints documented in auto-generated OpenAPI spec.
8. Database migrations created and reversible (`drizzle-kit generate`, `drizzle-kit push`).
9. New configuration options documented in `.env.example`.
10. Feature works end-to-end with `docker compose up` from a clean state.
11. No regressions — all tests from previous phases continue to pass.
12. Audit logging covers all new write operations.
