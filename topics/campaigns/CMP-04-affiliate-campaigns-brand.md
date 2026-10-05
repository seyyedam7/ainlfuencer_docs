---
id: CMP-04
title: Affiliate campaigns (brand)
domain: Campaigns (brand side)
audience: [brand, system]
apps: [web, email, background]
related: [CMP-01, CMP-03, MKT-03, AFX-01, AFX-02, AFX-03, AFS-03, AFS-10, INT-01, SUB-04, CHAT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, affiliate/ainflu-affiliate-backend@v1.5.27
---

# CMP-04 · Affiliate campaigns (brand)

> An affiliate campaign pays influencers a commission on each sale from their tracked links instead of a fixed fee. The brand sets it up inside the Ainfluencer web app, but the store connection, the products, the commissions, the links and the billing all live on the separate **Ainfluencer Affiliate platform**. Ainfluencer keeps a copy of the campaign so influencers can find it in the mobile app.

## Who and where

- **Brand**, web app, sidebar "Affiliate" → `/affiliate/dashboard` ("Affiliate Dashboard"). Tabs: "Programs", "Campaigns", "Affiliates", "Performance", "Payouts".
- **Setup screens** (`/affiliate/<step>`): `sync_account` (connect stores), `sync_payment` (card), `product_list` (products and commissions), `affiliate_campaign_setup` (pick the store), `affiliate_campaign_details` ("Campaign Information"), `affiliate_campaign_products` ("Select products and set affiliate commissions"), `affiliate-campaign-list` (list of affiliate campaigns).
- **Advanced dashboard:** the "Advanced Affiliate Dashboard" button opens the seller app (seller.ainfluencer.com) by single sign-on, see AFX-01.
- **Influencers** join from the mobile app (MKT-03).

## How it works

1. **Connect a store** ("Programs" tab → "Create Amazon Affiliate Program", "Create Shopify Affiliate Program" or "Connect a store"). A pop-up opens the Amazon or Shopify login ("Connecting your store..." / "Don't close this window..."). When it finishes, the store and its products appear as a "program". "Re-sync Products" refreshes the products. Brands on another e-commerce platform can send a request form instead. Details in AFX-03.
2. **Add a card** ("Payouts" tab, or `/affiliate/sync_payment`). The brand is sent to a Stripe billing page and comes back to `/affiliate/dashboard?tab=payouts`. This card is what the affiliate platform **charges the commissions to** (AFS-10). Without a card, affiliate campaigns can't be created.
3. **Set commissions** (`/affiliate/product_list`, "Edit Products"): a percentage per product (0–100%), set one by one or in bulk, plus switching products on or off. Lower commissions may need a higher plan; the affiliate platform answers with the upgrade pop-up.
4. **Create the campaign:**
   - start from the type picker → "Affiliate Campaign" → "Track sales through Ainfluencer". This works only when a store **and** a card are connected; otherwise the brand is sent to the dashboard;
   - pick the store or Amazon profile ("I am a..." → "Seller on Amazon" / "Seller on Shopify");
   - fill in "Campaign Information": "Campaign Name", "Start Date", optional "End Date", optional "Description", categories, countries ("All" allowed) and a campaign image. "Continue" creates the campaign on the affiliate platform (already active there). In Ainfluencer it stays a draft for now;
   - choose products and confirm their commissions (10% is used for any product without a commission), then save. **This is when the campaign goes live in Ainfluencer.** It's approved automatically with **no admin review** (unlike CMP-02).
5. **While it runs:**
   - influencers who join from the app get an automatically approved partnership and tracked links on the affiliate platform (MKT-03, AFX-02);
   - stats (affiliates, products, average commission, sales) are sent back to Ainfluencer whenever something changes and again every hour. They show on the cards and in "Performance";
   - "Affiliates" (and `/campaigns/affiliate-influencers/:id`) lists the influencers, with a chat drawer that uses the **Ainfluencer chat** (CHAT-01).
6. **Managing:** from "Campaigns" or `/affiliate/affiliate-campaign-list` the brand can edit (name, description, dates, image, categories, countries), pause, resume or delete. On these screens, pausing archives the campaign on the affiliate platform, and deleting removes it there too. Commission and product changes are done in "Edit Products".
7. **End:** on the end date the campaign finishes on the affiliate platform (hourly check). The next night at 02:00 UTC all its partnership contracts are revoked, so affiliate links stop earning. In Ainfluencer it shows "Expired" within a minute of the end date, and the brand gets "Campaign Expiry".

## Rules and limits

- **A card and a connected store are required.** The picker checks both. Creating a campaign without a card fails with "Please connect a payment method before creating an affiliate campaign."
- **Campaign slots:** affiliate campaigns use the same plan campaign limit as other campaigns (SUB-04). The limit is checked at "Continue" on "Campaign Information" and again when products are assigned.
- **Plan limits enforced by the affiliate platform:** the commission percentage and the number of connected affiliates (and number of brands/products) are limited by plan. The refusal shows as the upgrade pop-up. On the Free plan, the campaign product screen doesn't accept a commission below **25%**.
- **Dates:** start date today or later; end date optional, in the future, and at least one day after the start ("Start date must be at least 1 day after End date" is the on-screen wording). No end date = runs until paused or deleted.
- **Registration:** brands are **not** registered on the affiliate platform at sign-up. Their seller account is created the first time they use an affiliate feature (connecting a store, adding a card, opening the advanced dashboard). Accounts are matched by email (AFX-01).
- **Store deletion** is refused while any partnership contract exists on it, even a revoked one (AFX-03).
- **Money:** the brand's card is charged the commission about 35 days after a sale (AFS-10). Affiliates are paid through Stripe Connect (AFA-08). Nothing goes through the Ainfluencer wallet.
- **Coupons:** `/affiliate/coupons` shows a coupon screen but it isn't connected to anything. Coupons are managed in the seller app.

## Statuses

| What the brand sees (Ainfluencer) | Meaning | Affiliate platform |
|---|---|---|
| "Draft" | Campaign details saved, no products assigned yet. Not visible to influencers. | Already `active` |
| "Approved" (live) | Products assigned. Visible in the app. | `active` |
| "Paused" | Paused by the brand. On the affiliate campaign screens, also archived on the platform. | `archived` (if paused from those screens) |
| "Expired" | End date passed. | `finished`; contracts revoked at the next 02:00 UTC |

The affiliate platform can also mark a campaign **paused** (`is_paused`, by `system`) when the seller has no card. Ainfluencer isn't told and keeps showing it as live.

## Timers and automatic actions

| What | When (UTC) | Where | Effect |
|---|---|---|---|
| `ads:expiry-check` | every minute | Ainfluencer | Campaign with end date in the past → "Expired" + "Campaign Expiry" email |
| `ExpireCampaignsCommand` | hourly | Affiliate platform | Active campaign past end date → finished |
| `RevokePartnershipContractsForEndedCampaignsCommand` | daily 02:00 | Affiliate platform | All active contracts of finished campaigns revoked; links stop earning |
| `campaign:pause-without-payment-method` | daily 02:30 | Affiliate platform | Campaigns of sellers with no card are paused. Never un-paused automatically. Ainfluencer is not updated. |
| `SyncCampaignsWebhookDataCommand` | hourly | Affiliate platform → Ainfluencer | Re-sends campaign stats |
| `CreatePaymentTransaction` | daily 03:00 | Affiliate platform | Charges sellers' cards for sales older than 35 days (AFS-10) |

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Campaign expires ("Campaign Expiry") | Brand | Email, push, in-app |
| Brand pauses a campaign | Brand | Email + in-app ("Turn back ON your Campaign...") |
| Affiliate-platform emails (partnership requests, statistics, billing) | Seller | Affiliate platform emails (AFS topics) |

## Common questions

### "The Affiliate Campaign option keeps sending me to the dashboard."
**Answer for the user:** You need two things first: a connected Amazon or Shopify store (Programs tab) and a payment card (Payouts tab). As soon as both are in place, the Affiliate option starts the campaign setup.
**What to check:** the affiliate status for the brand (store connected? card on file?) on `/affiliate/dashboard`.
**Escalate when:** both show as connected and the redirect continues.

### "I get 'Please connect a payment method before creating an affiliate campaign'."
**Answer for the user:** Affiliate commissions are charged to a card, so a card is required before a campaign can be created. Add one in the Payouts tab and try again.
**What to check:** whether the card sync returned to `/affiliate/dashboard?tab=payouts` and the card appears there.
**Escalate when:** the card shows in the Payouts tab but creation still fails.

### "My affiliate campaign isn't visible to influencers."
**Answer for the user:** An affiliate campaign goes live once you've chosen its products and saved them. Open the campaign and add products.
**What to check:** Ainfluencer status. "Draft" means no products assigned. Check also whether it's paused, expired, or past its end date.
**Escalate when:** it has products, is live in Ainfluencer and still isn't visible in the app.

### "I can't set a commission below 25% / I get an upgrade pop-up when changing commission."
**Answer for the user:** Lower commission rates are part of the paid plans. Upgrade, or keep the commission at or above your plan's minimum.
**What to check:** the brand's plan (SUB-04, AFS-12).
**Escalate when:** the brand is on a plan that should allow the rate.

### "My campaign ended and affiliates say their links don't work anymore."
**Answer for the user:** When an affiliate campaign reaches its end date, the partnerships are closed the next night and the links stop earning. To continue, extend the end date in "Edit".
**What to check:** `ads_affiliate.end_date` and Ainfluencer status "Expired". On the affiliate platform: campaign finished, contracts revoked.
**Escalate when:** the end date hasn't passed and links stopped (see AFX-02).

### "The stats on my affiliate campaign look out of date."
**Answer for the user:** Numbers come from the affiliate platform and refresh at least once an hour. Sales from Amazon can take a few days to arrive.
**What to check:** wait an hour. Compare with the advanced dashboard (seller app).
**Escalate when:** stats differ from the seller app for more than a few hours.

### "Can I talk to the affiliates in my campaign?"
**Answer for the user:** Yes. Use the chat from the "Affiliates" list in Ainfluencer (or the Ainfluencer inbox). The chat inside the affiliate platform isn't a supported channel.
**What to check:** n/a (CHAT-01).
**Escalate when:** messages fail to send.

## Known issues

- **Pausing or deleting from the main `/campaigns` page doesn't update the affiliate platform** (only the Ainfluencer copy changes), while the same actions on the affiliate screens do. Affiliates may keep promoting a campaign the brand thinks is off. Workaround: pause or delete from `/affiliate/affiliate-campaign-list` or the dashboard "Campaigns" tab. Escalate to dev if earnings continue after deletion.
- **The nightly no-card pause on the affiliate platform is invisible in Ainfluencer.** If the brand removes its card, the campaign is paused there at 02:30 UTC and never resumes by itself, but Ainfluencer still shows it as live. Escalate to dev to un-pause after the card is added again.
- **UI promise about card requirements** *(to verify)*: the Payouts tab says the card needs "at least 1 year of validity remaining" and a "minimum $5,000 available limit", and that programs are "hidden until a payment card is connected". The code doesn't check validity or limit. Don't promise these to users.
- **`/affiliate/coupons` does nothing** (no data is loaded or saved). Point users to the seller app.
- **`/affiliate/programs`, `/affiliate/settings`, `/affiliate/review`** show a blank page.

## Related topics

- AFX-01: SSO to the seller app and account matching. AFX-02: how campaigns live on the affiliate platform. AFX-03: store connections.
- AFS-03: products and commissions in the seller app. AFS-10: commission billing and card. AFA-08: affiliate payouts.
- MKT-03: influencers joining affiliate campaigns.
- CMP-01: type picker. CMP-03: the Campaigns page. SUB-04: campaign slots.

## Technical reference

- **Core endpoints:**
  - `GET user-third-party/ainfluencer-affiliate/v1/status` (`AinfluencerAffiliateProgramController@status`: `card_status`, `amazon_account_status`, `shopify_account_status`).
  - Store sync: `…/amazon-account/sync`, `…/shopify-account/sync`, `…/{amazon-account/profiles|shopify-account/stores}/check/{id}`, `…/affiliate-programs/list`, `DELETE …/account/delete/{account_id}`.
  - Cards: `…/cards/list`, `…/cards/sync`. Products: `…/products/{amazon|shopify}/list`, `PUT …/products/bulk-update-commission`, `POST …/products/bulk-change-status`, `…/brands/{amazon|shopify}/drop-down`. SSO: `…/sso/authorize-url`.
  - Create: `POST /api/v3/campaigns/affiliate` (`AdCreationProcessAffiliateController` → `AdCreationProcess/Services/AdCreationProcessAffiliateService::store`: `hasCard` first (422 `AffiliatePaymentMethodRequiredException`), creates `ads` (`status=draft, approval=draft-pending, campaign_type=performance`) + `ads_affiliate`, then the driver `createCampaign`). Middleware `feature:campaigns`.
  - Assign: `POST …/campaigns/assign-items` (`AinfluencerAffiliateService::assignCampaignItems`: `CampaignSlotGate`, then `updateAdAffiliateProductStatsFromAssignedItems` sets `status=active, approval=approved`).
  - Manage: `GET/PATCH/DELETE /api/v2/campaigns-affiliate/{id}` (`v2/Frontend/AdAffiliateController`, PATCH has `campaign.activation.quota`; status map active→active, paused→archived, expired→finished, draft→draft, sent to the platform via `updateCampaignAmazonForSeller`; DELETE calls `deleteCampaignAmazonForSeller` then archives the ad), `GET …/{id}/influencers`, `GET …/{id}/products`.
  - Webhooks from the platform: `POST …/v1/webhook` (`campaign.updated` → `webhookUpdateCampaign` updates `ads_affiliate` counters only; `profile.*_account.deleted`; `profile.shopify_account.synced`). `campaign.paused` is not an accepted event key.
- **Affiliate backend:** integration routes `seller/v1/integrations/ainfluencer/*` (`campaigns/create` behind `seller_stripe_connected` → HTTP 499 without a card; `products/bulk-update-commission` and `bulk-change-status` behind `seller_access_feature` → 402). Scheduler in `app/Console/Kernel.php` (production only): `ExpireCampaignsCommand` hourly, `SyncCampaignsWebhookDataCommand` hourly, `RevokePartnershipContractsForEndedCampaignsCommand` 02:00, `PauseCampaignsWithoutPaymentMethodCommand` 02:30.
- **Data (core):** `ads` (`campaign_type=performance`, `status`, `approval`, `paused_by`, `third_party` amazon/shopify, `extra.shopify_product_ids`), `ads_affiliate` (`account_id`, `profile_id`, `item_type` product_amazon/product_shopify, `start_date`, `end_date`, `products_count`, `average_commission`, `affiliates_count`, `sale_amount`, `sale_currency`), `user_third_parties` (provider = affiliate provider, `type` seller, `third_party_unique_identity` = affiliate UserType id), `user_third_party_accounts`. **Affiliate DB:** `campaigns` (`status`, `is_paused`, `paused_by`, `source_campaign_id` = core ad id), `partnership_contracts`.
- **Key files:** `core/frontend/web/src/components/pages/affiliate/AffiliateContent.tsx` and `parts/*`, `core/frontend/web/src/components/pages/affiliateDashboard2/*`, `core/backend/api/app/Modules/UserThirdParty/Services/AinfluencerAffiliateService.php`, `core/backend/api/app/Services/Implementation/Brand/AdAffiliateService.php`.
