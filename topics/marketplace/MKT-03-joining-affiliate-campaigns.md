---
id: MKT-03
title: Joining affiliate campaigns (influencer)
domain: Campaign marketplace (influencer side)
audience: [influencer, system]
apps: [mobile]
related: [CMP-04, AFX-01, AFX-02, AFA-04, AFA-05, AFA-08, AFS-10, INF-03, CHAT-01, INT-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2), affiliate/ainflu-affiliate-backend@v1.5.27
---

# MKT-03 · Joining affiliate campaigns (influencer)

> In an affiliate campaign the brand pays a commission on each sale made through the influencer's tracked links, instead of a fixed fee. The influencer opens the campaign in the mobile app, taps "Generate Link" to get links for the campaign's products, shares them, and follows clicks, sales and payouts in "Link analytics". Everything behind the scenes (links, sales, commission, payouts) runs on the separate **Ainfluencer Affiliate platform**, not in the Ainfluencer wallet.

## Who and where

- **Influencer**, mobile app:
  - Explore → **Affiliate** tab and the affiliate section on All (first visit shows the "Ready to Start Earning?" intro: Generate → Share → Earn);
  - brand profile → "Affiliate Campaigns"; invitations (MKT-02); chat offer cards; email links `affiliate/invitation?ad_id=`;
  - Home → affiliate performance widget.
- Screens: **AFFILIATE_DETAIL** (campaign, products, "Generate Link", "Message brand") and **Analytics** (link analytics).

## How it works

1. **Open the campaign**: header, products with search, commission per product ("Commission:"), requirements and an earnings slider.
2. **Generate Link**: one tap creates links for **all products** of the campaign that don't have a link yet. Behind the scenes:
   - the first time, Ainfluencer registers the influencer on the Affiliate platform as an **affiliate**, matched by email (a new affiliate account is created if none exists);
   - the affiliate platform **approves the partnership automatically** and creates a contract with each product's commission and the tracked links (AFX-02).
   Success: "Affiliate links generated"; when everything already has a link: "You've already generated links for all products". A BrandID pop-up may follow ("Build My Storefront", INT-02).
3. **Share**: each product has **"Copy My Link"** / "Copy" and "View Product". The influencer posts the links in bio, stories, videos, etc. Shoppers buy on Amazon or Shopify (AFB-01).
4. **Link analytics** ("Link analytics — See clicks, sales, and earnings from your tracking links." → "View analytics"): periods 7 d / 30 d / 90 d / all; Clicks, Sales, "Earned over time", split by AMAZON / SHOPIFY, **"Product Performance"** (LINKS / REMOVED / EARNED per product) and **"Payouts"** with status (e.g. PROCESSING, PAID, FAILED) and "Next payout".
5. **Message brand** opens or starts an Ainfluencer chat thread for this campaign (CHAT-01). Do not use the affiliate platform's own chat (AFC-01 is not supported).

## Rules and limits

- **At least one Active social account is required** (approved in review, not archived). Otherwise: "You need at least one active social media account to take part in affiliate campaigns. Complete your profile and wait for admin approval, then try again." (the app currently shows a generic error instead, see Known issues).
- Commission rates are set by the brand per product; the app sends the displayed commission, the influencer can't negotiate it here.
- **Money** (AFS-10, AFA-08): commission is charged to the brand's card about **35 days after the sale**; the influencer receives it **minus the platform fee** (5% by default) through **Stripe Connect** on the Affiliate platform. Payouts shown in the app are already net of the fee. Nothing goes into the Ainfluencer wallet (PAY-03).
- **Nothing is paid until the influencer's Stripe Connect account is connected** on the Affiliate platform (AFA-08). The mobile app has no button to connect Stripe or to open the affiliate app; the influencer must sign in at affiliate.ainfluencer.com with the same email *(to verify the recommended login path, e.g. password reset or Google sign-in)*.
- When the campaign ends, its contracts are revoked the next night at 02:00 UTC and links stop earning (CMP-04, AFA-05 for what shoppers then see).
- Only Amazon sales generate commission today; Shopify sales are recorded but **no commission is calculated** (AFS-10 known gap).

## Statuses

| Shown in Analytics → Payouts | Affiliate platform status | Meaning |
|---|---|---|
| PROCESSING | `pending` | Charge to brand in progress |
| PAID | `succeeded` | Sent to the influencer's Stripe account (Stripe then pays out on its own schedule) |
| FAILED | `failed` / `server_error` | Brand's card charge failed; retried (AFS-10) |
| CANCELED / REFUNDED / PARTIALLY REFUNDED | same | Rare, admin actions |

Product link "REMOVED": the link was revoked or the product deactivated.

## Timers and automatic actions

- Sales older than **35 days** are billed nightly at 03:00 UTC (AFS-10).
- Campaign end: finished hourly on the affiliate platform; contracts revoked at 02:00 UTC next night.
- Campaign stats sync back to Ainfluencer hourly.

