# SIH 02: Complaint Resolution System — Frontend Implementation Guide

---

## 1. Project Initialization & Dependencies

### 1.1 Project Setup with Vite & TypeScript

```bash
# Create Vite React-TS Project
npm create vite@latest sih02-frontend -- --template react-ts
cd sih02-frontend

# Install Core Dependencies
npm install @tanstack/react-query @tanstack/react-query-devtools zustand clsx tailwind-merge lucide-react framer-motion axios zod react-hook-form @hookform/resolvers
npm install @radix-ui/react-dialog @radix-ui/react-tabs @radix-ui/react-dropdown-menu @radix-ui/react-toast @radix-ui/react-tooltip @radix-ui/react-progress @radix-ui/react-accordion
npm install recharts leaflet react-leaflet @types/leaflet
npm install -D tailwindcss postcss autoprefixer @types/node
```

### 1.2 Tailwind CSS Configuration (`tailwind.config.js`)

```javascript
/** @type {import('tailwindcss').Config} */
export default {
  darkMode: ["class"],
  content: ["./index.html", "./src/**/*.{js,ts,jsx,tsx}"],
  theme: {
    extend: {
      colors: {
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        primary: {
          DEFAULT: "#2563eb",
          foreground: "#ffffff",
        },
        department: {
          dep01: "#f97316", // Roads & Infrastructure (Orange)
          dep02: "#0284c7", // Water Supply & Drainage (Blue)
          dep03: "#eab308", // Electricity & Power (Amber)
        },
        sla: {
          normal: "#22c55e",  // 0 - 24 hrs
          warning: "#f59e0b", // 24 - 36 hrs
          state: "#ea580c",   // 36 - 72 hrs (L1 State DBMS)
          central: "#dc2626", // > 72 hrs (L2 Central DBMS)
        }
      },
      animation: {
        "pulse-fast": "pulse 1.2s cubic-bezier(0.4, 0, 0.6, 1) infinite",
      }
    },
  },
  plugins: [],
};
```

---

## 2. Type Definitions (`src/types/index.ts`)

```typescript
export type DepartmentId = 'dep_01' | 'dep_02' | 'dep_03';
export type PriorityLevel = 'P1-CRITICAL' | 'P2-HIGH' | 'P3-MEDIUM' | 'P4-LOW';
export type SlaStage = 'NORMAL_24H' | 'STAGE_36H_STATE' | 'STAGE_72H_CENTRAL' | 'RESOLVED_CLOSED';

export interface AIAnalysisResult {
  category: 'ROADS_INFRASTRUCTURE' | 'WATER_SUPPLY' | 'ELECTRICITY' | 'SANITATION' | 'OTHER';
  keywords: string[];
  urgency_score: number; // 1 to 10
  priority_level: PriorityLevel;
  summary: string;
  target_department: DepartmentId;
}

export interface PRRAuditResult {
  is_valid_proof: boolean;
  confidence_score: number; // 0.0 to 1.0
  visual_evidence_observed: string;
  audit_decision: 'PASS' | 'REJECT_INSUFFICIENT_PROOF' | 'REJECT_MISMATCH';
  reasoning: string;
}

export interface Complaint {
  id: string;
  tracking_hash: string; // e.g. TRK-8F29A10B
  encrypted_details: string;
  department_id: DepartmentId;
  priority_level: PriorityLevel;
  status: SlaStage;
  ai_context: AIAnalysisResult;
  created_at: string;
  sla_deadline_24h: string;
  sla_deadline_36h: string;
  sla_deadline_72h: string;
  resolution_proof_url?: string;
  prr_result?: PRRAuditResult;
}

export interface UserSession {
  user_id: string;
  name: string;
  role: 'CITIZEN' | 'OFFICER_DEP01' | 'OFFICER_DEP02' | 'OFFICER_DEP03' | 'AUDITOR_SRCS';
  token: string;
  reverify_token?: string;
  reverify_expires_at?: number;
}
```

---

## 3. Cryptographic Barrier & Security Layer (`src/lib/crypto.ts`)

