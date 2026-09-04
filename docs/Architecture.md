# System Architecture & Tech Stack (Architecture.md) — Flask Backend

## 1. Technology Stack & Component Overview

| Layer | Technology | Specification / Purpose |
| :--- | :--- | :--- |
| **Reverse Proxy & Load Balancer** | NGINX 1.24+ | SSL Termination, DDoS mitigation, rate limiting, and request balancing across Gunicorn workers. |
| **API Gateway Framework** | Python 3.11+ / Flask 3.x | Asynchronous & Blueprint-based REST API framework. |
| **WSGI Server** | Gunicorn (gevent worker model) | Multi-process concurrent request handling. |
| **Cache & Session Store** | Redis 7.x | Sub-millisecond session authentication, rate limiting, and tracking lookup cache. |
| **Security Barriers** | AES-256-GCM & HMAC-SHA256 | Dual-layer Encryption Barrier & Encryption Salting Barrier for payload privacy and tamper-proof hashes. |
| **AI Processing Engine** | Gemini 2.5 Flash LLM (Google GenAI SDK) | Automated context extraction, priority classification (`P1-P4`), and PRR vision proof matching. |
| **Primary Database (main_db)** | PostgreSQL 16+ (SQLAlchemy 2.0 ORM) | Core complaint ledger, user records, and departmental routing index. |
| **Departmental DBs (`dep_01..03`)** | Isolated PostgreSQL Schemas / Databases | Multi-tenant schema isolation for department 01 (Roads), department 02 (Water), and department 03 (Electricity). |
| **SRCS Subsystem** | Celery + Redis Broker | Stage Resolve Commit System managing SLA escalation (24h alert, 36h State DBMS, 72h Central DBMS). |
| **Escalation Data Warehouses** | State & Central Gov. DBMS | External persistence endpoints for L1 (36h) & L2 (72h) SLA breaches. |
| **PRR Engine** | Gemini 2.5 Flash Vision | Problem Resolve Re-evaluation engine auditing officer-submitted resolution proof photos before closure. |

---

## 2. Complete End-to-End System Architecture Topology

```mermaid
flowchart TD
    Client[Citizen / API Client] -->|HTTPS Requests| NGINX[NGINX Load Balancer]
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
```

---

## 3. Recommended Directory Structure (`SIH_02` Flask Project)

```
SIH_02/
├── README.md                    # Main Repository Landing Page with Architecture Diagram
├── Sih02.drawio                 # Master Draw.io System Architecture Diagram
├── docs/                        # Architecture & Strategy Prompt Engineering Docs
│   ├── PRD.md
│   ├── Architecture.md
│   ├── Authentication.md
│   ├── Security_db.md
│   ├── Security_audits.md
│   ├── Skill.md
│   ├── Memory.md
│   ├── Phases.md
│   ├── Rules.md
│   └── README.md
├── nginx/
│   └── nginx.conf               # NGINX Load Balancer & Rate Limit Configuration
├── app/
│   ├── __init__.py              # Flask App Factory & Blueprint Registration
│   ├── config.py                # Environment Config (Prod/Dev/Test)
│   ├── extensions.py            # SQLAlchemy, Redis, Celery, Marshmallow Init
│   ├── api/
│   │   ├── auth.py              # User Session & Re-verification API
│   │   ├── complaints.py        # Complaint Ingestion & Encryption API
│   │   ├── ai_engine.py         # Gemini 2.5 Flash LLM Integration
│   │   ├── departments.py       # dep_01, dep_02, dep_03 Interface Endpoints
│   │   └── srcs_engine.py       # SRCS SLA Escalation & PRR Engine
│   ├── services/
│   │   ├── encryption_service.py # AES-256-GCM Barrier & Salting Hash Logic
│   │   ├── gemini_service.py    # Gemini LLM Context & Priority Prompt Execution
│   │   ├── redis_service.py     # Cache Wrappers & Session Store
│   │   └── srcs_service.py      # SLA 24h/36h/72h Checker & State/Central Sync
│   ├── models/
│   │   ├── user.py              # Citizen & Officer Accounts
│   │   ├── complaint.py         # main_db Master Complaint Model
│   │   ├── department.py        # dep_01, dep_02, dep_03 Department Tables
│   │   └── srcs_audit.py        # SRCS Escalation Audit Log
│   └── tasks/
│       ├── celery_app.py        # Celery Worker Configuration
│       └── sla_tasks.py         # Scheduled Background SLA Audit Tasks
├── requirements.txt             # Python Dependencies
├── Dockerfile                   # Flask App Containerization
├── docker-compose.yml           # Local Orchestration (Flask, Redis, Postgres, NGINX)
└── run.py                       # Application Entry Point
```

---

## 4. Flask WSGI Configuration (`gunicorn.conf.py`)

```python
import multiprocessing

bind = "0.0.0.0:5000"
workers = multiprocessing.cpu_count() * 2 + 1
worker_class = "gevent"
keepalive = 65
timeout = 120
accesslog = "-"
errorlog = "-"
loglevel = "info"
```
