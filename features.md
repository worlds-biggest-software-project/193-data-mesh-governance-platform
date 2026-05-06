# Data Mesh Governance Platform — Feature & Functionality Survey

> Candidate #193 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Collibra Data Intelligence Cloud | Enterprise data governance & catalog | Commercial SaaS | https://www.collibra.com |
| Atlan | Active metadata platform / data catalog | Commercial SaaS | https://atlan.com |
| DataHub (LinkedIn / Acryl) | Open-source metadata platform | Apache 2.0 / Commercial (Acryl Cloud) | https://datahub.com |
| OpenMetadata | Unified metadata platform | Apache 2.0 / Managed cloud | https://open-metadata.org |
| Databricks Unity Catalog | Lakehouse governance layer | Commercial (bundled) | https://www.databricks.com/product/unity-catalog |
| Snowflake Horizon Catalog | Cloud DW governance layer | Commercial (bundled) | https://www.snowflake.com/en/product/features/horizon |
| Alation | Data catalog & governance | Commercial SaaS | https://www.alation.com |
| Select Star | Automated data catalog & lineage | Commercial SaaS | https://www.selectstar.com |
| Data Contract CLI | Open-source contract enforcement tool | Apache 2.0 | https://cli.datacontract.com |
| Soda Core | Data quality / contracts engine | Apache 2.0 | https://github.com/sodadata/soda-core |

---

## Feature Analysis by Solution

### Collibra Data Intelligence Cloud

**Core features**
- Business glossary and data dictionary management
- Policy management with automated workflow enforcement
- Data lineage tracking (end-to-end, multi-system)
- Data catalog with classification and tagging
- Access governance with integration to cloud IAM (AWS, Azure, Google)
- AI governance module for cataloging ML models and agents
- Data quality monitoring with natural-language quality rule definition
- Compliance reporting for GDPR, HIPAA, CCPA, SOX
- Unstructured data governance (documents, transcripts — via 2025 Deasy Labs acquisition)
- Audit trail for regulatory readiness

**Differentiating features**
- Authoritative source for data classification metadata that syncs to cloud access control systems
- AI governance extending lineage from source data through model training, inference, and deployment
- 2025.10 release with GA features scheduled for March 2026

**UX patterns**
- Complex workflow-driven UI aimed at governance stewards and CDOs
- High configuration burden; requires dedicated admin personnel
- Dashboards for policy compliance status and governance maturity

**Integration points**
- REST API and Knowledge Graph API (GraphQL) — OpenAPI 3.0 documented
- Java API in addition to REST
- Connectors to 100+ data sources; deep integration with Snowflake, Databricks, AWS
- Import API for bulk asset creation via integration partners

**Known gaps**
- High total cost of ownership; 3–9 month implementation cycles reported
- Governance depth is strong but UX complexity creates adoption barriers
- Limited self-serve data product publishing for domain teams
- Weak native support for data contract enforcement between producers and consumers

**Licence / IP notes**
- Proprietary commercial SaaS. No open-source components. Patent portfolio around workflow automation.

---

### Atlan

**Core features**
- Active metadata engine continuously parsing query activity and dbt runs
- Column-level lineage across ingestion, transformation, and BI layers
- Natural-language search understanding context and intent
- Bidirectional integration with Slack, Jira, and collaboration tools (embedded collaboration)
- 100+ certified connectors (Snowflake, Databricks, Redshift, BigQuery, dbt, Fivetran, Tableau, Looker, Power BI)
- Data access request and approval workflows
- PII tagging and classification
- Data quality integration (external tools)

**Differentiating features**
- Enterprise Data Graph: unified, traversable metadata layer queryable by data teams and AI agents
- Production deployment in 4–6 weeks vs 3–9 months for legacy platforms
- Embedded collaboration — approvals happen inside Slack without leaving a data asset
- Gartner Magic Quadrant Leader and Forrester Wave Leader 2025

**UX patterns**
- Third-generation catalog design influenced by GitHub, Figma, Slack, Notion
- Progressive disclosure: simple search-first experience; deeper lineage and governance on demand
- Self-service access requests via contextual links in any data tool

**Integration points**
- REST API documented at developer.atlan.com
- Java, Python SDKs (atlanhq/atlan-java on GitHub)
- GraphQL API
- MCP-compatible metadata exposure planned

**Known gaps**
- Data quality testing is integration-dependent, not native
- Policy enforcement engine less mature than Collibra for complex rule hierarchies
- Data contract management not natively supported; relies on external tools
- Domain-level governance configuration requires platform admin involvement

**Licence / IP notes**
- Proprietary commercial SaaS. Open-source connectors under various licences.

---

### DataHub (LinkedIn / Acryl)

