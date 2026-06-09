# JTL Stripe Integration — Green Model Implementation Guide

**Companion to:** "JTL Stripe Integration — Data Model Architecture Handover"  
**Scope:** Migration to the **green/centralized** model. This document specifies exactly how new objects must be created, with verified Stripe API references you can run against the Sandbox today.

> **📚 Document map** (JTL Stripe integration notes):
> - [`overview.md`](./overview.md) — high-level overview: red vs green data model + the real benefits
> - [`solution-docs/md/jtl-billing-connect-solution-plan.md`](./solution-docs/md/jtl-billing-connect-solution-plan.md) — original Stripe Solution Plan (David Joos)
> - [`solution-docs/md/summary.md`](./solution-docs/md/summary.md) — one-page summary of the Solution Plan
> - **`implementation-guide.md`** — step-by-step implementation guide *(this document — the how)*
> - [`tax.md`](./tax.md) — detailed DACH + EU VAT tax handling (the tax deep-dive)

All API patterns are drawn from the canonical Stripe docs current as of May 2026:
- [Connect subscriptions](https://docs.stripe.com/connect/subscriptions)
- [Destination charges](https://docs.stripe.com/connect/destination-charges)
- [Checkout Session API reference](https://docs.stripe.com/api/checkout/sessions/create)

> All curl commands use `$STRIPE_KEY`. Export your Stripe test secret key before running: `export STRIPE_KEY=sk_test_...`

---

## Migration Plan

### 0. Pre-flight: nuclear option runbook (Sandbox-only)

We have no production data. The simplest path is to **delete the entire Sandbox and start fresh** rather than surgically removing each account. A fresh Sandbox is a clean slate: no lingering Products, Prices, Customers, webhook endpoints, or event logs.

Steps:
1. Delete the current Sandbox via Stripe Dashboard.
2. A new Sandbox is provisioned automatically.
3. Truncate JTL's database mappings in Cosmos DB (including the Publisher table).
4. Re-wire Stripe configuration: new Platform API keys, new webhook signing secret, new webhook endpoint URI.
5. Verify cleanness before starting the green-model implementation.

> ⚠️ **Do not use this in production.** Sandbox deletion is non-recoverable. The production path requires archiving data and preserving account IDs.

### Step 1 — Delete the current Sandbox

1. Stripe Dashboard → **Settings → Account → Sandbox Data**.
2. Click **Delete all test data**.
3. Confirm.

This wipes everything: Connected Accounts, Products, Prices, Customers, Subscriptions, Charges, Events, API keys, and webhook endpoints. Stripe immediately provisions a fresh Sandbox.

**Before you do this:**
- Save your current Platform API keys — you'll generate new ones after.
- Any webhook endpoint configuration is gone — re-create it in Step 4.
- If another developer shares this Sandbox, **coordinate** before deleting.

### Step 2 — Truncate JTL's Cosmos DB mappings

| Container | Reason |
|---|---|
| **Publisher** | Holds `pub_xxx → acct_xxx` mapping — Publishers will be re-onboarded fresh |
| **TenantStripeCustomers** | New mappings created on first checkout against the green-model Platform account |
| **AppProductPriceMapping** | CP-3100 will re-create entries on the Platform |
| **TenantAppSubscriptionMapping** | Re-created when Tenants re-subscribe |

**Do NOT truncate:** webhook event log tables (keep for audit) or commission/accounting tables (keep for historical reference).

### Step 3 — Capture new Platform API keys

1. Stripe Dashboard → **Developers → API keys**.
2. Copy the new Secret Key.
3. Update your app's secrets: `.env` file, CI/CD secrets, etc.
4. Restart your app or reload the config.

### Step 4 — Re-wire the webhook endpoint

1. Stripe Dashboard → **Developers → Webhooks**.
2. Add an endpoint pointing to your webhook handler.
3. Subscribe to: `checkout.session.completed`, `customer.subscription.created`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.deleted`.
4. Copy the new signing secret (`whsec_...`) and update your app config.

### Step 5 — Verify cleanness

Check that the fresh Platform account has zero objects before starting:

```bash
curl -G https://api.stripe.com/v1/products -u "$STRIPE_KEY:" -d "active=true" -d "limit=10"
curl -G https://api.stripe.com/v1/customers -u "$STRIPE_KEY:" -d "limit=10"
curl -G https://api.stripe.com/v1/subscriptions -u "$STRIPE_KEY:" -d "limit=10"
curl -G https://api.stripe.com/v1/accounts -u "$STRIPE_KEY:" -d "limit=10"
# → all should return empty data arrays
```

---

## Object-creation flow

> **Concepts first:** the entity model (Customer, Product, Subscription, etc.) and the three green-model parameters (`on_behalf_of`, `transfer_data[destination]`, `application_fee_percent`) are explained in [`overview.md`](./overview.md) → *Stripe Data Model & Entities* and *Three Stripe Parameters*. This section is the hands-on build sequence.

> Replace `acct_xxx`, `prod_xxx`, `price_xxx`, and `cus_xxx` placeholders with the real IDs returned by earlier steps.

### Step 2.1 — Onboard a Publisher

Onboarding a Publisher takes three calls: create the account, generate a KYC link for them to complete, then check the result. Connected Accounts use the **Accounts v2 API**; everything else (Products, Prices, Customers, Checkout Sessions, Subscriptions) stays on v1. See [accounts-v2-reference.md](./accounts-v2-reference.md) for the full v1→v2 field mapping.

**1. Create the Publisher's Connected Account:**

```bash
curl https://api.stripe.com/v2/core/accounts \
  -u "$STRIPE_KEY:" \
  -H "Content-Type: application/json" \
  -H "Stripe-Version: 2025-12-15.preview" \
  -d '{
    "dashboard": "express",
    "contact_email": "publisherEmail",
    "identity": { "country": "DE", "entity_type": "company" },
    "configuration": {
      "merchant":   { "capabilities": { "card_payments": { "requested": true } } },
      "recipient":  { "capabilities": { "stripe_balance": { "stripe_transfers": { "requested": true } } } }
    },
    "defaults": {
      "responsibilities": { "fees_collector": "application", "losses_collector": "application" }
    },
    "metadata": { "publisherId": "publisherId" },
    "include": ["configuration.merchant", "configuration.recipient", "identity"]
  }'
# → returns object "v2.core.account" with id acct_xxx
```

A few fields are easy to get wrong:
- `contact_email` is **required** whenever you request a `recipient` configuration. Stripe rejects the call without it.
- The `merchant` configuration is needed because the Publisher is the MoR (via `on_behalf_of`). Keep it; drop it only if JTL later becomes MoR.
- `losses_collector` and `fees_collector` must **both** be `application` — see the note below.

> ⚠️ **Both collectors must be `application`.** The Express Dashboard only accepts `fees_collector` and `losses_collector` set to `application`; any other combination fails with `account_controller_unsupported_configuration`. The practical effect: **JTL carries chargeback and negative-balance liability at launch**, even though `on_behalf_of` makes the Publisher the MoR. Tax liability (Publisher) and loss liability (JTL) are separate things in Stripe's model. To put losses on the Publisher instead, you'd have to use `dashboard=full` or `dashboard=none`. Confirm the split with Finance/Legal before launch (Open Question #4).

> ⚠️ **A new account isn't usable immediately** — it can take up to ~10 minutes. Stripe sends a `v2.core.account.created` event when it's ready.

**2. Generate the KYC onboarding link** so the Publisher can verify their identity. Use **Account Links v2** and request both configurations:

```bash
curl https://api.stripe.com/v2/core/account_links \
  -u "$STRIPE_KEY:" \
  -H "Content-Type: application/json" \
  -H "Stripe-Version: 2025-12-15.preview" \
  -d '{
    "account": "acct_PUBLISHER",
    "use_case": {
      "type": "account_onboarding",
      "account_onboarding": {
        "configurations": ["recipient", "merchant"],
        "refresh_url": "https://partner.jtl-cloud.com",
        "return_url": "https://partner.jtl-cloud.com",
        "collection_options": { "fields": "eventually_due" }
      }
    }
  }'
# → open the returned `url` in a browser to walk the Publisher through KYC
```

> ⚠️ **Don't use the v1 `/v1/account_links` endpoint here.** It only handles the `recipient` configuration and silently skips the `merchant`/`card_payments` KYC — which means `on_behalf_of` charges fail later. Only the v2 endpoint's `configurations` array runs both KYC tracks in one flow.

**3. Check the account** once the Publisher has gone through KYC:

```bash
curl https://api.stripe.com/v1/accounts/acct_PUBLISHER \
  -u "$STRIPE_KEY:"
# Look at: capabilities.card_payments / capabilities.transfers (active vs pending)
# and requirements.currently_due for any outstanding KYC items.
```

Save the returned `acct_PUBLISHER` on the Publisher's row in your DB.

### Step 2.1b — Enable Stripe Tax on the Connected Account (required for Publisher-as-MoR)

> **Does the account-creation call in Step 2.1 need a tax parameter? No — it's correct as-is.** You declare MoR per-charge with `on_behalf_of` (Step 2.5), not on the account, and there's no "tax" capability to request when creating it. Tax lives in a **separate subsystem** because KYC and tax registration are different legal domains — Stripe can't pre-fill a Publisher's tax registrations during KYC (they're the Publisher's own legal facts and change over time). For the full *why*, see `tax.md` §7.1.

> **What can vs. can't ride along with KYC:** the Publisher's *head-office address + default tax code* **can** be collected inside the hosted onboarding flow if you enable **Dashboard → Connect → Onboarding options → Tax**. The **registrations themselves cannot** — they're added afterward via the calls below (or, in production, the embedded Tax components). So Step 2.1b is the post-KYC remainder, not the whole story.

Run the three calls below **after** the Publisher finishes KYC. Each one targets the Connected Account with the `Stripe-Account: acct_PUBLISHER` header.

**1. Set the head office (the Publisher's origin address)** — required before you can add any registration:

```bash
curl https://api.stripe.com/v1/tax/settings \
  -u "$STRIPE_KEY:" \
  -H "Stripe-Account: acct_PUBLISHER" \
  -d "head_office[address][country]=DE" \
  -d "head_office[address][postal_code]=70619" \
  -d "head_office[address][city]=Stuttgart" \
  -d "head_office[address][line1]=Hauptstrasse 2" \
  -d "defaults[tax_code]=txcd_10103000" \
  -d "defaults[tax_behavior]=exclusive"
# Without a head office, adding a registration returns invalid_request_error.
# defaults[tax_code]: the connected account's default product tax category
#   (txcd_10103000 = Software as a service; verify against Stripe's tax-code list).
# defaults[tax_behavior]: exclusive (price is net, VAT added) or inclusive (price includes VAT) — see tax.md §5.
```

**2. Add a tax registration for each jurisdiction the Publisher is obligated in:**

```bash
# Germany (domestic):
curl https://api.stripe.com/v1/tax/registrations \
  -u "$STRIPE_KEY:" \
  -H "Stripe-Account: acct_PUBLISHER" \
  -d "country=DE" \
  -d "country_options[de][type]=standard" \
  -d "active_from=now"

# EU One-Stop-Shop (covers cross-border B2C across the EU) — add when the Publisher needs it:
curl https://api.stripe.com/v1/tax/registrations \
  -u "$STRIPE_KEY:" \
  -H "Stripe-Account: acct_PUBLISHER" \
  -d "country=DE" \
  -d "country_options[de][type]=oss_union" \
  -d "active_from=now"

# Switzerland is non-EU — a separate registration with its own type; see tax.md §5.
```

**3. Verify the account is tax-ready:**

```bash
curl https://api.stripe.com/v1/tax/settings \
  -u "$STRIPE_KEY:" \
  -H "Stripe-Account: acct_PUBLISHER"
# Ready when status = "active". If "pending", the head office or a registration is missing.
```

> In production, **don't hard-code registrations** — they're legal facts only the Publisher knows. Let Publishers enter steps 1–2 themselves through the Connect **embedded "Tax settings" / "Tax registrations" components** in the partner portal. See §5.4 and `tax.md` §7.

> **Go-live gate:** don't let a Publisher sell into a country they aren't registered in. If `on_behalf_of` is set and there's no matching registration, Stripe quietly returns `tax_amount = 0.00` / `not_collecting` and the invoice goes out with 0% VAT — non-compliant, with no error to warn you.

### Step 2.2 — Create a Product (when an app is approved)

No `Stripe-Account` header. The Publisher link is metadata only — this is the core green-model invariant.

All values should come from the app record at approval time:
- `APP_NAME` — the app's display name. Shown on the **customer's invoice and receipt**, so it should match exactly what the Tenant recognizes.
- `APP_DESCRIPTION` — the app's description. Shown in the Stripe Dashboard and on hosted invoice views; used by ops/support to identify what a charge is for.
- `APP_ID` / `PUBLISHER_ID` — your internal IDs for the app and its owning Publisher.

```bash
curl https://api.stripe.com/v1/products \
  -u "$STRIPE_KEY:" \
  -d "name=APP_NAME" \
  -d "description=APP_DESCRIPTION" \
  -d "metadata[appId]=APP_ID" \
  -d "metadata[publisherId]=PUBLISHER_ID"
# → capture prod_xxx
```

Verify it lives on the Platform, not the Connected Account:

```bash
# Retrieve from Platform — should exist:
curl https://api.stripe.com/v1/products/prod_xxx \
  -u "$STRIPE_KEY:"

# Retrieve scoped to the Connected Account — must NOT appear here:
curl -G https://api.stripe.com/v1/products \
  -u "$STRIPE_KEY:" \
  -H "Stripe-Account: acct_PUBLISHER_ID"
```

Persist `prod_xxx ↔ app_12345` in your DB.

### Step 2.3 — Create a Price

One Price per (tier × interval). If an app has two tiers (Basic, Pro) each with monthly and annual billing, that's **4 separate Price creation calls**, all pointing at the same `prod_xxx`.

The Price object represents the billing plan only — the trial period (if any) and the Connect parameters (`on_behalf_of`, `transfer_data`, `application_fee_percent`) are set later at Checkout Session time (Step 2.5), not here.

Metadata fields:
- `plan_tier` — the plan name as defined by the Publisher (e.g. `Basic`, `Pro`, `Enterprise`)
- `plan_interval` — two valid values only: `monthly` or `annual`

**Monthly:**
```bash
curl https://api.stripe.com/v1/prices \
  -u "$STRIPE_KEY:" \
  -d "product=prod_xxx" \
  -d "unit_amount=3000" \
  -d "currency=eur" \
  -d "recurring[interval]=month" \
  -d "metadata[appId]=APP_ID" \
  -d "metadata[plan_tier]=Basic" \
  -d "metadata[plan_interval]=monthly"
# → capture price_xxx
```

**Annual:**
```bash
curl https://api.stripe.com/v1/prices \
  -u "$STRIPE_KEY:" \
  -d "product=prod_xxx" \
  -d "unit_amount=28800" \
  -d "currency=eur" \
  -d "recurring[interval]=year" \
  -d "metadata[appId]=APP_ID" \
  -d "metadata[plan_tier]=Basic" \
  -d "metadata[plan_interval]=annual"
# → capture price_xxx
```

Verify:

```bash
curl https://api.stripe.com/v1/prices/price_xxx \
  -u "$STRIPE_KEY:"
# Confirm: recurring.interval, unit_amount, currency

# List all prices for a product:
curl -G https://api.stripe.com/v1/prices \
  -u "$STRIPE_KEY:" \
  -d "product=prod_xxx"
```

Persist each `price_xxx` against the plan record in your DB (tier + interval as the key).

### Step 2.4 — Create a Customer (on first Tenant purchase)

One Customer per Tenant, forever — regardless of which Publisher's apps they subscribe to. This is the whole point of the centralized model.

```bash
curl https://api.stripe.com/v1/customers \
  -u "$STRIPE_KEY:" \
  -d "name=Mustermann GmbH" \
  --data-urlencode "email=billing@mustermann.de" \
  -d "address[city]=Stuttgart" \
  -d "address[country]=DE" \
  -d "address[line1]=Hauptstrasse 2" \
  -d "address[postal_code]=70619" \
  -d "metadata[tenantId]=TENANT_ID"
# → capture cus_xxx
```

**Field Reference:**

| Field | What it is | Importance | Impact in Stripe | Source |
|---|---|---|---|---|
| `name` | Tenant's name | Important | Appears on every invoice and receipt the Tenant receives. If wrong, the Tenant's accounting team can't match it to their own records. | Tenant Name from Account Service |
| `email` | Billing contact email address | Important | Stripe sends payment receipts, failed payment alerts, and dunning emails here. If missing or wrong, the Tenant is blind to billing events. | `email` from KC |
| `address[line1]` | Street address | Required for tax | Used by Stripe Tax to compute the correct VAT rate. Also printed on invoices — legally required for B2B invoices in DE/AT/CH. | `street` from KC |
| `address[city]` | City | Required for tax | Part of the address Stripe Tax uses for jurisdiction determination. | `city` from KC |
| `address[postal_code]` | Postal / ZIP code | Required for tax | Part of the address Stripe Tax uses for jurisdiction determination. | `postal_code` from KC |
| `address[country]` | ISO 3166-1 alpha-2 country code | Required for tax | The most important address field — determines which VAT regime applies (DE 19%, AT 20%, CH 8.1%). Wrong value = wrong tax. | `country_iso` from KC |
| `metadata[tenantId]` | Internal Tenant ID (primary key) | Critical | Your internal join key. When a webhook fires (`invoice.paid`, `invoice.payment_failed`, etc.), this is the only way to know which Tenant the event belongs to without an extra DB lookup. Without it, webhook routing breaks. | JTL Cloud Platform Tenant record |

Always check before creating — never create a duplicate Customer for the same Tenant:

```bash
curl -G https://api.stripe.com/v1/customers \
  -u "$STRIPE_KEY:" \
  --data-urlencode "email=billing@mustermann.de"
# → if a row is returned, reuse that cus_xxx
```

### Step 2.5 — Create the Checkout Session

This is where all three parameters are bound together under `subscription_data`:

```bash
curl https://api.stripe.com/v1/checkout/sessions \
  -u "$STRIPE_KEY:" \
  --data-urlencode "success_url=https://hub.jtl-cloud.com/apps/{CHECKOUT_SESSION_ID}/success" \
  --data-urlencode "cancel_url=https://hub.jtl-cloud.com/apps/cancel" \
  -d "mode=subscription" \
  -d "customer=cus_PLATFORM_CUSTOMER_ID" \
  -d "line_items[0][price]=price_PLATFORM_PRICE_ID" \
  -d "line_items[0][quantity]=1" \
  -d "subscription_data[on_behalf_of]=acct_PUBLISHER_ID" \
  -d "subscription_data[transfer_data][destination]=acct_PUBLISHER_ID" \
  -d "subscription_data[application_fee_percent]=15" \
  -d "subscription_data[metadata][tenantId]=TENANT_ID" \
  -d "subscription_data[metadata][appId]=APP_ID" \
  -d "subscription_data[metadata][publisherId]=PUBLISHER_ID"
# → capture cs_xxx and the `url` field
```

Open the returned `url` in a browser and pay with test card `4242 4242 4242 4242`, any future expiry, any CVC. Stripe redirects to `success_url` and creates the Subscription.

**Field reference:**

| Stripe field | Variable | What it is |
|---|---|---|
| `success_url` | `https://hub.jtl-cloud.com/apps/{CHECKOUT_SESSION_ID}/success` | Where Stripe redirects after successful payment. `{CHECKOUT_SESSION_ID}` is a literal placeholder — Stripe fills it in at redirect time. |
| `cancel_url` | `https://hub.jtl-cloud.com/apps/cancel` | Where Stripe redirects if the Tenant abandons checkout. |
| `mode` | `subscription` | Fixed. Tells Stripe this is a recurring subscription, not a one-time charge. |
| `customer` | `cus_PLATFORM_CUSTOMER_ID` | The Stripe Customer ID for the Tenant. Created in Step 2.4 |
| `line_items[0][price]` | `price_PLATFORM_PRICE_ID` | The price of the product the Tenant is buying. Created in Step 2.3. |
| `line_items[0][quantity]` | `1` | Number of licences. Always `1` for single-tenant SaaS. |
| `subscription_data[on_behalf_of]` | `acct_PUBLISHER_ID` | The Publisher's Connected Account. Makes the Publisher the legal seller — their name appears on the Tenant's bank statement. |
| `subscription_data[transfer_data][destination]` | `acct_PUBLISHER_ID` | Same account as `on_behalf_of`. Tells Stripe to send the net revenue to the Publisher after each payment. |
| `subscription_data[application_fee_percent]` | `15` | JTL's commission percentage. Stripe keeps this from each payment before transferring the rest to the Publisher. |
| `subscription_data[metadata][tenantId]` | `TENANT_ID` | The Tenant's internal UUID. Used to route webhooks (`invoice.paid`, etc.) back to the right Tenant. |
| `subscription_data[metadata][appId]` | `APP_ID` | The internal App UUID. Links the Subscription to the app being purchased. |
| `subscription_data[metadata][publisherId]` | `PUBLISHER_ID` | The internal Publisher UUID. Links the Subscription to the Publisher who owns the app. |

Non-obvious points that have bitten teams before:
- The three parameters belong under **`subscription_data`**, not `payment_intent_data`. The destination-charge docs default to one-time payments and will mislead you.
- Use **`application_fee_percent`** (recurring, calculated every cycle) — not `application_fee_amount` (flat, per-charge only, does not recur).
- Create the Customer explicitly in Step 2.4. Don't rely on `customer_creation` in the Session, or you lose the unified `cus_xxx` across Publishers.

### Step 2.6 — Verify the money flow

After Checkout completes, walk the object chain to confirm the three parameters flowed correctly:

```bash
# 1. Subscription — parameters stored here, recur forever
curl -G https://api.stripe.com/v1/subscriptions \
  -u "$STRIPE_KEY:" -d customer=cus_xxx -d limit=1
# Confirm: on_behalf_of, transfer_data.destination, application_fee_percent=15

# 2. First invoice
curl -G https://api.stripe.com/v1/invoices \
  -u "$STRIPE_KEY:" -d subscription=sub_xxx -d limit=1
# Confirm: status=paid, amount_paid=3000

# 3. Charge — shows the money split
curl https://api.stripe.com/v1/charges/ch_xxx \
  -u "$STRIPE_KEY:"
# Confirm: amount=3000, application_fee → fee_xxx (~€4.50), transfer → tr_xxx (~€24.95)

# 4. Transfer — money to Publisher
curl -G https://api.stripe.com/v1/transfers \
  -u "$STRIPE_KEY:" -d limit=1
# Confirm: destination=acct_PUBLISHER_ID

# 5. ApplicationFee — JTL's commission
curl -G https://api.stripe.com/v1/application_fees \
  -u "$STRIPE_KEY:" -d limit=1
# Confirm: amount=450 (15% of €30)
```

If all five line up, the full green-model money flow is verified end-to-end.

**Webhook events to handle** (all arrive at the single Platform endpoint):

| Event | Action |
|---|---|
| `checkout.session.completed` | Provision access |
| `customer.subscription.created` | Record `sub_xxx` against the Tenant |
| `invoice.paid` | Confirm continued access; finalize commission accounting |
| `invoice.payment_failed` | Start dunning; suspend access |
| `customer.subscription.deleted` | Deprovision |

---

## Concrete ticket changes

### CP-3100 "Sync App Subscription Plan to Stripe after approval"
**Before:** `POST /v1/products` with `Stripe-Account: acct_publisher_xxx`.  
**After:** `POST /v1/products` without the `Stripe-Account` header, with `metadata.publisher_account=acct_publisher_xxx`. Same for the Price create. The owning Publisher's `acct_xxx` goes into product metadata, not authentication headers.

### Checkout session creation
Must construct the request with **`subscription_data`** wrapping the three Connect parameters (Step 2.5). If you have existing `payment_intent_data` logic for one-time charges, do not copy that pattern here.

### Application fee logic
Delete any code issuing a separate `POST /v1/transfers` after a charge. Replace with `subscription_data[application_fee_percent]=15` on the Checkout Session. Stripe computes and routes the fee automatically every billing cycle.

### Customer creation flow
1. Look up Tenant in `TenantStripeCustomers`.
2. If found → reuse `cus_xxx`.
3. If not → `POST /v1/customers` on Platform → store mapping → use new `cus_xxx`.
4. Pass `customer=cus_xxx` into the Checkout Session.

Never let Checkout auto-create a Customer — you'd lose the cross-Publisher unification.

---

## Edge cases

### 5.1 — Cross-border settlement
If a Publisher's Connected Account is in a different region from the Platform, Stripe requires `on_behalf_of` to settle in the Publisher's country. You're already including it, so you're covered — but removing it (e.g. when JTL becomes MoR) means all settlement moves to the Platform's country.

### 5.2 — Publisher disconnect
If a Publisher leaves and you deauthorize their Connected Account, subscriptions do **not** auto-cancel. With `on_behalf_of` active, invoices for those subscriptions become drafts with `auto_advance=false`. Handle explicitly: reconnect, migrate to a different `transfer_data[destination]`, or cancel and refund the Tenant.

### 5.3 — Refunds and the application fee
By default, refunds do **not** reverse the application fee — JTL keeps its commission. Set `refund_application_fee=true` on the refund to reverse it. Decide and document this policy before launch.

### 5.4 — Stripe Tax (with Publisher-as-MoR)
Enabling Stripe Tax on the Platform account and adding `automatic_tax[enabled]=true` to the Checkout Session is **necessary but not sufficient** in our confirmed posture. Because every Session sets `on_behalf_of=acct_PUBLISHER_ID`, Stripe treats the **Publisher** as the tax merchant of record and calculates VAT against the **Publisher's** Stripe Tax registrations and origin address — not the Platform's.

Concretely, this means:
- Each Publisher must enable Stripe Tax in their own Connected Account and add a tax registration for every jurisdiction where they're obligated. Since Publishers use the **Express** dashboard, surface this through the Connect embedded **Tax registrations** / **Tax settings** components rather than expecting them to find it themselves.
- If a Publisher has **no registration** in the buyer's jurisdiction, Stripe returns `tax_amount = 0.00` with reason `not_collecting` — i.e. no VAT is charged and the invoice is silently non-compliant. There is no Platform-level fallback while `on_behalf_of` is set.
- On the Checkout Session, set `automatic_tax[enabled]=true` and assign liability to the Publisher: `automatic_tax[liability][type]=account` and `automatic_tax[liability][account]=acct_PUBLISHER_ID`. Also enable `tax_id_collection[enabled]=true` so B2B customers can supply a VAT ID and trigger reverse charge.
- The Publisher's Connected Account must already be tax-ready (`tax.settings.status=active` with the right registrations) — that setup is **Step 2.1b**, done once per Publisher after onboarding, not at Checkout time.
- **EU invoice-issuer rule:** in the EU the invoice issuer must match the entity liable for tax. With `on_behalf_of`, Checkout already issues the invoice under the Connected Account (confirmed in our data: `issuer.account` = the Publisher), so this is handled automatically — no extra `invoice_settings[issuer]` needed on the Session.
- Set a `tax_code` on each Product (e.g. the SaaS / electronically-supplied-services code) so Stripe knows the tax category; otherwise it falls back to the account default set in Step 2.1b.

The centralized "configure tax once on the Platform" model only becomes available if/when JTL switches to JTL-as-MoR (drop `on_behalf_of`) — see §5.1 and Open Question #4.

### 5.5 — Platform constraints
- The Platform **cannot update or cancel a Subscription it didn't create**. Safe in the green model since the Platform creates everything — but don't let any Subscriptions be created on Connected Accounts.
- The Platform **cannot add `application_fee_amount`** to an Invoice it didn't create. Same reasoning.

---

## Smoke-test sequence

After implementation, run this in Sandbox end-to-end:

1. Create 2 Publishers (Connected Accounts).
2. Create 1 App per Publisher → 2 Products + 2 Prices on the Platform.
3. Create 1 Tenant → 1 Customer on the Platform.
4. Tenant subscribes to App A → confirm `sub_xxx` on Platform with all three green params.
5. Same Tenant subscribes to App B → confirm second `sub_xxx` reusing the same `cus_xxx`.
6. Open the Customer Portal for `cus_xxx` → should show **both** subscriptions.
7. Force-advance billing with `stripe billing test-clocks` → confirm two separate invoices, two transfers to separate Publisher accounts, two ApplicationFees on the Platform balance.

If step 6 shows one unified portal with both subscriptions, the customer-experience benefit is real and verified end-to-end.

---

## Open questions

1. **Commission rate.** Confirm with Finance — 15% (product mockups) vs. 50% (Stripe example). Harden as a config value, ideally per-Publisher to allow future negotiation.
2. **`refund_application_fee` policy.** See §5.3.
3. **Stripe Tax enablement.** See §5.4. **Note the gotcha:** with Publisher-as-MoR confirmed, enabling Stripe Tax on the Platform alone does nothing — tax is calculated per-Publisher against their own registrations. Plan for collecting and surfacing Publisher tax registrations (embedded Connect components) as part of onboarding, not just a Platform-side toggle.
4. **Loss liability with Finance/Legal.** MoR posture is now **confirmed as Publisher-as-MoR** (`on_behalf_of` set on every Session). However, the Express Dashboard forces `losses_collector=application`, so JTL carries chargeback and negative-balance liability at launch — even while the Publisher is MoR for the charge. These are separate axes. If Finance/Legal need the Publisher to bear losses too, that requires moving to `dashboard=full` or `dashboard=none`. Get this split (Publisher = tax/MoR liability, JTL = loss liability) explicitly acknowledged.
5. **Accounts v2 API version to pin.** The working version verified in the Sandbox is `2025-12-15.preview` (the Solution Plan's original `2025-01-27.acacia` is rejected as invalid). Confirm the GA version string with David Joos before launch.
6. **v2 account in v1 destination charge.** The general guarantee is stated in Stripe docs, but get a one-line sign-off from David Joos that there's no edge case.

---

## Reference links

- [Connect subscriptions](https://docs.stripe.com/connect/subscriptions)
- [Destination charges](https://docs.stripe.com/connect/destination-charges)
- [Checkout Session API — `subscription_data`](https://docs.stripe.com/api/checkout/sessions/create)
- [Subscription `application_fee_percent`](https://docs.stripe.com/api/subscriptions/object#subscription_object-application_fee_percent)
- [Stripe Tax with Connect](https://docs.stripe.com/tax/connect)
- [Tax for platforms (`on_behalf_of`, liability, issuer)](https://docs.stripe.com/tax/tax-for-platforms)
- [Tax Settings API (head office)](https://docs.stripe.com/tax/settings-api)
- [Tax Registrations API](https://docs.stripe.com/tax/registrations-api)
- [Connect webhooks](https://docs.stripe.com/connect/webhooks)
- [Accounts v2 reference](./accounts-v2-reference.md)
