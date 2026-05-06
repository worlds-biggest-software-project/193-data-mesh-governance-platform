# Standards & API Reference

> Project: Data Mesh Governance Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO/IEC 38505 — Data Governance of IT**
- URL: https://digital.nemko.com/standards/iso-iec-38505
- The primary ISO standard for data governance, providing six governing principles: responsibility, strategy, data acquisition, performance, conformance, and human behaviour. Directly applicable as the framework for domain ownership, policy definition, and compliance monitoring in a data mesh governance platform. Widely used by financial institutions and regulated enterprises to evidence GDPR and risk-management compliance.

**ISO/IEC 27001 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The foundational ISMS standard. Relevant for governing access to sensitive data assets across domain boundaries in a multi-tenant data mesh governance deployment. Required by many enterprise procurement processes.

**ISO/IEC 27701:2025 — Privacy Information Management System (PIMS)**
- URL: https://www.iso.org/standard/27701
- Published July 2025. Extends ISO 27001 with privacy-specific controls for both data controllers and processors, aligned with GDPR obligations. A data mesh governance platform handling PII lineage and access policies should be designed to support 27701 audit evidence.

**ISO/IEC 25012 — Data Quality Model**
- Defines characteristics of data quality (accuracy, completeness, consistency, timeliness, etc.). Provides the conceptual vocabulary for data product SLO/SLA definitions and quality rule categories within data contracts.

---

### W3C & IETF Standards

**DCAT — Data Catalog Vocabulary (W3C Recommendation, Version 3)**
- URL: https://www.w3.org/TR/vocab-dcat-3/
- The W3C standard for describing datasets and data services in a machine-readable way. DCAT 3 (current) extends the model to cover dataset series, versioning, and services in addition to datasets. Directly applicable as the metadata schema for data product catalog entries. Widely adopted by government data portals; DCAT-US 3.0 profile required for US federal agencies (aligned with March 2025 Executive Order). Enables cross-catalog interoperability between domain registries.

**DCAT-US 3.0 Profile**
- URL: https://doi-do.github.io/dcat-us/
- US federal government profile of DCAT 3. Relevant for deployments serving US government data domains. Aligns with the March 2025 EO on eliminating information silos.

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Foundation for REST API design. A DCAT-based data product registry API should conform to HTTP semantics for resource-oriented design.

**RFC 8288 — Web Linking (rel= links)**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Hypermedia linking standard. Enables HATEOAS-style navigation between related data products, lineage nodes, and governance policies in a REST API.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard authorization framework for securing API access to governance resources. Applicable to domain team authentication, cross-domain data product access requests, and policy enforcement gateways.

**RFC 9110 — HTTP Semantics (2022)**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- Updated comprehensive HTTP semantics standard, superseding RFC 7231. Should be the reference for REST API implementation.

---

### Data Model & API Specifications

**Open Data Contract Standard (ODCS) v3.1.0**
- URL: https://bitol-io.github.io/open-data-contract-standard/v3.1.0/
- GitHub: https://github.com/bitol-io/open-data-contract-standard
- The emerging industry standard for data contracts, governed by the Linux Foundation / Bitol organisation (formed November 2023 from AIDA User Group + LF AI & Data). ODCS v3.1.0 is a YAML-based format that defines data product infrastructure, schemas, quality rules, SLA agreements, team ownership, and access control in a single document. Supports complex structures including JSON, Avro, and Protobuf models. The previously competing "Data Contract Specification" was deprecated with the v3.1.0 release; ODCS is now the single convergence point. Supported until end of 2026 for legacy spec migration.
- The Data Contract CLI implements this standard: http://cli.datacontract.com/

**OpenLineage Specification**
- URL: https://openlineage.io/docs/
- GitHub: https://github.com/OpenLineage/OpenLineage
- Spec: https://github.com/OpenLineage/OpenLineage/blob/main/spec/OpenLineage.md
- An LF AI & Data Foundation Graduate project defining an open standard for lineage metadata collection across data pipelines. Formalised as JSON Schema (OpenLineage.json) with an OpenAPI spec for HTTP-based implementations. Defines a generic model of Dataset, Job, and Run entities with extensible Facets for user-defined metadata enrichment. Supported by 15+ platforms including Apache Airflow, Apache Spark, Flink, and dbt. Marquez is the reference implementation. Column-level lineage market reached ~$873M in 2025, growing at >15% CAGR — strong incentive to adopt this standard.

