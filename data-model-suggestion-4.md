# Data Model Suggestion 4: Graph-Relational (Property Graph + Relational CRUD)

> Project: Data Mesh Governance Platform · Created: 2026-05-20

## Philosophy

This model splits the data layer into two complementary stores: a PostgreSQL relational database for operational CRUD (creating domains, publishing data products, managing contracts) and a property graph layer for relationship-heavy queries (lineage traversal, ownership chains, impact analysis, dependency graphs). The graph layer can be implemented either as a dedicated graph database (Neo4j, Amazon Neptune) or as a graph-on-relational pattern using `graph_nodes` and `graph_edges` tables in PostgreSQL with recursive CTEs.

The rationale is that data mesh governance is fundamentally a graph problem. The core questions — "What downstream consumers are affected if I change this column?", "What is the full ownership chain from data source to dashboard?", "Are there circular dependencies between domains?" — require multi-hop traversals that are expensive in normalised relational models but native operations in graph databases. DataHub's internal architecture uses a metadata graph (GMA) for exactly this reason, and Neo4j is widely used for data lineage in financial institutions (Basel Committee BCBS 239 compliance).

The relational layer handles transactional operations: CRUD on data products, contract lifecycle management, user authentication, access requests. The graph layer handles analytical and discovery operations: lineage traversal, impact analysis, dependency detection, and AI-powered recommendations. Writes go to the relational layer first, then a synchroniser propagates changes to the graph.

**Best for:** Organisations where lineage traversal, impact analysis, ownership chain queries, and dependency detection are primary use cases — especially financial institutions and enterprises with complex data supply chains.

**Trade-offs:**
- Pro: Multi-hop lineage and impact queries execute in milliseconds vs. seconds for recursive CTEs
- Pro: Natural representation of relationships — edges are first-class with properties
- Pro: Pattern matching queries (cycles, shortest paths, connected components) are native
- Pro: AI recommendation models can traverse the graph efficiently
- Pro: Relationship-heavy queries scale better than join-heavy relational equivalents
- Con: Two data stores to operate, monitor, and keep in sync
- Con: Graph databases have different operational characteristics (memory-heavy, different backup/restore)
- Con: Dual-write consistency requires careful synchronisation design
- Con: Graph query languages (Cypher, Gremlin) require specialist knowledge
- Con: Higher infrastructure cost and operational complexity

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenLineage | Lineage events decomposed into graph nodes (datasets, jobs) and edges (input_of, output_of, column_derives_from) |
| ODCS v3.1.0 | Contract entities in relational layer; producer/consumer relationships as graph edges |
| W3C DCAT 3 | Data product catalog entries in relational layer; `belongs_to_domain`, `distributed_via` edges in graph |
| ISO/IEC 38505 | Policy relationships modelled as `governed_by` edges connecting entities to policies |
| ISO 3166 | Jurisdiction nodes enable "find all products in EU" graph traversals |
| FAIR Principles | FAIR scores stored as node properties on data product nodes |
| OpenMetadata Standard | Entity type hierarchy modelled as graph node labels and `is_type` edges |

---

## Relational Layer (PostgreSQL — Operational CRUD)