## Notifications

- None from Ainfluencer for link generation or sales. Payment success/failure is an in-app notification on the affiliate platform only (AFA-08).

## Common questions

### "Generate Link says it failed."
**Answer for the user:** Affiliate campaigns need at least one approved (Active) social account. Check your Profile; if your account is still pending review, please wait for approval and try again.
**What to check:** the user's integrations `qc_status` (needs `active`, not archived).
**Escalate when:** the user has an Active account and it still fails.

### "Where is my affiliate money? It's not in my wallet."
**Answer for the user:** Affiliate commissions don't go to your Ainfluencer wallet. They're paid through Stripe on the Ainfluencer Affiliate platform, about 35 days after the sale, once the brand's card is charged, and only if your Stripe account is connected there.
**What to check:** Analytics → Payouts; on the affiliate platform: sale dates, Stripe Connect status, commission transactions (AFA-08).
**Escalate when:** a sale is older than ~40 days, Stripe is connected and nothing is processing.

### "I see sales but earnings are $0."
**Answer for the user:** Earnings appear once the sale is processed; Shopify-store sales currently don't generate commission. We're checking with the team.
**Escalate when:** Amazon sales show $0 commission (to dev).

### "My links stopped working."
**Answer for the user:** When a campaign ends or a product is removed, links stop earning. Check the campaign end date and the "REMOVED" count in Product Performance.

### "Can I change the commission?"
**Answer for the user:** Commissions are set by the brand. Message the brand in the Ainfluencer chat if you'd like to discuss it.

### "Analytics says reports are unavailable."
**Answer for the user:** The affiliate data service didn't answer. Please try again in a few minutes ("Too many requests" means wait a moment).
**Escalate when:** it persists for hours.

## Known issues

- When joining is refused because there's no Active account, the app shows the generic "Failed to generate affiliate links. Please try again." instead of the server's explanation.
- No in-app way to connect Stripe or open the affiliate app; influencers who never sign in to the affiliate platform cannot receive payouts *(to verify)*.
- Shopify sales generate no commission (AFS-10).

## Related topics

- CMP-04 brand side, AFX-01 accounts linked by email, AFX-02 campaigns on the affiliate platform, AFA-05 links and redirects, AFA-08 earnings and payouts, AFS-10 billing, INF-03 account approval, CHAT-01 messaging, INT-02 BrandID.

## Technical reference

- Endpoints (`api/{locale}/v3/mobile`):
  - `GET ads/affiliate`, `GET ads/affiliate/from-brand`, `GET ads/affiliate/{id}` → `Modules/Mobile/Http/Controllers/API/AdAffiliateController` (`AdAffiliateService`).
  - `GET ads/affiliate/{id}/products` → `AinfluencerAffiliateService::listCampaignAmazonProductItemForAffiliate` → affiliate `GET affiliate/v1/integrations/ainfluencer/campaigns/{id}/products`.
  - `POST ads/affiliate/{id}/assign-influencer` `{custom_items:[{item_id, commission}]}` → `AdAffiliateController@assignToAd` (requires `qc_status=active` non-archived integration, else 403 `affiliate.join.no_active_account`) → `AinfluencerAffiliateService::assignCampaignAmazonProductToUserForAffiliate` → affiliate `POST affiliate/v1/integrations/ainfluencer/campaigns/assign-affiliate` (`AssignCampaignToAffiliateIntegrationController`: auto-approve partnership, contract, links, `campaign.updated` webhook).
  - `GET ainfluencer-affiliate/reports/{summary,timeline,products}`, `GET ainfluencer-affiliate/payouts` → `AffiliateReportController` → `AinfluencerAffiliateService::affiliateReport` (merges `reference_id`), affiliate `affiliate/v1/integrations/ainfluencer/{reports/*,payouts}` (payout net = amount − application_fee_amount).
- Core → affiliate proxy: `app/Modules/UserThirdParty/Services/AinfluencerAffiliateService.php` (`getUserThirdParty` auto-registers via `registerAffiliate` → affiliate `POST affiliate/v1/integrations/ainfluencer/register`, which links by email or creates a user with a generated password), driver `app/Modules/UserThirdParty/Driver/AinfluencerAffiliateDriver.php`.
- Data (core): `user_third_parties` (provider affiliate, type `affiliate`, third_party_unique_identity), `ad_affiliates` (products_count, affiliates_count, sale_amount…). Affiliate platform: `partnership_requests`, `partnership_contracts`, `affiliate_product_links`, `amazon_reports`, `commission_transactions`, `stripe_accounts`.
- Mobile: `features/explore/screens/affiliate/{AffiliateDetailScreen,AffiliateAnalyticsScreen,AffiliateScreen}.tsx`, `features/explore/affiliate/hooks/useGenerateAffiliateLinks.ts`, `features/ad/actions/useAssignAffiliateInfluencer.ts`; strings `localization/translations/en/campaign.js`.
