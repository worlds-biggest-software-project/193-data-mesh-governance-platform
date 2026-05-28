# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Data Mesh Governance Platform · Created: 2026-05-20

## Philosophy

This model treats the governance platform as an event-sourced system where every change — publishing a data product, activating a contract, recording a lineage event, approving an access request — is stored as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimised materialised views (projections) serve query workloads.

This approach is architecturally aligned with the OpenLineage specification, which is itself event-based (RunEvents with START, RUNNING, COMPLETE, ABORT, FAIL states). A governance platform that natively stores all metadata changes as events can treat OpenLineage events, contract state transitions, and policy changes uniformly. The event store also provides a complete temporal audit trail without any additional audit logging infrastructure — every past state is recoverable by replaying events up to a given timestamp.

The trade-off is operational complexity. Materialised views must be maintained, event replay can be slow for large aggregates, and the team must understand CQRS patterns. This model is best suited when full audit trails are a hard requirement (regulated industries), when temporal queries ("what was the contract state on March 15?") are common, or when the platform needs to feed AI analytics that learn from change patterns.

**Best for:** Regulated environments requiring complete audit trails, temporal queries, and AI-powered change pattern analysis.

**Trade-offs:**
- Pro: Complete, immutable audit trail with no additional logging infrastructure
- Pro: Full temporal querying — reconstruct any entity state at any point in time
- Pro: Natural fit for OpenLineage events which are already event-shaped
- Pro: Event stream can feed real-time analytics, AI models, and downstream consumers
- Pro: Schema evolution is easier — new event types don't require table migrations
- Con: Operational complexity of maintaining materialised views
- Con: Event replay can be slow for aggregates with long event histories
- Con: Developers must understand CQRS patterns (command vs. query separation)
- Con: Cross-aggregate queries require well-maintained projections
- Con: Storage grows faster than a mutable state model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenLineage | RunEvents ingested directly into the event store as `lineage.*` event types — no transformation needed |
| ODCS v3.1.0 | Contract lifecycle modelled as event transitions: `contract.drafted`, `contract.activated`, `contract.violated`, `contract.deprecated` |
| W3C DCAT 3 | DCAT metadata stored as facets within `data_product.published` events |
| ISO/IEC 38505 | Governance policy events categorised by the six ISO 38505 principles |
| ISO/IEC 25012 | Quality dimension identifiers used in `quality.check_completed` event payloads |
| ISO 3166 | Jurisdiction codes embedded in event payloads for compliance filtering |
| FAIR Principles | FAIR assessment scores emitted as `data_product.fair_assessed` events |

---

## Event Store (Core)

```sql
-- The single source of truth: an append-only event store
-- All governance state is derived from replaying this table
CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- Aggregate identification
    aggregate_type  VARCHAR(100) NOT NULL,          -- 'data_product', 'contract', 'domain', 'lineage_run', etc.
    aggregate_id    UUID NOT NULL,                  -- ID of the entity this event belongs to
    -- Event metadata
    event_type      VARCHAR(200) NOT NULL,          -- e.g., 'data_product.published', 'contract.violated'
    event_version   INTEGER NOT NULL DEFAULT 1,     -- Schema version of the event payload
    -- Sequencing
    sequence_number BIGINT NOT NULL,                -- Per-aggregate sequence for ordering
    global_sequence BIGSERIAL NOT NULL UNIQUE,      -- Global ordering across all aggregates
    -- Payload
    payload         JSONB NOT NULL,                 -- Event-specific data
    -- Context
    actor_id        UUID,                           -- User or service that caused the event
    actor_type      VARCHAR(50) NOT NULL DEFAULT 'user'
                    CHECK (actor_type IN ('user', 'system', 'api_key', 'service', 'openlineage')),
    correlation_id  UUID,                           -- Groups related events across aggregates
    causation_id    UUID REFERENCES events(id),     -- The event that directly caused this event
    -- Timestamp
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    -- Uniqueness constraint prevents duplicate events per aggregate
    UNIQUE (aggregate_type, aggregate_id, sequence_number)
);

-- Primary query patterns
CREATE INDEX idx_events_aggregate ON events(aggregate_type, aggregate_id, sequence_number);
CREATE INDEX idx_events_type ON events(event_type);
CREATE INDEX idx_events_occurred ON events(occurred_at);
CREATE INDEX idx_events_global_seq ON events(global_sequence);
CREATE INDEX idx_events_correlation ON events(correlation_id);
CREATE INDEX idx_events_actor ON events(actor_id);

-- Partition by month for large deployments
-- CREATE TABLE events_2026_05 PARTITION OF events FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Aggregate version tracking for optimistic concurrency
CREATE TABLE aggregates (
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    UUID NOT NULL,
    current_version BIGINT NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id)
);

-- Snapshots for performance: periodically checkpoint aggregate state
-- to avoid replaying entire event history
CREATE TABLE snapshots (
    aggregate_type  VARCHAR(100) NOT NULL,
    aggregate_id    UUID NOT NULL,
    version         BIGINT NOT NULL,
    state           JSONB NOT NULL,                 -- Serialised aggregate state at this version
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (aggregate_type, aggregate_id, version)
);
```

