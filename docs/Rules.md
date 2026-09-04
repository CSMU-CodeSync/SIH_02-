# System Rules & Guidelines (Rules.md) — Full-Stack

## 1. Core Architectural Principles
1. **Explicit Architecture Mapping**: Every developer edit MUST map to a component in the master architecture topology (React UI, NGINX, Redis, Encryption Barrier, Salting Barrier, Gemini LLM, `main_db`, `dep_01..03`, or SRCS).
2. **Defensive AI Prompting**: All Gemini 2.5 Flash prompts MUST enforce strict JSON schemas (`response_mime_type="application/json"`) with deterministic low temperature (`0.1`).
3. **Zero-Trust Data Protection**: All sensitive citizen grievance data MUST pass through the **Encryption Barrier** (AES-256-GCM) before DB persistence, and all lookup hashes MUST pass through the **Salting Barrier** (HMAC-SHA256).
4. **Mandatory Runtime Verification**: Never declare an API endpoint or UI feature complete without running automated test suites or CURL verification.

---

## 2. System Architecture Reference

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

## 3. Implementation Rules: What to Do vs. What to Avoid

### REQUIRED (Do This)
- **Flask Application Factory**: Use `create_app()` factory pattern with modular Flask Feature Blueprints (`app/features/*`).
- **SQLAlchemy 2.0 Syntax**: Use explicit `db.session.execute(select(...))` and `db.session.commit()`.
- **Redis Caching Decorator**: Wrap all citizen lookup routes with Redis cache check to guarantee sub-50ms response times.
- **Frontend Interceptor & Re-Auth Modal**: Capture `401`/`403` responses in Axios interceptors and launch Re-Auth Modal without losing user form draft.

---

### STRICTLY FORBIDDEN (Avoid This)
- **DO NOT** write raw SQL query strings with string concatenation (`f"SELECT * FROM users WHERE id='{user_id}'"`). Always use SQLAlchemy ORM parameterization.
- **DO NOT** execute blocking LLM or HTTP calls on the main Flask request loop. Use Celery background tasks for long operations.
- **DO NOT** store unencrypted plain text passwords or secrets in codebase / git repos. Use `.env` with environment variable validation.
- **DO NOT** allow officers to close complaints without passing through the **SRCS PRR Engine** proof checking in `PRRStudio.tsx`.
