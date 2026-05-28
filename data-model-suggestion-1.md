# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Data Mesh Governance Platform · Created: 2026-05-20

## Philosophy

This model applies classic third-normal-form relational design to every concept in the data mesh governance domain. Each entity — domain, data product, data contract, lineage event, policy, glossary term — gets its own table with explicit foreign keys and junction tables for many-to-many relationships. Reference data (jurisdictions, quality dimensions, event types) is modelled as lookup tables aligned with ISO and W3C standards.

The approach mirrors how DataHub and OpenMetadata structure their persistence layers: a typed entity registry where each metadata entity has a well-defined schema and relationships are first-class database constructs. This maximises query flexibility — any cross-entity question can be answered with standard SQL joins — and enforces referential integrity at the database level.

The trade-off is schema rigidity. Adding a new entity type or relationship requires a migration. Multi-jurisdiction variations (different compliance fields per region) require either nullable columns or additional junction tables. This model is best suited for teams that value data integrity and have a mature migration workflow.

**Best for:** Regulated enterprises where referential integrity, audit compliance, and complex cross-entity reporting are paramount.

**Trade-offs:**
- Pro: Maximum referential integrity enforced at the database level
- Pro: Standard SQL joins for any cross-entity query without special tooling
- Pro: Well-understood by any team with relational database experience
- Pro: Straightforward indexing and query optimisation
- Con: High table count increases migration complexity
- Con: Schema changes require DDL migrations for every new entity type or field
- Con: Multi-jurisdiction or multi-type variability creates nullable column sprawl
- Con: Deep join chains for lineage traversal queries can be slow without materialised views

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ODCS v3.1.0 | Data contract structure mapped to `data_contracts`, `contract_schemas`, `contract_quality_rules`, and `contract_sla` tables |
| OpenLineage | Lineage events stored in `lineage_events` with `lineage_datasets`, `lineage_jobs`, `lineage_runs` mirroring the OpenLineage entity model |
| W3C DCAT 3 | Data products modelled as DCAT `dcat:Resource` with `distributions` table for access endpoints |
| OpenMetadata Standard | Entity type registry inspired by OpenMetadata's 700+ JSON schema entity definitions |
| ISO/IEC 38505 | Governance policy categories aligned with the six governing principles |
| ISO/IEC 25012 | Quality dimension lookup table based on ISO 25012 quality characteristics |
| ISO 3166 | Jurisdiction reference table using ISO 3166-1 alpha-2 codes |
| OpenAPI 3.1 | API endpoint metadata stored in `api_endpoints` table for registered data product APIs |

---

## Domain Management

```sql
-- Organisational domains following data mesh domain ownership principle
CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    parent_id       UUID REFERENCES domains(id),
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'archived', 'draft')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_domains_parent ON domains(parent_id);
CREATE INDEX idx_domains_slug ON domains(slug);
CREATE INDEX idx_domains_owner_team ON domains(owner_team_id);

-- Teams that own domains and data products
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    email           VARCHAR(255),
    slack_channel   VARCHAR(255),
    domain_id       UUID REFERENCES domains(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Team membership
CREATE TABLE team_members (
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role            VARCHAR(50) NOT NULL DEFAULT 'member'
                    CHECK (role IN ('owner', 'maintainer', 'member', 'viewer')),
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (team_id, user_id)
);

-- Platform users
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id     VARCHAR(255) UNIQUE,          -- SSO / OIDC subject identifier
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    avatar_url      TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

## Data Product Registry

```sql
-- Data products: the core publishable unit in a data mesh
-- Aligned with DCAT 3 dcat:Resource and DataHub DataProduct entity
CREATE TABLE data_products (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    domain_id       UUID NOT NULL REFERENCES domains(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    purpose         TEXT,                          -- Why this data product exists
    version         VARCHAR(50) NOT NULL DEFAULT '1.0.0',
    status          VARCHAR(50) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'published', 'deprecated', 'archived')),
    visibility      VARCHAR(50) NOT NULL DEFAULT 'domain'
                    CHECK (visibility IN ('private', 'domain', 'organisation', 'public')),
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    -- DCAT 3 alignment
    dcat_theme      VARCHAR(255),                  -- dcat:theme URI
    dcat_keyword    TEXT[],                         -- dcat:keyword array
    dcat_landing_page TEXT,                         -- dcat:landingPage
    -- FAIR assessment scores (0-100)
    fair_findable   SMALLINT CHECK (fair_findable BETWEEN 0 AND 100),
    fair_accessible SMALLINT CHECK (fair_accessible BETWEEN 0 AND 100),
    fair_interoperable SMALLINT CHECK (fair_interoperable BETWEEN 0 AND 100),
    fair_reusable   SMALLINT CHECK (fair_reusable BETWEEN 0 AND 100),
    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (domain_id, slug)
);

