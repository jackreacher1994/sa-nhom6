# Architecture Decisions

## 1. Document Purpose

This document records the principal architecture decisions for the **AI Document Audit & Knowledge Assistant System**. It explains the selected approach, the reasons for selecting it, and the trade-offs that should be revisited when requirements, scale, or platform constraints change.

The decisions are aligned with:

- [SRS.md](SRS.md)
- [FR-specification.md](FR-specification.md)
- [NFR-specification.md](NFR-specification.md)
- [cloud_architecture_full.html](cloud_architecture_full.html)

## 2. Decision Status

Unless stated otherwise, each decision is **accepted for the current release scope**. The document describes the target architecture; deployment sizing, service tiers, and cost assumptions still require validation during implementation and load testing.

## 3. Architecture Context

The system supports two primary roles:

- **Vendor / Contractor:** submits handover packages, adds document sources, views summaries, and queries the knowledge base.
- **Customer / Reviewer:** reviews audit findings, manages audit templates, makes final decisions, and identifies subject-matter experts.

The main workflow is:

1. Receive a document package and validate completeness.
2. Store the original package without modifying it.
3. Extract document text and structure.
4. Run template-driven AI audit checks.
5. Produce an auditable report with findings and evidence.
6. Allow a customer to accept or reject the package.
7. Index approved content for natural-language search and summaries.

The architecture must support Azure Entra ID, Azure Content Understanding, Azure OpenAI Service, enterprise content sources, and an HR / organizational directory. It must also meet the stated availability, performance, security, scalability, retention, and monthly cost constraints.

## 4. Slide-Ready Summary

**AD-001: Zone-Redundant Azure Deployment:** Use Azure as the primary cloud platform and deploy in Japan East across Availability Zones.  
Ensures availability when one Availability Zone fails.

**AD-002: Secure Web Entry Path:** Use Front Door, WAF, and private Application Gateway for system access.  
Protects the application from threats and reduces access-point failures.

**AD-004: Independent AKS Services:** Deploy Backend, AI, ETL, and Expert services separately on AKS.  
Allows each service to scale or recover independently.

**AD-005: Asynchronous ETL Pipeline:** Use queues, automatic retries, idempotent tasks, and a dead-letter queue.  
Prevents processing failures from causing data loss.

**AD-006: Resilient Data Storage:** Use redundant Blob Storage, PostgreSQL backups, and point-in-time recovery.  
Protects documents and audit records during system failures.

**AD-009: Private Service Connectivity:** Keep application workloads in a protected private network and access documents, databases, AI services, and container images through private endpoints instead of the public Internet.

**AD-010: Monitoring and Recovery:** Use Azure Monitor, health checks, logs, alerts, and audit tracking.  
Detects failures quickly and supports faster service restoration.

**AD-011: Automated CI/CD and Blue-Green Deployment:** Use Azure DevOps, ACR, Helm, and blue-green deployment.  
Enables safe releases, quick rollback, and minimal downtime.

## 5. Security, Observability, and Scaling

### Security - NFR-4

- **AD-003: Entra ID and RBAC:** Use Entra ID with OAuth2/OIDC sign-in. Short-lived JWTs are validated in every FastAPI dependency, and RBAC enforces Vendor and Customer scopes.
- **AD-002: WAF-protected ingress:** Apply one WAF policy across Front Door and Application Gateway, including OWASP managed rules, bot protection, rate limiting, and platform DDoS protection.
- **AD-009: Private connectivity and secure access:** Keep workloads in `app-vnet` and access document storage, databases, AI services, and container images through private endpoints instead of public network access. Use Key Vault, managed identities, and TLS for secure service communication.
- **AD-008: Protected document data:** Use TLS for data in transit, encrypt sensitive user fields at rest, and keep original documents immutable in Blob Storage under lifecycle and retention policies.
- **AD-011: Secure software supply chain:** Use ACR Premium for signed and vulnerability-scanned images, then deploy blue/green Helm releases through Azure DevOps workload identity without long-lived secrets.

### Observability

- **AD-010: Centralized monitoring:** Use Azure Monitor, Application Insights for traces and metrics, and Container Insights for AKS pod and node logs.
- **AD-010: End-to-end tracing:** Correlate SPA, Backend, AI, ETL, Expert, and every Celery task using the submission ID.
- **AD-010: Operational dashboards:** Track queue depth per ETL stage, audit duration, AI Search latency, OpenAI token consumption, and service availability.
- **AD-010: SLO and cost alerts:** Alert when p95 query latency exceeds 3 seconds, an audit exceeds 10 minutes, the dead-letter queue is non-empty, or availability falls below 99.9%. Trigger budget alerts at 80%, 90%, and 100% of the USD 5,000 monthly cap.
- **AD-010: Auditable history:** Store every submission, AI finding, and reviewer accept/reject decision in immutable PostgreSQL audit history with timestamps.

