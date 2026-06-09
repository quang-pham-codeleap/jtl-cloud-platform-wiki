# Flow 2: Product Setup (Steps 800-1100)

**Duration:** ~5-10 minutes per item  
**User Role:** Merchant  
**Entry Point:** Onboarding success screen CTA OR Dashboard → Items menu  
**Goal:** Create first product and activate for JTL-Shop  

---

## 📐 User Journey Map

```
┌─────────────────────────────────────────────────────────────┐
│              PRODUCT SETUP FLOW                             │
│   (Simplified item creation + shop sync)                    │
└─────────────────────────────────────────────────────────────┘

[AFTER ONBOARDING COMPLETE]
          ↓
┌──────────────────────────────────────────────────────────────┐
│ ENTRY POINT                                                  │
│                                                              │
│ Option A: Success Screen CTA                                │
│ "Create your first item now" → Step 1000                    │
│                                                              │
│ Option B: Main Navigation                                   │
│ Dashboard → "Items" button → Step 900                       │
│                                                              │
│ Option C: Returning Merchant                                │
│ Dashboard → "Items" menu → Step 900                         │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 800: MAIN MENU NAVIGATION                              │
│ Route: /dashboard (implicit)                                │
│ Design: CERP-967                                            │
│                                                              │
│ Dashboard displays main menu with 4 sections:               │
│ • Dashboard (overview)                                      │
│ • Items (product management)                                │
│ • Orders (order fulfillment)                                │
│ • Invoices (billing overview - legacy Q1)                   │
│                                                              │
│ User actions:                                               │
│ • Click "Items" → navigates to Step 900 (empty state)      │
│                                                              │
│ Backend: None (routing only)                                │
│ Frontend: CERP-975                                          │
│ QA: QCERP-504                                               │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 900: ITEM LIST (EMPTY STATE)                           │
│ Route: /items                                               │
│ Design: CERP-961                                            │
│                                                              │
│ Empty state message:                                        │
│ "You haven't added any items yet"                           │
│                                                              │
│ Call-to-Action Button:                                      │
│ "Create your first item" → Steps to 1000                    │
│                                                              │
│ OR                                                          │
│                                                              │
│ Floating Action Button (FAB):                               │
│ "+" icon always visible → Steps to 1000                     │
│                                                              │
│ Once items exist:                                           │
│ • Displays item list table                                  │
│ • Columns: Name, Price, Status (active/inactive),           │
│   Shop Channel status, Last Modified                         │
│ • Click item → opens edit form (Step 1000 variant)          │
│ • Click "+" FAB → new item form                             │
│                                                              │
│ Backend: WAWI-88077 (Get items list)                        │
│ Frontend: CERP-977                                          │
│ QA: QCERP-505 / QWAWI-1292                                  │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 1000: ITEM CREATION FORM                               │
│ Route: /items/new OR /items/{itemId}/edit                   │
│ Design: CERP-969                                            │
│                                                              │
│ FORM STRUCTURE (Simplified MVP):                            │
│                                                              │
│ [General Tab] ← Active by default                           │
│ ┌─────────────────────────────────────────────────────┐    │
│ │ Item Name *                    [text input]         │    │
│ │                                                     │    │
│ │ Price (EUR) *                  [number input]       │    │
│ │                                                     │    │
│ │ [✓] Status: Active             [toggle]            │    │
│ │                                                     │    │
│ │ [EXPAND: Sales Channels]                           │    │
│ │ ┌─────────────────────────────────────────────┐   │    │
│ │ │ Channel: JTL-Shop (Active / Inactive toggle)│   │    │
│ │ │ [More channels if configured]               │   │    │
│ │ └─────────────────────────────────────────────┘   │    │
│ │                                                     │    │
│ │ Item Category *                [dropdown]          │    │
│ │                                [✓ Standard (default)]   │    │
│ │                                                     │    │
│ └─────────────────────────────────────────────────────┘    │
│                                                              │
│ [Image Tab] ← Secondary                                     │
│ ┌─────────────────────────────────────────────────────┐    │
│ │ Item Image                    [upload / preview]    │    │
│ │ (optional)                                          │    │
│ │                                                     │    │
│ │ [Upload] [Remove]                                  │    │
│ └─────────────────────────────────────────────────────┘    │
│                                                              │
│ [Description Tab] ← Secondary                               │
│ ┌─────────────────────────────────────────────────────┐    │
│ │ Description                   [rich text editor]    │    │
│ │ (optional)                                          │    │
│ │ [Formatting: Bold, Italic, Lists, Links]           │    │
│ └─────────────────────────────────────────────────────┘    │
│                                                              │
│ [Buttons at bottom]                                         │
│ [Save]  [Cancel]  [Save and Add Another]                   │
│                                                              │
│ VALIDATION RULES:                                          │
│ • Name: Required, min 1 char, max 255 chars                │
│ • Price: Required, ≥ 0, decimal precision 2                │
│ • Category: Required (defaults to "Standard")               │
│ • Image: Max 5MB, JPG/PNG/WebP only                         │
│ • Description: No validation (optional)                     │
│                                                              │
│ PRE-SELECTIONS:                                            │
│ • Status: Active (default)                                  │
│ • Category: Standard (if available)                         │
│ • Sales Channels: First shop auto-activated                 │
│                                                              │
│ Backend: WAWI-88089 (Save item, activate channel)          │
│        : WAWI-89221 (List available sales channels)         │
│ Frontend: CERP-976 (Item form UI)                           │
│ QA: QWAWI-1295 / QWAWI-1297                                │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ USER FILLS FORM                                              │
│                                                              │
│ Example:                                                    │
│ • Name: "Retro T-Shirt - Blue"                              │
│ • Price: "24.99"                                            │
│ • Image: [uploads image.jpg]                                │
│ • Description: "100% cotton, size S-XXL available"          │
│ • Category: Standard (pre-selected)                         │
│ • Sales Channels: JTL-Shop [✓ Active]                       │
│                                                              │
│ Validation on Save:                                         │
│ ✓ Name required → "Retro T-Shirt - Blue" ✓                  │
│ ✓ Price required → "24.99" ✓                                │
│ ✓ Price valid → "24.99" ✓                                   │
│ ✓ Image format → "image.jpg" ✓                              │
│ ✓ Category selected → "Standard" ✓                          │
│                                                              │
│ → All validations pass                                      │
│ → Click [Save]                                              │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ STEP 1100: SAVE & TRANSFER (Shop Sync)                      │
│ Route: /items (implicit, after save)                        │
│ Design: CERP-965                                            │
│                                                              │
│ SAVE SEQUENCE:                                             │
│                                                              │
│ 1. Frontend calls: POST /cloudapi/items                     │
│    └─→ Sends item data to Wawi CloudAPI                     │
│    └─→ Item persisted in Wawi database                      │
│    └─→ Backend returns itemId + status                      │
│                                                              │
│ 2. Frontend waits for response                              │
│    └─→ Shows loading spinner                                │
│    └─→ "Saving item..."                                     │
│                                                              │
│ 3. On successful persistence:                               │
│    └─→ Check if Sales Channel "JTL-Shop" is active          │
│    └─→ If YES: Emit save event to Eventbus                  │
│       └─→ Eventbus immediately triggers sync                │
│    └─→ If NO: Skip sync trigger                             │
│                                                              │
│ SHOP SYNC TRIGGER (Event-Based):                            │
│                                                              │
│ • Eventbus listens for item save events                      │
│ • Immediately calls: POST /wawi/sync/shop                   │
│ • Sync includes: Item data delta                            │
│ • Sync checks: JTL-Worker status                            │
│   ├─→ If Worker ACTIVE: Worker handles sync (suppress API) │
│   └─→ If Worker INACTIVE: API sync executes                │
│                                                              │
│ FRONTEND BEHAVIOR:                                          │
│                                                              │
│ • Loading message: "Saving item..."                         │
│ • Then: "Syncing to shop..."                                │
│ • Success: "Item created! Added to JTL-Shop"                │
│ • Redirect: Item list (Step 900)                            │
│ • OR: Next empty form (if "Save and Add Another")           │
│                                                              │
│ Backend: CERP-987 (Orchestration)                           │
│        : WAWI-88090 (Item save endpoint)                    │
│        : WAWI-88088 (Trigger shop sync)                     │
│ Frontend: CERP-978 (Save & redirect logic)                  │
│ QA: QCERP-511 / QWAWI-1299                                  │
│                                                              │
│ API CALL EXAMPLE:                                           │
│                                                              │
│ POST /cloudapi/items                                        │
│ {                                                           │
│   "name": "Retro T-Shirt - Blue",                           │
│   "price": 24.99,                                           │
│   "categoryId": "standard",                                 │
│   "imageUrl": "https://cdn.../image.jpg",                   │
│   "description": "100% cotton, size S-XXL",                 │
│   "salesChannels": [                                        │
│     {                                                       │
│       "channelId": "shop-001",                              │
│       "active": true                                        │
│     }                                                       │
│   ]                                                         │
│ }                                                           │
│                                                              │
│ Response (201 Created):                                     │
│ {                                                           │
│   "itemId": "item-12345",                                   │
│   "name": "Retro T-Shirt - Blue",                           │
│   "status": "persisted",                                    │
│   "syncTriggered": true                                     │
│ }                                                           │
│                                                              │
│ → Item now in Wawi                                          │
│ → Eventbus triggered sync                                   │
│ → Item syncs to JTL-Shop (async)                            │
│ → Appears in shop within seconds                            │
└──────────────────────────────────────────────────────────────┘
          ↓
┌──────────────────────────────────────────────────────────────┐
│ SUCCESS STATE                                                │
│                                                              │
│ User sees:                                                  │
│ • Confirmation: "Item created successfully!"                │
│ • Item list updated with new item                           │
│ • Can create another item                                   │
│ • Can proceed to Orders / Invoices                          │
│                                                              │
│ Background:                                                 │
│ • Item persisted in Wawi                                    │
│ • Item metadata synced to JTL-Shop                          │
│ • Item now visible in shop storefront                       │
│ • Item ready for customer orders                            │
│                                                              │
│ → Flow complete. Ready for Order Fulfillment flow.         │
└──────────────────────────────────────────────────────────────┘
```

