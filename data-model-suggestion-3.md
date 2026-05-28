# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Data Mesh Governance Platform · Created: 2026-05-20

## Philosophy

This model uses a relational backbone for core entities and relationships (domains, teams, data products, contracts) but pushes variable, domain-specific, and standards-aligned metadata into JSONB columns. The key insight is that data mesh governance spans many jurisdictions, data source types, and contract variations — and no single relational schema can anticipate all the fields needed without constant migrations.

The approach draws from how Atlan's Enterprise Data Graph and DataHub's aspect-based model work in practice: a typed entity with a fixed set of core relational columns, plus a flexible `metadata` or `aspects` JSONB column that can hold any additional structured data. ODCS v3.1.0 contracts, for example, are stored as native YAML/JSON in a JSONB column alongside relational columns for the fields most commonly queried (status, version, product reference). OpenLineage event facets — which are explicitly designed as extensible JSON blobs — are stored as JSONB without schema decomposition.

This model strikes a balance: core queries (find all published products in a domain, list active contracts) use efficient relational indexes, while standards-specific detail (the full ODCS contract, OpenLineage facets, DCAT metadata) is stored faithfully in its native JSON form without lossy normalisation.

**Best for:** Rapid MVP development, multi-region deployments with jurisdiction-specific variations, and teams that need to ingest standards-based metadata (ODCS, OpenLineage, DCAT) without decomposing every field into columns.

**Trade-offs:**
- Pro: Fastest path to MVP — core schema is small, JSONB absorbs variability
- Pro: Standards-aligned metadata stored in native format — no lossy normalisation of ODCS or OpenLineage
- Pro: Multi-region and multi-type variability handled without nullable column sprawl
- Pro: JSONB GIN indexes enable efficient containment and existence queries
- Pro: New metadata facets can be added without DDL migrations
- Con: JSONB contents are not referentially constrained by the database
- Con: Application-layer validation required for JSONB structure
- Con: JSONB query performance degrades for deeply nested paths under high cardinality
- Con: Reporting queries on JSONB fields are less intuitive than relational columns
- Con: Risk of schema drift if JSONB structures are not validated against JSON Schema

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ODCS v3.1.0 | Full contract YAML stored as JSONB in `data_contracts.contract_body`; key fields extracted to relational columns for indexing |
| OpenLineage | Lineage events stored with facets as native JSONB; core fields (namespace, name, runId) relational |
| W3C DCAT 3 | DCAT metadata stored as JSONB in `data_products.dcat_metadata` following the DCAT 3 JSON-LD structure |
| OpenMetadata Standard | Entity metadata facets follow the OpenMetadata aspect pattern: typed JSONB blobs attached to entities |
| ISO/IEC 25012 | Quality dimensions referenced by ID in quality rules within contract JSONB |
| ISO 3166 | Jurisdiction array stored in `data_products.jurisdictions` as JSONB array of ISO 3166-1 codes |
| JSON Schema Draft 2020-12 | All JSONB columns validated against JSON Schema definitions at the application layer |

---

## Core Entities

```sql
-- Users (relational — universally queried)
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    external_id     VARCHAR(255) UNIQUE,
    email           VARCHAR(255) NOT NULL UNIQUE,
    display_name    VARCHAR(255) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    profile         JSONB NOT NULL DEFAULT '{}',
    -- profile example: { "avatar_url": "...", "timezone": "US/Pacific", "preferences": {...} }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Teams
CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    contact         JSONB NOT NULL DEFAULT '{}',
    -- contact example: { "email": "team@co.com", "slack": "#data-platform", "pagerduty": "..." }
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

-- Domains
CREATE TABLE domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    description     TEXT,
    parent_id       UUID REFERENCES domains(id),
    owner_team_id   UUID NOT NULL REFERENCES teams(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    -- config example: {
    --   "governance_level": "strict",
    --   "default_classification": "internal",
    --   "required_contract_sections": ["schema", "quality", "sla"],
    --   "approval_workflow": "two_person"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_domains_parent ON domains(parent_id);
CREATE INDEX idx_domains_slug ON domains(slug);
```

## Data Products (Relational Core + JSONB Metadata)

