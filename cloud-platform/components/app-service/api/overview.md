## JTL Platform App Service — Architect Overview

**Role:** This is the **marketplace platform service** for JTL's ecosystem. It owns the full lifecycle of third-party apps — from developer submission to tenant installation.
**Destination:** https://api.jtl-cloud.com/app-service
**Source:**  /Users/quang/Documents/jtl/repos/jtl-platform-app-service/src/backend

---

### Core Responsibilities

1. **App Registry** — Canonical store of apps and their metadata (manifests, images, listings). Acts as the source of truth for "what apps exist and what state are they in."

2. **Tenant App Installations** — Tracks which tenants have installed which apps. Manages the install/uninstall/configure lifecycle per tenant.

3. **Publisher Management** — Onboards ISVs (independent software vendors) as publishers, including Stripe Connect integration for revenue sharing.

4. **Approval Workflow** — Apps go through a review pipeline (Draft → Submitted → InReview → Approved → Published) before appearing in the store.

5. **Payment Processing** — Stripe as the payment backbone (billing portal, Connect onboarding, webhook handling).

---

### Boundaries & Integration Points

```
                    ┌─────────────────────────────────┐
  Tenant UIs ──────▶│         Main API Host            │
  Publisher UIs ───▶│  (apps, instances, publishers,   │
  Partner Portal ──▶│   listings, payments)            │
                    └──────────────┬──────────────────┘
                                   │
                    ┌──────────────▼──────────────────┐
  JTL Admin ───────▶│         Admin API Host           │
                    │  (lifecycle mgmt, stats, audit)  │
                    └──────────────┬──────────────────┘
                                   │
              ┌────────────────────┼───────────────────┐
              ▼                    ▼                   ▼
         Cosmos DB           Azure Service Bus      Stripe
       (app/tenant data)    (async events)       (payments)
```

**Inbound integrations:**
- JWT/OAuth2 from Ory for identity
- Stripe webhooks for payment events
- M2M OAuth2 scope (`app.read.all`) for internal service-to-service queries

**Outbound integrations:**
- Stripe Connect (publisher onboarding + billing portal)
- Azure Service Bus (publishes app lifecycle events downstream)
- Azure Blob Storage (app images/screenshots)

---

### Architectural Highlights

- **Two deployment surfaces:** Main API (customer/publisher-facing) and Admin API (internal ops tooling) are separate hosts — independently deployable, different auth models.
- **Clean Architecture** with strict layer separation (API → Application → Domain → Infrastructure) per module. Business logic never bleeds into the API layer.
- **Module boundaries:** `Core` (apps, publishers, listings) and `Payment` (Stripe) are independent modules — the payment concern is fully encapsulated.
- **Multi-tenancy:** All app instance data is tenant-scoped; the Cosmos DB partition strategy reflects this.
- **CQRS:** All operations go through explicit command/query objects — no fat service classes.

---

### What It Does NOT Own

- Identity/auth (delegated to Ory)
- Billing logic beyond Stripe pass-through
- App runtime execution (it tracks installations, it doesn't run the apps)
- Tenant provisioning itself (it reacts to tenant events, doesn't create tenants)
