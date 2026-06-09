# Flow 3: Order Fulfillment (Steps 1200-1700)

**Duration:** ~2-3 minutes per order  
**User Role:** Merchant  
**Entry Point:** Dashboard → Orders menu  
**Goal:** Process shop orders from receipt through shipping  

---

## 📐 User Journey Map

```
┌─────────────────────────────────────────────────────────────┐
│          ORDER FULFILLMENT FLOW                             │
│   (Order receipt → Invoice → Payment → Shipping)            │
└─────────────────────────────────────────────────────────────┘

[MERCHANT HAS CREATED ITEMS AND SHOP IS LIVE]
          ↓
┌──────────────────────────────────────────────────────────────┐
│ EXTERNAL TRIGGER: Customer Places Order                      │
│                                                              │
│ 1. Customer buys item on JTL-Shop                           │
│ 2. Shop creates order (status: "pending payment")            │
│ 3. Order appears in Wawi (via Shop sync)                    │
│ 4. Cloud Eventbus picks up order event                      │
│                                                              │
│ → Order ready for merchant visibility in ERP Cloud          │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 1200: ORDER LIST                                        │
│ Route: /orders                                              │
│ Design: CERP-963                                            │
│                                                              │
│ AUTO-SYNC BEHAVIOR:                                         │
│ • Eventbus interval: Every 5 minutes                         │
│   └─→ Calls: WAWI-88088 (Order sync endpoint)               │
│   └─→ Fetches new orders from Shop                          │
│   └─→ Updates order list                                    │
│                                                              │
│ • Event-based trigger (immediate):                          │
│   └─→ Order created in Shop → Event fired                   │
│   └─→ Eventbus immediately calls sync                       │
│   └─→ Order appears in list within seconds                  │
│                                                              │
│ NO MANUAL REFRESH REQUIRED                                  │
│                                                              │
│ TABLE STRUCTURE:                                            │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ Order ID │ Customer │ Total │ Payment │ Shipping │ Inv. │  │
│ ├────────────────────────────────────────────────────────┤  │
│ │ #12345   │ John D.  │ €49.98│ Unpaid  │ Not Sent │ —   │  │
│ │ #12344   │ Jane S.  │ €24.99│ Paid    │ Shipped  │ 100 │  │
│ │ #12343   │ Bob M.   │ €99.97│ Partial │ Not Sent │ 99  │  │
│ │ (new)    │ ...      │ ...   │ ...     │ ...      │ ... │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
│ COLUMNS:                                                    │
│ • Order ID (descending by default - newest first)           │
│ • Customer Name                                             │
│ • Order Total (EUR)                                         │
│ • Payment Status:                                           │
│   ├─→ "Paid" (full payment received)                        │
│   ├─→ "Unpaid" (no payment)                                 │
│   └─→ "Partially Paid" (if Wawi supports)                   │
│ • Shipping Status:                                          │
│   ├─→ "Shipped" (label generated, order complete)          │
│   ├─→ "Not Shipped" (pending)                               │
│   └─→ Other granular states (if supported)                  │
│ • Invoice Number (first one shown; comma-separated values)  │
│   └─→ Empty if no invoice created yet                       │
│                                                              │
│ USER ACTIONS:                                               │
│ • Click order row → Step 1300 (Order Details)              │
│ • Table does NOT auto-scroll or auto-jump                   │
│ • Table sorts by ID descending on load                      │
│                                                              │
│ Backend: WAWI-88088 (Order sync endpoint)                   │
│        : CERP-994 (Eventbus orchestration)                  │
│ Frontend: CERP-980 (Order list display)                     │
│ QA: QCERP-513 / QWAWI-1311 / QWAWI-1339                     │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 1300: ORDER DETAILS - INVOICE                          │
│ Route: /orders/{orderId}                                    │
│ Design: CERP-962                                            │
│                                                              │
│ ORDER DETAIL VIEW:                                          │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ Order #12345 | Customer: John D.                      │  │
│ │ ────────────────────────────────────────────────────── │  │
│ │ Items:                                                 │  │
│ │ • Retro T-Shirt (Blue) x2 @ €24.99 = €49.98          │  │
│ │                                                        │  │
│ │ Total: €49.98                                          │  │
│ │ ────────────────────────────────────────────────────── │  │
│ │                                                        │  │
│ │ [Create Invoice] button (if no invoice exists)        │  │
│ │ OR                                                     │  │
│ │ [Invoice #100] [Print/Download PDF]                   │  │
│ │ [Invoice #101] [Print/Download PDF]                   │  │
│ │                                                        │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
│ INVOICE CREATION:                                          │
│                                                              │
│ 1. User clicks [Create Invoice]                            │
│ 2. Frontend calls: POST /wawi/invoices                      │
│    └─→ Wawi backend generates invoice                      │
│    └─→ Wawi stores invoice in database                     │
│    └─→ Returns invoice number (e.g., 100)                  │
│                                                              │
│ 3. Frontend calls: GET /wawi/invoices/{invoiceId}/pdf      │
│    └─→ Wawi backend generates PDF                          │
│    └─→ Gateway transmits PDF bytes to frontend             │
│                                                              │
│ 4. PDF rendered in browser print dialog                    │
│    └─→ User can print / print-to-file                      │
│    └─→ Download option available                          │
│                                                              │
│ 5. Invoice now visible in order details                    │
│    └─→ Can create additional invoices if needed             │
│                                                              │
│ DISPLAY:                                                    │
│ • Multiple invoices per order (if partial shipments)        │
│ • Most recent invoice shown first                          │
│ • Each invoice shows number + Print/Download actions        │
│                                                              │
│ Backend: WAWI-88098 (Create invoice endpoint)              │
│        : WAWI-88099 (Get invoice PDF endpoint)             │
│        : CERP-991 (Invoice PDF transmission)               │
│ Frontend: CERP-985 (Invoice creation UI)                    │
│ QA: QCERP-514 / QWAWI-1301 / QWAWI-1302                     │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 1400: ORDER DETAILS - PAYMENT                          │
│ Route: /orders/{orderId}#payment (section)                  │
│ Design: CERP-960                                            │
│                                                              │
│ PAYMENT STATUS DISPLAY:                                     │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ Payment Status                                         │  │
│ │ ────────────────────────────────────────────────────── │  │
│ │                                                        │  │
│ │ Status: ⚪ Unpaid                                       │  │
│ │ Amount: €49.98                                         │  │
│ │                                                        │  │
│ │ [Mark as Paid]  [Undo Payment]  [View Timeline]        │  │
│ │                                                        │  │
│ │ Payment Timeline:                                      │  │
│ │ • Order created: 2026-05-29 10:15 (no payment)        │  │
│ │                                                        │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
│ PAYMENT FLOW:                                              │
│                                                              │
│ Case 1: Customer paid via "Purchase on Account"             │
│   • Status shows: "Unpaid" (awaiting manual mark)           │
│   • Merchant clicks [Mark as Paid]                          │
│   • Frontend calls: POST /wawi/orders/{orderId}/mark-paid   │
│   • Status changes to: "Paid"                               │
│   • Timeline updated                                        │
│                                                              │
│ Case 2: Customer paid (if other gateways supported later)   │
│   • Status shows: "Paid" (auto-detected from payment)       │
│   • No action needed                                        │
│   • Timeline shows payment receipt time                     │
│                                                              │
│ Case 3: Merchant needs to undo payment                      │
│   • Clicks [Undo Payment]                                   │
│   • Status reverts to: "Unpaid"                             │
│   • Timeline updated                                        │
│                                                              │
│ PAYMENT METHOD (MVP):                                       │
│ • Only "Purchase on Account" supported                      │
│ • Payment gated: No automatic charge                        │
│ • Merchant manually marks when payment received             │
│ • No third-party gateway integration (Q1 constraint)        │
│                                                              │
│ Backend: WAWI-88101 (Mark payment status)                   │
│ Frontend: CERP-986 (Payment status UI)                      │
│ QA: QCERP-515 / QWAWI-1303                                  │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 1500: SHIPPING RELEASE LOGIC                           │
│ Route: /orders/{orderId}#shipping (implicit)                │
│ Design: CERP-966                                            │
│                                                              │
│ SHIPPING ELIGIBILITY CHECK:                                │
│                                                              │
│ Shipping button [Ship Order] is available IF:               │
│                                                              │
│ Condition A: Order is PAID                                 │
│    └─→ Status = "Paid" (customer payment received)         │
│    └─→ Merchant can immediately generate label             │
│                                                              │
│ Condition B: OR Order has no stock-managed items            │
│            AND Payment method = "Purchase on Account"        │
│    └─→ No stock items → Shipping allowed even if unpaid    │
│    └─→ Typical for merchants with immediate fulfillment    │
│                                                              │
│ SHIPPING DISABLED IF:                                       │
│ • Order is not paid AND payment method ≠ "Acct"             │
│ • Order contains stock-managed items (not in MVP)           │
│ • Order already shipped (status = "Shipped")                │
│                                                              │
│ DISABLED STATE:                                            │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ [Ship Order] (disabled/greyed out)                     │  │
│ │ Reason: "Payment required before shipping"             │  │
│ │ OR                                                     │  │
│ │ "Item(s) require payment or stock availability"        │  │
│ │                                                        │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
│ Backend: CERP-992 (Shipping eligibility check)             │
│ Frontend: CERP-986 (Shipping button enable/disable)         │
│ QA: QCERP-516                                               │
└──────────────────────────────────────────────────────────────┘
          ↓
         [USER CLICKS: "Ship Order"]
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 1600: SHIPPING DIALOG & LABEL GENERATION               │
│ Route: /orders/{orderId}/ship (modal dialog)               │
│ Design: CERP-964                                            │
│                                                              │
│ SHIPPING DIALOG FLOW:                                       │
│                                                              │
│ Step 1: Confirm Shipping                                    │
│ ┌────────────────────────────────────────────────────────┐  │
│ │ Confirm Order Shipment                                 │  │
│ │ ────────────────────────────────────────────────────── │  │
│ │                                                        │  │
│ │ Order #12345                                           │  │
│ │ Customer: John D.                                      │  │
│ │ Items:                                                 │  │
│ │ • Retro T-Shirt (Blue) x2                              │  │
│ │                                                        │  │
│ │ Shipping to:                                           │  │
│ │ John Doe                                               │  │
│ │ 123 Main St, Berlin 10115                              │  │
│ │                                                        │  │
│ │ Shipping method: DHL Paket Inland                      │  │
│ │                                                        │  │
│ │ [Proceed to Label] [Cancel]                            │  │
│ │                                                        │  │
│ └────────────────────────────────────────────────────────┘  │
│                                                              │
│ Step 2: Generate Shipping Label                             │
│                                                              │
│ 1. User clicks [Proceed to Label]                           │
│ 2. Frontend calls: POST /wawi/orders/{orderId}/delivery     │
│    └─→ Creates delivery note in Wawi                       │
│    └─→ Returns delivery note ID                            │
│                                                              │
│ 3. Frontend calls: POST /shipping/labels/generate (direct)  │
│    └─→ Calls JTL-Shipping App directly (v2.1 change)       │
│    └─→ Passes: Shipping details, delivery note ID          │
│    └─→ JTL-Shipping generates DHL label                    │
│    └─→ Returns PDF bytes                                   │
│                                                              │
│ 4. PDF opened in browser print dialog                       │
│    ┌────────────────────────────────────────────────────┐  │
│    │ Print Dialog                   [Print] [Cancel]    │  │
│    │ ────────────────────────────────────────────────── │  │
│    │ [DHL Shipping Label Preview]                        │  │
│    │ Order #12345                                        │  │
│    │ TRACKING: 1234567890123                             │  │
│    │ Recipient: John Doe, Berlin                         │  │
│    │                                                    │  │
│    └────────────────────────────────────────────────────┘  │
│                                                              │
│ 5. User prints label (physical printer or PDF save)         │
│                                                              │
│ 6. Order marked as "Shipped" in Wawi                        │
│    └─→ Order status sync'd to Shop                         │
│    └─→ Customer sees tracking info (if shop supports)      │
│                                                              │
│ LABEL GENERATION LOGIC (v2.1):                             │
│ • Direct call: ERP Cloud → JTL-Shipping App                │
│ • NOT via Wawi proxy (old architecture)                     │
│ • Faster, fewer intermediate hops                          │
│ • Architecture decision: PM alignment 2026-05-22            │
│                                                              │
│ API CALLS:                                                 │
│                                                              │
│ POST /wawi/orders/{orderId}/delivery                        │
│ {                                                           │
│   "action": "create_delivery_note"                          │
│ }                                                           │
│                                                              │
│ Response:                                                  │
│ {                                                           │
│   "deliveryNoteId": "dn-12345",                             │
│   "orderItems": [                                           │
│     {                                                       │
│       "itemId": "item-1",                                   │
│       "qty": 2,                                             │
│       "description": "Retro T-Shirt (Blue)"                 │
│     }                                                       │
│   ]                                                         │
│ }                                                           │
│                                                              │
│ POST /shipping/labels/generate                              │
│ {                                                           │
│   "deliveryNoteId": "dn-12345",                             │
│   "shippingMethod": "dhl_paket_inland",                     │
│   "recipient": {                                            │
│     "name": "John Doe",                                     │
│     "street": "123 Main St",                                │
│     "city": "Berlin",                                       │
│     "postalCode": "10115",                                  │
│     "country": "DE"                                         │
│   }                                                         │
│ }                                                           │
│                                                              │
│ Response (200):                                            │
│ {                                                           │
│   "labelId": "label-xyz",                                   │
│   "trackingNumber": "1234567890123",                        │
│   "pdfUrl": "https://cdn.../label-xyz.pdf",                │
│   "pdfBytes": "<binary PDF content>"                        │
│ }                                                           │
│                                                              │
│ Backend: WAWI-88101 (Delivery note creation)               │
│        : WAWI-88102 (Get shipping label endpoint)          │
│        : CERP-993 (Label generation orchestration)          │
│ Frontend: CERP-989 (Shipping dialog + print)                │
│ QA: QCERP-517 / QWAWI-1304 / QWAWI-1338                     │
└──────────────────────────────────────────────────────────────┘
          ↓
        [LABEL PRINTED]
          ↓
┌──────────────────────────────────────────────────────────────┐
│ POST-SHIPPING STATE                                          │
│                                                              │
│ • Order marked as "Shipped" in Wawi                          │
│ • Shipping status sync'd to Shop                             │
│ • Order status in list shows: "Shipped" ✓                    │
│ • Tracking number associated with order                     │
│ • Shop may display tracking link to customer                │
│                                                              │
│ → Order fulfillment complete!                               │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 1700: UI TEXT CHANGE                                    │
│ Minor change across UI:                                      │
│ "Kundschaft" (German) → "Kunden" (German)                   │
│ Terminology standardization                                 │
│ Backend: CERP-1097                                          │
│ Frontend: CERP-1098                                         │
│ QA: QCERP-518                                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔌 API Specification

### GET /orders
**Purpose:** Fetch order list with auto-sync  
**Base URL:** `https://api.jtl-cloud.com/`

