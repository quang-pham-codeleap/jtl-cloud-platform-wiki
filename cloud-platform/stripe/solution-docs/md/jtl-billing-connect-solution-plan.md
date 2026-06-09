# Stripe Solution Plan for JTL

| Author | Last Updated | Reviewed with | Status |
|--------|--------------|---------------|--------|
| David Joos | Mar 31, 2026 | | In Progress |

> **📚 Document map** (JTL Stripe integration notes):
> - [`overview.md`](../../overview.md) — high-level overview: red vs green data model + the real benefits
> - **`jtl-billing-connect-solution-plan.md`** — original Stripe Solution Plan (David Joos) *(this document — the source of record)*
> - [`summary.md`](./summary.md) — one-page summary of this plan
> - [`implementation-guide.md`](../../implementation-guide.md) — step-by-step implementation guide (the *how*)
> - [`tax.md`](../../tax.md) — detailed DACH + EU VAT tax handling (the tax deep-dive)

## Stripe Stakeholders

| Role | Name and Contact Details |
|------|--------------------------|
| Account Executive(s) | Jens Natelberg, Eyüp Akkurt |
| Solution Architect(s) | David Joos |
| Industries / | Payments SaaS Platform |
| Professional Services | Christian Stock, Kasia Luczak |

---

## Business Objectives

JTL Software is building a new App Store to replace its fragmented Extension Store, aiming to provide a unified, scalable platform for its **15,000+ merchants** and **1,000+ partners**. The primary goal is to centralize payments, automate subscription management, and calculate JTL's commissions for partners, moving away from the current complex system where each partner manages their own PayPal account. This solution will create a seamless experience for merchants and enable JTL to monetize its ecosystem effectively.

## Technical Requirements

- **Business model:** A multi-party marketplace platform using Stripe Connect.
- **Merchant of record:** Third-party partners (connected accounts) must be the Merchant of Record (MoR) for their app sales. JTL should have the flexibility to become MoR at a later stage.
- **Charge type:** The integration must use **destination charges** with the `on_behalf_of` parameter to correctly assign MoR status to the partner.
- **Data model:** Customer, product, and subscription data will be centralized and stored on the JTL platform's Stripe account.
- **Partner experience:** Partners will be onboarded to **Express accounts** for a streamlined experience with transaction visibility.
- **Core functionality:** The solution must support monthly recurring subscriptions using **Stripe Billing** and automated tax calculation with **Stripe Tax**.
- **Markets:** The initial launch will target Germany (DACH region), with plans to expand across the EU.
- **Timeline:** An MVP is required by the Partner Convention in **May 2026**, with a full rollout by the **end of 2026**.

---

## Solution Architecture

> **Testing note (added for hands-on verification):** every `curl` in this section runs against your Stripe **Sandbox** as-is — just substitute your **test** secret key (`sk_test_…`) for the `sk_live_PLATFORM_SECRET_KEY` / `REPLACE_WITH_YOUR_SECRET_KEY` placeholders, and pay with the test card `4242 4242 4242 4242` (any future expiry, any CVC). Each step below now pairs the original **create** call with a **🔍 Feel the data** call so you can immediately retrieve/list what you just created and watch the object chain build up.

### Onboarding of partner as connected accounts

JTL will have to onboard connected accounts before enabling payments and payouts via the app marketplace.

You will have to request the following capabilities:

- **Merchant** ("Card payments capability") if partner is MoR
- **Recipient** ("Stripe Transfers")

We recommend the use of our **accounts v2 API** to create and onboard partners as connected accounts. You can find the comprehensive documentation on accounts v2 here.

#### Example Account Creation

