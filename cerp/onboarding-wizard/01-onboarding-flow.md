# Flow 1: Onboarding Wizard (Steps 100-700)

**Duration:** ~30-45 minutes  
**User Role:** New micro-retailer  
**Trigger:** First login to ERP Cloud with no existing Wawi connection  
**Feature Flag:** "Start with new instance" (gated)  

---

## 📐 User Journey Map

```
┌─────────────────────────────────────────────────────────────┐
│                    ONBOARDING FLOW                          │
│              (User Input: Steps 100-500)                    │
│         (Automated Setup: Steps 600-700)                    │
└─────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ LOGIN & REDIRECTION                                          │
│ • User logs in via JTL-Hub                                  │
│ • Redirected to erp.jtl-cloud.com                           │
│ • ERP checks: isOnboarded = false                           │
│ → Redirect to Step 100 wizard                               │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 100: INSTANCE SELECTION                                │
│ Route: /onboarding                                          │
│ Design: CERP-971                                            │
│                                                              │
│ Welcome screen:                                             │
│ "How would you like to start?"                              │
│                                                              │
│ Option A: "Connect to existing instance"                    │
│   └─→ Opens Confluence guide (always active)                │
│   └─→ Does NOT enter wizard                                 │
│                                                              │
│ Option B: "Start with new instance"                         │
│   └─→ Feature flag gated                                    │
│   └─→ Calls POST /api/v1/start                              │
│   └─→ Backend creates CosmosDB session                      │
│   └─→ Returns sessionId                                     │
│   └─→ Navigates to Step 200                                 │
│   └─→ Stepper now visible                                   │
│                                                              │
│ Backend: CERP-1087 (POST /api/v1/start)                     │
│ Frontend: CERP-1086                                         │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 200: COMPANY DATA - GENERAL                            │
│ Route: /onboarding/company-data                             │
│ Design: CERP-970                                            │
│                                                              │
│ ISI GATEWAY PRE-FILL (via Customer ID from Hub):            │
│ • Company name                                              │
│ • Address (street, city, postal code, country)              │
│ • Contact (phone, email)                                    │
│ • Default language (user's locale)                          │
│                                                              │
│ User actions:                                               │
│ • Verify/correct pre-filled data                            │
│ • Edit any field as needed                                  │
│ • Click "Next" → POST /api/v1/company                       │
│                                                              │
│ Backend: CERP-1088                                          │
│ Frontend: CERP-1085                                         │
│ QA: QCERP-506 / QWAWI-1293                                  │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 300: COMPANY DATA - PAYMENT                            │
│ Route: /onboarding/payment-data                             │
│ Design: CERP-968                                            │
│                                                              │
│ ISI GATEWAY PRE-FILL (via Customer ID):                     │
│ • IBAN                                                      │
│ • BIC                                                       │
│ • Tax ID                                                    │
│ • Small Business checkbox (if applicable)                   │
│                                                              │
│ User actions:                                               │
│ • Verify/correct payment data                               │
│ • Acknowledge payment method: "Purchase on Account" only    │
│ • Click "Next" → POST /api/v1/payment-data                  │
│                                                              │
│ Backend: CERP-1089                                          │
│ Frontend: CERP-1090                                         │
│ QA: QCERP-507 / QWAWI-1294                                  │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 400: SHIPPING TERMS ACCEPTANCE (REMOVED IN v1.6)       │
│ → No longer a wizard step                                   │
│ → Terms accepted externally / auto-accepted                 │
│                                                              │
│ Shipping method auto-created: DHL Paket Inland              │
│ JTL-Shipping App: Auto-installed via Hub API                │
└──────────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 500: SHOP TERMS + SHOP NAME                            │
│ Route: /onboarding/shop                                     │
│ Design: CERP-972                                            │
│                                                              │
│ Two-part confirmation:                                      │
│                                                              │
│ 1. Online Shop Terms of Service                             │
│    • Checkbox: "I agree to the T&Cs"                        │
│    • Link to JTL-Shop backend for editing T&Cs              │
│    • Liability notice: "JTL assumes no liability..."        │
│                                                              │
│ 2. Shop Name Assignment                                     │
│    • Pre-filled: Company name from Step 200                 │
│    • User can edit                                          │
│    • Becomes the shop identifier                            │
│                                                              │
│ User actions:                                               │
│ • Check T&C checkbox                                        │
│ • Edit shop name (optional)                                 │
│ • Click "Complete Setup" → POST /api/v1/shop               │
│                                                              │
│ Backend: CERP-1091                                          │
│ Frontend: CERP-1092                                         │
│ QA: QCERP-508 / QWAWI-1296                                  │
└──────────────────────────────────────────────────────────────┘
                           ↓
                    [ALL INPUT COLLECTED]
                    [PROVISIONING BEGINS]
                           ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 600: SETUP PROGRESS (Automated)                        │
│ Route: /onboarding/completion                               │
│ Design: CERP-973                                            │
│ Backend: CERP-1093 / WAWI-88101 / WAWI-88095               │
│                                                              │
│ SQL SERVER PRINCIPLE in Action:                             │
│ All data collected. Now system provisions infrastructure.    │
│                                                              │
│ Progress Screen displays 3 aggregated sub-steps:            │
│                                                              │
│ 1. "Set up shipping"                                        │
│    ├─ Call JTL-Hosting API: POST /shop/provision            │
│    ├─ Extract shopApiKey from response                      │
│    ├─ Configure Shop: POST /api/wizard/settings             │
│    ├─ Call JTL-Hub API: Install JTL-Shipping App            │
│    └─ Map shipping method in Wawi                           │
│                                                              │
│ 2. "Boot up shop"                                           │
│    ├─ Call JTL-Hosting API: POST /shop/provision            │
│    ├─ Poll for status: GET /shop/{ref_id}/status            │
│    ├─ Push shop settings: PUT /api/wizard/settings           │
│    └─ Mark wizard as done in Shop                           │
│                                                              │
│ 3. "Set up ERP"                                             │
│    ├─ Call JTL-Hosting API: POST /mssql/provision           │
│    ├─ Create provisioning.json on target VM                 │
│    ├─ Execute JTL-Administrator.Console.exe                 │
│    ├─ Initialize databases (eazybusiness)                   │
│    ├─ Create company, warehouse, payment/shipping methods   │
│    ├─ Register Shop connection in Wawi                      │
│    ├─ Fetch M2M credentials → Link tenant                   │
│    ├─ Start REST API + sync worker                          │
│    └─ Trigger initial Wawi-Shop sync                        │
│                                                              │
│ Real-Time Updates:                                          │
│ • Server-Sent Events (SSE) stream status updates            │
│ • Polling fallback: Check every 2 seconds                   │
│ • Errors displayed inline: "Set up shipping: ✗ Failed"      │
│                                                              │
│ Parallel Execution:                                         │
│ • Steps 1 & 2 run in parallel (infrastructure independent)  │
│ • Step 3 (Console) runs after infra ready                   │
│                                                              │
│ Frontend: CERP-1094                                         │
│ QA: QCERP-509 / QWAWI-1298                                  │
└──────────────────────────────────────────────────────────────┘
                           ↓
                    [PROVISIONING COMPLETE]
                           ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 700: SUCCESS SCREEN                                    │
│ Route: /onboarding/completion                               │
│ Design: CERP-974                                            │
│                                                              │
│ Confirmation message:                                       │
│ "🎉 Your business is ready!"                                │
│                                                              │
│ Status summary:                                             │
│ ✓ Company configured                                        │
│ ✓ Shop provisioned                                          │
│ ✓ Shipping set up                                           │
│ ✓ ERP initialized                                           │
│                                                              │
│ Call-to-Action Button:                                      │
│ "Create your first item now"                                │
│ → Navigates to Step 1000 (Item Creation Form)               │
│ → OR → Dashboard if user skips                              │
│                                                              │
│ Backend: CERP-1095                                          │
│ Frontend: CERP-1096                                         │
│ QA: QCERP-510                                               │
└──────────────────────────────────────────────────────────────┘
                           ↓
              [WIZARD COMPLETE - ONBOARDED = TRUE]
              [USER ENTERS PRODUCT SETUP FLOW]
```

