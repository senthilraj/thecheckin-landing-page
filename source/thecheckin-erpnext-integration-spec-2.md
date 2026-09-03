# TheCheckIn ↔ ERPNext Integration — Technical Spec

**Purpose:** Push face-recognition attendance events from TheCheckIn into ERPNext's `Employee Checkin` doctype in real time, so ERPNext's built-in auto-attendance and payroll pick them up automatically.

**Two codebases, two licenses:**
| | Repo | Language | License | Hosting |
|---|---|---|---|---|
| A | `thecheckin_connector` (new) | Python (Frappe app) | Public — MIT | Runs inside customer's ERPNext instance |
| B | TheCheckIn backend (existing) | Node.js | Private | Your AWS EC2 |

**Source-of-truth rule (core design decision):**
Once a tenant enables this integration, **ERPNext is authoritative for Employees and Branches**. TheCheckIn stops being editable for that org data — no create, edit, or delete of employees/branches inside the TheCheckIn app. All changes must happen in ERPNext and sync in.

**One deliberate exception:** face enrollment. ERPNext has no equivalent field for a biometric-grade photo/consent capture, so **adding or updating a face** stays a TheCheckIn-app action, per employee, always — regardless of integration status.

---

## 1. Data flow (both directions)

```
ERPNext (source of truth: Employee, Branch)
        │
        │  doc-event hooks + "Sync Now" button
        ▼
thecheckin_connector (Frappe app, Repo A)
        │
        │  outbound sync, HMAC-signed
        ▼
POST /api/erpnext-integration/sync-employee
POST /api/erpnext-integration/sync-branch
        │
        ▼
TheCheckIn backend (Node.js, Repo B) — upserts Employee/Branch,
stores erpnext_employee_id / erpnext_branch_id as the link key
        │
        ▼
Customer adds face via TheCheckIn app (one-time, per employee)
        │
        ▼
Face scan at check-in → attendance event
        │
        │  outbound webhook, HMAC-signed
        ▼
POST /api/method/thecheckin_connector.api.receive_checkin
   (payload carries erpnext_employee_id directly — no matching needed,
    since the employee was created via the sync above)
        │
        ▼
thecheckin_connector creates Employee Checkin
        │
        ▼
ERPNext auto-attendance (native) → Attendance record → Payroll
```

Two directions, two different authoritative systems, no field is ever synced both ways — this avoids conflict-resolution logic entirely.

---

## 2. Repo A — `thecheckin_connector` (Frappe app)

