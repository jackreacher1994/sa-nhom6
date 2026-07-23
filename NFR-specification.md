# AI Document Audit System - Non-Functional Requirements

Each NFR is stated, motivated for this system, and paired with one or two concrete solution options mapped to the current stack (React, FastAPI, Celery/Redis, PostgreSQL, Azure Blob, Azure AI Search, Azure OpenAI, Azure Content Understanding, Entra ID).

### NFR-1 Performance / Latency

- **Requirement.** Interactive NL queries should return a first meaningful response quickly (target: first token/summary under ~2-3 s p95), and audit runs should complete within an agreed SLA (target: minutes, not hours, per package).
- **Why it matters.** RAG answers that call an LLM plus vector search feel slow if returned as one blocking response; vendors and reviewers work interactively.
- **Solution options.**
  - **A. Streaming + caching.** Stream tokens from Azure OpenAI to the SPA over Server-Sent Events so users see progressive output; cache embeddings and frequent query results in Redis to skip repeat work.
  - **B. Two-tier retrieval + precomputation.** Use cheap vector top-k retrieval followed by a reranking pass, and precompute document summaries during ETL so the read path serves cached summaries instead of generating on demand.

### NFR-2 Scalability / Throughput

- **Requirement.** The system must absorb bursty submission and query load (e.g., end-of-phase document dumps) without degrading interactive latency.
- **Why it matters.** Handover submissions arrive in bursts; heavy ETL/audit work must not block the interactive read path.
- **Solution options.**
  - **A. Queue-depth autoscaling.** Scale Celery workers horizontally based on Redis queue depth (e.g., KEDA on Azure Container Apps/AKS), and scale AI Search replicas/partitions for query throughput.
  - **B. Staged pipelines.** Split ETL into separate queues (extract, chunk, embed, index) so slow stages don't block fast ones and each stage scales independently.

### NFR-3 Availability / Reliability

- **Requirement.** Target 99.9% availability for the interactive path; no submitted document or audit result is ever silently lost.
- **Why it matters.** A lost submission or dropped audit undermines trust in a compliance-oriented system.
- **Solution options.**
  - **A. Managed, redundant services.** Use zone-redundant managed Azure services (PostgreSQL Flexible Server HA, Blob geo-redundant storage, multi-replica AI Search) plus liveness/readiness probes on each container.
  - **B. Idempotent, retryable tasks.** Make Celery tasks idempotent with automatic retries and a dead-letter queue, so a transient failure in extraction or audit is retried rather than dropped.

### NFR-4 Security

- **Requirement.** All data in transit and at rest is encrypted; secrets are never stored in code; internal service-to-service traffic is not publicly reachable.
- **Why it matters.** The system handles confidential contractual and factory documents.
- **Solution options.**
  - **A. Identity + secrets hygiene.** Authenticate via Entra ID (OAuth2/OIDC) with short-lived JWTs validated in FastAPI dependencies; keep all secrets/keys in Azure Key Vault; enforce TLS everywhere.
  - **B. Private networking.** Place backend, stores, and Azure services behind a VNet with private endpoints so Blob, AI Search, and OpenAI have no public exposure.

### NFR-5 Authorization / RBAC and Data Isolation

- **Requirement.** Users can only access documents, audits, and answers they are authorized to see, based on role, department, and permissions.
- **Why it matters.** Vendors must not see other vendors' packages, and RAG answers must never surface unauthorized content.
- **Solution options.**
  - **A. Policy checks from claims.** Map role/department claims from Entra ID onto authorization checks implemented as FastAPI dependencies guarding every endpoint.
  - **B. Search-side security trimming.** Store ACL/security-group metadata on each chunk in Azure AI Search and filter by the caller's identity at query time, so retrieval itself never returns forbidden documents.

### NFR-6 Privacy / Compliance and Data Governance

- **Requirement.** The system complies with data-governance policies: PII is controlled, access is auditable, and retention/deletion obligations are enforced.
- **Why it matters.** [requirement.md](requirement.md) explicitly calls for security and compliance with data-governance policies.
- **Solution options.**
  - **A. Redaction + access audit.** Detect and redact PII during the ETL step, and write an immutable audit log of every document access and audit action.
  - **B. Retention + deletion propagation.** Apply data-residency, retention, and customer-managed-key encryption policies, and propagate "right to delete" requests to Blob storage and the AI Search index together.

### NFR-7 AI Quality: Accuracy, Grounding, and Explainability

- **Requirement.** Audit findings and answers are grounded in source content, cite their evidence, and expose confidence; ungrounded output is flagged, not presented as fact.
- **Why it matters.** Reviewers make accept/reject decisions based on AI output; hallucinated findings are unacceptable in an audit context.
- **Solution options.**
  - **A. Mandatory citations.** Require every audit finding and answer to link to the specific source chunks used (as the search activity diagram already implies), and flag or withhold answers that cannot be grounded.
  - **B. Structured, human-in-the-loop audits.** Drive audits from templates that force structured JSON output with per-finding confidence scores, and keep the reviewer sign-off step (already in the reviewer-decision flow) as the authoritative gate.

### NFR-8 Observability

