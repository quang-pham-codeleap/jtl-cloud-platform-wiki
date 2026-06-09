# Stripe API — Response Log

**Project:** JTL-Software-GmbH Tenant, Sandbox: CK-Sandbox  
**Platform account:** `acct_1TXGr0F8IpKa3SF4`

Each response below has a `REQ-XX` ID that matches its request entry in [curl-log.md](./curl-log.md).

---

<a name="req-01"></a>
## REQ-01 — Create a Connected Account (Onboard a Publisher)

**Date:** 2026-06-05  
**Request:** [REQ-01 in curl-log.md](./curl-log.md#req-01)  
**Result:** `acct_1Tes6sF8IpBM1G52`

Key fields to note:
- `card_payments.status: "restricted"` — KYC not yet completed (expected at this stage)
- `stripe_transfers.status: "restricted"` — same reason
- `defaults: null` — the responsibilities block wasn't echoed back; this is normal

```json
{
  "id": "acct_1Tes6sF8IpBM1G52",
  "object": "v2.core.account",
  "applied_configurations": ["recipient", "merchant"],
  "closed": false,
  "configuration": {
    "customer": null,
    "merchant": {
      "applied": true,
      "capabilities": {
        "card_payments": {
          "status": "restricted",
          "status_details": [{ "code": "requirements_past_due", "resolution": "provide_info" }]
        },
        "stripe_balance": {
          "payouts": {
            "status": "restricted",
            "status_details": [{ "code": "requirements_past_due", "resolution": "provide_info" }]
          }
        }
      },
      "card_payments": {
        "decline_on": { "avs_failure": false, "cvc_failure": false }
      },
      "statement_descriptor": { "descriptor": null, "prefix": null },
      "support": {
        "address": { "city": null, "country": null, "line1": null, "line2": null, "postal_code": null, "state": null, "town": null },
        "email": null, "phone": null, "url": null
      }
    },
    "recipient": {
      "applied": true,
      "capabilities": {
        "stripe_balance": {
          "payouts": {
            "status": "restricted",
            "status_details": [{ "code": "requirements_past_due", "resolution": "provide_info" }]
          },
          "stripe_transfers": {
            "status": "restricted",
            "status_details": [{ "code": "requirements_past_due", "resolution": "provide_info" }]
          }
        }
      },
      "default_outbound_destination": null
    }
  },
  "contact_email": "quang.pham@coduct.com",
  "created": "2026-06-05T07:24:09.000Z",
  "dashboard": "express",
  "identity": {
    "country": "DE",
    "entity_type": "company",
    "business_details": null,
    "individual": null,
    "attestations": {
      "persons_provided": { "directors": false, "executives": false, "owners": false, "ownership_exemption_reason": null }
    }
  },
  "defaults": null,
  "display_name": null,
  "metadata": { "publisherId": "cd9ed8a1-5845-4157-92b4-5e7137a1dd96" },
  "requirements": null,
  "livemode": false
}
```

---

<a name="req-02"></a>
## REQ-02 — Generate a KYC Onboarding Link

**Date:** 2026-06-05  
**Request:** [REQ-02 in curl-log.md](./curl-log.md#req-02)  
**Result:** Onboarding URL (expires 5 min after creation)

Key fields to note:
- `url` — send this to the Publisher to complete KYC
- `expires_at` — link is single-use and short-lived; generate a fresh one if it expires

```json
{
  "object": "v2.core.account_link",
  "account": "acct_1TepE8F8Iph8xSIh",
  "created": "2026-06-05T04:57:42.000Z",
  "expires_at": "2026-06-05T05:02:42.000Z",
  "url": "https://connect.stripe.com/setup/e/acct_1TepE8F8Iph8xSIh/J65GgEN6f7f6",
  "use_case": {
    "type": "account_onboarding",
    "account_onboarding": {
      "configurations": ["recipient", "merchant"],
      "collection_options": { "fields": "eventually_due", "future_requirements": "omit" },
      "refresh_url": "https://partner.jtl-cloud.com",
      "return_url": "https://partner.jtl-cloud.com"
    }
  },
  "livemode": false
}
```

---

<a name="req-03"></a>
## REQ-03 — Create a Product

**Date:** 2026-06-05  
**Request:** [REQ-03 in curl-log.md](./curl-log.md#req-03)  
**Result:** `prod_xxx` ← update with returned ID

Key fields to note:
- `id` — the Stripe Product ID; persist this against `appId` in your DB
- `metadata.appId` / `metadata.publisherId` — confirm these are echoed back correctly
- `active: true` — product is immediately active

```json
{
  "id": "prod_UeBJdwdgw5Iuly",

  "object": "product",
  "active": true,
  "attributes": [],
  "created": 1780647492,
  "default_price": null,
  "description": "Lorem ipsum.",
  "images": [],
  "livemode": false,
  "marketing_features": [],
  "metadata": {
    "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
    "publisherId": "cd9ed8a1-5845-4157-92b4-5e7137a1dd96"
  },
  "name": "Hello local",
  "package_dimensions": null,
  "shippable": null,
  "statement_descriptor": null,
  "tax_code": null,
  "tax_details": null,
  "type": "service",
  "unit_label": null,
  "updated": 1780647492,
  "url": null
}
```

---

<a name="req-04"></a>
## REQ-04 — Create a Price (Basic / Monthly)

**Date:** 2026-06-05  
**Request:** [REQ-04 in curl-log.md](./curl-log.md#req-04)  
**Result:** `price_1TetJHF8IpKa3SF4sSLjrSQf`

Key fields to note:
- `id` — the Stripe Price ID; persist this against `appId` + `plan_tier` + `plan_interval` in your DB
- `product` — confirms this price is attached to `prod_UeBJdwdgw5Iuly`
- `unit_amount: 3000` — €30.00 (amounts are in cents)
- `recurring.interval: "month"` / `metadata.plan_interval: "monthly"` — Stripe interval vs JTL metadata value
- `recurring.trial_period_days: null` — no trial on the Price itself; trials are set at Checkout Session time
- `metadata.plan_tier: "Basic"` / `metadata.plan_interval: "monthly"` — confirmed echoed back correctly

```json
{
  "id": "price_1TetJHF8IpKa3SF4sSLjrSQf",
  "object": "price",
  "active": true,
  "billing_scheme": "per_unit",
  "created": 1780648859,
  "currency": "eur",
  "custom_unit_amount": null,
  "livemode": false,
  "lookup_key": null,
  "metadata": {
    "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
    "plan_interval": "monthly",
    "plan_tier": "Basic"
  },
  "nickname": null,
  "product": "prod_UeBJdwdgw5Iuly",
  "recurring": {
    "interval": "month",
    "interval_count": 1,
    "meter": null,
    "trial_period_days": null,
    "usage_type": "licensed"
  },
  "tax_behavior": "unspecified",
  "tiers_mode": null,
  "transform_quantity": null,
  "type": "recurring",
  "unit_amount": 3000,
  "unit_amount_decimal": "3000"
}
```

---

<a name="req-05"></a>
## REQ-05 — Create a Customer

**Date:** 2026-06-05  
**Request:** [REQ-05 in curl-log.md](./curl-log.md#req-05)  
**Result:** `cus_UeCsQ6rAAIxgtQ`

Key fields to note:
- `id` — the Stripe Customer ID; persist this against `tenantId` in your DB (`TenantStripeCustomers` table)
- `address.country: "DE"` — drives the VAT regime for all future invoices (DE 19%); this is the most important address field
- `metadata.tenantId: "5bb461dd-..."` — confirmed echoed back; this is your webhook join key
- `tax_exempt: "none"` — Stripe Tax will compute and apply VAT on invoices; change to `"exempt"` only for VAT-registered B2B customers who provide a valid VAT ID
- `delinquent: false` — expected at creation; flips to `true` after a failed payment is left unresolved

```json
{
  "id": "cus_UeCsQ6rAAIxgtQ",
  "object": "customer",
  "address": {
    "city": "Berlin",
    "country": "DE",
    "line1": "Hauptstrasse 2",
    "line2": null,
    "postal_code": "10827",
    "state": null
  },
  "balance": 0,
  "created": 1780653276,
  "currency": null,
  "customer_account": null,
  "default_source": null,
  "delinquent": false,
  "description": null,
  "discount": null,
  "email": "quang.pham@coduct.com",
  "invoice_prefix": "CI54WJH2",
  "invoice_settings": {
    "custom_fields": null,
    "default_payment_method": null,
    "footer": null,
    "rendering_options": null
  },
  "livemode": false,
  "metadata": {
    "tenantId": "5bb461dd-eb16-4704-af56-68c7730d136b"
  },
  "name": "Coduct",
  "phone": null,
  "preferred_locales": [],
  "shipping": null,
  "tax_exempt": "none",
  "test_clock": null
}
```

---

<a name="req-06"></a>
## REQ-06 — Create a Checkout Session

**Date:** 2026-06-05
**Request:** [REQ-06 in curl-log.md](./curl-log.md#req-06)
**Result:** `cs_test_a1EMUGCh2nzyX7ox0t2s9LSgGNyLq9OfQK1zdlX0O5RkOJIg2vnuFc5ZET`

Key fields to note:
- `id` — the Checkout Session ID; store this to track the session state until payment completes
- `url` — open this in a browser to complete payment; expires at `expires_at`
- `status: "open"` / `payment_status: "unpaid"` — expected before the Tenant pays; both flip after successful checkout
- `subscription: null` — will be populated with `sub_xxx` once payment completes
- `metadata: {}` — session-level metadata is intentionally empty; the `tenantId` / `appId` / `publisherId` are under `subscription_data` and flow to the Subscription object, not the Session
- `automatic_tax.enabled: false` — Stripe Tax is not yet enabled; enable it on the Platform account and add `automatic_tax[enabled]=true` to the request to activate VAT computation

```json
{
  "id": "cs_test_a1EMUGCh2nzyX7ox0t2s9LSgGNyLq9OfQK1zdlX0O5RkOJIg2vnuFc5ZET",
  "object": "checkout.session",
  "adaptive_pricing": {
    "enabled": true
  },
  "after_expiration": null,
  "allow_promotion_codes": null,
  "amount_subtotal": 3000,
  "amount_total": 3000,
  "automatic_tax": {
    "enabled": false,
    "liability": null,
    "provider": null,
    "status": null
  },
  "billing_address_collection": null,
  "branding_settings": {
    "background_color": "#ffffff",
    "border_style": "rounded",
    "button_color": "#0074d4",
    "display_name": "Aoecoduct",
    "font_family": "default",
    "icon": null,
    "logo": null
  },
  "cancel_url": "https://hub.jtl-cloud.com",
  "client_reference_id": null,
  "client_secret": null,
  "collected_information": {
    "business_name": null,
    "individual_name": null,
    "shipping_details": null
  },
  "consent": null,
  "consent_collection": null,
  "created": 1780656670,
  "currency": "eur",
  "currency_conversion": null,
  "custom_fields": [],
  "custom_text": {
    "after_submit": null,
    "shipping_address": null,
    "submit": null,
    "terms_of_service_acceptance": null
  },
  "customer": "cus_UeCsQ6rAAIxgtQ",
  "customer_account": null,
  "customer_creation": null,
  "customer_details": {
    "address": null,
    "business_name": null,
    "email": "quang.pham@coduct.com",
    "individual_name": null,
    "name": null,
    "phone": null,
    "tax_exempt": "none",
    "tax_ids": null
  },
  "customer_email": null,
  "discounts": [],
  "expires_at": 1780743070,
  "integration_identifier": null,
  "invoice": null,
  "invoice_creation": null,
  "livemode": false,
  "locale": null,
  "managed_payments": {
    "enabled": false
  },
  "metadata": {},
  "mode": "subscription",
  "origin_context": null,
  "payment_intent": null,
  "payment_link": null,
  "payment_method_collection": "always",
  "payment_method_configuration_details": {
    "id": "pmc_1Tes7UF8IpBM1G52CCnZretY",
    "parent": "pmc_1TXGsMF8IpKa3SF4mqFRcEMz"
  },
  "payment_method_options": {
    "card": {
      "request_three_d_secure": "automatic"
    }
  },
  "payment_method_types": [
    "card",
    "klarna",
    "link"
  ],
  "payment_status": "unpaid",
  "permissions": null,
  "phone_number_collection": {
    "enabled": false
  },
  "recovered_from": null,
  "saved_payment_method_options": {
    "allow_redisplay_filters": [
      "always"
    ],
    "payment_method_remove": "disabled",
    "payment_method_save": null
  },
  "setup_intent": null,
  "shipping_address_collection": null,
  "shipping_cost": null,
  "shipping_options": [],
  "status": "open",
  "submit_type": null,
  "subscription": null,
  "success_url": "https://hub.jtl-cloud.com",
  "total_details": {
    "amount_discount": 0,
    "amount_shipping": 0,
    "amount_tax": 0
  },
  "ui_mode": "hosted_page",
  "url": "https://checkout.stripe.com/c/pay/cs_test_a1EMUGCh2nzyX7ox0t2s9LSgGNyLq9OfQK1zdlX0O5RkOJIg2vnuFc5ZET#fidnandhYHdWcXxpYCc%2FJ2FgY2RwaXEnKSdicGRmZGhqaWBTZHdsZGtxJz8nZmprcXdqaScpJ2R1bE5gfCc%2FJ3VuWnFgdnFaMDRRXUJ3NUM9THVOZDZWQzFMbGA2UFR%2FT2tgX2NHTktpTWJcX0REM2JocF1ENjd%2FXEFDSURXf1IyXXFsVF9VcDJIdWRCbWFqVXJJYFJUSGxsU2BQc39TdEo1NVFCN2ZVfXxqJyknY3dqaFZgd3Ngdyc%2FcXdwYCknZ2RmbmJ3anBrYUZqaWp3Jz8nJmNjY2NjYycpJ2lkfGpwcVF8dWAnPyd2bGtiaWBabHFgaCcpJ2BrZGdpYFVpZGZgbWppYWB3dic%2FcXdwYHgl",
  "wallet_options": null
}
```

---

<a name="req-07"></a>
## REQ-07+ — Verify Result

**Date:** 2026-06-05  
**Request:** [REQ-07 in curl-log.md](./curl-log.md#req-07)

---

<a name="req-07-subscription"></a>
### Subscription

**Result:** `sub_1TevFgF8IpKa3SF4RARXHuI0`

Key fields to note:
- `on_behalf_of: "acct_1Tes6sF8IpBM1G52"` ✓ — Publisher is MoR; their name on Tenant's bank statement
- `transfer_data.destination: "acct_1Tes6sF8IpBM1G52"` ✓ — auto-payout wired to Publisher after each payment
- `application_fee_percent: 15.0` ✓ — JTL retains 15% of every invoice automatically
- `status: "active"` ✓
- `invoice_settings.issuer.account: "acct_1Tes6sF8IpBM1G52"` ✓ — invoice issued under Publisher's account
- `metadata.tenantId: "cus_UeCsQ6rAAIxgtQ"` ⚠️ **BUG** — this is the Stripe Customer ID, not the internal tenant UUID. Webhook handlers looking up `tenantId` will fail to find a matching Tenant record. Correct value should be `"5bb461dd-eb16-4704-af56-68c7730d136b"`. Root cause: the checkout session passed `cus_xxx` instead of the JTL UUID into `subscription_data[metadata][tenantId]`.

```json
{
  "object": "list",
  "data": [
    {
      "id": "sub_1TevLnF8IpKa3SF4eNwiacDI",
      "object": "subscription",
      "application": null,
      "application_fee_percent": 15.0,
      "automatic_tax": {
        "disabled_reason": null,
        "enabled": false,
        "liability": null
      },
      "billing_cycle_anchor": 1780656700,
      "billing_cycle_anchor_config": null,
      "billing_mode": {
        "flexible": {
          "proration_discounts": "included"
        },
        "type": "flexible",
        "updated_at": 1780656670
      },
      "billing_thresholds": null,
      "cancel_at": null,
      "cancel_at_period_end": false,
      "canceled_at": null,
      "cancellation_details": {
        "comment": null,
        "feedback": null,
        "reason": null
      },
      "collection_method": "charge_automatically",
      "created": 1780656700,
      "currency": "eur",
      "customer": "cus_UeCsQ6rAAIxgtQ",
      "customer_account": null,
      "days_until_due": null,
      "default_payment_method": "pm_1TevLjF8IpKa3SF4ZPnyWxT5",
      "default_source": null,
      "default_tax_rates": [],
      "description": null,
      "discounts": [],
      "ended_at": null,
      "invoice_settings": {
        "account_tax_ids": null,
        "issuer": {
          "account": "acct_1Tes6sF8IpBM1G52",
          "type": "account"
        }
      },
      "items": {
        "object": "list",
        "data": [
          {
            "id": "si_UeDnQ27T0cXz1k",
            "object": "subscription_item",
            "billing_thresholds": null,
            "created": 1780656700,
            "current_period_end": 1783248700,
            "current_period_start": 1780656700,
            "discounts": [],
            "metadata": {},
            "plan": {
              "id": "price_1TetJHF8IpKa3SF4sSLjrSQf",
              "object": "plan",
              "active": true,
              "amount": 3000,
              "amount_decimal": "3000",
              "billing_scheme": "per_unit",
              "created": 1780648859,
              "currency": "eur",
              "interval": "month",
              "interval_count": 1,
              "livemode": false,
              "metadata": {
                "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
                "plan_interval": "monthly",
                "plan_tier": "Basic"
              },
              "meter": null,
              "nickname": null,
              "product": "prod_UeBJdwdgw5Iuly",
              "tiers_mode": null,
              "transform_usage": null,
              "trial_period_days": null,
              "usage_type": "licensed"
            },
            "price": {
              "id": "price_1TetJHF8IpKa3SF4sSLjrSQf",
              "object": "price",
              "active": true,
              "billing_scheme": "per_unit",
              "created": 1780648859,
              "currency": "eur",
              "custom_unit_amount": null,
              "livemode": false,
              "lookup_key": null,
              "metadata": {
                "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
                "plan_interval": "monthly",
                "plan_tier": "Basic"
              },
              "nickname": null,
              "product": "prod_UeBJdwdgw5Iuly",
              "recurring": {
                "interval": "month",
                "interval_count": 1,
                "meter": null,
                "trial_period_days": null,
                "usage_type": "licensed"
              },
              "tax_behavior": "unspecified",
              "tiers_mode": null,
              "transform_quantity": null,
              "type": "recurring",
              "unit_amount": 3000,
              "unit_amount_decimal": "3000"
            },
            "quantity": 1,
            "subscription": "sub_1TevLnF8IpKa3SF4eNwiacDI",
            "tax_rates": []
          }
        ],
        "has_more": false,
        "total_count": 1,
        "url": "/v1/subscription_items?subscription=sub_1TevLnF8IpKa3SF4eNwiacDI"
      },
      "latest_invoice": "in_1TevLkF8IpKa3SF45i7kRMq9",
      "livemode": false,
      "managed_payments": {
        "enabled": false
      },
      "metadata": {
        "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
        "publisherId": "cd9ed8a1-5845-4157-92b4-5e7137a1dd96",
        "tenantId": "5bb461dd-eb16-4704-af56-68c7730d136b"
      },
      "next_pending_invoice_item_invoice": null,
      "on_behalf_of": "acct_1Tes6sF8IpBM1G52",
      "pause_collection": null,
      "payment_settings": {
        "payment_method_options": {
          "acss_debit": null,
          "bancontact": null,
          "card": {
            "network": null,
            "request_three_d_secure": "automatic"
          },
          "customer_balance": null,
          "konbini": null,
          "payto": null,
          "pix": null,
          "sepa_debit": null,
          "upi": null,
          "us_bank_account": null
        },
        "payment_method_types": null,
        "save_default_payment_method": "off"
      },
      "pending_invoice_item_interval": null,
      "pending_setup_intent": null,
      "pending_update": null,
      "plan": {
        "id": "price_1TetJHF8IpKa3SF4sSLjrSQf",
        "object": "plan",
        "active": true,
        "amount": 3000,
        "amount_decimal": "3000",
        "billing_scheme": "per_unit",
        "created": 1780648859,
        "currency": "eur",
        "interval": "month",
        "interval_count": 1,
        "livemode": false,
        "metadata": {
          "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
          "plan_interval": "monthly",
          "plan_tier": "Basic"
        },
        "meter": null,
        "nickname": null,
        "product": "prod_UeBJdwdgw5Iuly",
        "tiers_mode": null,
        "transform_usage": null,
        "trial_period_days": null,
        "usage_type": "licensed"
      },
      "presentment_details": {
        "presentment_currency": "vnd"
      },
      "quantity": 1,
      "schedule": null,
      "start_date": 1780656700,
      "status": "active",
      "test_clock": null,
      "transfer_data": {
        "amount_percent": null,
        "destination": "acct_1Tes6sF8IpBM1G52"
      },
      "trial_end": null,
      "trial_settings": {
        "end_behavior": {
          "missing_payment_method": "create_invoice"
        }
      },
      "trial_start": null
    },
  ],
  "has_more": false,
  "url": "/v1/subscriptions"
}
```

---

<a name="req-07-invoice"></a>
### Invoice

**Result:** `in_1TevLkF8IpKa3SF45i7kRMq9`

Key fields to note:
- `status: "paid"` ✓ — invoice collected successfully
- `amount_paid: 3000` ✓ — full €30.00 collected
- `on_behalf_of: "acct_1Tes6sF8IpBM1G52"` ✓ — inherited from Subscription; Publisher is MoR for this charge
- `issuer.account: "acct_1Tes6sF8IpBM1G52"` ✓ — invoice legally issued by Publisher
- `parent.subscription_details.metadata.tenantId: "5bb461dd-eb16-4704-af56-68c7730d136b"` ✓ — internal UUID correct here; this is the field webhook handlers should read
- `billing_reason: "subscription_create"` — first invoice generated at subscription creation; subsequent ones will show `subscription_cycle`

```json
{
  "object": "list",
  "data": [
    {
      "id": "in_1TevLkF8IpKa3SF45i7kRMq9",
      "object": "invoice",
      "account_country": "DE",
      "account_name": "Aoecoduct",
      "account_tax_ids": null,
      "amount_due": 3000,
      "amount_overpaid": 0,
      "amount_paid": 3000,
      "amount_remaining": 0,
      "amount_shipping": 0,
      "application": null,
      "attempt_count": 0,
      "attempted": true,
      "auto_advance": false,
      "automatic_tax": {
        "disabled_reason": null,
        "enabled": false,
        "liability": null,
        "provider": null,
        "status": null
      },
      "automatically_finalizes_at": null,
      "billing_reason": "subscription_create",
      "collection_method": "charge_automatically",
      "created": 1780656700,
      "currency": "eur",
      "custom_fields": null,
      "customer": "cus_UeCsQ6rAAIxgtQ",
      "customer_account": null,
      "customer_address": {
        "city": "Berlin",
        "country": "DE",
        "line1": "Hauptstrasse 2",
        "line2": null,
        "postal_code": "10827",
        "state": null
      },
      "customer_email": "quang.pham@coduct.com",
      "customer_name": "Coduct",
      "customer_phone": null,
      "customer_shipping": null,
      "customer_tax_exempt": "none",
      "customer_tax_ids": [],
      "default_payment_method": null,
      "default_source": null,
      "default_tax_rates": [],
      "description": null,
      "discounts": [],
      "due_date": null,
      "effective_at": 1780656700,
      "ending_balance": 0,
      "footer": null,
      "from_invoice": null,
      "hosted_invoice_url": "https://invoice.stripe.com/i/acct_1TXGr0F8IpKa3SF4/test_YWNjdF8xVFhHcjBGOElwS2EzU0Y0LF9VZURuUnZEaW1ZSjlBNzVZdTJVMnRybGxSYlhoNFhDLDE3MTQ1NDM3MQ0200brHOI4T3?s=ap",
      "invoice_pdf": "https://pay.stripe.com/invoice/acct_1TXGr0F8IpKa3SF4/test_YWNjdF8xVFhHcjBGOElwS2EzU0Y0LF9VZURuUnZEaW1ZSjlBNzVZdTJVMnRybGxSYlhoNFhDLDE3MTQ1NDM3MQ0200brHOI4T3/pdf?s=ap",
      "issuer": {
        "account": "acct_1Tes6sF8IpBM1G52",
        "type": "account"
      },
      "last_finalization_error": null,
      "latest_revision": null,
      "lines": {
        "object": "list",
        "data": [
          {
            "id": "il_1TevLkF8IpKa3SF48Y9vXzZz",
            "object": "line_item",
            "amount": 3000,
            "currency": "eur",
            "description": "1 × Hello local (at €30.00 / month)",
            "discount_amounts": [],
            "discountable": true,
            "discounts": [],
            "invoice": "in_1TevLkF8IpKa3SF45i7kRMq9",
            "livemode": false,
            "metadata": {
              "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
              "publisherId": "cd9ed8a1-5845-4157-92b4-5e7137a1dd96",
              "tenantId": "5bb461dd-eb16-4704-af56-68c7730d136b"
            },
            "parent": {
              "invoice_item_details": null,
              "subscription_item_details": {
                "invoice_item": null,
                "proration": false,
                "proration_details": {
                  "credited_items": null
                },
                "subscription": "sub_1TevLnF8IpKa3SF4eNwiacDI",
                "subscription_item": "si_UeDnQ27T0cXz1k"
              },
              "type": "subscription_item_details"
            },
            "period": {
              "end": 1783248700,
              "start": 1780656700
            },
            "pretax_credit_amounts": [],
            "pricing": {
              "price_details": {
                "price": "price_1TetJHF8IpKa3SF4sSLjrSQf",
                "product": "prod_UeBJdwdgw5Iuly"
              },
              "type": "price_details",
              "unit_amount_decimal": "3000"
            },
            "quantity": 1,
            "quantity_decimal": "1",
            "subtotal": 3000,
            "taxes": []
          }
        ],
        "has_more": false,
        "total_count": 1,
        "url": "/v1/invoices/in_1TevLkF8IpKa3SF45i7kRMq9/lines"
      },
      "livemode": false,
      "metadata": {},
      "next_payment_attempt": null,
      "number": "AP8EHZMR-0002",
      "on_behalf_of": "acct_1Tes6sF8IpBM1G52",
      "parent": {
        "quote_details": null,
        "subscription_details": {
          "metadata": {
            "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
            "publisherId": "cd9ed8a1-5845-4157-92b4-5e7137a1dd96",
            "tenantId": "5bb461dd-eb16-4704-af56-68c7730d136b"
          },
          "subscription": "sub_1TevLnF8IpKa3SF4eNwiacDI"
        },
        "type": "subscription_details"
      },
      "payment_settings": {
        "default_mandate": null,
        "payment_method_options": {
          "acss_debit": null,
          "bancontact": null,
          "card": {
            "request_three_d_secure": "automatic"
          },
          "customer_balance": null,
          "konbini": null,
          "payto": null,
          "pix": null,
          "sepa_debit": null,
          "upi": null,
          "us_bank_account": null
        },
        "payment_method_types": null
      },
      "period_end": 1780656700,
      "period_start": 1780656700,
      "post_payment_credit_notes_amount": 0,
      "pre_payment_credit_notes_amount": 0,
      "receipt_number": null,
      "rendering": null,
      "shipping_cost": null,
      "shipping_details": null,
      "starting_balance": 0,
      "statement_descriptor": null,
      "status": "paid",
      "status_transitions": {
        "finalized_at": 1780656700,
        "marked_uncollectible_at": null,
        "paid_at": 1780656701,
        "voided_at": null
      },
      "subtotal": 3000,
      "subtotal_excluding_tax": 3000,
      "test_clock": null,
      "total": 3000,
      "total_discount_amounts": [],
      "total_excluding_tax": 3000,
      "total_pretax_credit_amounts": [],
      "total_taxes": [],
      "webhooks_delivered_at": 1780656700
    }
  ],
  "has_more": false,
  "url": "/v1/invoices"
}
```

---

<a name="req-08-payment-expand"></a>
### Payment Expand from Invoices

Key fields to note:
- `payments.data[0].payment.type: "payment_intent"` — modern invoices route through a PaymentIntent, not a direct charge ID
- `payments.data[0].payment.payment_intent: "pi_3TevLkF8IpKa3SF41UBaHGDX"` — use this to fetch the charge and money split in the next step
- `payments.data[0].status: "paid"` ✓
- `payments.data[0].amount_paid: 3000` ✓

```json
{
  "id": "in_1TevLkF8IpKa3SF45i7kRMq9",
  "object": "invoice",
  "account_country": "DE",
  "account_name": "Aoecoduct",
  "account_tax_ids": null,
  "amount_due": 3000,
  "amount_overpaid": 0,
  "amount_paid": 3000,
  "amount_remaining": 0,
  "amount_shipping": 0,
  "application": null,
  "attempt_count": 0,
  "attempted": true,
  "auto_advance": false,
  "automatic_tax": {
    "disabled_reason": null,
    "enabled": false,
    "liability": null,
    "provider": null,
    "status": null
  },
  "automatically_finalizes_at": null,
  "billing_reason": "subscription_create",
  "collection_method": "charge_automatically",
  "created": 1780656700,
  "currency": "eur",
  "custom_fields": null,
  "customer": "cus_UeCsQ6rAAIxgtQ",
  "customer_account": null,
  "customer_address": {
    "city": "Berlin",
    "country": "DE",
    "line1": "Hauptstrasse 2",
    "line2": null,
    "postal_code": "10827",
    "state": null
  },
  "customer_email": "quang.pham@coduct.com",
  "customer_name": "Coduct",
  "customer_phone": null,
  "customer_shipping": null,
  "customer_tax_exempt": "none",
  "customer_tax_ids": [],
  "default_payment_method": null,
  "default_source": null,
  "default_tax_rates": [],
  "description": null,
  "discounts": [],
  "due_date": null,
  "effective_at": 1780656700,
  "ending_balance": 0,
  "footer": null,
  "from_invoice": null,
  "hosted_invoice_url": "https://invoice.stripe.com/i/acct_1TXGr0F8IpKa3SF4/test_YWNjdF8xVFhHcjBGOElwS2EzU0Y0LF9VZURuUnZEaW1ZSjlBNzVZdTJVMnRybGxSYlhoNFhDLDE3MTQ1NjcyMg02005RpVxRtk?s=ap",
  "invoice_pdf": "https://pay.stripe.com/invoice/acct_1TXGr0F8IpKa3SF4/test_YWNjdF8xVFhHcjBGOElwS2EzU0Y0LF9VZURuUnZEaW1ZSjlBNzVZdTJVMnRybGxSYlhoNFhDLDE3MTQ1NjcyMg02005RpVxRtk/pdf?s=ap",
  "issuer": {
    "account": "acct_1Tes6sF8IpBM1G52",
    "type": "account"
  },
  "last_finalization_error": null,
  "latest_revision": null,
  "lines": {
    "object": "list",
    "data": [
      {
        "id": "il_1TevLkF8IpKa3SF48Y9vXzZz",
        "object": "line_item",
        "amount": 3000,
        "currency": "eur",
        "description": "1 × Hello local (at €30.00 / month)",
        "discount_amounts": [],
        "discountable": true,
        "discounts": [],
        "invoice": "in_1TevLkF8IpKa3SF45i7kRMq9",
        "livemode": false,
        "metadata": {
          "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
          "publisherId": "cd9ed8a1-5845-4157-92b4-5e7137a1dd96",
          "tenantId": "5bb461dd-eb16-4704-af56-68c7730d136b"
        },
        "parent": {
          "invoice_item_details": null,
          "subscription_item_details": {
            "invoice_item": null,
            "proration": false,
            "proration_details": {
              "credited_items": null
            },
            "subscription": "sub_1TevLnF8IpKa3SF4eNwiacDI",
            "subscription_item": "si_UeDnQ27T0cXz1k"
          },
          "type": "subscription_item_details"
        },
        "period": {
          "end": 1783248700,
          "start": 1780656700
        },
        "pretax_credit_amounts": [],
        "pricing": {
          "price_details": {
            "price": "price_1TetJHF8IpKa3SF4sSLjrSQf",
            "product": "prod_UeBJdwdgw5Iuly"
          },
          "type": "price_details",
          "unit_amount_decimal": "3000"
        },
        "quantity": 1,
        "quantity_decimal": "1",
        "subtotal": 3000,
        "taxes": []
      }
    ],
    "has_more": false,
    "total_count": 1,
    "url": "/v1/invoices/in_1TevLkF8IpKa3SF45i7kRMq9/lines"
  },
  "livemode": false,
  "metadata": {},
  "next_payment_attempt": null,
  "number": "AP8EHZMR-0002",
  "on_behalf_of": "acct_1Tes6sF8IpBM1G52",
  "parent": {
    "quote_details": null,
    "subscription_details": {
      "metadata": {
        "appId": "5d2947f7-018c-4ad0-9dd8-3929a5641cf3",
        "publisherId": "cd9ed8a1-5845-4157-92b4-5e7137a1dd96",
        "tenantId": "5bb461dd-eb16-4704-af56-68c7730d136b"
      },
      "subscription": "sub_1TevLnF8IpKa3SF4eNwiacDI"
    },
    "type": "subscription_details"
  },
  "payment_settings": {
    "default_mandate": null,
    "payment_method_options": {
      "acss_debit": null,
      "bancontact": null,
      "card": {
        "request_three_d_secure": "automatic"
      },
      "customer_balance": null,
      "konbini": null,
      "payto": null,
      "pix": null,
      "sepa_debit": null,
      "upi": null,
      "us_bank_account": null
    },
    "payment_method_types": null
  },
  "payments": {
    "object": "list",
    "data": [
      {
        "id": "inpay_1TevLoF8IpKa3SF4BzVHWwok",
        "object": "invoice_payment",
        "amount_paid": 3000,
        "amount_requested": 3000,
        "created": 1780656700,
        "currency": "eur",
        "invoice": "in_1TevLkF8IpKa3SF45i7kRMq9",
        "is_default": true,
        "livemode": false,
        "payment": {
          "payment_intent": "pi_3TevLkF8IpKa3SF41UBaHGDX",
          "type": "payment_intent"
        },
        "status": "paid",
        "status_transitions": {
          "canceled_at": null,
          "paid_at": 1780656701
        }
      }
    ],
    "has_more": false,
    "url": "/v1/invoices/payments"
  },
  "period_end": 1780656700,
  "period_start": 1780656700,
  "post_payment_credit_notes_amount": 0,
  "pre_payment_credit_notes_amount": 0,
  "receipt_number": null,
  "rendering": null,
  "shipping_cost": null,
  "shipping_details": null,
  "starting_balance": 0,
  "statement_descriptor": null,
  "status": "paid",
  "status_transitions": {
    "finalized_at": 1780656700,
    "marked_uncollectible_at": null,
    "paid_at": 1780656701,
    "voided_at": null
  },
  "subtotal": 3000,
  "subtotal_excluding_tax": 3000,
  "test_clock": null,
  "total": 3000,
  "total_discount_amounts": [],
  "total_excluding_tax": 3000,
  "total_pretax_credit_amounts": [],
  "total_taxes": [],
  "webhooks_delivered_at": 1780656700
}
```

<a name="req-08-payment-intent"></a>
### Money Split from Payment Intents

Key fields to note:
- `latest_charge.id: "ch_3TevLkF8IpKa3SF41smUY3ie"` ✓ — the Charge object; use this ID to look up refunds if needed
- `latest_charge.application_fee: "fee_1TevLmF8IpBM1G52tait4HjV"` ✓ — ApplicationFee created; JTL's commission retained on Platform
- `latest_charge.application_fee_amount: 450` ✓ — 15% of €30.00 = €4.50
- `latest_charge.transfer: "tr_3TevLkF8IpKa3SF41S4P8dW2"` ✓ — auto-Transfer created; Publisher received the net payout
- `latest_charge.destination: "acct_1Tes6sF8IpBM1G52"` ✓ — Publisher's Connected Account
- `latest_charge.on_behalf_of: "acct_1Tes6sF8IpBM1G52"` ✓ — Publisher is MoR on this charge
- `latest_charge.status: "succeeded"` ✓
- `on_behalf_of: "acct_1Tes6sF8IpBM1G52"` ✓ — echoed at the PaymentIntent level too

```json
{
  "id": "pi_3TevLkF8IpKa3SF41UBaHGDX",
  "object": "payment_intent",
  "amount": 3000,
  "amount_capturable": 0,
  "amount_details": {
    "tip": {}
  },
  "amount_received": 3000,
  "application": null,
  "application_fee_amount": 450,
  "automatic_payment_methods": null,
  "canceled_at": null,
  "cancellation_reason": null,
  "capture_method": "automatic",
  "client_secret": "pi_3TevLkF8IpKa3SF41UBaHGDX_secret_Y8umMS93zKsqlcme0kCoVbQAR",
  "confirmation_method": "automatic",
  "created": 1780656700,
  "currency": "eur",
  "customer": "cus_UeCsQ6rAAIxgtQ",
  "customer_account": null,
  "description": "Subscription creation",
  "excluded_payment_method_types": null,
  "last_payment_error": null,
  "latest_charge": {
    "id": "ch_3TevLkF8IpKa3SF41smUY3ie",
    "object": "charge",
    "amount": 3000,
    "amount_captured": 3000,
    "amount_refunded": 0,
    "application": null,
    "application_fee": "fee_1TevLmF8IpBM1G52tait4HjV",
    "application_fee_amount": 450,
    "balance_transaction": "txn_3TevLkF8IpKa3SF41nNZmjzr",
    "billing_details": {
      "address": {
        "city": null,
        "country": "DE",
        "line1": null,
        "line2": null,
        "postal_code": null,
        "state": null
      },
      "email": "quang.pham@coduct.com",
      "name": "Quang Pham",
      "phone": null,
      "tax_id": null
    },
    "calculated_statement_descriptor": "AOECODUCT",
    "captured": true,
    "created": 1780656701,
    "currency": "eur",
    "customer": "cus_UeCsQ6rAAIxgtQ",
    "description": "Subscription creation",
    "destination": "acct_1Tes6sF8IpBM1G52",
    "dispute": null,
    "disputed": false,
    "failure_balance_transaction": null,
    "failure_code": null,
    "failure_message": null,
    "fraud_details": {},
    "livemode": false,
    "metadata": {},
    "on_behalf_of": "acct_1Tes6sF8IpBM1G52",
    "order": null,
    "outcome": {
      "advice_code": null,
      "network_advice_code": null,
      "network_decline_code": null,
      "network_status": "approved_by_network",
      "reason": null,
      "risk_level": "normal",
      "risk_score": 40,
      "seller_message": "Payment complete.",
      "type": "authorized"
    },
    "paid": true,
    "payment_intent": "pi_3TevLkF8IpKa3SF41UBaHGDX",
    "payment_method": "pm_1TevLjF8IpKa3SF4ZPnyWxT5",
    "payment_method_details": {
      "card": {
        "amount_authorized": 3000,
        "authorization_code": "803098",
        "brand": "visa",
        "checks": {
          "address_line1_check": null,
          "address_postal_code_check": null,
          "cvc_check": "pass"
        },
        "country": "US",
        "ds_transaction_id": null,
        "exp_month": 12,
        "exp_year": 2027,
        "extended_authorization": {
          "status": "disabled"
        },
        "fingerprint": "o3JV8NQ69k6pGA2w",
        "funding": "credit",
        "incremental_authorization": {
          "status": "unavailable"
        },
        "installments": null,
        "last4": "4242",
        "mandate": null,
        "multicapture": {
          "status": "unavailable"
        },
        "network": "visa",
        "network_token": {
          "used": false
        },
        "network_transaction_id": "111517486567881",
        "overcapture": {
          "maximum_amount_capturable": 3000,
          "status": "unavailable"
        },
        "regulated_status": "unregulated",
        "three_d_secure": null,
        "transaction_link_id": null,
        "wallet": null
      },
      "type": "card"
    },
    "presentment_details": {
      "presentment_amount": 956096,
      "presentment_currency": "vnd"
    },
    "radar_options": {},
    "receipt_email": null,
    "receipt_number": null,
    "receipt_url": "https://pay.stripe.com/receipts/invoices/CAcaFwoVYWNjdF8xVFhHcjBGOElwS2EzU0Y0KLfBmtEGMgZ3Tw-aVaY6LBbwPNBA_ozsQZgjXWNcvFxXSwkRz0dsGKgiDYPlsHlStfsKnOkty1y9g9Uo?s=ap",
    "refunded": false,
    "review": null,
    "shipping": null,
    "source": null,
    "source_transfer": null,
    "statement_descriptor": null,
    "statement_descriptor_suffix": null,
    "status": "succeeded",
    "transfer": "tr_3TevLkF8IpKa3SF41S4P8dW2",
    "transfer_data": {
      "amount": null,
      "destination": "acct_1Tes6sF8IpBM1G52"
    },
    "transfer_group": "group_pi_3TevLkF8IpKa3SF41UBaHGDX"
  },
  "livemode": false,
  "managed_payments": {
    "enabled": false
  },
  "metadata": {},
  "next_action": null,
  "on_behalf_of": "acct_1Tes6sF8IpBM1G52",
  "payment_details": {
    "customer_reference": null,
    "order_reference": "cs_test_a1EMUGCh2nzyX7ox0t2s9LSgGNyLq9OfQK1zdlX0O5RkOJIg2vnuFc5ZET"
  },
  "payment_method": "pm_1TevLjF8IpKa3SF4ZPnyWxT5",
  "payment_method_configuration_details": null,
  "payment_method_options": {
    "card": {
      "installments": null,
      "mandate_options": null,
      "network": null,
      "request_three_d_secure": "automatic",
      "setup_future_usage": "off_session"
    }
  },
  "payment_method_types": [
    "card"
  ],
  "presentment_details": {
    "presentment_amount": 956096,
    "presentment_currency": "vnd"
  },
  "processing": null,
  "receipt_email": null,
  "review": null,
  "setup_future_usage": "off_session",
  "shared_payment_granted_token": null,
  "shipping": null,
  "source": null,
  "statement_descriptor": null,
  "statement_descriptor_suffix": null,
  "status": "succeeded",
  "transfer_data": {
    "destination": "acct_1Tes6sF8IpBM1G52"
  },
  "transfer_group": "group_pi_3TevLkF8IpKa3SF41UBaHGDX"
}
```

<a name="req-09-transfer"></a>
## REQ-09

### Transfer

Key fields to note:
- `amount: 3000` ✓ — gross amount (€30.00) transferred to Publisher. The ApplicationFee is charged separately from the Publisher's Stripe balance, not deducted here. Net to Publisher = €30.00 - €4.50 = €25.50.
- `destination: "acct_1Tes6sF8IpBM1G52"` ✓ — Publisher's Connected Account received the transfer
- `source_transaction: "ch_3TevLkF8IpKa3SF41smUY3ie"` — links back to the originating Charge
- `reversed: false` ✓ — no reversal; transfer is final

```json
{
  "id": "tr_3TevLkF8IpKa3SF41S4P8dW2",
  "object": "transfer",
  "amount": 3000,
  "amount_reversed": 0,
  "balance_transaction": "txn_3TevLkF8IpKa3SF41MHI6HKl",
  "created": 1780656701,
  "currency": "eur",
  "description": null,
  "destination": "acct_1Tes6sF8IpBM1G52",
  "destination_payment": "py_1TevLmF8IpBM1G52OlT9DDC1",
  "livemode": false,
  "metadata": {},
  "reversals": {
    "object": "list",
    "data": [],
    "has_more": false,
    "total_count": 0,
    "url": "/v1/transfers/tr_3TevLkF8IpKa3SF41S4P8dW2/reversals"
  },
  "reversed": false,
  "source_transaction": "ch_3TevLkF8IpKa3SF41smUY3ie",
  "source_type": "card",
  "transfer_group": "group_pi_3TevLkF8IpKa3SF41UBaHGDX"
}
```

<a name="req-09-application-fee"></a>
### ApplicationFee

Key fields to note:
- `amount: 450` ✓ — €4.50 = 15% of €30.00; retained on the Platform balance
- `account: "acct_1Tes6sF8IpBM1G52"` ✓ — the Publisher's Connected Account the fee was charged against
- `originating_transaction: "ch_3TevLkF8IpKa3SF41smUY3ie"` — links back to the originating Charge
- `charge: "py_1TevLmF8IpBM1G52OlT9DDC1"` — the `py_xxx` is the payment on the Publisher's side (destination payment); different from the platform-side `ch_xxx`
- `refunded: false` ✓

```json
{
  "id": "fee_1TevLmF8IpBM1G52tait4HjV",
  "object": "application_fee",
  "account": "acct_1Tes6sF8IpBM1G52",
  "amount": 450,
  "amount_refunded": 0,
  "application": "ca_UWJUPVC1i8cuPCY1iq3DhvRXRiLXCRc4",
  "balance_transaction": "txn_1TevLoF8IpKa3SF4nFnJJQM2",
  "charge": "py_1TevLmF8IpBM1G52OlT9DDC1",
  "created": 1780656702,
  "currency": "eur",
  "fee_source": {
    "charge": "py_1TevLmF8IpBM1G52OlT9DDC1",
    "type": "charge"
  },
  "livemode": false,
  "originating_transaction": "ch_3TevLkF8IpKa3SF41smUY3ie",
  "refunded": false,
  "refunds": {
    "object": "list",
    "data": [],
    "has_more": false,
    "total_count": 0,
    "url": "/v1/application_fees/fee_1TevLmF8IpBM1G52tait4HjV/refunds"
  }
}
```
