# SIH 02 — Smart India Hackathon Full-Stack Project Repository

Welcome to the official repository for **SIH 02: Complaint Resolution & Tracking System**.

## Project Overview
The **SIH 02 Complaint Resolution System** is an enterprise-grade, AI-driven public grievance and complaint resolution platform. It combines a high-performance **React + Vite + TypeScript Frontend** with a resilient **Python Flask Backend**. 

The system automates grievance intake, performs intelligent context and priority analysis using **Gemini 2.5 Flash LLM**, isolates departmental workflows (`dep_01`, `dep_02`, `dep_03`), and enforces strict resolution timelines through the **SRCS (Stage Resolve Commit System)**.

---

## Full-Stack System Architecture Diagram
The system architecture is visually documented for both frontend and backend components, fully compatible with [draw.io](https://app.diagrams.net):

- Master Backend Draw.io Diagram: [`Sih02.drawio`](Sih02.drawio)
- Master Frontend Draw.io Diagram: [`Frontend/structure/Sih02_Frontend.drawio`](Frontend/structure/Sih02_Frontend.drawio)

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

## Full-Stack Architectural Components Directory

| Layer | Component Name | Technology | Primary Function in SIH 02 |
| :--- | :--- | :--- | :--- |
| **Frontend** | **Client Application** | React 18 + Vite + TypeScript | Responsive web client with Radix UI, Tailwind CSS, Lucide icons, and Recharts analytics. |
| **Frontend** | **State & Offline Store** | Zustand + IndexedDB | Manages auth session state, draft persistence during offline connectivity, and re-auth modals. |
| **Frontend** | **Client Crypto Barrier** | Web Crypto API (AES-256 & HMAC) | Client-side payload encryption and tracking ID formatting prior to transmission. |
| **Frontend** | **API Interceptors** | Axios Interceptors | Attaches Bearer session tokens, handles `401` session expiry, and manages exponential backoff for `429` rate limits. |
| **Backend** | **NGINX Load Balancer** | NGINX 1.24+ | Reverse proxy, SSL termination, DDoS protection, and load balancing across Gunicorn workers. |
| **Backend** | **User Session & Auth Re-verify** | Flask + Redis | Sub-millisecond session authentication and password/OTP re-verification for critical actions. |
| **Backend** | **Redis Cache & Session Store** | Redis 7.x | In-memory storage for active user sessions, rate-limiting counters, and cached complaint status lookups. |
| **Backend** | **Encryption Barrier** | AES-256-GCM | Encrypts sensitive citizen complaint payloads in transit and at rest. |
| **Backend** | **Tracking Complaint System** | SHA-256 Hash | Generates unique tracking hashes (`TRK-XXXXXXXX`) for public citizen status queries. |
| **Backend** | **Encryption Salting Barrier** | HMAC-SHA256 | Adds dynamic secret salts to tracking hashes to prevent hash forgery or unauthorized lookups. |
| **Backend** | **Gemini 2.5 Flash LLM** | Google GenAI SDK | AI engine performing Context Analysis, Priority Classification (`P1-P4`), and PRR Vision Proof Verification. |
| **Backend** | **Main Database (`main_db`)** | PostgreSQL 16+ | Core relational database storing master complaint ledgers, user accounts, and department routing tables. |
| **Backend** | **Departmental DBs (`dep_01..03`)** | Schema-Isolated SQL | Dedicated schema databases for department 01 (Roads), department 02 (Water), and department 03 (Electricity). |
| **Backend** | **Department Interfaces** | Flask Blueprints | Isolated REST API interfaces for departmental officers to process complaints. |
| **Backend** | **SRCS Subsystem** | Celery + Flask Engine | SLA escalation engine managing 24h alerts, 36h State Gov. DBMS sync, 72h Central Gov. DBMS sync, and PRR verification. |
| **Backend** | **State Gov. DBMS (L1 Escalation)** | External PostgreSQL | Persistence warehouse for Level 1 SLA breaches (> 36 hours). |
| **Backend** | **Central Gov. DBMS (L2 Escalation)** | External PostgreSQL | Persistence warehouse for Level 2 SLA breaches (> 72 hours). |
| **Backend** | **PRR Engine & Proof Checking** | Gemini 2.5 Flash Vision | Problem Resolve Re-evaluation engine auditing officer-submitted resolution photos before closing complaints. |

---

## Strategy & Specification Documentation Directory (`docs/` & `Frontend/`)

All prompt engineering, frontend planning, and system strategy documents are stored in the repository:

| Document | Description |
| :--- | :--- |
| [**`docs/PRD.md`**](docs/PRD.md) | **Product Requirement Document**: Vision, API endpoints, NFRs, and feature specifications. |
| [**`docs/Architecture.md`**](docs/Architecture.md) | **System Architecture & Tech Stack**: Full-stack topology, Flask Feature-Based Folder Structure, and Vite setup. |
| [**`docs/Authentication.md`**](docs/Authentication.md) | **Auth & Session Strategy**: Redis session caching, Axios interceptors, Argon2id hashing, user re-verification middleware. |
| [**`docs/Security_db.md`**](docs/Security_db.md) | **Database Security & Encryption**: AES-256-GCM Encryption Barrier, HMAC Salting Barrier, and Schema Isolation. |
| [**`docs/Security_audits.md`**](docs/Security_audits.md) | **Security Audits & OWASP**: OWASP Top 10 mitigations, audit trails, and SRCS SLA audit verification rules. |
| [**`docs/Skill.md`**](docs/Skill.md) | **AI Integration & Prompt Engineering**: Gemini 2.5 Flash LLM JSON prompts for context analysis, priority & PRR vision proof matching. |
| [**`docs/Memory.md`**](docs/Memory.md) | **State Management & Memory**: Redis key namespace standards, Zustand client store, IndexedDB offline drafts, and SRCS state machine. |
| [**`docs/Phases.md`**](docs/Phases.md) | **Development Roadmap**: 6-phase implementation Gantt chart from environment setup to production audit. |
| [**`docs/Rules.md`**](docs/Rules.md) | **Coding Guidelines & Blueprint Standards**: Flask coding conventions, React standards, safety rules, and code templates. |
| [**`docs/README.md`**](docs/README.md) | **Docs Index**: Full strategy guide and environment configuration. |
| [**`Frontend/structure/AUTH_scenerioes.md`**](Frontend/structure/AUTH_scenerioes.md) | **Frontend Error Recovery Matrix**: 5 key UI recovery scenarios (401 Re-Auth, 429 Backoff, Offline IndexedDB, PRR Override). |
| [**`Frontend/structure/frontend_archi.md`**](Frontend/structure/frontend_archi.md) | **Frontend Architecture Specs**: Vite + React + TS architecture, Types, Components, PRR Studio UI, Leaflet Heatmap, and Recharts. |

---

## Quickstart

```bash
# Clone the repository
git clone https://github.com/CSMU-CodeSync/SIH_02-.git
cd SIH_02-

# Backend Setup
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt

# Frontend Setup
cd sih02-frontend
npm install
npm run dev
```
