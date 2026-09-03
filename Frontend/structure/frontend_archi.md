# SIH 02: Grievance Prioritization System — Front End 

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
npm install react-leaflet-cluster
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

## 9. Analytics Dashboard and Visual Components

The following components make every chart named in the architecture visible from one responsive SRCS dashboard. The dashboard uses a 12-column grid on desktop and one column on mobile.

### 9.1 Dashboard route and data contract

Route: `/srcs/analytics`

```typescript
export interface MapPoint {
  id: string;
  trackingHash: string;
  latitude: number;
  longitude: number;
  priority: PriorityLevel;
  department: DepartmentId;
  status: SlaStage;
  createdAt: string;
}

export interface AnalyticsSnapshot {
  totals: {
    open: number;
    resolved: number;
    breached: number;
    averageResolutionHours: number;
  };
  dailyVolume: Array<{ date: string; received: number; resolved: number }>;
  departmentVelocity: Array<{
    department: DepartmentId;
    open: number;
    resolved: number;
    averageResolutionHours: number;
  }>;
  slaDistribution: Array<{
    stage: SlaStage;
    count: number;
  }>;
  mapPoints: MapPoint[];
}
```

The page must show the selected date range, department filter, refresh state, last-updated timestamp, and an accessible empty state. Every request must be scoped to the authenticated user's role.

### 9.2 Required chart components

Create these components under `src/features/analytics/components/`:

| Component | Chart | Visible information |
| --- | --- | --- |
| `ComplaintVolumeChart` | Recharts `AreaChart` | Received versus resolved complaints by day |
| `DepartmentVelocityChart` | Recharts `BarChart` | Open, resolved, and average resolution hours per department |
| `SlaDistributionChart` | Recharts `PieChart` | Normal, state escalation, central escalation, and resolved counts |
| `SlaThresholdGauge` | CSS conic gauge | Current percentage within each 24h, 36h, and 72h threshold |
| `ComplaintHeatmap` | Leaflet map with clustered circles | Geographic complaint density and priority |

Example chart implementation:

```tsx
import {
  Area,
  AreaChart,
  CartesianGrid,
  Legend,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from 'recharts';

interface ComplaintVolumeChartProps {
  data: Array<{ date: string; received: number; resolved: number }>;
}

export function ComplaintVolumeChart({ data }: ComplaintVolumeChartProps) {
  return (
    <section aria-labelledby="complaint-volume-title" className="min-w-0">
      <h2 id="complaint-volume-title" className="text-base font-semibold">Complaint volume</h2>
      <div className="h-72 min-h-72 w-full">
        <ResponsiveContainer width="100%" height="100%">
          <AreaChart data={data}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="date" />
            <YAxis allowDecimals={false} />
            <Tooltip />
            <Legend />
            <Area dataKey="received" name="Received" stroke="#2563eb" fill="#bfdbfe" />
            <Area dataKey="resolved" name="Resolved" stroke="#16a34a" fill="#bbf7d0" />
          </AreaChart>
        </ResponsiveContainer>
      </div>
    </section>
  );
}
```

The gauge must expose the same value as text for screen readers. Use green for under 24 hours, amber for 24 to 36 hours, orange for 36 to 72 hours, and red for over 72 hours. Do not communicate status by color alone.

### 9.3 Heatmap behavior

`ComplaintHeatmap` must:

- Use `react-leaflet` with a fixed `min-h-80` map region so the layout cannot collapse while tiles load.
- Cluster nearby complaints and show a count at low zoom.
- Show a detail popup with tracking hash, priority, department, status, and created time.
- Include a legend for priority levels and a visible loading, error, and no-location state.
- Avoid exposing encrypted complaint details or personally identifiable information.

## 10. Tables, Queues, and Tracking Views

### 10.1 Shared table component

Create `src/components/DataTable.tsx` with typed columns and these required behaviors:

```typescript
export interface DataTableColumn<Row> {
  key: string;
  header: string;
  render: (row: Row) => React.ReactNode;
  sortable?: boolean;
}

interface DataTableProps<Row> {
  rows: Row[];
  columns: DataTableColumn<Row>[];
  rowKey: (row: Row) => string;
  loading?: boolean;
  errorMessage?: string;
  emptyMessage: string;
  search?: string;
  onSearchChange?: (value: string) => void;
  filterControls?: React.ReactNode;
  page?: number;
  pageCount?: number;
  onPageChange?: (page: number) => void;
}
```

