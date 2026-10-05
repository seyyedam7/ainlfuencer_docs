---
id: AFS-02
title: Connecting and switching stores
domain: Seller app (brands)
audience: [seller]
apps: [seller-app]
related: [AFX-03, AFS-01, AFS-03, AFS-04, AFS-07, AFS-12, AFS-13]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-02 · Connecting and switching stores

> In the seller app every page works on **one selected store at a time** (an Amazon marketplace profile or a Shopify store). This topic covers the "Manage accounts" screen where stores are added, re-synced and removed, and the store switcher in the header. What happens behind a connection (OAuth, health checks, sync with Ainfluencer) is in AFX-03.

## Who and where

- **Seller** (or a team member with "Edit profile → Accounts" permission, AFS-13).
- **Manage accounts:** user menu → "Edit profile" → tab "Manage accounts" (`/app/profile?tab=manage_accounts`). Sections "Amazon Ads Accounts" and "Shopify Accounts".
- **Store switcher:** the dropdown at the top of every page. It lists the connected stores and ends with "Add New Store" (opens Manage accounts).
- Other entry points: the setup wizard (AFS-01), My Brands → "Add Shopify/Amazon Store".

## How it works

**Adding a store**
1. "Connect New Amazon Store" (or "Connect Amazon Store" when none exists) opens Amazon's login in a popup.
2. "Connect New Shopify Store" asks for the shop domain, then redirects to Shopify.
3. On return the store appears in the list and in the switcher. Amazon shows one row per marketplace profile with "Store Name", "Marketplace", "Currency", "# of Brands", "# of Products".

**Switching**
1. Pick a store in the header dropdown. The choice is remembered in the browser and the page reloads (from a brand or product detail page it goes to the dashboard first).
2. Every screen then shows only that store's data: brands/stores, products, partnerships, promotions, reports.
3. Wording follows the store type: Amazon shows "Brand"/"My Brands" and ASINs; Shopify shows "Store"/"My Stores" and products.

**Keeping Amazon data fresh**
- **"Recheck"** appears when a profile isn't active (e.g. disconnected). It retries the Amazon connection for that profile ("Amazon account profile rechecked successfully").
- **"Re-Sync"** appears on active profiles once a day (24 h after the last sync). It refreshes the profile; products are re-imported only if the last import is older than 48 hours ("Amazon account profile synced successfully", then the page reloads). Products are also refreshed automatically every 48 h (AFX-03).

**Removing a store**
- Amazon: "Delete" on the account. Blocked if any partnership contract (active or revoked) exists on its profiles. Otherwise its brands, products, promotions and pending requests are deleted.
- Shopify: the "Delete" button doesn't work (see Known issues).

**Amazon notice**: My Brands has an "Important notice" button ("Important Notice from Ainfluencer"): Ainfluencer creates ad groups and campaigns in the seller's Amazon Ads account for Attribution tracking, and those ad groups must not be deleted or renamed. Changing them can break tracking and therefore commission billing.

## Rules and limits

- **Nothing shows until a store is selected.** If no store has been chosen, the app picks one automatically after login; with no connected store, the layout stays empty.
- **Account limit:** "Seller Accounts Limit: used / limit" on Manage accounts. At the limit the app shows "Account Limit Reached – Upgrade your plan to add more accounts" and the server may answer with the "Seller Account Limit Reached" pop-up. Only Amazon connections are counted on the server (AFS-12).
- Amazon: only seller/vendor profiles in US, CA, GB, DE, FR, IT, ES, MX (AFX-03).
- Shopify: a store can belong to only one seller account.
- Re-Sync: once per 24 h per profile (button hidden in between).
- Deletion: blocked by any existing contract (AFX-03).

## Statuses

| Shown | Internal | Meaning |
|---|---|---|
| Green "Connected" dot | Amazon account `connected` / Shopify store active (1) | Working |
| Red / disconnected dot | Amazon account `disconnected` / Shopify store inactive (2) | Needs reconnect |
| Profile status text with an info icon | Profile `syncing`, `disconnected` (+ `error_message`), `inactive` | Hover the icon to see Amazon's error |

## Timers and automatic actions

- Amazon products: refreshed every 48 h; sales reports every 72 h (AFX-03, AFS-09).
- Shopify stores: checked hourly; an uninstalled app or closed shop makes the store inactive and hides all its products.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Amazon profile starts syncing | Seller | Web notification |
| Store connected | Seller | Email |

## Common questions

### "The seller app is blank after I log in."
**Answer for the user:** The app needs a store selected. Choose one in the dropdown at the top, or connect a store under Profile → Manage accounts.
**What to check:** the seller has at least one active Amazon profile or Shopify store; ask them to clear the site data if the remembered store was deleted.
**Escalate when:** a store exists and is active but the layout stays blank.

