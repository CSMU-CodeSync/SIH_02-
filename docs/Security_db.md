# Database Security & Data Privacy Strategy (Security_db.md) — Full-Stack

## 1. Executive Summary
This document specifies the database security, encryption protocols, and multi-tenant schema isolation architecture for the **SIH 02 Complaint Resolution System**. 

The database architecture employs **dual-layer cryptographic barriers**:
1. **Encryption Barrier**: AES-256-GCM symmetric encryption for sensitive citizen complaint text and attachments in transit and at rest (supported by Web Crypto API on client and Cryptography library on Flask).
2. **Encryption Salting Barrier**: HMAC-SHA256 salted hashing for complaint tracking IDs and verification lookup tokens.

---

## 2. Security Architecture Placement

The Encryption Barrier, Salting Barrier, main_db, isolated departmental schemas (`dep_01..03`), and external State/Central DBMS fit into the master architecture as shown:

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

## 3. Implementation Specifications: Dual Barrier Services

### 3.1 Client-Side Crypto Barrier (`src/lib/crypto.ts`)

```typescript
export class ClientCryptoBarrier {
  static async encryptPayload(plainText: string, secretKeyHex: string): Promise<string> {
    const enc = new TextEncoder();
    const keyData = new Uint8Array(secretKeyHex.match(/.{1,2}/g)!.map(byte => parseInt(byte, 16)));
    const key = await window.crypto.subtle.importKey("raw", keyData, { name: "AES-GCM" }, false, ["encrypt"]);
    const nonce = window.crypto.getRandomValues(new Uint8Array(12));
    const encrypted = await window.crypto.subtle.encrypt({ name: "AES-GCM", iv: nonce }, key, enc.encode(plainText));
    const combined = new Uint8Array(nonce.length + encrypted.byteLength);
    combined.set(nonce);
    combined.set(new Uint8Array(encrypted), nonce.length);
    return btoa(String.fromCharCode(...combined));
  }
}
```

### 3.2 Server Encryption Barrier Service (`app/core/security.py`)

```python
import os
import base64
from cryptography.hazmat.primitives.ciphers.aead import AESGCM

class EncryptionBarrierService:
    def __init__(self, master_key_b64: str):
        self.key = base64.b64decode(master_key_b64)
        self.aesgcm = AESGCM(self.key)

    def encrypt_payload(self, plain_text: str) -> str:
        """Encrypts sensitive complaint text using AES-256-GCM with a 96-bit nonce."""
        nonce = os.urandom(12)
        ciphertext = self.aesgcm.encrypt(nonce, plain_text.encode("utf-8"), None)
        packed = nonce + ciphertext
        return base64.b64encode(packed).decode("utf-8")

    def decrypt_payload(self, encrypted_b64: str) -> str:
        """Decrypts AES-256-GCM ciphertext payload."""
        packed = base64.b64decode(encrypted_b64.encode("utf-8"))
        nonce = packed[:12]
        ciphertext = packed[12:]
        decrypted_bytes = self.aesgcm.decrypt(nonce, ciphertext, None)
        return decrypted_bytes.decode("utf-8")
```

---

## 4. Multi-Tenant Departmental DB Isolation (`dep_01`, `dep_02`, `dep_03`)

To guarantee strict data security between different government sectors (e.g. Roads/Infrastructure, Water Supply, Electricity), the system enforces **PostgreSQL Schema Isolation**:

```sql
-- Schema creation for isolated department databases
CREATE SCHEMA IF NOT EXISTS dep_01_schema;
CREATE SCHEMA IF NOT EXISTS dep_02_schema;
CREATE SCHEMA IF NOT EXISTS dep_03_schema;

-- Isolated Complaint Table for Department 01
CREATE TABLE dep_01_schema.complaints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    master_complaint_id UUID NOT NULL,
    encrypted_details TEXT NOT NULL,
    priority_level VARCHAR(20) NOT NULL,
    status VARCHAR(30) DEFAULT 'PENDING_RESOLVE',
    assigned_officer_id UUID,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);
```

---

## 5. External Escalation DBMS Persistence (State & Central DBMS)

When complaints breach the 36h or 72h SLA boundaries within the **SRCS (Stage Resolve Commit System)**, the system persists audit records to State and Central Government databases:

| Escalation Tier | SLA Boundary | Target DBMS | Security Protocol |
| :--- | :--- | :--- | :--- |
| **Level 1 Escalation** | > 36 Hours | `State Gov. DBMS` | TLS 1.3 + Client Certificate (mTLS) + HMAC Audit Signature |
| **Level 2 Escalation** | > 72 Hours | `Central Gov. DBMS` | TLS 1.3 + mTLS + Immutable Append-Only Ledger Transaction |
