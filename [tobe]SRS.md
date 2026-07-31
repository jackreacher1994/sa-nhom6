# Software Requirements Specification
## AI Document Audit & Knowledge Assistant System

---

## 1. Introduction

### 1.1 Purpose
This document defines the software requirements for the **AI Document Audit & Knowledge Assistant System** (TO-BE). The system automates document handover and audit review between vendors and customers using AI-assisted processing, natural language querying, and intelligent knowledge management.

### 1.2 Scope
The system enables vendors to submit handover documents from multiple enterprise sources, runs AI-driven audit checks, allows customers to review and make final decisions, provides natural language search across the knowledge base, and identifies subject matter experts — all secured via role-based access control.

---

## 2. Actors

| Actor | Description |
|-------|-------------|
| **Vendor / Contractor** | Submits handover documents, adds document sources, views summaries, and checks audit results. |
| **Customer / Reviewer** | Reviews audit results, manages audit templates, makes final review decisions, and identifies subject matter experts. |
| **System** | AI Document Audit & Knowledge Assistant System — processes documents, generates audits, indexes content, and facilitates knowledge queries. |

---

## 3. Business Flows

### 3.1 Flow 1: Submission to Audit Notification

**Actors involved:** Vendor, System, Customer

**Description:** The end-to-end process of document submission, automated completeness validation, AI-powered processing and auditing, and notification of results.

**Steps:**

| Step | Actor | Action | Description |
|------|-------|--------|-------------|
| 1 | Vendor | Prepare handover document package | Vendor gathers and organizes all handover documents required for audit. |
| 2 | Vendor | Fill in submission checklist | Vendor completes a structured checklist describing what is being submitted. |
| 3 | Vendor | Submit package to system | Vendor uploads the document package via the system frontend. |
| 4 | System | Receive and log submission | System accepts the uploaded package, logs the submission record, and stores documents in Azure Blob Storage. |
| 5 | System | Check completeness automatically | System validates the submission against the checklist and required document types to verify completeness. |
| 6 | System | Completeness decision | **If incomplete:** System notifies vendor of specific missing items. Vendor revises and resubmits (returns to step 3). **If complete:** Proceed to step 7. |
| 7 | System | Start document processing | System initiates the document processing pipeline. |
| 8 | System | Extract content and run audit checks | System extracts text and structure via Azure Content Understanding, then runs AI-based audit checks using the configured audit template via Azure OpenAI Service. |
| 9 | System | Generate and complete audit report | System compiles audit findings into a structured report with pass/fail assessments and detailed comments. |
| 10 | System | Send notification after audit complete | System notifies both vendor and customer that the audit report is ready for review. |
| 11 | Customer | Receive audit notification and review result | Customer accesses the audit report through the system to review findings. |

---

### 3.2 Flow 2: Reviewer Decision Process

**Actors involved:** Customer, System, Vendor

**Description:** The customer reviews AI-generated audit results and makes a final acceptance or rejection decision, with corresponding system actions and vendor follow-up.

**Steps:**

| Step | Actor | Action | Description |
|------|-------|--------|-------------|
| 1 | Customer | Receive audit report notification | Customer is notified that an audit report is ready (continuation from Flow 1). |
| 2 | Customer | Open and read audit findings | Customer opens the audit report and examines the AI-generated pass/fail results and comments. |
| 3 | Customer | Study supporting evidence | Customer reviews the referenced document sections and supporting evidence attached to each audit finding. |
| 4 | Customer | Decision | Customer makes a decision on the audit package: **Accept** or **Reject**. |
| 5a | Customer | **Accept:** Mark package as approved | Customer formally marks the handover package as approved in the system. |
| 6a | Customer | **Accept:** Write acceptance memo | Customer documents the acceptance rationale in an acceptance memo. |
| 7a | System | Record decision in audit log | System logs the accept decision with timestamp and memo into the audit history. |
| 8a | System | Update status and notify vendor | System updates the submission status to "Approved" and notifies the vendor. |
| 5b | Customer | **Reject:** List required fixes | Customer specifies the deficiencies and required corrective actions. |
| 6b | Vendor | Revise and resubmit | Vendor addresses the listed issues, revises the documents, and resubmits the package (loops back to Flow 1, step 3). |
| 7b | System | Record decision in audit log | System logs the reject decision with required fixes into the audit history. |
| 8b | System | Update status and notify vendor | System updates the submission status to "Rejected — Changes Required" and notifies the vendor with the fix list. |

---

### 3.3 Flow 3: Search, Query, and Summary

**Actors involved:** Customer, Vendor, System

**Description:** Users can query the knowledge base using natural language, receive AI-generated answers with source references, and refine queries interactively.

**Steps:**