```sql
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

    -- DCAT 3 metadata stored as native JSON-LD structure
    dcat_metadata   JSONB NOT NULL DEFAULT '{}',
    -- dcat_metadata example: {
    --   "@type": "dcat:Resource",
    --   "dcat:theme": ["http://publications.europa.eu/resource/authority/data-theme/ECON"],
    --   "dcat:keyword": ["revenue", "quarterly", "finance"],
    --   "dcat:landingPage": "https://catalog.example.com/products/quarterly-revenue",
    --   "dcterms:accrualPeriodicity": "http://purl.org/cld/freq/quarterly",
    --   "dcterms:spatial": "http://sws.geonames.org/6252001/",
    --   "dcterms:temporal": { "dcat:startDate": "2020-01-01", "dcat:endDate": "2026-12-31" }
    -- }

    -- FAIR scores
    fair_scores     JSONB NOT NULL DEFAULT '{}',
    -- fair_scores example: { "findable": 85, "accessible": 90, "interoperable": 70, "reusable": 80 }

    -- Jurisdictions (ISO 3166-1 alpha-2 codes)
    jurisdictions   JSONB NOT NULL DEFAULT '[]',
    -- jurisdictions example: ["US", "DE", "GB"]

    -- Distributions / access endpoints
    distributions   JSONB NOT NULL DEFAULT '[]',
    -- distributions example: [
    --   { "name": "REST API", "type": "api", "format": "json", "url": "https://api.example.com/v1/revenue" },
    --   { "name": "Parquet Export", "type": "file", "format": "parquet", "url": "s3://bucket/revenue/" },
    --   { "name": "Delta Share", "type": "delta_sharing", "share": "quarterly_revenue" }
    -- ]

    -- Extensible metadata facets (custom per domain)
    custom_metadata JSONB NOT NULL DEFAULT '{}',
    -- custom_metadata example: { "cost_center": "CC-1234", "data_sensitivity": "high", "retention_days": 365 }

    published_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (domain_id, slug)
);

CREATE INDEX idx_products_domain ON data_products(domain_id);
CREATE INDEX idx_products_status ON data_products(status);
CREATE INDEX idx_products_owner ON data_products(owner_team_id);
CREATE INDEX idx_products_jurisdictions ON data_products USING gin(jurisdictions);
CREATE INDEX idx_products_dcat ON data_products USING gin(dcat_metadata);
CREATE INDEX idx_products_custom ON data_products USING gin(custom_metadata);
```

## Data Contracts (ODCS v3.1.0 Native)

```sql
CREATE TABLE data_contracts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_product_id UUID NOT NULL REFERENCES data_products(id),
    version         VARCHAR(50) NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'draft'
                    CHECK (status IN ('draft', 'active', 'deprecated', 'violated')),
    owner_team_id   UUID NOT NULL REFERENCES teams(id),

    -- The full ODCS v3.1.0 contract stored as native JSONB
    -- This preserves the complete contract without lossy decomposition
    contract_body   JSONB NOT NULL,
    -- contract_body example (ODCS v3.1.0 structure): {
    --   "apiVersion": "v3.1.0",
    --   "kind": "DataContract",
    --   "id": "urn:datacontract:quarterly-revenue:v2",
    --   "info": {
    --     "title": "Quarterly Revenue Data Product",
    --     "version": "2.0.0",
    --     "description": "Aggregated quarterly revenue by region and product line",
    --     "owner": { "team": "finance-data", "email": "finance-data@example.com" }
    --   },
    --   "terms": { "usage": "Internal analytics only", "limitations": "No PII included" },
    --   "schema": [
    --     {
    --       "name": "quarterly_revenue",
    --       "type": "table",
    --       "properties": [
    --         { "name": "quarter", "type": "date", "required": true, "description": "Quarter start date" },
    --         { "name": "region", "type": "string", "required": true, "classification": "internal" },
    --         { "name": "revenue_usd", "type": "decimal", "required": true, "precision": 2 }
    --       ]
    --     }
    --   ],
    --   "quality": [
    --     { "dimension": "completeness", "rule": "revenue_usd IS NOT NULL", "threshold": 99.9, "unit": "percent" },
    --     { "dimension": "freshness", "rule": "max_age", "threshold": 24, "unit": "hours" }
    --   ],
    --   "sla": [
    --     { "metric": "availability", "target": 99.5, "unit": "percent", "window": "monthly" },
    --     { "metric": "freshness", "target": 60, "unit": "minutes", "window": "daily" }
    --   ]
    -- }

    -- Extracted fields for efficient relational queries
    effective_from  DATE,
    effective_to    DATE,
    schema_hash     VARCHAR(64),                    -- SHA-256 of the schema section for change detection
    quality_rule_count INTEGER DEFAULT 0,
    sla_count       INTEGER DEFAULT 0,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (data_product_id, version)
);

CREATE INDEX idx_contracts_product ON data_contracts(data_product_id);
CREATE INDEX idx_contracts_status ON data_contracts(status);
CREATE INDEX idx_contracts_body ON data_contracts USING gin(contract_body);

-- Contract subscriptions
CREATE TABLE contract_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id),
    consumer_team_id UUID NOT NULL REFERENCES teams(id),
    purpose         TEXT,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'approved', 'rejected', 'revoked')),
    approved_by     UUID REFERENCES users(id),
    approved_at     TIMESTAMPTZ,
    access_config   JSONB NOT NULL DEFAULT '{}',
    -- access_config example: { "level": "read", "expires_at": "2027-01-01", "ip_whitelist": ["10.0.0.0/8"] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_subscriptions_contract ON contract_subscriptions(contract_id);
CREATE INDEX idx_subscriptions_consumer ON contract_subscriptions(consumer_team_id);
```

