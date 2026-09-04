# Authentication Strategy & Session Protocols (Authentication.md) — Full-Stack

## 1. Executive Overview
This document defines the authentication architecture, token lifecycle, and session caching strategy for the **SIH 02 Complaint Resolution System**. 

Identity management uses a hybrid model combining **Redis-backed session tokens** for citizen tracking and **JWT (JSON Web Tokens) with Argon2id password hashing** for departmental officers (`dep_01`, `dep_02`, `dep_03`). On the frontend client, session state is managed via **Zustand with persistence** and **Axios Interceptors**.

---

## 2. Authentication Context in System Architecture

Authentication operates directly between React Frontend, NGINX, Flask API, Redis, and User Re-verification modules as shown in the system architecture topology:

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
            Flask --> MainDB[(Main Master DB Ledger - main_db)]
            
            subgraph Isolated Department Database Schemas
                MainDB --> Dep1[(dep_01 Roads Schema)]
                MainDB --> Dep2[(dep_02 Water Schema)]
                MainDB --> Dep3[(dep_03 Electricity Schema)]
            end
            
            subgraph Unified Department Portal UI - Role-Based Rendering
                Dep1 --> Int1[dep_01 Kanban Interface]
                Dep2 --> Int2[dep_02 Ledger Table Interface]
                Dep3 --> Int3[dep_03 Outage Queue Interface]
            end
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

## 3. Authentication Flow Sequence & Frontend Interceptors

```mermaid
sequenceDiagram
    autonumber
    actor User as Citizen / Officer
    participant React as React Client (Zustand)
    participant NGINX as NGINX Load Balancer
    participant Flask as Flask API Auth Blueprint
    participant Redis as Redis Session Store
    participant DB as main_db (PostgreSQL)

    User->>React: Enter Credentials / Password
    React->>NGINX: POST /api/v1/auth/login
    NGINX->>Flask: Forward Request + Client IP
    Flask->>DB: Validate User / Officer Hash (Argon2id)
    DB-->>Flask: Account Verified
    Flask->>Redis: SET session:<token> (TTL=86400s, JSON User Data)
    Redis-->>Flask: OK
    Flask-->>React: Return Session Token + User Role
    React->>React: Save session to Zustand Persist Store

    Note over React, Redis: Subsequent Requests & Interceptors
    React->>NGINX: GET /api/v1/departments/dep_01/complaints (Bearer Token)
    NGINX->>Flask: Forward Request
    Flask->>Redis: GET session:<token>
    Redis-->>Flask: Active Session Data
    Flask-->>React: Return Complaints Data

    Note over React, Flask: Session Expiry (401) or Re-Verification (403)
    Flask-->>React: HTTP 403 (require_reverify=true)
    React->>React: Open Re-Auth Modal (Freeze Draft in Zustand)
    User->>React: Re-enter Password
    React->>Flask: POST /api/v1/auth/reverify
    Flask-->>React: Return short-lived X-Reverify-Token (TTL 300s)
    React->>Flask: Retry Initial Pending Request with X-Reverify-Token
```

---

## 4. Frontend Error Recovery Matrix (`AUTH_scenerioes.md` Integration)

| Scenario ID | Trigger Condition | Interceptor / Handling Mechanism | UI State & Action | Final Resolution |
| :--- | :--- | :--- | :--- | :--- |
| **Scenario 1** | Form Submission (Network OK) | Direct API Call (`POST`) | Displays **Step Progress Spinner** | `201 Created` -> Token saved to `LocalStorage` -> Redirects to `/submit/success` |
| **Scenario 2** | Session Expired | UI Interceptor (`401 Unauthorized`) | Freezes form state in `DraftStore` & opens **Re-Auth Modal** | Auto-retries pending API call upon successful authentication |
| **Scenario 3** | High Traffic | Rate Limiter (`429 Rate Limit`) | Displays *'System busy, retrying...'* Toast | Executes Exponential Backoff (1s, 2s, 4s) to retry request seamlessly |
| **Scenario 4** | Connection Loss | Network Status Listener | Switches to **Offline Indicator** & saves draft to `IndexedDB` | Prompts citizen *'Resume previous complaint?'* when back online |
| **Scenario 5** | PRR Vision Mismatch | Automated Confidence Gate (<70%) | Flags *'AI detected proof discrepancy'* on Officer UI | Requires **Officer Supervisor Manual Override Signature** to proceed |

---

## 5. Security & Token Lifecycle Protocols

| Parameter | Specification | Purpose |
| :--- | :--- | :--- |
| **Password Hashing** | `Argon2id` (memory_cost=65536, time_cost=3, parallelism=4) | Protection against GPU/ASIC password cracking. |
| **Session Token Format** | Cryptographically secure 256-bit random hex string | Opaque token immune to signature forgery. |
| **Redis Session TTL** | 24 Hours (86,400 seconds) | Automatic session invalidation. |
| **Re-verification TTL** | 5 Minutes (300 seconds) | Limits window for critical SRCS state changes. |
| **Rate Limiting** | 60 requests/min per IP via Redis Sliding Window | Prevents brute force and API flooding at NGINX & Flask layer. |