```typescript
/**
 * Client-Side AES-256-GCM & HMAC-SHA256 Utilities
 * Compatible with Flask Encryption Barrier & Salting Barrier
 */
export class ClientCryptoBarrier {
  /**
   * Generates a client-side pre-encrypted payload buffer
   */
  static async encryptPayload(plainText: string, secretKeyHex: string): Promise<string> {
    const enc = new TextEncoder();
    const keyData = new Uint8Array(secretKeyHex.match(/.{1,2}/g)!.map(byte => parseInt(byte, 16)));
    
    const key = await window.crypto.subtle.importKey(
      "raw",
      keyData,
      { name: "AES-GCM" },
      false,
      ["encrypt"]
    );

    const nonce = window.crypto.getRandomValues(new Uint8Array(12));
    const encrypted = await window.crypto.subtle.encrypt(
      { name: "AES-GCM", iv: nonce },
      key,
      enc.encode(plainText)
    );

    // Pack Nonce + Ciphertext
    const combined = new Uint8Array(nonce.length + encrypted.byteLength);
    combined.set(nonce);
    combined.set(new Uint8Array(encrypted), nonce.length);

    return btoa(String.fromCharCode(...combined));
  }

  /**
   * Client-Side Tracking Hash Formatter
   */
  static formatTrackingId(rawHash: string): string {
    return rawHash.startsWith("TRK-") ? rawHash : `TRK-${rawHash.substring(0, 12).toUpperCase()}`;
  }
}
```

---

## 4. API Client & Interceptors (`src/lib/api-client.ts`)

```typescript
import axios from 'axios';
import { useAuthStore } from '../store/authStore';

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api/v1',
  headers: {
    'Content-Type': 'application/json',
  },
  timeout: 10000,
});

// Request Interceptor: Attach Session Bearer Token & Re-verify Token
apiClient.interceptors.request.use((config) => {
  const session = useAuthStore.getState().session;
  if (session?.token) {
    config.headers.Authorization = `Bearer ${session.token}`;
  }
  if (session?.reverify_token) {
    config.headers['X-Reverify-Token'] = session.reverify_token;
  }
  return config;
});

// Response Interceptor: Handle Re-verification Required (403) or Session Expired (401)
apiClient.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().logout();
      window.location.href = '/login';
    } else if (error.response?.status === 403 && error.response?.data?.require_reverify) {
      useAuthStore.getState().openReverifyModal();
    }
    return Promise.reject(error);
  }
);
```

---

## 5. Global State Store (`src/store/authStore.ts`)

```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';
import { UserSession } from '../types';

interface AuthState {
  session: UserSession | null;
  isReverifyModalOpen: boolean;
  setSession: (session: UserSession) => void;
  setReverifyToken: (token: string, ttlSeconds: number) => void;
  openReverifyModal: () => void;
  closeReverifyModal: () => void;
  logout: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      session: null,
      isReverifyModalOpen: false,
      setSession: (session) => set({ session }),
      setReverifyToken: (token, ttlSeconds) =>
        set((state) => ({
          session: state.session
            ? {
                ...state.session,
                reverify_token: token,
                reverify_expires_at: Date.now() + ttlSeconds * 1000,
              }
            : null,
          isReverifyModalOpen: false,
        })),
      openReverifyModal: () => set({ isReverifyModalOpen: true }),
      closeReverifyModal: () => set({ isReverifyModalOpen: false }),
      logout: () => set({ session: null, isReverifyModalOpen: false }),
    }),
    { name: 'sih02_auth_store' }
  )
);
```

---

## 6. Real-Time Tracking Barometer Hook (`src/features/tracking/useTrackStatus.ts`)

```typescript
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '../../lib/api-client';
import { Complaint } from '../../types';

export function useTrackStatus(trackingHash: string) {
  return useQuery({
    queryKey: ['complaint', trackingHash],
    queryFn: async () => {
      // Sub-50ms endpoint querying Redis Cache first, falling back to PostgreSQL
      const { data } = await apiClient.get<Complaint>(`/complaints/track/${trackingHash}`);
      return data;
    },
    enabled: !!trackingHash && trackingHash.length >= 8,
    staleTime: 30000, // 30 seconds fresh cache
    refetchInterval: (query) => {
      const status = query.state.data?.status;
      return status === 'RESOLVED_CLOSED' ? false : 15000; // Poll active complaints every 15s
    },
  });
}
```

---

## 7. Gemini 2.5 Flash PRR Vision Proof Auditor (`src/features/prr-auditor/PRRStudio.tsx`)