The table must support keyboard focus, column sorting, server-side pagination, search, department/priority/status filters, a mobile stacked-row layout, and an explicit empty state. Keep tracking hashes and status visible; never render encrypted details in a list.

### 10.2 Required table and queue routes

| Route | Component | Required columns or content |
| --- | --- | --- |
| `/srcs/audit` | `AuditTrailTable` | Event time, tracking hash, actor role, action, source system, result, correlation ID |
| `/officer/dep_01` | `DepartmentKanban` | New, assigned, in progress, awaiting proof, resolved; each card shows priority, SLA badge, hash, and assignee |
| `/officer/dep_02` | `PipelineLedgerTable` | Tracking hash, category, priority, stage, owner, SLA deadline, last update, action |
| `/officer/dep_03` | `OutageSafetyQueue` | Safety flag, outage area, priority, assigned crew, SLA deadline, escalation stage, action |
| `/srcs/dispatch` | `DispatchLogTable` | Dispatched at, destination DBMS, tracking hash, escalation stage, HTTP result, retry count, last error |
| `/track/:trackingHash` | `ComplaintTimeline` | Submitted, classified, assigned, escalated, proof received, resolved/closed |

`DepartmentKanban` is the only drag-and-drop view. Dragging a card must call `PATCH /complaints/:id/status`, preserve the server response as the source of truth, and show an error toast if the update fails. The other department views are tables because they need sorting, pagination, and auditability.

### 10.3 Audit and dispatch log example

```tsx
const auditColumns: DataTableColumn<AuditEvent>[] = [
  { key: 'createdAt', header: 'Time', render: (row) => formatDate(row.createdAt), sortable: true },
  { key: 'trackingHash', header: 'Tracking hash', render: (row) => row.trackingHash },
  { key: 'actorRole', header: 'Actor', render: (row) => row.actorRole },
  { key: 'action', header: 'Action', render: (row) => row.action, sortable: true },
  { key: 'source', header: 'Source', render: (row) => row.sourceSystem },
  { key: 'result', header: 'Result', render: (row) => <StatusBadge value={row.result} /> },
  { key: 'correlationId', header: 'Correlation ID', render: (row) => row.correlationId },
];

export function AuditTrailTable({ rows }: { rows: AuditEvent[] }) {
  return (
    <DataTable
      rows={rows}
      columns={auditColumns}
      rowKey={(row) => row.correlationId}
      emptyMessage="No audit events match the selected filters."
    />
  );
}
```

### 10.4 Citizen progress timeline

`ComplaintTimeline` renders a vertical stepper on mobile and a horizontal stepper on desktop. It must mark the current stage, show completed timestamps, display escalation handoffs to State or Central DBMS, and show a plain-language error state when a complaint cannot be found. The timeline is read-only for citizens.

## 11. Visible Dashboard Layout

```tsx
export function SrcsAnalyticsPage({ snapshot }: { snapshot: AnalyticsSnapshot }) {
  return (
    <main className="space-y-6 p-4 lg:p-6">
      <header className="flex flex-wrap items-end justify-between gap-4">
        <div>
          <p className="text-sm text-slate-500">SRCS operations</p>
          <h1 className="text-2xl font-bold">Grievance analytics</h1>
        </div>
        <AnalyticsFilters />
      </header>

      <section className="grid grid-cols-2 gap-4 lg:grid-cols-4" aria-label="Summary metrics">
        <MetricCard label="Open" value={snapshot.totals.open} />
        <MetricCard label="Resolved" value={snapshot.totals.resolved} />
        <MetricCard label="SLA breached" value={snapshot.totals.breached} />
        <MetricCard label="Average resolution" value={`${snapshot.totals.averageResolutionHours}h`} />
      </section>

      <section className="grid gap-6 lg:grid-cols-2">
        <ComplaintVolumeChart data={snapshot.dailyVolume} />
        <DepartmentVelocityChart data={snapshot.departmentVelocity} />
        <SlaDistributionChart data={snapshot.slaDistribution} />
        <SlaThresholdGauge data={snapshot.slaDistribution} />
      </section>

      <ComplaintHeatmap points={snapshot.mapPoints} />
    </main>
  );
}
```

This layout makes all five analytics visuals visible without horizontal scrolling. On narrow screens, charts stack in document order and each chart retains a stable height.

## 12. Verification & Testing Checklist