```json
Response (200):
{
  "orders": [
    {
      "orderId": "order-12345",
      "customerId": "cust-001",
      "customerName": "John Doe",
      "totalAmount": 49.98,
      "currency": "EUR",
      "paymentStatus": "unpaid",
      "shippingStatus": "not_shipped",
      "invoiceNumbers": [],
      "created": "2026-05-29T09:30:00Z",
      "lastModified": "2026-05-29T09:30:00Z"
    },
    {
      "orderId": "order-12344",
      "customerId": "cust-002",
      "customerName": "Jane Smith",
      "totalAmount": 24.99,
      "currency": "EUR",
      "paymentStatus": "paid",
      "shippingStatus": "shipped",
      "invoiceNumbers": ["100"],
      "created": "2026-05-28T14:15:00Z",
      "lastModified": "2026-05-29T08:45:00Z"
    }
  ],
  "total": 2,
  "syncedAt": "2026-05-29T10:30:00Z"
}
```

### GET /orders/{orderId}
**Purpose:** Fetch order details

```json
Response (200):
{
  "orderId": "order-12345",
  "customerId": "cust-001",
  "customerName": "John Doe",
  "shippingAddress": {
    "street": "123 Main St",
    "city": "Berlin",
    "postalCode": "10115",
    "country": "DE"
  },
  "items": [
    {
      "itemId": "item-1",
      "name": "Retro T-Shirt (Blue)",
      "quantity": 2,
      "price": 24.99,
      "subtotal": 49.98
    }
  ],
  "totalAmount": 49.98,
  "paymentStatus": "unpaid",
  "shippingStatus": "not_shipped",
  "paymentMethod": "purchase_on_account",
  "shippingMethod": "dhl_paket_inland",
  "invoices": [],
  "canShip": true,
  "canMarkPaid": true
}
```

