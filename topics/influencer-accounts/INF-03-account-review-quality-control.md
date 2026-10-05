---
id: INF-03
title: Account review (quality control)
domain: Influencer social accounts
audience: [influencer, admin]
apps: [mobile, email]
related: [INF-01, INF-02, INF-04, OFR-01, NTF-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# INF-03 · Account review (quality control)

> Ainfluencer staff review influencer accounts in two separate places: **manual ownership verification** (when the app could not prove automatically that the influencer owns the account) and **quality review** (after the media kit is finished). Until an account passes quality review, offers it makes are held back and brands don't see them.

## Who and where

- **Admin**: reviews in the admin panel (manual verification requests list; influencer list with quality-control status).
- **Influencer**: sees the result on the Profile tab account card, by push and by email. Push notifications about account changes open the Profile tab.

## How it works

### A. Manual ownership verification ("Pending Verification")

1. An account lands here when the Instagram DM check failed twice, the influencer asked for manual review, or the automatic step could not finish (INF-01). The card shows **"Pending Verification"**.
2. A background job fills in follower numbers and the profile picture every 30 minutes so the admin can judge the account (INF-06).
3. The admin either:
   - **verifies** it → the influencer gets "Congratulations! Your <platform> @username was approved for Ainfluencer Marketplace." The account is created as a **Draft** and the influencer must finish the media kit (INF-02);
   - **rejects** it with reasons → card shows **"Verification Rejected"** and a pop-up lists the reasons. Preset reasons include "The account is not authentic", "The account doesn't meet the minimum required followers (5K)", "The account is not public", "The account violated the platform guidelines and terms", "Verification of the ownership of account failed.", "Username is wrong or page doesn't exist."
4. To try again: Profile → card → **"Resubmit Verification Request"** → confirmation ("Before submitting a new request for account verification, please ensure that you have addressed the issues…") → "yes". For Instagram the app opens the verification screen; tapping "Confirm you did it" resubmits the request immediately. For TikTok the old code screen opens.

### B. Quality review ("Pending Admin Approval")

1. When the influencer saves the last media-kit step (Goals), the account becomes **"Pending Admin Approval"** (`pending`).
2. While pending, the influencer **can** send offers, but each one is stored as a **temporary offer**: the brand does not see it and no chat opens yet.
3. The admin sets the account to:
   - **Active** → the influencer is notified; all temporary offers are turned into real offers (the brand now sees them, chats open, awaiting-payment reminders start) and automatic insight requests are sent;
   - **Temporary suspend** or **Permanent suspend** with reasons → the influencer is notified and **all temporary offers are deleted** (INF-04).
4. There is no "reject" in quality review: the only outcomes are Active or suspended. An admin cannot moderate a **Draft** account (it must finish the media kit first).

## Rules and limits

- Manual verification: maximum **5** rejected requests per account within **7 days** (default settings), then the influencer must wait. A new request is refused while one is pending.
- Pending accounts can make offers (as temporary offers). Draft and suspended accounts cannot.
- Affiliate campaigns need at least one **Active** account (MKT-03).
- Account status changes made directly in the separate admin tool may skip some of the side effects above (temporary offers, notifications) *(to verify with the dev team)*.

## Statuses

| Card label | Internal value | Meaning | Next |
|---|---|---|---|
| Pending Verification | `temporary_integrations.status = pending` | Waiting for manual ownership check | Verified → Draft, or Rejected |
| Verification Rejected | `rejected` | Ownership check failed | Resubmit |
| Draft | `integrations.qc_status = draft` | Media kit not finished | INF-02 |
| Pending Admin Approval | `pending` | Waiting for quality review | Active or suspended |
| Active | `active` | Approved | — |
| Disapproved (suspended) | `temporary suspend` / `permanent suspend` | See INF-04 | Appeal |

## Timers and automatic actions

- Profile data for pending manual requests is filled every 30 min (Instagram only).
- No review deadline; the "up to 24 hours" text is a target, not a timer.
- On activation, converting temporary offers runs in the background (a few minutes).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Manual request verified | Influencer | Email, push, in-app |
| Manual request rejected | Influencer | Email, push |
| Account set Active | Influencer | Email, push, in-app (opens Profile) |
| Account suspended | Influencer | Email, push, in-app (INF-04) |
| Temporary offers converted | Brand | Normal "new offer" notifications (OFR-01) |

## Common questions

### "How long does approval take?"
**Answer for the user:** Our team reviews new accounts, usually within 24 hours. You'll get a push and an email. Meanwhile you can browse campaigns and even send offers; brands will see them as soon as you're approved.
**What to check:** `integrations.qc_status` and `updated_at`; for manual verification, `temporary_integrations.status`.
**Escalate when:** pending for more than 48 hours.

### "I sent offers but the brand says they never got them."
**Answer for the user:** Offers sent while your account is waiting for approval are delivered to brands once the account is approved.
**What to check:** account `qc_status = pending`; rows in temporary offers for that integration.
**Escalate when:** the account is Active and the temporary offers did not appear for the brand within an hour.

### "My verification was rejected. What now?"
**Answer for the user:** The rejection reasons are shown on your Profile. Fix them (for example make the account public), then tap "Resubmit Verification Request".
**What to check:** `rejection_reasons`, `tries`, `last_tried_at`.
**Escalate when:** the user disputes the decision; an admin must review it again.

### "Resubmit says I can't submit for 7 days."
**Answer for the user:** After five rejected requests a new request is blocked for 7 days. Please make sure the rejection reasons are fixed before you try again.

### "My offers disappeared after my account was suspended."
**Answer for the user:** When an account under review is suspended, its pending offers are cancelled. If you think the suspension is a mistake, use "Appeal Suspension" on your Profile (INF-04).

## Known issues

- Editing an Active profile sends the account back to "Pending Admin Approval" (see INF-02 Known issues) *(to verify)*.
- Some Instagram accounts go to manual verification even though the DM was correct (see INF-01 Known issues) *(to verify)*.

## Related topics

- INF-01 connecting, INF-02 media kit, INF-04 suspension and appeal, OFR-01 making offers.

## Technical reference

- Endpoint (influencer): `POST api/{locale}/v3/mobile/temporary-integrations/{id}/resubmit` → `Modules/Mobile/Http/Controllers/API/TemporaryIntegrationController@resubmit` → `Services/Implementation/Mobile/TemporaryIntegrationService::resubmit` (refuses pending; on rejected with remaining attempts resets status to pending, `tries+1`).
- Admin verification: `Services/Implementation/Admin/TemporaryIntegrationService::{verify,reject}` → notifications `Modules/Mobile/Notifications/Account/Temporary/Account{Verified,Rejected}Notification` (events `events:mobile:account:temporary:verified|rejected`); conversion job `TemporaryIntegrationConversion` → `Admin/TemporaryIntegrationConverterService` (creates integration with `qc_status = draft`, `was_temporary = 1`).
- Admin quality review: `Services/Implementation/Admin/IntegrationService::update` + `getScenario` (allowed: pending→active/suspend, active→suspend, suspend→active/other suspend). On active: `showSuspendedOffers`, and from pending: jobs `StoreTemporaryOfferPermanently` → `SendAutomaticInsightRequest`. On suspend with reasons: `addSuspensionDetails`, `deleteTemporaryOffers`. Event `UserQualityControlEvent` → `Listeners/v2/Integration/UserQualityControlListener` → `Account{Activated,TemporarilySuspended,PermanentlySuspended}Notification`.
- Temporary offers: `Modules/Mobile/Services/Implementation/Offer/OfferCreatorService` (if `qc_status === pending` → `temporaryOfferRepository`).
- Data: `temporary_integrations` (status, tries, last_tried_at, rejection_reasons, rejection_description, reviewer_id), `integrations.qc_status`, temporary offers table (via `TemporaryOfferRepository`), `suspension_details`.
- Settings: `maximum_tries` (5), `tries_time_interval` (7), rejection reasons (`manual_verification_rejection_reason`) in `database/seeders/data/production_settings.json`.