### Scaling and Resilience - NFR-2 / NFR-3

- **AD-001: Zone redundancy:** Run AKS across Availability Zones 1, 2, and 3 in Japan East, with PostgreSQL zone-redundant HA, Redis Premium, Blob ZRS, and multi-replica AI Search.
- **AD-004: Independent scaling:** Scale Backend, AI, ETL, and Expert workloads independently using AKS replicas, CPU, and Redis queue depth behind an internal load balancer.
- **AD-005: Staged ETL:** Separate `extract -> chunk -> embed -> index` queues so a burst in one stage does not block interactive queries.
- **AD-006: Retrieval and cache scaling:** Scale AI Search with replicas for query throughput and partitions for index size. Cache embeddings and frequent query results in Redis.
- **AD-005: Recoverable background work:** Use idempotent Celery tasks, automatic retries, and a dead-letter queue so failed work is replayable and never silently lost.
- **AD-007: Fast interactive path:** Stream responses over SSE and precompute document summaries during ETL while audits run asynchronously.

### Acceptance Targets

- First token under 2 seconds at p95
- 95% of audit runs under 10 minutes per package
- End-to-end audit completion under 15 minutes at p99
- 500 document submissions per hour
- 50 concurrent interactive queries
- p95 query latency under 3 seconds during bursts
- 99.9% monthly availability
- RTO under 30 minutes and RPO under 15 minutes
- Zero silent data loss

## 6. Decision Summary

| ID | Decision | Status |
| --- | --- | --- |
| AD-001 | Use Azure as the primary cloud platform and deploy in Japan East. | Accepted |
| AD-002 | Use Azure Front Door, Static Web Apps, WAF, and private Application Gateway for the web entry path. | Accepted |
| AD-003 | Use Azure Entra ID with OAuth2/OIDC and application roles for authentication and authorization. | Accepted |
| AD-004 | Use AKS to host separately deployable Backend, AI, ETL, and Expert services. | Accepted |
| AD-005 | Use asynchronous, staged ETL with Celery and Redis for document ingestion. | Accepted |
| AD-006 | Separate operational data, original documents, search indexes, and transient queues by storage technology. | Accepted |
| AD-007 | Use hybrid retrieval-augmented generation with Azure AI Search and Azure OpenAI Service. | Accepted |
| AD-008 | Use Azure Content Understanding for document extraction and preserve immutable originals. | Accepted |
| AD-009 | Protect internal services with private networking, managed identities, and Key Vault. | Accepted |
| AD-010 | Make processing idempotent, retryable, observable, and auditable. | Accepted |
| AD-011 | Use Azure DevOps, ACR, Helm, and blue/green deployment practices for delivery. | Accepted |
| AD-012 | Keep human review as the final authority for audit acceptance or rejection. | Accepted |

## 7. Detailed Decisions

### AD-001: Azure-first deployment

**Decision:** Host the system on Azure, with the primary deployment region in Japan East and zone redundancy where the selected service tier supports it.

**Rationale:** The requirements mandate Azure Entra ID, Azure Content Understanding, and Azure OpenAI Service. Keeping the application and managed dependencies on the same cloud reduces integration and identity complexity, while the selected region and availability zones support the availability and data-governance goals.

**Trade-offs:** This creates strong dependence on Azure service availability, pricing, quotas, and regional feature support. Portability to another cloud is reduced, but introducing cross-cloud abstractions would add cost and delivery risk for the current scope.

**Consequences:** Service tiers, regional quotas, private endpoint support, and monthly cost must be checked before production deployment.

### AD-002: Layered web entry path with private origin

**Decision:** Use Azure DNS and Azure Front Door at the edge, Static Web Apps for the React SPA, WAF policies for request filtering, and an internal Application Gateway as the private API origin for AKS.

**Rationale:** This separates static delivery from API ingress, provides a single public edge, centralizes TLS and WAF controls, and keeps the application origin out of direct public reach. Front Door routes `/` to the SPA and `/api/*` through Private Link to the internal ingress path.

**Trade-offs:** The path has more components than a directly exposed application endpoint and therefore increases configuration and operational complexity. It provides stronger isolation, routing, and protection that justify the added complexity for this system.

**Consequences:** Health probes, routing rules, certificates, WAF policies, and private connectivity must be tested together.

### AD-003: Entra ID and role-based access control

**Decision:** Authenticate users through Azure Entra ID using OAuth2/OIDC. Use short-lived JWTs and application roles such as `Vendor` and `Customer`, with authorization enforced by the backend.

**Rationale:** Entra ID is a stated constraint and provides centralized enterprise identity, lifecycle management, and role assignment. Backend validation ensures that access control is not dependent only on frontend behavior.