---

## 🔌 API Specification

### POST /cloudapi/items
**Purpose:** Create or update an item in Wawi  
**Base URL:** `https://wawi-{tenantId}.jtl-cloud.com/cloudapi`

```json
Request Headers:
Authorization: Bearer <m2m-token>
Content-Type: application/json

Request Body:
{
  "name": "Retro T-Shirt - Blue",
  "price": 24.99,
  "status": "active",
  "categoryId": "standard",
  "imageUrl": "https://cdn.example.com/image.jpg",
  "description": "100% cotton, size S-XXL available",
  "salesChannels": [
    {
      "channelId": "shop-001",
      "channelName": "JTL-Shop",
      "active": true
    }
  ],
  "metaData": {
    "sku": null,
    "weight": null,
    "dimensions": null
  }
}

Response (201 Created):
{
  "itemId": "item-12345",
  "name": "Retro T-Shirt - Blue",
  "price": 24.99,
  "status": "active",
  "category": "standard",
  "salesChannels": [
    {
      "channelId": "shop-001",
      "channelName": "JTL-Shop",
      "active": true,
      "externalId": "shop-item-abc123"
    }
  ],
  "created": "2026-05-29T10:30:00Z",
  "lastModified": "2026-05-29T10:30:00Z"
}

Response (400 Bad Request):
{
  "error": "Validation failed",
  "details": [
    "Price must be ≥ 0",
    "Name is required"
  ]
}
```