## Lineage (OpenLineage-Native JSONB)

```sql
-- Lineage events stored with OpenLineage structure preserved in JSONB
CREATE TABLE lineage_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Core relational fields for indexing
    run_id          UUID NOT NULL,
    job_namespace   VARCHAR(500) NOT NULL,
    job_name        VARCHAR(500) NOT NULL,
    event_type      VARCHAR(50) NOT NULL
                    CHECK (event_type IN ('START', 'RUNNING', 'COMPLETE', 'ABORT', 'FAIL', 'OTHER')),
    event_time      TIMESTAMPTZ NOT NULL,

    -- Full OpenLineage event as native JSONB
    -- Preserves all facets without decomposition
    event_body      JSONB NOT NULL,
    -- event_body example: {
    --   "eventType": "COMPLETE",
    --   "eventTime": "2026-05-20T10:30:00Z",
    --   "run": { "runId": "...", "facets": { "nominalTime": {...}, "parent": {...} } },
    --   "job": { "namespace": "airflow://prod", "name": "finance.quarterly_revenue", "facets": {...} },
    --   "inputs": [
    --     { "namespace": "postgres://prod", "name": "raw.transactions",
    --       "facets": { "schema": { "fields": [...] }, "dataQualityMetrics": {...} } }
    --   ],
    --   "outputs": [
    --     { "namespace": "snowflake://acme.us-east-1", "name": "analytics.quarterly_revenue",
    --       "facets": { "schema": { "fields": [...] }, "columnLineage": {...} } }
    --   ]
    -- }

    -- Extracted input/output dataset references for graph queries
    input_datasets  JSONB NOT NULL DEFAULT '[]',
    -- input_datasets example: [{"namespace": "postgres://prod", "name": "raw.transactions"}]
    output_datasets JSONB NOT NULL DEFAULT '[]',

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (event_time);

-- Create partitions
CREATE TABLE lineage_events_2026_q2 PARTITION OF lineage_events
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');
CREATE TABLE lineage_events_2026_q3 PARTITION OF lineage_events
    FOR VALUES FROM ('2026-07-01') TO ('2026-10-01');

CREATE INDEX idx_lineage_run ON lineage_events(run_id);
CREATE INDEX idx_lineage_job ON lineage_events(job_namespace, job_name);
CREATE INDEX idx_lineage_type ON lineage_events(event_type);
CREATE INDEX idx_lineage_time ON lineage_events(event_time);
CREATE INDEX idx_lineage_inputs ON lineage_events USING gin(input_datasets);
CREATE INDEX idx_lineage_outputs ON lineage_events USING gin(output_datasets);

-- Materialised dataset registry (built from lineage events)
CREATE TABLE datasets (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    namespace       VARCHAR(500) NOT NULL,
    name            VARCHAR(500) NOT NULL,
    data_product_id UUID REFERENCES data_products(id),
    source_type     VARCHAR(100),
    latest_schema   JSONB,                         -- Most recent schema facet
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (namespace, name)
);

CREATE INDEX idx_datasets_product ON datasets(data_product_id);
```

## Governance, Glossary, and Classification

