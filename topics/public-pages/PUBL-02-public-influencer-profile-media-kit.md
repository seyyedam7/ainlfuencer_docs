---
id: PUBL-02
title: Public influencer profile (media kit)
domain: Public (no-login) pages
audience: [influencer, brand]
apps: [web, mobile]
related: [INF-02, INF-05, DSC-01, PUB-05, PUBL-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# PUBL-02 · Public influencer profile (media kit)

> Every connected social account has a shareable web page, the influencer's **media kit**, at `app.ainfluencer.com/public/profile/<account id>`. Influencers share it with brands outside Ainfluencer; anyone can open it without logging in.

## Who and where

- **Influencers** get the link in the mobile app:
  - at the end of onboarding, "CONGRATULATIONS" ("Your media kit is live": copy link / share) (INF-02);
  - on the Profile tab, the account card's "Share" button (INF-05).
- **Brands** can copy or share a profile link from the influencer pop-up in search and favourites (DSC-01).
- **Visitors** open `/public/profile/<id>` in a browser. Logged-out visitors see the page with an Ainfluencer logo bar at the top; logged-in brands see the same profile as on `/account/<id>` (with their usual brand actions).

## How it works

1. The page loads the account's media kit: profile photo, name and bio, platform and audience statistics, categories, gallery photos, past collaborations, content the influencer delivered through Ainfluencer (approved deliveries), reviews from brands, brands they worked with, other connected accounts of the same influencer, and similar influencers.
2. Each visit counts as a **profile view** for the influencer.
3. Content items can be opened from the page; delivered content uses the same viewer as PUBL-03.

## Rules and limits

- The link is per **social account**, not per person: an influencer with Instagram and TikTok has two links.
- The page is **cached for up to 12 hours**: changes made in the app (bio, photos, categories) may take up to 12 hours to appear.
- Follower counts and statistics come from Ainfluencer's data refresh, which is monthly for Instagram (INF-06), so they may be out of date.
- Removing a social account from the app (INF-05) makes its page stop loading.
- Reviews shown are those approved by Ainfluencer (PUB-05).

## Statuses

Not applicable.

## Timers and automatic actions

- Profile data cache: 12 hours.
- Monthly Instagram data refresh (INF-06).

## Notifications

None.

## Common questions

### "Where do I find my media kit link?" (influencer)
**Answer for the user:** In the app, go to the Profile tab and tap "Share" on your social account card. You can copy the link or share it directly.
**What to check:** The account is connected and active (INF-05).
**Escalate when:** The Share button is missing for an active account.

### "I updated my bio/photos but my media kit still shows the old version."
**Answer for the user:** Your public page refreshes within 12 hours. Please check again later.
**What to check:** The change is saved in the app.
**Escalate when:** Still not updated after 24 hours.

### "My follower count on the media kit is wrong."
**Answer for the user:** Statistics are refreshed periodically (monthly for Instagram), so recent growth may not show yet.
**What to check:** INF-06 (last refresh).
**Escalate when:** The numbers are clearly from another account.

### "The link shows an empty page." (visitor or influencer)
**Answer for the user:** The link may be for an account that was removed or no longer exists. Please ask the influencer for a new link from the app.
**What to check:** The account (integration) id in the link exists and isn't deleted.
**Escalate when:** The account is active and the page still doesn't load.

### "Can brands contact me from my media kit?" (influencer)
**Answer for the user:** Brands who are on Ainfluencer can open your profile, invite you to campaigns, and chat with you after you send or accept an offer. Contact details aren't shown; all conversations happen in the Ainfluencer chat.
**What to check:** Nothing.
**Escalate when:** Never.

### "I have two accounts. Which link do I share?" (influencer)
**Answer for the user:** Each social account has its own page. Share the one that fits the brand; the page also lists your other connected accounts.
**What to check:** Nothing.
**Escalate when:** Never.

## Known issues

- Changes take up to 12 hours to appear because of caching.

## Related topics

- INF-02: media-kit onboarding that fills this page.
- INF-05: managing social accounts and the Share button.
- DSC-01: the brand's view of an influencer profile.
- PUB-05: reviews shown on the profile.
- PUBL-03: viewing delivered content.

## Technical reference

- **Endpoints** (no login needed): `GET /api/v3/influencer/profile/public/all-data/{integrationId}`, `…/elastic-data/{integrationId}`, `…/{integrationId}/all-accounts`, `…/preview-data/{integrationId}`, `…/published-contents/{integrationId}`, `PATCH …/hit-profile-views/{integrationId}` → `v3/Influencer/PublicProfileController`, `PublicPublishedContentController` → `Services/Implementation/Influencer/InfluencerProfileService::getAllData` (cache `INFLUENCER_DETAIL_CACHE_TTL`, default 43,200 s = 12 h).
- **Data**: `integrations` (the account; id in the URL), `identities`, `influencer_details`, reviews, published contents; Elastic `integration` index for statistics.
- **Web**: route `/public/profile/:id` → `src/components/pages/influencerProfile` (logged-out branch uses the `…PublicPage` services). **Mobile**: link built as `https://app.ainfluencer.com/public/profile/{integration_id}` (Congratulations screen, Profile account card).
