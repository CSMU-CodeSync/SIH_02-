# SIH 02 Complaint Resolution System — Flask Backend Strategy & Specification

Welcome to the official **Flask Backend Strategy & Architecture Specification** directory for the **SIH 02 Complaint Resolution System**. 

This suite of documentation defines the end-to-end architecture, API contracts, security barriers, AI integration, and SLA escalation pipelines for the Flask backend engine.

---

## Strategy Documentation Directory Map

| Document | Description | Key Modules Covered |
| :--- | :--- | :--- |
| 📄 [**`PRD.md`**](PRD.md) | **Product Requirement Document** | Project vision, API endpoints, NFRs, and feature specifications. |
| 🏗️ [**`Architecture.md`**](Architecture.md) | **System Architecture & Topology** | Flask App Factory, NGINX Load Balancer, Redis, Gemini 2.5 Flash, and DB routing. |
| 🔐 [**`Authentication.md`**](Authentication.md) | **Auth & Session Strategy** | Redis session caching, Argon2id hashing, user re-verification middleware. |
| 🛡️ [**`Security_db.md`**](Security_db.md) | **Database Security & Encryption** | AES-256-GCM Encryption Barrier, HMAC Salting Barrier, and Schema Isolation. |
| 🔍 [**`Security_audits.md`**](Security_audits.md) | **Security Audits & OWASP** | OWASP Top 10 mitigations, audit trails, and SRCS SLA audit verification rules. |
| 🤖 [**`Skill.md`**](Skill.md) | **AI Integration & Prompt Engineering** | Gemini 2.5 Flash LLM JSON prompts for context analysis, priority & PRR vision proof matching. |
| 🧠 [**`Memory.md`**](Memory.md) | **State Management & Memory** | Redis namespace standards, transient vs persistent memory, and SRCS state machine. |
| 🚀 [**`Phases.md`**](Phases.md) | **Development Roadmap** | 6-phase implementation Gantt chart from environment setup to production audit. |
| 📜 [**`Rules.md`**](Rules.md) | **Coding Guidelines & Blueprint Standards** | Flask coding conventions, safety rules, error handling schemas, and blueprint code templates. |

---

## Core System Highlights

```mermaid
flowchart LR
    Client[Citizen / Client] --> NGINX[NGINX Load Balancer]
    NGINX --> Flask[Flask API App]
    Flask <--> Redis[(Redis Session & Cache)]
    Flask <--> Gemini[Gemini 2.5 Flash LLM]
    Flask --> EncB[Encryption Barrier]
    EncB --> MainDB[(main_db Core Ledger)]
    MainDB --> Dep1[(dep_01)] & Dep2[(dep_02)] & Dep3[(dep_03)]
    Dep1 & Dep2 & Dep3 --> SRCS[SRCS 24h/36h/72h SLA Escalation Engine]
    SRCS --> StateDB[(State Gov. DBMS)] & CentralDB[(Central Gov. DBMS)] & PRR[PRR Vision Proof Verification]
```

---

## Quickstart & Environment Setup

### 1. Local Development Setup
```bash
# Clone repository
git clone <your-sih-repo-url>
cd SIH_02

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
