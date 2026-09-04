# Frontend State Machine & Error Recovery Matrix



## Architecture Overview

| Scenario ID | Trigger Condition | Interceptor / Handling Mechanism | UI State & Action | Final Resolution |
| :--- | :--- | :--- | :--- | :--- |
| **Scenario 1** | Form Submission (Network OK) | Direct API Call (`POST`) | Displays **Step Progress Spinner** | `201 Created` → Token saved to `LocalStorage` → Redirects to `/submit/success` |
| **Scenario 2** | Session Expired | UI Interceptor (`401 Unauthorized`) | Freezes form state in `DraftStore` & opens **Re-Auth Modal** | Auto-retries pending API call upon successful authentication |
| **Scenario 3** | High Traffic | Rate Limiter (`429 Rate Limit`) | Displays *'System busy, retrying...'* Toast | Executes Exponential Backoff (1s, 2s, 4s) to retry request seamlessly |
| **Scenario 4** | Connection Loss | Network Status Listener | Switches to **Offline Indicator** & saves draft to `IndexedDB` | Prompts citizen *'Resume previous complaint?'* when back online |
| **Scenario 5** | PRR Vision Mismatch | Automated Confidence Gate (<70%) | Flags *'AI detected proof discrepancy'* on Officer UI | Requires **Officer Supervisor Manual Override Signature** to proceed |

---

## Scenario Details & Technical Execution

### 1. Citizen Form Submission (Happy Path)
* **Trigger:** Citizen completes and submits the complaint form under normal network conditions.
* **Flow:**
  1. UI shifts to pending state and displays a **Step Progress Spinner**.
  2. Sends payload via `POST /complaints/submit`.
  3. Receives `201 Created` HTTP response.
  4. Stores the generated transaction token in `LocalStorage`.
  5. Redirects the user to the confirmation page at `/submit/success`.

---

### 2. Officer Session Expiry Handling (`401 Unauthorized`)
* **Trigger:** An officer attempts an action while their authentication session token has expired or invalidated.
* **Flow:**
  1. The UI HTTP Interceptor captures the `401 Unauthorized` response.
  2. The current form state is instantly frozen and persisted to `DraftStore` to prevent data loss.
  3. A **Re-Authentication Modal** is launched over the current view.
  4. Upon successful re-authentication, the interceptor automatically retries the initial pending API call.

---

### 3. Server Rate Limiting & Backoff (`429 Too Many Requests`)
* **Trigger:** High server traffic results in rate limiting from API endpoints.
* **Flow:**
  1. The UI captures the `429 Rate Limit` response.
  2. A toast notification is displayed: *"System busy, retrying..."*.
  3. The request retry pipeline applies **Exponential Backoff** delays ($1	ext{s} 	o 2	ext{s} 	o 4	ext{s}$).
  4. The request retries automatically in the background without requiring user action.

---

### 4. Network Disconnection & Offline Storage
* **Trigger:** Client device loses internet connectivity during input or submission.
* **Flow:**
  1. UI switches to the **Offline Indicator** state.
  2. The current complaint draft is serialized and stored locally in **IndexedDB**.
  3. Once network connectivity is restored, the application prompts the citizen: *"Resume previous complaint?"*.
  4. Selecting resume restores the saved form state directly from IndexedDB.

---

### 5. Vision AI Verification Mismatch & Manual Override
* **Trigger:** PRR Vision Check calculates a proof verification confidence score lower than 70%.
* **Flow:**
  1. Officer UI captures the low-confidence flag from the backend.
  2. The UI highlights an explicit warning: *"AI detected proof discrepancy"*.
  3. Submission or state transition is blocked until an **Officer Supervisor Manual Override Signature** is attached.