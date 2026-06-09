# Stripe API — Request Log

**Project:** JTL-Software-GmbH, CK-Sandbox  
**Platform account:** `acct_1TXGr0F8IpKa3SF4`

Each request below has a `REQ-XX` ID that matches its response entry in [response-log.md](./response-log.md).

## Pre-requisite

1. Get the API Key from **Stripe Dashboard → Developers → API Keys → Standard Keys**.
2. Export it as an environment variable: `export STRIPE_KEY=sk_test_...`

**Cloud Platform**:
- Environment: Dev
- Tenant: `aoecoduct`
- PublisherId: `cd9ed8a1-5845-4157-92b4-5e7137a1dd96`

---

## REQ-01 — Create a Connected Account (Onboard a Publisher)

**Date:** 2026-06-05  
**Response:** [REQ-01 in response-log.md](./response-log.md#req-01)
**Result:** `acct_1Tes6sF8IpBM1G52`, view the account in Stripe Dashboard: [Stripe Dashboard](https://dashboard.stripe.com/acct_1TXGr0F8IpKa3SF4/test/connect/accounts/acct_1Tes6sF8IpBM1G52/activity)

```bash
curl https://api.stripe.com/v2/core/accounts \
  -u "$STRIPE_KEY:" \
  -H "Content-Type: application/json" \
  -H "Stripe-Version: 2025-12-15.preview" \
  -d '{
    "dashboard": "express",
    "contact_email": "quang.pham@coduct.com",
    "identity": { "country": "DE", "entity_type": "company" },
    "configuration": {
      "merchant":   { "capabilities": { "card_payments": { "requested": true } } },
      "recipient":  { "capabilities": { "stripe_balance": { "stripe_transfers": { "requested": true } } } }
    },
    "defaults": {
      "responsibilities": { "fees_collector": "application", "losses_collector": "application" }
    },
    "metadata": { "publisherId": "cd9ed8a1-5845-4157-92b4-5e7137a1dd96" },
    "include": ["configuration.merchant", "configuration.recipient", "identity"]
  }'
```

---

## REQ-02 — Generate a KYC Onboarding Link

**Date:** 2026-06-05  
**Uses account:** `acct_1Tes6sF8IpBM1G52` (created in REQ-01)  
**Response:** [REQ-02 in response-log.md](./response-log.md#req-02)

```bash
curl https://api.stripe.com/v2/core/account_links \
  -u "$STRIPE_KEY:" \
  -H "Content-Type: application/json" \
  -H "Stripe-Version: 2025-12-15.preview" \
  -d '{
    "account": "acct_1Tes6sF8IpBM1G52",
    "use_case": {
      "type": "account_onboarding",
      "account_onboarding": {
        "configurations": ["recipient", "merchant"],
        "refresh_url": "https://partner.dev.jtl-cloud.com",
        "return_url": "https://partner.dev.jtl-cloud.com",
        "collection_options": { "fields": "eventually_due" }
      }
    }
  }'
```

---

## REQ-03 — Create a Product

**Date:** 2026-06-05
**Uses account:** `acct_1Tes6sF8IpBM1G52` (created in REQ-01)

**Product:**:
- `appId`: `5d2947f7-018c-4ad0-9dd8-3929a5641cf3`
- `publisherId`: `cd9ed8a1-5845-4157-92b4-5e7137a1dd96`
- `appName`: Hello local
- Note: This app has not been approved yet, I'm just testing
**Response**: 
- Full Log: [REQ-03 in response-log.md](./response-log.md#req-03)
- `productId`: `prod_UeBJdwdgw5Iuly`

```bash
curl https://api.stripe.com/v1/products \
  -u "$STRIPE_KEY:" \
  -d "name=Hello local" \
  -d "description=Lorem ipsum." \
  -d "metadata[appId]=5d2947f7-018c-4ad0-9dd8-3929a5641cf3" \
  -d "metadata[publisherId]=cd9ed8a1-5845-4157-92b4-5e7137a1dd96"
```

---

## REQ-04 — Create Price(s) for the Product

**Date:** 2026-06-05
**Uses account:** `acct_1Tes6sF8IpBM1G52` (created in REQ-01)

**Input:**:
- `appId`: `5d2947f7-018c-4ad0-9dd8-3929a5641cf3`
- `productId`: `prod_UeBJdwdgw5Iuly` (created in REQ-03)
- `plan_tier`: Basic
- `plan_interval`: monthly
- `unit_amount`: 3000 (in cents, i.e. €30.00)
- `currency`: EUR
**Response**: 
- Full: [REQ-04 in response-log.md](./response-log.md#req-04)
- `priceId`: `price_1TetJHF8IpKa3SF4sSLjrSQf`

```bash
curl https://api.stripe.com/v1/prices \
  -u "$STRIPE_KEY:" \
  -d "product=prod_UeBJdwdgw5Iuly" \
  -d "unit_amount=3000" \
  -d "currency=eur" \
  -d "recurring[interval]=month" \
  -d "metadata[appId]=5d2947f7-018c-4ad0-9dd8-3929a5641cf3" \
  -d "metadata[plan_tier]=Basic" \
  -d "metadata[plan_interval]=monthly"
```

---

## REQ-05 — Create a Customer

**Date:** 2026-06-05

**Input:**:
- `tenantId`: `5bb461dd-eb16-4704-af56-68c7730d136b` (Coduct Slug)
- `email`: quang.pham@coduct.com
- `address`: Hauptstrasse 2
- `city`: Berlin
- `postal_code`: 10827
- `country`: DE
- **Note:** The info above should come from KC, this is just a test entry
**Response**: 
- Full: [REQ-05 in response-log.md](./response-log.md#req-05)
- `customerId`: `cus_UeCsQ6rAAIxgtQ`

```bash
curl https://api.stripe.com/v1/customers \
  -u "$STRIPE_KEY:" \
  -d "name=Coduct" \
  --data-urlencode "email=quang.pham@coduct.com" \
  -d "address[city]=Berlin" \
  -d "address[country]=DE" \
  -d "address[line1]=Hauptstrasse 2" \
  -d "address[postal_code]=10827" \
  -d "metadata[tenantId]=5bb461dd-eb16-4704-af56-68c7730d136b"
```

---

## REQ-06 — Create a Checkout Session

**Date:** 2026-06-05
**Response:** [REQ-06 in response-log.md](./response-log.md#req-06)
**Result:** `cs_test_a1EMUGCh2nzyX7ox0t2s9LSgGNyLq9OfQK1zdlX0O5RkOJIg2vnuFc5ZET`

**Input:**:
- `customer`: `cus_UeCsQ6rAAIxgtQ` (Created in REQ-05)
- `price`: `price_1TetJHF8IpKa3SF4sSLjrSQf` (Created in REQ-04)
- `seller`: `acct_1Tes6sF8IpBM1G52` (Created in REQ-01) - Connected Account
- `tenantId`: `5bb461dd-eb16-4704-af56-68c7730d136b` (Coduct Slug) - Buyer
- `publisherId`: `cd9ed8a1-5845-4157-92b4-5e7137a1dd96` - Seller
- `appId`: `5d2947f7-018c-4ad0-9dd8-3929a5641cf3` - App being purchased

```bash
curl https://api.stripe.com/v1/checkout/sessions \
  -u "$STRIPE_KEY:" \
  --data-urlencode "success_url=https://hub.jtl-cloud.com" \
  --data-urlencode "cancel_url=https://hub.jtl-cloud.com" \
  -d "mode=subscription" \
  -d "customer=cus_UeCsQ6rAAIxgtQ" \
  -d "line_items[0][price]=price_1TetJHF8IpKa3SF4sSLjrSQf" \
  -d "line_items[0][quantity]=1" \
  -d "subscription_data[on_behalf_of]=acct_1Tes6sF8IpBM1G52" \
  -d "subscription_data[transfer_data][destination]=acct_1Tes6sF8IpBM1G52" \
  -d "subscription_data[application_fee_percent]=15" \
  -d "subscription_data[metadata][tenantId]=5bb461dd-eb16-4704-af56-68c7730d136b" \
  -d "subscription_data[metadata][appId]=5d2947f7-018c-4ad0-9dd8-3929a5641cf3" \
  -d "subscription_data[metadata][publisherId]=cd9ed8a1-5845-4157-92b4-5e7137a1dd96"
```

---

## REQ-07+ — Verify Result

**Date:** 2026-06-05

### Subscription Information

**Input:**
- `customer`: `cus_UeCsQ6rAAIxgtQ` (Created in REQ-05)

**Key Outputs:**
- `on_behalf_of` → Publisher's `acct_xxx` — confirms publisher-as-MoR; their name on Tenant's bank statement
- `transfer_data.destination` → same `acct_xxx` as `on_behalf_of` — confirms auto-payout routing to Publisher
- `application_fee_percent` → `15` — confirms JTL's commission is set and will recur every billing cycle
- `status` → `"active"`
- `invoice_settings.issuer.account` → Publisher's `acct_xxx` — invoice is issued under Publisher's account (not Platform)
- `metadata.tenantId` → **internal tenant UUID** (e.g. `5bb461dd-...`), never a Stripe `cus_xxx` ID — this is the webhook join key
- `metadata.appId` → internal app UUID
- `metadata.publisherId` → internal publisher UUID

**Response:** [REQ-07 — Subscription in response-log.md](./response-log.md#req-07-subscription)

```bash
curl -G https://api.stripe.com/v1/subscriptions \
  -u "$STRIPE_KEY:" -d customer=cus_UeCsQ6rAAIxgtQ -d limit=2
```

---

### Invoice Information

**Input:**
- `subscription`: `sub_1TevLnF8IpKa3SF4eNwiacDI` (from subscription query above)

**Key Outputs:**
- `status` → `"paid"`
- `amount_paid` → matches price in cents (`3000` = €30.00)
- `on_behalf_of` → Publisher's `acct_xxx` — inherited from Subscription
- `issuer.account` → Publisher's `acct_xxx` — invoice legally issued by Publisher
- `parent.subscription_details.metadata.tenantId` → internal tenant UUID — the field webhooks read

**Response:** [REQ-07 — Invoice in response-log.md](./response-log.md#req-07-invoice)

```bash
curl -G https://api.stripe.com/v1/invoices \
  -u "$STRIPE_KEY:" -d subscription=sub_1TevLnF8IpKa3SF4eNwiacDI -d limit=1
```

---

### Payment Expand from Invoices

**Input:**
- `invoice`: `in_1TevLkF8IpKa3SF45i7kRMq9` (from invoice query above)

**Key Outputs:**
- `payments.data[0].payment.type` → `"payment_intent"` — modern invoices use PaymentIntent, not a direct charge
- `payments.data[0].payment.payment_intent` → `pi_3TevLkF8IpKa3SF41UBaHGDX` — use this ID in the next step
- `payments.data[0].status` → `"paid"`

**Response:** [Payment Expand from Invoices in response-log.md](./response-log.md#req-08-payment-expand)

```bash
curl "https://api.stripe.com/v1/invoices/in_1TevLkF8IpKa3SF45i7kRMq9?expand[]=payments" \
    -u "$STRIPE_KEY:"
```

---

### Money Split from Payment Intents

**Input:**
- `payment_intent`: `pi_3TevLkF8IpKa3SF41UBaHGDX` (from Payment Expand query above)

**Key Outputs:**
- `latest_charge.id` → `ch_3TevLkF8IpKa3SF41smUY3ie` — the Charge that executed the payment
- `latest_charge.application_fee` → `fee_1TevLmF8IpBM1G52tait4HjV` — JTL's ApplicationFee retained on the Platform
- `latest_charge.application_fee_amount` → `450` — 15% of 3000 ✓
- `latest_charge.transfer` → `tr_3TevLkF8IpKa3SF41S4P8dW2` — auto-Transfer to Publisher was created
- `latest_charge.destination` → `acct_1Tes6sF8IpBM1G52` — Publisher's Connected Account
- `latest_charge.on_behalf_of` → `acct_1Tes6sF8IpBM1G52` — Publisher is MoR on the charge
- `latest_charge.status` → `"succeeded"`

**Response:** [Money Split from Payment Intents in response-log.md](./response-log.md#req-08-payment-intent)

```bash
curl "https://api.stripe.com/v1/payment_intents/pi_3TevLkF8IpKa3SF41UBaHGDX?expand[]=latest_charge" \
    -u "$STRIPE_KEY:"
```

---

## REQ-09 — Verify Transfer and ApplicationFee

**Date:**  
**Depends on:** REQ-08 — IDs come from `latest_charge` on the PaymentIntent.

### Transfer

**Input:**
- `transfer`: `tr_3TevLkF8IpKa3SF41S4P8dW2`

**Key Outputs:**
- `destination` → `acct_1Tes6sF8IpBM1G52` — money landed at Publisher's Connected Account
- `amount` → `3000` — gross €30.00 transferred; ApplicationFee is charged separately from Publisher's balance (net to Publisher = €25.50)
- `source_transaction` → `ch_3TevLkF8IpKa3SF41smUY3ie` — links back to the originating Charge
- `currency` → `eur`

**Response:** [REQ-09 — Transfer in response-log.md](./response-log.md#req-09-transfer)

```bash
curl https://api.stripe.com/v1/transfers/tr_3TevLkF8IpKa3SF41S4P8dW2 \
    -u "$STRIPE_KEY:"
```

---

### ApplicationFee

**Input:**
- `application_fee`: `fee_1TevLmF8IpBM1G52tait4HjV`

**Key Outputs:**
- `amount` → `450` — €4.50 (15% of €30.00) retained on Platform
- `account` → `acct_1Tes6sF8IpBM1G52` — the Connected Account this fee was charged against
- `currency` → `eur`

**Response:** [REQ-09 — ApplicationFee in response-log.md](./response-log.md#req-09-application-fee)

```bash
curl https://api.stripe.com/v1/application_fees/fee_1TevLmF8IpBM1G52tait4HjV \
    -u "$STRIPE_KEY:"
```