### GET /cloudapi/sales-channels
**Purpose:** List available sales channels for activation  
**Base URL:** `https://wawi-{tenantId}.jtl-cloud.com/cloudapi`

```json
Response (200):
{
  "channels": [
    {
      "channelId": "shop-001",
      "channelName": "JTL-Shop",
      "type": "shop",
      "active": true,
      "url": "https://shop-xyz.jtl-cloud.com"
    },
    {
      "channelId": "channel-002",
      "channelName": "Marketplace Connector",
      "type": "connector",
      "active": false,
      "url": null
    }
  ]
}
```

### POST /wawi/sync/shop
**Purpose:** Trigger item sync to JTL-Shop  
**Base URL:** `https://api.jtl-cloud.com/`  
**Caller:** Cloud Eventbus (internal)

```json
Request:
{
  "action": "sync_items",
  "itemIds": ["item-12345"],
  "tenantId": "tenant-001",
  "triggerType": "event-based",
  "timestamp": "2026-05-29T10:30:05Z"
}

Response (202 Accepted):
{
  "syncId": "sync-xyz",
  "status": "queued",
  "itemCount": 1,
  "estimatedTime": "5 seconds"
}
```

---

## 💾 Data Model

### Item (Simplified for MVP)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `itemId` | UUID | Yes (auto) | Generated on creation |
| `name` | String | Yes | Max 255 chars |
| `price` | Decimal | Yes | EUR, precision 2 |
| `status` | Enum | Yes | "active", "inactive" |
| `categoryId` | UUID | Yes | References category; "standard" pre-selected |
| `imageUrl` | String | No | Max 5MB, JPEG/PNG/WebP |
| `description` | String | No | Rich text, no length limit |
| `created` | DateTime | Yes (auto) | ISO 8601 |
| `lastModified` | DateTime | Yes (auto) | ISO 8601 |

