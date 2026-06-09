# JTL Stripe Integration — Tax Handling (DACH + EU)

**Status:** Analysis / open decisions — needs tax-advisor sign-off before launch
**Scope:** How VAT works for the JTL App Store under the confirmed **Publisher-as-Merchant-of-Record** posture, what Stripe Tax does in that posture, the DACH rules, and the EU "deemed supplier" risk that could override the MoR choice.
**Date:** 2026-06-08

> ⚠️ **Not tax advice.** Engineering/architecture analysis only. The legal determinations (especially Art. 9a in §4) need a qualified DACH/EU VAT advisor. Cited rules are starting points for that advisor, not conclusions.

> **📚 Document map** (JTL Stripe integration notes):
> - [`overview.md`](./overview.md) — high-level overview: red vs green data model + the real benefits
> - [`solution-docs/md/jtl-billing-connect-solution-plan.md`](./solution-docs/md/jtl-billing-connect-solution-plan.md) — original Stripe Solution Plan (David Joos)
> - [`solution-docs/md/summary.md`](./solution-docs/md/summary.md) — one-page summary of the Solution Plan
> - [`implementation-guide.md`](./implementation-guide.md) — step-by-step implementation guide (the *how*)
> - **`tax.md`** — detailed DACH + EU VAT tax handling *(this document — the tax deep-dive)*

---

## 0. TL;DR

- **POC decision: Publisher-as-MoR — confirmed and working.** Verified end-to-end in `curl-logs/`. It's the lowest-liability, fastest path; tax + Art. 9a are **go-live gates, not POC gates**.
- **MoR = Publisher** is a JTL requirement from the Solution Plan, not a Stripe default. Implemented via `on_behalf_of=acct_PUBLISHER_ID` on every Checkout Session.
- **`on_behalf_of` routes tax to the Publisher.** Stripe Tax computes VAT against the **Publisher's** registrations/origin — not the Platform's. The promised "enable Stripe Tax once on the Platform" benefit does **not** apply while the Publisher is MoR.
- **Tax is currently OFF.** Every logged object shows `automatic_tax.enabled: false`, `total_taxes: []`, `amount_tax: 0` — €30.00 billed to a Berlin customer with €0 VAT. Tax handling is **untested**.
- **DACH rates:** DE 19%, AT 20%, CH 8.1% (CH is outside the EU VAT system; rise to 8.8% delayed to 2028).
- **Biggest risk:** the EU **Art. 9a deemed-supplier rule** may make **JTL** the VAT supplier *regardless of `on_behalf_of`*, because JTL authorises the charge and sets the marketplace T&Cs. Decision-critical — see §4.

---

## 1. Why Publisher-as-MoR (decision provenance)

It's a top-level requirement JTL gave Stripe, recorded in the Solution Plan (Technical Requirements):

> "Third-party partners (connected accounts) **must** be the Merchant of Record… JTL should have the flexibility to become MoR at a later stage."
> "The integration must use **destination charges** with the `on_behalf_of` parameter to correctly assign MoR status to the partner."

Two consequences:

1. **It explains the green/centralized data model.** Centralized data + `on_behalf_of` is the only setup where "partners are MoR now, JTL can flip later" is a one-parameter change, not a multi-week migration. MoR requirement and data-model choice are the same decision.
2. **The business *why* is undocumented.** The plan states partners *must* be MoR but gives no rationale. Inferable reasons: 1,000+ partners sell their *own* software; JTL avoids being the legal seller (VAT filings, chargebacks, product-delivery/consumer liability) for apps it didn't build; faster launch; net (commission-only) revenue accounting vs gross. Plausible but unwritten — **record it** (§7.8).

**Caveat:** the plan treats MoR as something JTL freely assigns via `on_behalf_of`. True for **commercial/contractual** purposes; may not hold for **VAT** — see §4.

---

## 2. DACH rates and place-of-supply

| Country | Standard VAT | In EU VAT system? | Notes |
|---|---|---|---|
| 🇩🇪 Germany | **19%** | Yes | Home market |
| 🇦🇹 Austria | **20%** | Yes | Same EU cross-border rules as DE |
| 🇨🇭 Switzerland | **8.1%** | **No** | Separate regime; 8.1% since Jan 2024; 8.8% delayed to 2028 |

