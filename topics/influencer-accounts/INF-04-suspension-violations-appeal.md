---
id: INF-04
title: Account suspension, violations and appeal
domain: Influencer social accounts
audience: [influencer, admin, system]
apps: [mobile, email, background]
related: [INF-03, INF-05, INF-06, CHAT-04, OFR-01, BRD-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# INF-04 · Account suspension, violations and appeal

> An Ainfluencer admin can suspend an influencer's social account (temporarily or permanently), for example for low quality or for breaking the rules such as asking brands to talk or pay outside the platform. A suspended account can't make offers or chat. The influencer can send one appeal per suspension from the app. Separately, accounts can be **archived** when the username no longer exists, and admins can send **violation warnings**.

## Who and where

- **Admin** suspends, reactivates, reviews appeals and sends violation warnings in the admin panel.
- **Influencer** (mobile): the suspended account card on **Profile** shows "Disapproved" and an **"Appeal Suspension"** button; an appeal pop-up also opens from **Explore** for suspended accounts that haven't appealed yet.

## How it works

### Suspension
1. An admin changes the account to **Temporary suspend** or **Permanent suspend** and picks reasons (for example "Low-Quality Score, or Engagement Rate", "Low Followers", "Low Publishing Activities", "High Spamming Rate", "Violation of Terms – Asking to chat or pay outside of the platform…").
2. The influencer gets an email, push and in-app notification. The push opens Profile.
3. Effects:
   - the account can't make offers ("Your account for Instagram ID @username is temporarily suspended for account quality or qualification issues…"), and the app shows "account suspended" on campaign details;
   - the chat message box is hidden for chats with this account;
   - pending **temporary offers** (made while under review) are deleted;
   - after **48 hours** of suspension, the nightly job hides the account's existing offers from brands;
   - profile edits (name, category, picture, "Edit Profile") are disabled in the app.
4. "Temporary" has **no end date**: the account stays suspended until an admin reactivates it.

### Appeal
1. Profile → **"Appeal Suspension"** opens a 4-step pop-up: "Account Suspended!" with "Reason(s) for Suspension:" → "Reason for appeal:" (text required: "Please provide a reason for your appeal") → "Appeal: Dont's to Follow" ("I will never ask brands: …") → "Dos & Don'ts to Watch" video, "Confirm Watched", then **"Submit Appeal"**.
2. The button then reads **"Appeal Pending"**; tapping it shows "Your appeal request is currently under review…".
3. The admin approves or disapproves:
   - **Approved** → the account becomes **Active** again (activation notification), and hidden offers are shown to brands again.
   - **Disapproved** → the card reads "Your appeal was denied. Your account remains suspended." No new appeal is possible for this suspension ("Your appeal request has been reviewed and rejected. No further appeals can be submitted at this time."). There is no notification for a denied appeal; the influencer sees it on the card.

### Violation warnings
- An admin can send a violation warning about an offer (for example sharing contact details, see CHAT-04). The influencer gets email, push and in-app; the push opens Explore. A warning alone does not suspend the account.

### Archived accounts
- An account is **archived** when its username can't be found anymore: during the monthly Instagram refresh (INF-06) or when the influencer taps Sync and the profile no longer exists. The card moves to the **Archived** tab: "This username is archived because it was changed or is no longer available. Please sync your account to refresh the status." Sync on an archived account answers: "We apologize for the inconvenience, but had to archive your account per company guidelines. Please email our support team…". Only an admin can restore it; the influencer gets an "unarchived" email and push.

## Rules and limits

- One appeal per suspension. A new suspension allows a new appeal.
- Suspended or archived accounts can't make offers, and the chat input is hidden.
- **Automatic suspension rules are not running.** The Instagram rule set (followers, ratio, quality score, spam rate…) is only wired to an old endpoint that the app no longer uses, and the periodic re-check is switched off. In practice every suspension is done by an admin.

## Statuses

| Card / button | Internal | Meaning |
|---|---|---|
| Disapproved + "Appeal Suspension" | `qc_status` = `temporary suspend` / `permanent suspend`, no appeal | Suspended, can appeal |
| "Appeal Pending" | `suspension_appeals.status = pending` | Appeal waiting for admin |
| "Your appeal was denied…" | `disapproved` | Final for this suspension |
| Active | `approved` appeal → `qc_status = active` | Restored |
| Archived tab | `integrations.archived_at` set | Username not found |

## Timers and automatic actions

- **Daily 00:00 UTC**: offers to accounts suspended for 48 hours or more are hidden from brands (`offers:hide-suspended-offers`). Reactivation shows them again.
- Monthly Instagram refresh can archive accounts whose username no longer exists (INF-06).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Suspended (temporary/permanent) | Influencer | Email, push, in-app (opens Profile) |
| Reactivated / appeal approved | Influencer | Email, push, in-app |
| Appeal denied | Influencer | None (card text only) |
| Violation warning | Influencer | Email, push, in-app (opens Explore) |
| Unarchived by admin | Influencer | Email, push, in-app |

## Common questions

### "Why was my account suspended?"
**Answer for the user:** The reasons are shown when you tap "Appeal Suspension" on your Profile and in the email we sent. If you believe it's a mistake, submit an appeal from there.
**What to check:** latest `suspension_details` (reason_ids, other_reason_note) for the integration.
**Escalate when:** no reasons are recorded.

### "How long does a temporary suspension last?"
**Answer for the user:** It lasts until our team reviews your account again. The fastest way is to submit an appeal.

### "I submitted an appeal. When will I hear back?"
**Answer for the user:** Your appeal is in our team's queue. If it's approved your account becomes active again and you'll be notified.
**Escalate when:** the appeal has been pending for more than 3 business days.

### "My appeal was denied. Can I appeal again?"
**Answer for the user:** Only one appeal is allowed per suspension, and this one was reviewed. Please follow our community guidelines; if circumstances change, contact support.
**Escalate when:** the user brings new evidence; an admin decides.

### "Brands say they can't see my offers anymore."
**Answer for the user:** Offers from accounts suspended for more than 48 hours are hidden from brands. They reappear if the account is reactivated.

### "My account moved to Archived."
**Answer for the user:** We couldn't find this username on the platform anymore (it may have been renamed or deactivated). Please email support with your current username so we can restore it.
**What to check:** `archived_at`, whether the username changed.

### "I can't type in the chat."
**Answer for the user:** Messaging is disabled while your account (or the brand's) is suspended.

## Known issues

- Denied appeals send no notification.
- In the Scheduler, the suspension alert's "Contact Support" button does nothing; use Settings → "Get Help".
- Suspensions made directly in the separate admin tool may skip side effects such as notifications or deleting temporary offers *(to verify with the dev team)*.

## Related topics

- INF-03 review outcomes, INF-05 managing accounts, INF-06 data refresh and archiving, CHAT-04 contact-sharing detection, BRD-04 brand suspension.

## Technical reference

- Endpoints: `GET api/v2/settings/influencer/suspension/appeal`, `GET api/v2/settings/influencer/suspension/reasons` → `Http/Controllers/v2/Common/SettingController`; `POST api/v2/influencer/suspension-appeal` `{integration_id, description}` → `v2/Frontend/SuspensionAppealController@storeInfluencerAppeal` → `Services/Implementation/SuspensionAppealService` (one appeal per suspension detail).
- Admin: `Services/Implementation/Admin/IntegrationService::update` (suspend/activate, `addSuspensionDetails`, `deleteTemporaryOffers`, `showSuspendedOffers`), `Admin/SuspensionAppealService::updateInfluencerAppeal` (approved → `qc_status = active`), violations `Admin/ViolationService` → `ViolationStoreEvent` → `Listeners/v2/NewViolationListener` → `Modules/Mobile/Notifications/Account/ViolationNotification`.
- Job: `offers:hide-suspended-offers` (`Console/Commands/Offer/HideSuspendedOffers.php`, daily) sets `offers.extra.influencer_suspended`.
- Automatic rules: `Modules/AutomaticSuspension/Instagram/*` used only by legacy `Http/Controllers/IntegrationController@store` (`POST api/integrations`, unused); checkers `IntegrationSuspensionChecker` / `IntegrationRestorationChecker` are commented out in `Console/Kernel.php`.
- Archiving: `Services/Implementation/Influencer/InfluencerService::syncInformation` → `archiveIntegration`; Crawlzo finalize (INF-06). Restore: `Admin/IntegrationService::restore` → `InfluencerUnarchived` mail + `AccountUnarchivedNotification`.
- Data: `integrations.qc_status`, `integrations.archived_at`, `suspension_details` (reason_ids, other_reason_note), `suspension_appeals` (status pending/approved/disapproved, appealable_type/id), `violations`, `offers.extra->influencer_suspended`.
- Mobile: `commonComponents/ApealModal/*`, `features/profile/components/Accounts/ActionButton/index.tsx`.
