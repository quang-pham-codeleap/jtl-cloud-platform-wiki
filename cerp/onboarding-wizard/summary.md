# 110 BRS - Cloud ERP End-to-End MVP: Quick Summary

**Status:** 🟡 In Progress (Q2 2026)  
**Initiative:** TCR-1167 | **Milestone:** TCR-1156  
**Last Updated:** 2026-05-26 (v2.1)

---

## 🎯 The Mission

Enable **micro-retailers** to go from sign-up to selling & shipping in **under 2 hours**, all through a web browser, with **zero technical ERP knowledge required**.

---

## 📋 What's Included in This Directory

| File | Purpose |
|------|---------|
| **00-overview.md** | High-level summary of all three flows + architecture |
| **01-onboarding-flow.md** | Detailed spec for onboarding wizard (Steps 100-700) |
| **02-product-setup-flow.md** | Detailed spec for item creation (Steps 800-1100) |
| **03-order-fulfillment-flow.md** | Detailed spec for order processing (Steps 1200-1700) |
| **100-steps-onboarding-specs.md** | Step 100 entry point details |
| **technical-concept.md** | Architecture, APIs, infrastructure layers |
| **202605-brs.md** | Full Business Requirements Specification (v2.1) |
| **20260331-wizard-finalize-flow.md** | Wizard design decision: "SQL Server Principle" |

---

## 🎬 The Three Flows at a Glance

### 1. Onboarding Wizard (30-45 min)
```
Login → Instance Selection
      → Company Data (General)
      → Company Data (Payment)
      → Shipping Terms
      → Shop Terms + Name
      → [SETUP PROGRESS: Automated provisioning]
      → Success Screen
```
**What happens in background:** Infrastructure provisioned (Wawi + Shop), apps installed, master data initialized.

### 2. Product Setup (5-10 min per item)
```
Dashboard → Items
         → [Empty State]
         → Item Form (Name, Price, Image, Description)
         → [Save]
         → [Sync to Shop]
         → Item appears in shop
```
**What happens:** Item saved in Wawi, immediately synced to JTL-Shop via Eventbus.

### 3. Order Fulfillment (2-3 min per order)
```
Orders List (auto-synced)
        → Order Details
        → [Create Invoice] → PDF print
        → [Mark as Paid]
        → [Ship Order] → Generate Label → Print Label
        → Order marked "Shipped"
```
**What happens:** Order flow through Wawi → Shop → ERP Cloud → JTL-Shipping.

---

## 🏗️ Architecture in 30 Seconds

```
┌─────────────────┐
│  ERP Cloud      │  ← User-facing app
│  (Orchestrator) │     (Web browser)
└────────┬────────┘
         │
    ┌────┴──────────────┬──────────────┬──────────────┐
    │                   │              │              │
    ▼                   ▼              ▼              ▼
┌────────────┐  ┌──────────────┐  ┌─────────┐  ┌──────────────┐
│ JTL-Wawi   │  │  JTL-Shop    │  │JTL Ship │  │ISI Gateway   │
│(Business   │  │ (Storefront) │  │(Labels) │  │(Master Data) │
│ Logic)     │  │              │  │         │  │              │
└────────────┘  └──────────────┘  └─────────┘  └──────────────┘
```

**Key Principle:** ERP Cloud = **Orchestrator** (not system of record). Business logic stays in Wawi.

---

## ✅ Key Features Delivered

### Onboarding
- ✓ Central wizard (7 steps)
- ✓ Master data pre-fill via ISI Gateway
- ✓ Automated infrastructure provisioning
- ✓ "SQL Server Principle" UX (collect input → provision)

### Items
- ✓ Simplified form (4 fields only)
- ✓ Direct shop activation
- ✓ Event-triggered sync (Eventbus)

### Orders
- ✓ Auto-sync every 5 minutes + event-triggered
- ✓ Invoice creation & PDF
- ✓ Payment status marking (manual for "Purchase on Account")
- ✓ Shipping label generation (direct to JTL-Shipping)

---

## ⏱️ Target Time-to-Value

| Phase | Duration | Cumulative |
|-------|----------|-----------|
| Onboarding Wizard | 30-45 min | 30-45 min |
| Create First Item | 5-10 min | 35-55 min |
| Customer Order → Ship | 2-3 min | 40-60 min |
| **Total Setup → First Shipment** | — | **≤ 120 min** |

---

## 🚫 What's NOT Included (Out of Scope)

- Mobile support
- Inventory/stock tracking
- Item variants, BOMs
- External payment gateways
- Multi-shop support
- Post-wizard master data editing
- Automated email notifications

---

## 🔗 Quick Navigation

**Want to dive deep?** Pick your flow:

- **Planning a new implementation?** Start with **00-overview.md**
- **Implementing the wizard?** Read **01-onboarding-flow.md**
- **Building the item form?** Read **02-product-setup-flow.md**
- **Working on order processing?** Read **03-order-fulfillment-flow.md**
- **Understanding the technical stack?** Read **technical-concept.md**

**Need the full spec?** See **202605-brs.md** (unabridged BRS v2.1)

---

## 🔐 Security & Compliance

- Feature flag protection (URL manipulation prevented)
- M2M token scoping (cross-tenant access prevented)
- No sensitive data in frontend
- Payment status controlled by merchant (no auto-charge)
- Input sanitization (XSS, SQL injection prevention)

---

## 📊 Critical Dependencies

| Dependency | Status | Impact |
|---|---|---|
| JTL-Hosting API Wawi Compute | 🔴 Blocking | Wawi provisioning |
| Shop Config API Endpoints | 🟡 In Progress | Shop setup |
| SQL eazybusiness Separation | 🔴 Unknown | Phase 2 scalability |

---

## 📈 Metrics to Track

- Wizard completion rate
- Time-to-first-item
- Order sync latency
- Shipping label generation success rate
- Feature flag adoption

---

## 🎓 Key Design Decisions

1. **SQL Server Principle** (Collect first, provision later) → Better UX, no intermittent screens
2. **ERP Cloud as Orchestrator** (not system of record) → Scalability, maintains Wawi as source of truth
3. **Direct JTL-Shipping Label Generation** (v2.1) → Faster, fewer hops
4. **Event-Based Sync** (no Workers) → Simpler infrastructure

---

## 📚 References

**Confluence:** [110 BRS - Cloud ERP End-to-End MVP (TCR-1167)](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1207992460)

**Jira:** [TCR-1167](https://jtl-software.atlassian.net/browse/TCR-1167)

**Figma:** [ERP Cloud Design v5 MVP Prototype](https://www.figma.com/)

---

## 🤔 Questions?

Each flow document has detailed API specs, error handling, and dependencies. Start with the overview, pick your flow, go deep.

**This directory is the single source of truth for implementation.** Confluence contains the original spec; these markdown files are the distilled, implementation-ready reference.