### Event Type Taxonomy

```
-- Domain events
domain.created              -- { name, slug, description, owner_team_id, parent_id }
domain.updated              -- { field, old_value, new_value }
domain.archived             -- { reason }

-- Data product events
data_product.registered     -- { name, slug, domain_id, description, owner_team_id }
data_product.published      -- { version, dcat_metadata: { theme, keywords, landing_page } }
data_product.deprecated     -- { reason, successor_product_id }
data_product.fair_assessed  -- { findable, accessible, interoperable, reusable }
data_product.distribution_added -- { name, access_type, format, endpoint_url }

-- Contract events (ODCS v3.1.0 lifecycle)
contract.drafted            -- { odcs_version, data_product_id, schema, quality_rules, slas }
contract.activated          -- { effective_from }
contract.schema_updated     -- { old_schema, new_schema, is_breaking }
contract.quality_rule_added -- { dimension, rule_name, rule_type, expression, threshold }
contract.sla_defined        -- { metric, target_value, target_unit, measurement_window }
contract.violated           -- { rule_id, actual_value, expected_value, severity }
contract.deprecated         -- { reason, successor_contract_id }

-- Subscription events
subscription.requested      -- { contract_id, consumer_team_id, purpose }
subscription.approved       -- { approved_by }
subscription.rejected       -- { rejected_by, reason }
subscription.revoked        -- { revoked_by, reason }

-- Lineage events (OpenLineage-native)
lineage.run_started         -- { job_namespace, job_name, run_id, nominal_start, inputs, outputs }
lineage.run_completed       -- { run_id, outputs_with_facets }
lineage.run_failed          -- { run_id, error_message }
lineage.column_lineage_recorded -- { run_id, source_dataset, source_column, target_dataset, target_column }

-- Policy events
policy.created              -- { name, category, scope, enforcement, rule_expression }
policy.activated            -- {}
policy.assigned             -- { target_type, target_id }
policy.retired              -- { reason }

-- Quality monitoring events
quality.check_completed     -- { contract_id, rule_id, passed, actual_value, expected_value }
quality.sla_breached        -- { contract_id, sla_id, actual_value, target_value }
quality.sla_resolved        -- { breach_event_id, resolution_note }

-- Access events
access.requested            -- { data_product_id, requester_id, team_id, purpose, level }
access.approved             -- { reviewed_by, expires_at }
access.rejected             -- { reviewed_by, reason }
access.expired              -- {}
access.revoked              -- { revoked_by, reason }

-- Classification events
tag.assigned                -- { tag_name, target_type, target_id }
tag.removed                 -- { tag_name, target_type, target_id }
glossary.term_created       -- { term, definition, domain_id }
glossary.term_approved      -- { approved_by }
```

## Materialised Read Models (Projections)