**OpenMetadata Standard**
- URL: https://openmetadatastandards.org/
- Community-driven open schema specification for metadata entities: tables, pipelines, dashboards, ML models, data products, and data contracts. Provides the type system and API contract for the OpenMetadata platform; can be adopted independently as a cross-tool metadata exchange format.

**OpenAPI Specification 3.1 (OAS 3.1)**
- URL: https://spec.openapis.org/oas/v3.1.0
- The standard for documenting REST APIs. A data mesh governance platform should publish an OAS 3.1-compliant API definition for the data product registry, contract management, and governance policy endpoints. Required by Collibra, OpenMetadata, and Atlan for their own API documentation.

**JSON Schema (Draft 2020-12)**
- URL: https://json-schema.org/specification
- Used by OpenLineage facets, ODCS schema definitions, and OpenMetadata entity schemas for validation. Relevant for data contract schema validation in the governance platform.

**Delta Sharing Protocol (Apache 2.0)**
- URL: https://delta.io/sharing/
- An open protocol for secure, cross-platform data sharing developed by Databricks. Governs how data products are shared across organisational boundaries. Relevant as a sharing transport layer for published data products in a cross-domain governance platform.

**Apache Iceberg REST Catalog API**
- URL: https://iceberg.apache.org/docs/latest/rest-catalog/
- The open REST API specification for Iceberg table catalogs. Increasingly relevant as Databricks Unity Catalog and Snowflake Horizon federate via Iceberg REST. A governance platform should be capable of federating metadata from Iceberg REST Catalogs.

**FAIR Data Principles**
- URL: https://www.go-fair.org/fair-principles/
- Findable, Accessible, Interoperable, Reusable — the widely cited evaluation framework for data product quality. Not a formal standard but a de facto benchmark used by research institutions and enterprise data governance programmes to assess data product readiness.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) and OpenID Connect 1.0**
- URL (OIDC): https://openid.net/connect/
- Standard framework for delegated authorisation and identity federation. Essential for cross-domain access request workflows where a consumer in Domain B requests access to a data product owned by Domain A, and approvals flow through SSO-integrated identity providers.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- The reference for securing governance platform REST APIs that expose sensitive metadata (PII classification, policy definitions, access policies). Key items: Broken Object Level Authorization, Security Misconfiguration, Improper Inventory Management.

**NIST SP 800-53 (Rev. 5) — Security and Privacy Controls**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- The US federal standard for security controls. Applicable for data mesh deployments in US government or regulated financial/healthcare sectors. Informs access control, audit logging, and incident response design.

**GDPR (Regulation EU 2016/679)**
- URL: https://gdpr-info.eu/
- The EU data protection regulation. A governance platform must support GDPR compliance obligations: data subject rights, PII lineage tracing, retention policy enforcement, and audit evidence. Data contract SLAs should include GDPR data classification fields.

---

### MCP Server Specifications

**Model Context Protocol (MCP) — Anthropic / Open Standard**
- URL: https://modelcontextprotocol.io/
- Spec (November 2025): https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/
- 2026 Roadmap: https://blog.modelcontextprotocol.io/posts/2026-mcp-roadmap/
- MCP is an open protocol that allows LLMs and AI agents to interact with external data sources and tools. Directly applicable: a data mesh governance platform should expose an MCP server so that AI coding assistants and LLM agents can query the data product catalog, check data contracts, and retrieve lineage context without leaving their development environment. Select Star's MCP server (2025) is an early example. Assured Consulting Solutions launched the first MCP server for Collibra in August 2025. A paper published January 2026 (arxiv.org/html/2601.08687v1) describes using MCP for data product access in a mesh architecture, translating natural-language requests to SQL queries governed by data contracts.

---

## Similar Products — Developer Documentation & APIs

### DataHub (Acryl / LinkedIn)

