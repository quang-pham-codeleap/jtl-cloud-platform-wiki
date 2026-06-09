# Technical Concept: JTL ERP Cloud – Initial Onboarding & Provisioning

**Version:** 1.7.1 | **As of:** May 2026

**ERP App:** [erp.jtl-cloud.com](https://erp.jtl-cloud.com)

**Primary Specification (SSoT):** 110 BRS – Cloud ERP End-to-End MVP

---

## 1. Overview & Setup Logic

### 1.1 Wizard Flow – 6 Steps ("SQL Server Principle")

The "SQL Server Principle" is the central architectural pattern: **All user input is collected upfront**, and the actual provisioning starts only after the user has completed all manual steps. The user interacts with Steps 100–500; Step 600 and beyond are fully automated.

* **Embedding Model:** The wizard is a standalone app. It is used in the CERP setup flow and as an iFrame within the JTL-Hub connection dialog.
* **Redirect Logic:** The ERP App redirects users to the wizard if no M2M credentials (`clientId`/`clientSecret`) exist for the tenant.

| Step | Route | Content | Notes |
| --- | --- | --- | --- |
| **100** | `/onboarding` | Entry & Mode Selection | Feature-flag gated; "Start with new instance." |
| **200** | `/onboarding` | Company Data – General | Name, address, contact, default language. |
| **300** | `.../payment-data` | Company Data – Payment | IBAN, BIC, Tax ID, Small Business checkbox. |
| **400** | — | **Removed in v1.6** | Shipping TOS accepted outside the wizard. |
| **500** | `.../shop` | Shop TOS + Shop Name | Shop name assignment and T&C acceptance. |
| **600** | `.../completion` | Setup Progress | Visualizes 3 aggregated sub-steps via SSE. |
| **700** | `.../completion` | Success Screen | Final CTA: "Create articles now." |

---

## 2. System Architecture & Data Flow

### 2.1 The Four Layers

1. **Provider Layer:** Azure/Hetzner (not directly accessible by Orchestrator).
2. **Hosting API Layer:** JTL-Hosting API v2.0.0 (External REST API).
3. **Orchestration Layer:** JTL Cloud Control Plane (The "Administrator").
4. **Management Layer:** The Onboarding Wizard App.

### 2.2 Data Flow (Happy Path)

1. **POST /api/v1/start:** Generates a `sessionId`.
2. **Infrastructure Parallelization:** Orchestrator calls JTL-Hosting API for MSSQL and Shop (Plain Infra).
3. **Shop Configuration:** Orchestrator polls for `shopApiKey` → calls Shop’s own API ([CERP-1112](https://www.google.com/search?q=https://jtl-software.atlassian.net/browse/CERP-1112)) to push legal texts and settings.
4. **M2M Sync:** Fetches credentials from ERP Backend.
5. **Console Execution:** Orchestrator creates `provisioning.json` → triggers `JTL-Administrator.Console.exe` on the target VM.
6. **Status Updates:** Console reports back via SSE (Server-Sent Events) with a polling fallback (2s).

---

## 3. Hosting Provisioning (JTL-Hosting API v2.0.0)

All calls require `Authorization: Bearer <serviceToken>` and `X-Correlation-ID: <sessionId>`.

### 3.1 Shop Endpoints (PHP Hosting)

`POST /shop/provision` handles **Plain Provisioning** only (VM, PHP 8.4, Redis, IonCube).

* **Post-Provisioning:** Once status is "completed," the `shopApiKey` is extracted.
* **Configuration:** The Orchestrator uses the key to call:
* `POST /api/wizard/settings`
* `POST /api/wizard/legal-texts`
* `PUT /api/wizard/settings` (to set `mark_wizard_done: true`)



### 3.2 MSSQL Endpoints (Database)

`POST /mssql/provision` creates a tenant-specific instance.

* **Default:** MSSQL 2025 Express.
* **Roles:** Console receives `db_owner` permissions to initialize schemas.

---

## 4. Detailed Provisioning Steps

### 4.4 Console Phase (Final Automated Execution)

The Console performs the following in a single call:

1. Establish DB connection.
2. Create database (`eazybusiness` for MVP).
3. Initial configuration (Company, Warehouse, Tax, Language).
4. Create Shipping & Payment methods (Direct DB access).
5. Create Shop connection & register Sales Channel.
6. Link Tenant via M2M credentials.
7. Install & start REST API + Worker.
8. Trigger initial Wawi-Shop Sync.

---

## 12. API Specification Overview

* **Base URL:** `https://api.jtl-cloud.com/`
* **Onboarding Endpoints:**
* `POST /api/v1/start` → Returns `{ sessionId }`.
* `POST /api/v1/company` / `POST /api/v1/payment-data` / `POST /api/v1/shop`.
* `POST /api/v1/complete` → Starts the background job.
* `GET /api/v1/status/{jobId}` → SSE Stream for UI updates.



---

## 15. Open Questions & Critical Risks

* 🔴 **Wawi-Compute:** JTL-Hosting API currently lacks a `/compute/*` endpoint for the Wawi VM. This is a **blocking** issue for [CERP-1097](https://www.google.com/search?q=https://jtl-software.atlassian.net/browse/CERP-1097).
* 🔴 **Shop API Extension:** The `GET /shop/{ref_id}/status` needs to return the `shopApiKey`, and the Shop-side configuration endpoints must be finalized ([CERP-1112](https://www.google.com/search?q=https://jtl-software.atlassian.net/browse/CERP-1112)).
* 🔴 **SQL Evolution:** Knowledge regarding the separation of `eazybusiness` dependencies for Phase 2 (Shared Instances) is limited; this is a high-risk task.

---

## 16. Changelog (Highlights)

* **v1.7.1:** Added [CERP-1112](https://www.google.com/search?q=https://jtl-software.atlassian.net/browse/CERP-1112) Shop Configuration API integration.
* **v1.7:** Standardized Correlation ID usage and "Plain Provisioning" logic.
* **v1.6:** Removal of Step 400 and integration of Hosting API v2.0.0.