### POST /wawi/invoices
**Purpose:** Create invoice for order

```json
Request:
{
  "orderId": "order-12345"
}

Response (201):
{
  "invoiceId": "inv-100",
  "invoiceNumber": "100",
  "orderId": "order-12345",
  "amount": 49.98,
  "created": "2026-05-29T10:35:00Z"
}
```

### GET /wawi/invoices/{invoiceId}/pdf
**Purpose:** Get invoice PDF

```
Response (200):
Content-Type: application/pdf
Content-Disposition: attachment; filename="invoice-100.pdf"

[PDF binary content]
```

### POST /wawi/orders/{orderId}/mark-paid
**Purpose:** Mark order as paid

```json
Request:
{
  "action": "mark_paid"
}

Response (200):
{
  "orderId": "order-12345",
  "paymentStatus": "paid",
  "updatedAt": "2026-05-29T10:40:00Z"
}
```

### POST /wawi/orders/{orderId}/delivery
**Purpose:** Create delivery note

```json
Request:
{
  "action": "create_delivery_note"
}

Response (201):
{
  "deliveryNoteId": "dn-12345",
  "orderId": "order-12345",
  "items": [
    {
      "itemId": "item-1",
      "quantity": 2
    }
  ]
}
```

### POST /shipping/labels/generate
**Purpose:** Generate shipping label (direct to JTL-Shipping)