- **Requirement.** Operators can trace any request end-to-end and inspect why a given audit or answer was produced, including cost and latency.
- **Why it matters.** A distributed RAG pipeline across four containers is hard to debug and cost-control without tracing.
- **Solution options.**
  - **A. Distributed tracing.** Instrument Frontend, Backend, ETL, and AI Service with OpenTelemetry exporting to Azure Monitor / Application Insights, with correlation IDs propagated across the Redis queue.
  - **B. LLM-specific tracing.** For each audit/query run, record the prompt, retrieved chunks, token counts, model, latency, and cost to enable debugging, quality review, and budget alerts.

### NFR-9 Maintainability / Evolvability

- **Requirement.** Components can evolve (including swapping AI models) without ripple effects, and audits remain reproducible over time.
- **Why it matters.** The system must grow from the audit use case toward the broader knowledge-assistant vision; models and prompts will change.
- **Solution options.**
  - **A. Contract-first coupling.** Keep the four containers loosely coupled behind explicit REST (OpenAPI schema) and queue contracts so the AI model or provider can be swapped behind the AI Service boundary.
  - **B. Versioned prompts/templates.** Store prompt and audit-template versions in PostgreSQL and stamp each audit result with the versions used, so any past audit can be reproduced and explained.

### NFR-10 Usability / Accessibility

- **Requirement.** The interface is clear, responsive, and accessible (WCAG 2.1 AA), and gracefully handles empty or low-confidence results.
- **Why it matters.** Users are domain experts, not AI specialists; the UI must build trust and be broadly usable.
- **Solution options.**
  - **A. Trust-building result UI.** Stream results progressively and always show sources and confidence alongside answers and findings.
  - **B. Accessible, graceful degradation.** Meet WCAG 2.1 AA, and implement the "no results -> suggest related topics" behavior (already in the search activity diagram) so dead ends still guide the user.

### NFR-11 Cost Efficiency

- **Requirement.** LLM and infrastructure costs scale sub-linearly with usage and stay within budget.
- **Why it matters.** Per-token LLM costs and always-on services can grow quickly with document volume.
- **Solution options.**
  - **A. Tiered models + caching.** Use a cheaper model for extraction/classification and reserve the premium model for final audit generation; cache embeddings and reuse them across runs.
  - **B. Batch + budget guards.** Run ETL in off-peak batches and enforce per-request token budgets with alerting when thresholds are exceeded.

### NFR-12 Data Freshness / Consistency (RAG)

- **Requirement.** Answers reflect the current state of source content; stale index entries do not produce outdated audits or answers.
- **Why it matters.** For the broader multi-source vision, source documents change frequently and answers must not lag reality.
- **Solution options.**
  - **A. Event-driven re-indexing.** Trigger ETL re-indexing from source-change events (e.g., SharePoint/Confluence webhooks) so the index updates when content changes.
  - **B. Incremental crawl + versioning.** Run scheduled incremental crawls with change detection and index versioning to avoid stale answers and allow rollback.

## 6. Assumptions and Open Questions

- **Assumption.** The AI Document Audit System is the first concrete realization of the broader Enterprise Knowledge Assistant vision; the same containers are expected to grow to support multi-source ingestion and cross-source query.
- **Assumption.** Deployment targets Azure (implied by Blob, AI Search, OpenAI, Content Understanding, Entra ID); container hosting (AKS vs Container Apps) is not yet fixed.
- **Open question.** Concrete SLA targets (audit turnaround time, query p95 latency, availability %) are not stated in the source material and are proposed here as defaults to be confirmed.
- **Open question.** The scope and mechanism of "Learn from user interactions" (FR-18) - explicit feedback, implicit signals, or fine-tuning - is undefined.
- **Open question.** Notification channel (email, in-app, webhook) is referenced in activity diagrams but not specified.
- **Open question.** Multi-tenancy boundaries between vendors (separate indexes vs security-trimmed shared index) need a decision; NFR-5 offers both options.

## 7. Traceability (NFR to containers / use cases)

- **NFR-1 Performance** - Frontend, AI Service, Search Index, Redis; FR-4, FR-6, FR-7.
- **NFR-2 Scalability** - ETL Service, AI Service, Redis, Search Index; FR-3, FR-4, FR-6.
- **NFR-3 Availability** - all data stores, Backend, ETL; FR-1, FR-2, FR-5, FR-11.
- **NFR-4 Security** - Backend, Entra ID, all data stores; FR-13.
- **NFR-5 Authorization / isolation** - Backend, Search Index, Entra ID; FR-14, FR-6, FR-8.
- **NFR-6 Privacy / compliance** - ETL Service, Database, Blob, Search Index; FR-3, FR-11, FR-16.
- **NFR-7 AI quality** - AI Service, Azure OpenAI, Search Index; FR-4, FR-7, FR-10.
- **NFR-8 Observability** - all containers, Redis; FR-4, FR-6.
- **NFR-9 Maintainability** - Backend, AI Service, Database; FR-4, FR-15.
- **NFR-10 Usability** - Frontend; FR-2, FR-6, FR-7, FR-8.
- **NFR-11 Cost efficiency** - AI Service, ETL Service, Azure OpenAI; FR-3, FR-4.
- **NFR-12 Data freshness** - ETL Service, Search Index; FR-3, FR-9, FR-16.