**Trade-offs:** The system depends on tenant configuration and user provisioning. Role changes and token expiry must be handled correctly, and service-to-service identities need a separate managed-identity approach.

**Consequences:** Every protected API must validate issuer, audience, signature, expiry, and required role claims. Authorization decisions should be recorded for sensitive actions.

### AD-004: AKS with separated application services

**Decision:** Run four logical services on Azure Kubernetes Service: Backend, AI, ETL, and Expert. Implement the services in Python using FastAPI, with Celery workers for background processing.

**Rationale:** The services have different responsibilities and scaling profiles. Separating them allows interactive API traffic, AI orchestration, ingestion workers, and expert matching to evolve and scale independently. AKS also provides a consistent deployment boundary for the Python services.

**Trade-offs:** Kubernetes introduces cluster, networking, image, deployment, and observability overhead. A simpler app-hosting service could be easier to operate at low scale, but would provide less control over independent workloads and staged workers.

**Consequences:** The initial cluster should remain operationally small. Resource requests, limits, health probes, autoscaling rules, and ingress policies are required before production use.

### AD-005: Asynchronous staged document ETL

**Decision:** Process ingestion through separate stages: `extract -> chunk -> embed -> index`. Use Redis-backed Celery queues, idempotent tasks, automatic retries, and a dead-letter queue.

**Rationale:** Document ingestion and AI processing are longer-running and bursty than interactive requests. A staged pipeline allows each step to scale independently, supports the target submission throughput, and prevents a slow document from blocking user-facing traffic.

**Trade-offs:** Queue-based processing introduces eventual consistency and requires status tracking, retry handling, and operational dashboards. It is more complex than a synchronous request, but is necessary for resilience and burst handling.

**Consequences:** Each submission needs a durable processing state and correlation identifier. Retries must not duplicate documents, audit results, embeddings, or notifications.

### AD-006: Polyglot data responsibilities

**Decision:** Use each data service for a distinct responsibility:

- **Azure Blob Storage:** immutable uploaded originals and crawled source files.
- **Azure Database for PostgreSQL:** users, submissions, audit reports, templates, feedback, learning data, and governance logs.
- **Azure AI Search:** the unified hybrid keyword and vector index.
- **Azure Cache for Redis:** Celery broker, staged queues, embeddings cache, and frequent-query cache.

**Rationale:** The data types have different access patterns and durability needs. Keeping originals, transactional records, retrieval indexes, and transient processing state separate improves fit, recovery options, and lifecycle management.

**Trade-offs:** Multiple services increase deployment and monitoring effort. A single database would be simpler but would be a poor fit for large binary files, vector retrieval, and transient queue state.

**Consequences:** The system must define source-of-truth ownership clearly. Search indexes and caches are rebuildable projections; PostgreSQL records and Blob originals require durable backup and retention policies.

### AD-007: Hybrid RAG for search and audit assistance

**Decision:** Use Azure AI Search for hybrid keyword/vector retrieval and Azure OpenAI Service for intent routing, reranking support, summaries, natural-language answers, and audit generation. Responses should include source references.

**Rationale:** Keyword search preserves exact terms and identifiers, while vector retrieval improves semantic matching across heterogeneous documents. Retrieval-augmented generation grounds AI responses in indexed organizational content and provides evidence for user review.

**Trade-offs:** AI output quality depends on chunking, metadata, index freshness, prompt design, model availability, and token cost. RAG reduces unsupported answers but does not remove the need for citations, confidence handling, and human verification.

**Consequences:** Retrieval metadata must retain document identity, version, location, and relevant section references. Prompts and model configuration should be versioned alongside audit templates.

### AD-008: Managed document understanding with immutable originals

**Decision:** Use Azure Content Understanding to extract text, layout, and structure. Preserve the submitted and crawled originals in Blob Storage and never modify source documents during ingestion or audit generation.

**Rationale:** Managed extraction reduces the need to maintain format-specific parsers and supports the document intelligence requirement. Immutable originals preserve evidence, enable reprocessing with improved extraction logic, and support audit traceability.

**Trade-offs:** Extraction quality varies by file type and document quality, and the managed service adds usage cost and dependency on supported formats. The pipeline must reject unsupported or oversized inputs clearly.

**Consequences:** Derived content must be traceable to an original object and version. Retention, deletion, and access policies must apply consistently to originals and derived artifacts.

### AD-009: Private service connectivity and centralized secret management

**Decision:** Place backend services and data services in a virtual network. Connect supported Azure services through private endpoints, use Azure Key Vault for secrets and certificates, and use managed identities for workload access. Do not store credentials in source code or container images.

**Rationale:** The requirements prohibit public internal service exposure and require secure handling of secrets. Private networking limits attack surface, while managed identities and Key Vault reduce long-lived credential leakage.

