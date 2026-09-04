# Project Development Roadmap & Phases (Phases.md) — Full-Stack

## 1. Executive Roadmap Overview

The execution roadmap implements the complete full-stack architecture across 6 structured phases:

```mermaid
flowchart TD
    subgraph Client App - React Vite TS Frontend
        UI[User Interface / React Components] --> ClientStore[Zustand Auth & Draft Store]
        UI --> ClientCrypto[Client Crypto Barrier - AES-256 & HMAC]
        ClientStore <-->|Offline Drafts| IndexedDB[(Browser IndexedDB)]
        UI --> Axios[Axios API Client & Interceptors]
    end

    Axios -->|HTTPS / REST API| NGINX[NGINX Load Balancer]
    
    subgraph Flask Backend Gateway
        NGINX -->|Reverse Proxy| Flask[Flask API Gateway]
        
        subgraph Authentication & Fast Memory
            Flask <-->|Sub-ms Auth Check| Redis[(Redis Cache & Session Store)]
            Flask --> Auth[User Session ID & Auth Re-verify]
        end

        subgraph Security & Ingestion Barriers
            Flask --> EncBarrier[Encryption Barrier - AES-256-GCM]
            EncBarrier --> Tracking[Tracking Complaint System]
            Tracking --> SaltBarrier[Encryption Salting Barrier - HMAC-SHA256]
        end

        subgraph AI Intelligence Engine
            Flask --> CtxAnalysis[Context Analysis]
            Flask --> ReEval[Re-evaluation of Complaint]
            Flask --> Priority[Priority Classification P1-P4]
            CtxAnalysis & ReEval & Priority <-->|JSON Prompt / Vision| Gemini[Gemini 2.5 Flash LLM]
        end

        subgraph Core Ledger & Departmental Routing
            Flask --> MainDB[(Main Database - main_db)]
            MainDB --> Dep1[(dep_01 Database)]
            MainDB --> Dep2[(dep_02 Database)]
            MainDB --> Dep3[(dep_03 Database)]
            Dep1 --> Int1[dep_01 Interface]
            Dep2 --> Int2[dep_02 Interface]
            Dep3 --> Int3[dep_03 Interface]
        end

        subgraph SRCS - Stage Resolve Commit System
            Int1 & Int2 & Int3 --> ResCheck{Is Problem Resolved?}
            ResCheck -- YES --> Resolved([Complaint Resolved & Closed])
            ResCheck -- NO --> SLA24[Resolve Period 24 hrs]
            SLA24 -- Over 24h --> SLA36[Staged Period 36 hrs] --> StateDB[(State Gov. DBMS - L1 Escalation)]
            SLA36 -- Over 36h --> SLA72[Staged Period 72 hrs] --> CentralDB[(Central Gov. DBMS - L2 Escalation)]
            
            SLA24 & StateDB & CentralDB --> PRR[Proof Checking & PRR Engine]
            PRR -- Validated --> Resolved
            PRR -. Status Re-sync .-> SaltBarrier
        end
    end
    
    PRR -. Vision Proof Audit Result .-> UI
```

---

## 2. Development Timeline Gantt Chart

```mermaid
gantt
    title SIH 02 Full-Stack Backend & Frontend Roadmap Execution
    dateFormat  YYYY-MM-DD
    section Phase 1: Setup & Infra
    Flask Factory & NGINX Scaffolding   :active, p1a, 2026-09-01, 3d
    Vite React-TS Setup & Radix UI      :p1b, 2026-09-01, 3d
    section Phase 2: Security & DB
    Postgres Schema & Encryption Barriers: p2a, after p1a, 4d
    Redis Session & Authentication API   : p2b, after p2a, 4d
    section Phase 3: AI Intelligence
    Gemini 2.5 Flash Context & Priority : p3, after p2b, 5d
    section Phase 4: Routing & Depts
    main_db Router & dep_01..03 DBs     : p4, after p3, 4d
    section Phase 5: SRCS & PRR Studio
    SLA 24h/36h/72h & PRR Studio UI     : p5, after p4, 5d
    section Phase 6: Audit & Prod
    OWASP Audit, Docker & Load Testing   : p6, after p5, 3d
```

---

## Phase 1: Environment Scaffolding, Vite & NGINX Setup
- Initialize Python 3.11 virtual environment & Flask 3.x Application Factory pattern.
- Scaffolding Vite React-TS project with Tailwind CSS, Zustand, and TanStack Query.
- Configure NGINX reverse proxy, SSL termination, and rate-limiting limits.

## Phase 2: Security Barriers & Authentication Engine
- Build **Encryption Barrier** (AES-256-GCM) and **Encryption Salting Barrier** (HMAC-SHA256) on client and server.
- Implement Redis Session Store middleware, Axios interceptors, and Argon2id password hashing.
- Build `/api/v1/auth/login`, `/api/v1/auth/verify`, and `/api/v1/auth/reverify` endpoints.

## Phase 3: Gemini 2.5 Flash AI Engine Integration
- Configure Google GenAI SDK with `gemini-2.5-flash` model.
- Implement structured JSON prompt engineering for **Context Analysis** and **Priority Classification** (`P1` to `P4`).

## Phase 4: Departmental Routing & Schema Isolation
- Create `main_db` master complaint ledger and isolated schemas (`dep_01_schema`, `dep_02_schema`, `dep_03_schema`).
- Implement Flask Blueprints and React UI pages for department interfaces (`dep_01` Roads, `dep_02` Water, `dep_03` Electricity).

## Phase 5: SRCS Engine & PRR Studio UI Integration
- Build Celery background worker for SLA monitoring (24h alert, 36h State DBMS sync, 72h Central DBMS sync).
- Implement PRR (Problem Resolve Re-evaluation) Vision proof matching with Gemini Flash multi-modal input and `PRRStudio.tsx` component.

## Phase 6: Security Audit, Dockerization & Production Verification
- Execute OWASP security vulnerability audit.
- Run load testing using Locust / Apache Bench (verifying NGINX + Redis concurrency).
- Finalize production Docker Compose and deployment manifests.