| Step | Actor | Action | Description |
|------|-------|--------|-------------|
| 1 | Customer | Enter natural language query | Customer types a question or search phrase in natural language via the system interface. |
| 2 | System | Receive and interpret query | System processes the natural language input and determines search intent using Azure OpenAI Service. |
| 3 | System | Search knowledge base for relevant content | System queries the indexed knowledge base (Azure AI Search) to retrieve relevant document chunks. |
| 4 | System | Rank and filter results by relevance | System applies relevance ranking and filters to return the most pertinent content. |
| 5 | System | Results found decision | **If results found:** Generate summary and key points (step 6). **If no results:** Suggest related topics (step 7). |
| 6 | System | Generate summary and key points | System synthesizes an answer with key points and source links using Azure OpenAI Service. |
| 7 | System | No results: suggest related topics | System identifies and presents related topics or alternative search terms based on query analysis. |
| 8 | Customer | View answer and source links | Customer reads the generated answer and can navigate to source documents for deeper context. |
| 9 | Vendor | View answer and take action | Vendor (if authorized) can also view the answer and take necessary follow-up actions. |
| 10 | User | Satisfied decision | **If satisfied:** Flow ends. **If not satisfied:** User refines the query and returns to step 1 for an improved search. |

---

### 3.4 Flow 4: Authentication and Access Control

**Actors involved:** Vendor, Customer, System

**Description:** All users authenticate via Azure Entra ID, and the system enforces role-based access control to restrict functions and data per user role.

**Steps:**

| Step | Actor | Action | Description |
|------|-------|--------|-------------|
| 1 | User | Initiate login | User accesses the system and is redirected to the Azure Entra ID login page. |
| 2 | System | Authenticate via Azure Entra ID | System validates user credentials through Azure Entra ID (OAuth2/OIDC) and issues a short-lived JWT. |
| 3 | System | Enforce role-based access | System checks the user's assigned role (Vendor or Customer) and grants access only to permitted functions (e.g., Vendor can submit documents; Customer can manage templates and make final decisions). |
| 4 | User | Access authorized functions | User interacts with the system within the scope of their role permissions. |

---

### 3.5 Flow 5: Document Source Addition

**Actors involved:** Vendor, System

**Description:** Vendors can add document sources from enterprise content repositories to be ingested and indexed by the system.

**Steps:**

| Step | Actor | Action | Description |
|------|-------|--------|-------------|
| 1 | Vendor | Select source type | Vendor chooses the enterprise source type: SharePoint, Confluence, file server, or internal database. |
| 2 | Vendor | Provide source credentials/connection | Vendor supplies connection details and access credentials for the selected source. |
| 3 | System | Validate and connect to source | System validates the connection and establishes access to the content source. |
| 4 | System | Ingest documents from source | System extracts documents from the connected source and processes them through the ETL pipeline (extract, chunk, embed, index). |
| 5 | System | Confirm source added | System confirms successful ingestion and makes the documents available for audit and search. |

---

### 3.6 Flow 6: Audit Template Management

**Actors involved:** Customer, System

**Description:** Customers create and manage audit templates that define the criteria and structure used by the AI when generating audit results.

**Steps:**

| Step | Actor | Action | Description |
|------|-------|--------|-------------|
| 1 | Customer | Navigate to template management | Customer opens the audit template management section. |
| 2 | Customer | Create or select a template | Customer creates a new template or selects an existing one to edit. |
| 3 | Customer | Define audit criteria | Customer sets the audit checklist items, pass/fail thresholds, and evaluation rules. |
| 4 | System | Save template | System persists the template configuration in the database. |
| 5 | Customer | Activate template | Customer selects and activates the template for use in upcoming audit runs. |
| 6 | System | Use template for audits | System references the active template when generating AI audit results (Flow 1, step 8). |

---

### 3.7 Flow 7: Subject Matter Expert Identification

**Actors involved:** Customer, System

**Description:** Customers can identify subject matter experts relevant to a specific document or audit issue using organizational data.

**Steps:**

| Step | Actor | Action | Description |
|------|-------|--------|-------------|
| 1 | Customer | Select document or audit topic | Customer picks a document or audit finding that requires expert consultation. |
| 2 | System | Analyze content and extract keywords | System analyzes the document content to extract domain keywords and expertise requirements. |
| 3 | System | Query HR / Org Directory | System searches the organizational directory for employees whose profiles match the required expertise. |
| 4 | System | Rank and present expert candidates | System returns a ranked list of potential subject matter experts with their relevant credentials. |
| 5 | Customer | Review and select expert | Customer reviews the suggestions and selects appropriate experts for consultation. |

---

### 3.8 Flow 8: Learning from User Interactions

**Actors involved:** System

**Description:** The system captures user interactions from the document query process and uses that history to improve future search and response relevance.

**Steps:**

| Step | Actor | Action | Description |
|------|-------|--------|-------------|
| 1 | System | Capture user query and feedback | System logs each natural language query, the results returned, and any explicit or implicit user feedback (e.g., satisfaction signals, follow-up queries). |
| 2 | System | Store interaction history | Interaction data is stored and associated with the relevant document context. |
| 3 | System | Analyze and improve relevance models | System periodically retrains or adjusts relevance ranking and response generation models based on interaction patterns and user corrections. |
| 4 | System | Apply improvements to future queries | The refined models are applied to enhance the quality of future search results and AI-generated answers. |
