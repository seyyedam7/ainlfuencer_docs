---
id: NTF-02
title: Push notifications and where they open
domain: Notifications and emails
audience: [influencer, system]
apps: [mobile, background]
related: [NTF-01, NTF-03, CHAT-02, OFR-02, OFR-04, OFR-05, PUB-01, PUB-03, DSC-03, MKT-02, MKT-04, INF-03, INF-04, CMP-06]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d
---

# NTF-02 · Push notifications and where they open

> Push notifications go to the influencer mobile app (brands have no push in the web app). This topic explains which events send a push, which screen a tap opens, how links from emails open the app, and the known cases where the app opens the wrong screen.

## Who and where

- **Influencers**, on every phone where they are logged into the Ainfluencer app and allowed notifications.
- A tap opens the app on a specific screen. The same rules apply to taps in the in-app "Notifications" list (NTF-01), with small differences noted below.
- Links in emails (for example campaign invitations) can also open the app directly ("deep links").

## How it works

1. An event happens (for example the brand pays an offer). Ainfluencer sends a push to every phone the influencer is currently logged in on.
2. If the app is open, the push appears as a banner inside the app and the app refreshes its data.
3. Tapping the push opens the screen below. If the app was closed, it starts, then opens that screen with the right tab underneath (so "Back" works).

| Push about | Opens |
|---|---|
| New chat message on an offer, offer updated, rejected, paid, start date changed, changed by an admin, revision requested by the brand, post reminder, post approved by the brand, post disapproved by an admin | That offer's **chat** (Inbox tab underneath) |
| Payment released to the wallet ("cost transferred") | **Wallet** (Profile tab underneath) |
| Post published | **Scheduler**, "Published" tab |
| Post failed | **Scheduler**, "Failed" tab |
| Campaign invitations (the grouped invitation push, twice a day) | **Invitations** list (Home tab underneath) |
| Account activated, suspended (temporary or permanent), under-review account verified or rejected, account check failed, account unarchived | **Profile** (social accounts are reloaded) |
| Review approved, contract/contact violation detected, new messages in **affiliate or gifting chat threads**, and **any other event** (for example gifting updates, insight requests, Shopify shipping updates) | **Explore** |

**Deep links** (from emails or shared links) open: `invitation?ad_id=…` or `view-campaign/<id>` → the campaign details; `affiliate/invitation?ad_id=…` or `view-campaign-affiliate/<id>` → the affiliate campaign; `gifting/invitation?ad_id=…` → the gifting campaign; plus `wallet`, `invitations`, `chat/<offerId>`, `profile`, `scheduler/<tab>`. Links work on `app.ainfluencer.com`, `ainfluencer.app.link` and `ainfluencer://`.

## Rules and limits

- **Push only works while logged in.** After logout (or before the first login) the phone receives nothing, and tapping an old push or a deep link just shows the login screen.
- Pushes go to each phone that has an active login session with a notification token. If the same phone is later used to log into another account, it receives only the newer account's pushes.
- The influencer can turn push off per category in "Notifications & Emails" (the "Push" switch; NTF-03), or for the whole app in the phone's settings.
- **Bulk messages and automatic system messages in chat don't send a push** (CHAT-02).
- Brands don't get push notifications (no brand mobile app is in use); they get email and in-app notifications.
- Every app store update is mandatory (ACC-02); old app versions may handle pushes differently.

## Statuses

Not applicable.

## Timers and automatic actions

- Grouped campaign-invitation pushes are sent twice a day, at 06:00 and 18:00 UTC (DSC-03).
- Other pushes are sent when the event happens (most within a minute, through the background queue).
- "Submit your post" reminders are sent before the delivery deadline (PUB-01).

## Notifications

This topic is the push channel itself. The events are listed in the table above; details about each event are in its own topic.

## Common questions

### "I don't get any push notifications."
**Answer for the user:** Please check that (1) you're logged into the app, (2) notifications for Ainfluencer are allowed in your phone settings, and (3) the "Push" switches are on in the app under Profile → settings → "Notifications & Emails". Then log out and log in again so your phone is registered again.
**What to check:** The user has a recent session with a notification token (`tokens.firebase_token`, recent `updated_at`); category settings (`notification_settings.receive_firebase`).
**Escalate when:** All of the above are fine and pushes still don't arrive while in-app notifications do.

