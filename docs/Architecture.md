# System Architecture & Tech Stack (Architecture.md) — Flask Backend

## 1. Technology Stack Overview

| Layer | Technology | Specification / Purpose |
| :--- | :--- | :--- |
| **Reverse Proxy & Load Balancer** | NGINX 1.24+ | SSL Termination, DDoS mitigation, and request balancing across Gunicorn workers |
| **API Gateway Framework** | Python 3.11+ / Flask 3.x | Asynchronous & Blueprint-based REST API framework |
| **WSGI Server** | Gunicorn (gevent worker model) | Multi-process concurrent request handling |
| **Cache & Session Store** | Redis 7.x | Sub-millisecond session authentication, rate limiting, and tracking lookup cache |
| **AI Processing Engine** | Gemini 2.5 Flash LLM (Google GenAI SDK) | Automated context extraction, priority classification & PRR vision proof matching |
| **Primary Database (main_db)** | PostgreSQL 16+ (SQLAlchemy 2.0 ORM) | Core complaint ledger, user records, and departmental routing index |
| **Departmental DBs (`dep_01..03`)** | Isolated PostgreSQL Schemas / Databases | Multi-tenant schema isolation for separate government departments |
| **Escalation Data Warehouses** | State & Central Gov. DBMS | External persistence endpoints for L1 (36h) & L2 (72h) SLA breaches |
| **Asynchronous Task Queue** | Celery + Redis Broker | Background SLA tracking, notification pushes, and batch PRR verification |

---

## 2. Directory Structure (`SIH_02` Flask Project)

```
SIH_02/
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
├── Sih02.drawio                 # Master Draw.io System Architecture Diagram
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

## 3. Data Flow & Subsystem Topology

```mermaid
graph TD
    subgraph Ingestion Layer
        Client([Citizen Client]) -->|HTTPS Request| NGINX[NGINX Load Balancer]
        NGINX -->|Reverse Proxy| Flask[Flask API App]
    end

    subgraph Fast Cache & Auth
        Flask <-->|Sub-ms Check| Redis[(Redis Session Store)]
        Flask -->|Verify Session| Auth[User Session ID Re-verify]
    end

    subgraph Security Barriers
        Flask -->|Raw Payload| EncB[Encryption Barrier AES-256]
        EncB -->|Encrypted String| Track[Tracking Complaint System]
        Track -->|Generate Salted Hash| SaltB[Encryption Salting Barrier]
    end

    subgraph AI Intelligence (Gemini 2.5 Flash)
        Flask -->|Raw Text / Media| Gemini[Gemini 2.5 Flash LLM]
        Gemini -->|Returns JSON| Ctx[Context Analysis]
        Gemini -->|Returns Category| Prio[Priority Classification P1-P4]
    end

    subgraph Persistence & Routing
        Prio -->|Save Complaint| MainDB[(main_db)]
        MainDB -->|Route to dep_01| Dep1[(dep_01 Database)]
        MainDB -->|Route to dep_02| Dep2[(dep_02 Database)]
        MainDB -->|Route to dep_03| Dep3[(dep_03 Database)]
    end

    subgraph SRCS Subsystem
        Dep1 & Dep2 & Dep3 --> Check{Is Problem Resolved?}
        Check -- YES --> Resolved([Complaint Resolved & Closed])
        Check -- NO --> SLA24[Resolve Period 24 hrs]
        SLA24 -- >24h --> SLA36[Staged Period 36 hrs] --> StateDB[(State Gov. DBMS L1)]
        SLA36 -- >36h --> SLA72[Staged Period 72 hrs] --> CentralDB[(Central Gov. DBMS L2)]
        SLA24 & StateDB & CentralDB --> PRR[PRR Engine + Proof Checking]
        PRR -- Verified --> Resolved
        PRR -. Re-sync Hash .-> SaltB
    end
```

---

## 4. Flask & Production WSGI Configuration (`gunicorn.conf.py`)

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