### 2.1 Scaffold
```
bench new-app thecheckin_connector
bench get-app thecheckin_connector
bench --site {site} install-app thecheckin_connector
```
Target: current stable ERPNext/Frappe version (v15 at time of writing — confirm against Frappe's stable branch when build starts).

### 2.2 DocType: `TheCheckIn Settings` (Single)

| Field | Type | Notes |
|---|---|---|
| `enabled` | Check | Master on/off switch |
| `api_key` | Data | Public identifier issued by TheCheckIn per tenant |
| `webhook_secret` | Password | Shared secret, used for HMAC signing in both directions |
| `default_log_source` | Data (read-only) | Fixed value `"TheCheckIn"`, tags every created record for traceability |
| `last_org_sync_at` | Datetime (read-only) | Updated after each Employee/Branch push |
| `last_checkin_sync_at` | Datetime (read-only) | Updated after each attendance event received |

*(Removed from the original draft: `employee_match_field` / `custom_match_fieldname`. No longer needed — see §2.5.)*

### 2.3 DocType: `TheCheckIn Sync Log`

Tracks **both** directions in one log, distinguished by `sync_type`.

| Field | Type | Notes |
|---|---|---|
| `sync_type` | Select: Employee Out / Branch Out / Checkin In | |
| `status` | Select: Success / Failed / Skipped | |
| `reference_doctype` | Data | e.g. `Employee`, `Branch`, `Employee Checkin` |
| `reference_name` | Data | The doctype's `name`/ID involved |
| `error_message` | Small Text | Populated on Failed/Skipped |
| `raw_payload` | Code (JSON) | Full payload sent or received, for debugging |
| `timestamp` | Datetime | |

Purpose: this is what turns silent integration failures into something the customer's admin can actually see and act on — critical for support load, in both directions now.

### 2.4 Attendance ingestion endpoint

**Route:** `POST /api/method/thecheckin_connector.api.receive_checkin`

**Headers:**
```
X-TheCheckIn-Api-Key: {api_key}
X-TheCheckIn-Signature: {HMAC-SHA256 of raw body, using webhook_secret}
X-TheCheckIn-Timestamp: {unix epoch, request send time}
Content-Type: application/json
```

**Request body:**
```json
{
  "erpnext_employee_id": "HR-EMP-00042",
  "log_type": "IN",
  "timestamp": "2026-08-07T09:14:32+00:00",
  "device_id": "kiosk-branch-02",
  "erpnext_branch_id": "Dubai Marina",
  "latitude": 25.0805,
  "longitude": 55.1403,
  "verification_method": "face_recognition",
  "confidence_score": 0.97
}
```

> **Design simplification vs. the earlier draft:** the payload now carries `erpnext_employee_id` directly — the exact ERPNext `Employee` record name. No fuzzy matching, no email fallback needed. This only works *because* §2.5 guarantees the employee already exists in ERPNext (it's the source, after all) and TheCheckIn stored that exact ID when the employee was synced in. Matching logic is only a concern if an employee somehow reaches TheCheckIn without going through the sync — which the source-of-truth rule prevents.

**Server-side validation, in order:**
1. `enabled` is true in Settings → else `403`
2. Timestamp within ±5 minutes of server time → else `401` (replay protection)
3. Signature matches HMAC-SHA256(raw_body, webhook_secret) → else `401`
4. `log_type` is one of `IN` / `OUT` → else `400`
5. `erpnext_employee_id` resolves to an existing, active `Employee` → else log to Sync Log as `Failed`, return `422`
6. Duplicate check: same `erpnext_employee_id` + `timestamp` already logged → treat as `Skipped`, return `200` (idempotency — safe for Repo B to retry on network failure)

**Success response:**
```json
{ "status": "success", "employee_checkin": "EMP-CKIN-2026-00042" }
```

**Failure response:**
```json
{ "status": "error", "reason": "employee_not_found", "erpnext_employee_id": "HR-EMP-00042" }
```

### 2.5 Employee & Branch sync — ERPNext → TheCheckIn (source of truth)

**Direction: outbound only, ERPNext → TheCheckIn.** This is what makes TheCheckIn's employee/branch data read-only once integration is enabled — there is no path for TheCheckIn to originate a change to these records; it only receives them.

**Triggers (`hooks.py` doc_events):**
```python
doc_events = {
    "Employee": {
        "after_insert": "thecheckin_connector.sync.push_employee",
        "on_update": "thecheckin_connector.sync.push_employee",
    },
    "Branch": {
        "after_insert": "thecheckin_connector.sync.push_branch",
        "on_update": "thecheckin_connector.sync.push_branch",
    },
}
```
Near-real-time by default — no waiting for a scheduled job on ordinary changes.

**Initial/backfill sync:** a **"Sync Now" button** in Settings, for onboarding an existing ERPNext org with pre-existing employees. Iterates all active `Employee` and `Branch` records and pushes each through the same functions above.

**Outbound payload — Employee** (`POST /api/erpnext-integration/sync-employee` on Repo B):
```json
{
  "erpnext_employee_id": "HR-EMP-00042",
  "full_name": "Jane Doe",
  "email": "jane@company.com",
  "phone": "+971501234567",
  "branch": "Dubai Marina",
  "designation": "Waitstaff",
  "status": "Active"
}
```

**Outbound payload — Branch** (`POST /api/erpnext-integration/sync-branch` on Repo B):
```json
{
  "erpnext_branch_id": "Dubai Marina",
  "branch_name": "Dubai Marina",
  "address": "JBR Walk, Dubai, UAE"
}
```

**Upsert logic on Repo B:** keyed strictly by `erpnext_employee_id` / `erpnext_branch_id` — if that ID already exists for the tenant, update; otherwise create.

**First-sync reconciliation (existing TheCheckIn customers only):** if a tenant already has manually-created employees in TheCheckIn *before* turning on this integration, the very first sync should attempt an email match before creating a duplicate:
- Match found → link the existing TheCheckIn employee to `erpnext_employee_id`, keep their attendance history intact
- No match → create fresh

After this one-time reconciliation, the tenant's `source_of_truth` flag flips to `erpnext` (see §3.4) and no further manual create/edit is permitted in TheCheckIn for that tenant — new employees only arrive via sync from that point on.

**Deactivation, not deletion:** if `Employee.status` changes to `Left` in ERPNext, sync pushes that status change; Repo B **deactivates** the corresponding TheCheckIn employee rather than deleting them, preserving historical attendance records.

### 2.6 Security notes
- `webhook_secret` configured once during setup, same value held by TheCheckIn backend per-tenant — never re-transmitted afterward
- Rate limit: reject if a single `api_key` sends >120 requests/minute (generous for real-world check-in bursts, blocks abuse)
- No PII beyond what's in the payloads above — no raw face image or biometric template is ever sent to or stored in ERPNext

---

## 3. Repo B — additions to existing TheCheckIn backend (Node.js)

### 3.1 New table: `erpnext_integrations`

| Column | Type | Notes |
|---|---|---|
| `tenant_id` | FK → your existing tenant/company table | |
| `erpnext_site_url` | varchar | e.g. `https://acme.frappe.cloud` |
| `webhook_secret` | varchar (encrypted at rest) | Shared secret, same value set in connector's Settings |
| `source_of_truth` | enum: `thecheckin` / `erpnext` | Gates whether Employee/Branch edits are allowed in TheCheckIn UI — see §3.4 |
| `enabled` | boolean | |
| `created_at` / `updated_at` | timestamp | |

### 3.2 Attendance webhook dispatcher (TheCheckIn → ERPNext)

**Trigger point:** wherever an attendance event is currently finalized (post face-match, post GPS validation) in your existing check-in flow.

**Logic:**
1. Look up `erpnext_integrations` for the tenant; skip if disabled/not configured
2. Build payload per §2.4, using the `erpnext_employee_id` already stored on the employee record (from the sync in §3.3)
3. Sign: `HMAC-SHA256(JSON.stringify(payload), webhook_secret)`
4. POST to `{erpnext_site_url}/api/method/thecheckin_connector.api.receive_checkin`
5. On non-2xx or timeout: retry with exponential backoff (e.g. 1m / 5m / 30m), max 3 attempts
6. Log every attempt to a `webhook_delivery_log` table (status, response code, attempt number)

### 3.3 New endpoints: org sync receivers (ERPNext → TheCheckIn)

**`POST /api/erpnext-integration/sync-employee`**
Auth: same `api_key` + signature scheme as §2.4, reversed direction (connector signs, Repo B verifies).
- Upserts by `erpnext_employee_id`
- On first sync for a tenant, runs the email-reconciliation step described in §2.5
- On `status: "Left"` → deactivates rather than deletes

**`POST /api/erpnext-integration/sync-branch`**
Same auth pattern. Upserts by `erpnext_branch_id`.

### 3.4 Enforcing read-only Employee/Branch in the TheCheckIn app

This has to be enforced **server-side**, not just hidden in the UI — otherwise a direct API call could still create drift.

- Every existing Employee/Branch create/update/delete endpoint in Repo B checks `erpnext_integrations.source_of_truth` for the requesting tenant
- If `source_of_truth = erpnext`: reject with a clear error (e.g. `"Managed by ERPNext — edit there and it will sync automatically"`), for every field **except** the face-enrollment action
- Face enrollment (photo capture/update) is explicitly exempted from this check at the route level — it remains fully editable regardless of `source_of_truth`

**UI implication (frontend, not in this spec's scope but worth flagging to whoever builds it):** Employee/Branch list and detail screens should show a "Managed by ERPNext" banner and grey out edit fields for integrated tenants, with only an "Add/Update Face" action left interactive per employee.

---

## 4. Open decisions before build starts

1. **Confirm push vs. pull for attendance events** — spec above assumes push (recommended, matches "instant verification" positioning).
2. **Setup order** — suggest: TheCheckIn admin panel generates `webhook_secret` first, customer pastes it into the connector's ERPNext Settings during onboarding, then triggers "Sync Now."
3. **Multi-branch handling** — confirm whether `erpnext_branch_id` needs to map to anything else in TheCheckIn's own schema, or is used as-is.
4. **Frappe/ERPNext version target** — confirm v15 vs. also supporting v14 for older self-hosted customers.
5. **Existing customers adopting the integration later** — confirm the email-reconciliation approach in §2.5 is acceptable, or whether you'd rather flag ambiguous matches for manual review instead of auto-linking.

## 5. Suggested build order
1. Repo A skeleton + `TheCheckIn Settings` / `TheCheckIn Sync Log` DocTypes (no logic yet)
2. Repo A: Employee/Branch outbound sync (§2.5) — confirm doc-event hooks fire and payloads are correct
3. Repo B: sync-receiver endpoints (§3.3) with upsert + first-sync reconciliation
4. Repo B: read-only enforcement (§3.4) on existing Employee/Branch routes
5. Face enrollment flow in TheCheckIn app, confirmed working per newly-synced employee
6. Repo A: `receive_checkin` endpoint (§2.4), now using direct `erpnext_employee_id` (no matching logic needed)
7. Repo B: attendance webhook dispatcher (§3.2), pointed at a local/dev ERPNext instance
8. End-to-end test: employee created in ERPNext → appears in TheCheckIn within seconds → face enrolled → face scan check-in → Employee Checkin appears in ERPNext within seconds
9. Error-path testing: bad signature, unknown employee, duplicate event, disabled integration, employee deactivated mid-flow