---

## 🔌 API Specification

All endpoints require `Authorization: Bearer <JWT>` and use `https://api.jtl-cloud.com/onboarding/` base.

### POST /api/v1/start
**Purpose:** Initialize wizard session  
**Feature Flag:** "start_with_new_instance"  

```json
Request: {} (empty body, auth via JWT)

Response (200):
{
  "sessionId": "uuid-session-id",
  "tenantId": "uuid-tenant-id",
  "step": 100
}

Response (403 Forbidden):
{
  "error": "Feature flag inactive"
}
```

### POST /api/v1/company
**Purpose:** Persist Step 200 company data

```json
Request:
{
  "sessionId": "uuid-session-id",
  "companyName": "Evers Retail",
  "address": {
    "street": "Hauptstr. 42",
    "city": "Berlin",
    "postalCode": "10115",
    "country": "DE"
  },
  "contact": {
    "phone": "+49 30 123456",
    "email": "info@evers.de"
  },
  "language": "de"
}

Response (200):
{
  "sessionId": "uuid-session-id",
  "step": 200,
  "status": "persisted"
}
```

### POST /api/v1/payment-data
**Purpose:** Persist Step 300 payment data

```json
Request:
{
  "sessionId": "uuid-session-id",
  "iban": "DE89370400440532013000",
  "bic": "COBADEFFXXX",
  "taxId": "DE123456789",
  "isSmallBusiness": false
}

Response (200):
{
  "sessionId": "uuid-session-id",
  "step": 300,
  "status": "persisted"
}
```