CREATE INDEX idx_data_products_domain ON data_products(domain_id);
CREATE INDEX idx_data_products_status ON data_products(status);
CREATE INDEX idx_data_products_owner ON data_products(owner_team_id);

-- Data product versions for tracking schema evolution
CREATE TABLE data_product_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_product_id UUID NOT NULL REFERENCES data_products(id),
    version         VARCHAR(50) NOT NULL,
    changelog       TEXT,
    is_breaking     BOOLEAN NOT NULL DEFAULT false,
    published_by    UUID REFERENCES users(id),
    published_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (data_product_id, version)
);

-- Distribution endpoints (DCAT 3 dcat:Distribution)
CREATE TABLE distributions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_product_id UUID NOT NULL REFERENCES data_products(id) ON DELETE CASCADE,
    name            VARCHAR(255) NOT NULL,
    access_type     VARCHAR(50) NOT NULL
                    CHECK (access_type IN ('api', 'file', 'stream', 'database', 'delta_sharing')),
    format          VARCHAR(100),                  -- e.g., 'parquet', 'json', 'csv', 'avro'
    endpoint_url    TEXT,
    media_type      VARCHAR(100),                  -- IANA media type
    byte_size       BIGINT,
    compress_format VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_distributions_product ON distributions(data_product_id);
```

## Data Contracts (ODCS v3.1.0 Aligned)

```sql
-- Data contracts following Open Data Contract Standard v3.1.0
CREATE TABLE data_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_product_id UUID NOT NULL REFERENCES data_products(id),
    version         VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'active', 'deprecated', 'violated')),
    -- ODCS fundamentals
    odcs_version    VARCHAR(20) NOT NULL DEFAULT '3.1.0',
    kind            VARCHAR(50) NOT NULL DEFAULT 'DataContract',
    description     TEXT,
    -- Ownership
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    -- Lifecycle
    effective_from  DATE,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (data_product_id, version)
);

CREATE INDEX idx_contracts_product ON data_contracts(data_product_id);
CREATE INDEX idx_contracts_status ON data_contracts(status);

-- Contract schema definitions (ODCS schema section)
CREATE TABLE contract_schemas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id) ON DELETE CASCADE,
    object_name     VARCHAR(255) NOT NULL,         -- ODCS "object" (table/topic/file)
    object_type     VARCHAR(50) NOT NULL
                    CHECK (object_type IN ('table', 'topic', 'file', 'api', 'view')),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Contract schema properties (columns/fields within an object)