- [x] **Sub-50ms Lookup Speed**: Verify Redis caching headers (`X-Cache: HIT`) for `/complaints/track/:trackingHash`.
- [x] **Client AES-256-GCM Encryption**: Confirm that grievance text is encrypted before transmission and matches Flask's `EncryptionBarrierService`.
- [x] **Gemini 2.5 Flash Structured JSON**: Test live pre-scan and ensure deterministic fallback if API latency exceeds 2.0 seconds.
- [x] **Department Schema Separation**: Validate that officers in `dep_01` cannot read or mutate complaints in `dep_02` or `dep_03`.
- [x] **SRCS 24h/36h/72h SLA State Transitions**: Test simulated Celery cron task triggers and verify State & Central DBMS audit logs.
- [x] **PRR Vision Verification**: Validate multimodal visual proof comparison with confidence score thresholds ($\ge 85\%$ for auto-pass).
- [ ] **Analytics visibility**: Render the analytics route at desktop and mobile widths; verify all five charts, filters, summary metrics, loading, empty, and error states.
- [ ] **Table visibility**: Verify audit, pipeline, outage, and dispatch tables render their required columns and become stacked rows on mobile.
- [ ] **Kanban workflow**: Verify department cards move only after a successful server response and return to their previous column on failure.
- [ ] **Heatmap privacy**: Verify map popups contain no encrypted details or personally identifiable information.
- [ ] **Citizen timeline**: Verify every SLA handoff and resolution event is visible with timestamps.


---

## 13. Visibility Hardening Patch

Apply this patch to ensure all analytics, maps, tables, queues, and related information remain visible at desktop and mobile sizes.

### 13.1 Global layout and Leaflet CSS (`src/index.css`)

```css
@import "leaflet/dist/leaflet.css";

@tailwind base;
@tailwind components;
@tailwind utilities;

html,
body,
#root {
  min-height: 100%;
  width: 100%;
}

body {
  margin: 0;
  overflow-x: hidden;
}

.leaflet-container {
  height: 100%;
  width: 100%;
  min-height: 20rem;
}
```

Do not put dashboard routes inside a parent using `h-screen overflow-hidden`, `h-0`, `max-h-0`, `hidden`, `invisible`, or `opacity-0` unless that state is deliberately controlled.

### 13.2 Visible analytics route (`src/features/analytics/SrcsAnalyticsPage.tsx`)

```tsx
import {
  ComplaintVolumeChart,
  DepartmentVelocityChart,
  SlaDistributionChart,
  SlaThresholdGauge,
  ComplaintHeatmap,
} from './components';

interface SrcsAnalyticsPageProps {
  snapshot: AnalyticsSnapshot;
}

export function SrcsAnalyticsPage({ snapshot }: SrcsAnalyticsPageProps) {
  return (
    <main className="min-h-screen w-full overflow-x-hidden bg-slate-50 p-4 text-slate-900 lg:p-6">
      <header className="mb-6 flex flex-wrap items-end justify-between gap-4">
        <div>
          <p className="text-sm text-slate-500">SRCS operations</p>
          <h1 className="text-2xl font-bold text-slate-900">Grievance analytics</h1>
        </div>
        <AnalyticsFilters />
      </header>

      <section className="mb-6 grid grid-cols-2 gap-4 lg:grid-cols-4" aria-label="Summary metrics">
        <MetricCard label="Open" value={snapshot.totals.open} />
        <MetricCard label="Resolved" value={snapshot.totals.resolved} />
        <MetricCard label="SLA breached" value={snapshot.totals.breached} />
        <MetricCard label="Average resolution" value={`${snapshot.totals.averageResolutionHours}h`} />
      </section>

      <section className="grid min-w-0 gap-6 lg:grid-cols-2">
        <div className="min-w-0 rounded-xl bg-white p-4 shadow-sm ring-1 ring-slate-200">
          <ComplaintVolumeChart data={snapshot.dailyVolume} />
        </div>
        <div className="min-w-0 rounded-xl bg-white p-4 shadow-sm ring-1 ring-slate-200">
          <DepartmentVelocityChart data={snapshot.departmentVelocity} />
        </div>
        <div className="min-w-0 rounded-xl bg-white p-4 shadow-sm ring-1 ring-slate-200">
          <SlaDistributionChart data={snapshot.slaDistribution} />
        </div>
        <div className="min-w-0 rounded-xl bg-white p-4 shadow-sm ring-1 ring-slate-200">
          <SlaThresholdGauge data={snapshot.slaDistribution} />
        </div>
      </section>

      <section className="mt-6 min-w-0 rounded-xl bg-white p-4 shadow-sm ring-1 ring-slate-200">
        <ComplaintHeatmap points={snapshot.mapPoints} />
      </section>
    </main>
  );
}
```