```sql
-- Policies with JSONB rule definitions
CREATE TABLE policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL UNIQUE,
    category        VARCHAR(100) NOT NULL,
    scope           VARCHAR(50) NOT NULL DEFAULT 'organisation',
    enforcement     VARCHAR(50) NOT NULL DEFAULT 'advisory',
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    -- Policy rules as structured JSONB
    rules           JSONB NOT NULL DEFAULT '[]',
    -- rules example: [
    --   { "type": "classification_required", "applies_to": "data_product", "min_level": "internal" },
    --   { "type": "contract_required", "applies_to": "data_product", "when_visibility": "organisation" },
    --   { "type": "freshness_sla_required", "max_hours": 24 }
    -- ]
    created_by      UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Policy assignments
CREATE TABLE policy_assignments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    policy_id       UUID NOT NULL REFERENCES policies(id),
    target_type     VARCHAR(50) NOT NULL,
    target_id       UUID,
    assigned_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_policy_assign_target ON policy_assignments(target_type, target_id);

-- Glossary terms
CREATE TABLE glossary_terms (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    term            VARCHAR(255) NOT NULL,
    definition      TEXT NOT NULL,
    domain_id       UUID REFERENCES domains(id),
    status          VARCHAR(50) NOT NULL DEFAULT 'draft',
    synonyms        JSONB NOT NULL DEFAULT '[]',    -- ["revenue", "income", "sales"]
    related_terms   JSONB NOT NULL DEFAULT '[]',    -- [{"id": "...", "relationship": "broader"}]
    approved_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tags (simple relational)
CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL UNIQUE,
    category        VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE tag_assignments (
    tag_id          UUID NOT NULL REFERENCES tags(id),
    target_type     VARCHAR(50) NOT NULL,
    target_id       UUID NOT NULL,
    assigned_by     UUID REFERENCES users(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tag_id, target_type, target_id)
);
```

## Quality Monitoring and Audit

```sql
-- Quality check results
CREATE TABLE quality_checks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL REFERENCES data_contracts(id),
    check_time      TIMESTAMPTZ NOT NULL DEFAULT now(),
    results         JSONB NOT NULL,
    -- results example: {
    --   "total_rules": 5, "passed": 4, "failed": 1,
    --   "details": [
    --     { "rule": "completeness", "expression": "revenue_usd IS NOT NULL", "passed": true, "value": 99.95 },
    --     { "rule": "freshness", "expression": "max_age", "passed": false, "value": 120, "threshold": 60, "unit": "minutes" }
    --   ]
    -- }
    overall_passed  BOOLEAN NOT NULL,
    run_duration_ms INTEGER
) PARTITION BY RANGE (check_time);

CREATE TABLE quality_checks_2026_q2 PARTITION OF quality_checks
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE INDEX idx_quality_contract ON quality_checks(contract_id);
CREATE INDEX idx_quality_time ON quality_checks(check_time);
CREATE INDEX idx_quality_passed ON quality_checks(overall_passed);

-- Audit log
CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    actor_id        UUID,
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user',
    action          VARCHAR(100) NOT NULL,
    target_type     VARCHAR(100) NOT NULL,
    target_id       UUID NOT NULL,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details example: { "before": {...}, "after": {...}, "reason": "..." }
    ip_address      INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (created_at);

CREATE TABLE audit_log_2026_q2 PARTITION OF audit_log
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

CREATE INDEX idx_audit_target ON audit_log(target_type, target_id);
CREATE INDEX idx_audit_actor ON audit_log(actor_id);
CREATE INDEX idx_audit_time ON audit_log(created_at);

-- Access requests
CREATE TABLE access_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    data_product_id UUID NOT NULL REFERENCES data_products(id),
    requester_id    UUID NOT NULL REFERENCES users(id),
    requester_team_id UUID NOT NULL REFERENCES teams(id),
    purpose         TEXT NOT NULL,
    request_config  JSONB NOT NULL DEFAULT '{}',
    -- request_config example: { "access_level": "read", "duration_days": 90, "use_case": "quarterly reporting" }
    status          VARCHAR(50) NOT NULL DEFAULT 'pending',
    reviewed_by     UUID REFERENCES users(id),
    reviewed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_access_product ON access_requests(data_product_id);
CREATE INDEX idx_access_status ON access_requests(status);
```

## JSONB Query Examples