CREATE TABLE contract_schema_properties (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_id       UUID NOT NULL REFERENCES contract_schemas(id) ON DELETE CASCADE,
    property_name   VARCHAR(255) NOT NULL,
    logical_type    VARCHAR(100) NOT NULL,          -- e.g., 'string', 'integer', 'date', 'decimal'
    physical_type   VARCHAR(100),                   -- e.g., 'VARCHAR(255)', 'BIGINT'
    is_required     BOOLEAN NOT NULL DEFAULT false,
    is_primary_key  BOOLEAN NOT NULL DEFAULT false,
    is_pii          BOOLEAN NOT NULL DEFAULT false,
    classification  VARCHAR(100),                   -- 'public', 'internal', 'confidential', 'restricted'
    description     TEXT,
    ordinal         INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_schema_props_schema ON contract_schema_properties(schema_id);

-- Quality rules attached to contracts (ODCS quality section)
CREATE TABLE contract_quality_rules (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id) ON DELETE CASCADE,
    dimension       VARCHAR(100) NOT NULL,          -- ISO 25012 dimension: accuracy, completeness, etc.
    rule_name       VARCHAR(255) NOT NULL,
    rule_type       VARCHAR(50) NOT NULL
                    CHECK (rule_type IN ('assertion', 'threshold', 'custom_sql', 'freshness', 'volume')),
    expression      TEXT NOT NULL,                  -- SQL or rule expression
    threshold_value NUMERIC,
    threshold_unit  VARCHAR(50),                    -- 'percent', 'count', 'minutes'
    severity        VARCHAR(50) NOT NULL DEFAULT 'warning'
                    CHECK (severity IN ('info', 'warning', 'error', 'critical')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_quality_rules_contract ON contract_quality_rules(contract_id);

-- SLA definitions (ODCS SLA section)
CREATE TABLE contract_slas (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id) ON DELETE CASCADE,
    metric          VARCHAR(100) NOT NULL,          -- 'freshness', 'availability', 'latency', 'completeness'
    target_value    NUMERIC NOT NULL,
    target_unit     VARCHAR(50) NOT NULL,           -- 'minutes', 'percent', 'milliseconds'
    measurement_window VARCHAR(50) NOT NULL,        -- 'hourly', 'daily', 'weekly', 'monthly'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Contract consumer subscriptions
CREATE TABLE contract_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id),
    consumer_team_id UUID NOT NULL REFERENCES teams(id),
    purpose         TEXT,                           -- Why the consumer needs this data
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'revoked')),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_contract ON contract_subscriptions(contract_id);
CREATE INDEX idx_subscriptions_consumer ON contract_subscriptions(consumer_team_id);
```

## Lineage (OpenLineage Aligned)

```sql
-- Lineage datasets — OpenLineage Dataset entity
CREATE TABLE lineage_datasets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace       VARCHAR(500) NOT NULL,          -- OpenLineage namespace (e.g., 'snowflake://account.region')
    name            VARCHAR(500) NOT NULL,          -- Fully qualified name (e.g., 'db.schema.table')
    data_product_id UUID REFERENCES data_products(id),  -- Link to governed data product if known
    source_type     VARCHAR(100),                   -- 'snowflake', 'bigquery', 'postgres', 's3', etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace, name)
);

CREATE INDEX idx_lineage_datasets_product ON lineage_datasets(data_product_id);

-- Lineage jobs — OpenLineage Job entity
CREATE TABLE lineage_jobs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace       VARCHAR(500) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    job_type        VARCHAR(100),                   -- 'dbt_model', 'airflow_task', 'spark_job', 'sql_query'
    owner_team_id   UUID REFERENCES teams(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace, name)
);

-- Lineage runs — OpenLineage Run entity
CREATE TABLE lineage_runs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL UNIQUE,           -- OpenLineage runId
    job_id          UUID NOT NULL REFERENCES lineage_jobs(id),
    status          VARCHAR(50) NOT NULL
                    CHECK (status IN ('START', 'RUNNING', 'COMPLETE', 'ABORT', 'FAIL', 'OTHER')),
    nominal_start   TIMESTAMPTZ,
    nominal_end     TIMESTAMPTZ,
    started_at      TIMESTAMPTZ,
    ended_at        TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lineage_runs_job ON lineage_runs(job_id);
CREATE INDEX idx_lineage_runs_status ON lineage_runs(status);
CREATE INDEX idx_lineage_runs_started ON lineage_runs(started_at);

-- Lineage edges: input/output relationships between runs and datasets
CREATE TABLE lineage_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES lineage_runs(id),
    dataset_id      UUID NOT NULL REFERENCES lineage_datasets(id),
    direction       VARCHAR(10) NOT NULL CHECK (direction IN ('input', 'output')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_lineage_edges_run ON lineage_edges(run_id);
CREATE INDEX idx_lineage_edges_dataset ON lineage_edges(dataset_id);
CREATE INDEX idx_lineage_edges_direction ON lineage_edges(direction);

-- Column-level lineage
CREATE TABLE column_lineage (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES lineage_runs(id),
    source_dataset_id UUID NOT NULL REFERENCES lineage_datasets(id),
    source_column   VARCHAR(255) NOT NULL,
    target_dataset_id UUID NOT NULL REFERENCES lineage_datasets(id),
    target_column   VARCHAR(255) NOT NULL,
    transformation  TEXT,                           -- SQL expression or transformation description
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_col_lineage_source ON column_lineage(source_dataset_id, source_column);
CREATE INDEX idx_col_lineage_target ON column_lineage(target_dataset_id, target_column);
```

## Governance Policies

```sql
-- Governance policies (aligned with ISO/IEC 38505 principles)
CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    category        VARCHAR(100) NOT NULL
                    CHECK (category IN (
                        'responsibility', 'strategy', 'acquisition',
                        'performance', 'conformance', 'human_behaviour',
                        'access_control', 'data_quality', 'retention',
                        'classification', 'privacy'
                    )),
    scope           VARCHAR(50) NOT NULL DEFAULT 'organisation'
                    CHECK (scope IN ('global', 'organisation', 'domain', 'data_product')),
    enforcement     VARCHAR(50) NOT NULL DEFAULT 'advisory'
                    CHECK (enforcement IN ('mandatory', 'advisory', 'automated')),
    rule_expression TEXT,                           -- SQL or policy-as-code expression
    status          VARCHAR(50) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'active', 'retired')),
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Policy assignments to domains or data products
CREATE TABLE policy_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policies(id),
    target_type     VARCHAR(50) NOT NULL CHECK (target_type IN ('domain', 'data_product', 'global')),
    target_id       UUID,                           -- domain_id or data_product_id; NULL for global
    assigned_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_assignments_policy ON policy_assignments(policy_id);
