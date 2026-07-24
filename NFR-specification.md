# AI Document Audit System - Non-Functional Requirements

### NFR-1 Performance / Latency

- **Requirement.** Interactive NL queries should return a first meaningful response quickly (target: first token/summary under ~2-3 s p95), and audit runs should complete within an agreed SLA (target: minutes, not hours, per package).
- **Solution options.**
  - **A. Streaming + caching.** Stream tokens from Azure OpenAI to the SPA over Server-Sent Events so users see progressive output; cache embeddings and frequent query results in Redis to skip repeat work.
  - **B. Two-tier retrieval + precomputation.** Use cheap vector top-k retrieval followed by a reranking pass, and precompute document summaries during ETL so the read path serves cached summaries instead of generating on demand.

### NFR-2 Scalability / Throughput

- **Requirement.** The system must absorb bursty submission and query load (e.g., end-of-phase document dumps) without degrading interactive latency.
- **Solution options.**
  - **A. Queue-depth autoscaling.** Scale Celery workers horizontally based on Redis queue depth (e.g., KEDA on Azure Container Apps/AKS), and scale AI Search replicas/partitions for query throughput.
  - **B. Staged pipelines.** Split ETL into separate queues (extract, chunk, embed, index) so slow stages don't block fast ones and each stage scales independently.

### NFR-3 Availability / Reliability

- **Requirement.** Target 99.9% availability for the interactive path; no submitted document or audit result is ever silently lost.
- **Solution options.**
  - **A. Managed, redundant services.** Use zone-redundant managed Azure services (PostgreSQL Flexible Server HA, Blob geo-redundant storage, multi-replica AI Search) plus liveness/readiness probes on each container.
  - **B. Idempotent, retryable tasks.** Make Celery tasks idempotent with automatic retries and a dead-letter queue, so a transient failure in extraction or audit is retried rather than dropped.

### NFR-4 Security

- **Requirement.** All data in transit and at rest is encrypted; secrets are never stored in code; internal service-to-service traffic is not publicly reachable.
- **Solution options.**
  - **A. Identity + secrets hygiene.** Authenticate via Entra ID (OAuth2/OIDC) with short-lived JWTs validated in FastAPI dependencies; keep all secrets/keys in Azure Key Vault; enforce TLS everywhere.
  - **B. Private networking.** Place backend, stores, and Azure services behind a VNet with private endpoints so Blob, AI Search, and OpenAI have no public exposure.
