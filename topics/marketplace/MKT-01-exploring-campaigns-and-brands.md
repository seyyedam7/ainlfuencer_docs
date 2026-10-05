---
id: MKT-01
title: Exploring campaigns and brands
domain: Campaign marketplace (influencer side)
audience: [influencer]
apps: [mobile]
related: [MKT-02, MKT-03, MKT-04, OFR-01, INF-01, INF-03, CMP-02, CMP-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# MKT-01 · Exploring campaigns and brands

> The Explore tab is the influencer's marketplace: lists of live campaigns (paid collaborations, affiliate and gifting), search, campaign details, the brand's profile, and the option to report or block a brand.

## Who and where

- **Influencer**, mobile app. **Explore** is the first tab after login.
- Top bar: social-account selector (campaigns are matched to the selected account), **"Search campaigns"**, notifications bell.
- Explore tabs, in order: **All, Affiliate, Gifting, Most paid, For You, Invited to, My country, My category**.
- Screens: CAMPAIGN_DETAILS (paid campaign), AFFILIATE_DETAIL (MKT-03), GIFTING_DETAIL (MKT-04), BRAND_PROFILE ("Brand Profile"), published-content gallery for a campaign.

## How it works

1. **Account selector**: the lists depend on the selected social account (platform, country, categories). With no account, Explore opens the onboarding after ~3 s (INF-01).
2. **All**: mixes sections — top campaigns, campaigns by category, an affiliate section and a gifting section ("Show all" → Gifting campaigns list).
3. **Other tabs**: Affiliate (MKT-03), Gifting with "My Gifting Pipeline" (MKT-04), Most paid, For You, Invited to (MKT-02), My country, My category. Empty states: "No campaigns available right now!", "No campaigns found in this location!", "You have not been invited to any campaigns yet!".
4. **Search**: type in "Search campaigns"; recent searches are kept on the phone.
5. **Campaign details** (paid campaign): brief, deliverables, "Promotional Details" (duration, link in bio, shipping), requirements (country, category, gender, engagement, language), and content other influencers already published. Opening it records a view. The main button reads **"Make an Offer"**, **"Accept Invitation"** (when invited) or **"View offer"** (already offered). Before opening the offer form the app checks, in order: maintenance → account suspended → account archived → no usable account (sends to onboarding) → offer form (OFR-01).
6. **⋯ menu** on campaign details: **Report** (pick a reason from the list) and **Block brand**.
7. **Brand profile**: about, tabs "All Campaigns", "Affiliate Campaigns", "Gifting Campaigns", "Ongoing Offers" (only tabs with data appear). Invite-only or paused campaigns show a restriction pop-up. Menu: "About Brand" and **"Block Brand"** → "Do you want to block this brand?" — "If you block this brand, we will remove all the campaigns by this brand for you!"

## Rules and limits

- Only campaigns that are **active and approved by an admin** and whose brand is not suspended appear (CMP-02, BRD-04).
- **Invite-only** campaigns are hidden from public lists; invited influencers reach them through invitations (MKT-02).
- Campaigns limited to certain countries show only to accounts located there.
- A campaign shows only if it targets the **platform** of the selected account (Instagram, TikTok or YouTube).
- Campaigns of **blocked** brands are removed from all lists.
- Campaigns you already reported are removed from your lists *(to verify for every tab)*.
- Order: by the campaign's ranking score (recalculated daily), then newest first.
- Making an offer needs an account that is Pending or Active (INF-02, INF-03).

## Statuses

Not applicable on this screen (offer statuses are OFR-02).

## Timers and automatic actions

- Campaign ranking score: recalculated daily at 00:00 UTC (`ads:calculate-scores`).
- Campaigns disappear when they expire (end date, CMP-03) or are paused.

## Notifications

None specific; the weekly "campaigns for you" email (Monday 08:00 UTC) is NTF-04.

## Common questions

### "Why don't I see any campaigns?"
**Answer for the user:** Campaigns are matched to the social account selected at the top of Explore: its platform, country and categories. Try another tab or account, and make sure your profile is complete.
**What to check:** selected account's platform, `countries`, categories; account status.
**Escalate when:** an Active account sees nothing on All while campaigns exist for its platform.

### "I can see a campaign on the brand's profile but not in Explore."
**Answer for the user:** It may be invite-only, paused, limited to another country or for another platform.

### "I blocked a brand by mistake. How do I unblock it?"
**Answer for the user:** Unblocking isn't available in the app yet. Tell us the brand's name and we'll ask the team to unblock it.
**Escalate when:** always (needs dev/admin).

### "I tapped Block brand and nothing happened."
**Answer for the user:** The block was most likely saved even though the screen didn't close. Go back to Explore; that brand's campaigns should be gone.
**What to check:** `blocks` row for the user and brand identity.

### "I reported a campaign — did it go through?"
**Answer for the user:** The "reported" message appears even if sending failed, so if you're unsure, try again or tell us the campaign name.
**What to check:** `reports` row for the user and ad.

### "The button says View offer, not Make an Offer."
**Answer for the user:** You already sent an offer for this campaign; it opens your offer chat.

## Known issues

- Blocking from campaign details doesn't close the screen or show the "blocked" message (the block is still saved).
- The "reported" toast shows even when the report failed.
- No unblock button in the app.

## Related topics

- MKT-02 invitations, MKT-03 affiliate, MKT-04 gifting, OFR-01 making an offer, CMP-02 approval, CMP-03 expiry and pause.

## Technical reference

- Endpoints (`api/{locale}/v3/mobile`, `Modules/Mobile/Http/Controllers/API/AdController`): `GET ads/all`, `ads/all-add-once`, `ads/most-paid`, `ads/for-you`, `ads/my-country`, `ads/my-categories`, `ads/from-brand`, `ads/{id}`, `ads/{id}/published-contents`; `POST api/v3/mobile/ads/{ad}/view`. Services `Modules/Mobile/Services/Implementation/{AdService,AllAdsService,ForYouAdsService}` (filters: `ads.status=active`, `approval=approved`, `is_owner_suspended=0`, `invite_only=0`, `limit_country`, `settings->platform_ids`, blocked brands; order `ads.score DESC, created_at DESC`; counts from settings `backend_all_ads_mobile_side`).
- Brand: `GET brands/{id}/profile` → `BrandProfileController`; `GET offers/with-brand`.
- Privacy: `POST privacy/brand/block` → `BlockController` → `Services/Implementation/Mobile/BlockService` (unblock route exists, unused by app); `GET privacy/reports/items`, `POST privacy/reports` → `ReportController`.
- Job: `ads:calculate-scores` (`Console/Commands/Ad/CampaignScoreCalculator.php`, cron `0 */72 * * *` = daily 00:00).
- Data: `ads` (status, approval, invite_only, limit_country, settings, score, is_owner_suspended), `blocks` (blocker_id, blocked_id), `reports`.
- Mobile: `navigation/ExploreTabs.tsx`, `features/explore/*`, `features/ad/screens/campaignDetails`, `features/brand-profile/BrandProfileScreen.tsx`; strings `localization/translations/en/{explore,campaign,brand-profile}.js`.