```sql
-- Find all data products with GDPR-applicable jurisdictions
SELECT id, name, jurisdictions
FROM data_products
WHERE jurisdictions @> '["DE"]'::jsonb
   OR jurisdictions @> '["FR"]'::jsonb;

-- Find all contracts with freshness SLAs under 60 minutes
SELECT dc.id, dp.name, sla.value
FROM data_contracts dc
JOIN data_products dp ON dp.id = dc.data_product_id
CROSS JOIN LATERAL jsonb_array_elements(dc.contract_body->'sla') AS sla(value)
WHERE sla.value->>'metric' = 'freshness'
  AND (sla.value->>'target')::numeric < 60;

-- Find all data products with a specific DCAT theme
SELECT id, name
FROM data_products
WHERE dcat_metadata @> '{"dcat:theme": ["http://publications.europa.eu/resource/authority/data-theme/ECON"]}'::jsonb;

-- Trace lineage: find all upstream datasets for a given output
SELECT
    input_ds->>'namespace' AS source_namespace,
    input_ds->>'name' AS source_name,
    le.job_namespace,
    le.job_name,
    le.event_time
FROM lineage_events le
CROSS JOIN LATERAL jsonb_array_elements(le.output_datasets) AS out_ds(value)
CROSS JOIN LATERAL jsonb_array_elements(le.input_datasets) AS input_ds(value)
WHERE out_ds.value->>'name' = 'analytics.quarterly_revenue'
  AND le.event_type = 'COMPLETE'
ORDER BY le.event_time DESC
LIMIT 10;

-- Find contracts with violated quality rules
SELECT dc.id, dp.name, dc.status,
       jsonb_array_length(dc.contract_body->'quality') AS total_rules
FROM data_contracts dc
JOIN data_products dp ON dp.id = dc.data_product_id
WHERE dc.status = 'violated';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Core Entities | 4 | users, teams, team_members, domains |
| Data Products | 1 | Single table with JSONB for distributions, DCAT, FAIR, jurisdictions |
| Data Contracts | 2 | contracts (with ODCS JSONB body), subscriptions |
| Lineage | 2 | lineage_events (partitioned, OpenLineage JSONB), datasets |
| Governance | 4 | policies, policy_assignments, glossary_terms, tags + tag_assignments |
| Monitoring & Audit | 3 | quality_checks (partitioned), audit_log (partitioned), access_requests |
| **Total** | **16** | Plus partition tables |

---

## Key Design Decisions

1. **ODCS contract stored as native JSONB** — the full ODCS v3.1.0 contract lives in `contract_body` as the authoritative representation. Key fields (`status`, `version`, `schema_hash`) are extracted to relational columns for indexing. This means the contract can be exported as valid ODCS YAML at any time without reconstruction from normalised tables.

2. **OpenLineage events preserved verbatim** — `lineage_events.event_body` contains the complete OpenLineage RunEvent including all facets. No facet information is lost during ingestion. The `input_datasets` and `output_datasets` JSONB arrays are extracted for efficient graph queries via GIN indexes.

3. **DCAT 3 metadata as JSON-LD** — `dcat_metadata` follows the DCAT 3 JSON-LD structure, meaning it can be serialised directly as linked data for interoperability with government data portals and DCAT-US 3.0 compliant systems.

4. **Time-series tables partitioned by quarter** — `lineage_events`, `quality_checks`, and `audit_log` are partitioned by time range. This enables efficient retention policies (drop old partitions) and faster range scans for temporal queries.

5. **GIN indexes on all JSONB columns** — enables PostgreSQL containment (`@>`) and existence (`?`) operators for efficient filtering. The `idx_products_jurisdictions` GIN index, for example, makes "find all products in GDPR jurisdictions" a fast index scan.

6. **Domain configuration as JSONB** — `domains.config` allows each domain to customise governance rules (required contract sections, approval workflows, default classification) without adding domain-specific columns.

7. **Policy rules as JSONB arrays** — policies can define heterogeneous rule types (classification required, contract required, freshness SLA required) in a single JSONB array. New rule types are added without schema changes.

8. **16 tables vs. 30 in the normalised model** — the JSONB approach consolidates many junction and detail tables into JSONB columns, cutting the table count nearly in half. This simplifies deployment and reduces migration overhead at the cost of database-level constraint enforcement.

9. **Application-layer JSON Schema validation** — since JSONB columns are not referentially constrained, the application must validate `contract_body` against the ODCS v3.1.0 JSON Schema, `event_body` against the OpenLineage JSON Schema, and `dcat_metadata` against the DCAT 3 profile. This is a critical requirement to prevent schema drift.

10. **Glossary synonyms and related terms as JSONB** — avoids separate synonym and relationship tables. The JSONB structure supports thesaurus-style relationships (broader, narrower, related) inline.