- **Description:** Open-source metadata platform (Apache 2.0) with 12,000+ GitHub stars. Provides data discovery, governance, and lineage for 80+ data sources. Acryl Cloud adds managed enterprise features.
- **API Documentation:** https://docs.datahub.com/docs/api/datahub-apis
- **GraphQL API:** https://docs.datahub.com/docs/api/graphql/overview (primary public API; interactive GraphiQL at /api/graphiql)
- **SDKs/Libraries:**
  - Python: `pip install acryl-datahub` — https://docs.datahub.com/docs/metadata-ingestion
  - Java SDK: https://docs.datahub.com/docs/datahub-graphql-core
- **Developer Guide:** https://datahubproject.io/docs/api/datahub-apis/
- **Standards:** GraphQL; internal REST.li persistence layer; Kafka for real-time streaming ingestion
- **Authentication:** Token-based (Bearer); OIDC/SSO supported in Acryl managed tier

---

### Collibra

- **Description:** Enterprise data governance platform (commercial SaaS). Authoritative source for data classification metadata with cloud IAM sync; AI governance for ML models and agents.
- **API Documentation:** https://developer.collibra.com/api
- **Developer Portal:** https://developer.collibra.com/
- **SDKs/Libraries:**
  - REST client generation from OpenAPI 3.0 spec (language agnostic)
  - Java API reference: https://developer.collibra.com/integrations
  - Import API examples: https://github.com/collibra/import-api-examples
- **Developer Guide:** https://developer.collibra.com/developer-tutorials/getting-started-with-the-rest-api/
- **Standards:** OpenAPI Specification 3.0; Knowledge Graph API (GraphQL, launched 2025)
- **Authentication:** REST API tokens; OAuth 2.0 / SSO for enterprise deployments

---

### Atlan

- **Description:** AI-native active metadata platform (commercial SaaS). Enterprise Data Graph connecting 100+ sources; Gartner Magic Quadrant Leader 2025. Production in 4–6 weeks.
- **API Documentation:** https://developer.atlan.com/
- **Endpoints Reference:** https://developer.atlan.com/endpoints/
- **SDKs/Libraries:**
  - Java SDK: https://github.com/atlanhq/atlan-java
  - Python SDK: https://docs.atlan.com/get-started/how-tos/getting-started-with-the-apis
  - Raw REST API: https://developer.atlan.com/sdks/raw/
