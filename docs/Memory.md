# AI System Memory & State Management (Memory.md) — Full-Stack

## 1. Executive Summary
This document specifies the state tracking protocols, memory persistence schema, client store management, and Redis key namespace standards for the **SIH 02 Complaint Resolution System**.

System memory is partitioned into three distinct tiers:
1. **Client Transient Memory (Zustand + IndexedDB)**: Persistent client session state and offline complaint draft storage.
2. **Server Fast Memory (Redis In-Memory)**: Sub-millisecond lookup cache for active sessions, user contexts, and tracking lookups.
3. **Persistent SRCS Audit Memory (PostgreSQL / main_db)**: Long-term immutable ledger of complaint state transitions, SLA breaches, and escalation events.

---

## 2. Memory & State Distribution Topology

Transient client memory, server Redis memory, and persistent SQL memory operate across the system architecture as shown:

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

## 3. Client State Store (`src/store/authStore.ts`) & IndexedDB

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

export const useAuthStore = create()(
  persist(
    (set) => ({
      session: null,
      isReverifyModalOpen: false,
      setSession: (session) => set({ session }),
      openReverifyModal: () => set({ isReverifyModalOpen: true }),
      closeReverifyModal: () => set({ isReverifyModalOpen: false }),
      logout: () => set({ session: null }),
    }),
    { name: 'sih02_auth_store' }
  )
);
```

---

## 4. Redis Key Namespace Architecture

| Key Pattern | Data Type | TTL | Description |
| :--- | :--- | :--- | :--- |
| `session:<token>` | JSON Hash | 86,400s (24h) | Citizen / Officer authenticated session data. |
| `reverify:<user_id>` | String | 300s (5m) | Short-lived re-authentication token for critical actions. |
| `trk:<hash>` | JSON Hash | 3,600s (1h) | Cached complaint tracking status for high-speed lookups. |
| `rate:<ip>` | Sorted Set | 60s | Sliding window rate limiting request counter. |
| `srcs:active_sla` | Set | Infinite | Set of unresolved complaint UUIDs currently monitored for SLA 24h/36h/72h. |

---

## 5. SRCS State Machine & Lifecycle Matrix

```mermaid
stateDiagram-v2
    [*] --> INGESTED: Complaint Submitted & Encrypted
    INGESTED --> AI_ANALYZED: Gemini LLM Context & Priority Set
    AI_ANALYZED --> ROUTED_DEP: Saved to main_db & Department Table
    
    state SRCS_Monitoring {
        ROUTED_DEP --> PENDING_RESOLVE: Normal Stage (Under 24h)
        PENDING_RESOLVE --> SLA_24H_BREACH: T > 24 Hours
        SLA_24H_BREACH --> ESCALATED_STATE_L1: T > 36 Hours (State DBMS Sync)
        ESCALATED_STATE_L1 --> ESCALATED_CENTRAL_L2: T > 72 Hours (Central DBMS Sync)
    }

    SRCS_Monitoring --> PRR_AUDIT: Officer Submits Resolution Proof in PRR Studio
    PRR_AUDIT --> RESOLVED_CLOSED: PRR Verification Passed
    PRR_AUDIT --> SLA_24H_BREACH: PRR Proof Rejected (Re-opened)
    
    RESOLVED_CLOSED --> [*]
```
