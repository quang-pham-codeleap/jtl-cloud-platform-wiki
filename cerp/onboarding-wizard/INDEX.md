# 110 BRS Documentation Index

**Initiative:** TCR-1167 — Cloud ERP End-to-End MVP  
**Last Updated:** 2026-05-29  
**Scope:** Complete onboarding wizard + product setup + order fulfillment

---

## 📚 Documentation Files

### Start Here

| File | Purpose | Read Time | Audience |
|------|---------|-----------|----------|
| **summary.md** | Quick 2-minute overview of all flows | 2 min | Everyone |
| **00-overview.md** | Complete high-level guide + architecture | 10 min | Product Managers, Architects |

### Flow-Specific Details (Implementation Guides)

| File | Purpose | Read Time | Audience |
|------|---------|-----------|----------|
| **01-onboarding-flow.md** | Onboarding wizard deep dive (Steps 100-700) | 20 min | Engineers (Frontend/Backend) |
| **02-product-setup-flow.md** | Item creation deep dive (Steps 800-1100) | 15 min | Engineers (Frontend/Backend) |
| **03-order-fulfillment-flow.md** | Order processing deep dive (Steps 1200-1700) | 25 min | Engineers (Frontend/Backend, QA) |

### Reference & Context

| File | Purpose | Read Time | Audience |
|------|---------|-----------|----------|
| **100-steps-onboarding-specs.md** | Step 100 entry point specification | 5 min | Frontend engineers |
| **technical-concept.md** | Architecture, APIs, infrastructure | 15 min | Backend engineers, DevOps |
| **202605-brs.md** | Full Business Requirements Spec (v2.1) | 20 min | Product managers, Architects |
| **20260331-wizard-finalize-flow.md** | Wizard design decision documentation | 5 min | Product managers, Architects |

---

## 🎯 How to Use This Documentation

### 👶 "I'm brand new to this project"
1. Read **summary.md** (2 min) → Understand what we're building
2. Read **00-overview.md** (10 min) → See the three flows
3. Pick your flow from Flow-Specific Details above

### 🔨 "I'm implementing [specific flow]"
1. Jump to the flow document (01, 02, or 03)
2. Read the journey map + API specs
3. Reference **technical-concept.md** for infrastructure details

### 🏗️ "I need to understand the architecture"
1. Read **00-overview.md** (section: "Technical Architecture")
2. Read **technical-concept.md** (full details)
3. Read **202605-brs.md** (if you need exhaustive spec)

### 🎬 "I'm presenting this to stakeholders"
1. Use **summary.md** for the 2-minute pitch
2. Use **00-overview.md** section "The Three Flows" for visuals
3. Cite **202605-brs.md** for the authoritative BRS

### 🐛 "I'm debugging a specific step"
1. Search the flow document (01, 02, or 03) for your step number
2. Find the journey map section with your step
3. Look up the API specs + error handling section

---

## 📋 Content Organization by Flow

### Flow 1: Onboarding (Steps 100-700)
**Primary Document:** `01-onboarding-flow.md`

Covers:
- Welcome screen & instance selection
- Company data forms (general + payment)
- Shipping & shop terms acceptance
- Automated provisioning (progress screen)
- Success screen
- API specifications
- Error handling
- Dependencies

**Related:** `technical-concept.md` (infrastructure details)

---

### Flow 2: Product Setup (Steps 800-1100)
**Primary Document:** `02-product-setup-flow.md`

Covers:
- Main menu navigation
- Item list (empty state)
- Item creation form (4-field simplified)
- Save & sync trigger
- Event-based shop sync
- API specifications
- Data model
- Sync behavior

**Related:** `202605-brs.md` (full item spec)

---

### Flow 3: Order Fulfillment (Steps 1200-1700)
**Primary Document:** `03-order-fulfillment-flow.md`

Covers:
- Order list (auto-synced)
- Order details
- Invoice creation & PDF
- Payment status marking
- Shipping release logic
- Shipping dialog & label generation
- API specifications
- Security & validation

**Related:** `technical-concept.md` (sync architecture)

---

## 🔍 Search Guide

**Looking for:**

| Topic | File | Section |
|-------|------|---------|
| API endpoints | 01, 02, 03 | "API Specification" |
| Error handling | 01, 02, 03 | "Error Handling" |
| Database/data model | 01, 02, 03 | "Data Model" |
| Security | 01, 02, 03 | "Security" |
| Dependencies | 01, 02, 03 | "Dependencies" |
| Metrics | 01, 02, 03 | "Metrics" |
| Feature flag | 01 | "Feature Flag Enforcement" |
| Eventbus | 02, 03 | "Sync Behavior" / "Event-Based Sync" |
| ISI Gateway | 01 | "User Journey Map" |
| JTL-Hosting API | technical-concept.md | "Section 3: Hosting Provisioning" |
| Shipping labels | 03 | "Step 1600" |
| Invoice creation | 03 | "Step 1300" |

