# 110 BRS - Cloud ERP End-to-End MVP: Complete Overview

**Initiative:** TCR-1167  
**Milestone:** TCR-1156  
**Status:** 🟡 In Progress (Q2 2026)  
**Last Updated:** 2026-05-26 (v2.1)  

---

## 📍 What This Is

A comprehensive Business Requirements Specification for the Cloud ERP's first end-to-end customer journey: **Buy-to-Ship** in under 2 hours for micro-retailers, with zero technical ERP knowledge required.

**Problem Solved:** Current JTL Cloud is a collection of isolated workflows. New customers cannot complete the journey from sign-up to physical shipping without the JTL-Wawi desktop client.

**Solution:** Automated onboarding wizard + simplified item/order UIs + event-based syncing across JTL-Wawi, JTL-Shop, and JTL-Shipping.

---

## 🎬 The Three Flows

### 1️⃣ ONBOARDING FLOW (Steps 100-700)

**Duration:** ~30-45 minutes  
**User Role:** New micro-retailer  
**Entry Point:** `erp.jtl-cloud.com` → "Start with new instance"

```
Step 100: Instance Selection
   ↓
Step 200: Company Data - General (ISI Gateway pre-fill)
   ↓
Step 300: Company Data - Payment (ISI Gateway pre-fill)
   ↓
Step 400: Shipping Terms Acceptance
   ↓
Step 500: Shop Terms + Shop Name
   ↓
Step 600: Setup Progress (Automated)
   • Set up shipping
   • Boot up shop
   • Set up ERP
   ↓
Step 700: Success Screen
   └─→ CTA: "Create your first item now"
```

**Architecture:** "SQL Server Principle"
- Collect ALL user input upfront (Steps 100-500)
- Single provisioning phase (Step 600)
- No intermittent loading screens
- Parallel infrastructure provisioning (Wawi + Shop)

**Key Integrations:**
- **ISI Gateway**: Pre-fills company data from JTL-Kundencenter
- **JTL-Hosting API v2.0.0**: Spins up MSSQL instance & PHP Shop
- **JTL-Administrator Console**: Initializes Wawi database & core config
- **JTL-Hub API**: Silent installation of JTL-Shipping App

---

### 2️⃣ PRODUCT SETUP FLOW (Steps 800-1100)

**Duration:** ~5-10 minutes per item  
**User Role:** Merchant  
**Entry Point:** Success Screen CTA or Dashboard → Items

```
Step 800: Main Menu Navigation
   ↓
Step 900: Item List (Empty State)
   ↓
Step 1000: Item Creation Form
   • Name (required)
   • Price (required)
   • Image (optional)
   • Description (optional)
   • Activate for JTL-Shop (checkbox per channel)
   ↓
Step 1100: Save & Transfer
   ├─→ Persist item in Wawi
   ├─→ Trigger Eventbus sync
   └─→ Item appears in JTL-Shop automatically
```

**Key Features:**
- **Reduced form**: Only 4 fields (vs. full Wawi item master)
- **Auto-selection**: Standard category pre-selected
- **Direct activation**: Enable shop channel from item form
- **Event-triggered sync**: No manual "sync now" button

---

### 3️⃣ ORDER FULFILLMENT FLOW (Steps 1200-1700)

**Duration:** ~2-3 minutes per order  
**User Role:** Merchant  
**Entry Point:** Dashboard → Orders

```
Step 1200: Order List
   • Payment Status (Paid/Unpaid, or Partially Paid if granular)
   • Shipping Status (Shipped/Not Shipped)
   • Invoice Numbers
   • Auto-synced every 5 minutes + event-triggered
   ↓
Step 1300: Order Details - Invoice
   ├─→ Create invoice in Wawi
   └─→ PDF rendered in browser print dialog
   ↓
Step 1400: Order Details - Payment
   ├─→ Mark as Paid (toggle full payment)
   └─→ Enables shipping action if "Purchase on Account"
   ↓
Step 1500: Shipping Release Logic
   └─→ Available if: Order paid OR Payment="Purchase on Account" + no stock items
   ↓
Step 1600: Shipping Dialog & Label Generation
   ├─→ Create delivery note in Wawi
   ├─→ Request label PDF from JTL-Shipping App (direct call, not via Wawi)
   ├─→ Browser print dialog
   └─→ Mark order as "Shipped"
   ↓
Step 1700: Minor UI Change
   └─→ Rename "Kundschaft" → "Kunden" across UI
```

**Key Features:**
- **Auto-sync**: Orders appear without manual refresh (Eventbus + polling every 5 min)
- **Invoice integration**: Wawi backend → PDF → browser print
- **Payment marking**: One-click "Mark as Paid" flow
- **Label generation**: Direct ERP Cloud → JTL-Shipping (no Wawi proxy)
- **Status granularity**: Supports "Partially Paid" if Wawi provides it

---

## 🏗️ Technical Architecture

### Core Principle: Orchestration Layer

ERP Cloud is the **central orchestrator**:
- Initiates hosting provisioning (JTL-Hosting API)
- Triggers infrastructure setup (JTL-Administrator Console)
- Manages app installations (Hub API)
- Coordinates data sync (Eventbus)
- Presents unified UI (no desktop client required)

**Business logic remains in JTL-Wawi** (by design). ERP Cloud communicates exclusively via APIs.

### Infrastructure Layers