```tsx
import React, { useState } from 'react';
import { PRRAuditResult } from '../../types';
import { CheckCircle2, XCircle, AlertTriangle, Eye } from 'lucide-react';
import { apiClient } from '../../lib/api-client';

interface PRRStudioProps {
  complaintId: string;
  originalImageUrl: string;
  originalSummary: string;
  proofImageUrl: string;
  onAuditComplete: () => void;
}

export const PRRStudio: React.FC<PRRStudioProps> = ({
  complaintId,
  originalImageUrl,
  originalSummary,
  proofImageUrl,
  onAuditComplete,
}) => {
  const [loading, setLoading] = useState(false);
  const [prrResult, setPrrResult] = useState<PRRAuditResult | null>(null);

  const handleRunPRRAnalysis = async () => {
    setLoading(true);
    try {
      const { data } = await apiClient.post<PRRAuditResult>(`/srcs/prr-verify`, {
        complaint_id: complaintId,
      });
      setPrrResult(data);
    } catch (err) {
      console.error('PRR Verification error:', err);
    } finally {
      setLoading(false);
    }
  };

  const handleFinalResolution = async (decision: 'APPROVE_CLOSE' | 'REJECT_REOPEN') => {
    await apiClient.post(`/srcs/commit-resolution`, {
      complaint_id: complaintId,
      action: decision,
    });
    onAuditComplete();
  };

  return (
    <div className="bg-slate-900 border border-slate-800 rounded-xl p-6 text-white shadow-2xl space-y-6">
      <div className="flex justify-between items-center border-b border-slate-800 pb-4">
        <div>
          <h3 className="text-lg font-bold text-purple-400 flex items-center gap-2">
            <Eye className="w-5 h-5" /> Gemini 2.5 Flash — PRR (Problem Resolve Re-evaluation) Studio
          </h3>
          <p className="text-xs text-slate-400">Comparing original reported incident against submitted officer proof</p>
        </div>
        <button
          onClick={handleRunPRRAnalysis}
          disabled={loading}
          className="px-4 py-2 bg-purple-600 hover:bg-purple-700 disabled:opacity-50 text-xs font-semibold rounded-lg shadow"
        >
          {loading ? 'Analyzing with Gemini Vision...' : 'Run PRR AI Audit'}
        </button>
      </div>

      {/* Side-by-Side Comparison */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
        <div className="bg-slate-950 p-4 rounded-lg border border-slate-800">
          <span className="text-xs font-semibold text-slate-400 uppercase tracking-wider block mb-2">Original Incident</span>
          <img src={originalImageUrl} alt="Original Incident" className="w-full h-48 object-cover rounded-md mb-2 border border-slate-800" />
          <p className="text-xs text-slate-300 italic">"{originalSummary}"</p>
        </div>

        <div className="bg-slate-950 p-4 rounded-lg border border-slate-800">
          <span className="text-xs font-semibold text-emerald-400 uppercase tracking-wider block mb-2">Submitted Resolution Proof</span>
          <img src={proofImageUrl} alt="Resolution Proof" className="w-full h-48 object-cover rounded-md mb-2 border border-slate-800" />
          <span className="text-xs text-slate-400">Captured by Field Officer</span>
        </div>
      </div>

      {/* Gemini AI Result Card */}
      {prrResult && (
        <div className={`p-4 rounded-lg border ${prrResult.is_valid_proof ? 'bg-emerald-950/40 border-emerald-500/50' : 'bg-rose-950/40 border-rose-500/50'} space-y-3`}>
          <div className="flex justify-between items-center">
            <span className="text-sm font-bold flex items-center gap-2">
              {prrResult.is_valid_proof ? <CheckCircle2 className="text-emerald-400 w-5 h-5" /> : <XCircle className="text-rose-400 w-5 h-5" />}
              AI Decision: {prrResult.audit_decision}
            </span>
            <span className="text-xs px-2.5 py-1 bg-slate-800 rounded-full font-mono text-purple-300">
              Confidence: {(prrResult.confidence_score * 100).toFixed(1)}%
            </span>
          </div>
          <p className="text-xs text-slate-300"><strong>Visual Evidence Observed:</strong> {prrResult.visual_evidence_observed}</p>
          <p className="text-xs text-slate-400"><strong>Reasoning:</strong> {prrResult.reasoning}</p>

          <div className="flex justify-end gap-3 pt-3 border-t border-slate-800">
            <button
              onClick={() => handleFinalResolution('REJECT_REOPEN')}
              className="px-4 py-1.5 bg-rose-700 hover:bg-rose-800 text-xs font-semibold rounded-md"
            >
              Reject Proof & Re-open
            </button>
            <button
              onClick={() => handleFinalResolution('APPROVE_CLOSE')}
              className="px-4 py-1.5 bg-emerald-600 hover:bg-emerald-700 text-xs font-semibold rounded-md"
            >
              Approve & Close Complaint
            </button>
          </div>
        </div>
      )}
    </div>
  );
};
```

