---
id: AFX-03
title: Store connections (Amazon, Shopify)
domain: Ainfluencer ↔ affiliate platform
audience: [brand, seller, system]
apps: [web, seller-app, background]
related: [CMP-04, INT-01, AFX-01, AFX-02, AFS-01, AFS-02, AFS-03, AFS-06, AFS-12]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFX-03 · Store connections (Amazon, Shopify)

> Every affiliate program starts from a connected store: an **Amazon** seller/vendor account (through Amazon Ads and Amazon Attribution) or a **Shopify** store. The connection always lives on the affiliate platform, whether the brand starts it in the Ainfluencer web app or in the seller app. This topic covers connecting, the automatic health checks, keeping Ainfluencer in sync, and removing a store. The seller-app screens (store switcher, Manage accounts) are in AFS-02.

## Who and where

- **Ainfluencer web app (brand):** `/affiliate/dashboard` → "Programs" tab, or `/affiliate/sync_account` ("Connect stores"). Also the Shopify step in campaign creation (`/create-campaign`). Connection opens in a popup (Amazon) or a redirect (Shopify).
- **Seller app:** Profile → "Manage accounts" (`/app/profile?tab=manage_accounts`), the onboarding step "Connect Your Stores to Setup Your Affiliate Program", the header switcher → "Add New Store", and My Brands → "Add Shopify/Amazon Store". See AFS-02.
- **System:** hourly Shopify checks, Amazon refresh jobs, and messages between the two platforms.

## How it works

**Amazon**
1. The brand clicks connect ("Connect Your Amazon" / "Connect New Amazon Store"). The brand is registered on the affiliate platform first if needed (AFX-01).
2. Amazon's login page opens. The brand signs in with the Amazon account that has **Amazon Ads** access and approves.
3. The platform reads the account's advertising profiles and keeps only **seller or vendor** profiles in Amazon Attribution marketplaces: **US, CA, GB, DE, FR, IT, ES, MX**. Each kept profile starts as "syncing".
4. For each profile it fetches the advertiser and the Attribution tag, marks the profile **active**, and imports the products and brands (default commission 15%, AFS-03). If Amazon refuses, the profile becomes **disconnected** with Amazon's error message.
5. Ainfluencer creates ad groups named `ainfluencer_…` in the seller's Amazon Ads account to track affiliate links. They must not be deleted or renamed.
6. If the brand started in Ainfluencer, the popup returns to the Ainfluencer page, which records the account and shows it in "Programs".

**Shopify**
1. The brand enters the shop's `.myshopify.com` domain. The Shopify authorisation page opens; the brand installs/approves the app.
2. The platform saves the store (as "active"), registers Shopify webhooks, installs the **Ainfluencer web pixel** (sales tracking) and starts the product import.
3. It then notifies Ainfluencer server-to-server ("Shopify store synced"). Ainfluencer saves its own Shopify connection too, the same one used for gifting and campaigns (INT-01). If Ainfluencer doesn't confirm, the platform retries every 5 minutes.
4. If the brand started from campaign creation, they are sent back to `/create-campaign` at step 2.

**Removing a store**
- From Ainfluencer ("Programs" → delete) or from the seller app (Amazon only, see AFS-02).
- The platform **refuses** if any partnership contract exists on that account's profiles/stores, **even revoked ones**.
- Otherwise it deletes the account's partnership requests, promotions, products and brands, and tells Ainfluencer, which removes its copy.

## Rules and limits

- **Amazon:** only seller/vendor profiles in US, CA, GB, DE, FR, IT, ES or MX can be connected. With no usable profile the connection fails ("No Amazon seller profiles found…" or "None of your Amazon seller profiles are in a marketplace that supports Amazon Attribution…").
- **Shopify:** one store can belong to only one seller account ("This store is already linked to another account…"). A wrong domain gives "We could not find a Shopify store at "…". Please enter your permanent .myshopify.com domain…".
- **Plan:** connecting Amazon is checked against the plan's seller-account allowance (AFS-12; unlimited on the current plan data *(to verify)*). Shopify connections are not limited.
- **Deletion is blocked by any contract** ever made on the store (active or revoked). There is no self-service way around it.
- Re-connecting an existing Amazon account or Shopify store updates it in place (same account, same products).

## Statuses