```sql
-- Projection: current state of all data products (read model)
CREATE TABLE mv_data_products (
    id              UUID PRIMARY KEY,
    domain_id       UUID NOT NULL,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    version         VARCHAR(50),
    status          VARCHAR(50) NOT NULL,
    visibility      VARCHAR(50),
    owner_team_id   UUID NOT NULL,
    dcat_theme      VARCHAR(255),
    dcat_keywords   TEXT[],
    fair_findable   SMALLINT,
    fair_accessible SMALLINT,
    fair_interoperable SMALLINT,
    fair_reusable   SMALLINT,
    published_at    TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,               -- Track which event was last processed
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mv_products_domain ON mv_data_products(domain_id);
CREATE INDEX idx_mv_products_status ON mv_data_products(status);
CREATE INDEX idx_mv_products_slug ON mv_data_products(slug);

-- Projection: current state of all contracts
CREATE TABLE mv_contracts (
    id              UUID PRIMARY KEY,
    data_product_id UUID NOT NULL,
    version         VARCHAR(50),
    status          VARCHAR(50) NOT NULL,
    odcs_version    VARCHAR(20),
    owner_team_id   UUID NOT NULL,
    effective_from  DATE,
    effective_to    DATE,
    total_quality_rules INTEGER DEFAULT 0,
    total_slas      INTEGER DEFAULT 0,
    active_violations INTEGER DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mv_contracts_product ON mv_contracts(data_product_id);
CREATE INDEX idx_mv_contracts_status ON mv_contracts(status);

-- Projection: active subscriptions
CREATE TABLE mv_subscriptions (
    id              UUID PRIMARY KEY,
    contract_id     UUID NOT NULL,
    consumer_team_id UUID NOT NULL,
    purpose         TEXT,
    status          VARCHAR(50) NOT NULL,
    approved_by     UUID,
    approved_at     TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: lineage graph (latest known state)
CREATE TABLE mv_lineage_graph (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_dataset_namespace VARCHAR(500) NOT NULL,
    source_dataset_name      VARCHAR(500) NOT NULL,
    target_dataset_namespace VARCHAR(500) NOT NULL,
    target_dataset_name      VARCHAR(500) NOT NULL,
    job_namespace   VARCHAR(500) NOT NULL,
    job_name        VARCHAR(500) NOT NULL,
    last_run_id     UUID,
    last_run_status VARCHAR(50),
    last_run_at     TIMESTAMPTZ,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mv_lineage_source ON mv_lineage_graph(source_dataset_namespace, source_dataset_name);
CREATE INDEX idx_mv_lineage_target ON mv_lineage_graph(target_dataset_namespace, target_dataset_name);
CREATE INDEX idx_mv_lineage_job ON mv_lineage_graph(job_namespace, job_name);

-- Projection: domain tree
CREATE TABLE mv_domains (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(255) NOT NULL,
    description     TEXT,
    parent_id       UUID,
    owner_team_id   UUID,
    status          VARCHAR(50),
    product_count   INTEGER DEFAULT 0,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection: quality monitoring dashboard
CREATE TABLE mv_quality_dashboard (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    contract_id     UUID NOT NULL,
    data_product_id UUID NOT NULL,
    domain_id       UUID NOT NULL,
    check_date      DATE NOT NULL,
    total_checks    INTEGER NOT NULL DEFAULT 0,
    passed_checks   INTEGER NOT NULL DEFAULT 0,
    failed_checks   INTEGER NOT NULL DEFAULT 0,
    pass_rate       NUMERIC(5,2),
    active_breaches INTEGER NOT NULL DEFAULT 0,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (contract_id, check_date)
);

CREATE INDEX idx_mv_quality_date ON mv_quality_dashboard(check_date);
CREATE INDEX idx_mv_quality_domain ON mv_quality_dashboard(domain_id);

-- Projection: search index for unified discovery
CREATE TABLE mv_search_index (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_type     VARCHAR(100) NOT NULL,
    entity_id       UUID NOT NULL,
    domain_id       UUID,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    tags            TEXT[],
    glossary_terms  TEXT[],
    owner_team      VARCHAR(255),
    status          VARCHAR(50),
    -- Full-text search
    search_vector   TSVECTOR,
    last_event_seq  BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_mv_search_fts ON mv_search_index USING gin(search_vector);
CREATE INDEX idx_mv_search_entity ON mv_search_index(entity_type, entity_id);
CREATE INDEX idx_mv_search_domain ON mv_search_index(domain_id);
CREATE INDEX idx_mv_search_tags ON mv_search_index USING gin(tags);
```

## Temporal Query Examples

```sql
-- "What was the state of data product X on March 15, 2026?"
-- Replay all events for the aggregate up to the target date
SELECT payload
FROM events
WHERE aggregate_type = 'data_product'
  AND aggregate_id = '550e8400-e29b-41d4-a716-446655440000'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY sequence_number ASC;

-- "Show all contract violations in Q1 2026 across all domains"
SELECT
    e.aggregate_id AS contract_id,
    e.payload->>'rule_id' AS rule_id,
    e.payload->>'severity' AS severity,
    e.payload->>'actual_value' AS actual_value,
    e.occurred_at
FROM events e
WHERE e.event_type = 'contract.violated'
  AND e.occurred_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY e.occurred_at DESC;

-- "Who changed the schema of contract Y and when?"
SELECT
    e.occurred_at,
    e.actor_id,
    e.payload->>'is_breaking' AS is_breaking,
    e.payload->'new_schema' AS new_schema
FROM events e
WHERE e.aggregate_type = 'contract'
  AND e.aggregate_id = '550e8400-e29b-41d4-a716-446655440001'
  AND e.event_type = 'contract.schema_updated'
ORDER BY e.sequence_number ASC;

-- "Reconstruct the full lineage graph as it existed on a specific date"
SELECT
    payload->>'job_namespace' AS job_ns,
    payload->>'job_name' AS job_name,
    payload->'inputs' AS inputs,
    payload->'outputs' AS outputs
FROM events
WHERE event_type = 'lineage.run_completed'
  AND occurred_at <= '2026-03-15T23:59:59Z'
ORDER BY occurred_at DESC;
```