---

## 8. SRCS SLA Escalation Countdown Component (`src/components/SlaCountdown.tsx`)

```tsx
import React, { useEffect, useState } from 'react';
import { SlaStage } from '../types';

interface SlaCountdownProps {
  deadline: string;
  stage: SlaStage;
}

export const SlaCountdown: React.FC<SlaCountdownProps> = ({ deadline, stage }) => {
  const [timeLeft, setTimeLeft] = useState({ hours: 0, minutes: 0, seconds: 0, isBreached: false });

  useEffect(() => {
    const target = new Date(deadline).getTime();

    const interval = setInterval(() => {
      const now = new Date().getTime();
      const difference = target - now;

      if (difference <= 0) {
        setTimeLeft({ hours: 0, minutes: 0, seconds: 0, isBreached: true });
      } else {
        const hours = Math.floor(difference / (1000 * 60 * 60));
        const minutes = Math.floor((difference % (1000 * 60 * 60)) / (1000 * 60));
        const seconds = Math.floor((difference % (1000 * 60)) / 1000);
        setTimeLeft({ hours, minutes, seconds, isBreached: false });
      }
    }, 1000);

    return () => clearInterval(interval);
  }, [deadline]);

  const getStageBadge = () => {
    switch (stage) {
      case 'NORMAL_24H':
        return <span className="bg-emerald-500/20 text-emerald-400 border border-emerald-500/30 px-2 py-0.5 rounded text-xs">Stage 1: Normal (24h)</span>;
      case 'STAGE_36H_STATE':
        return <span className="bg-amber-500/20 text-amber-400 border border-amber-500/30 px-2 py-0.5 rounded text-xs animate-pulse">Stage 2: L1 State DBMS (36h)</span>;
      case 'STAGE_72H_CENTRAL':
        return <span className="bg-rose-500/20 text-rose-400 border border-rose-500/30 px-2 py-0.5 rounded text-xs animate-pulse-fast">Stage 3: L2 Central DBMS (&gt;72h)</span>;
      case 'RESOLVED_CLOSED':
        return <span className="bg-blue-500/20 text-blue-400 border border-blue-500/30 px-2 py-0.5 rounded text-xs">Resolved & Closed</span>;
    }
  };

  return (
    <div className="flex items-center gap-3">
      {getStageBadge()}
      {!timeLeft.isBreached && stage !== 'RESOLVED_CLOSED' ? (
        <span className="font-mono text-sm text-slate-200 font-semibold">
          {String(timeLeft.hours).padStart(2, '0')}h : {String(timeLeft.minutes).padStart(2, '0')}m : {String(timeLeft.seconds).padStart(2, '0')}s
        </span>
      ) : stage !== 'RESOLVED_CLOSED' ? (
        <span className="text-xs text-rose-400 font-bold uppercase tracking-wider">SLA Breached — Escalated</span>
      ) : null}
    </div>
  );
};
```

---

## 9. Verification & Testing Checklist

- [x] **Sub-50ms Lookup Speed**: Verify Redis caching headers (`X-Cache: HIT`) for `/complaints/track/:trackingHash`.
- [x] **Client AES-256-GCM Encryption**: Confirm that grievance text is encrypted before transmission and matches Flask's `EncryptionBarrierService`.
- [x] **Gemini 2.5 Flash Structured JSON**: Test live pre-scan and ensure deterministic fallback if API latency exceeds 2.0 seconds.
- [x] **Department Schema Separation**: Validate that officers in `dep_01` cannot read or mutate complaints in `dep_02` or `dep_03`.
- [x] **SRCS 24h/36h/72h SLA State Transitions**: Test simulated Celery cron task triggers and verify State & Central DBMS audit logs.
- [x] **PRR Vision Verification**: Validate multimodal visual proof comparison with confidence score thresholds ($\ge 85\%$ for auto-pass).
