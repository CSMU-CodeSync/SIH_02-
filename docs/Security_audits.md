# Security Audits, SLA Compliance & Verification Rules (Security_audits.md) — Flask Backend

## 1. Executive Summary
This document defines the security audit standards, OWASP compliance rules, and audit verification protocols for the **SIH 02 Complaint Resolution System**.

Special focus is given to auditing the **SRCS (Stage Resolve Commit System)** SLA transitions (24h/36h/72h) and verifying resolution proofs via the **PRR (Problem Resolve Re-evaluation) Engine**.

---

## 2. System Security Audit Topology

Security auditing monitors every layer of the system architecture from NGINX load balancing to external State/Central DBMS persistence:

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

## 3. OWASP Top 10 Flask Mitigation Matrix

| Vulnerability | Threat Description | Flask Backend Mitigation |
| :--- | :--- | :--- |
| **A01: Broken Access Control** | Unauthorized access to department endpoints (`dep_01..03`) | Role-Based Access Control (RBAC) middleware verifying session role claims in Redis. |
| **A02: Cryptographic Failures** | Plaintext transmission of citizen grievances | Mandatory SSL/TLS 1.3 at NGINX, AES-256-GCM Encryption Barrier, and HMAC-SHA256 Salting Barrier. |
| **A03: Injection** | SQL Injection via raw query parameters | Mandatory SQLAlchemy 2.0 ORM parameterization; zero raw SQL strings. |
| **A04: Insecure Design** | Premature or fake complaint resolution by officers | **SRCS PRR Engine**: Mandatory proof checking with Gemini 2.5 Flash vision/text context match. |
| **A05: Security Misconfiguration** | Exposed debug modes, default secret keys | Environment variables validated at boot via Flask config schema. |
| **A07: Identification Failures** | Brute force session hijacking | Redis Sliding Window Rate Limiting (max 60 req/min per IP, 5 login attempts/min). |

---

## 4. SRCS SLA Audit & Escalation Engine Service (`app/services/srcs_service.py`)

```python
from datetime import datetime, timezone
from app.models.complaint import Complaint
from app.extensions import db, redis_client

class SRCSAuditEngine:
    @staticmethod
    def audit_complaint_sla(complaint_id: str):
        """Evaluates SLA boundary timestamps and triggers automated governance escalation."""
        complaint = Complaint.query.get(complaint_id)
        if not complaint or complaint.status == "RESOLVED":
            return
        
        now = datetime.now(timezone.utc)
        elapsed_hours = (now - complaint.created_at).total_seconds() / 3600.0

        if elapsed_hours >= 72 and complaint.srcs_stage != "ESCALATED_CENTRAL":
            complaint.srcs_stage = "ESCALATED_CENTRAL"
            complaint.escalated_central_at = now
            # Trigger persistence to Central Gov. DBMS (L2)
            db.session.commit()
            redis_client.publish("srcs_alerts", f"CENTRAL_ESCALATION:{complaint_id}")

        elif elapsed_hours >= 36 and complaint.srcs_stage not in ["ESCALATED_STATE", "ESCALATED_CENTRAL"]:
            complaint.srcs_stage = "ESCALATED_STATE"
            complaint.escalated_state_at = now
            # Trigger persistence to State Gov. DBMS (L1)
            db.session.commit()
            redis_client.publish("srcs_alerts", f"STATE_ESCALATION:{complaint_id}")

        elif elapsed_hours >= 24 and complaint.srcs_stage == "NORMAL":
            complaint.srcs_stage = "SLA_24H_BREACH"
            db.session.commit()
```

---

## 5. Security Logging & Audit Trail Standards

Every security-sensitive event (login, token re-verification, encryption barrier access, department routing, SRCS SLA escalation) produces an **immutable audit log entry**:

```json
{
  "timestamp": "2026-09-04T10:10:00Z",
  "event_type": "SRCS_SLA_ESCALATION",
  "complaint_id": "c7a8e912-3b4f-4d1a-8e99-123456789abc",
  "department_id": "dep_01",
  "elapsed_hours": 36.5,
  "action_taken": "ESCALATED_TO_STATE_GOV_DBMS_L1",
  "hash_signature": "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
  "client_ip": "10.0.4.12"
}
```
