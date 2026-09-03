# SIH 02 — Smart India Hackathon Project Repository

Welcome to the official organization repository for **SIH 02: Complaint Resolution & Tracking System**.

## 📌 Project Overview
The **SIH 02 Complaint Resolution System** is an enterprise-grade, AI-driven public grievance and complaint resolution platform built using Python (**Flask**). The system automates grievance intake, performs intelligent context and priority analysis using **Gemini 2.5 Flash LLM**, isolates departmental workflows (`dep_01`, `dep_02`, `dep_03`), and enforces strict resolution timelines through the **SRCS (Stage Resolve Commit System)**.

---

## 🏛️ System Architecture Diagram
The architecture is visually documented and compatible with [draw.io](https://app.diagrams.net):
- 📐 [**`Sih02.drawio`**](Sih02.drawio) — Master Backend Architecture Diagram (including NGINX Load Balancer, Redis Cache & Session Store, Encryption & Salting Barriers, Gemini 2.5 Flash LLM, and SRCS SLA Escalation Pipeline).

---

## 📚 Strategy & Specification Documentation Directory (`docs/`)

All prompt engineering and system strategy documents for the Flask backend are stored in the [`docs/`](docs/) directory:

| Document | Description |
| :--- | :--- |
| 📄 [**`docs/PRD.md`**](docs/PRD.md) | **Product Requirement Document**: Vision, API endpoints, NFRs, and feature specifications. |
| 🏗️ [**`docs/Architecture.md`**](docs/Architecture.md) | **System Architecture & Tech Stack**: NGINX Load Balancer, Flask App Factory, Redis, Gemini 2.5 Flash, and DB routing. |
| 🔐 [**`docs/Authentication.md`**](docs/Authentication.md) | **Auth & Session Strategy**: Redis session caching, Argon2id hashing, user re-verification middleware. |
| 🛡️ [**`docs/Security_db.md`**](docs/Security_db.md) | **Database Security & Encryption**: AES-256-GCM Encryption Barrier, HMAC Salting Barrier, and Schema Isolation. |
| 🔍 [**`docs/Security_audits.md`**](docs/Security_audits.md) | **Security Audits & OWASP**: OWASP Top 10 mitigations, audit trails, and SRCS SLA audit verification rules. |
| 🤖 [**`docs/Skill.md`**](docs/Skill.md) | **AI Integration & Prompt Engineering**: Gemini 2.5 Flash LLM JSON prompts for context analysis, priority & PRR vision proof matching. |
| 🧠 [**`docs/Memory.md`**](docs/Memory.md) | **State Management & Memory**: Redis key namespace standards, transient vs persistent memory, and SRCS state machine. |
| 🚀 [**`docs/Phases.md`**](docs/Phases.md) | **Development Roadmap**: 6-phase implementation Gantt chart from environment setup to production audit. |
| 📜 [**`docs/Rules.md`**](docs/Rules.md) | **Coding Guidelines & Blueprint Standards**: Flask coding conventions, safety rules, error handling schemas, and blueprint code templates. |
| 📖 [**`docs/README.md`**](docs/README.md) | **Docs Index**: Full strategy guide and environment configuration. |

---

## 🚀 Quickstart

```bash
# Clone the repository
git clone https://github.com/CSMU-CodeSync/SIH_02-.git
cd SIH_02-

# Set up virtual environment & dependencies
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt
```