```json
Request:
{
  "deliveryNoteId": "dn-12345",
  "shippingMethod": "dhl_paket_inland",
  "recipient": {
    "name": "John Doe",
    "street": "123 Main St",
    "city": "Berlin",
    "postalCode": "10115",
    "country": "DE"
  }
}

Response (200):
{
  "labelId": "label-xyz",
  "trackingNumber": "1234567890123",
  "pdfUrl": "https://cdn.../label-xyz.pdf",
  "pdfBytes": "<binary>"
}
```

---

## 🔐 Security & Validation

### Order Access
- Only merchant can view their own orders
- M2M token scoped to tenant
- No cross-tenant data leakage

### Payment Status
- Payment status set only by merchant (manual marking)
- No automatic charge for "Purchase on Account"
- Audit trail maintained

### Shipping Label
- Label generated only after payment/eligibility check
- Tracking number issued by JTL-Shipping
- PDF secured with temporary download token

---

## 📈 Metrics

- **Order sync latency** (Time from Shop → Visible in list)
- **Invoice creation success rate**
- **Shipping label generation success rate**
- **Average time per order (receipt → shipped)**
- **Payment marked status distribution**

---

## 🔗 Dependencies

### External APIs
- **Wawi CloudAPI** → Order CRUD, invoices
- **JTL-Shipping App** → Label generation
- **Shop API** → Order status updates

### Internal Systems
- **Cloud Eventbus** → Order sync orchestration
- **Cloud Auth** → M2M validation
- **Payment Service** → Payment status management

---

## 📚 Related Documents

- **Full BRS:** `202605-brs.md` (complete spec)
- **Technical Concept:** `technical-concept.md` (architecture)
- **Product Setup Flow:** `02-product-setup-flow.md` (previous phase)
- **Confluence Step 1200-1700:** [Order Details](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1236926465)