---

## 📊 Document Metadata

### Version & History

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.0 | 2026-05-29 | Claude Code | Initial markdown documentation (consolidated from Confluence) |

**Confluence Source:** [110 BRS - Cloud ERP End-to-End MVP (TCR-1167)](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1207992460)

### Document Structure

- **Flow documents** (01-03): User journey map → API specs → Error handling → Dependencies
- **Overview** (00): High-level architecture + three flows summary
- **Reference** (technical-concept, 202605-brs): Detailed specs + architecture

---

## 🎯 What Each Document Is For

### `summary.md`
**When to use:** You have 2 minutes and need to understand what this project does.  
**What you get:** Quick overview of three flows, time-to-value, key features.

### `00-overview.md`
**When to use:** You're new to the project or planning the implementation.  
**What you get:** Complete high-level guide, architecture diagram, scope & constraints, user stories.

### `01-onboarding-flow.md`
**When to use:** You're implementing the onboarding wizard (Steps 100-700).  
**What you get:** Detailed user journey, API specs, feature flag logic, error handling, dependencies.

### `02-product-setup-flow.md`
**When to use:** You're implementing the item creation flow (Steps 800-1100).  
**What you get:** User journey, form structure, save & sync logic, API specs, validation rules.

### `03-order-fulfillment-flow.md`
**When to use:** You're implementing order processing (Steps 1200-1700).  
**What you get:** User journey, order list, invoice, payment, shipping, label generation specs.

### `100-steps-onboarding-specs.md`
**When to use:** You need detailed Step 100 (entry point) specification.  
**What you get:** Entry point UI, feature flag, backend/frontend requirements, QA notes.

### `technical-concept.md`
**When to use:** You need to understand the architecture, APIs, or infrastructure.  
**What you get:** System architecture (4 layers), data flow, hosting provisioning, API overview.

### `202605-brs.md`
**When to use:** You need the authoritative, complete Business Requirements Specification.  
**What you get:** Full BRS v2.1, all requirements, out-of-scope items, future requirements.

### `20260331-wizard-finalize-flow.md`
**When to use:** You need to understand the "SQL Server Principle" design decision.  
**What you get:** Wizard design rationale, the finalized user flow structure.

---

## 🔄 Document Relationships

```
                    summary.md
                        ↓
                   00-overview.md
                   /    |    \
                  /     |     \
            01-flow  02-flow  03-flow
               ↓        ↓        ↓
        (referenced by flow docs)
            │          │        │
            └──────────┼────────┘
                       ↓
            technical-concept.md
                       ↓
              202605-brs.md (authoritative source)
```

**Read order for comprehensive understanding:**
1. summary.md
2. 00-overview.md
3. Pick your flow (01, 02, or 03)
4. technical-concept.md (for architecture)
5. 202605-brs.md (if you need complete spec)

---

## 📞 When to Reference vs. When to Update

### Reference These (Read-Only)
- Confluence source docs (authoritative)
- JIRA tickets (status of work)
- Figma prototypes (UI design)

### Update These (Living Docs)
- Flow documents (01-03) — as implementation evolves
- technical-concept.md — as architecture changes
- summary.md — as features ship

---

## ✅ Checklist: Are You Ready?

- [ ] Read **summary.md** (2 min)
- [ ] Read **00-overview.md** (10 min)
- [ ] Read your flow document (15-25 min)
- [ ] Skim **technical-concept.md** (10 min)
- [ ] Bookmarked **202605-brs.md** for reference
- [ ] Saved Confluence link for original specs

**You're ready to implement!**

---

## 🤝 Contributing to This Documentation

These are the **implementation-ready** markdown versions of the Confluence specs. When Confluence is updated with new requirements:

1. Update the relevant flow document (01-03)
2. Update overview.md if scope changes
3. Update technical-concept.md if architecture changes
4. Update this INDEX.md if new docs are added
5. Keep summary.md current with latest timeline/status

**Goal:** This directory should be the primary reference for implementation, not Confluence.

---

## 📎 External References

| Resource | Purpose | Link |
|----------|---------|------|
| Confluence | Original spec | [110 BRS - Cloud ERP End-to-End MVP (TCR-1167)](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1207992460) |
| Jira Initiative | Tracking | [TCR-1167](https://jtl-software.atlassian.net/browse/TCR-1167) |
| Jira Milestone | Tracking | [TCR-1156](https://jtl-software.atlassian.net/browse/TCR-1156) |
| Figma Design | UI Reference | [ERP Cloud Design v5 MVP](https://www.figma.com/) |
| Technical Concept | Architecture | [110 - ERP End2End Flow - Technical Concept](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1284767746) |

---

**Last Updated:** 2026-05-29 | **Version:** 1.0 | **Status:** 🟡 In Progress (Q2 2026)