```bash
curl https://api.stripe.com/v2/core/accounts \
  -u "sk_live_PLATFORM_SECRET_KEY:" \
  -H "Content-Type: application/json" \
  -H "Stripe-Version: 2025-01-27.acacia" \
  -d '{
    "country": "DE",
    "dashboard": "express",
    "identity": {
      "country": "DE",
      "entity_type": "company"
    },
    "configuration": {
      "recipient": {
        "capabilities": {
          "stripe_balance": {
            "stripe_transfers": {
              "requested": true
            }
          }
        }
      },
      "merchant": {
        "capabilities": {
          "card_payments": {
            "requested": true
          }
        }
      }
    },
    "defaults": {
      "currency": "eur"
    }
  }'
# → returns object "v2.core.account" with id acct_… (the partner's connected account)
```

**🔍 Feel the data — retrieve the account you just created:**

```bash
curl -G https://api.stripe.com/v2/core/accounts/acct_PARTNER_ID \
  -u "sk_test_PLATFORM_SECRET_KEY:" \
  -H "Stripe-Version: 2025-01-27.acacia" \
  -d "include[]=configuration.merchant" \
  -d "include[]=configuration.recipient" \
  -d "include[]=identity" \
  -d "include[]=requirements"
# Read configuration.merchant.capabilities.card_payments and
# configuration.recipient.capabilities.stripe_balance.stripe_transfers (requested vs active),
# and requirements.* to see what KYC the partner still owes before they can be paid out.
```

#### Example Hosted Onboarding URL

```bash
curl https://api.stripe.com/v2/core/account_links \
  -u "sk_live_PLATFORM_SECRET_KEY:" \
  -H "Content-Type: application/json" \
  -H "Stripe-Version: 2025-01-27.acacia" \
  -d '{
    "account": "acct_PARTNER_ID",
    "use_case": {
      "type": "account_onboarding",
      "account_onboarding": {
        "refresh_url": "https://hub.jtl.com/partner/onboarding/refresh",
        "return_url": "https://hub.jtl.com/partner/onboarding/complete",
        "collection_options": {
          "fields": "eventually_due"
        }
      }
    }
  }'
# → returns a `url`; open it in a browser to walk the partner's Express KYC flow.
```

**🔍 Feel the data — re-check the account after the partner finishes onboarding:**

```bash
# Re-run the account retrieve above. Once KYC is done you should see
# capabilities flip requested → active and requirements.currently_due empty.
curl -G https://api.stripe.com/v2/core/accounts/acct_PARTNER_ID \
  -u "sk_test_PLATFORM_SECRET_KEY:" \
  -H "Stripe-Version: 2025-01-27.acacia" \
  -d "include[]=configuration.recipient" \
  -d "include[]=requirements"
```

### Facilitate subscriptions

#### Product/Price Management

JTL will allow for simple monthly/yearly subscriptions with the launch in May. As outlined in our scoping session, we recommend the creation of products and prices on the **platform account**. JTL will **not** create products and prices on the connected accounts.

**Step 1 — Create product**

```bash
curl https://api.stripe.com/v1/products \
  -u "sk_live_PLATFORM_SECRET_KEY:" \
  -d "name=Super Inventory Manager" \
  -d "description=Advanced inventory management extension for JTL-Wawi" \
  -d "metadata[jtl_app_id]=app_12345" \
  -d "metadata[partner_account]=acct_PARTNER_ID" \
  -d "metadata[partner_name]=Acme Software GmbH"
# → capture prod_…
```

**🔍 Feel the data — confirm the product lives on the PLATFORM, not the connected account:**

```bash
curl https://api.stripe.com/v1/products/prod_ABC123 \
  -u "sk_test_PLATFORM_SECRET_KEY:"
# Confirm metadata.partner_account is set (the partner link is metadata only).

# Same list scoped to the connected account — prod_… must NOT appear here:
curl -G https://api.stripe.com/v1/products \
  -u "sk_test_PLATFORM_SECRET_KEY:" \
  -H "Stripe-Account: acct_PARTNER_ID"
```

**Step 2 — Create price**

