---
id: DSC-03
title: Campaign invitations
domain: Finding and inviting influencers (brand)
audience: [brand, influencer, system]
apps: [web, mobile, email, background]
related: [DSC-01, DSC-02, MKT-02, CMP-01, CMP-02, CMP-05, SUB-04, ACC-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# DSC-03 · Campaign invitations

> A brand invites influencers to a campaign. Invitations are not sent instantly: twice a day Ainfluencer sends each influencer **one grouped email** (plus push and in-app for app users) listing the campaigns they were invited to. Influencers who aren't on Ainfluencer yet get an email with a sign-up link. The influencer then sees and answers the invitation in the app (MKT-02).

## Who and where

- **Brand** (web):
  - Influencer profile `/account/:id` and the quick-view pop-ups → "Invite to campaign" modal;
  - **`/favorite-influencers`** → "Invite selected to campaign" (DSC-02);
  - `/analytics` recommended influencers;
  - **Find Influencers** `/search` → "Invite to campaign" / "Invite to program" (DSC-01);
  - the campaign creation wizard (influencers picked while creating, CMP-01).
  - Product Gifting campaigns use their own "Request to send product" flow (CMP-05), not this one.
- **Influencer**: email; mobile Invited tab and "Campaigns Invited You To" screen (MKT-02). Not-yet-registered influencers use the web page `/invitation?ad_id=…&email=…` (or `/affiliate/invitation`, `/gifting/invitation`).
- **System**: job `invitations:send` at **06:00 and 18:00 UTC**.

## How it works

1. **Brand picks influencers and a campaign** and sends. Success message (profile/favourites): "{n} Invitation(s) were sent! Influencers will receive notification shortly…" — some of these pop-ups link to the old `/offers` list.
2. Two technical paths exist:
   - **Direct invitations** (profile, favourites, analytics): saved right away as invitations.
   - **Pending invitations** (Find Influencers and the campaign wizard): saved as "temporary" invitations and turned into real invitations **when an admin approves the campaign** (CMP-02). The search pop-up says "After admin approval, your invitations will be sent."
3. **06:00 / 18:00 UTC**: for every influencer with unsent invitations (at least 1 hour old), Ainfluencer sends **one email** listing the campaigns, plus an in-app notification and push for registered users ("You are invited to some brand campaigns!" — "You've been invited by N brand(s) to make offers for their campaigns."). The push opens the app's invitation list.
4. **Registered influencers** see the invitation in the app (MKT-02) once it has been emailed and the campaign is **active and approved**. They can open it, make an offer ("Accept Invitation") or hide it.
5. **Not-registered influencers** (profiles found by Ainfluencer's search but without an account) get the email with a link to `/invitation`. The page shows "You have been invited to collaborate…", then "Create My Account" (email code verification, category and location), then a "Download App" step. Their social account is created as Draft and they continue in the app (INF-02).

## Rules and limits

- The campaign must be **active** and approved or waiting for approval: "can't invite to inactive campaigns". Brands can't invite to another brand's campaign or invite themselves.
- **Only influencers with an email address on file are invited.** Others are silently skipped.
- On an approved campaign, influencers who already sent an offer for it are skipped.
- Re-inviting the same influencer to the same campaign only resends if the earlier email failed or was sent **14+ days** ago.
- **Plan limit "influencer invites"** (HTTP 402 pop-up, SUB-04). Counted per influencer selected. Default monthly allowances: Free 1, Starter 50, Pro 500, Ultimate 50,000; add-on packs shown in the pop-up (+100 for $29, +250 for $59, +500 for $99).
- Invitations sent to a campaign that is **still waiting for approval** are **deleted after 3 days** if it is still not approved by then.
- **Expiry**: invitations not yet emailed **1 week** after creation are expired and removed. Invitations already emailed don't expire by time; they disappear from the influencer's list when the campaign stops being active/approved or the influencer makes an offer.
- If an admin **rejects** the campaign, its pending invitations (both kinds) are deleted (CMP-02).
- A suspended brand's pending invitations are deleted (BRD-04).

## Statuses

| Internal (`invitations.status`) | Meaning |
|---|---|
| `pending` | Created; may or may not be emailed yet (`sent_at`, `email_status`) |
| `seen` | Influencer opened it in the app (MKT-02) |
| `expired` | Not emailed within 1 week; soft-deleted |

Plus `is_hidden` (influencer hid it) and `offer_id` (influencer made an offer from it).

## Timers and automatic actions

| When (UTC) | What |
|---|---|
| 06:00, 18:00 | Expire unsent invitations older than 1 week; send grouped emails/push for cycles at least 1 hour old |
| 3 days after inviting | Delete invitations of campaigns still pending approval |
| Every 6 h | Hard-delete cancelled invitations older than 24 h |
| On admin approval | Pending (temporary) invitations become real invitations |

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Invitation run | Registered influencer | Email (grouped), push + in-app "You are invited to some brand campaigns!" |
| Invitation run | Not-registered influencer | Email with sign-up link (sent through the marketing mail provider) |
| Nothing | Brand | No notification when invitations are delivered |

## Common questions

### "I invited influencers but they say they got nothing."
**Answer for the user:** Invitations are grouped and emailed twice a day, at 06:00 and 18:00 UTC. Influencers without an email on file can't be invited, and invitations to a campaign under review are sent only after it's approved.
**What to check:** `invitations` rows for the ad and influencer (`email_status`, `sent_at`, `deleted_at`); `temporary_invitations` for the ad; campaign `approval`/`status`.
**Escalate when:** rows exist, the campaign is approved and active, and nothing was sent after two runs.

### "I invited people from Find Influencers to my live campaign and nothing happened."
**Answer for the user:** We're checking this with our team. In the meantime, invite them from their profile page or a favourites list, which sends invitations directly.
**What to check:** `temporary_invitations` rows for the ad with no matching `invitations` rows.
**Escalate when:** always (see Known issues).

### "Why does it say I reached my invite limit?"
**Answer for the user:** Your plan includes a monthly number of influencer invites. You can buy an invite pack, invite fewer influencers, or upgrade.

### "My invitations disappeared after the admin rejected my campaign."
**Answer for the user:** Rejecting a campaign cancels its pending invitations. Please fix the campaign, resubmit, and invite again after approval.

### "An influencer can't find my invitation in the app."
**Answer for the user:** It shows in their app after the next 06:00/18:00 UTC email run and only while the campaign is active and approved. If they already made an offer, it moves to their offers.

### "I'm an influencer and I got an invite email but I don't have an account."
**Answer for the user:** Open the link in the email, create your account and verify your email, then download the app to connect your social account and reply to the brand.

## Known issues

- **Invites from Find Influencers / "Invite to program" to an already-approved campaign may never be sent** *(to verify with the dev team)*: these are saved as pending invitations, which are only converted when an admin approves the campaign. Affiliate campaigns skip admin approval (CMP-02), so their "Invite to program" invites may never be converted. Workaround: invite from the influencer's profile or a favourites list. Escalate to dev.
- Invite-allowance rollback on rejection is applied to the admin's account instead of the brand's (see CMP-02) *(to verify)*.
- "Invite sent" pop-ups link to the old `/offers` list.

## Related topics

- MKT-02 influencer side, DSC-01 search, DSC-02 favourites, CMP-02 approval, CMP-05 gifting invitations, SUB-04 invite limits, ACC-01 sign-up.

## Technical reference

- Endpoints:
  - `POST api/v2/user/invitations` (`feature:affiliates_invites`) → `Http/Controllers/v2/Frontend/InvitationController@store` → `Services/Implementation/InvitationService::store` / `refreshAndStoreInvitations` (jobs `Jobs/Ad/InviteInsertion` → `CreateInvitationCycle`, `AfterInviteInsertion` (14-day resend), `Jobs/Ad/PendingApproval/DeleteInvitations` delayed `Constants::LIMIT_DELETE_INVITATIONS_DAYS` = 3).
  - `POST api/v2/temp-invitations` (`feature:affiliates_invites`) → `TemporaryInvitationController` → `Services/Implementation/TemporaryInvitationService` (`temporary_invitations`); converted only by `Jobs/Ad/TemporaryInvitationSender` from `Admin/AdService::handlePostAdApproval` (`AdApprovedEvent`).
  - `GET api/v2/invitations/find?ad_id&email` → `InvitationController@findByDetail`; guest sign-up `POST api/v2/guest/signup` → `Services/Implementation/Guest/InvitationSignupService`.
  - Mobile side endpoints: see MKT-02.
- Job: `invitations:send` → `Console/Commands/SendInvitations.php` (`EXPIRES_AFTER_IN_WEEKS = 1`, cycles `updated_at <= now-1h`), `Jobs/Ad/Invitation/BulkMarkAsExpired` (only `sent_at IS NULL`), mail `Mail/Invitations/AggregatedInvitation` / `AggregatedExternalInvitation` via `Helpers/InvitationHelper`, push `Modules/Mobile/Notifications/Invitation/AggregatedInvitationSentNotification` (event `events:mobile:invitation:internal:sent`). `invitations:delete` every 6 h.
- Data: `invitations` (ad_id, source_id, target_id, status, email_status, sent_at, is_hidden, offer_id, deleted_at), `invitation_cycles` (identity_id, ad_ids, is_sent), `temporary_invitations` (ad_id, brand_id, influencer_id), `invitation_logs`, `feature_usages`.
- Plan values: feature `affiliates_invites` (seed `plan_price_feature_data_production.php`); web limit modal `components/common/modals/limitModals`.