### Sales Channel Activation

| Field | Type | Notes |
|-------|------|-------|
| `channelId` | UUID | Identifies channel (e.g., shop-001) |
| `channelName` | String | Display name (e.g., "JTL-Shop") |
| `active` | Boolean | Item active on this channel? |
| `externalId` | String | Channel's item ID (e.g., shop-item-abc123) |

---

## 🔐 Security & Validation

### Input Validation
- **Name**: Non-empty, max 255 chars, sanitized (no XSS)
- **Price**: Decimal ≥ 0, precision 2
- **Image**: Max 5MB, whitelist MIME types (JPEG, PNG, WebP)
- **Description**: Sanitize HTML, prevent XSS
- **Category**: Must exist in system

### Authorization
- Only authenticated merchant can create items in their tenant
- M2M token validates tenant scope
- No cross-tenant data access

---

## 📊 Sync Behavior

### Event-Based Sync (Immediate)
- Triggered on item save
- Immediate post-save (within seconds)
- Includes only the saved item
- Real-time appearance in shop

### Interval Sync (Every 5 minutes)
- Fallback for missed events
- Includes all changes since last sync
- Runs via Eventbus scheduler
- Catches edge cases / retries

### Worker Status Check
- If JTL-Worker is ACTIVE: Worker handles sync (API call suppressed)
- If JTL-Worker is INACTIVE: API sync executes
- Prevents duplicate syncs

---

## ⚠️ Error Handling

### Validation Errors (400)
```json
{
  "error": "Validation failed",
  "field": "price",
  "message": "Price must be ≥ 0",
  "code": "INVALID_PRICE"
}
```

### Sync Errors (Asynchronous)
```
Item saved ✓
Sync triggered ✗ (Shop API timeout)
→ Retry in 5 minutes via interval sync
→ User can manually retry via dashboard
```

### Image Upload Errors (413)
```json
{
  "error": "File too large",
  "maxSize": "5MB",
  "uploadedSize": "6.2MB"
}
```

---

## 📈 Metrics

- **Item creation success rate** (Save → Persisted)
- **Sync trigger success rate** (Event fires → Shop receives)
- **Time to shop appearance** (Save → Visible in shop)
- **Form abandonment rate** (Started → Not submitted)
- **Image upload failure rate** (by file size, format)

---

## 🔗 Dependencies

### External APIs
- **Wawi CloudAPI** → Item CRUD
- **Shop API** → Item sync, channel management
- **CDN** → Image storage & serving

### Internal Systems
- **Cloud Eventbus** → Event-based sync
- **Cloud Auth** → M2M token validation
- **Merchant DB** → Item persistence

---

## 📚 Related Documents

- **Full BRS:** `202605-brs.md` (complete spec including item details)
- **Technical Concept:** `technical-concept.md` (architecture & APIs)
- **Order Fulfillment Flow:** `03-order-fulfillment-flow.md` (next phase)
- **Confluence Step 1000:** [Item Creation Form Details](https://jtl-software.atlassian.net/wiki/spaces/CLOUD/pages/1235255420)