For digital/SaaS services the **place of supply is the customer's location**, then splits on **B2B vs B2C**. JTL's tenants are overwhelmingly businesses, so the B2B/B2C distinction dominates.

---

## 3. Scenario walk-through — German Publisher as MoR

**Quick glossary:**

- **VAT** = sales tax (DE 19% / AT 20% / CH 8.1%).
- **The seller** = the Publisher (MoR), legally responsible for charging/paying VAT.
- **B2B** = buyer is a company with a **VAT ID** (validated via the EU **VIES** lookup); **B2C** = private person / no VAT ID.
- **Reverse charge** = cross-border EU B2B rule: seller charges **0%**, buyer self-reports VAT at home — so the seller needn't register abroad.
- **Input tax** (*Vorsteuer*) = VAT a business pays is reclaimable, so it's cash-flow not cost.
- **OSS** (One-Stop-Shop) = report B2C VAT for all EU countries via one home registration.
- **€10k threshold** = under €10k/yr cross-border B2C, may charge home rate.
- **Acquisition tax** (*Bezugsteuer*) = the Swiss reverse-charge equivalent.
- **Swiss worldwide threshold** = foreign sellers must register for Swiss VAT once **global** turnover passes CHF 100k.

**Who pays, who collects (keep three roles separate):** the **buyer bears** the VAT (added to price). The **Publisher (MoR) collects and forwards** it — never its own money. The **buyer's tax authority receives** it. Two softeners: business buyers usually **reclaim** it as input tax (so it's cash-flow, not real cost); reverse charge moves **no VAT money** at all.

**Open pricing decision — is €30 inclusive or exclusive of VAT?** This is the Price's `tax_behavior` (currently `unspecified`):
- **Inclusive:** tenant pays **€30 total**, VAT carved out (Publisher keeps less). Typical for consumers — German law generally requires gross prices to consumers.
- **Exclusive:** tenant pays **€30 + VAT** (e.g. €35.70 @19%). Typical B2B net pricing.

Decide **before** enabling tax. Table below assumes **exclusive** to make numbers visible.

### The scenarios (German Publisher selling at €30 net/mo)

| Who is buying | VAT applied | Buyer pays | Who remits | Publisher setup needed |
|---|---|---|---|---|
| **German business** (VAT ID) | **19%** (no reverse charge within one country) | **€35.70** — reclaims €5.70, net ≈ €0 | Publisher → DE | None (existing DE reg) |
| **German consumer** | **19%** | **€35.70** (final cost) | Publisher → DE | None |
| **Austrian business** (VAT ID) | **0%** reverse charge | **€30.00** | **Buyer** self-reports 20% in AT | Check VAT ID (VIES) + EU sales report |
| **Austrian consumer** | **20%** | **€36.00** | Publisher → AT via **OSS** | OSS registration |
| **Swiss business** | **None** (non-EU) | **€30.00** | **Buyer** self-reports (acquisition tax 8.1%) | Usually none |
| **Swiss consumer** | **8.1%** | **€32.43** | Publisher → CH | Swiss VAT reg (+ likely Swiss representative); triggered by CHF 100k **worldwide** threshold |

**Shape of the burden:** if mostly **B2B with reliable VAT IDs**, the everyday case collapses to "19% domestic" or "0% reverse-charge cross-border" — manageable on each Publisher's home registration. Pain is in the tail: **B2C/no-VAT-ID** (triggers OSS) and **Switzerland** (non-EU, worldwide threshold, separate registration — treat as a special case). Stripe Tax computes every row correctly **only where that Publisher has the registration**.

---

## 4. The decisive risk: `on_behalf_of` ≠ VAT MoR (EU Art. 9a "deemed supplier")

The single point that could invalidate the Publisher-as-MoR tax posture — and Stripe's parameter can't settle it.

Under **Art. 9a of EU VAT Implementing Regulation 282/2011** (validity upheld by CJEU in **Fenix International, C-695/20**), a platform facilitating an **electronically-supplied service** through its interface is **presumed the supplier** for VAT. The presumption is **irrebuttable** if the platform **authorises the charge**, **authorises delivery**, *or* **sets the general T&Cs** — in which case it "shall not be permitted to explicitly indicate another person as the supplier."

