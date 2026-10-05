---
id: MKT-02
title: Invitations (influencer side)
domain: Campaign marketplace (influencer side)
audience: [influencer]
apps: [mobile, email]
related: [DSC-03, MKT-01, MKT-03, MKT-04, OFR-01, NTF-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# MKT-02 · Invitations (influencer side)

> When a brand invites an influencer (DSC-03), the influencer gets a grouped email and push, then sees the invitation in the app. From there they open the campaign and make an offer (paid), generate links (affiliate) or accept the gift (gifting), or hide the invitation.

## Who and where

- **Influencer**, mobile app:
  - Explore → **"Invited to"** tab;
  - Home → invitations widget ("<Brand> has invited you to make an offer on their campaign") and **"Invited Campaigns"**;
  - **"Campaigns Invited You To"** screen (opened by the invitation push and the `invitations` deep link).
- Email links: `…/invitation?ad_id=`, `…/affiliate/invitation?ad_id=`, `…/gifting/invitation?ad_id=&email=` and `view-campaign/<id>` links open the matching campaign in the app.

## How it works

1. After the 06:00 / 18:00 UTC run (DSC-03) the influencer gets a push "You are invited to some brand campaigns!". Tapping it opens "Campaigns Invited You To".
2. Each row shows "Brand: …" and "Campaign: …". Actions: **"Mark as Read"** / **"Mark All as Read"** (marks as seen), delete one or **"Delete All"** (hides them; "Are you sure you want to delete …"). Opening a campaign also marks its invitation as seen.
3. Tapping a row opens the right screen: paid → campaign details with **"Accept Invitation"**; affiliate → affiliate detail (MKT-03); gifting → gifting detail with "Accept Invite" (MKT-04).
4. **Paid campaign**: "Accept Invitation" opens the normal offer form (OFR-01). When the influencer came from an email link and has several accounts, the app first asks **"Accept Invitation for Which Account?"**. An offer started from an **email invitation link** **skips the campaign's audience requirements** (gender/location checks). Offers started from the in-app invitation list go through the normal checks.
5. Once an offer is made, the invitation leaves the list.

## Rules and limits

- An invitation appears in the app only when **all** are true: its email was sent successfully, the campaign is **active and approved**, the brand is not suspended, the campaign isn't archived, the influencer hasn't hidden it, and no offer was made from it yet.
- Hiding is permanent in the app (no "unhide").
- Deep links need the user to be **logged in**; a logged-out user just lands on Login and must open the link again after logging in.
- Account rules for offers still apply: suspended, archived or Draft accounts can't make offers (INF-02, INF-04).
- Gifting invitations from Product Gifting campaigns are answered on the gifting screen (MKT-04).

## Statuses

| What the influencer sees | Internal | Meaning |
|---|---|---|
| Unread row | `invitations.status = pending` | Not opened yet |
| Read row | `seen` | Opened / marked as read |
| Gone | `is_hidden = 1`, `offer_id` set, campaign no longer live, or expired | Removed from the list |

## Timers and automatic actions

- Invitations reach the app only after the twice-daily email run (06:00 / 18:00 UTC).
- Unsent invitations expire after 1 week (DSC-03).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Invitation run | Influencer | Grouped email; push + in-app "You've been invited by N brand(s)…" (opens invitation list) |

## Common questions

### "The brand says they invited me but I don't see it."
**Answer for the user:** Invitations are delivered twice a day (06:00 and 18:00 UTC). If the campaign is paused, expired or still under review, the invitation won't show. Pull to refresh the Invited tab after the next delivery time.
**What to check:** `invitations` row (email_status success, sent_at, is_hidden, offer_id) and the campaign's status/approval.
**Escalate when:** the row was sent successfully, the campaign is live, and it still doesn't show.

### "I tapped the link in the email and the app only opened Login."
**Answer for the user:** Log in first, then tap the link in the email again; it will open the campaign.

### "I deleted an invitation by mistake."
**Answer for the user:** Deleted invitations can't be restored in the app, but you can still find the campaign in Explore or on the brand's profile and make an offer, unless it's invite-only. For invite-only campaigns, ask the brand to invite you again.
**Escalate when:** the campaign is invite-only and the brand can't re-invite (an admin can clear `is_hidden`).

### "Can I accept an invitation if I don't meet the campaign's requirements?"
**Answer for the user:** Open the invitation from the link in your invitation email; offers started that way aren't blocked by the campaign's gender/location requirements. From inside the app the normal requirements apply.

### "The invitation disappeared."
**Answer for the user:** It's removed when you make an offer, when you delete it, or when the campaign ends, is paused or is no longer approved.

## Known issues

- Deep links don't work when logged out (lands on Login).
- In-app notification taps for some events open Explore instead of the invitation list (NTF-02).

## Related topics

- DSC-03 brand side and sending rules, MKT-01 campaign details, MKT-03 affiliate, MKT-04 gifting, OFR-01 offers, NTF-02 push routing.

## Technical reference

- Endpoints (`api/{locale}/v3/mobile`): `GET invitations`, `POST invitations/see` `{invitation_ids[]|select_all}`, `POST invitations/hide` → `Modules/Mobile/Http/Controllers/API/InvitationController` → `Services/Implementation/Mobile/InvitationService` (list filter: target user, `is_hidden=0`, `email_status=success`, ad active+approved, `is_owner_suspended=0`, `archived_at` null, `offer_id` null). `GET ads/invited/summary` → `AdController@getInvitedCampaignsStatistics` (Home widget).
- Offer from an invitation deep link: the campaign's location/gender requirements are not applied to that offer (`OfferCreatorService`, mobile `features/make-offer/hook/useMakeOffer.ts`; see OFR-01).
- Push event `events:mobile:invitation:internal:sent` → `INVITED_CAMPAIGN_DETAILS`.
- Mobile: `features/home/screens/InvitedCampaignScreen.tsx`, `features/explore/screens/invitedAds`, `features/{InvitationRouter,AffiliateInvitationRouter,GiftingInvitationRouter}`, linking in `navigation/Container.tsx`; strings `localization/translations/en/home.js` (`invitation.*`).
- Data: `invitations` (status, is_hidden, email_status, sent_at, offer_id, target_id, ad_id).