```sql
-- Core entities for transactional operations
-- These tables handle authentication, CRUD, and configuration

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id     VARCHAR(255) UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    email           VARCHAR(255),
    slack_channel   VARCHAR(255),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);

CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    parent_id       UUID REFERENCES domains(id),
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE data_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    version         VARCHAR(50) NOT NULL DEFAULT '1.0.0',
    status          VARCHAR(50) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'deprecated', 'archived')),
    visibility      VARCHAR(50) NOT NULL DEFAULT 'domain',
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (domain_id, slug)
);

CREATE INDEX idx_products_domain ON data_products(domain_id);
CREATE INDEX idx_products_status ON data_products(status);

CREATE TABLE data_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_product_id UUID NOT NULL REFERENCES data_products(id),
    version         VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'active', 'deprecated', 'violated')),
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    contract_body   JSONB NOT NULL,                 -- Full ODCS v3.1.0 contract
    effective_from  DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (data_product_id, version)
);

CREATE TABLE contract_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id),
    consumer_team_id UUID NOT NULL REFERENCES teams(id),
    purpose         TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    category        VARCHAR(100) NOT NULL,
    scope           VARCHAR(50) NOT NULL DEFAULT 'organisation',
    enforcement     VARCHAR(50) NOT NULL DEFAULT 'advisory',
    rule_expression TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE glossary_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    term            VARCHAR(255) NOT NULL,
    definition      TEXT NOT NULL,
    domain_id       UUID REFERENCES domains(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    approved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    category        VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Quality check results (time-series)
CREATE TABLE quality_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id),
    check_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    results         JSONB NOT NULL,
    overall_passed  BOOLEAN NOT NULL,
    run_duration_ms INTEGER
);

CREATE INDEX idx_quality_contract ON quality_checks(contract_id);
CREATE INDEX idx_quality_time ON quality_checks(check_time);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id        UUID,
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user',
    action          VARCHAR(100) NOT NULL,
    target_type     VARCHAR(100) NOT NULL,
    target_id       UUID NOT NULL,
    details         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_target ON audit_log(target_type, target_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);

-- Access requests
CREATE TABLE access_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_product_id UUID NOT NULL REFERENCES data_products(id),
    requester_id    UUID NOT NULL REFERENCES users(id),
    requester_team_id UUID NOT NULL REFERENCES teams(id),
    purpose         TEXT NOT NULL,
    access_level    VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Graph Layer (Property Graph)

The graph layer can be implemented as either:
- **Option A:** A dedicated graph database (Neo4j Community/Enterprise, Amazon Neptune)
- **Option B:** PostgreSQL tables simulating a property graph (shown below)

### Option A: Neo4j Cypher Schema

```cypher
// --- Node definitions ---

// Domain nodes
CREATE CONSTRAINT domain_id IF NOT EXISTS FOR (d:Domain) REQUIRE d.id IS UNIQUE;
// Properties: id, name, slug, status

// Data product nodes
CREATE CONSTRAINT product_id IF NOT EXISTS FOR (p:DataProduct) REQUIRE p.id IS UNIQUE;
// Properties: id, name, slug, version, status, visibility, fair_findable, fair_accessible,
//             fair_interoperable, fair_reusable, published_at

// Contract nodes
CREATE CONSTRAINT contract_id IF NOT EXISTS FOR (c:Contract) REQUIRE c.id IS UNIQUE;
// Properties: id, version, status, effective_from, effective_to

// Dataset nodes (OpenLineage entities)
CREATE CONSTRAINT dataset_key IF NOT EXISTS FOR (ds:Dataset) REQUIRE (ds.namespace, ds.name) IS UNIQUE;
// Properties: namespace, name, source_type

// Job nodes (OpenLineage entities)
CREATE CONSTRAINT job_key IF NOT EXISTS FOR (j:Job) REQUIRE (j.namespace, j.name) IS UNIQUE;
// Properties: namespace, name, job_type

// Column nodes (for column-level lineage)
CREATE CONSTRAINT column_key IF NOT EXISTS FOR (col:Column) REQUIRE col.qualified_name IS UNIQUE;
// Properties: qualified_name (dataset.namespace:dataset.name:column_name), column_name, data_type

// Team nodes
CREATE CONSTRAINT team_id IF NOT EXISTS FOR (t:Team) REQUIRE t.id IS UNIQUE;
// Properties: id, name, slug

// User nodes
CREATE CONSTRAINT user_id IF NOT EXISTS FOR (u:User) REQUIRE u.id IS UNIQUE;
// Properties: id, email, display_name

// Policy nodes
CREATE CONSTRAINT policy_id IF NOT EXISTS FOR (pol:Policy) REQUIRE pol.id IS UNIQUE;
// Properties: id, name, category, scope, enforcement

// Tag nodes
CREATE CONSTRAINT tag_name IF NOT EXISTS FOR (tag:Tag) REQUIRE tag.name IS UNIQUE;
// Properties: name, category

// Glossary term nodes
CREATE CONSTRAINT glossary_id IF NOT EXISTS FOR (g:GlossaryTerm) REQUIRE g.id IS UNIQUE;
// Properties: id, term, status

// Jurisdiction nodes (ISO 3166)
CREATE CONSTRAINT jurisdiction_code IF NOT EXISTS FOR (j:Jurisdiction) REQUIRE j.code IS UNIQUE;
// Properties: code, name, region, gdpr_applicable


// --- Relationship definitions ---

// Domain hierarchy
// (:Domain)-[:CHILD_OF]->(:Domain)

