---
id: INT-02
title: BrandID (sister product)
domain: Store and partner connections
audience: [brand, influencer, system]
apps: [web, mobile, background]
related: [MKT-03, CMP-04, PUB-06, ACC-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# INT-02 · BrandID (sister product)

> BrandID (brandid.app) is a sister product: a link-in-bio page and storefront for creators, part of the same "brandID family" of apps. Ainfluencer promotes it in a few places and exchanges a small amount of account data with it. Ainfluencer and BrandID are **separate accounts and separate products**. BrandID support questions are out of scope for Ainfluencer support, apart from the touchpoints below.

## Who and where

| Touchpoint | Who | Where | What it does |
|---|---|---|---|
| Storefront promo | Influencer | Mobile app, affiliate campaign details (AFFILIATE_DETAIL), right after "generate affiliate links" succeeds | Pop-up "Your storefront at your fingertip. Now earn from 5 million more products". "Build My Storefront" opens the BrandID affiliate-program page in the browser ("Free forever. No approval needed."). "Maybe Later" closes it. |
| "Our apps" page | Brand | Web, sidebar → `/our-apps` ("The brandID family · 8 apps") | Links to BrandID, DMpro, ContactUs, SignID, Chatgram, BookMe. Just links: no data is shared. |
| "Book strategy call" | Brand | Web sidebar | Opens a BookMe (bookme.brandid.app) booking page. |
| "List in the brandID storefront" | Brand | Web, affiliate "Grow Program" pop-up (CMP-04), Pro & Ultimate | "Add to brandID storefront" opens a BrandID demo page and shows "Requested". See Known issues. |
| Account link (BrandID → Ainfluencer) | System | Server to server | When a user signs up on BrandID through Ainfluencer, or saves a BrandID username, BrandID tells Ainfluencer, which records the link. BrandID can also sign a user in with their Ainfluencer account (creating one if the email is new). |
| Video translation | Brand | Web, `/influencer-content`, `/profile/translated-contents` | Paid translation/dubbing runs on BrandID's translation service. See PUB-06. |

## How it works

1. **Influencer storefront promo:** an influencer joins an affiliate campaign and generates links (MKT-03). On success, the BrandID promo pop-up opens. Tapping "Build My Storefront" only opens a web page; it doesn't create anything in Ainfluencer and doesn't move the affiliate links to BrandID.
2. **Account link:** when a BrandID account is created from an Ainfluencer sign-in, BrandID calls Ainfluencer:
   - Ainfluencer marks the user as "connected to BrandID";
   - a BrandID username, when sent, is stored as a BrandID connection on the user.

   These calls are signed by BrandID and need no action from the user.
3. **Sign in to BrandID with Ainfluencer:** BrandID's app can log a person in using their Ainfluencer identity. If no Ainfluencer account exists for that email, one is registered.

## Rules and limits

- The influencer "Get Started" card for BrandID in chat (and the API behind it that builds a BrandID sign-in link) is shown **only in development builds** of the mobile app, not in the store version. Brands have no BrandID link (not implemented).
- Ainfluencer doesn't show BrandID data (storefront, earnings, billboards, tips) anywhere in the live apps.
- Earnings made on BrandID are not part of the Ainfluencer wallet (PAY-03) or the affiliate platform payouts (AFA-08).

## Statuses

None in Ainfluencer apart from internal flags: `users` "connected to BrandID" and a `user_third_parties` row with the BrandID provider.

## Timers and automatic actions

None.

## Notifications

None.

## Common questions

### "I tapped 'Build My Storefront'. Are my Ainfluencer affiliate links in BrandID now?"
**Answer for the user:** No. BrandID is a separate app; the button just opens its website. Your Ainfluencer affiliate links stay in the Ainfluencer app. If you want them in a BrandID storefront, add them there yourself.
**What to check:** n/a.
**Escalate when:** n/a.

### "I signed up to BrandID with my Ainfluencer account. Why do I have two accounts / two passwords?"
**Answer for the user:** BrandID and Ainfluencer are separate products that can share your sign-in, but each keeps its own account and settings. Account issues on BrandID are handled by BrandID support.
**What to check:** if needed, whether the Ainfluencer user is marked as connected to BrandID (Technical reference).
**Escalate when:** the person can't log in to Ainfluencer itself (ACC-01).

### "I clicked 'Add to brandID storefront' and it says 'Requested'. When will my products appear?"
**Answer for the user:** That button currently only opens BrandID's information page. No request reaches our team. Contact us if you want to be listed and we'll pass it on.
**What to check:** n/a (see Known issues).
**Escalate when:** the brand wants the listing (pass to the partnerships/BrandID team).

### "Where can I see the BrandID card in chat that a colleague mentioned?"
**Answer for the user:** That card isn't available in the current app version. You can go to brandid.app directly.
**What to check:** n/a.
**Escalate when:** n/a.

## Known issues

- **"Add to brandID storefront" sends nothing.** The pop-up shows "Requested" after the click, but no request is recorded anywhere; it only opens a demo page. Escalate to product if brands rely on it.
- **BrandID link for brands is not implemented.** The server can only build a sign-in link for influencers.

## Related topics

- MKT-03: generating affiliate links (where the promo appears).
- CMP-04: the affiliate "Grow Program" pop-up.
- PUB-06: content translation powered by BrandID.
- ACC-01: Ainfluencer sign-in.

## Technical reference

- **Endpoints (`core/backend/api/app/Modules/UserThirdParty/Routes/api.php`, prefix `api/user-third-party/brandid/v1`, controller `BrandIDIntegrationController`):**
  - `GET link` (auth; influencers only; builds a signed BrandID sign-in URL via `BrandidService::getRedirectUrl`; used by mobile `features/chat/components/bubbles/BrandIdIntegration.tsx`, rendered only when `__DEV__`).
  - `POST user-created` (`verify.brandid.jwt`; `BrandidService::registeredInBrandid` → `UserRepository::connectToBrandid`).
  - `POST username` (`verify.brandid.jwt`; `storeBrandidUserAsAThirdParty` → `user_third_parties` provider = BrandID provider name).
  - `POST auth-by-ain` (guest; BrandID-signed token in `AuthByAinRequest`; logs in or registers by email via `AuthService::loginByService` / `registerByService`).
  - `GET billboard` and `GET tips` exist but no live app calls them.
- **Mobile promo:** `src/features/explore/affiliate/components/detail/BrandIdAffiliatePromoModal.tsx` (opens `https://brandid.app/affiliate-program/...`), triggered from `src/features/explore/screens/affiliate/AffiliateDetailScreen.tsx` on link generation success.
- **Web:** `core/frontend/web/src/components/pages/ourApps/index.tsx` (static links); `core/frontend/web/src/components/pages/affiliate/parts/syncAffiliate/GrowProgramModal.tsx` (storefront button, local state only).
- **Translation driver:** `core/backend/api/app/Modules/Translation/Drivers/BrandId/*`, callback `POST /api/v2/webhooks/translations/callback` (PUB-06).
- **Data:** `user_third_parties` (BrandID provider rows), the user's "connected to BrandID" flag written by `connectToBrandid`.
