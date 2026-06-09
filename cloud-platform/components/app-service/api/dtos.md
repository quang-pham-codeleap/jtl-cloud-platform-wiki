# App-Service Key DTOs

Covers the load-bearing DTOs used across most conceptual and implementation work. Leaf/simple request objects (empty bodies, single-field wrappers) are omitted — see the swagger for those.

**Base fields (inherited by all `EntityBaseDto` types):** `id: string`, `createdAt: datetime`, `updatedAt: datetime`

---

## Publishers

### `PublisherDto` ← `EntityBaseDto`
| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `name` | string | no | Display name |
| `state` | `PublisherState` | no | `Pending \| Active \| Rejected \| Suspended` |
| `focusAreas` | string[] | no | Partner focus areas |
| `businessAddressLine1` | string | yes | |
| `businessAddressLine2` | string | yes | |
| `postalCode` | string | yes | |
| `city` | string | yes | |
| `country` | string | yes | ISO 3166-1 alpha-2 |
| `businessEmail` | string | yes | |
| `createdBy` | string | yes | User ID |
| `updatedBy` | string | yes | User ID |
| `paymentAccount` | `PublisherPaymentAccountDto` | yes | null until Stripe onboarding initiated |

### `PublisherPaymentAccountDto`
| Field | Type | Notes |
|-------|------|-------|
| `status` | `PaymentIntegrationStatus` | `NotConnected \| Pending \| Connected` |

### `RegisterPublisherRequest`
| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | |
| `focusAreas` | string[] | yes | |
| `businessEmail` | string | yes | |
| `businessAddressLine1` | string | yes | |
| `businessAddressLine2` | string | no | |
| `postalCode` | string | yes | |
| `city` | string | yes | |
| `country` | string | yes | ISO 3166-1 alpha-2 |
| `legalAgreementAccepted` | boolean | yes | Must be true |

### `UpdatePublisherProfileRequest` (PATCH — all fields optional)
| Field | Type | Notes |
|-------|------|-------|
| `name` | string | |
| `focusAreas` | string[] | |
| `businessAddressLine1` | string | |
| `businessAddressLine2` | string | |
| `postalCode` | string | |
| `city` | string | |
| `country` | string | ISO 3166-1 alpha-2 |
| `businessEmail` | string | |

### `QueryPublisherPaymentInvoicesResponse`
| Field | Type | Notes |
|-------|------|-------|
| `invoiceNumber` | string | Stripe invoice number, e.g. `INV-0001` |
| `customerName` | string | Stripe customer display name |
| `appTechnicalName` | string | Empty string if no app association |
| `createdAt` | datetime | UTC |
| `status` | string | `draft \| open \| paid \| void \| uncollectible` |
| `hostedInvoiceUrl` | string | Stripe-hosted invoice URL |

---

## App Listings

### `AppListingDto` ← `EntityBaseDto`
| Field | Type | Notes |
|-------|------|-------|
| `slug` | string | Immutable after first publish. SEO URL slug, e.g. `trylens` |
| `appListingVersion` | `AppListingVersionDto` | The current version (non-localized, full dict form) |
| `publisherId` | string | |
| `publisherName` | string | |

### `AppListingVersionDto` ← `EntityBaseDto`
All i18n fields are **locale-keyed dictionaries** (e.g. `{ "en-GB": {...}, "de-DE": {...} }`).

| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `appId` | string | yes | Matches app manifest AppId |
| `version` | string | yes | Publisher-managed display version, e.g. `1.0.0` |
| `listingVersion` | string | yes | System-managed manifest version; auto-bumped on structural change |
| `defaultLocale` | string | yes | Fallback locale, e.g. `en`. Must exist in `name`/`description` |
| `name` | `{ [locale]: NameDto }` | yes | |
| `description` | `{ [locale]: DescriptionDto }` | yes | |
| `benefits` | `{ [locale]: string[] }` | yes | Max 5 per locale |
| `category` | `CategoryDto` | yes | |
| `media` | `MediaDto` | yes | |
| `compatibility` | `CompatibilityDto` | yes | |
| `pricing` | `PricingDto` | yes | |
| `support` | `SupportDto` | yes | |
| `legal` | `LegalDto` | yes | |
| `editorial` | `EditorialDto` | yes | Platform-controlled, not settable by publisher |
| `distributionType` | `DistributionType` | no | `Private \| Public`. Default: `Private` |
| `listingVersionStatus` | `AppListingVersionStatus` | no | `Draft \| Submitted \| InReview \| Approved \| Rejected` |

### `AppListingInputDto` (shared body for POST + PUT on `/app-listings`)
Same shape as `AppListingVersionDto` minus `appId`, `listingVersion`, and `listingVersionStatus` (those are server-managed). `distributionType` is settable here.

### Sub-objects for listing version content

**`NameDto`**
| Field | Notes |
|-------|-------|
| `short` | Short display name for list views |
| `full` | Full name for detail page |