// Domain ownership
// (:Domain)-[:OWNED_BY]->(:Team)

// Data product to domain
// (:DataProduct)-[:BELONGS_TO]->(:Domain)
// (:DataProduct)-[:OWNED_BY]->(:Team)

// Contract to product
// (:Contract)-[:GOVERNS]->(:DataProduct)
// (:Contract)-[:OWNED_BY]->(:Team)

// Subscription (consumer relationship)
// (:Team)-[:SUBSCRIBES_TO {purpose, status, approved_at}]->(:Contract)

// Lineage edges (OpenLineage)
// (:Dataset)-[:INPUT_OF {run_id, event_time}]->(:Job)
// (:Job)-[:OUTPUT_OF {run_id, event_time}]->(:Dataset)

// Column-level lineage
// (:Column)-[:DERIVES_FROM {transformation, run_id}]->(:Column)

// Column to dataset
// (:Column)-[:BELONGS_TO]->(:Dataset)

// Dataset to data product mapping
// (:Dataset)-[:MATERIALISES]->(:DataProduct)

// Policy governance
// (:Policy)-[:APPLIES_TO]->(:Domain)
// (:Policy)-[:APPLIES_TO]->(:DataProduct)

// Tagging
// (:Tag)-[:TAGGED_ON]->(:DataProduct)
// (:Tag)-[:TAGGED_ON]->(:Dataset)
// (:Tag)-[:TAGGED_ON]->(:Column)

// Glossary
// (:GlossaryTerm)-[:DEFINED_IN]->(:Domain)
// (:GlossaryTerm)-[:DESCRIBES]->(:Column)
// (:GlossaryTerm)-[:RELATED_TO]->(:GlossaryTerm)

// Team membership
// (:User)-[:MEMBER_OF {role}]->(:Team)

// Jurisdiction
// (:DataProduct)-[:SUBJECT_TO]->(:Jurisdiction)
```

### Option B: Graph-on-Relational (PostgreSQL)

```sql
-- Generic property graph tables in PostgreSQL
-- Use this if a dedicated graph database is not operationally feasible

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(100) NOT NULL,          -- 'DataProduct', 'Dataset', 'Job', 'Column', 'Team', etc.
    entity_id       UUID NOT NULL,                  -- FK to the relational entity table
    label           VARCHAR(500) NOT NULL,          -- Human-readable label
    properties      JSONB NOT NULL DEFAULT '{}',    -- Additional graph-queryable properties
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (entity_type, entity_id)
);

CREATE INDEX idx_graph_nodes_type ON graph_nodes(entity_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes(entity_type, entity_id);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING gin(properties);

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id) ON DELETE CASCADE,
    relationship    VARCHAR(100) NOT NULL,          -- 'BELONGS_TO', 'INPUT_OF', 'DERIVES_FROM', etc.
    properties      JSONB NOT NULL DEFAULT '{}',    -- Edge properties (run_id, event_time, etc.)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Prevent duplicate edges
    UNIQUE (source_node_id, target_node_id, relationship)
);