### POST /api/v1/shop
**Purpose:** Persist Step 500 shop data + trigger provisioning

```json
Request:
{
  "sessionId": "uuid-session-id",
  "agreeToTerms": true,
  "shopName": "Evers Shop"
}

Response (202 Accepted):
{
  "sessionId": "uuid-session-id",
  "jobId": "uuid-job-id",
  "step": 600,
  "status": "provisioning_started"
}
```

### GET /api/v1/status/{jobId}
**Purpose:** Stream provisioning progress via SSE

```
Stream Events:
event: progress
data: {
  "jobId": "uuid-job-id",
  "step": 1,
  "substepLabel": "Set up shipping",
  "status": "in_progress"
}

event: progress
data: {
  "jobId": "uuid-job-id",
  "step": 1,
  "substepLabel": "Set up shipping",
  "status": "completed"
}

event: progress
data: {
  "jobId": "uuid-job-id",
  "step": 2,
  "substepLabel": "Boot up shop",
  "status": "in_progress"
}

event: error
data: {
  "jobId": "uuid-job-id",
  "step": 2,
  "substepLabel": "Boot up shop",
  "error": "Timeout waiting for shop provisioning",
  "retryable": true
}

event: complete
data: {
  "jobId": "uuid-job-id",
  "status": "success",
  "credentials": {
    "wawiBaseUrl": "https://wawi-xyz.jtl-cloud.com",
    "shopUrl": "https://shop-xyz.jtl-cloud.com",
    "m2mClientId": "...",
    "m2mClientSecret": "..."
  }
}
```

---

## 🔐 Security Considerations

### Feature Flag Enforcement
- "Start with new instance" is server-side validated
- Client cannot bypass via URL manipulation
- Returns 403 if flag inactive

### Data Validation
- All input validated against schema (company name, IBAN format, etc.)
- ISI Gateway responses validated before pre-fill
- No SQL injection or XSS via form inputs (sanitization required)

### Credential Management
- M2M credentials generated during provisioning
- Stored securely (not sent to frontend during wizard)
- Transmitted only after provisioning complete

---

## ⚠️ Error Handling

### Step-Level Errors (600)

Displayed inline on progress screen:

```
1. Set up shipping: ✗ Timeout
   Retry? [Yes] [Skip for now]

2. Boot up shop: ⏳ In Progress...

3. Set up ERP: ⏳ Not started
```

### Recoverable Errors
- Network timeouts → Retry available
- Temporary API unavailability → Retry available
- Resource quota exceeded → User contacted

### Fatal Errors
- Infrastructure provider down → Offer support contact
- Invalid input data → Restart wizard
- Credential generation failed → Contact support

---

## 📊 Metrics & Monitoring

- **Session creation success rate** (Step 100)
- **Completion rate per step** (dropoff analysis)
- **Time spent in wizard** (benchmark: 30-45 min)
- **Provisioning time** (Step 600: benchmark: 10-15 min)
- **Error frequency** (sub-step, error type)
- **Feature flag adoption** (% seeing "Start with new instance")

---

## 🔗 Dependencies

### External APIs
- **JTL-Hosting API v2.0.0** → Infrastructure provisioning
- **ISI Gateway** → Master data pre-fill
- **JTL-Hub API** → App installation
- **JTL-Administrator Console** → Wawi initialization
- **JTL Shop API** → Settings/T&C push

### Internal Systems
- **CosmosDB** → Session persistence
- **Cloud Eventbus** → Async orchestration
- **Cloud Authentication** → JWT validation
- **Feature Flags** → Flag gating ("start_with_new_instance")

---

## 📋 Known Issues & Open Questions

🔴 **Blocking:**
- JTL-Hosting API lacks `/compute/*` endpoint for Wawi VM
- Shop API needs `shopApiKey` return + config endpoints

🟡 **In Progress:**
- SQL evolution (eazybusiness separation for Phase 2)
- Shipping label API finalization

🟢 **Resolved:**
- Wizard flow finalized (SQL Server Principle) — v1.1
- Step numbering (×100 scheme) — v1.3
- Architecture update (direct label generation) — v2.1

---

## 📚 Related Documents

- **Technical Concept:** `technical-concept.md` (detailed infrastructure)
- **Wizard Design Finalization:** `20260331-wizard-finalize-flow.md` (SQL Server Principle rationale)
- **Full BRS:** `202605-brs.md` (complete spec)
- **Confluence:** [Step 100-700 details](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1207992460)
