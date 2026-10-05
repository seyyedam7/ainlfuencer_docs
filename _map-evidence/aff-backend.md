# Ainfluencer Affiliate backend — business-logic inventory (support flow map)

Source: `aff-backend` snapshot at production tag **v1.5.27** (Laravel, Postgres). Route data: `aff_routes.json` (895 routes).
Core counterpart: `src/api/app/Modules/UserThirdParty`. Machine-readable contract: `aff-core-contract.json`.
All paths below are relative to `aff-backend/` unless prefixed `core:` (= `src/api/`).

The system has four user types (README): **Admin, Seller (brand), Affiliate (influencer/creator), Buyer** (end shopper on `buy.ainfluencer.com`). A single `users` row can hold several `user_types` rows (seller and affiliate are separate profiles of the same login). Most "user" ids in the API are **UserType** ids, hash-encoded (`decode_hash_id` middleware).

---

## 1. Core ↔ Affiliate contract

### 1.1 How core authenticates to the affiliate backend
- Core driver: `core:app/Modules/UserThirdParty/Driver/AinfluencerAffiliateDriver.php`. Every call sends `X-API-KEY: user_third_party.ainfluencer_affiliate.api_key`, JSON, and retries 3× at 100 ms, **only** on connection errors or 5xx (a 4xx or 429 is not retried).
- Base paths: `{base}/seller/v1/integrations/ainfluencer`, `{base}/affiliate/v1/integrations/ainfluencer`, `{base}/admin/v1/integrations/ainfluencer`.
- Affiliate side: all these routes use the **`api_key`** middleware → `app/Http/Middleware/ApiKeyAuth.php` → `app/Services/General/ApiKey/ApiKeyAuthChecker.php`:
  - The key is read from `X-API-Key`, or from `Authorization: Bearer`. The `api_keys` row is cached for 300 s and must be `is_active`.
  - Key `type` must equal the route prefix (`seller` / `affiliate` / `admin`) **or be `INTERNAL`**. Admin routes also accept an `ADMIN` key.
  - Optional permission check derived from the route name (`{prefix}.v1.<section>.<action>`, matched against `api_keys.permissions`).
  - Per-key rate limit per minute (`api_keys.rate_limit`), enforced with a cache counter. Exceeding it returns **429**.
  - **Admin routes** run as the seeded super-user admin (`config('ainfluencer.super_user.username')`).
  - **Seller/affiliate routes** run as the user who owns UserType `key.user_type_id ?? request.auth_id ?? request.reference_id`. Core always passes `reference_id` = the affiliate-side UserType hash id it stored in `user_third_parties.third_party_unique_identity`.
- Extra gates on some integration routes: `seller_stripe_connected` (`app/Http/Middleware/Billing/SellerStripeAccountConnectedMiddleware.php`) returns **HTTP 499** "stripe_account_not_connected" if the seller has no saved Stripe payment method. This gate is on `campaigns/create`. `seller_access_feature:<feature>` (`SellerAccessFeatureMiddleware`) returns **HTTP 402** when the plan limit is reached, unless `IS_CHECK_PLAN_FEATURES=false` or the request sets `status=inactive`. This gate is on `products/bulk-update-commission` (commission) and `products/bulk-change-status` (brand, asin). `validate_multiple_platforms` reads the `X-Platform` header (`amazon-<marketplace>` list or `shopify`).

### 1.2 Core → Affiliate calls (all 45 matched to a v1.5.27 route)

