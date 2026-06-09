# App-Service API

**Base auth:** JWT Bearer on all endpoints except `GET /healthz` and `POST /webhooks/stripe`.

**Source:** `swagger/api-swagger.json` · `swagger/webhook-swagger.json`

---

## Domains

| Tag | Owns |
|-----|------|
| Publishers | Publisher registration, profile, Stripe onboarding |
| Customer | Customer Stripe billing portal |
| Partner | Partner-facing app listing queries |
| App-Listings | Listing CRUD, media upload, submission workflow |
| Apps | App instance lifecycle, metadata, subscription checkout |
| Store | Public store, app creation/provisioning/install, activation codes |
| Tenants | Tenant-scoped app queries |
| Webhooks | Stripe event receiver |
| Health | `GET /healthz` |

---

## Publishers

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/publishers/register` | Register new publisher. `legalAgreementAccepted` required. |
| `GET` | `/publishers` | List publishers. `?only-publishers-of-tenant=true` to scope. |
| `GET` | `/publishers/{id}` | Get publisher by ID. |
| `PATCH` | `/publishers/{id}/profile` | Partial update: name, focusAreas, business address, businessEmail. |
| `POST` | `/publishers/{id}/payment/onboarding-url` | Create Stripe Express onboarding URL. Creates Stripe account on first call. `409` if already fully connected. Single-use — never cache. |
| `POST` | `/publishers/{id}/payment/dashboard-url` | Single-use Stripe dashboard login link. Regenerated on every call. |
| `GET` | `/publishers/{id}/payment/invoices` | Cursor-paginated Stripe invoices. Filters: `status`, `createdAfter`, `createdBefore`. Returns `nextToken` for next page. |

**Publisher states:** `Pending` → `Active` / `Rejected` / `Suspended`

**Payment integration status:** `NotConnected` → `Pending` → `Connected`

---

## Customer

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/customer/billing/portal-session` | Stripe billing portal URL for the tenant's customer. Body: `{ publisherId }`. `404` if no Stripe customer for tenant. |

---

## Partner

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/partner/app-listings` | Paginated, localized listing overviews scoped to the authenticated partner's publisher. Filters: `locale`, `search`, `status`. |

---

## App-Listings

App listing versions have a status lifecycle: `Draft → Submitted → InReview → Approved / Rejected`

**Version behavior:** PUT on a `Draft` listing updates in-place. PUT on a `Submitted` listing creates a new version with a semver bump.

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/app-listings` | Create new listing for an app. Publisher must be Active. |
| `GET` | `/app-listings` | Query listings by `appId` + optional `locale`. |
| `GET` | `/app-listings/{slug}` | Get localized listing by slug. |
| `PUT` | `/app-listings/{slug}` | Update listing (see version behavior above). `204` if no changes. |
| `DELETE` | `/app-listings/{slug}` | Delete listing + all versions + all media blobs. Listing must be Draft; app must have no active installs. |
| `PUT` | `/app-listings/{slug}/submit` | Transition Draft → Submitted. App must be provisioned. `409` if not Draft. |
| `POST` | `/app-listings/{slug}/media/screenshots` | Upload screenshot (multipart). Returns `imageUrl`; append via PUT. Limits: max 1024×1024, JPG/PNG only. |
| `DELETE` | `/app-listings/{slug}/media/screenshots/{imageId}` | Remove screenshot by GUID embedded in blob name. Listing must be Draft or Submitted. |
| `POST` | `/app-listings/{slug}/media/icons` | Upload icon (multipart). Body: `imageType` (`IconLight`/`IconDark`) + `file`. Returns `imageUrl`. |

**Distribution types:** `Private` (default) / `Public`

---

## Apps