```bash
curl https://api.stripe.com/v1/prices \
  -u "sk_live_PLATFORM_SECRET_KEY:" \
  -d "product=prod_ABC123" \
  -d "unit_amount=3000" \
  -d "currency=eur" \
  -d "recurring[interval]=month" \
  -d "metadata[jtl_app_id]=app_12345" \
  -d "metadata[plan_type]=monthly"
# → capture price_…
```

**🔍 Feel the data — list every price for this product (handy once you add a yearly plan):**

```bash
curl -G https://api.stripe.com/v1/prices \
  -u "sk_test_PLATFORM_SECRET_KEY:" \
  -d product=prod_ABC123
# Confirm recurring.interval=month, unit_amount=3000, currency=eur.
```

### Payment + Subscription Creation

#### Step 1 – Customer creation

As a general best practice, we recommend creating a customer object ahead of the subscription creation for consistent customer data management.

```bash
curl https://api.stripe.com/v1/customers \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  -d name="Kundenname" \
  --data-urlencode email="option2@test.com" \
  -d "address[city]"=Stuttgart \
  -d "address[country]"=DE \
  -d "address[line1]"="Hauptstrasse 2" \
  -d "address[line2]"= \
  -d "address[postal_code]"=70619 \
  -d "address[state]"="Baden-Württemberg"
# → capture cus_…
```

**🔍 Feel the data — simulate the "look up before create" check your endpoint must do:**

```bash
curl -G https://api.stripe.com/v1/customers \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  --data-urlencode email="option2@test.com"
# If this returns a row, REUSE that cus_… — never create a second customer for the
# same tenant, or you lose the "one customer reused across all partners" benefit.
```

#### Step 2 – Stripe Checkout & Subscription Creation

We recommend Stripe Checkout for an easy and optimized Checkout and subscription creation flow.

```bash
curl https://api.stripe.com/v1/checkout/sessions \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  --data-urlencode success_url="URL" \
  -d customer=customer_ID \
  -d "line_items[0][price]"=price_1TD3N0PM469EdcdJlG2yI68c \
  -d "line_items[0][quantity]"=1 \
  -d mode=subscription \
  -d "subscription_data[transfer_data][amount_percent]"=50 \
  -d "subscription_data[transfer_data][destination]"=CA_ID \
  -d "subscription_data[on_behalf_of]"=CA_ID
# → returns cs_… and a `url`. Open the url and pay with 4242 4242 4242 4242.
```

**🔍 Feel the data — after paying, walk the whole object chain the charge produced:**

```bash
# 1. The subscription that resulted (the params are stored here and recur forever)
curl -G https://api.stripe.com/v1/subscriptions \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  -d customer=customer_ID -d limit=1
# Confirm transfer_data.destination, transfer_data.amount_percent, on_behalf_of.

# 2. The charge — shows the actual money split
curl -G https://api.stripe.com/v1/charges \
  -u REPLACE_WITH_YOUR_SECRET_KEY: -d limit=1
# Read off on_behalf_of, transfer (→ tr_…) and application_fee (→ fee_…).

# 3. The transfer routed to the partner's connected account
curl -G https://api.stripe.com/v1/transfers \
  -u REPLACE_WITH_YOUR_SECRET_KEY: -d limit=1
# Confirm destination=CA_ID and the amount = charge − fee − Stripe fee.
```

### Tax Handling

Invoices generated by Stripe can show the applicable VAT. In order to calculate the appropriate tax rate, you can either apply a manual tax rate or use Stripe Tax to automatically apply the appropriate tax rate.

#### Option 1 – Use Tax Rates

Create a tax rate via the Dashboard or API and add the tax rate in your Checkout session.

**Create Tax Rate**

```bash
curl https://api.stripe.com/v1/tax_rates \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  -d display_name=USt \
  -d inclusive=true \
  -d percentage=19 \
  -d country=DE
# → capture txr_…
```

**🔍 Feel the data — list your tax rates to grab the txr_… id for the checkout below:**

```bash
curl -G https://api.stripe.com/v1/tax_rates \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  -d limit=3
# Confirm display_name=USt, percentage=19, country=DE, inclusive=true.
```