### 13.3 Stable Recharts pattern

Use the same outer structure for every Recharts component. `min-w-0` prevents grid overflow, while `h-72` prevents `ResponsiveContainer` from rendering with zero height.

```tsx
import {
  Area,
  AreaChart,
  CartesianGrid,
  Legend,
  ResponsiveContainer,
  Tooltip,
  XAxis,
  YAxis,
} from 'recharts';

interface ComplaintVolumeChartProps {
  data: Array<{ date: string; received: number; resolved: number }>;
}

export function ComplaintVolumeChart({ data }: ComplaintVolumeChartProps) {
  if (data.length === 0) {
    return (
      <section className="min-w-0" aria-labelledby="complaint-volume-title">
        <h2 id="complaint-volume-title" className="mb-4 text-base font-semibold">Complaint volume</h2>
        <div className="flex h-72 items-center justify-center rounded-lg border border-dashed border-slate-300 text-sm text-slate-500">
          No complaint-volume data is available for the selected filters.
        </div>
      </section>
    );
  }

  return (
    <section className="min-w-0" aria-labelledby="complaint-volume-title">
      <h2 id="complaint-volume-title" className="mb-4 text-base font-semibold">Complaint volume</h2>
      <div className="h-72 min-h-72 w-full min-w-0">
        <ResponsiveContainer width="100%" height="100%">
          <AreaChart data={data} margin={{ top: 8, right: 16, left: 0, bottom: 0 }}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="date" />
            <YAxis allowDecimals={false} />
            <Tooltip />
            <Legend />
            <Area type="monotone" dataKey="received" name="Received" stroke="#2563eb" fill="#bfdbfe" />
            <Area type="monotone" dataKey="resolved" name="Resolved" stroke="#16a34a" fill="#bbf7d0" />
          </AreaChart>
        </ResponsiveContainer>
      </div>
    </section>
  );
}
```

For `BarChart` and `PieChart`, use the exact same `div` class:

```tsx
<div className="h-72 min-h-72 w-full min-w-0">
  <ResponsiveContainer width="100%" height="100%">
    {/* BarChart or PieChart */}
  </ResponsiveContainer>
</div>
```

### 13.4 Visible heatmap (`src/features/analytics/components/ComplaintHeatmap.tsx`)

```tsx
import { MapContainer, TileLayer, CircleMarker, Popup } from 'react-leaflet';
import MarkerClusterGroup from 'react-leaflet-cluster';
import type { MapPoint } from '../../../types';

const priorityColor: Record<MapPoint['priority'], string> = {
  'P1-CRITICAL': '#dc2626',
  'P2-HIGH': '#ea580c',
  'P3-MEDIUM': '#eab308',
  'P4-LOW': '#16a34a',
};

interface ComplaintHeatmapProps {
  points: MapPoint[];
  loading?: boolean;
  error?: string;
}

export function ComplaintHeatmap({ points, loading = false, error }: ComplaintHeatmapProps) {
  if (loading) {
    return <div className="flex min-h-80 items-center justify-center text-sm text-slate-500" role="status">Loading complaint locations...</div>;
  }

  if (error) {
    return <div className="flex min-h-80 items-center justify-center rounded-lg border border-rose-200 bg-rose-50 p-6 text-sm text-rose-700" role="alert">{error}</div>;
  }

  if (points.length === 0) {
    return (
      <section className="min-w-0" aria-labelledby="complaint-map-title">
        <h2 id="complaint-map-title" className="mb-4 text-base font-semibold">Complaint heatmap</h2>
        <div className="flex min-h-80 items-center justify-center rounded-lg border border-dashed border-slate-300 text-sm text-slate-500">
          No complaint locations are available for the selected filters.
        </div>
      </section>
    );
  }

  const center: [number, number] = [points[0].latitude, points[0].longitude];

  return (
    <section className="min-w-0" aria-labelledby="complaint-map-title">
      <div className="mb-4 flex flex-wrap items-center justify-between gap-3">
        <h2 id="complaint-map-title" className="text-base font-semibold">Complaint heatmap</h2>
        <p className="text-xs text-slate-500">Markers show priority; popups exclude encrypted details and personal data.</p>
      </div>

      <div className="relative h-96 min-h-80 w-full overflow-hidden rounded-lg border border-slate-200">
        <MapContainer center={center} zoom={12} className="z-0 h-full w-full" style={{ height: '100%', width: '100%' }}>
          <TileLayer
            attribution='&copy; OpenStreetMap contributors'
            url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png"
          />
          <MarkerClusterGroup chunkedLoading>
            {points.map((point) => (
              <CircleMarker
                key={point.id}
                center={[point.latitude, point.longitude]}
                radius={8}
                pathOptions={{ color: priorityColor[point.priority], fillOpacity: 0.75 }}
              >
                <Popup>
                  <div className="space-y-1 text-sm">
                    <p><strong>Tracking hash:</strong> {point.trackingHash}</p>
                    <p><strong>Priority:</strong> {point.priority}</p>
                    <p><strong>Department:</strong> {point.department}</p>
                    <p><strong>Status:</strong> {point.status}</p>
                    <p><strong>Created:</strong> {new Date(point.createdAt).toLocaleString()}</p>
                  </div>
                </Popup>
              </CircleMarker>
            ))}
          </MarkerClusterGroup>
        </MapContainer>
      </div>
    </section>
  );
}
```

