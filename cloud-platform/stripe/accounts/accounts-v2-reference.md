# Accounts v2 — v1→v2 Mapping & Integration Impact

**Context:** JTL creates Connected Accounts using the **Accounts v2 API** and uses v1 for everything else (Products, Prices, Customers, Checkout Sessions, Subscriptions). This document maps the v2 call back to the old v1 shape and covers everything the hybrid split touches.

Sources fetched May 2026:
- [Connect and the Accounts v2 API](https://docs.stripe.com/connect/accounts-v2)
- [Use the Accounts v2 API in your existing integration](https://docs.stripe.com/connect/accounts-v2/migrate-integration)
- [Compare SaaS platform configurations for Accounts v1 and v2](https://docs.stripe.com/connect/accounts-v2/saas-platform-payments-billing)
- [Accounts v2 create reference](https://docs.stripe.com/api/v2/core/accounts/create)
- [Event destinations / thin events](https://docs.stripe.com/event-destinations)

---

## The core conceptual shift

v1 had a single account `type` (`express` / `standard` / `custom`) that bundled together *what the account can do* and *what dashboard it gets*. v2 splits these apart:

- **What it can do** → one or more **configurations**: `merchant` (accept card payments), `recipient` (receive transfers/payouts), `customer` (be billed).
- **What dashboard it gets** → a separate `dashboard` field (`express` / `full` / `none`).
- **Who it is** → a unified `identity` object (`entity_type`, `country`, `business_details`) instead of v1's flat `business_type` + `company[...]` fields.

The old "Express account that can take card payments and receive transfers" becomes in v2: `dashboard=express` + a `merchant` configuration (for `card_payments`) + a `recipient` configuration (for transfers).

---

## Field-by-field mapping: v1 → v2

| v1 | v2 |
|---|---|
| `--type=express` | `dashboard=express` + the merchant/recipient configs below |
| `--country=DE` | `identity[country]=DE` |
| `business_type=company` | `identity[entity_type]=company` |
| `capabilities[card_payments][requested]=true` | `configuration[merchant][capabilities][card_payments][requested]=true` |
| `capabilities[transfers][requested]=true` | `configuration[recipient][capabilities][stripe_balance][stripe_transfers][requested]=true` |
| `metadata[jtl_publisher_id]=...` | `metadata[jtl_publisher_id]=...` (unchanged) |
| *(n/a)* | `defaults[responsibilities][fees_collector]=application` + `losses_collector=application` — v2 makes platform responsibility explicit |
| *(n/a)* | `contact_email=…` — required when a `recipient` configuration is requested |
| *(n/a)* | `include[0]=configuration.merchant` … — v2 only returns configs in the response if listed in `include` |

Note the deeper nesting for transfers: v1 `transfers` → v2 `stripe_balance.stripe_transfers`.

---

## JTL-as-MoR variant (future — recipient only, lighter KYC)

If JTL becomes MoR, drop the `merchant` configuration entirely:

```bash
curl https://api.stripe.com/v2/core/accounts \
  -u "$STRIPE_KEY:" \
  -H "Content-Type: application/json" \
  -H "Stripe-Version: 2025-12-15.preview" \
  -d '{
    "dashboard": "express",
    "contact_email": "partner@acme-software.de",
    "identity": { "country": "DE", "entity_type": "company" },
    "configuration": {
      "recipient": { "capabilities": { "stripe_balance": { "stripe_transfers": { "requested": true } } } }
    },
    "defaults": {
      "responsibilities": { "fees_collector": "application", "losses_collector": "application" }
    },
    "metadata": { "jtl_publisher_id": "pub_acme_software" },
    "include": ["configuration.recipient", "identity"]
  }'
# No merchant config → no card_payments KYC burden.
# contact_email still required (recipient config).
# losses/fees_collector still both 'application' because dashboard=express.
```

---

## Integration impact: what the v2-for-accounts / v1-for-everything-else split touches

### 1. Compatibility with the v1 destination-charge flow ✅

From the Stripe docs:
> "You can call `/v2/core/accounts` endpoints for Accounts created using API v1, and call `/v1/accounts` endpoints for Accounts created using API v2."
> "If you reference v2 objects in v1 endpoints, the response returns the v2 data in the v1 object structure."

A v2-created `acct_xxx` works unchanged in the v1 Checkout / destination-charge flow — `on_behalf_of`, `transfer_data[destination]`, `application_fee_percent` are byte-for-byte the same. **The Steps 2.2–2.6 money flow does not change.**

> ⚠️ The docs don't show a destination-charge example specifically with a v2 account. Get a one-line sign-off from David Joos that there's no edge case before launch.

### 2. Capability names are renamed and re-nested ✅

| v1 | v2 |
|---|---|
| `transfers` | `stripe_balance.stripe_transfers` |
| `payouts` | `stripe_balance.payouts` |
| `card_payments` | `configuration.merchant.capabilities.card_payments` |

### 3. Webhooks — new endpoint required ⚠️

Stripe's recommendation: set up new endpoints to listen to v2 events before migrating.

- v2 Accounts emit **both** v1 (snapshot) events **and** v2 (thin) events. Thin events carry a smaller payload — you fetch the full object separately.
- **Scope flips:** v2 events use the **Your account** scope, while v1 events use the **Connected accounts** scope, even when triggered by the same v2 Account. Account-lifecycle events that today arrive as `account.updated` will also arrive as v2 thin events like `v2.core.account[configuration.recipient].updated`.
- **Good news for JTL:** the billing events the green model depends on — `checkout.session.completed`, `customer.subscription.created`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.deleted` — are **v1 events and are unaffected**. The v2 event work is scoped to account lifecycle/onboarding only. This is an **additive** endpoint, not a rewrite of the billing webhook handler.

Stripe recommends sequencing the webhook endpoint setup **first**, before switching account creation to v2.

### 4. Account ⇄ Customer unification — N/A for JTL

v2's headline feature is that one Account can also act as a customer, eliminating the Account↔Customer mapping table. This applies to Connected Accounts that are *also* billed by the Platform. JTL's Customers are **Tenants (buyers)**, not Connected Accounts — so this doesn't apply to the tenant Customer flow. No change to the `TenantStripeCustomers` design.

### 5. API version ⚠️

The v2 endpoints require an explicit preview version header. The version verified working against the Sandbox is **`Stripe-Version: 2025-12-15.preview`** — the Solution Plan's original `2025-01-27.acacia` is rejected as invalid.

Confirm the GA (non-preview) version string to pin with David Joos before launch, and verify the SDK in use supports it.

### 6. Hard v1-only fallbacks ✅

- Payout settings must be managed via v1 (`/v1/accounts`), not v2.
- Accounts connected via OAuth must use v1.
- A new v1 account may take up to 10 minutes before it's usable with a v2 endpoint.

---

## Summary: what actually needs to change

| Area | Changes under v2? | Effort |
|---|---|---|
| Steps 2.2–2.6 money flow (Checkout, destination charge, app fee) | No — v2 accounts work in v1 charge flow | None |
| `accounts create` call | Yes — configurations + identity shape (mapped above) | Small |
| Capability names (`transfers` → `stripe_balance.stripe_transfers`) | Yes | Small |
| Webhooks — new v2 event endpoint for account lifecycle | Yes — additive; billing events unaffected | Medium |
| Tenant Customer flow / `TenantStripeCustomers` | No — unification doesn't apply | None |
| API version | Preview header required | Risk, not effort |

The core money flow is unchanged. The concrete work is: (a) the v2 `accounts create` call shape, (b) renamed capabilities, and (c) a new additive webhook endpoint for account-lifecycle thin events. Sequence webhooks first.
