# BeyondCorp Resource Request Workflow

> **Scenario:** Engineer on a managed laptop requests access to `https://code.corp.example.com` (internal Git repository).

---

## Phase 1 — Device Bootstrapping
*Occurs once during device enrollment — before any resource request.*

1. Device is enrolled in MDM (Jamf / Intune / internal PKI scripts).
2. Internal CA issues a **device certificate**:
   - `Subject: CN=laptop-001, OU=engineering`
   - Extensions: serial number, trust tier embedded in SAN or custom OID
   - Private key stored in TPM — **not exportable**
3. Device Inventory DB is updated:
   ```
   serial:          laptop-001
   owner:           john@corp
   os_patched:      true
   disk_encrypted:  true
   trust_tier:      2
   ```
4. `osquery` agent starts running — continuously reports device posture back to Fleet.

---

## Phase 1b — Continuous Posture Reporting (osquery → Fleet)
*Runs persistently in the background on every managed device — independent of any user request.*

### How osquery Works

osquery exposes the entire OS state as queryable SQL tables. Fleet schedules queries against these tables and collects results centrally.

```sql
-- Is disk encrypted?
SELECT encrypted FROM disk_encryption;

-- Is EDR agent running?
SELECT * FROM processes WHERE name = 'falcon-sensor';

-- OS patch level
SELECT version, build FROM os_version;

-- Firewall status (macOS)
SELECT global_state FROM alf;

-- Screensaver lock enforced?
SELECT value FROM preferences WHERE domain = 'com.apple.screensaver' AND key = 'askForPassword';
```

### Two Update Modes

**1. Scheduled polling** — Fleet pushes query packs to agents; agents run them on a fixed interval and return diffs:

```json
{
  "schedule": {
    "disk_encryption": {
      "query": "SELECT encrypted FROM disk_encryption;",
      "interval": 300
    },
    "edr_running": {
      "query": "SELECT pid FROM processes WHERE name = 'falcon-sensor';",
      "interval": 60
    },
    "os_version": {
      "query": "SELECT version FROM os_version;",
      "interval": 3600
    },
    "screensaver_lock": {
      "query": "SELECT value FROM preferences WHERE domain='com.apple.screensaver' AND key='askForPassword';",
      "interval": 600
    }
  }
}
```

**2. Event-driven (kernel-level, immediate)** — osquery subscribes to OS event streams and pushes on change without waiting for the next poll interval:

```
Kernel event fires (file write, process start, network socket open)
  → osquery event subscriber captures immediately
  → diff computed against last known state
  → pushed to Fleet over TLS
```

Used for high-sensitivity events: sensitive file modified, new SUID binary, unexpected outbound connection.

### osquery → Fleet Pipeline

```
Device (laptop-001)
  └─ osquery agent
       ├─ scheduled queries  (every 60s – 1hr depending on check)
       └─ event subscribers  (immediate on kernel event)
            │
            │  TLS-encrypted POST
            │  osquery TLS enrollment + node key auth
            ▼
         Fleet (Device Inventory Service)
            ├─ stores latest posture snapshot per device
            ├─ evaluates compliance policies:
            │    disk_encrypted = true         → pass
            │    os_build >= required_version  → pass
            │    edr_running = true            → pass
            │    screensaver_lock = true       → pass
            │    firewall_enabled = true       → pass
            │
            ├─ computes trust tier:
            │    all checks pass  → trust_tier: 2
            │    any check fails  → trust_tier: 1 (or 0)
            │
            └─ exposes REST API for OPA:
                 GET /api/v1/fleet/hosts/laptop-001
                 → { "trust_tier": 2, "compliant": true, ... }
```

### Trust Tier Compliance Matrix

| Posture Check | Required for Tier 1 | Required for Tier 2 |
|---|:---:|:---:|
| Device cert issued by internal CA | ✓ | ✓ |
| Disk encryption enabled | | ✓ |
| OS on supported + patched build | | ✓ |
| EDR agent running | | ✓ |
| Screensaver lock ≤ 5 min | | ✓ |
| Firewall enabled | | ✓ |
| No unauthorized SUID binaries | | ✓ |

### Propagation Lag and Trust Window

The push model introduces a small window where a posture change hasn't reached OPA yet:

```
Disk encryption disabled at T+0
  → osquery detects at next poll: T+300 (5 min interval)
  → Fleet updates trust_tier: T+301
  → OPA reads new tier on next authz call: T+301+

During T+0 → T+301: stale tier 2 may still be returned → access still granted
```

**Mitigations:**
- Shorten polling interval for high-sensitivity checks (e.g. EDR: 60s)
- Use event-driven subscribers for critical checks (eliminates lag for those checks)
- Critical resources can call Fleet API inline during authz (real-time, higher latency)
- Session expiry forces re-evaluation periodically regardless

---

## Phase 2 — User Authentication (SSO)
*Occurs once per session.*

5. User opens browser and navigates to `code.corp.example.com`.
6. Browser hits **Pomerium** (Network PEP) — no session cookie present yet.
7. Pomerium redirects to **Keycloak** (IdP) at `/authorize`.
8. User authenticates: `username` + `password` + `TOTP` (Vaultwarden / YubiKey).
9. Keycloak issues tokens:
   - **ID Token** (JWT): `sub`, `email`, `groups: ["engineering"]`
   - **Access Token**
10. Pomerium validates token signature against Keycloak JWKS endpoint.
11. Pomerium stores an encrypted session cookie in the browser.

---

## Phase 3 — Per-Request Authorization
*Runs on **every single request** — the core BeyondCorp enforcement loop.*

### Step 12 — Browser sends request

```http
GET /repo/project HTTP/1.1
Host: code.corp.example.com
Cookie: _pomerium=<encrypted session>

TLS ClientCertificate: laptop-001.crt   ← mTLS device certificate
```

