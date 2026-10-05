---
id: INT-01
title: Shopify store connection
domain: Store and partner connections
audience: [brand, system]
apps: [web, background]
related: [CMP-01, CMP-04, CMP-05, CMP-06, AFX-01, AFX-03, ACC-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, affiliate/ainflu-affiliate-backend@v1.5.27
---

# INT-01 · Shopify store connection

> A brand connects its Shopify store so it can build a campaign around a Shopify product, import products into affiliate and gifting programs, and ship products to influencers from Shopify. In today's app the connection runs **through the Ainfluencer Affiliate platform**: Shopify is authorised there, and the store is then copied to Ainfluencer automatically. A brand can also arrive from the Shopify app, which creates or opens its Ainfluencer account.

## Who and where

- **Brand**, web app:
  - paid-collaboration wizard `/create-campaign`, store step: "I am a Shopify user" → "Select Your Shopify Store" / "Connect To Shopify" (CMP-01);
  - affiliate dashboard "Programs" tab → "Create Shopify Affiliate Program" / "Connect a store" (CMP-04);
  - gifting wizard, "Products" step (CMP-05);
  - inbox "Product Tracking" box, which reconnects when needed (CMP-06).
- **Brand arriving from Shopify:** opening the Ainfluencer app in Shopify logs the brand in and sends it to `/oauth/set-password` ("Create Password for Your Ainfluencer Account") before anything else.
- **System:** Shopify webhooks (app uninstalled, GDPR requests) and the affiliate platform's "store synced" message.

## How it works

1. The brand opens the Shopify connection window ("Connect To Shopify": "Sync your store and products to easily set up your affiliate program with custom commissions."). It enters its store address. Help is under "Where is my Shopify store URL?": "Go to Settings, then navigate to Domains" in Shopify to find the `.myshopify.com` address.
2. A pop-up opens Shopify, where the brand approves the app. This happens on the affiliate platform, which registers the brand there as a seller if needed (AFX-01).
3. When Shopify finishes, the affiliate platform:
   - saves the store and imports its products;
   - tells Ainfluencer the store is connected. If Ainfluencer doesn't confirm, it retries every 5 minutes until it does.

   Ainfluencer then saves its own copy of the connection. The window shows "Syncing..." and then "Shopify account synced successfully.".
4. The store now appears in "Select Your Shopify Store" ("{count} Products"), with "Or Add another Shopify account" to connect more stores.
5. **Campaign from a Shopify product (CMP-01):**
   - the brand picks the store, then "Select a product from your Shopify store to create a campaign for";
   - "Proceed with AI" pre-fills the title, description, countries, gender, hashtags and product type from the product, and copies up to 9 product images as campaign content. "Skip AI & Continue" skips this;
   - the product can't be changed later; a new campaign is needed for another product.
6. **Uninstalling the app in Shopify** deletes the Shopify connection in Ainfluencer. On the affiliate platform, an hourly check marks the store inactive and switches its products off (AFX-03).

## Rules and limits

- Only `*.myshopify.com` store addresses are accepted.
- A brand can connect several stores. Each campaign is tied to the store it was created from.
- Products in the picker come live from Shopify. A store with no products shows "You have no products available to create a campaign. Please go to your Shopify store and add products." with "Go to Shopify Store".
- Campaign creation and fulfilment need a valid connection. If the store was uninstalled or the access was revoked, the brand sees "Your Shopify store isn’t connected yet. Please connect your store to continue." and must reconnect.
- Deleting a store on the affiliate side is refused while any partnership contract exists on it (AFX-03).
- Shopify sales are tracked on the affiliate platform with a pixel (AFS topics). Ainfluencer itself doesn't read Shopify orders, except the free orders it creates for influencer shipping (CMP-06).

## Statuses

| What the brand sees | Meaning |
|---|---|
| Store listed in "Select Your Shopify Store" | Connected on the affiliate platform (and normally in Ainfluencer) |
| "Syncing..." | Waiting for the affiliate platform to finish the import |
| "Connect Your Shopify Store" prompt | No valid connection for this account |

Affiliate side: store `status` 1 active / 2 inactive / 3 suspended. `core_synced_at` is empty until Ainfluencer confirms the copy (AFX-03).

## Timers and automatic actions

| Job | When (UTC) | Effect |
|---|---|---|
| `shopify:reconcile-core-sync` (affiliate) | every 5 min | Re-sends "store synced" to Ainfluencer until confirmed |
| `shopify:check-store-connections` (affiliate) | hourly | Uninstalled app or closed shop → store inactive, products deactivated |
| Shopify "app uninstalled" webhook (Ainfluencer) | when Shopify sends it | Ainfluencer Shopify connection deleted |

## Notifications

None specific to connecting. Users only see on-screen messages.

## Common questions

### "I connected Shopify but the store doesn't appear in the campaign wizard."
**Answer for the user:** The import can take a few minutes. Please close the window, wait a moment and open "Select Your Shopify Store" again.
**What to check:** affiliate platform: store exists and is active. `core_synced_at` set? Core: a `user_third_parties` row with provider `shopify` for this brand and store domain.
**Escalate when:** more than ~15 minutes after connecting the store is still missing in Ainfluencer (the core sync has failed).

### "Where do I find my myshopify URL?"
**Answer for the user:** In Shopify admin go to Settings → Domains. Your address ends in `.myshopify.com`.
**What to check:** n/a.
**Escalate when:** n/a.

### "Shopify sent me to a 'Create Password' page."
**Answer for the user:** Your Ainfluencer account was opened from Shopify. Set a password so you can also log in directly at app.ainfluencer.com.
**What to check:** n/a (ACC-01).
**Escalate when:** saving the password fails.

### "I reinstalled the Shopify app and my campaigns can't ship anymore."
**Answer for the user:** Uninstalling disconnects the store. Please connect it again from the campaign or the affiliate dashboard; existing campaigns will use the new connection.
**What to check:** a new Shopify connection row and access token exist.
**Escalate when:** reconnected and shipping or product loading still fails.

### "Products are missing or out of date."
**Answer for the user:** Use "Re-sync Products" in the affiliate dashboard. Products that are inactive or deleted in Shopify won't show.
**What to check:** product status in Shopify and on the affiliate platform.
**Escalate when:** active Shopify products are still missing after a re-sync.

## Known issues

- **Two copies of the connection.** The store lives on the affiliate platform and is copied to Ainfluencer. If the copy fails, Shopify campaigns and shipping fail even though the store looks connected in the affiliate dashboard. Escalate to dev.
- **Uninstall is handled differently on each side:** Ainfluencer deletes its copy at once, while the affiliate platform marks the store inactive within the hour.

## Related topics

- CMP-01: Shopify campaigns. CMP-04: affiliate programs. CMP-05: gifting products. CMP-06: shipping via Shopify.
- AFX-01: affiliate account matching. AFX-03: store connections and health checks on the affiliate platform.
- ACC-01: account and sign-in.

## Technical reference

- **Connection (web → core → affiliate):**
  - `POST user-third-party/ainfluencer-affiliate/v1/shopify-account/sync` (`AinfluencerAffiliateService::syncShopifyAccount`, returns the Shopify OAuth `redirect_link`), `GET …/shopify-account/stores/list`, `GET …/shopify-account/stores/check/{account_id}`, browser return `GET …/shopify-account/callbacks/sync`.
  - The affiliate webhook `profile.shopify_account.synced` (HMAC-signed `state`) → `HandleShopifyAccountCoreSyncedJob` → `handleShopifyAccountCoreSynced` → `ShopifyService::login` (upserts `user_third_parties` provider `shopify` + `user_third_party_tokens`).
- **Older direct path:** `POST user-third-party/shopify/v1/connect` (Shopify authorize URL), web route `GET user-third-party/shopify/auth/callback` → `ShopifyController@authCallback` (redirects to `/create-campaign?adId&step=2` or `/oauth/set-password`). Shopify app home `GET user-third-party/shopify/` (`EnsureShopifyInstalled`, `EnsureShopifyAuthenticated`).
- **Products and campaign:** `GET user-third-party/shopify/v1/products` (`shop_name`), `POST …/campaign/start-process` (marks ad `third_party=shopify`), `…/save-product` (`ads.extra.shopify_product_id`), `…/finish-process` (AI fill via `ShopifyCampaign`, up to 9 product images as media).
- **Webhooks:** `POST user-third-party/shopify/webhook/app-uninstall` (verified, `UserThirdPartyService::deleteThirdParty`), `…/get-customer`, `…/erase-customer` (GDPR, stubs).
- **Data:** core `user_third_parties` (`provider`=`shopify`, `third_party_unique_identity` = shop domain, `user_id`), `user_third_party_tokens` (`access_token`), `user_third_party_accounts`, `ads.third_party`, `ads.user_third_party_id`. Affiliate: Shopify store rows (`status`, `shop_store_status`, `core_synced_at`).
- **Key files:** `core/backend/api/app/Modules/UserThirdParty/Http/Controllers/ShopifyController.php`, `…/Services/ShopifyService.php`, `…/Http/Controllers/ShopifyWebhookController.php`; `core/frontend/web/src/components/common/modals/shopifyConnection/ShopifyConnectionModal.tsx`, `core/frontend/web/src/components/pages/createAd/parts/froms/parts/createCampaignThirdParty/index.tsx`, `core/frontend/web/src/components/pages/shopifyCreatePassowrd/index.tsx`.