**Apply Tax Rate(s) to Checkout**

```bash
curl https://api.stripe.com/v1/checkout/sessions \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  --data-urlencode success_url="https://bob-tinker-123456789.stripedemos.com/scenario/2938bc9b-9c1d-479b-8c0d-e6f53aea7e00?checkout_session={CHECKOUT_SESSION_ID}" \
  -d customer=cus_UFStVlFAXBSONT \
  -d "line_items[0][price]"=price_1TD3N0PM469EdcdJlG2yI68c \
  -d "line_items[0][quantity]"=1 \
  -d "line_items[0][tax_rates][0]"=txr_1TGyFgPM469EdcdJbKCwhndx \
  -d mode=subscription \
  -d "subscription_data[transfer_data][amount_percent]"=50 \
  -d "subscription_data[transfer_data][destination]"=acct_1TECJoArPxFd1PFJ \
  -d "subscription_data[on_behalf_of]"=acct_1TECJoArPxFd1PFJ
```

#### Option 2 – Use Stripe Tax

Please use this documentation to automatically determine tax rates on behalf of your connected accounts.

Once Stripe Tax is enabled on the platform account (Dashboard → **Settings → Tax**, set the platform's origin address and register your tax jurisdictions), you no longer create/attach `tax_rates` manually — you just flip `automatic_tax[enabled]=true` on the Checkout session and Stripe computes the correct VAT from the customer's address.

> ⚠️ **Clarification (added post-review — see [`tax.md`](../../tax.md) §2 / §7):** "enable Stripe Tax once on the platform" is only fully true under **JTL-as-MoR**. Because this plan also requires `on_behalf_of` (Publisher-as-MoR), Stripe Tax actually computes against **each connected account's own registrations** — not the platform's — as the note under the next code block correctly says. So enabling tax on the platform is *necessary but not sufficient*: every Publisher must also set up their own Stripe Tax settings + registrations. This is the single biggest tax gotcha in the plan; `tax.md` covers the why and the per-Publisher onboarding sequence.

**Create checkout with automatic tax:**

```bash
curl https://api.stripe.com/v1/checkout/sessions \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  --data-urlencode success_url="https://hub.jtl.com/success?checkout_session={CHECKOUT_SESSION_ID}" \
  -d customer=cus_UFStVlFAXBSONT \
  -d "line_items[0][price]"=price_1TD3N0PM469EdcdJlG2yI68c \
  -d "line_items[0][quantity]"=1 \
  -d mode=subscription \
  -d "automatic_tax[enabled]"=true \
  -d "subscription_data[transfer_data][amount_percent]"=50 \
  -d "subscription_data[transfer_data][destination]"=acct_1TECJoArPxFd1PFJ \
  -d "subscription_data[on_behalf_of]"=acct_1TECJoArPxFd1PFJ
# Note: the customer must have a valid address for automatic_tax to resolve a rate;
# with on_behalf_of, tax is computed against the connected account's registrations.
```

**🔍 Feel the data — confirm tax was computed on the resulting invoice:**

```bash
curl -G https://api.stripe.com/v1/invoices \
  -u REPLACE_WITH_YOUR_SECRET_KEY: \
  -d customer=cus_UFStVlFAXBSONT -d limit=1
# Look at total_tax (or total_taxes[]), tax, and automatic_tax.status=complete.
```

---

## Stripe's Recommended Services

| Service | Recommendation | Reason / Notes |
|---------|----------------|----------------|
| Migration and Implementation | Custom Proserv integration | Complexity of Billing + Connect Integration (+Tax). Custom work needed to build out Invoicing and Subscription UIs needed, and JTL will benefit from Proserv consulting on best practices. |

---

## JTL's Selected Services

> **Not confirmed as of 31.03.2026**

| Service | Selection | Reason / Notes |
|---------|-----------|----------------|
| Migration and Implementation | | |
| Ongoing Technical & Operational Support | | |
