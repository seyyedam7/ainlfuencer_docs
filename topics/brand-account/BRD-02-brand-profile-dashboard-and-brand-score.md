---
id: BRD-02
title: Brand profile, dashboard and brand score
domain: Brand account and team
audience: [brand]
apps: [web]
related: [BRD-01, BRD-03, ACC-05, CMP-01, OFR-02, OFR-03, PUB-02, PUB-05, PAY-01, NTF-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# BRD-02 · Brand profile, dashboard and brand score

> The brand's company profile (what influencers see about the brand), the home dashboard, and the brand score with its Bronze, Silver, Gold and Platinum tiers.

## Who and where

- **Company profile**: web `/profile` (header avatar → Profile). Sections "Brand identity" ("How influencers see your brand on Ainfluencer."), "Campaign targeting", "Discovery", "Password & security" and "Delete Account". Team members working inside another brand's team can't open it.
- **Dashboard**: web `/` ("Dashboard" in the sidebar), the first page after login.
- **Brand score**: web `/profile/points` (header menu). Shows "Overall score", the tier ("{tier} brand"), "Boost your score", "Where your score comes from" and "Reviews you have written".

## How it works

**Company profile**
- Brand identity: "Company Name", "Location", "Language", "About Your Brand" ("Influencers read this when deciding whether to collaborate with you."), and the logo ("Upload logo", "PNG or JPG, square recommended, up to 5 MB").
- Campaign targeting: "Target Platform(s)", "Target Influencers" (Nano, Micro, Macro, Mega, Any range), "Company Size", "Estimated Budget".
- Discovery: "Categories" and "Related hashtags for your business".
- Changes are saved with the save bar at the bottom ("Unsaved changes" → save, or "Discard"). The personal fields (first/last name) and account deletion are covered in ACC-05.

**Dashboard** shows: a greeting with "+ Create Campaign", key numbers (campaigns, offers, invitations sent, deals closed), recent inbox messages, offers waiting for payment, deals waiting for a review, the wallet balance and score summary ("Your Score", "Wallet balance", "Number of deals", "Total payments"), the publishing schedule, team members, tutorial videos and promotions for affiliate, gifting and managed services.

**Brand score** is the sum of points for six activities:

| Activity (as shown) | Points |
|---|---|
| Active campaigns | 5 per live (active and approved) campaign |
| Deals closed | 10 per closed deal |
| Amount paid | 1 per $20 paid |
| Profile steps completed | 1 per filled profile item (name, logo, company size, location, categories, hashtags, budget, target influencers, objectives, ...) |
| Responses to offers (in less than one day) | 2 per offer answered within 1 day |
| Written reviews | 10 per review written about an influencer |

Tiers (shown on `/profile/points`):

| Tier | Score |
|---|---|
| Bronze | 0–499 |
| Silver | 500–1,499 |
| Gold | 1,500–2,499 |
| Platinum | 2,500 and above |

The page shows how many points are needed for the next tier and suggests actions ("Approve completed offers", "Write reviews", "Answer waiting offers", "Launch more campaigns").

## Rules and limits

- The score is **recalculated from the current state**, so it can go down: for example when campaigns end, "Active campaigns" points drop.
- Points update when the related event happens (a deal closes, a review is written, an offer is answered, a payment is made) and in a daily recalculation at 03:00 UTC (production).
- Logo: PNG or JPG, up to 5 MB on the profile page (8 MB during onboarding).
- "About Your Brand" and the company name can't contain links or contact details.
- The page text says higher tiers "rank higher in influencer search and carry a verified badge on every offer". In the code, the brand score is used to order campaigns in the weekly "campaigns for you" email to influencers (NTF-04); an effect on in-app ranking or a badge was not found *(to verify)*.
- Team members (in another brand's team) can see the dashboard and score of that brand but can't edit its profile.

## Statuses

Not applicable (tiers are described above).

## Timers and automatic actions

- Daily at 03:00 UTC (production only): brand scores older than a day are recalculated.

## Notifications

None.

## Common questions

### "How is my brand score calculated?"
**Answer for the user:** You earn points for live campaigns (5 each), closed deals (10 each), money paid (1 per $20), completed profile items (1 each), answering offers within a day (2 each) and reviews you write (10 each). You can see the breakdown under "Where your score comes from" on your Points page.
**What to check:** The breakdown on `/profile/points`; `brand_details.score`.
**Escalate when:** The breakdown clearly misses activity (for example no "Amount paid" points despite paid deals).

### "My score went down. Why?"
**Answer for the user:** Some points depend on what's live right now. For example, when a campaign ends or is paused, its "Active campaigns" points are removed.
**What to check:** Number of active, approved campaigns; recent campaign expiries (CMP-03).
**Escalate when:** Points dropped with no change in campaigns.

### "How do I reach Gold or Platinum?"
**Answer for the user:** Gold starts at 1,500 points and Platinum at 2,500. The fastest ways are closing deals, writing reviews for influencers, and answering offers within a day.
**What to check:** Nothing.
**Escalate when:** Never.

### "How do I change my logo or company description?"
**Answer for the user:** Go to Profile, update the logo or "About Your Brand", then save using the bar at the bottom of the page.
**What to check:** If they can't open Profile, they are probably inside another brand's team (BRD-03).
**Escalate when:** Saving or uploading fails with a valid file.

### "My team member can't edit our profile."
**Answer for the user:** Only the account owner can edit the company profile. Team members can use campaigns, offers and the inbox.
**What to check:** BRD-03.
**Escalate when:** Never.

## Known issues

- "Amount paid" points may not count payments made through the current payment system, because they are read from an older wallet report *(to verify)*.
- The tier badge text promises search ranking and a verified badge that weren't found in the code *(to verify with product)*.

## Related topics

- BRD-01: onboarding answers become profile fields.
- BRD-03: team member restrictions.
- ACC-05: personal details and deleting the account.
- OFR-02 / OFR-03: answering and paying offers (score activities).
- PUB-02 / PUB-05: approving deliveries and writing reviews.
- PAY-01: wallet balance shown on the dashboard.
- NTF-04: weekly emails that use the brand score.

## Technical reference

- **Endpoints**: `PUT /api/v3/brand/profile` → `ProfileController@update` (`team.blocker`); `PATCH /api/v3/brand/profile/{brandIdentityId}/change-logo`; `GET /api/v3/brand/profile/total-point` and `GET /api/v3/brand/profile/score-activities`; `GET /api/v3/brand/dashboard` → `BrandDashboardController@allData` → `Services/Implementation/v3/BrandDashboardService::allData` (waiting reviews, score summary, awaiting payments).
- **Score**: `app/Modules/Score` (config `Config/events.php` with ratios; actions in `Actions/Brand/*`; `Services/ScoreService::totalBrandPoint`; job `Jobs/UpdateBrandScore` writes `brand_details.score`; command `score:recalculate` daily 03:00, production only).
- **Data**: brand `identities` (`name`, `description`, `url`, `country_id`, `logo_id`), `brand_details` (`company_size`, `budget`, `target_influencers`, `objectives`, `score`), `scores`, `score_events`.
- **Web**: `src/components/pages/userInfo2`, `dashboardV3`, `points2` (tier thresholds are in the page code: 500 / 1,500 / 2,500). Strings `locales/en/userInfo.json`, `dashboard.json`, `points2.json`.
