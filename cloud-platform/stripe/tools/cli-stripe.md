## CLI Stripe

This document details some useful Stripe commands

## Login

```
stripe login
```

## List Config Information

Note: Because every sandbox Stripe environment is considered a different "account". This is a way for you to check which account you are currently logged into.

```bash
stripe config --list
```

Sample Response:

```
color = ''
project-name = 'default'

[default]
account_id = 'acct_...'
device_name = 'Quangs-MacBook-Pro.local'
display_name = 'DEV'
test_mode_api_key = 'sk_test_...'
test_mode_key_expires_at = '2026-08-24'
test_mode_pub_key = 'pk_test_...'
```

## List All Customers

```bash
stripe customers list --limit 10
```

Sample Response:
```
ID                  Email                          Name                 Created
cus_AAAAAAAAAA       customer1@example.com          Test Company 1       2026-05-28
cus_BBBBBBBBBB       customer2@example.com          Test Company 2       2026-05-28
cus_CCCCCCCCCC       customer3@example.com          Test Company 3       2026-05-27
```

## List All Products

```bash
stripe products list --limit 10
```

Sample Response:
```
ID              Name                        Type       Created      Active
prod_AAAA       Super Inventory Manager     service    2026-05-28   true
prod_BBBB       Shipping Optimizer          service    2026-05-28   true
prod_CCCC       Analytics Dashboard         service    2026-05-27   true
```

## List All Prices

```bash
stripe prices list --limit 10
```

Sample Response:
```
ID                Unit Amount  Currency  Interval  Product
price_AAAA        3000         eur       month     prod_AAAA
price_BBBB        2500         eur       month     prod_BBBB
price_CCCC        4999         eur       year      prod_CCCC
```

## List All Subscriptions

```bash
stripe subscriptions list --limit 10
```

Sample Response:
```
ID              Customer         Status    Current Period Start  Current Period End
sub_AAAA        cus_AAAAAAAAAA   active    2026-05-28            2026-06-28
sub_BBBB        cus_BBBBBBBBBB   active    2026-05-28            2026-06-28
sub_CCCC        cus_CCCCCCCCCC   canceled  2026-05-15            2026-06-15
```

## List All Charges

```bash
stripe charges list --limit 10
```

Sample Response:
```
ID              Amount   Currency  Status    Created      Customer
ch_AAAA         3000     eur       succeeded 2026-05-28   cus_AAAAAAAAAA
ch_BBBB         2500     eur       succeeded 2026-05-28   cus_BBBBBBBBBB
ch_CCCC         4999     eur       failed    2026-05-27   cus_CCCCCCCCCC
```

## List All Invoices

```bash
stripe invoices list --limit 10
```

Sample Response:
```
ID              Customer         Status    Amount    Currency  Created
in_AAAA         cus_AAAAAAAAAA   paid      3000      eur       2026-05-28
in_BBBB         cus_BBBBBBBBBB   paid      2500      eur       2026-05-28
in_CCCC         cus_CCCCCCCCCC   open      4999      eur       2026-05-27
```

## Create a Product

```bash
stripe products create --name="Test App" --description="A test application"
```

Sample Response:
```
{
  "id": "prod_NEWID123",
  "object": "product",
  "name": "Test App",
  "description": "A test application",
  "created": 1685270400,
  "updated": 1685270400,
  "active": true
}
```

## Create a Price

```bash
stripe prices create \
  --product=prod_NEWID123 \
  --unit-amount=3000 \
  --currency=eur \
  -d "recurring[interval]=month"
```

Sample Response:
```
{
  "id": "price_NEWPRICE123",
  "object": "price",
  "product": "prod_NEWID123",
  "unit_amount": 3000,
  "currency": "eur",
  "type": "recurring",
  "recurring": {
    "interval": "month",
    "interval_count": 1
  },
  "created": 1685270401
}
```

## Create a Customer

```bash
stripe customers create \
  --name="New Customer GmbH" \
  --email="customer@example.com" \
  -d "address[country]=DE" \
  -d "address[city]=Stuttgart" \
  -d "metadata[jtl_tenant_id]=tenant_12345"
```

Sample Response:
```
{
  "id": "cus_NEWCUST123",
  "object": "customer",
  "name": "New Customer GmbH",
  "email": "customer@example.com",
  "address": {
    "country": "DE",
    "city": "Stuttgart"
  },
  "created": 1685270402
}
```

## Create a Checkout Session