| Item | Status (internal) | Meaning |
|---|---|---|
| Amazon account | `connected`, `disconnected`, `deleted` | Disconnected after Amazon refuses access during a report fetch, or by a plan downgrade |
| Amazon profile (one per marketplace) | `syncing`, `active`, `disconnected`, `inactive`, `deleted` | "syncing" while importing; "disconnected" shows Amazon's error on hover; "inactive" from a plan downgrade |
| Shopify store | `status` 1 active, 2 inactive, 3 suspended | Inactive after uninstall, lost access, shop not found, or the seller deactivating the store |
| Shopify shop state | `shop_store_status`: open, paused, frozen, dormant, closed, unknown | Copied from Shopify's plan info |
| Shopify ↔ Ainfluencer sync | `core_synced_at` empty | Ainfluencer hasn't confirmed yet; retried every 5 min |

## Timers and automatic actions

| Job | When (UTC) | Effect |
|---|---|---|
| `shopify:check-store-connections` | hourly | Calls Shopify for each store. Access refused (401/403) or no access token → store **inactive**, shop "closed", **all its products deactivated**. Shop not found (404) → same. Otherwise refreshes email, currency, timezone, plan. A passing check does **not** re-activate an inactive store. |
| Shopify `app/uninstalled` webhook | immediately | Same as above: store inactive, products deactivated, pixel removed |
| `shopify:reconcile-core-sync` | every 5 min | Re-sends "store synced" to Ainfluencer for active stores not yet confirmed (older than 5 min) |
| `seller-account:update-all` | daily | For **active** Amazon profiles last synced 48 h+ ago: refreshes advertiser/tag and re-imports products. Disconnected profiles are not retried automatically. |
| `seller-account:update-report` | daily 06:00 | Pulls Amazon Attribution sales for the last 60 days for profiles not updated in 72 h (AFS-09, AFS-10) |

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Amazon profile set to "syncing" | Seller | Web notification |
| Amazon/Shopify account synced | Seller | Email |
| Products/brand deactivated because the store went inactive | Partnered affiliates | Web notification / email (AFS-03) |

## Common questions

### "Amazon says my account can't be connected / no profiles found."
**Answer for the user:** We can only connect Amazon seller or vendor accounts that have Amazon Ads access in a marketplace supported by Amazon Attribution (US, Canada, UK, Germany, France, Italy, Spain, Mexico). Please sign in with the Amazon account that owns the Seller Central/Ads profile for one of those marketplaces.
**What to check:** which Amazon login the user used; whether they have an Ads profile; the marketplace.
**Escalate when:** the account clearly has an eligible seller profile and the error persists.

### "My Amazon profile shows Disconnected."
**Answer for the user:** Amazon refused our access for that marketplace. Use "Recheck" on Profile → Manage accounts; if that doesn't help, connect the Amazon account again.
**What to check:** `amazon_account_profiles.status` and `error_message`; the account `status`.
**Escalate when:** reconnecting succeeds but the profile goes back to disconnected.

### "My Shopify store became inactive and all my products are hidden."
**Answer for the user:** The hourly check found the Ainfluencer app was uninstalled or lost access, or the shop is closed. Please reconnect the store. After reconnecting, check that your products are active again.
**What to check:** `shopify_account_stores.status` = 2, `shop_store_status`, the time it changed; whether the app is still installed in Shopify admin.
**Escalate when:** the app is installed and the store keeps turning inactive.

### "I can't delete my store."
**Answer for the user:** A store that has ever had an affiliate partnership can't be removed, because those partnerships and their sales history depend on it. You can deactivate its brands/products instead so affiliates can't promote them.
**What to check:** any row in `partnership_contracts` for the account's profiles/stores (any status). The user may see a raw message text (see Known issues).
**Escalate when:** the user insists on removal (dev decision).

### "I connected Shopify in Ainfluencer but the campaign page doesn't see my store."
**Answer for the user:** Give it a few minutes; the two systems confirm the connection in the background and retry every 5 minutes.
**What to check:** `shopify_account_stores.core_synced_at`; Ainfluencer `user_third_parties` (provider shopify) and `user_third_party_accounts` (type shopify).
**Escalate when:** still not confirmed after 30 minutes.

### "My new Amazon products don't show up."
**Answer for the user:** Products are re-imported automatically about every 48 hours. You can also press "Re-Sync" on Profile → Manage accounts (available once a day).
**What to check:** profile `synced_at`, `status` = active.
**Escalate when:** synced in the last day and products are still missing.

## Known issues

- **Deleting a store shows a raw text** like "messages.exceptions.platform_account_has_partnership_contracts" instead of a readable message (translation missing). Meaning: contracts exist; deletion is blocked.
- **Deleting a Shopify store from the seller app fails** (no backend route). Workaround: delete it from the Ainfluencer "Programs" tab (still blocked if contracts exist). See AFS-02.
- **Amazon errors started from Ainfluencer are generic.** The specific Amazon reason isn't passed back to the Ainfluencer page *(to verify on screen)*; ask the user to try in the seller app to see the exact message.
- **Wrong account returned to Ainfluencer** *(to verify)*: when a seller with several Amazon accounts connects another one from Ainfluencer, the return link carries the seller's first Amazon account.
- **Store doesn't reactivate by itself** after Shopify access is restored; the seller must reconnect, and deactivated products must be re-activated.
- **Re-imports reset product settings** *(to verify)*: product re-imports (Amazon every 48 h, Shopify on every product change) write the default 15% commission and the platform's in-stock status back onto existing products. See AFS-03.