| # | Method + path (affiliate) | Core caller (driver ← service) | Affiliate controller | Notes |
|---|---|---|---|---|
| 1 | POST `/seller/v1/integrations/ainfluencer/register` | `registerSeller` ← `AinfluencerAffiliateService::register` (job `RegisterUserInAinfluencerAffiliateJob`) | `Seller\…\RegisterIntegrationController` → `RegisterIntegrationService` | Find or create the seller by personal email (30 s lock). Verifies the email, records `users.reference_id` = core user id (`CoreUserRecorder`), syncs `stripe_id`. Returns user_id, user_type_id, amazon/shopify account ids |
| 2 | POST `/affiliate/v1/integrations/ainfluencer/register` | `registerAffiliate` ← `registerAffiliate` (mobile influencer) | `Affiliate\…\RegisterIntegrationController` | Same idea for the affiliate profile |
| 3 | POST `…/seller/…/amazon-account/sync` | `syncAmazonAccount` | `SyncAmazonAccountIntegrationController` → `SyncAmazonAccountService` | Returns `redirect_link` (Amazon Ads OAuth). The OAuth callback lands at `web/v1/seller/login/amazon/callback` |
| 4 | POST `…/amazon-account/delete` | `deleteAmazonAccount` ← `deleteAccount` | `DeleteAmazonAccountIntegrationController` → `DeletePlatformAccountService` → `General/Amazon/DeleteAmazonAccount` | **Refused if any PartnershipContract (even revoked) exists on the account's profiles.** Otherwise deletes requests, promotions, products, brands and email, then emits webhook `profile.amazon_account.deleted` |
| 5 | POST `…/amazon-account/profiles/check` | `checkAmazonAccountProfile` | `CheckAmazonAccountProfileIntegrationController` | |
| 6 | GET `…/amazon-account/profiles/list` | `listAmazonAccountProfile` | `IndexAmazonAccountProfileIntegrationController` | |
| 7 | POST `…/shopify-account/sync-with-token` | `syncShopifyAccountWithToken` | `Client\Callback\V1\Shopify\ReceiveShopifyTokenSyncController` | Core hands over a Shopify access_token obtained by core's own Shopify app install |
| 8 | POST `…/shopify-account/sync` | `syncShopifyAccount` | `SyncShopifyAccountIntegrationController` | Returns Shopify OAuth `redirect_link`. The callback lands at `web/v1/seller/login/shopify/callback` |
| 9 | POST `…/shopify-account/delete` | `deleteShopifyAccount` | `DeleteShopifyAccountIntegrationController` → `General/Shopify/DeleteShopifyAccount` | Emits `profile.shopify_account.deleted` |
| 10 | POST `…/shopify-account/stores/check` | `checkShopifyAccountStore` | `CheckShopifyAccountStoreIntegrationController` | |
| 11 | GET `…/shopify-account/stores/list` | `listShopifyAccountStore` | `IndexShopifyAccountStoreIntegrationController` | |
| 12 | GET `…/accounts/list` | `listAccount` (also `hasAmazonAccount`/`hasShopifyAccount`) | `IndexAccountIntegrationController` | |
| 13 | GET `…/products/list` (X-Platform amazon-xx / shopify) | `listAmazonProduct` / `listShopifyProduct` | `IndexProductsIntegrationController` | paginated |
| 14 | PUT `…/products/bulk-update-commission` | `bulkUpdateCommissionProduct` | `BulkUpdateCommissionProductIntegrationController` | plan gate `commission` (402). Fires `ProductCommissionsUpdated` |
| 15 | POST `…/products/bulk-change-status` | `bulkChangeStatusProduct` | `BulkUpdateSellerProductStatusIntegrationController` | plan gates `brand`, `asin` (402) |
| 16 | POST `…/products/bulk-generate-link` | `bulkGenerateLink{Amazon,Shopify}Product` | `BulkGenerateLinkProductIntegrationController` | **Auto-approves** a partnership request for the affiliate if one is not already APPROVED, then creates links |
| 17 | GET `…/brands/drop-down` | `list{Amazon,Shopify}BrandDropdown` | `BrandDropDownIntegrationController` | |
| 18 | GET `…/cards/list` | `listCard` / `hasCard` | `IndexCardsIntegrationController` | Core uses this as its campaign "payment method" gate |
| 19 | POST `…/cards/sync` | `syncCard` | `SyncCardIntegrationController` → `CreateBillingPortal` (source=ainfluencer) | Returns a Stripe Billing Portal URL. Return URL is `AINFLUENCER_SELLER_SYNC_CARD_FRONT_RETURN_URL` (core front) |
| 20 | POST `…/campaigns/create` | `createCampaign` | `CreateCampaignIntegrationController` | `seller_stripe_connected` (499). Campaign is created **ACTIVE** with `source_campaign_id` = core campaign id. Sends `campaign.updated` |
| 21 | POST `…/campaigns/assign-items` | `assignCampaignItems` | `AssignCampaignToItemsIntegrationController` | sends `campaign.updated` |
| 22 | POST `…/campaigns/products/assign` | `assignCampaignAmazonProductToUser` | `AssignCampaignProductsToUsersIntegrationController` | Auto-approves the partnership request and creates the contract and links. Sends `campaign.updated` |
| 23 | GET `…/campaigns/products/items` | `listCampaignAmazonProductItem` | `IndexCampaignProductsItemsIntegrationController` | (also dispatches `campaign.updated`) |
| 24 | GET `/affiliate/…/campaigns/list` | `listCampaignsForAffiliate` | `Affiliate\…\ListCampaignIntegrationController` | |
| 25 | GET `/seller/…/affiliate-programs/list` | `listAffiliateProgramsForAffiliate` ← `listAffiliateProgram` | `IndexAffiliateProgramIntegrationController` | |
| 26 | GET `/affiliate/…/campaigns/{id}/products` | `listCampaignAmazonProductItemForAffiliate` | `Affiliate\…\IndexCampaignProductsItemsIntegrationController` | |
| 27 | POST `/affiliate/…/campaigns/assign-affiliate` | `assignCampaignAmazonProductToUserForAffiliate` (influencer joins a campaign) | `AssignCampaignToAffiliateIntegrationController` | Auto-approves the partnership request and creates the contract and links. Sends `campaign.updated` |
| 28–31 | GET `/admin/…/campaigns/{id}/products/list`, GET `…/{id}/affiliates/list`, DELETE `…/delete/{id}`, PUT `…/update/{id}` | `…ForAdmin` (core admin panel) | `Admin\V1\Integrations\Ainfluencer\Campaign\*` | update may set any `CampaignStatusEnum` value |
| 32–34 | GET `/seller/…/campaigns/{id}/affiliates/list`, DELETE `…/delete/{id}`, PUT `…/update/{id}` | `…ForSeller` | `Seller\…\Campaign\*` | list returns `meta` campaign totals |
| 35–38 | GET `/affiliate/…/reports/summary`, `…/reports/timeline`, `…/reports/products`, `…/payouts` | `report*ForAffiliate`, `listPayoutsForAffiliate` (core Mobile `AffiliateReport` controllers) | `Affiliate\…\Report\*`, `Payout\IndexPayoutIntegrationController` | payouts = CommissionTransactions where the affiliate is receiver, shown **net of platform fee** (see §5) |
| 39–41 | POST `…/coupons/create`, GET `…/coupons/list`, GET `…/coupons/show` | `createCoupon`, `listCoupon`, `showCoupon` | `Seller\…\Coupon\*` → `CreatePromotionService` | Coupon = `SellerPromotion`, created **ACTIVE + ACCEPTED** (skips the admin approval that UI-created Shopify promotions need) |
| 42–43 | GET `/admin/…/digests/sellers`, GET `/admin/…/digests` | `listDigestSellersForAdmin`, `listSellerDigestsForAdmin` (core BrandDigest `AffiliateDigestSource`, daily) | `Admin\…\Digest\*` | |
| 44 | POST `/admin/…/accounts/core-users` | `linkCoreUsersForAdmin` (core `LinkAffiliateCoreUsersCommand`) | `LinkCoreUsersIntegrationController` | Back-fills `users.reference_id` (only where NULL) |
| 45 | GET (browser) `/web/v1/sso/login?token=…` | `getRedirectUrl` ← core `authorize-url` endpoints | `Web\Sso\SsoLoginController` | see SSO below |

Integration routes core never calls: `admin/v1/integrations/ainfluencer/campaigns/list` and `seller/v1/integrations/ainfluencer/campaigns/list`.

### 1.3 Affiliate → Core (outbound)

| Event / call | Trigger (affiliate side) | Core endpoint |
|---|---|---|
| SSO acknowledge: POST `{acknowledge_url}` with a JWT (`event=user_created_acknowledge`, aff `user_id`, `unique_identity`=UserType id, `type`). Body `{user_id: core id}` | Every successful `web/v1/sso/login` whose token has `acknowledge_url` (`AinfluencerSsoSource::acknowledgeAinfluencer`). Fire-and-forget; failures are ignored | `POST api/user-third-party/ainfluencer-affiliate/v1/sso/callbacks/user-registered` (core `VerifyAinfluencerAffiliateMiddleware`: JWT HS256 with the shared secret, or X-API-KEY). Sets `users.connect_ainfluencer_affiliate_account` and upserts `user_third_parties` |
| SSO social fetch: **GET** `{AINFLUENCER_GET_SOCIAL_MEDIA_URL}?user_id=` (JWT `event=get_social_media`) | First SSO registration of an **affiliate** only | Core route is **POST** `…/sso/callbacks/social-media-sync`. If the env points there, this is a method mismatch (405) and the affiliate silently creates no social accounts |
| Webhook `profile.amazon_account.deleted` `{amazon_account_id}` | `DeleteAmazonAccount` | `POST api/user-third-party/ainfluencer-affiliate/v1/webhook` (event routed by `AinfluencerAffiliateWebhookListener`, queue `ainfluencer-affiliate:webhooks`) |
| Webhook `profile.shopify_account.deleted` | `DeleteShopifyAccount` | same |
| Webhook `campaign.updated` `{campaign_id: source_campaign_id, data}` | Integration campaign create/assign/list and affiliate join; `ExpireCampaignJob`; product status or commission change on a campaign item (`DispatchCampaignUpdatedWebhookOnProductChangeListener`); **hourly** `SyncCampaignsWebhookDataCommand` for every ACTIVE or PENDING campaign that has a `source_campaign_id` | same → `webhookUpdateCampaign` |
| Webhook `profile.shopify_account.synced` (HMAC-signed `state` incl. Shopify access token) | `NotifyCoreShopifyStoreSyncedJob` after Shopify OAuth. Re-tried every 5 min by `ReconcileCoreShopifyStoreSyncCommand` while `core_synced_at IS NULL`. `core_synced_at` is set only on 2xx | same; core verifies the HMAC, then `HandleShopifyAccountCoreSyncedJob` |
| Webhook `campaign.paused` `{campaign_id, is_paused, paused_by, reason: seller_payment_method_missing}` | `PauseCampaignWithoutPaymentMethodJob` (daily 02:30) | **Not a key core accepts**: core's enum `AinfluencerAffiliateWebhookEventKey` lacks it, so core answers 400. Delivery also requires a `webhooks` row subscribed to it (unverified) |
| Browser redirect `{AINFLUENCER_SELLER_AMAZON_CALLBACK_URL}?user_type_id&user_id&amazon_account_id&status` | `AmazonLoginCallbackService::getRedirectUrl` (source=ainfluencer) | core `GET …/amazon-account/callbacks/sync` |
| Browser redirect `{AINFLUENCER_SELLER_SHOPIFY_CALLBACK_URL}?state&signature&…` | `ShopifyLoginCallbackService::processAinfluencerLogic` | core `GET …/shopify-account/callbacks/sync` (HMAC verified with `INTEGRATION_AINFLUENCER_INTERNAL_SHARED_SECRET`) |
| Stripe Billing Portal return | `cards/sync` | core front page |