```bash
stripe checkout sessions create \
  --success-url="https://example.com/success?session_id={CHECKOUT_SESSION_ID}" \
  --cancel-url="https://example.com/cancel" \
  --mode=subscription \
  --customer=cus_NEWCUST123 \
  -d "line_items[0][price]=price_NEWPRICE123" \
  -d "line_items[0][quantity]=1" \
  -d "subscription_data[on_behalf_of]=acct_PUBLISHER_ID" \
  -d "subscription_data[transfer_data][destination]=acct_PUBLISHER_ID" \
  -d "subscription_data[application_fee_percent]=15"
```

Sample Response:
```
{
  "id": "cs_NEWSESSION123",
  "object": "checkout.session",
  "customer": "cus_NEWCUST123",
  "mode": "subscription",
  "status": "open",
  "url": "https://checkout.stripe.com/pay/cs_test_...",
  "success_url": "https://example.com/success?session_id={CHECKOUT_SESSION_ID}",
  "cancel_url": "https://example.com/cancel",
  "created": 1685270403
}
```

## Retrieve a Subscription (with all details)

```bash
stripe subscriptions retrieve sub_AAAA
```

Sample Response:
```
{
  "id": "sub_AAAA",
  "object": "subscription",
  "customer": "cus_AAAAAAAAAA",
  "status": "active",
  "current_period_start": 1685270400,
  "current_period_end": 1688862400,
  "on_behalf_of": "acct_PUBLISHER_ID",
  "transfer_data": {
    "destination": "acct_PUBLISHER_ID"
  },
  "application_fee_percent": 15,
  "created": 1685270401
}
```

## List Subscriptions for a Specific Customer

```bash
stripe subscriptions list --customer=cus_AAAAAAAAAA
```

Sample Response:
```
ID          Status   Product              Next Billing
sub_AAAA    active   prod_AAAA            2026-06-28
sub_BBBB    active   prod_BBBB            2026-06-30
```

## List Application Fees (JTL Commission Tracking)

```bash
stripe application-fees list --limit 10
```

Sample Response:
```
ID              Amount   Currency  Application  Charge         Created
fee_AAAA        450      eur       acct_PLAT    ch_AAAA       2026-05-28
fee_BBBB        375      eur       acct_PLAT    ch_BBBB       2026-05-28
```

## List Transfers to Publishers

```bash
stripe transfers list --limit 10
```

Sample Response:
```
ID              Amount   Currency  Destination           Status    Created
tr_AAAA         2550     eur       acct_PUBLISHER_ID     pending   2026-05-28
tr_BBBB         2125     eur       acct_PUBLISHER_ID     pending   2026-05-28
```

## List Connected Accounts (Publishers)

```bash
stripe accounts list --limit 100
```

Sample Response:
```
ID                   Type      Country  Status      Created
acct_PUBLISHER_ID    express   DE       active      2026-05-20
acct_PUBLISHER_2_ID  express   DE       active      2026-05-22
acct_PUBLISHER_3_ID  express   DE       verified    2026-05-25
```

## Note: Capability names differ between Accounts v1 and v2

If you create accounts with the newer **Accounts v2** API (`stripe post /v2/core/accounts`), the capability field paths are renamed and re-nested vs. the v1 `stripe accounts create` call. When reading or requesting capabilities, watch for:

| Capability | v1 (`accounts create`) | v2 (`/v2/core/accounts`) |
|---|---|---|
| Accept card payments | `capabilities[card_payments][requested]` | `configuration[merchant][capabilities][card_payments][requested]` |
| Receive transfers (payouts destination) | `capabilities[transfers][requested]` | `configuration[recipient][capabilities][stripe_balance][stripe_transfers][requested]` |
| Pay out to bank | `capabilities[payouts][requested]` | `configuration[merchant][capabilities][stripe_balance][payouts][requested]` |

So `transfers` → `stripe_balance.stripe_transfers` and `payouts` → `stripe_balance.payouts`; `card_payments` keeps its name but moves under `configuration.merchant`.

**Which one JTL uses:** per the Solution Plan, JTL **creates accounts with v2** (`stripe post /v2/core/accounts`), so the **v2 paths above are what you request at account-creation time**. Everything else stays on v1 (charges, Checkout, subscriptions), and because v1↔v2 accounts are cross-compatible, reading an account back through the v1 `stripe accounts retrieve` returns the familiar v1-shaped `capabilities.transfers` / `capabilities.card_payments` fields. The v1 column above is therefore mostly what you'll see on *read* and in the sandbox quick-test shortcut. See the Accounts v1-vs-v2 sidebars in `implementation-guide.md` for the full impact analysis.