CREATE INDEX idx_policy_assignments_target ON policy_assignments(target_type, target_id);
```

## Business Glossary and Classification

```sql
-- Business glossary terms
CREATE TABLE glossary_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    term            VARCHAR(255) NOT NULL,
    definition      TEXT NOT NULL,
    domain_id       UUID REFERENCES domains(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'approved', 'deprecated')),
    approved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_glossary_domain ON glossary_terms(domain_id);

-- Tags for classification
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    category        VARCHAR(100),                   -- 'pii', 'sensitivity', 'compliance', 'custom'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tag assignments (polymorphic: can tag data products, schemas, columns)
CREATE TABLE tag_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tag_id          UUID NOT NULL REFERENCES tags(id),
    target_type     VARCHAR(50) NOT NULL
                    CHECK (target_type IN ('data_product', 'contract_schema', 'schema_property', 'lineage_dataset')),
    target_id       UUID NOT NULL,
    assigned_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tag_assignments_tag ON tag_assignments(tag_id);
CREATE INDEX idx_tag_assignments_target ON tag_assignments(target_type, target_id);
```

## Access Control and Audit

```sql
-- Access requests for data products
CREATE TABLE access_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_product_id UUID NOT NULL REFERENCES data_products(id),
    requester_id    UUID NOT NULL REFERENCES users(id),
    requester_team_id UUID NOT NULL REFERENCES teams(id),
    purpose         TEXT NOT NULL,
    access_level    VARCHAR(50) NOT NULL CHECK (access_level IN ('read', 'write', 'admin')),
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'expired', 'revoked')),
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    expires_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_access_requests_product ON access_requests(data_product_id);
CREATE INDEX idx_access_requests_status ON access_requests(status);

-- Audit log for compliance (GDPR, SOX, HIPAA)
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id        UUID REFERENCES users(id),
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user'
                    CHECK (actor_type IN ('user', 'system', 'api_key', 'service')),
    action          VARCHAR(100) NOT NULL,          -- 'create', 'update', 'delete', 'publish', 'approve', 'access'
    target_type     VARCHAR(100) NOT NULL,          -- Entity type
    target_id       UUID NOT NULL,
    changes         JSONB,                          -- Before/after diff for update actions
    ip_address      INET,
    user_agent      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_audit_log_actor ON audit_log(actor_id);
CREATE INDEX idx_audit_log_target ON audit_log(target_type, target_id);
CREATE INDEX idx_audit_log_action ON audit_log(action);
CREATE INDEX idx_audit_log_created ON audit_log(created_at);
```

## Reference Data

```sql
-- Jurisdictions (ISO 3166-1 alpha-2)
CREATE TABLE jurisdictions (
    code            VARCHAR(2) PRIMARY KEY,         -- ISO 3166-1 alpha-2
    name            VARCHAR(255) NOT NULL,
    region          VARCHAR(100),                   -- 'EU', 'APAC', 'NA', etc.
    gdpr_applicable BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Quality dimensions (ISO/IEC 25012)
CREATE TABLE quality_dimensions (
    id              VARCHAR(50) PRIMARY KEY,        -- 'accuracy', 'completeness', 'consistency', etc.
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    iso_25012_ref   VARCHAR(100)                    -- ISO 25012 section reference
);

-- Data product jurisdiction mapping
CREATE TABLE data_product_jurisdictions (
    data_product_id UUID NOT NULL REFERENCES data_products(id) ON DELETE CASCADE,
    jurisdiction_code VARCHAR(2) NOT NULL REFERENCES jurisdictions(code),
    PRIMARY KEY (data_product_id, jurisdiction_code)
);
```

## Quality Monitoring

```sql
-- Quality check results (SLO/SLA monitoring)
CREATE TABLE quality_check_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id),
    rule_id         UUID NOT NULL REFERENCES contract_quality_rules(id),
    check_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    passed          BOOLEAN NOT NULL,
    actual_value    NUMERIC,
    expected_value  NUMERIC,
    error_message   TEXT,
    run_duration_ms INTEGER
);