**Trade-offs:** Private endpoints, DNS, routing, firewall rules, and deployment identities add setup and troubleshooting effort. They also require a clear operational model for development and production connectivity.

**Consequences:** Network policies and access paths must be tested from each AKS workload. Key Vault access should be least-privilege and auditable, with rotation procedures defined for source credentials and certificates.

### AD-010: Durable, observable, and auditable processing

**Decision:** Make submission, audit, review, and ingestion operations traceable through correlation IDs, durable status records, audit history, retries, and Azure Monitor telemetry.

**Rationale:** The system must avoid silent data loss, support recovery objectives, and retain a trustworthy record of AI findings and customer decisions. Observability is also needed to manage queue depth, latency, failed tasks, AI usage, and cost alerts.

**Trade-offs:** Logging and audit retention increase storage and governance work. Excessive payload logging could expose sensitive document content, so telemetry must use structured, minimal, access-controlled data.

**Consequences:** Logs should capture identifiers, states, timings, outcomes, and error categories without unnecessarily copying sensitive content. Alerts should cover failed processing, queue growth, latency, availability, budget thresholds, and security events.

### AD-011: Repeatable CI/CD with signed images

**Decision:** Use Azure DevOps for source control and pipelines, Azure Container Registry for versioned and scanned images, Helm for AKS deployments, and blue/green release practices. Use workload identity rather than long-lived pipeline secrets.

**Rationale:** The service-oriented deployment needs repeatable builds, automated tests, controlled rollout, and rollback. Image signing and scanning reduce supply-chain risk, while blue/green deployment limits disruption to the interactive path.

**Trade-offs:** The delivery pipeline requires more infrastructure and release discipline than manual deployment. Blue/green releases can temporarily consume additional compute and require database migration compatibility.

**Consequences:** Builds must produce immutable versioned artifacts. Deployment gates should include tests, image scanning, configuration validation, health checks, and rollback readiness.

### AD-012: Human-in-the-loop final decision

**Decision:** Treat AI audit output as decision support. A Customer / Reviewer remains responsible for the final accept or reject decision and its memo or required-fix list.

**Rationale:** The business flow explicitly assigns final review authority to the customer. Human review is important when evidence is incomplete, extraction is ambiguous, or an AI result needs contextual judgment.

**Trade-offs:** The workflow retains a human approval step and cannot be fully automated. This adds review time, but it reduces the risk of an unverified AI result becoming an authoritative business decision.

**Consequences:** Reports must distinguish AI-generated findings from customer decisions, preserve supporting evidence, and maintain a complete decision history with timestamps and actor identity.

## 8. Quality Attribute Mapping

| Quality attribute | Architectural response |
| --- | --- |
| Performance | Static edge delivery, cached embeddings and queries, precomputed summaries, hybrid retrieval, and streamed responses. |
| Scalability | Independent AKS workloads, queue-depth autoscaling, staged ETL, and scalable AI Search partitions/replicas. |
| Availability | Zone-redundant managed services, storage redundancy, health probes, retries, and recoverable background tasks. |
| Recovery | PostgreSQL point-in-time recovery, durable Blob originals, rebuildable search indexes, and dead-letter processing. |
| Security | Entra ID, RBAC, TLS, WAF, private endpoints, managed identities, Key Vault, and least-privilege access. |
| Auditability | Immutable originals, evidence references, correlation IDs, durable audit logs, and human decision records. |
| Cost control | Managed services sized for the release scope, caching, budget alerts at 80%, 90%, and 100%, and explicit load testing. |

## 9. Risks and Revisit Triggers

These decisions should be revisited when one of the following occurs:

- The monthly cloud cost approaches or exceeds the USD 5,000 cap.
- Actual load materially exceeds 500 submissions per hour or 50 concurrent queries.
- AI Search or model quality is insufficient for required audit accuracy.
- A required Azure service is unavailable in Japan East or lacks a needed private-network capability.
- Retention, residency, or enterprise integration requirements change.
- The team cannot operate AKS reliably within the delivery schedule and support capacity.
- Automated audit recommendations begin to be used as final decisions without the required customer review.

## 10. Open Implementation Questions

1. Which exact Azure service tiers and quotas satisfy the NFR targets within the monthly budget?
2. What file formats, maximum package sizes, and retention periods will be configured for the first production release?
3. Which protocols and read-only integration patterns are available for each enterprise content source and the HR / organizational directory?
4. What evaluation dataset and acceptance thresholds will be used to measure extraction, retrieval, citation, and audit accuracy?
5. Which audit-template and prompt versions must be retained with each generated report?
6. What production support ownership and recovery runbooks are required for AKS, queues, AI dependencies, and private networking?