**Why this likely bites JTL:** the Checkout Session/PaymentIntent are created on the **JTL Platform account** (JTL authorises the charge) **and** JTL sets the marketplace T&Cs — both explicit triggers. So for **EU VAT on these ESS, JTL may be the deemed supplier regardless of `on_behalf_of`.** That parameter governs the *settlement merchant* and *Stripe Tax account*; EU VAT law has its own test, and JTL fails it as the supplier.

**If the advisor confirms 9a applies:** the VAT obligation may sit with **JTL** (file via JTL's OSS, JTL named as supplier) even while the Stripe charge says Publisher-as-MoR. Our current invoices (Publisher as issuer, no tax line) could be wrong on **both** issuer identity **and** VAT. A contradiction you don't want to find after launch.

**Strategic read:** if EU law treats JTL as supplier anyway, the cleanest alignment is **JTL-as-MoR** (drop `on_behalf_of`, register JTL once for OSS, compute centrally) — also the only config where "centralized tax" is real. **Publisher-as-MoR risks the worst of both worlds:** per-Publisher operational burden *and* deemed-supplier liability.

**Sources:** [Art. 9a, Council Implementing Regulation (EU) No 282/2011 (EUR-Lex)](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A02011R0282-20190101) · [CJEU *Fenix International* C-695/20 — judgment (EUR-Lex/CURIA)](https://curia.europa.eu/juris/liste.jsf?num=C-695/20) · [Validity of Art. 9a upheld (DLA Piper analysis)](https://www.dlapiper.com/en-us/insights/publications/2023/03/validity-of-article-9a-of-council-implementing-regulation-282-2011).

> The Solution Plan is a Stripe *integration* document — correct on mechanics, but **not** a tax-law opinion; it doesn't address Art. 9a. Out of its scope.

---

## 5. How to enable Stripe Tax — two paths

Tax architecture and the MoR decision are the **same** decision. No "just turn it on centrally" while `on_behalf_of` is set.

### 5.1 Why tax setup is *not* part of the KYC onboarding form

A common assumption is that the Express onboarding link (the hosted KYC flow) should collect everything a Publisher needs — including tax. It doesn't, and that's by design. **KYC and tax registration are two different legal domains owned by two different authorities, so Stripe keeps them on separate tracks:**

- **KYC/KYB onboarding** exists to satisfy *financial* regulation (anti-money-laundering, sanctions, card-network rules). It's Stripe verifying *"is this a real business, who owns it, what's the bank account"* so it can switch on the `card_payments` / `transfers` capabilities. Stripe **knows exactly what to ask** because the requirements are fixed by the account's country and entity type.
- **Tax registrations** are the Publisher's *own* obligations with tax authorities (Finanzamt, ESTV, etc.). Stripe **cannot pre-fill or verify these** — it has no way to know which jurisdictions a Publisher is registered in, and that set **changes over time** as they cross thresholds (€10k OSS line, CHF 100k for Switzerland). They're legal facts only the Publisher knows and must keep current. Forcing them into one-time KYC would be both wrong (Stripe can't validate them) and stale the moment a threshold moves.

This is also *why a missing registration fails silently* (`tax_amount=0.00` / `not_collecting`) instead of erroring — Stripe treats "not registered here" as a legitimate business state, not a misconfiguration. The compliance burden is the merchant's, not Stripe's.

**So is there a separate endpoint? Yes — tax is its own subsystem,** with two APIs reachable three ways (all scoped to the connected account via the `Stripe-Account` header):

| Method | What it is | Fit for JTL |
|---|---|---|
| **Embedded Connect components** (`ConnectTaxSettings` + `ConnectTaxRegistrations`, enabled via `/v1/account_sessions`) | Stripe-hosted UI you drop into the partner portal; the Publisher fills it in themselves | ✅ **Recommended** for Express — Publishers own their own legal facts |
| **Tax Settings API + Tax Registrations API** | The raw `POST /v1/tax/settings` + `/v1/tax/registrations` calls (see `implementation-guide.md` Step 2.1b) | ✅ For programmatic setup / testing in Sandbox |
| **Dashboard** | Self-service Tax settings UI | ❌ **Standard accounts only** — Express Publishers have no such UI |

**One nuance:** the *head-office address + default tax code* part **can** ride along with the hosted onboarding flow — toggle it on at **Dashboard → Connect → Onboarding options → Tax**. But the **registrations themselves still come afterward** through one of the three methods above; they can't be collected in the KYC form, for the reasons above.

### Path A — Publisher-as-MoR: onboarding sequence (current posture)

Per-Publisher workflow, not a single toggle. The order matters — registrations require a head office first:

1. **Platform (one-time):** enable Stripe Tax + a default tax category. Makes the feature available; doesn't compute anything for `on_behalf_of` charges by itself.
2. **Per Publisher, during KYC onboarding (optional):** if you enabled the Connect onboarding Tax toggle, the hosted flow collects the Publisher's **head-office address + default tax code** alongside identity/bank KYC.
3. **Per Publisher, after KYC (required):** the Publisher adds a **tax registration per jurisdiction** they're obligated in (DE standard → OSS as they cross EU thresholds → CH separately). Surface this through the **embedded Tax registrations component** (or push via API for testing). Account is ready when `tax.settings.status=active`.
4. **Per Product:** set a `tax_code` (SaaS/ESS category) or it falls back to the account default.
5. **Per Checkout Session:** add the tax params below.
6. **Customer:** address already collected; enable VAT-ID collection for B2B reverse charge.

> Concrete curl for steps 2–3 (Tax Settings + Tax Registrations, with the `Stripe-Account` header) lives in `implementation-guide.md` **Step 2.1b** — this section is the *why* and the sequence; that one is the *how*.

**curl deltas to the existing Checkout Session:**

```bash
  -d "automatic_tax[enabled]=true" \
  -d "automatic_tax[liability][type]=account" \
  -d "automatic_tax[liability][account]=acct_PUBLISHER_ID" \
  -d "tax_id_collection[enabled]=true" \
  -d "customer_update[address]=auto" \
  -d "customer_update[name]=auto"
```

- `automatic_tax[liability]` pins liability to the Publisher, consistent with `on_behalf_of`.
- `tax_id_collection` lets a B2B tenant enter a VAT ID → reverse charge (0% with note).
- `customer_update[address]=auto` is required by Stripe when `automatic_tax` is on with an existing customer.

> **Open verification (David Joos / Stripe):** with `on_behalf_of` + `automatic_tax[liability][type]=account`, does Checkout auto-use the connected account's registrations, or is the `Stripe-Account`-header Tax Calculation path still needed? Docs are explicit on the manual API, less crisp on Checkout/subscription. Source: [Tax for platforms](https://docs.stripe.com/tax/tax-for-platforms).

**Gate to add:** if the Publisher isn't registered in the buyer's country, Stripe returns `0.00`/`not_collecting` and the invoice ships with no VAT and no error. Don't let a Publisher sell into an unregistered jurisdiction.

### Path B — JTL-as-MoR (the "centralized tax" model; future option)

Drop `on_behalf_of` so JTL is MoR → enable Stripe Tax on the Platform, register JTL once (DE + EU OSS; CH separately) → `automatic_tax[enabled]=true` computes against **JTL's** registrations for all Publishers. A **legal/commercial** decision (JTL's name on invoices, JTL files VAT, JTL owns chargebacks), not a code toggle — but the only path where centralized tax is real, and it aligns with §4.

---

## 6. Commission-mechanism discrepancy (flag for reconciliation)

Solution Plan uses `transfer_data[amount_percent]=50`; the implementation uses `application_fee_percent=15`. Two differences: **mechanism** (`application_fee_percent` produces the `ApplicationFee` ledger object and is correct for recurring subscriptions, vs the plan's `transfer_data` simplification) and **rate** (15% mockups vs 50% example). Not strictly tax, but it sits on the same Checkout Session and affects invoice net amounts. Reconcile explicitly (Solution Plan Open Q#1) so nobody ships the 50%/`amount_percent` example.

---

## 7. Recommendations / open items

1. **Get the Art. 9a determination before launch (§4).** Frame precisely: *"We run a Connect marketplace; the platform creates the PaymentIntent and sets marketplace T&Cs but uses `on_behalf_of` to name the Publisher as settlement merchant. For EU VAT on electronically-supplied services, are we the deemed supplier under Art. 9a despite `on_behalf_of`?"* The answer decides whether `on_behalf_of` stays in the Checkout call at all. Sources: [CJEU *Fenix* C-695/20 (DLA Piper)](https://www.dlapiper.com/en-us/insights/publications/2023/03/validity-of-article-9a-of-council-implementing-regulation-282-2011), [EU deemed-supplier / platform classification (ITR)](https://www.internationaltaxreview.com/article/2ffcwtbrymcmhjrxgodts/sponsored/from-matchmaker-to-supplier-deemed-supplies-and-platform-classification-under-eu-vat), [Stripe Tax with Connect (liability)](https://docs.stripe.com/tax/connect).
2. **Collect VAT IDs from day one** (`tax_id_collection[enabled]=true`) — turns most cross-border B2B into clean 0% reverse charge, sidesteps most OSS exposure, whoever is MoR. Source: [Stripe Tax for platforms](https://docs.stripe.com/tax/tax-for-platforms).
3. **Gate go-live by registration** so Publishers can't ship `not_collecting` (0% VAT) invoices into unregistered jurisdictions. Source: [Tax for platforms — `not_collecting` behavior](https://docs.stripe.com/tax/tax-for-platforms).
4. **Treat Switzerland as a special case** — non-EU, CHF 100k **worldwide** threshold, separate registration. Sources: [Swiss VAT for foreign digital providers (RSM)](https://www.rsm.global/switzerland/en/news/foreign-providers-digital-services-switzerland-overview-vat-obligations), [Switzerland VAT 2024 (Taxually)](https://www.taxually.com/blog/vat-rate-switzerland-and-compliance-2024).
5. **Build per-Publisher tax-registration onboarding** (embedded Connect components) if staying on Path A — net-new work. Source: [Stripe Tax with Connect](https://docs.stripe.com/tax/connect).
6. **Run the missing tax verification** end-to-end (Stripe Tax + registered test Publisher, `automatic_tax[enabled]=true`, confirm non-empty `total_taxes` + a VAT line). Current logs don't cover this.
7. **Reconcile the commission mechanism/rate** (§6).
8. **Document the business rationale for Publisher-as-MoR** (§1).

---

## 8. Sources

- Stripe — [Tax for platforms](https://docs.stripe.com/tax/tax-for-platforms) (`on_behalf_of` calc, `Stripe-Account` header, `not_collecting`, the three setup methods, onboarding Tax toggle)
- Stripe — [Tax with Connect](https://docs.stripe.com/tax/connect) (liability determination)
- Stripe — [Tax registrations embedded component](https://docs.stripe.com/connect/supported-embedded-components/tax-registrations) + [Tax Settings API](https://docs.stripe.com/tax/settings-api) (per-Publisher setup, §5.1)
- Stripe — [Identity verification for connected accounts](https://docs.stripe.com/connect/identity-verification) / [KYC requirements](https://support.stripe.com/questions/know-your-customer-(kyc)-requirements-for-connected-accounts) — what KYC covers (no tax), §5.1
- CJEU — [*Fenix International* C-695/20 (DLA Piper)](https://www.dlapiper.com/en-us/insights/publications/2023/03/validity-of-article-9a-of-council-implementing-regulation-282-2011) — Art. 9a validity
- [EU deemed-supplier / platform classification (International Tax Review)](https://www.internationaltaxreview.com/article/2ffcwtbrymcmhjrxgodts/sponsored/from-matchmaker-to-supplier-deemed-supplies-and-platform-classification-under-eu-vat)
- [Swiss VAT for foreign digital providers (RSM)](https://www.rsm.global/switzerland/en/news/foreign-providers-digital-services-switzerland-overview-vat-obligations) — 8.1%, CHF 100k worldwide threshold
- [Switzerland VAT rate & compliance 2024 (Taxually)](https://www.taxually.com/blog/vat-rate-switzerland-and-compliance-2024)

---

## Related documents

See the **📚 Document map** at the top for the full set. Tax-relevant pointers:

- [`implementation-guide.md`](./implementation-guide.md) — **Step 2.1b** (per-Publisher Tax Settings + Registrations curl), §5.4 (Checkout tax params), Open Questions #3/#4
- [`overview.md`](./overview.md) — §2 Tax/VAT and §6 Onboarding (where per-Publisher tax setup is flagged)
- `curl-logs/curl-log.md` + `response-log.md` — end-to-end verification run (tax disabled)