### Step 13 — Pomerium (PEP) processes the request

- Decrypts session cookie → extracts user identity
- Validates device certificate in mTLS handshake:
  - Is the cert valid and unexpired?
  - Is it signed by the internal CA?
  - Extracts `CN: laptop-001`

### Step 14 — Pomerium calls OPA (PDP)

```http
POST http://opa:8181/v1/data/authz/allow
Content-Type: application/json

{
  "input": {
    "user": {
      "email": "john@corp",
      "groups": ["engineering"]
    },
    "device": {
      "serial": "laptop-001",
      "cert_valid": true,
      "trust_tier": 2
    },
    "resource": "code-repo",
    "action": "GET",
    "path": "/repo/project"
  }
}
```

> OPA fetches `trust_tier` dynamically from Fleet / Device Inventory at evaluation time.

### Step 15 — OPA evaluates Rego policy

```rego
allow {
  input.user.groups[_] == "engineering"
  input.device.cert_valid == true
  input.device.trust_tier >= 2
  input.resource == "code-repo"
}
```

Returns:
```json
{ "result": true }
```

### Step 16 — Decision branching

| Result | Action |
|--------|--------|
| `true` | Pomerium proxies request to backend |
| `false` | Pomerium returns `HTTP 403`, logs denial event |

### Step 17 — Pomerium proxies to backend (Application PEP layer)

```http
GET /repo/project HTTP/1.1
Host: code-internal.svc
X-Forwarded-User:   john@corp
X-Forwarded-Groups: engineering
X-Device-Serial:    laptop-001
X-Trust-Tier:       2
```

> mTLS between Pomerium and the backend service is handled by an Istio sidecar.

### Step 18 — Backend Application PEP

- Reads `X-Forwarded-User` header
- Optionally calls OPA again for fine-grained resource ACL:
  *"Can `john@corp` READ `/repo/project` specifically?"*
- Serves response or returns `HTTP 403`

### Step 19 — Response returned to client

```
Backend → Pomerium → Browser
```

> Pomerium **strips all internal headers** (`X-Forwarded-*`, `X-Device-*`) before returning the response to the client.

---

## Phase 4 — Continuous Posture Re-evaluation
*What distinguishes BeyondCorp from plain SSO.*

20. `osquery` on the device detects a posture change:
    - Disk encryption disabled
    - OS patch missing
    - EDR agent stopped

21. Fleet updates Device Inventory DB:
    ```
    laptop-001 → trust_tier: 1   (downgraded from 2)
    ```

22. On the next request, OPA evaluates:
    ```
    input.device.trust_tier = 1  →  policy requires >= 2  →  DENY
    ```

23. Access is silently blocked or the session is invalidated. The user must remediate the device to regain access.

---

## Full Request Flow

```
  Device (background)         Browser           Pomerium (PEP)      OPA (PDP)      Fleet (Inventory)     Backend
         |                       |                     |                 |                  |                  |
[osquery scheduled poll]         |                     |                 |                  |                  |
         |-- posture data --------------------------------(TLS POST)-------------------->  |                  |
         |                       |                     |                 |                  |                  |
         |               [trust_tier updated to 2 in Fleet DB]          |                  |                  |
         |                       |                     |                 |                  |                  |
[kernel event fires]             |                     |                 |                  |                  |
         |-- immediate push ----------------------------------(TLS)-------------------->   |                  |
         |                       |                     |                 |                  |                  |
         |              [user navigates to code.corp.example.com]        |                  |                  |
         |                       |-- GET /repo (mTLS cert) ----------> |                  |                  |
         |                       |                     |-- validate cert |                  |                  |
         |                       |                     |-- POST /authz ->|                  |                  |
         |                       |                     |                 |-- GET /hosts/laptop-001 ---------->|
         |                       |                     |                 |<-- { trust_tier: 2, compliant: true }|
         |                       |                     |                 |-- eval Rego     |                  |
         |                       |                     |<-- allow: true -|                  |                  |
         |                       |                     |-- proxy + inject headers -------------------------------->|
         |                       |                     |                 |                  |    app PEP check |
         |                       |<------- 200 OK ----------------------------------------------------------------|
```

---

## Protocol Summary

| Leg | Protocol |
|-----|----------|
| osquery agent → Fleet | HTTPS (osquery TLS protocol, node key auth) |
| Fleet → Device Inventory DB | Internal SQL (PostgreSQL) |
| User → PEP | HTTPS + mTLS (device certificate) |
| PEP → IdP | OIDC / OAuth2 |
| PEP → PDP | HTTP REST (OPA) or gRPC |
| PDP → Fleet (Device Inventory) | REST API |
| PEP → Backend | HTTPS + mTLS (service certificate) |
| Service → Service | mTLS via Istio / Envoy sidecar |

---

## Key Security Guarantee

The **device certificate in the mTLS handshake** cryptographically binds every request to a specific managed device. A stolen session cookie alone is insufficient — the attacker also needs the hardware-bound private key stored in the device TPM.

This is the fundamental security property that separates BeyondCorp from VPN-based or plain SSO-based access models.

---

## OSS Component Mapping

| BeyondCorp Component | OSS Implementation |
|---|---|
| Network PEP (Access Proxy) | Pomerium / Envoy + ext_authz |
| PDP (Access Control Engine) | Open Policy Agent (OPA) |
| Trust Inferer | osquery (scheduled + event-driven queries) |
| Device Inventory / Posture Store | Fleet (osquery manager) |
| Certificate Authority | step-ca / CFSSL |
| Identity Provider | Keycloak |
| Service Mesh (App PEP) | Istio / Envoy sidecars |
| Policy Language | Rego (OPA) |