**Core features**
- Metadata graph covering datasets, pipelines, dashboards, ML features, ML models, users, and groups
- 80+ ingestion connectors via batch and real-time streaming (Kafka)
- Domain-oriented ownership: maintainers, documentation, and governance rules per asset
- Tags, glossary terms, and business domains
- Column-level lineage with graph visualisation
- Fine-grained access policies
- Federated metadata services: domain teams can run their own metadata service syncing to central graph
- Data products entity model
- Assertions / data quality checks (Acryl managed tier)

**Differentiating features**
- Federated architecture native to data mesh: each domain team owns their metadata service; Kafka syncs to global index
- 12,000+ GitHub stars; largest open-source data catalog community
- Python SDK (acryl-datahub package) and Java SDK for programmatic metadata management
- GraphQL API as primary public API; REST.li as internal persistence layer

**UX patterns**
- Developer-first; strong CLI and SDK for pipeline integration
- Web UI for search, lineage, and governance browsing
- Acryl Cloud adds managed observability dashboards

**Integration points**
- GraphQL API at /api/graphql; interactive GraphiQL explorer
- Python SDK (acryl-datahub), Java SDK
- Kafka-based real-time streaming ingestion
- Supports 80+ source connectors

**Known gaps**
- Enterprise policy enforcement, RBAC at column level, and data contracts require Acryl managed tier
- Self-hosted deployment carries significant operational overhead
- UI less polished than Atlan or Alation for business users
- Data quality framework less integrated than OpenMetadata

**Licence / IP notes**
- Core: Apache 2.0. Acryl Cloud enterprise features are proprietary. No known patents restricting open-source use.

---

### OpenMetadata

**Core features**
- Unified platform covering discovery, lineage, quality, collaboration, and governance
- 80+ source connectors (databases, dashboards, messaging, pipelines)
- Built-in data quality framework: out-of-the-box tests, custom test creation, native data profiling
- Column-level lineage with no-code lineage editor for manual overrides
- PII tagging and sensitivity classification
- Data contract support with schema, quality rules, and SLA definitions (native)
- Observability alerts for quality degradation
- Test suite ownership mapped to domain teams

**Differentiating features**
- Native data contract support with schema + quality + SLA in a single entity
- No-code lineage editor — unique among open-source tools
- All-in-one: no need to integrate separate quality engine
- OpenMetadata Standard schema specification (community open standard)

**UX patterns**
- Clean modern UI with collaborative annotation
- Role-based views for data stewards vs analysts vs engineers
- Integrated quality dashboards alongside catalog entries

**Integration points**
- REST API with Swagger/OpenAPI documentation at docs.open-metadata.org
- Python SDK, Java SDK, Go SDK
- Webhook-based event system for triggering downstream actions

**Known gaps**
- Smaller community than DataHub (fewer enterprise adopters)
- Federated multi-domain architecture less mature than DataHub
- Limited workflow automation for access governance vs Collibra
- Managed cloud offering less established

**Licence / IP notes**
- Apache 2.0. No known patent claims. Community-governed schema standard.

---

### Databricks Unity Catalog

**Core features**
- Unified governance for tables, files, ML models, dashboards, and volumes (unstructured data)
- Fine-grained access control (table, column, row level)
- Attribute-Based Access Control (ABAC) — enforces access dynamically on user/data/environment attributes
- Column-level lineage native to the Databricks platform
- Certified business metrics with auditing and lineage
- Iceberg catalog federation (AWS Glue, Hive Metastore, Snowflake Horizon interoperability)
- AI-native governance: proactive, intelligent, scalable vs reactive human-based
- Data sharing via Delta Sharing protocol

**Differentiating features**
- Only unified governance solution covering data and AI assets in the same catalog
- ABAC at scale — policy decisions driven by attributes, not static role lists
- Iceberg federation enables cross-platform governance without data movement
- Native integration with Delta Lake, Apache Spark, MLflow, and notebooks

**UX patterns**
- Aimed at data engineers and data scientists; less business-user focused
- Catalog Explorer UI for browsing and governance; programmatic management via SDK/REST
- AI Copilot for Horizon Catalog introduced at Summit 2025

**Integration points**
- Databricks REST API (docs.databricks.com/api/workspace)
- Databricks SDK for Python (databricks-sdk package), Java SDK
- Delta Sharing open protocol for cross-platform data sharing
- Integrations with Atlan, Collibra for external catalog federation

**Known gaps**
- Locked into Databricks ecosystem; limited value for non-Databricks workloads
- No native support for domain-oriented data product publishing outside Databricks
- Business user experience less developed than Atlan or Alation
- Data contract enforcement not natively supported

**Licence / IP notes**
- Proprietary commercial feature of Databricks platform. Delta Sharing is Apache 2.0 open source. Iceberg is Apache 2.0.

---

### Snowflake Horizon Catalog

