# Effort Estimation — AI Document Audit & Knowledge Assistant System

**Estimate basis:** Man-days (MD). Assumes a team of 5 (1 PM, 2 backend, 1 frontend, 1 QA).  
**Constraint:** Go-live within 3 calendar months (~66 working days per person).

---

## 1. Project Setup & Cloud Infrastructure

| Task | MD |
|------|:--:|
| Architecture & tech stack finalization | 2 |
| Azure resource provisioning (Entra ID, Blob, AI Search, OpenAI, PostgreSQL, Redis, VNet) | 3 |
| CI/CD pipeline (GitHub Actions / Azure DevOps) | 2 |
| Monitoring & logging setup (App Insights, alerts) | 1 |
| **Subtotal** | **8** |

## 2. Authentication & Authorization (FR-1)

| Task | MD |
|------|:--:|
| Azure Entra ID app registration & OAuth2/OIDC configuration | 2 |
| Backend JWT validation & FastAPI auth middleware | 2 |
| RBAC (Vendor / Customer role enforcement) | 2 |
| Login/logout UI (React) | 2 |
| **Subtotal** | **8** |

## 3. Frontend Application (React SPA)

| Task | MD |
|------|:--:|
| Project scaffolding, routing, shared components | 3 |
| Document submission & checklist UI | 4 |
| Audit report viewing & evidence browsing UI | 4 |
| Accept/reject decision workflow UI | 3 |
| Natural language search / Q&A interface | 5 |
| Audit template management UI | 3 |
| Document source addition UI | 2 |
| Expert identification UI | 2 |
| Responsive design & cross-browser polish (Chrome/Edge 130+) | 2 |
| **Subtotal** | **28** |

## 4. Backend API (Python / FastAPI)

| Task | MD |
|------|:--:|
| Project scaffolding, DB models (PostgreSQL), migrations | 3 |
| Document upload & submission API (incl. Blob Storage) | 3 |
| Audit template CRUD API | 2 |
| Audit report & review decision API | 3 |
| Search query API (NL → AI Search → OpenAI) | 4 |
| Expert identification API | 2 |
| Document source connector API (SharePoint, Confluence, file servers, DB) | 4 |
| Interaction logging & feedback API | 2 |
| **Subtotal** | **23** |

## 5. Document Processing Pipeline (FR-3, FR-4)

| Task | MD |
|------|:--:|
| Azure Content Understanding integration for text/structure extraction | 3 |
| ETL pipeline: extract → chunk → embed → index (Celery + Redis) | 5 |
| AI audit generation engine (Azure OpenAI prompt chains + template engine) | 5 |
| Audit report compilation & storage | 2 |
| Queue-depth autoscaling & staged pipelines (NFR-2) | 2 |
| **Subtotal** | **17** |

## 6. AI Search & Knowledge Base (FR-7)

| Task | MD |
|------|:--:|
| Azure AI Search index schema design | 2 |
| Embedding pipeline integration (text-embedding-ada or similar) | 2 |
| NL query interpretation & intent mapping (Azure OpenAI) | 3 |
| Relevance ranking, filtering, hybrid search | 3 |
| Summary generation with source references | 2 |
| Caching layer (Redis for embeddings & frequent queries) | 1 |
| **Subtotal** | **13** |

## 7. Expert Identification (FR-8)

| Task | MD |
|------|:--:|
| HR / Org Directory integration | 2 |
| Keyword extraction from document content | 1 |
| Matching & ranking algorithm | 2 |
| **Subtotal** | **5** |

## 8. Learning from Interactions (FR-9)

| Task | MD |
|------|:--:|
| Query & feedback capture pipeline | 2 |
| Interaction history storage | 1 |
| Relevance model adjustment based on feedback signals | 2 |
| **Subtotal** | **5** |

## 9. Non-Functional Requirements

| Task | MD |
|------|:--:|
| Performance: streaming (SSE), caching, two-tier retrieval (NFR-1) | 3 |
| Scalability: horizontal autoscaling, staged pipelines (NFR-2) | 3 |
| Availability: idempotent tasks, dead-letter queue, health probes, zone redundancy (NFR-3) | 3 |
| Security: managed identities, Key Vault, VNet + private endpoints, TLS (NFR-4) | 4 |
| **Subtotal** | **13** |

## 10. Testing

| Task | MD |
|------|:--:|
| Unit tests (backend: pytest, frontend: Jest / React Testing Library) | 4 |
| Integration tests (API + DB + Azure service mocks) | 3 |
| End-to-end tests (Cypress or Playwright) | 3 |
| Performance & load tests (locust / k6) | 2 |
| User acceptance testing support | 2 |
| **Subtotal** | **14** |

## 11. Deployment, Documentation & Training

| Task | MD |
|------|:--:|
| Infrastructure as Code (Bicep / Terraform) | 3 |
| Deployment runbook & release management | 2 |
| User manual & system documentation | 2 |
| Stakeholder training & handover | 2 |
| **Subtotal** | **9** |

---

## Summary

| Phase | Man-Days |
|-------|:--------:|
| 1. Project Setup & Cloud Infrastructure | 8 |
| 2. Authentication & Authorization | 8 |
| 3. Frontend Application | 28 |
| 4. Backend API | 23 |
| 5. Document Processing Pipeline | 17 |
| 6. AI Search & Knowledge Base | 13 |
| 7. Expert Identification | 5 |
| 8. Learning from Interactions | 5 |
| 9. Non-Functional Requirements | 13 |
| 10. Testing | 14 |
| 11. Deployment, Documentation & Training | 9 |
| **Total** | **143** |

## Team Loading & Timeline

| Metric | Value |
|--------|-------|
| Total effort | 143 man-days |
| Team size | 5 (1 PM, 2 BE, 1 FE, 1 QA) |
| Available days (3 months) | ~66 days/person = 330 man-days |
| Utilization | 143 / 330 = **43%** |
| Feasibility | **Achievable** with buffer for unknowns |

### Recommended Parallel Tracks

```
Month 1     │ Setup + Auth + Frontend scaffolding + Backend core
Month 1-2   │ Pipeline + AI Audit + Search + Frontend features
Month 2-3   │ Expert ID + Learning + NFR hardening + Testing + Deployment
```

### Risk Buffers

- Azure service integration delays (quota limits, API changes): +10 MD
- Requirements clarification / rework: +10 MD
- Learning curve on AI/ML components: +8 MD
- **Recommended contingency:** **+28 MD** (~20% of total)
- **Total with contingency:** **171 MD** (52% of available team capacity)

**Conclusion:** The estimate of **143–171 man-days** (with contingency) is well within the **3-month / 5-person** constraint, leaving ample room for iteration, refinement, and unplanned work.