- **Developer Guide:** https://developer.atlan.com/getting-started/
- **Standards:** REST/JSON; GraphQL (https://docs.atlan.com/tags/graphql); OpenAPI
- **Authentication:** API Key; OAuth 2.0 / SSO for enterprise

---

### OpenMetadata

- **Description:** Open-source (Apache 2.0) unified metadata platform covering discovery, lineage, quality, and native data contract support. 80+ connectors. REST API documented via Swagger/OpenAPI.
- **API Documentation:** https://docs.open-metadata.org/v1.12.x/api-reference
- **Developer Portal:** https://docs.open-metadata.org
- **SDKs/Libraries:**
  - Python SDK: https://pypi.org/project/openmetadata-ingestion/
  - Java SDK: available in source repository
  - Go SDK: available in source repository
- **Developer Guide:** https://docs.open-metadata.org
- **Standards:** OpenAPI 3.0 (Swagger-generated); REST/JSON; base URL pattern: `https://{host}/api/v1`
- **Authentication:** JWT token; OIDC/SSO supported

---

### Databricks Unity Catalog

- **Description:** Unified governance layer for the Databricks Lakehouse (commercial, bundled). Covers tables, files, ML models, volumes; ABAC; Iceberg federation; Delta Sharing.
- **API Documentation:** https://docs.databricks.com/api/workspace/introduction
- **Catalogs API:** https://docs.databricks.com/api/workspace/catalogs
- **SDKs/Libraries:**
  - Python SDK: `pip install databricks-sdk` — https://databricks-sdk-py.readthedocs.io
  - Java SDK: https://github.com/databricks/databricks-sdk-java
  - Go SDK: https://github.com/databricks/databricks-sdk-go
- **Developer Guide:** https://docs.databricks.com/aws/en/data-governance/unity-catalog/
- **Standards:** REST/JSON; Delta Sharing protocol (Apache 2.0); Iceberg REST Catalog API
- **Authentication:** Databricks Personal Access Token; OAuth 2.0 Machine-to-Machine (M2M)

---

### Snowflake Horizon Catalog

- **Description:** Built-in governance and data product catalog for Snowflake (commercial, bundled). SQL-native policies; internal marketplace; Iceberg REST federation; Copilot AI assistance (2025).
- **API Documentation:** https://docs.snowflake.com/en/user-guide/snowflake-horizon
- **Developer Guide:** https://www.snowflake.com/en/developers/guides/getting-started-with-horizon-for-data-governance-in-snowflake/
- **SDKs/Libraries:**
  - Python Connector: `pip install snowflake-connector-python`
  - Snowpark (Python, Java, Scala): https://docs.snowflake.com/en/developer-guide/snowpark/
  - JDBC/ODBC drivers available
- **Standards:** SQL; REST API; Delta Sharing (Apache 2.0); Iceberg REST Catalog API
- **Authentication:** Username/password; Key Pair; OAuth 2.0; SAML SSO

---

### Alation

- **Description:** Commercial data catalog (SaaS) with ML-driven behavioural curation, data stewardship workflows, and the Alation Agentic Platform (AI-driven governance, data products marketplace — 2025/2026).
- **API Documentation:** https://developer.alation.com/ (developer portal)
- **SDKs/Libraries:**
  - Python SDK available via developer portal
  - REST API with OpenAPI documentation
- **Standards:** REST/JSON; OpenAPI
- **Authentication:** API Token; OAuth 2.0 / SSO

---

### Select Star

- **Description:** Automated data catalog (commercial SaaS) populated from query log analysis; column-level lineage derived from real usage; MCP server for AI agent integration (2025).
- **API Documentation:** https://www.selectstar.com/product/data-catalog
- **MCP Server:** https://www.selectstar.com/resources/data-mcp-for-ai-agents
- **Standards:** REST/JSON; MCP (Model Context Protocol)
- **Authentication:** API Key

---

### Data Contract CLI

- **Description:** Open-source (Apache 2.0) command-line tool for working with data contracts in ODCS v3.1.0 format. Schema linting, quality test execution, CI/CD integration.
- **API Documentation / Spec:** https://cli.datacontract.com/
- **GitHub:** https://github.com/datacontract/datacontract-cli
- **SDKs/Libraries:** Python package (`pip install datacontract-cli`)
- **Developer Guide:** https://datacontract.com/
- **Standards:** Open Data Contract Standard (ODCS) v3.1.0; JSON Schema; OpenAPI
- **Authentication:** N/A (CLI tool; connects to data sources via environment credentials)

---

## Notes

**Convergence on ODCS:** The data contract ecosystem converged on ODCS v3.1.0 (Linux Foundation / Bitol) during 2024–2025. The competing Data Contract Specification is deprecated and will only receive maintenance until end of 2026. New tooling should adopt ODCS v3.1.0 from the outset.

**OpenLineage maturity:** OpenLineage has reached graduate status at LF AI & Data and is supported by 15+ major platforms. The column-level lineage market is growing at >15% CAGR. OpenLineage is the safe choice for lineage event emission. The absence of a standard lineage storage and query layer (Marquez is the reference implementation but is not widely deployed at scale) represents a tooling gap.

**MCP as data mesh infrastructure:** The January 2026 arxiv paper on "Data Product MCP" demonstrates MCP as a governance enforcement layer — LLMs submit data access requests through an MCP server that enforces data contracts and purpose-bound access. This pattern is emerging as a practical way to implement federated computational governance, one of the four core data mesh principles.

**DCAT 3 adoption lag:** While DCAT 3 was finalised by W3C, production catalog implementations have been slow to migrate from DCAT 2. The DCAT-US 3.0 mandate for US federal agencies is expected to accelerate adoption in 2026. An AI-native platform building on DCAT 3 from the start would have an interoperability advantage with government and research data domains.

**Iceberg as governance substrate:** Both Databricks Unity Catalog and Snowflake Horizon now federate via the Apache Iceberg REST Catalog API. This positions Iceberg as the cross-platform data access layer, and governance metadata stored in a standards-based catalog above the Iceberg REST layer can achieve genuine platform neutrality.