### "I see different brands/products/partnerships than yesterday."
**Answer for the user:** Everything in the seller app follows the store selected in the top dropdown. Switch to the other store to see its data.
**What to check:** which store is selected; number of connected stores.
**Escalate when:** data is missing for the correct store.

### "I can't delete my Amazon store."
**Answer for the user:** A store that has had affiliate partnerships can't be removed. You can deactivate its brands or products instead (My Brands / My Products).
**What to check:** contracts on the profiles (any status). The error text may look like a code (AFX-03 Known issues).
**Escalate when:** the user needs it removed anyway.

### "Delete does nothing for my Shopify store."
**Answer for the user:** Removing a Shopify store isn't possible from the seller app right now. If you also use Ainfluencer, you can remove it from the Affiliate dashboard → Programs tab, provided it has never had affiliate partnerships. Otherwise contact support.
**What to check:** contracts on the store; whether the user has an Ainfluencer brand account.
**Escalate when:** removal is needed and the Ainfluencer route isn't available (dev).

### "The Re-Sync button disappeared."
**Answer for the user:** Re-Sync is available once a day per store. Products also refresh automatically every two days.
**What to check:** profile `synced_at`.
**Escalate when:** more than 48 h passed since the last sync and products are stale.

### "Should I worry about ad groups in my Amazon Ads account?"
**Answer for the user:** Yes, please don't delete or rename the ad groups Ainfluencer created. They're how affiliate sales are tracked; changing them can stop tracking and commissions.
**What to check:** if sales stopped appearing, ask whether ad groups were changed.
**Escalate when:** tracking stopped after a change (dev/Amazon investigation).

## Known issues

- **Deleting a Shopify store fails** (the app calls an address the server doesn't have; error 404). Workaround above.
- **The two Amazon notices disagree on the ad-group prefix**: My Brands says "ainfluencer_", the profile notice says "algo_" *(to verify which is current)*. Advise not touching ad groups created by Ainfluencer under either name.
- **Re-Sync within 48 h doesn't re-import products** (only refreshes the connection), although it says "synced successfully".
- **Re-imports reset product commission to 15% and the in-stock status** *(to verify)*, see AFS-03.

## Related topics

- AFX-03 — What happens during connection, health checks, deletion rules.
- AFS-01 — Store step in the setup wizard.
- AFS-03 — Products and brands of the selected store.
- AFS-12 — Seller account limit by plan.
- AFS-13 — Permission "Edit profile → Accounts" for team members.

## Technical reference

- **Endpoints:** `GET seller/v1/profile/platform-accounts` (`SellerPlatformAccountController`, list with status), `GET seller/v1/profile/platform-accounts/dropdown` (switcher), `POST seller/v1/profile/sync/account/amazon` (plan gate `seller_account`), `POST seller/v1/profile/sync/account/shopify` (no plan gate), `POST seller/v1/profile/sync/account/amazon/profile/{amazonProfileId}` (Recheck/Re-Sync → `SyncAmazonAccountProfileService`: status `syncing` + `SyncAmazonProfileJob`), `DELETE seller/v1/profile/synced-amazon/{userEmailId}` (`DeletePlatformAccountService` → `PlatformService::deleteAccount` → `DeleteAmazonAccount`), `GET seller/v1/profile/profiles` (Amazon profile dropdown), `GET seller/v1/profile/synced-accounts`. Missing route: `DELETE seller/v1/profile/synced-shopify/{id}`.
- **Header `X-Platform`:** every seller-app request carries the selected store (`amazon-<marketplace>` or `shopify`); `ValidateMultiPlatform` middleware turns it into `platform` (+ marketplace).
- **Resources:** `Http/Resources/Seller/V1/Profile/AmazonProfileAccountResource.php`, `ShopifyAccountStoreResource.php` (`resynced_after` = last sync + 1 day).
- **Data:** `seller_platform_accounts`, `amazon_accounts`, `amazon_account_profiles` (`status`, `error_message`, `synced_at`), `shopify_accounts`, `shopify_account_stores` (`status`, `shop_store_status`), `user_emails` (type amazon/shopify).
- **Frontend:** `shared-components/algo-layout/layout-header/amazon-dropdown.component.vue` (switcher, stored in `localStorage.platform`), `shared-components/algo-components/profile/profile-manage-social-accounts/*` (tables, Recheck/Re-Sync), `shared-components/algo-components/profile/profile.i18n.js`, `spa-seller/src/components/_utils/algorift-notice/*` (Amazon notice).