**Core features**
- Built-in data discovery and classification for all Snowflake objects
- Sensitive data classification with automatic tag propagation
- Object tagging and column-level tagging
- Data lineage within Snowflake
- Internal Marketplace: curated directory of data products within an organisation
- Organisational Profiles and Listings for secure internal sharing with preserved tags and permissions
- Policy-as-code: row/column access policies defined in SQL
- Cross-cloud data sharing via Snowflake Marketplace
- Apache Iceberg REST Catalog federation with external catalogs

**Differentiating features**
- Tags and access rules travel with data products through internal marketplace — governance-preserving sharing
- Copilot for Horizon Catalog (AI-assisted governance): introduced at Summit 2025
- Iceberg Catalog-Linked Databases for external Iceberg REST Catalog sync without data copying

**UX patterns**
- Integrated directly into Snowsight UI — zero additional tooling for Snowflake shops
- SQL-native policy definition appeals to data engineers
- Business user self-service catalog browsing within Snowsight

**Integration points**
- Snowflake REST API, SQL, Python connector, JDBC/ODBC
- Snowpark SDK (Python, Java, Scala)
- Native integration with dbt, Fivetran, Tableau, Sigma
- Delta Sharing compatible for cross-platform sharing

**Known gaps**
- Governance applies only to Snowflake objects — no cross-platform lineage
- Data contract enforcement between domains not supported
- Domain-ownership model not formalised; no data product publishing API
- No open API for external catalog consumers

**Licence / IP notes**
- Proprietary commercial feature bundled with Snowflake. Delta Sharing is Apache 2.0.

---

### Alation

**Core features**
- Data catalog with ML-driven automated curation (behavioural analysis of query patterns)
- Business glossary and data stewardship workflows
- Collaborative annotation (trusted flags, wiki-style documentation)
- Lineage tracking with regulatory audit trail
- Policy management with access approval workflows
- Governance automation: centralises policies, automates stewardship, enforces access and masking
- Data products marketplace (Agentic Platform — 2025/2026)
- AI model cataloging and governance

**Differentiating features**
- Behavioural intelligence: usage frequency, popularity, and endorsements surface the most trusted assets
- Agentic Platform (2025): AI-driven agents for automated metadata management and data product marketplace
- Strong data steward community with curated governance use cases

**UX patterns**
- Collaboration-first design with threaded conversations on data assets
- Governance dashboards for compliance officers
- Search results ranked by data popularity and trust signals

**Integration points**
- REST API and Python SDK
- Connectors to major warehouses, BI tools, and ETL platforms
- Slack and Jira integration for workflow notifications

**Known gaps**
- Policy enforcement less mature than Collibra for highly regulated industries
- Column-level lineage weaker than Atlan or Select Star
- Data contract enforcement not natively supported
- Agentic platform is early-stage as of 2026

**Licence / IP notes**
- Proprietary commercial SaaS. No open-source components.

---

### Select Star

**Core features**
- Automatic data catalog populated from query log analysis (no manual curation required)
- Column-level lineage derived from actual query activity
- Entity Relationship Diagrams auto-generated
- Auto-populated documentation from query usage patterns
- Usage analytics: table and column-level popularity tracking
- Semantic modelling layer for business-friendly labels
- AI assistants for metadata enrichment and data Q&A
- MCP server for AI agent integration with catalog

**Differentiating features**
- 24-hour time-to-value: automated catalog + column-level lineage within one day of connection
- Query-log-driven lineage: lineage reflects real usage, not declared metadata
- MCP server exposing catalog context to AI coding assistants and LLM agents

**UX patterns**
- Low-friction onboarding: connect and auto-populate; minimal configuration
- Usage-ranked search results prioritise popular and recently queried assets
- Embedded AI assistant for natural language data Q&A

**Integration points**
- REST API
- Integrations with Snowflake, BigQuery, Redshift, Looker, Tableau, dbt
- MCP server for AI agent metadata access

**Known gaps**
- Governance policy engine is lightweight — not suitable for complex regulatory enforcement
- No business glossary or data contract management
- Domain-ownership model absent
- Limited workflow automation for access management

**Licence / IP notes**
- Proprietary commercial SaaS.

---

### Data Contract CLI / Open Data Contract Standard (ODCS)

**Core features**
- CLI tool for linting, testing, and enforcing data contracts in YAML
- Native support for Open Data Contract Standard (ODCS) v3.1.0
- Schema validation against live data sources
- Quality rule execution (assertions, completeness, freshness)
- Integration with dbt, Soda, and CI/CD pipelines
- Contract import/export between teams and systems

**Differentiating features**
- Implements the ODCS v3.1.0 standard (Linux Foundation / Bitol governance)
- Deprecates the old Data Contract Specification — single open standard for the industry
- Supports complex data structures: JSON, Avro, Protobuf model definitions