**`DescriptionDto`**
| Field | Notes |
|-------|-------|
| `short` | Teaser shown in search/list views |
| `full` | Full description on detail page |

**`CategoryDto`**
| Field | Notes |
|-------|-------|
| `main` | Exactly one main category |
| `sub` | One or more subcategories |

**`MediaDto`**
| Field | Type | Notes |
|-------|------|-------|
| `icons` | `MediaIconsDto` | Light + dark mode icons |
| `screenshots` | string[] | Azure Blob Storage URLs. Min 1, max 10. JPG/PNG, max 1024×1024 |
| `video` | string | Optional YouTube URL |

**`MediaIconsDto`**
| Field | Notes |
|-------|-------|
| `light` | Storage reference for light mode icon |
| `lightImageTimestamp` | Cache-busting timestamp |
| `dark` | Storage reference for dark mode icon |
| `darkImageTimestamp` | Cache-busting timestamp |

**`CompatibilityDto`**
| Field | Type | Notes |
|-------|------|-------|
| `products` | `{ [locale]: string[] }` | Supported JTL products, localized |
| `productsNotes` | string | Free-text note, not localized |
| `minVersions` | `{ [product]: string }` | e.g. `{ "JTL-Wawi": "1.9.0" }`. Not localized |

**`PricingDto`**
| Field | Type | Notes |
|-------|------|-------|
| `subscriptionTiers` | `SubscriptionTierDto[]` | Max 4. At most one free, at most one `isMostPopular` |

**`SubscriptionTierDto`**
| Field | Type | Notes |
|-------|------|-------|
| `id` | string | Tier identifier |
| `content` | `{ [locale]: SubscriptionTierContentDto }` | Localized name, description, features |
| `isFree` | boolean | At most one per listing |
| `monthlyPriceEur` | decimal | Required if not free |
| `annualPriceEur` | decimal | Optional; if any paid tier has it, all paid tiers must |
| `isMostPopular` | boolean | At most one per listing |
| `trial` | `SubscriptionTierTrialDto` | Optional |
| `hasSubscribers` | boolean | From billing system; tier is read-only when true |

**`SubscriptionTierContentDto`**
| Field | Notes |
|-------|-------|
| `name` | Tier name |
| `description` | Tier description |
| `features` | Max 10 feature strings |

**`SubscriptionTierTrialDto`**
| Field | Notes |
|-------|-------|
| `enabled` | Whether trial is active |
| `preset` | Trial duration preset string |
| `days` | Number of trial days |

**`SupportDto`**
| Field | Type | Notes |
|-------|------|-------|
| `url` | `{ [locale]: string }` | Support URL, localized |
| `documentation` | `{ [locale]: string }` | Docs URL, localized. Optional |

**`LegalDto`**
| Field | Notes |
|-------|-------|
| `privacyPolicy` | URL |
| `termsOfUse` | URL |
| `gdpr` | `GdprDto` — `request` URL + `delete` URL |

**`EditorialDto`** (platform-controlled)
| Field | Notes |
|-------|-------|
| `status` | boolean — App Store visibility |
| `publishedAt` | datetime — null until first publication |

---

## Apps (store entity)

### `AppDto` ← `EntityBaseDto`
| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `technicalName` | string | no | |
| `activeVersion` | `AppVersionDto` | no | |
| `publisherId` | string | no | |
| `publisherName` | string | no | |
| `clientId` | string | no | OAuth client ID |
| `slug` | string | yes | null until listing created |
| `tc` | `TermsAndConditionsDto` | yes | `acceptedAt` + `version` |
| `listingVersionStatus` | `AppListingVersionStatus` | yes | Mirror from listing; null when no listing |
| `mediaIcons` | `MediaIconsDto` | yes | Mirror from listing |
| `legal` | `LegalDto` | yes | Mirror from listing |
| `pricing` | `PricingDto` | yes | Mirror from listing |
| `isPubliclyVisible` | boolean | no | Derived: `Approved AND Public AND Editorial.status=true` |
| `defaultLocale` | string | yes | Mirror from listing |

### `AppCompositeDto` ← `AppDto`
Adds two BC-only legacy fields alongside the flat `AppDto` fields:

| Field | Type | Notes |
|-------|------|-------|
| `app` | `AppDto` | Nested copy of the same data — BC only, do not use for new code |
| `appId` | string | |
| `listing` | `AppListingLocalizedDto` | Localized listing block — BC only |

### `AppVersionDto` ← `EntityBaseDto`
| Field | Type | Notes |
|-------|------|-------|
| `manifest` | `AppManifestDto` | |
| `promotionStatus` | `AppPromotionStatus` | `None \| JtlGroup \| VerifiedPartner` |
| `provisioningStatus` | `AppProvisioningStatus` | `Draft \| Provisioned \| Suspended` |

