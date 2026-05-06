# Data Mesh Governance Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An open, AI-native platform for domain-oriented data ownership, contract enforcement, and data product cataloging across federated teams.

The Data Mesh Governance Platform gives data platform engineers, domain teams, and Chief Data Officers a single place to publish, govern, and consume data products across a federated mesh. It targets organisations that have outgrown monolithic catalogs but cannot justify the six-figure price tags or multi-month deployments of incumbent enterprise governance suites.

---

## Why Data Mesh Governance Platform?

- **Enterprise governance is priced out of reach.** Collibra deployments commonly exceed $170K/yr base and $500K/yr at scale; Informatica's IDMC suite reaches $250K–$500K+/yr — pricing that excludes mid-market teams entirely.
- **Open-source incumbents push enterprise features behind a paywall.** DataHub's core is Apache 2.0, but column-level RBAC, data contracts, and policy enforcement require the Acryl managed tier.
- **Native lake/warehouse catalogs lock buyers in.** Databricks Unity Catalog and Snowflake Horizon only govern their own platforms, leaving cross-platform lineage and contract enforcement unsolved.
- **No tool covers the full data contract lifecycle.** Authoring, validation, versioning, SLO monitoring, and consumer breach notification are spread across 4–6 separate tools today.
- **Domain self-service is still missing.** Even Atlan and Alation require platform admin involvement to configure domain-level governance, undermining the core data mesh promise of autonomous domains.

---

## Key Features

### Domain-Oriented Data Product Registry

- Register, describe, and version data products per domain
- Domain ownership model with maintainers, documentation, and per-asset governance rules
- Federated metadata services so domain teams can run their own service syncing to a central graph
- Unified search across all registered data products (semantic and keyword)

### Data Contracts and Quality

- Native data contract authoring and validation using Open Data Contract Standard (ODCS) v3.1.0
- Schema, quality rules, and SLA definitions in a single contract entity
- Policy enforcement for data contracts with consumer notification on breach
- SLO/SLA monitoring dashboard with historical quality metrics per data product

### Lineage and Discovery

- Column-level lineage ingestion via OpenLineage-compatible events
- Query-log-driven lineage that reflects real usage rather than declared metadata
- PII tagging and sensitivity classification
- Business glossary and data dictionary management

### Policy, Access, and Compliance

- Role-based access control with policy enforcement
- Access request and approval workflow integrated with cloud IAM
- Audit trail for compliance reporting (GDPR, HIPAA, SOX)
- Federated governance across multiple domain deployments

### Integrations and Extensibility

- REST API with OpenAPI 3.1 documentation for platform integration
- dbt, Airflow, and Spark integration via OpenLineage ingestion
- MCP server exposing catalog metadata to AI coding assistants and LLM agents
- Webhook-based event system for triggering downstream actions

---

## AI-Native Advantage

AI is used to close the gaps left by both legacy and modern catalogs. The platform infers likely schema, SLOs, and ownership from query logs and pipeline metadata to generate data contracts without manual documentation. LLM-powered discovery answers natural-language questions across the catalog, and natural-language policy authoring turns plain-English rules into enforceable YAML or SQL. Proactive contract breach detection attributes quality drops to upstream changes, and intelligent domain boundary suggestion analyses query co-access patterns and team structure to recommend optimal data product decomposition.

---

## Tech Stack & Deployment

The platform is designed around open standards: ODCS v3.1.0 for data contracts (Linux Foundation / Bitol), OpenLineage for lineage events, W3C DCAT for catalog interoperability, and the OpenMetadata schema specification for entity modelling. Deployment supports self-hosted operation for domain-owned metadata services with a central graph for federation. Integration is API-first via REST (OpenAPI 3.1) and GraphQL, with SDKs and an MCP server for AI agent access.

---

## Market Context

The data governance and catalog market is estimated at $3–5 billion in 2025, sitting inside a broader data management market exceeding $100 billion. Incumbent pricing ranges from $50K–$500K+/yr (Collibra, Atlan, Alation, Informatica), with Snowflake Horizon and Databricks Unity Catalog bundled into platform spend. Primary buyers are Chief Data Officers and data governance teams at enterprises with 200+ engineers, data platform engineers building domain-oriented architectures, compliance officers needing lineage audit trails, and data product owners in domain teams.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
