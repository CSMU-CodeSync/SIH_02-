# SIH 02 Complaint Resolution System — Flask Backend Strategy & Specification

Welcome to the official **Flask Backend Strategy & Architecture Specification** directory for the **SIH 02 Complaint Resolution System**. 

This suite of documentation defines the end-to-end architecture, API contracts, security barriers, AI integration, and SLA escalation pipelines for the Flask backend engine.

---

## Master Architecture Topology

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

## Strategy Documentation Directory Map

| Document | Description | Key Architectural Modules Covered |
| :--- | :--- | :--- |
| [**`PRD.md`**](PRD.md) | **Product Requirement Document** | Project vision, API endpoints, NFRs, and feature specifications. |
| [**`Architecture.md`**](Architecture.md) | **System Architecture & Topology** | Flask App Factory, NGINX Load Balancer, Redis, Gemini 2.5 Flash, and DB routing. |
| [**`Authentication.md`**](Authentication.md) | **Auth & Session Strategy** | Redis session caching, Argon2id hashing, user re-verification middleware. |
| [**`Security_db.md`**](Security_db.md) | **Database Security & Encryption** | AES-256-GCM Encryption Barrier, HMAC Salting Barrier, and Schema Isolation. |
| [**`Security_audits.md`**](Security_audits.md) | **Security Audits & OWASP** | OWASP Top 10 mitigations, audit trails, and SRCS SLA audit verification rules. |
| [**`Skill.md`**](Skill.md) | **AI Integration & Prompt Engineering** | Gemini 2.5 Flash LLM JSON prompts for context analysis, priority & PRR vision proof matching. |
| [**`Memory.md`**](Memory.md) | **State Management & Memory** | Redis key namespace standards, transient vs persistent memory, and SRCS state machine. |
| [**`Phases.md`**](Phases.md) | **Development Roadmap** | 6-phase implementation Gantt chart from environment setup to production audit. |
| [**`Rules.md`**](Rules.md) | **Coding Guidelines & Blueprint Standards** | Flask coding conventions, safety rules, error handling schemas, and blueprint code templates. |

---

## Quickstart & Environment Setup

### 1. Local Development Setup
```bash
# Clone repository
git clone https://github.com/CSMU-CodeSync/SIH_02-.git
cd SIH_02-

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Start Redis & PostgreSQL via Docker
docker-compose up -d redis postgres

# Run Flask backend server
python run.py
```

### 2. Environment Variables (`.env`)
```ini
FLASK_ENV=development
SECRET_KEY=your-super-secret-flask-key
MASTER_ENCRYPTION_KEY_B64=your-base64-aes-256-key
SALT_SECRET=your-hmac-salting-secret
DATABASE_URL=postgresql://user:password@localhost:5432/sih02_db
REDIS_URL=redis://localhost:6379/0
GEMINI_API_KEY=your-google-gemini-api-key
```