**Webhook delivery mechanics** (`app/Services/General/Webhook/SendWebhookService.php`, `Jobs/Webhook/SendWebhookJob`):
- Event keys must exist in `webhook_events`. Targets are ACTIVE `webhooks` rows (admin CRUD at `admin/v1/webhooks`) whose `apiKey` is active. The target URL (core) is **configured in the DB**, not in code.
- Each send gets header `X-API-KEY = webhook.apiKey.key`, `X-Webhook-Event`, and a payload of `{event_key, timestamp, …}`.
- Retries default to 3 (`webhook.retry_count`), with exponential backoff `sleep(2^n)` inside the job. Every attempt is logged to `webhook_logs` (statuses pending, retrying, success, failed, timeout).
- `dispatch()` swallows failures (log only). Only the Shopify-synced path uses `dispatchOrFail` and retries.

### 1.4 SSO: how a core brand or influencer lands in the seller or affiliate app
1. Core front calls `GET core:/api/user-third-party/ainfluencer-affiliate/v1/sso/authorize-url` (auth:api2). This sends `type=brand` or `influencer` depending on `is_influencer`. The mobile app instead calls `GET /api/{locale}/v3/mobile/ainfluencer-affiliate/sso/authorize-url` (always `type=affiliate`).
2. Core builds an HS256 JWT signed with `user_third_party.ainfluencer_affiliate.shared_secret`. Claims: `type, user_id (core), email, name, issuer=ainfluencer, audience, event=sign_in_by_ainfluencer, acknowledge_url=route(callbacks.user-registered), jti, iat, exp`. Core returns `{base}/web/v1/sso/login?token=…`.
3. Affiliate `Web\Sso\SsoLoginController` → `app/Services/General/Sso/Sso.php`:
   - Decodes the JWT with `SSO_JWT_SECRET` (must equal core's shared secret) and picks the source by `issuer` (`AinfluencerSsoSource`).
   - `type=brand` maps to **seller**. Anything else (`influencer` or `affiliate`) maps to **affiliate**.
   - Looks up an existing UserType by email + type. If none exists, it registers a user (`register_source=ainfluencer`, random password, `reference_id`=core user id). It then fills profile fields (name, gender, avatar, bio, country) and passes the PROFILE step when country and gender are present. For affiliates it may also import social accounts (see §1.3).
   - Marks the email verified and runs the post-verify actions. If `acknowledge_url` is present it POSTs the acknowledgement.
   - Issues a Sanctum token (`role:<type_id>`) and redirects to `{seller|affiliate front}/user/email-verified/{token}` (an optional `redirect` query is forwarded).
   - Errors render `resources/views/sso/login_error`: expired or invalid token, or a generic error.
4. **Account linking**:
   - Core `user_third_parties` (provider = affiliate provider name, `type` = seller/affiliate/brand/influencer as echoed back) stores `third_party_user_id` (aff users.id) and `third_party_unique_identity` (aff UserType id).
   - Core `user_third_party_accounts` stores Amazon and Shopify account ids returned by `register`.
   - Affiliate side stores `users.reference_id` = core user id (`CoreUserRecorder`, write-once).
   - Server-to-server calls identify the user by `reference_id` = the aff UserType hash id.
   - Linking is by email: an existing affiliate account with the same personal email is reused (a fake/imported `is_fake` user is not reused in `register`).

---

## 2. Route groups (895 routes)

| Area | Count | Auth | Purpose |
|---|---|---|---|
| `seller/v1/*` | 280 | `auth_or_api_key` (Sanctum or user API key) + `seller_access`; team members are limited by `impersonation_permission:seller.*` | Brand dashboard |
| `affiliate/v1/*` + `v2` | 271 + 10 | `auth_or_api_key` + `affiliate_access` | Creator dashboard |
| `admin/v1/*` | 187 | `auth:admin` (+ `can:` permissions); `integrations` use `api_key` | Back-office |
| `buyer/v1/*` + `buyers/v1/extension` | 64 + 2 | Sanctum + `buyer_access`; the extension routes are public closures | Shopper account, plus browser-extension config |
| `client/*` | 26 | mostly public | Public landing (`buy.ainfluencer.com`), product-link resolution, inbound callbacks |
| `web/*` | 23 | public (signed for email verify) | OAuth callbacks, SSO, two debug closures |
| `stripe/*` | 2 | Cashier | Stripe webhook + payment page |
| horizon / docs / _ignition / sanctum / broadcasting | 22 / 3 / 3 / 1 / 1 | — | Infrastructure |

### 2.1 Seller sub-groups (count — purpose)
- `integrations` 30 — the server-to-server API for core (§1.2).
- `products` 27 — products and brands: list/show, commission (single and bulk), status, auto-partnership-approval toggle, brand terms file, brand landing page and socials, assets, Amazon brand-link, manual product sync request (`daily_product_sync_request` throttle).
- `profile` 28 — profile, security/sessions, platform accounts (sync Amazon, Amazon profile, Shopify; delete platform account), payment accounts.
- `reports` 20 — Performance, affiliate, brand, product, BRB, and sale-detail reports (list, chart, export).
- `partnership` 18 — send brand/product invites, act on applied/invited requests, contracts (list, show, revoke), affiliate drop-downs.
- `teams` 18 — team members, invitations, roles and permissions, impersonation token (docs/TeamImpersonationToken.md).
- `user-type-lists` 15 — affiliate and unlisted-affiliate lists (manual/smart), membership, revealed stats.
- `chats` 15 — contacts, messages, tags, saved messages, bulk/list messaging.
- `auth` 15 — register, login, social login, verification, become-seller, skip step.
- `dashboard` 10 — KPI widgets. `payments` 9 — commission statements (list, show, items, export, **retry**), invoices, charges, subscription transactions, billing portal.
- `promotions` 9 — promotion/coupon CRUD, status, accessibility, assign affiliates. `plans` 7 — subscribe, update, cancel, resume, status.
- `email-campaigns` 5 + `smtp-connections` 6 — seller email outreach. `reveal-information` 3 — pay (plan quota or wallet) to reveal affiliate contact info (docs/RevealInformation.md). `wallet` 4 — Stripe Checkout deposit into wallet (docs/Wallet.md).
- `discover` 3, `unlisted-affiliates` 3 — affiliate discovery and imported (unregistered) affiliates. Smaller groups: api-keys 5, message-templates 5, message-variables 2, notifications 5, tickets 5, settings 4, stores 2, questions 2, pins 1, guidelines 1, categories 1, system 1, user-fcm-tokens 1.

### 2.2 Affiliate sub-groups
- `products` 33 — browse partnered products and brands, create/delete/set-default product links and link sources, landing-page promote, brand sale-detail report.
- `partnerships` 23 — apply to brands/products, act on invites, contracts (list, revoke). `profile` 22.
- `reports` 16 — performance, brand, product, source, landing-page, store reports.
- `landing-pages` 16 — creator storefront pages, collections, assignees, templates.
- `teams` 18, `user-type-lists` 14 (brand/product lists), `chats` 14, `associates` 14 (Amazon Associates program: promote tags, sale reports, associate products scraped via Keepa, associate social media), `auth` 15, `dashboard` 10.
- `payments` 9 — Stripe Connect onboarding link, upcoming commission, commission transactions (list, items, export), invoices, charges, subscriptions. `plans` 7.
- `social-medias` v1 9 + v2 10 — social accounts (v2 = OAuth for YouTube, TikTok, Instagram, Pinterest, Facebook; website verification token).
- `integrations` 8 (core API). Others: api-keys 5, message-templates 5, notifications 5, tickets 5, settings 6, stores 2, amazon-marketplaces 2, managements 2, questions 2, temp_partnership 1 (confirm brand request made pre-registration), promotions 1, pins 1, guidelines 1, categories 1, system 1, user-fcm-tokens 1.

### 2.3 Admin sub-groups
- user-type-lists 22, associates 20 (CSV imports of Associates product/sale reports, fee earning/tracking, promote tags, shorten links), affiliates 13 (list, detail, status, bulk email/SMS/notification, product links, social requests), integrations 8, bar-statistics 9, sellers 9, promotions 9, webhooks 10 (webhook + event CRUD, which **wires the outbound core webhooks**), brands 7, buyers 7, admins 6, roles 5, guideline-pages 6, requests 6 (approve seller promotions, brand Amazon links, email campaigns), smtp-connections 6, api-keys 5, payments 5 (commission and subscription transactions, **refund**), products 4, emails 4 / notifications 4 (bulk messages), tickets 4, auth 3 (incl. impersonate login), unlisted-affiliates 3 (import), register-steps 2, social-medias 2, plans 1 (custom plan), and 1 each for categories, email-campaigns, features, permissions, system, upload-media, users.

### 2.4 Buyer / client / web
- `buyer/v1` — account (auth 14, profile 17, chats 13, tickets, notifications, settings, questions); `products` 1.
- `client/buyer/v1` 17 — public shopper auth plus **product link resolution** (`products/links/{productLinkId}` → `ShowProductLinkController`, applying redirect rules: product removed/disabled, link revoked/inactive) and the public affiliate profile.
- `client/affiliate/v1` 5 — public brand and store pages (`check_brand_landing_page`), pre-registration brand request (`temp_partnership`).
- `client/v1/callback` 4 — inbound third-party webhooks (below).
- `web/v1` — social login callbacks (Google, Facebook, Amazon, **Shopify**) for seller/affiliate/buyer/client, signed email-verify links, `sso/login`. `web/v2/affiliate/social-medias/login/*/callback` — creator OAuth.

### 2.5 Inbound third-party webhooks

| Endpoint | Source | Verification | Effect |
|---|---|---|---|
| POST `stripe/webhook` | Stripe platform account (Laravel Cashier) | Cashier signature check | Cashier's built-in subscription handlers, then `WebhookReceived` → `Listeners/Payment/StripeWebhook` |
| POST `client/v1/callback/stripe` | Stripe **Connect** events | `Webhook::constructEvent` with `STRIPE_CONNECTED_WEBHOOK_SECRET` | same listener |
| POST `client/v1/callback/shopify/webhooks` | Shopify app webhooks | HMAC of the raw body with `services.shopify.client_secret` | `ShopifyWebhookHandlerJob` → handlers in `Services/General/Shopify/Webhook/Handlers`: app/uninstalled, products create/update/delete, discounts create/update/delete, shop/update, orders/create |
| POST `client/v1/callback/shopify/track/event` | Ainfluencer **web pixel** on the merchant storefront | see code | `ShopifyPixelEventHandlerJob`. Page-view, product-view, add-to-cart, checkout and purchase events increment the daily `shopify_reports` row per store and link |
| GET `client/v1/callback/email/unsubscribe/{token}` | email links | token | unsubscribe from a notification group |
| GET `web/v1/seller/login/amazon/callback` | Amazon Ads OAuth | state | creates/updates the AmazonAccount and profiles (status `syncing`) and queues profile sync. With source=ainfluencer, redirects to core |
| GET `web/v1/seller/login/shopify/callback` | Shopify OAuth | state | upserts ShopifyAccount and Store; side effects (webhooks, pixel install, product sync); notifies core |

`StripeWebhook` listener actions:
- `customer.created` and `customer.subscription.created` → resolve or link the Stripe customer to a UserType.
- `customer.updated` → default payment method.
- `payment_method.attached/detached` → `stripe_customer_payment_methods`.
- `account.updated` (Connect) → `stripe_accounts` (CONNECTED only when charges_enabled && payouts_enabled), and the `StripeAccountStatusUpdatedNotification` email.
- `invoice.*` → `stripe_invoices`. `charge.*` (incl. `charge.dispute.*`) → `stripe_charges` upsert only. `charge.refunded` → negative sale to FirstPromoter.
- `payment_intent.*` → `stripe_payment_intents`; `payment_intent.succeeded` with metadata source=wallet_checkout_create → **confirms the wallet deposit**.
- In production, events with `livemode=false` are ignored.

Shopify webhook handler effects:
- `app/uninstalled` → uninstall pixel; store INACTIVE + `shop_store_status=closed`; deactivate store products; brand-status event.
- Product delete → remove the products and fire `SellerProductStatusUpdated`, which notifies partners and sends campaign webhooks.
- Discount handlers keep `shopify_discounts` ↔ `seller_promotions` in sync. Delete deactivates the promotion. An update for an unknown discount creates an ACTIVE, ACCEPTED, public promotion.
- `shop/update` → shop plan status, email, domain.
- **`orders/create` is log-only (no-op).**

---

## 3. Scheduler (`app/Console/Kernel.php`; everything runs only when `APP_ENV=production`)

| Command | Schedule | Business effect |
|---|---|---|
| `promotion:update-status` (`UpdatePromotionStatusCommand`) | hourly | Every `seller_promotions` row with `end_date < now` → `expired`. Applies to any status, including rejected or deleted |
| `ExpireCampaignsCommand` → `ExpireCampaignJob` | hourly, no overlap | ACTIVE campaign past `end_date` → FINISHED, then `campaign.updated` webhook |
| `SyncCampaignsWebhookDataCommand` | hourly | Re-sends `campaign.updated` for every ACTIVE or PENDING campaign that has a `source_campaign_id` (keeps core stats fresh) |
| `seller-account:update-all` (`UpdateAllSellerAccounts`) | daily | For ACTIVE Amazon profiles with `synced_at` older than 48 h: `SyncAmazonProfileJob` refreshes advertisers and the attribution tag, sets profile ACTIVE or DISCONNECTED, and re-syncs seller products |
| `CreatePaymentTransaction` | daily 03:00 | **Commission billing run** (§5): up to 10 sellers per run (`SELLER_COMMISSION_CALCULATION_SYNC_COUNT`), each at most once per 24 h; charges sellers for report rows older than 35 days |
| `email:new-partnership-requests` | daily 08:00 | Emails each receiver about pending partnership requests (if `email.types.send_new_partnership_requests`) |
| `email:new-partnership-requests-to-fake-users` | daily 09:00 | For imported or fake affiliates: POSTs to an external email webhook (`NEW_PARTNERSHIP_REQUEST_EMAIL_TO_FAKE_USERS_WEBHOOK_URL`) listing brands that invited them |
| `seller-account:update-report` (`UpdateSellerAmazonReport`) | daily 06:00 | For ACTIVE profiles whose report is older than 72 h: `SyncSellerProfileAmazonReportJob` pulls the Amazon Attribution ad-group report for the **last 60 days** and upserts `amazon_reports` with commission |
| `aggregate:notification` | hourly | **No-op** (body commented out). Aggregated activity notifications are never produced |
| `notification:process-queue` | daily 06:00 and 18:00 | Sends queued email notifications (`user_type_notification_queues` PENDING → PROCESSED). This makes most "email" notifications twice-daily digests |
| `payment:send-stripe-connect-nudges` | daily 10:00 | Meant to remind affiliates without Stripe Connect and sellers without a card. **Likely no-op**: reads `payment.stripe_connect_nudges.*`, which is not in `config/payment.php` → `''`, so the loop guard `0 <= ''` is false under PHP 8 |
| `chat:process-unread-messages` | Mondays 00:00 | One `UnreadChatMessagesNotification` per recipient with unread messages |
| `RevokePartnershipContractsForEndedCampaignsCommand` | daily 02:00 | All ACTIVE contracts of FINISHED campaigns → REVOKED (→ product links revoked, private promotion access removed) |
| `campaign:pause-without-payment-method` | daily 02:30 | ACTIVE, not-paused campaigns whose seller has no Stripe payment method → `is_paused=true, paused_by=system`, then `campaign.paused` webhook. Nothing in the affiliate code **ever un-pauses** a campaign, and `is_paused` has no other effect inside the affiliate backend |
| `statistics:send-seller weekly|monthly` | Mon 00:00 / last day of month 00:00 | Statistics email to every non-fake seller with a primary email |
| `statistics:send-affiliate weekly|monthly` | Mon 01:00 / last day of month 01:00 | Same for affiliates |
| `keepa:sync-associate-products` | weekly | Refresh associate products (Keepa scraper jobs per marketplace chunk) |
| `keepa:fetch-associate-products` | monthly | Discover new associate products via Keepa Product Finder filters |
| `shopify:reconcile-core-sync` | every 5 min | Re-push `profile.shopify_account.synced` for ACTIVE stores with `core_synced_at` NULL older than 5 min |
| `shopify:check-store-connections` | hourly | Per store: Shopify shop-info call. 401/403 → app uninstalled (store INACTIVE, products deactivated). 404 → INACTIVE (closed). Otherwise refresh `shop_store_status`, currency, timezone, plan |
| commented out | — | `ImportAffiliatesCommand`, `admin-bulk-message:process`, `email-campaign:process-notification`. **So admin bulk emails/SMS/notifications and seller email campaigns are stored but never sent by the scheduler** (they stay PENDING unless run manually) |

---

## 4. State machines

### 4.1 Partnership (seller ↔ affiliate)
**PartnershipRequest** (`partnership_requests.status`, `PartnershipRequestStatusEnum`): `pending → approved | rejected | canceled`.
- Created by:
  - affiliate apply (`affiliate/v1/partnerships/{brand|product}/request`) — `AffiliateCreatePartnershipRequest`;
  - seller invite (`seller/v1/partnership/{brand|product}/request`) — `SellerCreatePartnershipRequest`;
  - pre-registration brand request (`client/affiliate/v1/temp_partnership/brand/request`, confirmed after signup by `affiliate/v1/temp_partnership/brand/confirm-request` / `ConfirmPartnershipRequestJob`).
- Receiver of the request ("invited" side) may set `approved` or `rejected`. The sender ("applied" side) may only set `canceled` (FormRequests `ChangeStatus{Applied,Invited}{Brand,Product}Request`, on both sides).
- **Auto-approve**:
  - `AutoPartnershipApproval` listener runs on affiliate-created requests. A request is approved immediately when the item is not custom-commission and the brand, Amazon product's brand, or Shopify store has `auto_partnership_approval=true`.
  - The flag is toggled by the seller (`seller/v1/products/brands/change-auto-partnership-approval`). It is forced on or off for all the seller's brands when the subscription becomes active or inactive (`UpdatedSubscriptionObserver`).
  - Core integrations (bulk-generate-link, campaign products assign, affiliate assign) also force APPROVED.
- On approved, the `{Seller|Affiliate}ChangeStatusOfPartnershipRequest` listeners run:
  - `CreatePartnershipContract` creates the contract, items and per-product commission snapshot, and reveals seller contact info to the affiliate for free;
  - `CreateDefaultProductLink` creates the affiliate's product links;
  - `CreateChangePartnershipNotification` notifies.

**Aggregated partnership status** (UI, `PartnershipStatusEnum`): `pending, partnered, not_partnered, revoked, rejected, expired`. This is derived, not stored.

**PartnershipContract** (`PartnershipContractStatusEnum`): `active → revoked` (`inactive` exists in the enum but no writer was found).
- Revoked by:
  - seller or affiliate `…/contracts/{id}/revoke` (`Services/General/Partnership/Contracts::revoke`, sets `revoked_at` and `revoked_by`);
  - the ended-campaign job (daily 02:00);
  - plan downgrade cascade (`Plan/Downgrade/{Seller,Affiliate}/Cascade/Nodes/PartnershipContractNode`).
- Revocation fires `PartnershipContractRevokedEvent`, which revokes the affiliate's links for those products and removes private promotion access.
- New products of a brand are auto-added to ACTIVE brand contracts (`AddNewProductToBrandContract`).
- **Commission changes on a product overwrite the commission on *all* non-custom contract items for that product** (`UpdateBrandContractProductCommissions`; there is no status filter, so revoked contracts are updated too).

### 4.2 Campaign / program (core "affiliate campaigns")
`campaigns.status` (`CampaignStatusEnum`): `draft, pending, active, finished, archived, cancelled`, plus the `is_paused` / `paused_by` flags.
- Created ACTIVE by core (`campaigns/create`). The DTO default is `pending`.
- `active → finished`: hourly expiry when `end_date` has passed.
- `finished → active`: automatically when `end_date` is extended into the future via update (`CampaignManager::update`).
- Any status (`archived`, `cancelled`, …) can be set by core seller or admin `campaigns/update/{id}`. Delete is a hard delete of the campaign, items and assignments.
- Pause: system-only, daily, for a seller with no card. No un-pause.
- Effects:
  - FINISHED triggers revocation of contracts the next day at 02:00.
  - Every change triggers `campaign.updated` to core.
- Per-affiliate status inside a campaign (`CampaignAffiliateStatusEnum`, computed): `active, inactive, revoked, pending, approved, rejected, canceled, assigned`.

### 4.3 Product / brand
- `seller_products.status` (`SellerProductStatusEnum`): `active ↔ inactive`, with `inactive_at`.
  - Moved by: seller (`seller/v1/products/.../status`, plan gates brand/asin → 402), admin (`admin/v1/products/.../status`), core bulk-change-status, Shopify product delete or store going inactive (`DeactivateShopifyStoreProductsJob`), and the plan-downgrade `SellerProductNode`.
  - Fires `SellerProductStatusUpdated`: activity log, partner notification, campaign webhook.
  - Amazon report rows dated after `inactive_at` are skipped (no commission).
- Amazon availability (`SellerProductAmazonStatusEnum`): `in_stock, out_of_stock, …, removed`. Comes from the product sync.
- Brand (`SellerBrandStatusEnum`): `active/inactive`. Seller or admin, or the downgrade `SellerBrandNode`. Brand Amazon-link approval (`SellerBrandAmazonLinkApprovalStatusEnum`): `waiting_for_approval → accepted|rejected` (admin `admin/v1/requests`).
- Commission: per product (`commission`, `commission_type` percent|amount; default 15 % `SELLER_DEFAULT_COMMISSION`). Edited by seller (single or bulk), admin, or core bulk-update. Fires `ProductCommissionsUpdated`: contract items updated, activity log, notification, campaign webhook.

### 4.4 Affiliate product link
`affiliate_product_links.status`: `active` (created on approval or manually) `→ revoked` (contract revoked, downgrade) / `inactive` / `deleted`.
- Public resolution (`client/buyer/v1/products/links/{id}`) treats revoked or inactive links as "removed".
- In that case the affiliate's setting `REDIRECT_IF_PRODUCT_REMOVED_OR_DISABLED` decides where the shopper goes.

### 4.5 Integration / store connection
- **AmazonAccount** (`AmazonAccountStatusEnum`): `connected → disconnected` (a 4xx from Amazon during report fetch in `GetAmazonReport`, or downgrade) `→ deleted`.
- **AmazonAccountProfile** (`AmazonAccountProfileStatusEnum`): `syncing` (on OAuth or manual sync) `→ active` (advertisers and template tag fetched) `| disconnected` (API error, with `error_message`). `inactive` comes from downgrade; `deleted`.
  - Report sync timers: product refresh every 48 h, reports every 72 h over a 60-day window.
  - `AmazonAccountProfileObserver` sends a "syncing" notification.
- **ShopifyAccountStore**:
  - `status` 1=active, 2=inactive, 3=suspended. Inactive is set by app uninstall, missing token, a 401/403/404 on the hourly check, or delete.
  - `shop_store_status`: open, paused, frozen, dormant, closed, unknown (from the Shopify plan).
  - `core_synced_at`: NULL until core acknowledges `profile.shopify_account.synced`.
  - Pixel refresh statuses: updated, installed, skipped, failed.
- Account deletion is blocked by any existing partnership contract (Amazon and Shopify).

### 4.6 Report → commission → payment
- `amazon_reports`: one row per date, link and profile, with `commission` computed at sync. `shopify_reports`: daily row per store and link, from pixel events. Rows become "billable" when `date <= now-35d`, they have a `partnership_contract_id`, and they are not yet in `commission_transaction_items`.
- **CommissionTransaction** (`CommissionTransactionStatusEnum`): `pending → succeeded | failed | server_error`. `refunded` and `partially_refunded` can be set by an admin refund; `canceled` has no writer found.
  - Attempts are tracked in `commission_transaction_logs` (one per PaymentIntent try).
  - Mirrored to `payment_transactions` (has a `tracking_code`; `PaymentTransactionStatusEnum` uses the same values).
- Affiliate-facing **payout status** (`PayoutStatusEnum`, a mapping only): pending → `processing`, succeeded → `paid`, failed or server_error → `failed`, canceled/refunded/partially_refunded map 1:1.
- Timers: 3 attempts (`PAYMENT_MAX_ATTEMPTS`) spaced 60 s (`PAYMENT_ATTEMPTS_DELAY`), plus manual seller retry (only from FAILED).

### 4.7 Billing: cards, subscriptions, plan limits, wallet
- Seller card = Stripe customer payment methods (`stripe_customer_payment_methods`, default flag), added via the Billing Portal and synced by webhook.
  - No card → 499 on campaign create; campaigns paused daily.
  - At billing time the first card is promoted to default if none is set.
- Affiliate payout account = **Stripe Connect Express** (`stripe_accounts`): `not_connected → connected` when charges and payouts are enabled (`account.updated`). Onboarding at `affiliate/v1/payments/onboarding-request`.
- Subscriptions (Cashier `subscriptions.stripe_status`, `PlanSubscriptionStatusEnum`): active, trial, past_due, unpaid, incomplete, incomplete_expired, canceled/cancelled, cancelled_by_admin, expired, reserve, pending.
  - Plans: seller free 3 / starter 12 / pro 4 / ultimate 5 / custom 10; affiliate free 6 / pro 7 / ultimate 8 / custom 11; buyer 9. Admin can create a custom plan (`admin/v1/plans/{userTypeId}/custom` → `ProcessCustomPlanJob` + `CustomPlanNotification`).
  - **Downgrade cascade** on `stripe_status → canceled` or a price change (`UpdatedSubscriptionObserver` → `{Seller,Affiliate}PlanDowngradeService`) disables over-limit brands, products, Amazon accounts/profiles, promotions and contracts, and revokes links, with notifications.
- Plan limits → **HTTP 402** via `seller_access_feature` and plan-access checkers (features: brand, asin, commission, private_commission, promotion, active_contract, campaigns, information_reveal, email_campaign, team_member_invitation, application_fee, …). Toggle with `IS_CHECK_PLAN_FEATURES`.
- **Wallet** (sellers): Stripe Checkout deposit → confirmed on `payment_intent.succeeded`. Spent on reveal-information (docs/Wallet.md, docs/RevealInformation.md).

### 4.8 Promotions / coupons
- `seller_promotions.status`: `inactive, active, expired, waiting_for_approval, rejected, accepted, deleted`. `approval_status`: `waiting_for_approval → accepted|rejected` (admin `admin/v1/requests/...`), `deleted`.
- Amazon promotions are auto-ACCEPTED. Shopify promotions created in the UI start WAITING_FOR_APPROVAL. Core coupons are created ACTIVE + ACCEPTED.
- Hourly job sets `expired` after `end_date`. Shopify discount sync: `shopify_discounts` status active(0) / expired(1) / deleted(2).
- Affiliate access per promotion: `SellerPromotionAffiliateStatusEnum` active/inactive. Private promotions are removed on contract revoke.

### 4.9 Other entities
| Entity | Values | Moved by |
|---|---|---|
| UserType (account) status | 1 pending, 2 requested, 3 accepted, 4 rejected, 5 suspended | admin `UpdateStatus*` / bulk. **Suspended sellers are excluded from the commission billing run** (`notSuspended()`) |
| Register status | email verified / not verified / completed / incomplete | register steps (skip-step routes, SSO passes steps) |
| Team member | 0 pending, 1 accepted, 2 rejected | owner invites (`SendTeamInvitation`, emails with or without registration), invitee `ChangeTeamInvitationStatus`, cancel/delete. Team owners can mint impersonation tokens; `impersonation_permission:*` limits member actions |
| User-type lists | manual vs smart (filter-based), global (admin) vs personal, audience seller/affiliate; types affiliates, brands, products, unlisted affiliates | CRUD/duplicate/assign; default lists via `AssignDefaultUserTypeListCommand` (docs/UserTypeList.md) |
| Chats | no status; read/unread per message (`SeenMessage` event), contact tags, saved messages, bulk/list sends | users. Weekly unread digest |
| Landing pages | types/templates/collections/assignees, default page | affiliate CRUD (plan feature `custom_landing_page`); seller brand landing page (`brand_landing_page`) |
| Associates social media | 0 pending, 1 accepted, 2 rejected | affiliate submit/update → pending; admin change status |
| Social accounts | system_status verified/pending/waiting_for_approval/rejected; legacy `UserSocialMediaStatusEnum` 1–5 | OAuth (verified), SSO import (verified), admin `UpdateSocialMediaStatus` → notification |
| Tickets | new_ticket, user_response, admin_response, in_progress, closed | user create/reply/close; admin reply/close → `TicketRepliedNotification` |
| Email campaigns | status pending/running/completed/cancelled; approval waiting/accepted/rejected/cancelled | seller create → admin approve; sending job not scheduled (see §3) |
| Admin bulk message | pending/running/completed/cancelled | admin send/cancel/resend; processing command not scheduled |
| Notification queue | pending/processed/cancelled/failed | `NotificationQueue::create` → 06:00/18:00 processor |
| Webhook log | pending/retrying/success/failed/timeout | `SendWebhookService` |

---

## 5. Money flow

1. **Attribution**
   - Amazon: affiliate links are Amazon Attribution tags. `campaignId` encodes the seller's AmazonAccountProfile and `adGroupId` encodes the `affiliate_product_link` id (`AmazonLinkManipulator`). The daily report sync (60-day window) maps each row to the link, the product, and the affiliate's contract item for that product and profile (`SaveAmazonReport::saveAdGroupReport`). It stores sales, units, clicks, BRB bonus, `commission`, and the seller's `application_fee` %.
   - Shopify: the web pixel posts purchase events, which increment `shopify_reports.total_revenue` / `units_sold` and attach the active contract id.
2. **Commission formula** (Amazon, `SaveAmazonReport::calculateCommission`): percent → `attributedSales14d × rate/100`; amount → `attributedSales14d − value`. The amount branch looks wrong: it subtracts instead of paying a fixed amount.
   - The commission rate comes from the current contract item. Because rows are re-synced over 60 days, **a later commission change re-prices not-yet-billed rows**.
   - **Shopify commission is never computed**: no production code writes `shopify_reports.total_commission` (default 0; only the fake-data seeder sets it). Shopify sales therefore generate $0 commission charges.
3. **Platform fee** ("wage" / Stripe `application_fee_amount`) = `commission × fee%`. fee% = the seller plan's `application_fee` feature value for paid plans, else `SELLER_DEFAULT_APPLICATION_FEE` = **5 %** (Shopify rows always use the default). Fees from referred affiliates (`ref_code` + promoter access) are reported to **FirstPromoter** as referral commission.
4. **Billing run** (daily 03:00, `CreatePaymentTransaction` → `Services/General/Calculation/Calculation::calculateAllSellerCost`):
   - Selects non-suspended sellers with Amazon profiles or Shopify stores, `report_synced_at` NULL or older than 24 h, oldest first, max 10 per day.
   - Bills report rows with `date <= today − 35 days` (`PAYMENT_TRANSACTION_PERIOD`) that have a contract and no transaction item.
   - Aggregates per affiliate + currency (marketplace or store currency).
   - Also overwrites the contract's `total_commission` / `total_buy_*` with **only this batch's** sums.
5. **Who pays / how**
   - `StripePaymentDriver::createCommissionTransaction` requires the seller to have a `stripe_id` and the affiliate to have a CONNECTED Stripe account. Otherwise the transaction is silently skipped; the rows stay unbilled and are retried on later runs.
   - It creates a `commission_transactions` row plus item rows (links the report rows) plus a `payment_transactions` row (PENDING, tracking code).
   - `PayCommissionTransaction` job creates a confirmed **Stripe PaymentIntent on the seller's saved card / US bank / Link**: `amount = commission`, `transfer_data.destination` = affiliate's Connect account, `on_behalf_of` = affiliate, `application_fee_amount = fee`. The **seller's card is charged the full commission**, the **affiliate receives commission − platform fee**, and the platform keeps the fee.
   - There is no escrow or wallet balance in this flow; funds move as a destination charge. Payout to the affiliate's bank follows Stripe Connect's own payout schedule. The plan feature `affiliate_auto_payouts_escrow` exists but no escrow logic was found.
   - On failure, the job tries the seller's other payment methods, then marks FAILED and re-dispatches after 60 s up to 3 attempts. At attempt 2 it sends `LastAttemptPaymentTransactionNotification`; after max it sends `FailedPaymentTransactionNotification` (web only). There is a `@TODO` to suspend the seller.
   - Non-Stripe exceptions → SERVER_ERROR, no retry. Seller can retry FAILED ones (`seller/v1/payments/commission-transactions/{id}/retry`).
   - Success → web notifications to both sides, plus CRM event (`AFF_PAYMENT_STATUS_UPDATED`) and FirstPromoter referral.
6. **Thresholds**: amounts that round to 0 are skipped; there is no minimum payout threshold in code.
7. **Refunds / chargebacks**
   - Admin `POST admin/v1/payments/transactions/refund` (tracking code + amount) → Stripe refund of the successful PaymentIntent and status update. The status logic references a misspelled relation (`transactionble`), so a successful refund is likely always recorded as `partially_refunded`. Refund `amount` units (cents vs dollars) are not normalised.
   - Stripe `charge.refunded` → FirstPromoter negative sale only.
   - `charge.dispute.*` → stored in `stripe_charges`, **no business effect** (no reversal of the affiliate transfer, no seller flag).
8. **Affiliate visibility**:
   - "Upcoming payment" sums unbilled report commission (`ShowUpcomingPaymentService`).
   - Commission transactions list, items and export; core mobile payouts endpoint shows net (`amount − application_fee_amount`) by payout status.
9. **Associates (Amazon Associates program)**: separate. Admin imports Associates CSV reports (fee earning/tracking) for affiliates using Ainfluencer promote tags (`AFFILIATE_AMAZON_TAG`, default `ainfluencer-20`). No automated payout code was found for associates earnings.

---

## 6. Notifications / emails / digests

Channels: `mail` and `webnotification` (DB + broadcast via `Notifications/Channels/WebNotificationChannel`), SMS for `TextMessageNotification`, FCM tokens registered via `user-fcm-tokens`. Most domain notifications go through `NotificationQueue::create`: the web notification is sent **immediately**, and the email is **queued** and sent by the 06:00/18:00 processor, subject to the user's notification settings and an unsubscribe token.

| Notification | Trigger | Recipient | Channel |
|---|---|---|---|
| VerificationNotification / VerificationByLinkNotification / AccountVerificationNotification / ForgetPasswordNotification | auth flows (`SendVerificationCode`, `SendVerificationLink`, `UserType`, `ResetPassword`) | user | mail |
| MobileVerificationNotification | mobile verify, change password, change mobile | user | SMS-style |
| TextMessageNotification, AdminNotification | admin bulk SMS / notification (`Services/General/UserType/UserType`) | selected users | sms / mail |
| NewPartnershipRequestNotification (seller & affiliate) | daily 08:00 `NewPartnershipRequestEmail` | receiver of pending requests | mail |
| Seller/AffiliatePartnershipRequestStatusChangedNotification | request approved/rejected/canceled (`CreateChangePartnershipNotification`) | the other party | web + mail |
| PartnershipContractRevokedNotifier (seller & affiliate) | plan downgrade nodes (and the dead aggregate path) | both | mail |
| SellerProductCommissionUpdatedNotification | `ProductCommissionsUpdated` listener | partnered affiliates | web (+ mail) |
| SellerProductStatusChangedNotification / SellerBrandStatusChangedNotification | product or brand status events | partnered affiliates | web |
| SellerProductDeactivatedNotification / SellerBrandDeactivatedNotification | brand product disabled listener, downgrade | affiliates | mail |
| SellerBrandTermsFileUpdatedNotification | brand terms file change (`SellerBrandTermsFileUpdatedEmail`) | partnered affiliates | mail |
| UpdateStatusSellerBrandAmazonLinkNotification / UpdateStatusSellerPromotionNotification / UpdateStatusSocialMediaNotification | admin approval decisions | seller / affiliate | web + mail |
| SellerPlatformAccountSyncedNotification | Amazon/Shopify account synced | seller | mail |
| AmazonAccountProfileSyncingNotification | profile status → syncing (observer) | seller | web |
| Seller/AffiliateCommissionPaymentSuccessNotification | commission paid | seller / affiliate | web only |
| LastAttemptPaymentTransactionNotification / FailedPaymentTransactionNotification | commission charge attempt 2 / exhausted | seller | web only |
| StripeAccountStatusUpdatedNotification | Connect `account.updated` | affiliate | mail |
| Affiliate/SellerStripeConnectNudgeNotification | daily nudge command | affiliate / seller | mail (+web). **Likely never sent** (§3) |
| UnlinkPromotionOnProductLinkNotification | downgrade unlinks promotion from link | affiliate | mail |
| CustomPlanNotification | admin custom plan | user | mail |
| SendInvitationTeamNotification / …WithRegistrationNotification | team invite | invitee | mail |
| UnreadChatMessagesNotification | weekly Monday | recipient | queue |
| TicketRepliedNotification | admin reply | ticket owner | queue |
| AssociateProductScrapingCompletedNotification | Keepa scrape done | requester | — |
| Weekly/Monthly Seller & Affiliate statistics mails (`*StatisticsMailFactory`) | scheduler | all non-fake sellers / affiliates | mail |
| Fake-user partnership digest | daily 09:00 | imported affiliates | external webhook (not Laravel mail) |

**Never dispatched (no referencing code):** `AffiliateDailyActivityNotification`, `SellerDailyActivityNotification`, `ApprovedBrandPartnershipRequestNotification`, `RejectedBrandPartnershipRequestNotification`, `RejectedProductPartnershipRequestNotification`, `Seller\ApprovedPartnershipRequestNotification`, `Seller\RejectedPartnershipRequestNotification`, `VerifyMessageNotification`. Mailables with no user: `FailedPaymentTransactionMail`, `LastAttemptPaymentTransactionMail`, `Seller/AffiliateCommissionPaymentSuccessMail` (so payment events produce **no email**), `EmailCampaignMail`, `SendBulkEmailMail`. `AggregateNotification` activity digests are dead because `aggregate:notification` is a no-op. Email campaigns and admin bulk messages are not scheduled (§3).

---

## 7. Existing docs worth citing
- `README.md` — coding conventions; the four user categories; Action vs General services; route and naming conventions. No business flows.
- `docs/UserTypeList.md` — lists model: manual vs smart, global vs personal, audience targeting, endpoints per role.
- `docs/Wallet.md` — wallet tables, Stripe Checkout deposit, transaction types and actions.
- `docs/RevealInformation.md` — paid contact reveal: model types, payment methods free/plan/wallet, daily limits.
- `docs/TeamImpersonationToken.md` — team owner impersonation token endpoint and usage.
- `docs/EmailCampaignProcess.md` — email campaign pipeline (command → job → recipient handlers). Note that the command is not scheduled in v1.5.27.
- `docs/MessageTemplate.md` — message templates and `{{variables}}` for email, chat and partnership.
- core: `app/Modules/UserThirdParty/Docs/ainfluencer-affiliate-{sso,campaign,card,coupon,program}-api.md` — core-side view of the same contract (SSO flow incl. mobile `type=affiliate`).

---

## 8. Support quick answers (derived)
- "Seller wasn't charged / affiliate wasn't paid". Check, in order:
  1. Is the report row older than 35 days?
  2. Does the seller have `stripe_id` and a card?
  3. Is the affiliate's Stripe Connect CONNECTED?
  4. Is the seller suspended?
  5. Is the row tied to a contract?
  6. Was the seller among the 10 processed that day?
  7. Is it a Shopify sale? Shopify commission is always 0.
- "Can't delete Amazon/Shopify account": any partnership contract (even revoked) blocks deletion.
- "Campaign paused": the seller had no payment method at 02:30. The affiliate backend never un-pauses; core sees `campaign.paused` only if it accepts that event (it currently doesn't).
- "Links stopped working": contract revoked (manual, campaign finished, plan downgrade) or product inactive/removed. Redirect behaviour follows the affiliate's link setting.
- "Didn't get email X": most emails are batched at 06:00/18:00. Payment success/failure is web-only. Email campaigns and admin bulk emails are not scheduled.