## Event Processing Infrastructure

```sql
-- Subscription tracking for projection workers
-- Each projection tracks its position in the global event stream
CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_global_sequence BIGINT NOT NULL DEFAULT 0,
    last_processed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    status          VARCHAR(50) NOT NULL DEFAULT 'running'
                    CHECK (status IN ('running', 'paused', 'rebuilding', 'error')),
    error_message   TEXT
);

-- Dead letter queue for events that failed to process
CREATE TABLE dead_letter_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id),
    projection_name VARCHAR(100) NOT NULL,
    error_message   TEXT NOT NULL,
    retry_count     INTEGER NOT NULL DEFAULT 0,
    last_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_dead_letter_projection ON dead_letter_events(projection_name);

-- Outbox pattern for reliable event publishing to external consumers
CREATE TABLE event_outbox (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id        UUID NOT NULL REFERENCES events(id),
    destination     VARCHAR(255) NOT NULL,          -- 'kafka', 'webhook', 'mcp'
    payload         JSONB NOT NULL,
    status          VARCHAR(50) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'published', 'failed')),
    retry_count     INTEGER NOT NULL DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    published_at    TIMESTAMPTZ
);

CREATE INDEX idx_outbox_status ON event_outbox(status);
CREATE INDEX idx_outbox_destination ON event_outbox(destination);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (write side) | 3 | events, aggregates, snapshots |
| Read Projections | 7 | mv_data_products, mv_contracts, mv_subscriptions, mv_lineage_graph, mv_domains, mv_quality_dashboard, mv_search_index |
| Infrastructure | 3 | projection_checkpoints, dead_letter_events, event_outbox |
| **Total** | **13** | Plus lookup tables for users, teams, jurisdictions if needed |

---

## Key Design Decisions

1. **Single event store table** — all events across all aggregate types go into one `events` table, partitioned by time. This simplifies the write path and enables global ordering via `global_sequence`. The alternative (one table per aggregate type) offers better per-type query performance but complicates cross-aggregate queries and projections.

2. **JSONB event payloads with versioned schemas** — the `event_version` field allows event schema evolution. A consumer processing `contract.drafted` events can handle v1 payloads (basic schema) differently from v2 payloads (schema + quality rules). This avoids the migration problem of normalised models.

3. **Correlation and causation IDs** — `correlation_id` groups related events across aggregates (e.g., a contract violation triggers a notification which triggers an access revocation). `causation_id` tracks direct cause-effect chains. Both are essential for debugging and compliance.

4. **OpenLineage events are first-class citizens** — `lineage.run_started`, `lineage.run_completed`, etc. are stored with `actor_type = 'openlineage'`, meaning the OpenLineage ingestion endpoint writes directly to the event store without transformation.

5. **Snapshots for performance** — the `snapshots` table periodically checkpoints aggregate state so reconstruction doesn't require replaying thousands of events. A snapshot every 100 events keeps replay time bounded.

6. **Materialised views are disposable** — every `mv_*` table can be dropped and rebuilt by replaying the event store. The `last_event_seq` column tracks which event was last processed, enabling incremental updates and full rebuilds.

7. **Outbox pattern for external integration** — rather than publishing events synchronously to Kafka or webhooks, events are written to `event_outbox` in the same transaction as the event store write. A separate publisher process drains the outbox, ensuring at-least-once delivery without distributed transactions.

8. **Full-text search projection** — `mv_search_index` with PostgreSQL `tsvector` provides unified discovery without requiring Elasticsearch. For large deployments, this can be replaced with an external search engine fed from the event stream.

9. **Temporal queries are natural** — the event store supports "what was true at time T?" queries natively. This is critical for compliance investigations ("what access did Team X have to Product Y on date Z?") and for AI models that learn from governance patterns.

10. **Lower table count, higher conceptual complexity** — at 13 tables vs. 30 in the normalised model, the schema is simpler to deploy. But the team must understand event sourcing patterns: commands produce events, projections consume events, reads go through materialised views. The CQRS learning curve is the primary trade-off.
