# Summary — Stripe Solution Plan for JTL (Billing + Connect)

> **Source:** `jtl-billing-connect-solution-plan.pdf`
> **Author:** David Joos (Stripe Solution Architect) · **Last updated:** Mar 31, 2026 · **Status:** In Progress

> **📚 Document map** (JTL Stripe integration notes):
> - [`overview.md`](../../overview.md) — high-level overview: red vs green data model + the real benefits
> - [`jtl-billing-connect-solution-plan.md`](./jtl-billing-connect-solution-plan.md) — original Stripe Solution Plan (David Joos)
> - **`summary.md`** — one-page summary of the Solution Plan *(this document)*
> - [`implementation-guide.md`](../../implementation-guide.md) — step-by-step implementation guide (the *how*)
> - [`tax.md`](../../tax.md) — detailed DACH + EU VAT tax handling (the tax deep-dive)

## What this is

Stripe's proposed technical solution for **JTL Software's new App Store**, which replaces JTL's fragmented Extension Store. It's a multi-party marketplace serving **15,000+ merchants** and **1,000+ partners**.

## Business objective

Centralize payments, automate subscription management, and calculate JTL's commissions on partner app sales — moving away from today's setup where each partner runs their own PayPal account, toward a unified, scalable, monetizable platform.

## Key technical decisions

| Area | Decision |
|------|----------|
| Architecture | Multi-party marketplace on **Stripe Connect** |
| Merchant of Record (MoR) | **Partners** (connected accounts) are MoR; JTL keeps flexibility to become MoR later |
| Charge type | **Destination charges** with `on_behalf_of` to assign MoR status to the partner |
| Account type | Partners onboarded as **Express accounts** via the **Accounts v2 API** |
| Data model | Customer, product & subscription data centralized on **JTL's platform account**; products/prices created on the platform, **not** on connected accounts |
| Billing | Monthly/yearly recurring subscriptions via **Stripe Billing** + **Stripe Checkout** |
| Tax | Manual tax rates **or** automated **Stripe Tax** (e.g. 19% German VAT). ⚠️ With Publisher-as-MoR (`on_behalf_of`), Stripe Tax computes against each **Publisher's own** registrations — not the platform's — so tax setup is per-Publisher. See [`tax.md`](../../tax.md). |
| Commissions | Split via `transfer_data[amount_percent]` (example uses a 50% split to the partner) |

## Markets & timeline

- **Launch:** Germany / DACH region, expanding across the EU
- **MVP:** by the Partner Convention in **May 2026**
- **Full rollout:** by **end of 2026**

## Implementation flow

1. **Onboard partners** as connected accounts — request *Merchant* (`card_payments`) + *Recipient* (`stripe_transfers`) capabilities, then send a hosted onboarding link.
2. **Create products & prices** on the platform account.
3. **Create customer** → **Stripe Checkout** session in `subscription` mode with the transfer split, `on_behalf_of`, and tax rates applied.

## Recommended services

Stripe recommends a **custom Professional Services (Proserv) engagement** due to the complexity of combining Billing + Connect + Tax — including custom Invoicing and Subscription UIs and best-practice consulting.

> JTL's selected services were **not yet confirmed as of 31.03.2026**.
