---
id: PUBL-01
title: Public campaign preview and join
domain: Public (no-login) pages
audience: [influencer, brand]
apps: [web, mobile]
related: [CMP-03, CMP-04, CMP-05, DSC-03, MKT-02, ACC-01, NTF-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# PUBL-01 · Public campaign preview and join

> A shareable page that shows a campaign to anyone, without logging in, with a **"Join Campaign to Collaborate"** button that leads influencers to sign up and continue in the mobile app. Brands get the link from their campaign list.

## Who and where

- **Brands** copy the public link from `/campaigns` (copy link action), from the affiliate campaign list, or from the affiliate dashboard's Campaigns tab (CMP-03, CMP-04).
- **Anyone** with the link can open it:
  - `/view-campaign/<id>`: paid collaboration campaigns (the copy-link button also uses this address for gifting campaigns);
  - `/view-campaign-affiliate/<id>`: affiliate campaigns;
  - `/view-campaign-seeding/<id>`: gifting campaigns (exists, but the copy-link button doesn't produce it).
- **Influencers with the app installed** who open a `view-campaign/<id>` or `view-campaign-affiliate/<id>` link on their phone may be taken straight into the app (NTF-02).

## How it works

1. The page shows the brand ("Brand Name"), the campaign images/videos and caption ("Copy Caption"), and "Requirements:", for example "Influencer Gender:", "Followers:", "Speak:", "Influencer Country:", and for affiliate campaigns "Commission:", "Date:", "Products:". Gifting campaigns show their requirement chips. Attached campaign files can be downloaded.
2. The visitor clicks **"Join Campaign to Collaborate"** (opens in a new tab):
   - paid collaboration → `/invitation?ad_id=<id>`;
   - affiliate → `/affiliate/invitation?ad_id=<id>`;
   - gifting → `/seeding/invitation?ad_id=<id>` (**broken**, see known issues).
3. On the invitation page the influencer sees "Welcome", then "Create My Account" (name, email, password, category, location, 5-digit email code), then "Download App" (ACC-01). The collaboration itself (offer, join, gift request) happens in the mobile app (MKT-01, MKT-03, MKT-04).
4. In the app, a campaign link opens the campaign details (paid), the affiliate campaign, or the gifting campaign, if the influencer is logged in.

## Rules and limits

- Only campaigns that are **active and approved** can be viewed. Others (draft, pending review, paused, expired, rejected, or taken down because the brand was suspended) return "This campaign is no longer available." *(to verify how this check applies to affiliate and gifting campaigns, which are stored differently)*.
- No login is needed to view the page. Joining requires an influencer account.
- Brands can't use the "Join" flow; a brand email on the invitation sign-up gets "This email is associated to a brand account, please try with another one!".
- Opening the link in the app works only when the influencer is logged in; otherwise they land on the login screen.

## Statuses

| What the visitor sees | Meaning |
|---|---|
| Full campaign page | Campaign active and approved |
| Empty page / "This campaign is no longer available." | Campaign not live (any other status) |

## Timers and automatic actions

None. The page simply stops working when the campaign expires or is paused (CMP-03).

## Notifications

None.

## Common questions

### "The campaign link I shared shows nothing / says it's no longer available."
**Answer for the user (brand):** Public links work only while the campaign is live (active and approved). If it's paused, expired, still in review or was rejected, the link won't show it.
**What to check:** `ads.status` and `ads.approval` (CMP-02, CMP-03).
**Escalate when:** The campaign is active and approved but the page doesn't load.

### "An influencer clicked 'Join' on my gifting campaign and got an error / login page."
**Answer for the user (brand):** This is a known problem with the join button on gifting campaign pages. Please invite the influencer from the gifting campaign instead, or ask them to find the campaign in the app's Gifting tab.
**What to check:** Campaign type is gifting.
**Escalate when:** Always report to dev (known bug), especially if brands rely on the link.

### "I clicked 'Join Campaign' but I already have an account." (influencer)
**Answer for the user:** Just open the Ainfluencer app, log in, and find the campaign (Explore, or the Invited tab if the brand invited you). The link also opens the campaign directly in the app when you're logged in.
**What to check:** The influencer account exists (`users.type` = influencer).
**Escalate when:** Never.

### "Can I share the campaign link on social media?" (brand)
**Answer for the user:** Yes. Anyone can open it without logging in, and influencers can join from it.
**What to check:** Nothing.
**Escalate when:** Never.

### "The link opens the website instead of the app." (influencer)
**Answer for the user:** Make sure the Ainfluencer app is installed and you're logged in, then tap the link again. You can also use the website's "Join Campaign to Collaborate" button.
**What to check:** App installed and logged in.
**Escalate when:** Never.

## Known issues

- **Gifting "Join" button is broken**: it links to `/seeding/invitation`, which doesn't exist (logged-out visitors land on login, logged-in users see "not found"). The working page is `/gifting/invitation`. Escalate to dev.
- **Gifting links are copied as `/view-campaign/<id>`** instead of `/view-campaign-seeding/<id>`; the page usually still detects gifting campaigns from their data.
- When the campaign isn't available, the page shows no friendly "not available" screen; it stays empty with an error message.

## Related topics

- CMP-03: copying the link from the campaign list. CMP-04, CMP-05: affiliate and gifting campaigns.
- DSC-03, MKT-02: invitations (the same invitation pages).
- ACC-01: sign-up on the invitation page.
- NTF-02: opening links in the app.

## Technical reference

- **Endpoints**: `GET /api/v2/guest/campaigns/{id}` → `v2/Guest/CampaignController@show` → `Modules/Mobile/Services/Implementation/AdService::show` → `findActiveAndApprovedById` (417 "This campaign is no longer available." otherwise); `GET /api/v2/guest/campaigns/contents/download/{contentId}` (and `/files/{fileId}`) → `CampaignContentController`. Sign-up: `POST /api/v2/guest/signup` (ACC-01).
- **Data**: `ads.status`, `ads.approval`, `ads.is_owner_suspended`.
- **Web**: `src/components/pages/viewCampaignNoAuth` (shared desktop/mobile views; `campaignPreviewModel.ts` `detectCampaignPreviewKind`; join URL built in `desktop/index.tsx` and `mobile/index.tsx` line ~93), `viewCampaignAffiliateNoAuth`, `viewCampaignSeedingNoAuth`, invitation pages `src/components/pages/invitation`. Copy-link logic: `src/components/pages/campaignsV2` (`handleCopy`).
- **Mobile deep links**: `view-campaign/<id>` → `InvitationRouter`; `view-campaign-affiliate/<id>` → `AffiliateInvitationRouter` (`src/navigation/Container.tsx`).
