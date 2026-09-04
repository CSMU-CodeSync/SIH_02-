# AI Skill & System Prompt Specification (Skill.md) — Gemini 2.5 Flash LLM

## 1. Executive Summary
This document specifies the prompt engineering strategies, system instructions, and structured JSON output schemas for integrating **Gemini 2.5 Flash LLM** into the **SIH 02 Complaint Resolution System**.

Gemini 2.5 Flash powers three critical intelligence modules:
1. **Context Analysis Engine**: Semantic intent extraction and automated tagging.
2. **Priority Classification Engine**: Urgency scoring (`P1-CRITICAL` to `P4-LOW`).
3. **PRR (Problem Resolve Re-evaluation) Engine**: Multi-modal vision and text context verification for resolution proof checking in `PRRStudio.tsx`.

---

## 2. AI Intelligence Engine in System Architecture

The Gemini 2.5 Flash LLM connects directly to React UI (`PRRStudio`), Flask API, `main_db`, Redis cache, and the SRCS PRR Engine as shown below:

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

## 3. Gemini 2.5 Flash Service Integration (`app/features/ai_engine/services.py`)

```python
import json
import google.generativeai as genai

class GeminiFlashEngine:
    def __init__(self, api_key: str):
        genai.configure(api_key=api_key)
        self.model = genai.GenerativeModel(
            model_name="gemini-2.5-flash",
            generation_config={"temperature": 0.1, "response_mime_type": "application/json"}
        )

    def analyze_and_classify(self, complaint_text: str) -> dict:
        """Executes combined context extraction and priority classification."""
        prompt = f"""
You are the AI Intelligence Engine for the SIH 02 Public Grievance Platform.
Analyze the following citizen complaint text and respond strictly in JSON.

Complaint Text:
"{complaint_text}"

Return JSON matching this schema:
{{
  "category": "ROADS_INFRASTRUCTURE | WATER_SUPPLY | ELECTRICITY | SANITATION | OTHER",
  "keywords": ["list", "of", "semantic", "keywords"],
  "urgency_score": <integer from 1 to 10>,
  "priority_level": "P1-CRITICAL | P2-HIGH | P3-MEDIUM | P4-LOW",
  "summary": "<1-sentence neutral summary>",
  "target_department": "dep_01 | dep_02 | dep_03"
}}
"""
        response = self.model.generate_content(prompt)
        return json.loads(response.text)
```

---

## 4. PRR (Problem Resolve Re-evaluation) Vision Proof Matcher

When an officer submits resolution proof (e.g. photo of fixed road/pipe), the PRR Engine compares original complaint context with submitted proof:

```python
    def verify_resolution_proof(self, complaint_summary: str, image_bytes: bytes, mime_type: str = "image/jpeg") -> dict:
        """Uses Gemini 2.5 Flash Vision capability to audit resolution proof."""
        image_part = {"mime_type": mime_type, "data": image_bytes}
        
        prompt = f"""
You are the SRCS PRR (Problem Resolve Re-evaluation) Auditor.
Original Complaint Summary: "{complaint_summary}"

Inspect the provided resolution proof image and evaluate whether it genuinely demonstrates that the reported issue has been resolved.

Return JSON matching this schema:
{{
  "is_valid_proof": <boolean true/false>,
  "confidence_score": <float between 0.0 and 1.0>,
  "visual_evidence_observed": "<description of visual proof>",
  "audit_decision": "PASS | REJECT_INSUFFICIENT_PROOF | REJECT_MISMATCH",
  "reasoning": "<explanation for the decision>"
}}
"""
        response = self.model.generate_content([prompt, image_part])
        return json.loads(response.text)
```