### 13.5 Fully visible reusable table (`src/components/DataTable.tsx`)

```tsx
import React, { useMemo, useState } from 'react';

export interface DataTableColumn<Row> {
  key: string;
  header: string;
  render: (row: Row) => React.ReactNode;
  sortable?: boolean;
  sortValue?: (row: Row) => string | number;
}

interface DataTableProps<Row> {
  rows: Row[];
  columns: DataTableColumn<Row>[];
  rowKey: (row: Row) => string;
  loading?: boolean;
  emptyMessage: string;
}

export function DataTable<Row>({
  rows,
  columns,
  rowKey,
  loading = false,
  errorMessage,
  emptyMessage,
  search,
  onSearchChange,
  filterControls,
  page,
  pageCount,
  onPageChange,
}: DataTableProps<Row>) {
  const [sortKey, setSortKey] = useState<string | null>(null);
  const [direction, setDirection] = useState<'asc' | 'desc'>('asc');

  const sortedRows = useMemo(() => {
    if (!sortKey) return rows;
    const column = columns.find((item) => item.key === sortKey);
    if (!column?.sortValue) return rows;

    return [...rows].sort((a, b) => {
      const left = column.sortValue!(a);
      const right = column.sortValue!(b);
      const result = typeof left === 'number' && typeof right === 'number'
        ? left - right
        : String(left).localeCompare(String(right));
      return direction === 'asc' ? result : -result;
    });
  }, [rows, columns, sortKey, direction]);

  const sortBy = (column: DataTableColumn<Row>) => {
    if (!column.sortable || !column.sortValue) return;
    if (sortKey === column.key) {
      setDirection((value) => (value === 'asc' ? 'desc' : 'asc'));
      return;
    }
    setSortKey(column.key);
    setDirection('asc');
  };

  if (loading) {
    return (
      <div className="flex min-h-40 items-center justify-center rounded-xl border border-slate-200 bg-white text-sm text-slate-500" role="status">
        Loading records...
      </div>
    );
  }

  if (errorMessage) {
    return (
      <div className="flex min-h-40 items-center justify-center rounded-xl border border-rose-200 bg-rose-50 p-6 text-center text-sm text-rose-700" role="alert">
        {errorMessage}
      </div>
    );
  }

  if (rows.length === 0) {
    return (
      <div className="flex min-h-40 items-center justify-center rounded-xl border border-dashed border-slate-300 bg-white p-6 text-center text-sm text-slate-500">
        {emptyMessage}
      </div>
    );
  }

  return (
    <section className="space-y-3">
      {(onSearchChange || filterControls) && (
        <div className="flex flex-wrap items-center gap-3">
          {onSearchChange && (
            <label className="flex min-w-56 flex-1 items-center gap-2 text-sm text-slate-700">
              <span className="sr-only">Search records</span>
              <input
                value={search ?? ''}
                onChange={(event) => onSearchChange(event.target.value)}
                placeholder="Search records"
                className="w-full rounded-lg border border-slate-300 px-3 py-2 outline-none focus-visible:ring-2 focus-visible:ring-blue-600"
              />
            </label>
          )}
          {filterControls}
        </div>
      )}

      <div className="hidden w-full overflow-x-auto rounded-xl border border-slate-200 bg-white shadow-sm md:block">
      <table className="w-full min-w-[900px] border-collapse text-left text-sm">
        <thead className="bg-slate-100 text-slate-700">
          <tr>
            {columns.map((column) => (
              <th key={column.key} scope="col" className="whitespace-nowrap px-4 py-3 font-semibold">
                {column.sortable && column.sortValue ? (
                  <button
                    type="button"
                    onClick={() => sortBy(column)}
                    className="inline-flex items-center gap-1 rounded outline-none hover:text-blue-700 focus-visible:ring-2 focus-visible:ring-blue-600"
                    aria-sort={sortKey === column.key ? (direction === 'asc' ? 'ascending' : 'descending') : 'none'}
                  >
                    {column.header}
                    <span aria-hidden="true">{sortKey === column.key ? (direction === 'asc' ? '▲' : '▼') : '↕'}</span>
                  </button>
                ) : column.header}
              </th>
            ))}
          </tr>
        </thead>
        <tbody className="divide-y divide-slate-200">
          {sortedRows.map((row) => (
            <tr key={rowKey(row)} className="hover:bg-slate-50 focus-within:bg-blue-50">
              {columns.map((column) => (
                <td key={column.key} className="whitespace-nowrap px-4 py-3 text-slate-700">
                  {column.render(row)}
                </td>
              ))}
            </tr>
          ))}
        </tbody>
      </table>
      </div>

      <div className="space-y-3 md:hidden">
        {sortedRows.map((row) => (
          <article key={rowKey(row)} className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
            {columns.map((column) => (
              <div key={column.key} className="flex items-start justify-between gap-4 border-b border-slate-100 py-2 last:border-b-0">
                <span className="shrink-0 text-xs font-semibold uppercase tracking-wide text-slate-500">{column.header}</span>
                <span className="min-w-0 break-words text-right text-sm text-slate-800">{column.render(row)}</span>
              </div>
            ))}
          </article>
        ))}
      </div>

      {page && pageCount && pageCount > 1 && onPageChange && (
        <nav className="flex items-center justify-between text-sm" aria-label="Table pagination">
          <button type="button" disabled={page <= 1} onClick={() => onPageChange(page - 1)} className="rounded px-3 py-2 focus-visible:ring-2 focus-visible:ring-blue-600 disabled:opacity-40">Previous</button>
          <span>Page {page} of {pageCount}</span>
          <button type="button" disabled={page >= pageCount} onClick={() => onPageChange(page + 1)} className="rounded px-3 py-2 focus-visible:ring-2 focus-visible:ring-blue-600 disabled:opacity-40">Next</button>
        </nav>
      )}
    </section>
  );
}
```