## List All Charges on a Specific Publisher's Account (Red Model)

```bash
stripe charges list --limit 10 --stripe-account acct_PUBLISHER_ID
```

Sample Response:
```
ID              Amount   Currency  Status    Customer              Created
ch_AAAA         3000     eur       succeeded cus_AAAAAAAAAA_PUB1   2026-05-28
ch_BBBB         2500     eur       succeeded cus_BBBBBBBBBB_PUB1   2026-05-28
```

## List Invoices for a Subscription

```bash
stripe invoices list --subscription sub_AAAA
```

Sample Response:
```
ID         Status   Amount    Currency   Customer           Created
in_AAAA    paid     3000      eur        cus_AAAAAAAAAA     2026-05-28
in_BBBB    open     3000      eur        cus_AAAAAAAAAA     2026-04-28
```

## Retrieve Full Charge Details

```bash
stripe charges retrieve ch_AAAA
```

Sample Response:
```
{
  "id": "ch_AAAA",
  "object": "charge",
  "amount": 3000,
  "currency": "eur",
  "customer": "cus_AAAAAAAAAA",
  "status": "succeeded",
  "on_behalf_of": "acct_PUBLISHER_ID",
  "transfer_data": {
    "destination": "acct_PUBLISHER_ID"
  },
  "application_fee": {
    "id": "fee_AAAA",
    "amount": 450,
    "currency": "eur"
  },
  "transfer": "tr_AAAA",
  "created": 1685270400
}
```

## Watch Webhooks in Real Time

```bash
stripe listen --forward-to localhost:3000/webhooks/stripe
```

Output:
```
> Ready! Your webhook signing secret is whsec_test_secret_abc123... (ctrl + c to quit)
2026-05-28 14:23:10   --> checkout.session.completed [evt_1234567890]
2026-05-28 14:23:10  <--  [200] POST http://localhost:3000/webhooks/stripe
2026-05-28 14:23:11   --> customer.subscription.created [evt_1234567891]
2026-05-28 14:23:12   --> invoice.paid [evt_1234567892]
```

## Trigger Test Webhook Events

```bash
stripe trigger checkout.session.completed
stripe trigger customer.subscription.created
stripe trigger invoice.paid
stripe trigger invoice.payment_failed
stripe trigger customer.subscription.deleted
stripe trigger charge.refunded
```

## View API Logs in Real Time

```bash
stripe logs tail
```

Output:
```
2026-05-28 14:25:30 POST /v1/checkout/sessions [200] 0.42s
2026-05-28 14:25:31 GET /v1/subscriptions [200] 0.18s
2026-05-28 14:25:32 POST /v1/customers [201] 0.25s
2026-05-28 14:25:33 POST /v1/charges [200] 0.56s
```

## Filter Charges by Date Range

```bash
stripe charges list \
  --created='{"gte":1685270400, "lte":1685356800}' \
  --limit 10
```

## Filter Invoices by Status

```bash
stripe invoices list --status=open --limit 10
stripe invoices list --status=paid --limit 10
stripe invoices list --status=draft --limit 10
```

## Create a Refund

```bash
stripe refunds create --charge=ch_AAAA
```

Sample Response:
```
{
  "id": "re_NEWREFUND123",
  "object": "refund",
  "charge": "ch_AAAA",
  "amount": 3000,
  "currency": "eur",
  "status": "succeeded",
  "created": 1685270500
}
```

## Cancel a Subscription

```bash
stripe subscriptions cancel sub_AAAA
```

Sample Response:
```
{
  "id": "sub_AAAA",
  "object": "subscription",
  "status": "canceled",
  "canceled_at": 1685270600,
  "ended_at": 1685270600
}
```

## Update a Subscription (e.g., change plan)

```bash
stripe subscriptions update sub_AAAA \
  -d "items[0][id]=si_existing_id" \
  -d "items[0][price]=price_NEWPRICE456"
```

## Search Customers by Email

```bash
stripe customers list --email "customer@example.com"
```

## Export Data to CSV (via Dashboard, but useful to know)

Go to **Developers → Event logs** or use:
```bash
stripe charges list --limit 100 | head -20
# Then pipe to grep, awk, or paste into a spreadsheet
```

## Quick Health Check

```bash
# Verify you're connected to the right Sandbox
stripe config --list

# Confirm API keys are valid
stripe customers list --limit 1

# Check webhook endpoint is listening (if running)
stripe webhook-endpoints list
```