CREATE INDEX idx_graph_edges_source ON graph_edges(source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges(target_node_id);
CREATE INDEX idx_graph_edges_rel ON graph_edges(relationship);
CREATE INDEX idx_graph_edges_props ON graph_edges USING gin(properties);
```

## Graph Query Examples

### Neo4j Cypher Queries

```cypher
// 1. Full upstream lineage for a dataset (multi-hop traversal)
MATCH path = (target:Dataset {name: 'analytics.quarterly_revenue'})<-[:OUTPUT_OF]-(:Job)<-[:INPUT_OF]-(upstream:Dataset)
RETURN path;

// 2. Extended multi-hop lineage (3+ hops upstream)
MATCH path = (target:Dataset {name: 'analytics.quarterly_revenue'})
             <-[:OUTPUT_OF|INPUT_OF*1..6]-(source)
RETURN path;

// 3. Impact analysis: what downstream products are affected by a column change?
MATCH (col:Column {qualified_name: 'postgres://prod:raw.transactions:amount'})
      -[:DERIVES_FROM*1..10]->(downstream:Column)
      -[:BELONGS_TO]->(ds:Dataset)
      -[:MATERIALISES]->(dp:DataProduct)
RETURN DISTINCT dp.name AS affected_product, downstream.column_name AS affected_column;

// 4. Circular dependency detection between domains
MATCH path = (d:Domain)-[:BELONGS_TO*..10]->(d)
RETURN path;

// 5. Find all data products consumed by a specific team (via contract subscriptions)
MATCH (t:Team {name: 'analytics-team'})-[:SUBSCRIBES_TO]->(c:Contract)-[:GOVERNS]->(dp:DataProduct)
RETURN dp.name, c.version, c.status;

// 6. Ownership chain: from raw source to executive dashboard
MATCH path = (source:Dataset)-[:INPUT_OF|OUTPUT_OF*1..20]->(sink:Dataset)
WHERE source.source_type = 'postgres'
  AND sink.name CONTAINS 'dashboard'
WITH path, [n IN nodes(path) WHERE n:Dataset | n] AS datasets
UNWIND datasets AS ds
OPTIONAL MATCH (ds)-[:MATERIALISES]->(dp:DataProduct)-[:OWNED_BY]->(team:Team)
RETURN ds.name, dp.name, team.name;

// 7. Find data products without active contracts (governance gap)
MATCH (dp:DataProduct {status: 'published'})
WHERE NOT (dp)<-[:GOVERNS]-(:Contract {status: 'active'})
RETURN dp.name, dp.slug;

// 8. GDPR-scoped lineage: trace all data paths touching EU jurisdictions
MATCH (dp:DataProduct)-[:SUBJECT_TO]->(j:Jurisdiction {gdpr_applicable: true})
MATCH (ds:Dataset)-[:MATERIALISES]->(dp)
MATCH path = (source:Dataset)-[:INPUT_OF|OUTPUT_OF*1..10]->(ds)
RETURN path, j.code AS jurisdiction;
```

### PostgreSQL Graph-on-Relational Queries (Option B)

```sql
-- Multi-hop upstream lineage using recursive CTE
WITH RECURSIVE upstream AS (
    -- Start from the target dataset
    SELECT
        gn.id AS node_id,
        gn.entity_type,
        gn.label,
        ge.relationship,
        1 AS depth,
        ARRAY[gn.id] AS path
    FROM graph_nodes gn
    JOIN graph_edges ge ON ge.target_node_id = gn.id
    WHERE gn.entity_type = 'Dataset'
      AND gn.label = 'analytics.quarterly_revenue'

    UNION ALL

    -- Traverse upstream
    SELECT
        gn2.id,
        gn2.entity_type,
        gn2.label,
        ge2.relationship,
        u.depth + 1,
        u.path || gn2.id
    FROM upstream u
    JOIN graph_edges ge2 ON ge2.target_node_id = u.node_id
    JOIN graph_nodes gn2 ON gn2.id = ge2.source_node_id
    WHERE u.depth < 10
      AND NOT gn2.id = ANY(u.path)  -- Prevent cycles
)
SELECT entity_type, label, relationship, depth
FROM upstream
ORDER BY depth;

-- Impact analysis: downstream data products affected by a dataset change
WITH RECURSIVE downstream AS (
    SELECT gn.id AS node_id, gn.entity_type, gn.label, 1 AS depth, ARRAY[gn.id] AS path
    FROM graph_nodes gn
    WHERE gn.entity_type = 'Dataset'
      AND gn.properties->>'name' = 'raw.transactions'

    UNION ALL

    SELECT gn2.id, gn2.entity_type, gn2.label, d.depth + 1, d.path || gn2.id
    FROM downstream d
    JOIN graph_edges ge ON ge.source_node_id = d.node_id
    JOIN graph_nodes gn2 ON gn2.id = ge.target_node_id
    WHERE d.depth < 10
      AND NOT gn2.id = ANY(d.path)
)
SELECT DISTINCT label, entity_type
FROM downstream
WHERE entity_type = 'DataProduct';
```

## Graph Synchronisation

```sql
-- Sync tracking table: records which relational changes have been propagated to the graph
CREATE TABLE graph_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_table    VARCHAR(100) NOT NULL,
    source_id       UUID NOT NULL,
    operation       VARCHAR(20) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    synced_to_graph BOOLEAN NOT NULL DEFAULT false,
    sync_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    synced_at       TIMESTAMPTZ
);

CREATE INDEX idx_graph_sync_pending ON graph_sync_log(synced_to_graph) WHERE synced_to_graph = false;