CREATE INDEX idx_quality_results_contract ON quality_check_results(contract_id);
CREATE INDEX idx_quality_results_time ON quality_check_results(check_time);
CREATE INDEX idx_quality_results_passed ON quality_check_results(passed);

-- SLA breach incidents
CREATE TABLE sla_breaches (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id),
    sla_id          UUID NOT NULL REFERENCES contract_slas(id),
    breach_time     TIMESTAMPTZ NOT NULL DEFAULT now(),
    actual_value    NUMERIC NOT NULL,
    target_value    NUMERIC NOT NULL,
    resolved_at     TIMESTAMPTZ,
    resolution_note TEXT,
    notified        BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_sla_breaches_contract ON sla_breaches(contract_id);
CREATE INDEX idx_sla_breaches_time ON sla_breaches(breach_time);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Domain Management | 4 | domains, teams, team_members, users |
| Data Product Registry | 3 | data_products, data_product_versions, distributions |
| Data Contracts | 6 | contracts, schemas, properties, quality_rules, SLAs, subscriptions |
| Lineage | 5 | datasets, jobs, runs, edges, column_lineage |
| Governance | 2 | policies, policy_assignments |
| Glossary & Classification | 3 | glossary_terms, tags, tag_assignments |
| Access & Audit | 2 | access_requests, audit_log |
| Reference Data | 3 | jurisdictions, quality_dimensions, data_product_jurisdictions |
| Quality Monitoring | 2 | quality_check_results, sla_breaches |
| **Total** | **30** | |

---

## Key Design Decisions

1. **UUID primary keys throughout** — supports distributed ID generation across federated domain deployments without coordination, essential for data mesh multi-domain architecture.

2. **OpenLineage entity mapping is direct** — `lineage_datasets`, `lineage_jobs`, and `lineage_runs` map 1:1 to the OpenLineage specification entities (Dataset, Job, Run), making ingestion from any OpenLineage-compatible system a straightforward insert.

3. **ODCS v3.1.0 contract structure decomposed into tables** — rather than storing the full YAML contract as a blob, each ODCS section (schema, quality, SLA) is a separate table. This enables SQL queries like "find all contracts with failing freshness SLAs" without YAML parsing.

4. **Polymorphic tag assignments** — `tag_assignments` uses a `target_type` + `target_id` pattern rather than separate junction tables for each taggable entity. This keeps the tag system extensible without migration when new entity types are added. The trade-off is loss of foreign key enforcement on `target_id`.

5. **Audit log uses JSONB for change diffs** — the one exception to pure normalisation. Storing before/after diffs as JSONB in the audit log avoids an explosion of audit detail tables while still supporting compliance queries.

6. **DCAT 3 vocabulary fields embedded in data_products** — rather than a separate DCAT metadata table, DCAT fields (`dcat_theme`, `dcat_keyword`, `dcat_landing_page`) are columns on `data_products`. This reduces join overhead for the most common catalog queries.

7. **Contract subscriptions model producer-consumer relationships** — each consumer team explicitly subscribes to a contract version, creating an auditable record of who consumes what and why.

8. **ISO 3166 jurisdiction reference table** — enables multi-region compliance filtering (e.g., "show all data products subject to GDPR") via a simple join through `data_product_jurisdictions`.

9. **Quality check results stored as individual rows** — enables time-series analysis of quality trends per contract and rule. Partition by `check_time` for large deployments.

10. **Column-level lineage as a separate table** — follows the OpenLineage pattern of field-level lineage facets, enabling impact analysis queries like "what downstream columns are affected if I change column X?"