## Related topics

- AFS-02 — Seller-app screens for stores and the store switcher.
- AFS-03 — Products, brands and commission that come from the store.
- CMP-04 — Brand affiliate setup in Ainfluencer.
- INT-01 — Ainfluencer's own Shopify connection (created at the same time).
- AFX-01 — Accounts are created before connecting.
- AFS-12 — Plan limits.

## Technical reference

- **Core endpoints (web):** `POST api/user-third-party/ainfluencer-affiliate/v1/amazon-account/sync`, `GET …/amazon-account/profiles/check/{account_id}`, `POST …/shopify-account/sync`, `GET …/shopify-account/stores/check/{account_id}`, `GET …/shopify-account/stores/list`, `GET …/affiliate-programs/list`, `DELETE …/account/delete/{account_id}`, `GET …/status`. Browser returns: `GET …/amazon-account/callbacks/sync`, `GET …/shopify-account/callbacks/sync` (HMAC-checked `state`). Webhook `POST …/v1/webhook` events `profile.amazon_account.deleted`, `profile.shopify_account.deleted`, `profile.shopify_account.synced` (→ `HandleShopifyAccountCoreSyncedJob`).
- **Core service:** `app/Modules/UserThirdParty/Services/AinfluencerAffiliateService.php` (`syncAmazonAccount`, `callbackSyncAmazonAccount`, `syncShopifyAccount`, `callbackSyncShopifyAccount`, `handleShopifyAccountCoreSynced`, `listAccount` reconciles local rows, `deleteAccount`, `webhookDelete*`). Manual backfill: `SyncShopifyTokenToInternalApiJob` from `BrandsWithShopifyStoreCommand` → affiliate `shopify-account/sync-with-token`.
- **Affiliate endpoints:** `seller/v1/integrations/ainfluencer/{amazon-account/sync, amazon-account/delete, amazon-account/profiles/check, amazon-account/profiles/list, shopify-account/sync, shopify-account/sync-with-token, shopify-account/delete, shopify-account/stores/check, shopify-account/stores/list, accounts/list}`; seller app `POST seller/v1/profile/sync/account/{amazon,shopify}`, `POST …/sync/account/amazon/profile/{amazonProfileId}` (Recheck/Re-Sync), `DELETE seller/v1/profile/synced-amazon/{userEmailId}`; OAuth returns `GET web/v1/seller/login/amazon/callback`, `GET web/v1/seller/login/shopify/callback`; Shopify `POST client/v1/callback/shopify/webhooks` (HMAC), pixel `POST client/v1/callback/shopify/track/event`; admin `POST admin/v1/integrations/ainfluencer/accounts/core-users`.
- **Key files (affiliate):** `Services/Actions/Seller/Auth/Social/AmazonLoginCallbackService.php` (profile filter, eligible marketplaces in `Services/General/Amazon/MarketplaceMapping.php`), `Jobs/AmazonProfile/SyncAmazonProfileJob.php`, `Services/Actions/Seller/Auth/Social/ShopifyLoginCallbackService.php`, `Services/General/Shopify/ShopifyAccountService.php`, `ShopifyStoreSyncSideEffectsService.php`, `ShopifyStoreConnectionCheckService.php`, `ShopifyStoreIntegrationStatusService.php`, `Webhook/Handlers/*`, `Services/General/Amazon/DeleteAmazonAccount.php`, `Services/General/Shopify/DeleteShopifyAccount.php`, `NotifyCoreShopifyStoreSyncedService.php`.
- **Data (affiliate):** `amazon_accounts.status`, `amazon_account_profiles` (`status`, `error_message`, `synced_at`, `country_code`, `amazon_profile_id`), `shopify_accounts`, `shopify_account_stores` (`status`, `shop_store_status`, `shop_name`, `core_synced_at`), `user_emails` (type amazon/shopify; the seller-app delete uses this id), `partnership_contracts` (`accountable_type/id` blocks deletion), `webhook_logs`. **Core:** `user_third_parties`, `user_third_party_accounts` (`type` amazon/shopify, `account_id`).
- **Config:** `ainfluencer.seller.update_amazon_profile.time` = 48 h, `amazon_update_report_period` = 60 days, `default_commission` = 15.