-- Change Data Capture (CDC) trigger example for data_products
CREATE OR REPLACE FUNCTION notify_graph_sync()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO graph_sync_log (source_table, source_id, operation)
    VALUES (TG_TABLE_NAME, COALESCE(NEW.id, OLD.id), TG_OP);
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_data_products_graph_sync
    AFTER INSERT OR UPDATE OR DELETE ON data_products
    FOR EACH ROW EXECUTE FUNCTION notify_graph_sync();

CREATE TRIGGER trg_domains_graph_sync
    AFTER INSERT OR UPDATE OR DELETE ON domains
    FOR EACH ROW EXECUTE FUNCTION notify_graph_sync();

CREATE TRIGGER trg_contracts_graph_sync
    AFTER INSERT OR UPDATE OR DELETE ON data_contracts
    FOR EACH ROW EXECUTE FUNCTION notify_graph_sync();
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Relational — Core Entities | 5 | users, teams, team_members, domains, data_products |
| Relational — Contracts & Access | 4 | data_contracts, contract_subscriptions, policies, access_requests |
| Relational — Classification | 3 | glossary_terms, tags, tag_assignments (if not using graph) |
| Relational — Monitoring & Audit | 2 | quality_checks, audit_log |
| Graph Layer (Option B) | 2 | graph_nodes, graph_edges |
| Sync Infrastructure | 1 | graph_sync_log |
| **Total (Option B)** | **17** | PostgreSQL-only deployment |
| **Total (Option A)** | **14** | PostgreSQL + Neo4j (graph tables replaced by Neo4j) |

---

## Key Design Decisions

1. **Dual-store architecture** — the relational layer handles transactional consistency (ACID writes for contract creation, access approvals, policy changes) while the graph layer handles relationship-heavy reads (lineage traversal, impact analysis, dependency detection). This plays to the strengths of each storage engine.

2. **Column-level lineage as graph edges** — `(:Column)-[:DERIVES_FROM]->(:Column)` edges enable column-level impact analysis with simple graph traversals. In a normalised relational model, this requires recursive CTEs on a junction table; in a graph, it is a native multi-hop query.

3. **OpenLineage events decomposed into graph nodes and edges** — rather than storing OpenLineage RunEvents as blobs, they are decomposed into `Dataset`, `Job`, and `Column` nodes with `INPUT_OF`, `OUTPUT_OF`, and `DERIVES_FROM` edges. This loses the raw event fidelity (addressed by keeping the event body in the relational audit log) but enables efficient graph traversal.

4. **CDC-based synchronisation** — PostgreSQL triggers capture changes to relational tables and write to `graph_sync_log`. A background worker reads pending sync entries and propagates changes to the graph. This eventual consistency model (typically sub-second lag) avoids distributed transactions while keeping the graph up to date.

5. **Option A vs. Option B** — for deployments processing millions of lineage edges, Neo4j (Option A) is strongly recommended. Its native graph storage engine handles multi-hop traversals orders of magnitude faster than PostgreSQL recursive CTEs. For smaller deployments (under 100K nodes), the graph-on-relational pattern (Option B) avoids the operational overhead of a second database.

6. **Graph nodes reference relational entities** — `graph_nodes.entity_id` points back to the relational table's primary key. This means the relational layer remains the source of truth for entity properties, and the graph layer stores only relationship structure plus frequently-traversed properties.

7. **Jurisdiction nodes in the graph** — modelling jurisdictions as graph nodes with `SUBJECT_TO` edges enables GDPR compliance queries as graph traversals: "trace all data paths that touch EU jurisdictions" becomes a single Cypher pattern match.

8. **Governance gap detection via graph patterns** — queries like "find published data products without active contracts" are simple negative pattern matches in Cypher (`WHERE NOT (dp)<-[:GOVERNS]-(:Contract)`) that would require LEFT JOIN / IS NULL patterns in SQL.

9. **AI-ready graph structure** — graph neural networks (GNNs) and graph embedding models can operate directly on the property graph for recommendations ("suggest data products similar to ones your team consumes"), anomaly detection ("unusual access pattern"), and automated classification.

10. **Tag and glossary relationships as graph edges** — tags and glossary terms connected via graph edges enable semantic queries like "find all data products tagged with terms related to 'revenue'" by traversing `RELATED_TO` edges between glossary terms.