### `AppManifestDto`
| Field | Type | Notes |
|-------|------|-------|
| `manifestVersion` | string | System-generated |
| `version` | string | App version |
| `appId` | string | |
| `technicalName` | string | |
| `requirements` | `AppManifestRequirementsDto` | `minCloudApiVersion` |
| `lifecycle` | `AppManifestLifecycleDto` | `configurationUrl`, `connectUrl`, `disconnectUrl` |
| `capabilities` | `AppManifestCapabilitiesDto` | Hub + ERP capabilities |

**`AppManifestCapabilitiesDto`**
- `hub.appLauncher` → `redirectUrl`, `closedPreviewUrl`
- `erp.menuItems[]` → `id`, `name`, `url`
- `erp.api.scopes[]` → string array
- `erp.pane[]` → `id`, `context`, `title`, `matchChildContext`, `url`, `requiredScopes[]`
- `erp.webhooks` → `apiVersion`, `subscriptions[].topics[]`, `subscriptions[].endpoint.url`

### `AppManifestInputDto` (request body for PUT `/store/apps/{appId}/manifest`)
Same as `AppManifestDto` but without `appId` and `manifestVersion` — those are server-assigned.

---

## App Instances

### `AppInstanceDto` ← `EntityBaseDto2`
| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `installationStatus` | `InstallationStatus` | no | `Uninstalled \| Installed \| Failed` |
| `configurationStatus` | `ConfigurationStatus` | no | `RequiresReconfiguration \| Unconfigured \| Configured \| Failed \| PaymentRequired` |
| `scopes` | string[] | no | Granted scopes, format: `api.scopes.{scope}` |
| `features` | string[] | no | Granted features, format: `{product}.{capability}.{feature}` |
| `metadata` | `{ [key]: string }` | no | App-developer key-value store |
| `slug` | string | yes | null until listing created |
| `publisherId` | string | no | |
| `publisherName` | string | no | |
| `activeVersion` | `AppVersionDto` | no | |
| `redeemedAt` | datetime | yes | Set when installed via activation code |
| `subscription` | `SubscriptionDto` | yes | null when no active subscription |
| `listingVersionStatus` | `AppListingVersionStatus` | yes | Mirror from listing |
| `mediaIcons` | `MediaIconsDto` | yes | Mirror from listing |
| `legal` | `LegalDto` | yes | Mirror from listing |
| `pricing` | `PricingDto` | yes | Mirror from listing |
| `isPubliclyVisible` | boolean | no | Derived visibility flag |
| `defaultLocale` | string | yes | Mirror from listing |

### `AppInstanceCompositeDto` ← `AppInstanceDto`
Adds two BC-only legacy fields alongside the flat `AppInstanceDto` fields:

| Field | Type | Notes |
|-------|------|-------|
| `appInstance` | `AppInstanceDto` | Nested copy of the same data — BC only, do not use for new code |
| `appInstanceId` | string | |
| `listing` | `AppListingLocalizedDto` | Localized listing block — BC only |

### `SubscriptionDto`
| Field | Type | Nullable | Notes |
|-------|------|----------|-------|
| `id` | string | no | Active subscription tier (plan) ID |
| `name` | string | no | Display name of the tier |
| `priceType` | `SubscriptionTierPriceType` | no | `Monthly \| Annual` |
| `status` | string | yes | Stripe lifecycle status: `active \| past_due \| canceled` |
| `dueDate` | datetime | yes | End of current billing period |
| `cancelAt` | datetime | yes | Scheduled cancellation date, if set |
| `payment` | `SubscriptionPaymentDto` | yes | null until first payment confirmed |

**`SubscriptionPaymentDto`**
| Field | Notes |
|-------|-------|
| `status` | `paid \| past_due \| canceled`. Intentionally excludes internal Stripe IDs |

### `UpdateAppInstanceMetadataRequest` (PATCH `/apps/{id}/metadata`)
| Field | Type | Notes |
|-------|------|-------|
| `set` | `{ [key]: string }` | Keys to add or overwrite |
| `remove` | string[] | Keys to delete |

---

## Shared enums (quick reference)

| Enum | Values |
|------|--------|
| `PublisherState` | `Pending`, `Active`, `Rejected`, `Suspended` |
| `PaymentIntegrationStatus` | `NotConnected`, `Pending`, `Connected` |
| `AppListingVersionStatus` | `Draft`, `Submitted`, `InReview`, `Approved`, `Rejected` |
| `DistributionType` | `Private`, `Public` |
| `ImageType` | `IconLight`, `IconDark` |
| `InstallationStatus` | `Uninstalled`, `Installed`, `Failed` |
| `ConfigurationStatus` | `RequiresReconfiguration`, `Unconfigured`, `Configured`, `Failed`, `PaymentRequired` |
| `AppPromotionStatus` | `None`, `JtlGroup`, `VerifiedPartner` |
| `AppProvisioningStatus` | `Draft`, `Provisioned`, `Suspended` |
| `SubscriptionTierPriceType` | `Monthly`, `Annual` |