The `overflow-x-auto` wrapper retains all columns, and the `min-w-[900px]` table width ensures long audit, dispatch, and pipeline tables do not collapse or hide fields.

### 13.6 Mobile stacked table behavior

`DataTable` now renders the same rows as stacked records below the `md` breakpoint while retaining the sortable desktop table above it. Use the following pattern only for custom table layouts that do not use `DataTable`:

```tsx
<div className="hidden md:block">
  <DataTable rows={rows} columns={columns} rowKey={(row) => row.id} emptyMessage="No records found." />
</div>

<div className="space-y-3 md:hidden">
  {rows.map((row) => (
    <article key={row.id} className="rounded-xl border border-slate-200 bg-white p-4 shadow-sm">
      {columns.map((column) => (
        <div key={column.key} className="flex items-start justify-between gap-4 border-b border-slate-100 py-2 last:border-b-0">
          <span className="shrink-0 text-xs font-semibold uppercase tracking-wide text-slate-500">{column.header}</span>
          <span className="min-w-0 break-words text-right text-sm text-slate-800">{column.render(row)}</span>
        </div>
      ))}
    </article>
  ))}
</div>
```

### 13.7 Required checks

- Every `ResponsiveContainer` has a parent with `h-72 min-h-72 w-full min-w-0`.
- Every Leaflet map has a `h-96 min-h-80 w-full` parent and Leaflet CSS is loaded.
- Every wide table is inside `w-full overflow-x-auto` and has a `min-w-*` width.
- Every data section renders a visible loading, empty, and error state.
- No dashboard ancestor clips content with `overflow-hidden` or limits its height.
- Dark containers always specify readable text colors such as `text-white` and `text-slate-300`.