### "I tapped a notification and it opened Explore instead of the right page."
**Answer for the user:** Some notifications (for example gifting updates and reviews) don't have their own page in the app yet, so they open Explore. You can find the details in your Inbox, Gifting tab or Profile.
**What to check:** The event type of the notification (see table).
**Escalate when:** A chat or offer notification opens Explore although it belongs to an offer.

### "The notification says my post failed but the app opened the chat."
**Answer for the user:** That's how the app handles "disapproved by admin" notifications today. You'll find the post under Scheduler → "Failed".
**What to check:** Nothing.
**Escalate when:** Never (known issue).

### "I clicked the invitation link in the email and only see the login screen."
**Answer for the user:** Please log into the app first, then tap the link in the email again. The invitation will open directly.
**What to check:** The user is logged in on that phone.
**Escalate when:** The link still doesn't open the campaign after login.

### "I logged out but still got a push."
**Answer for the user:** That can happen if you're logged in on another phone or tablet. Pushes go to every device where you're logged in.
**What to check:** Several active sessions in `tokens`.
**Escalate when:** The user has no other device.

### "I don't get notified about new chat messages."
**Answer for the user:** Messages sent in bulk by a brand, and automatic system messages, don't send a push. Regular messages do if "Push" is on for chat messages in "Notifications & Emails".
**What to check:** Message type; the "Chat Messages" category setting.
**Escalate when:** Regular one-to-one messages never notify.

## Known issues

- **Unmapped events open Explore**: new messages in affiliate and gifting chat threads, gifting updates, insight requests, Shopify shipping updates, "approved by admin" (new event name), email-changed and similar notifications have no target screen.
- **"Disapproved by admin" opens the chat**, not the Scheduler "Failed" tab.
- **In-app list taps on publishing notifications always open the "Scheduled" tab** (the push itself opens the right tab).
- **Deep links don't work when logged out**: the user lands on login and must tap the link again after logging in.
- A push about a chat message without an offer id opens an empty chat screen *(to verify)*.

## Related topics

- NTF-01: the in-app notification list.
- NTF-03: turning push on or off per category.
- CHAT-02: unread chat counts and message pushes.
- OFR-02, OFR-04, OFR-05: offer events.
- PUB-01, PUB-03: posting, reminders and payment release.
- DSC-03, MKT-02: invitations.
- MKT-04, CMP-06: gifting and shipping.
- INF-03, INF-04: account review and suspension events.

## Technical reference

- **Sending**: notifications with an `fcm` channel call `toFcm()` → `app/Services/Implementation/FcmService.php` (`sendToFcmWithData`; device list from `getDeviceTokens`: every `tokens.firebase_token` of the user, newest session wins when a device token is shared). Payload `data.event` and `data.offer_id`.
- **Token registration**: `app.firebase_token` sent at login/register/refresh; `POST /api/en/v3/mobile/firebase-token` → `AuthService::updateFirebaseToken`. Mobile logout deletes the device token locally only.
- **Mobile routing**: `src/utils/notificationNavigation.ts` (`handleNotificationPress`, cold start `resetFromColdStartNotification`), `src/features/notification/utils/redirectTo.ts` (event → screen), `src/utils/handleDeeplink.ts` (`openFailedTabIfContains`), `src/hooks/useNotification.ts` (mounted only when logged in), `src/lib/registerNotificationHandlers.ts` (background taps), event names in `src/types/enums.ts` (`EventTypes`). Deep link config: `src/navigation/Container.tsx`.
- **Event names (examples)**: `events:mobile:offer:paid`, `events:mobile:offer:message-received`, `events:mobile:publish-job:cost-transferred`, `events:mobile:publish-job:failed`, `events:mobile:invitation:internal:sent`, `events:mobile:account:temporarily-suspended`, `events:chat:message-received` (chat service, not mapped), `events:gifting:*` (not mapped), `events:mobile:offer:insight-requested` (not mapped).