| Layer | Component | Role |
|-------|-----------|------|
| **Provider** | Azure / Hetzner | Physical compute |
| **Hosting API** | JTL-Hosting API v2.0.0 | MSSQL + Shop provisioning |
| **Orchestration** | JTL Cloud Control Plane | Coordinates workflow |
| **Management** | Onboarding Wizard App | User-facing UI |

### Data Sync Strategy

- **Interval Sync**: Eventbus polls Wawi every 5 minutes (orders, item sync)
- **Event-Triggered Sync**: Immediate sync on item save or key actions
- **No JTL-Worker**: All sync via Cloud Eventbus + APIs (Workers disabled for MVP)
- **Direction**: JTL-Shop → Wawi → ERP Cloud (read-only for Cloud)

---

## 📊 Scope & Constraints

### ✅ IN SCOPE

- Central onboarding wizard (7 steps)
- Master data pre-fill (ISI Gateway)
- Infrastructure provisioning (Wawi + Shop)
- Simplified item form (4 fields)
- Shop sync via Eventbus
- Order list with payment/shipping status
- Invoice creation & PDF
- Shipping label generation (direct to JTL-Shipping)
- Payment marking
- **Desktop web browser only**

### ❌ OUT OF SCOPE

- Mobile support
- Inventory/stock tracking
- Item variants, BOMs, configurable items
- External payment gateways (PayPal, etc.)
- Partial shipments (1:1 only)
- Post-wizard master data editing (TCR-1190)
- Multi-shop support
- Label reprint in UI (done in JTL-Shipping)
- Automated email notifications
- Domain/DNS management

---

## 🚀 Non-Functional Requirements

| Requirement | Target | Notes |
|---|---|---|
| **Time-to-Value** | ≤120 minutes (setup → first item ready) | MVP target; 10 min target requires pre-spawning |
| **Platform** | Desktop web only | No mobile |
| **Feature Flag** | Yes | "Start with new instance" gated |
| **Breaking Change** | Yes | Replaces manual setup flow |
| **Error Handling** | Centralized | Errors shown at corresponding step on progress screen |

---

## 📖 Document References

### On Confluence (110 BRS Parent)

- **Step Details**: 17 detailed specs (Step 100, 200, 300, ..., 1700)
- **User Flow Ticket Mapping**: Step-by-step Jira ticket breakdown
- **110 - ERP End2End Flow - Technical Concept**: v1.7.1 detailed architecture
- **Error Handling**: Full error scenario matrix
- **Change Request**: Architecture update log (v2.0 → v2.1)
- **Hub/Wizard Integration**: Infrastructure setup docs

### In This Directory

- `100-steps-onboarding-specs.md` — Step 100 (Entry) detailed spec
- `technical-concept.md` — Architecture, APIs, data flow
- `202605-brs.md` — Full BRS v2.1 text
- `20260331-wizard-finalize-flow.md` — Wizard design decision (SQL Server Principle)
- `00-overview.md` — This file (quick reference)

---

## 🔄 User Stories

### Story 1: Onboarding
> As a micro-retailer (Sebastian Evers), I want to set up my entire system (ERP + Shop + Shipping) automatically via a guided wizard, so I can be ready to sell in under 2 hours without technical ERP knowledge.

### Story 2: Item Creation
> As a micro-retailer, I want to create my first item through an intuitive flow, activate it for JTL-Shop directly, and start sync automatically, so I can get my first product live quickly.

### Story 3: Order Processing
> As a micro-retailer, I want to process shop orders, create invoices, and complete shipping including label printing through a guided interface, so I can quickly deliver without prior ERP knowledge.

---

## 🎯 Key Decisions

### "SQL Server Principle"
Collect all user input upfront before provisioning begins. Eliminates intermittent loading screens and provides a smoother UX.

### "ERP Cloud as Orchestrator"
Cloud acts as the single entry point; Wawi remains the system of record for business logic.

### Direct Shipping Label Generation (v2.1)
Labels generated directly from ERP Cloud → JTL-Shipping (not via Wawi proxy). Decided 2026-05-22.

### Event-Based Sync (No JTL-Worker)
Eventbus handles all sync orchestration. Workers disabled for MVP to simplify infrastructure.

### ISI Gateway Master Data Pre-Fill
Company data auto-populated from JTL-Kundencenter using customer ID. Reduces wizard friction.

---

## 🚨 Critical Dependencies

| Blocker | Status | Impact |
|---------|--------|--------|
| JTL-Hosting API v2.0.0 Wawi Compute Endpoint | 🔴 **Missing** | Blocks MSSQL provisioning (CERP-1097) |
| Shop API Configuration Endpoints | 🟡 Partial | Needs `shopApiKey` return + config endpoints (CERP-1112) |
| SQL Evolution (eazybusiness separation) | 🔴 **Unknown** | Phase 2 shared instances at risk |

---

## 📅 Timeline

| Phase          | Period  | Status     |
| -------------- | ------- | ---------- |
| Concept        | Q1/26   | ✅ Complete |
| Implementation | Q2/26   | 🟡 Active  |
| Delivery       | EOQ2/26 | Planned    |

---
## New Requirements

One-click provisioning

## 🔗 Quick Links

- **Confluence Home**: [110 BRS - Cloud ERP End-to-End MVP](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1207992460)
- **Jira Initiative**: [TCR-1167](https://jtl-software.atlassian.net/browse/TCR-1167)
- **Figma Prototype**: [ERP Cloud Design v5 MVP](https://www.figma.com/design/...)
- **Technical Concept**: [110 - ERP End2End Flow - Technical Concept](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1284767746)