**UX patterns**
- Developer CLI tool; CI/CD-first design
- YAML-based contract definition in version control (GitOps)
- No GUI — intended as a building block for platform tooling

**Integration points**
- Open Data Contract Standard v3.1.0 (bitol-io.github.io)
- dbt model contracts, Soda Core quality engine
- GitHub Actions, GitLab CI integration examples

**Known gaps**
- No governance UI or catalog integration out of the box
- No producer/consumer discovery or domain registry
- No SLO alerting or monitoring layer

**Licence / IP notes**
- Apache 2.0. ODCS standard governed by Linux Foundation / Bitol. No patent concerns.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Data asset discovery via search (keyword and semantic)
- Business glossary and data dictionary management
- Column-level data lineage (automated, not manual)
- Data classification and PII/sensitivity tagging
- Role-based access control with policy enforcement
- Integration connectors to major data warehouses and BI tools
- Audit trail for compliance reporting (GDPR, HIPAA, SOX)

### Differentiating Features
- Federated metadata architecture supporting domain-owned governance (DataHub)
- Active metadata that pushes context to where data is consumed (Atlan)
- Native data contract authoring, validation, and enforcement (OpenMetadata, Data Contract CLI)
- AI-native governance with proactive quality monitoring and anomaly detection (Databricks, Collibra 2025)
- Usage-based lineage derived from query logs rather than declared metadata (Select Star)
- MCP server exposing catalog metadata to AI agents and coding assistants (Select Star, Collibra via third-party)

### Underserved Areas / Opportunities
- **End-to-end data contract lifecycle management** across producers and consumers with SLO alerting — no single tool handles authoring, enforcement, versioning, and incident routing in one place
- **Cross-platform federated governance** that spans Snowflake, Databricks, and open lakes without requiring each vendor's native tool
- **Domain self-service data product publishing** with automated quality checks, contract generation, and discovery registration — currently requires orchestrating 4–6 separate tools
- **Natural-language policy definition and enforcement** — most tools require SQL or YAML; AI-assisted policy authoring is nascent
- **Data product SLO/SLA monitoring with automated consumer notification** when contracts are breached
- **Lightweight federation for SMB and mid-market** — existing federated tools (DataHub) are operationally heavy; commercial tools are priced for large enterprises
- **AI asset governance** (models, agents, training datasets) integrated with data governance — Collibra and Databricks are early; an open-source alternative is absent

### AI-Augmentation Candidates
- Automated data contract generation from query patterns, schema diffs, and pipeline metadata
- NLP-powered policy authoring: describe a policy in plain English; AI generates the enforcement rule
- Proactive contract breach detection with root-cause attribution (quality drop → upstream change)
- Intelligent domain boundary recommendation from team structure and query co-access patterns
- Auto-enrichment of data asset documentation from code comments, dbt descriptions, and query logs
- Automated lineage gap detection and repair suggestions

---

## Legal & IP Summary

The open-source tools in this space (DataHub, OpenMetadata, Data Contract CLI, Soda Core) are all Apache 2.0 licensed with no known patent encumbrances. The Open Data Contract Standard (ODCS) v3.1.0 is governed by the Linux Foundation / Bitol organisation under Apache 2.0. OpenLineage is an LF AI & Data Graduate project under Apache 2.0. W3C DCAT is an open W3C Recommendation with no licensing restrictions. The commercial tools (Collibra, Atlan, Alation, Databricks Unity Catalog, Snowflake Horizon) are proprietary SaaS products; their APIs and connector specifications are publicly documented but their platform code is not available for reuse. No patent claims were found that would restrict building an open-source data mesh governance platform using public standards. Building on ODCS, OpenLineage, and DCAT carries no IP risk.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Domain-oriented data product registry: register, describe, and version data products per domain
- Automated data contract authoring and validation using ODCS v3.1.0 format
- Column-level lineage ingestion via OpenLineage-compatible events
- Policy enforcement for data contracts with consumer notification on breach
- Unified search across all registered data products (semantic + keyword)
- REST API with OpenAPI 3.1 documentation for platform integration

**Should-have (v1.1)**
- AI-assisted contract generation from schema and query log analysis
- SLO/SLA monitoring dashboard with historical quality metrics per data product
- Federated governance: multiple domain deployments syncing to a central graph
- Access request and approval workflow integrated with cloud IAM
- dbt, Airflow, and Spark integration via OpenLineage ingestion
- MCP server exposing catalog to AI coding assistants and LLM agents

**Nice-to-have (backlog)**
- Natural-language policy authoring (LLM-generated YAML/SQL enforcement rules)
- AI model and agent governance (input datasets, training lineage, deployment metadata)
- Cross-platform governance spanning Snowflake, Databricks, and open lake formats
- Data product marketplace with internal discovery and consumption tracking
- Auto-generated data product documentation from pipeline code and schema history
