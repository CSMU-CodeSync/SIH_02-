# Product Requirement Document (PRD.md) — SIH 02 Complaint Resolution System

## 1. Executive Summary & Vision
The **SIH 02 Complaint Resolution System** is an enterprise-grade, AI-driven public grievance and complaint resolution platform built using Python (**Flask**). The system automates grievance intake, performs intelligent context and priority analysis using **Gemini 2.5 Flash LLM**, isolates departmental workflows (`dep_01`, `dep_02`, `dep_03`), and enforces strict resolution timelines through a novel **SRCS (Stage Resolve Commit System)**.

By leveraging **NGINX Load Balancing**, **Redis In-Memory Caching**, dual-layer **Encryption & Salting Barriers**, and multi-tier government database escalation (**State Gov. DBMS** & **Central Gov. DBMS**), the platform guarantees zero downtime, sub-millisecond session authentication, and transparent SLA enforcement.

---

## 2. Target Audience & Core Use Cases
- **Citizens (Complainants)**: Submit complaints, receive instant AI-classified priority, and track status in real-time via cached lookups without system delay.
- **Departmental Officers (`dep_01`, `dep_02`, `dep_03`)**: View routed complaints, submit resolution proofs, and manage department-level complaint tables.
- **System Administrators & Auditors**: Monitor SLA breach timelines (24h/36h/72h), oversee automatic escalation to State and Central DBMS, and run PRR (Problem Resolve Re-evaluation) verifications.

---

## 3. Core System Architecture & Backend Modules

```mermaid
flowchart TD
 Client[Citizen / API Client] --> NGINX[NGINX Load Balancer]
 NGINX --> FlaskAPI[Flask API Gateway]
 
 subgraph Security & Ingestion
 FlaskAPI --> Auth[User Session & Auth Re-verify]
 Auth <--> Redis[Redis Cache & Session Store]
 FlaskAPI --> EncBarrier[Encryption Barrier]
 EncBarrier --> Tracking[Tracking Complaint System]
 Tracking --> SaltBarrier[Encryption Salting Barrier]
 end

 subgraph AI Intelligence Engine
 FlaskAPI --> CtxAnalysis[Context Analysis]
 FlaskAPI --> ReEval[Re-evaluation of Complaint]
 FlaskAPI --> Priority[Priority Classification]
 CtxAnalysis & ReEval & Priority <--> Gemini[Gemini 2.5 Flash LLM]
 end

 subgraph Routing & Departmental DBs
 FlaskAPI --> MainDB[(Main Database / main_db)]
 MainDB --> Dep1[(dep_01 DB)] & Dep2[(dep_02 DB)] & Dep3[(dep_03 DB)]
 Dep1 --> Int1[dep_01 Interface]
 Dep2 --> Int2[dep_02 Interface]
 Dep3 --> Int3[dep_03 Interface]
 end

 subgraph SRCS (Stage Resolve Commit System)
 Int1 & Int2 & Int3 --> ResCheck{Is Problem Resolved?}
 ResCheck -- YES --> Resolved[Complaint Resolved & Closed]
 ResCheck -- NO --> SLA24[Resolve Period 24 hrs]
 SLA24 -- Over 24h --> SLA36[Staged Period 36 hrs] --> StateDB[(State Gov. DBMS - L1)]
 SLA36 -- Over 36h --> SLA72[Staged Period 72 hrs] --> CentralDB[(Central Gov. DBMS - L2)]
 
 SLA24 & StateDB & CentralDB --> PRR[PRR Engine & Proof Checking]
 PRR -- Validated --> Resolved
 PRR -. Re-sync .-> SaltBarrier
 end
```

---

## 4. Key Feature Specifications & API Endpoints

### 4.1 Ingestion & Authentication API
- **`POST /api/v1/auth/verify`**: Validates citizen session token via **Redis Session Store**.
- **`POST /api/v1/complaints/submit`**: Encrypts payload via **Encryption Barrier** (AES-256-GCM), assigns tracking hash, and pushes to pipeline.

### 4.2 AI Context & Priority Engine (`/api/v1/ai/...`)
- **`POST /api/v1/ai/analyze-context`**: Calls **Gemini 2.5 Flash LLM** to extract semantic keywords, category, and urgency.
- **`POST /api/v1/ai/classify-priority`**: Automatically assigns priority (`P1-CRITICAL`, `P2-HIGH`, `P3-MEDIUM`, `P4-LOW`) based on Gemini analysis.

### 4.3 Routing & Departmental Interfaces
- **`GET /api/v1/departments/<dep_id>/complaints`**: Fetches department-specific complaints from isolated `dep_01`, `dep_02`, or `dep_03` SQL tables.
- **`PUT /api/v1/departments/<dep_id>/resolve`**: Officer submits resolution details + image/pdf proof.

### 4.4 SRCS (Stage Resolve Commit System) Engine
- **`POST /api/v1/srcs/check-sla`**: Celery background task evaluating 24h, 36h, and 72h SLA boundaries.
- **`POST /api/v1/srcs/escalate`**: Triggers state/central DBMS persistence upon SLA breach.
- **`POST /api/v1/srcs/prr-verify`**: Runs **PRR (Problem Resolve Re-evaluation)** using Gemini LLM vision/text context match against submitted resolution proof.

---

## 5. Non-Functional Requirements (NFRs)
- **Performance**: Response time < 50ms for cached status queries via Redis; < 1.2s for Gemini LLM context classification.
- **Scalability**: NGINX load balancer distributing requests across Gunicorn Flask workers.
- **Data Security**: AES-256 encryption at rest/in transit; salting barrier for hash verification.
- **Availability**: 99.9% uptime guaranteed via decoupled worker architecture.