App instances are scoped per tenant. The composite response (`AppInstanceCompositeDto`) includes denormalized listing fields at root plus legacy BC fields `appInstance` and `listing` nested inside.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/apps` | Query app instances. Filters: `capability`, `promotionStatus`, `search`, pagination. |
| `GET` | `/apps/overviews` | Lightweight overview list for installed apps. |
| `GET` | `/apps/ids` | Array of installed app instance IDs for the current tenant. |
| `GET` | `/apps/{id}` | Get single app instance. |
| `PATCH` | `/apps/{id}` | Configure app instance. |
| `PUT` | `/apps/{id}/status` | Update app instance status. |
| `DELETE` | `/apps/{id}` | Uninstall app instance. `504` on Wawi timeout. |
| `GET` | `/apps/{id}/metadata` | Get app instance metadata (string dictionary). |
| `PATCH` | `/apps/{id}/metadata` | Partial update of metadata. |
| `PUT` | `/apps/{id}/metadata` | Replace entire metadata dictionary. |
| `GET` | `/apps/lifecycle-logs` | Paginated install/uninstall audit log. Filter by `appId`. Newest-first. |
| `GET` | `/apps/clients/{clientId}` | Get app info (listing + instance + publisher + scopes) by OAuth client ID. |
| `POST` | `/apps/{appId}/subscription/checkout-session` | Create Stripe Checkout session. App instance must be in `PaymentRequired` state. Returns client secret for embedded payment UI. `422` if publisher has no connected Stripe account. |

---

## Store

The Store tag owns the app entity (distinct from app instances). Apps start as Draft and are provisioned before listing.

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/store/apps` | Query public apps. Filters: `publisherId`, `promotionStatus`, `provisioningStatus`, `search`, pagination. |
| `POST` | `/store/apps` | Create new draft app. Terms must be accepted. |
| `GET` | `/store/apps/{id}` | Get app by ID. |
| `DELETE` | `/store/apps/{id}` | Delete app. No active instances allowed. |
| `PUT` | `/store/apps/{appId}/manifest` | Update app manifest. `204` if no changes. |
| `PUT` | `/store/apps/{appId}/provision` | Provision app (Draft → provisioned). Required before listing can be submitted. |
| `POST` | `/store/apps/{appId}/install` | Install app for current tenant. Returns app instance. `504` on Wawi timeout. |
| `POST` | `/store/apps/{slug}/images` | Upload icon image (multipart). `imageType`: `IconLight` / `IconDark`. `overwrite` flag controls duplicate behavior. |
| `DELETE` | `/store/apps/{slug}/images/{imageId}` | Delete icon by type identifier. |
| `GET` | `/store/apps/{appId}/activation-codes` | List activation codes for an app. |
| `POST` | `/store/apps/{appId}/activation-codes` | Generate activation code. |
| `DELETE` | `/store/apps/{appId}/activation-codes/{activationCode}` | Delete activation code. |
| `POST` | `/store/apps/{appId}/activation-codes/{activationCode}/deactivate` | Deactivate activation code (keeps record, disables use). |
| `POST` | `/store/activation-codes/{activationCode}/redeem` | Tenant redeems an activation code. `409` if already redeemed, at max redemptions, or inactive. |

---

## Tenants

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/tenants/apps` | Public apps + apps installed by the tenant. Requires `X-Tenant-Id` header. Filters: `publisherId`, `onlyAppsOfTenant`, `promotionStatus`, `provisioningStatus`, `search`. |

---

## Webhooks

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/webhooks/stripe` | Stripe event receiver. Verifies `Stripe-Signature` header, schedules `HandleStripeWebhookCommand`. No auth token required. `412` on signature verification failure. |

---

## Pagination

Two styles used:

- **Offset** (`pageNumber`, `pageSize`, `returnAll`) — used by Apps, Store, App-Listings, Tenants. Response: `{ items, totalItems }`.
- **Cursor** (`pageSize`, `nextToken`) — used by publisher invoices only. Response: `{ items, nextToken }`. Pass `nextToken` from previous response to get next page.

---

## Key enums

| Enum | Values |
|------|--------|
| `PublisherState` | `Pending`, `Active`, `Rejected`, `Suspended` |
| `PaymentIntegrationStatus` | `NotConnected`, `Pending`, `Connected` |
| `AppListingVersionStatus` | `Draft`, `Submitted`, `InReview`, `Approved`, `Rejected` |
| `DistributionType` | `Private`, `Public` |
| `ImageType` | `IconLight`, `IconDark` |
