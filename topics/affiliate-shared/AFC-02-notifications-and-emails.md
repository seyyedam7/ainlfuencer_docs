---
id: AFC-02
title: Notifications and emails
domain: Shared by seller and affiliate apps
audience: [seller, affiliate, system]
apps: [seller-app, affiliate-app, email, background]
related: [AFS-06, AFS-10, AFS-12, AFS-13, AFA-04, AFA-08, AFC-01, AFC-03, AFC-04]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFC-02 · Notifications and emails

> The affiliate platform tells sellers and affiliates about events (partnership requests and decisions, product and commission changes, payments, tickets…) through the bell in the app and by email. In-app notifications arrive at once; **most emails are collected and sent twice a day, at 06:00 and 18:00 UTC**. Users choose which ones they get in a settings matrix.

## Who and where
- **Seller and affiliate apps:**
  - Header **bell** opens a side drawer with search suggestions/filters, bookmarks, "mark as read" per item and "Mark all as read".
  - **Browser push**: after login the browser asks for permission; if allowed, the device is registered for push.
  - **Account settings → Notifications** ("Manage Notifications"): a matrix with "In-App" and "Email" switches per notification type, grouped as "Messages", "Brands", "Products", "General" (affiliates also "Partnership Contract").
- **Email:** sent to the account's primary email; queued emails include an unsubscribe link.

## How it works
1. Something happens (for example a partnership request is approved).
2. The **in-app notification** is created immediately (if the user's "In-App" switch for that type is on) and appears in the bell; browser push may show it too.
3. If the type also has an email and the user's "Email" switch is on, the email is **put in a queue**. Several events of the same type for the same user are **merged into one email**.
4. At **06:00 and 18:00 UTC** the system sends all queued emails.
5. Some emails don't use the queue and go out on their own schedule or immediately (see Timers and the table below).
6. Clicking the **unsubscribe link** in a queued email switches off the "Email" switch for that notification type.

## Rules and limits
- **"Important Notifications" can't be switched off** (locked). Payment notifications use this type.
- **Payment notifications are in-app only** (no email): commission paid/received, last payment attempt, payment failed (AFS-10, AFA-08).
- Unread affiliate-chat summary email: weekly (the chat itself is not supported, AFC-01).
- Statistics emails respect the "Seller Statistics" / "Affiliate Statistics" email switch.
- Notification settings by type (as shown in the app):
  - Seller: New Messages, Unread Chat Messages; Brand Partnership Requests / Accepted / Rejected, Brand Amazon Link' status Changed; Product Partnership Requests / Accepted / Rejected; Promotions, Events & Updates, Important Notifications (locked), Social Media' status Changed, Seller Promotion' status changed, Seller Statistics.
  - Affiliate: same message and partnership types, plus Brand Reactive or De active, Brand Terms File Update, Product Reactive or De active, Product Commission Change, Affiliate Statistics, Partnership Contract Revoked.
- Several notification types exist in the code but are **never sent** (daily activity digests, some old approve/reject notices).
- Admin bulk emails/SMS/notifications to users are stored but **not sent automatically**.

## Timers and automatic actions
| When (UTC) | What |
|---|---|
| Immediately | In-app notifications; team invitation emails; email verification / password reset emails. |
| 06:00 and 18:00 daily | Queued emails sent (partnership status changes, commission changes, contract revoked, product/brand deactivated, ticket replies, unread chat summary, etc.). |
| 08:00 daily | "New partnership requests" email to each user with pending requests received (if enabled). |
| Monday 00:00 | Weekly statistics email to sellers (previous Monday–Sunday). |
| Monday 01:00 | Weekly statistics email to affiliates. |
| Last day of month 00:00 / 01:00 | Monthly statistics email to sellers / affiliates (meant to cover the previous month). |
| Monday 00:00 | Weekly unread affiliate-chat summary (queued, so it goes out at 06:00). |

## Notifications
Main ones support will be asked about:

| Trigger | Who | Channel |
|---|---|---|
| Partnership request approved / rejected / cancelled | The other party | In-app + email (queued) |
| Pending partnership requests | Receiver | Email, daily 08:00 |
| Product commission changed | Partnered affiliates | In-app + email (queued) |
| Product or brand status changed / deactivated | Partnered affiliates | In-app / email (queued) |
| Brand terms file updated | Partnered affiliates | Email (queued) |
| Contract revoked by plan downgrade | Seller and affiliate | Email (queued) |
| Admin decision on promotion, brand Amazon link, social account | Seller / affiliate | In-app + email (queued) |
| Store account synced / Amazon profile syncing | Seller | Email / in-app |
| Commission paid, last attempt, failed | Seller / affiliate | **In-app only** |
| Stripe Connect account status changed | Affiliate | Email (queued) |
| Admin replied to a ticket | Ticket owner | In-app + email (queued) (AFC-03) |
| Team invitation | Invitee | Email, immediate (AFS-13) |
| Custom plan created | Seller/affiliate | Email |
| Weekly / monthly statistics | All real sellers / affiliates with a primary email | Email |

## Common questions

### "I got the notification in the app but no email."
**Answer for the user:** Most emails are sent in two batches a day, at 06:00 and 18:00 UTC, so it may still be on its way. Some notifications, like payment updates, are in-app only.
**What to check:** `user_type_notification_queues` for the user: `notification_class`, `status` (pending/processed), `sent_at`; the user's Email switch for that type.
**Escalate when:** a row is `processed` with `sent_at` set but the user never received it (check spam first).

### "How do I stop these emails?"
**Answer for the user:** Go to Account settings → Notifications and switch off "Email" for the types you don't want, or use the unsubscribe link in the email. "Important Notifications" can't be turned off.
**What to check:** `user_notification_settings` for the user.
**Escalate when:** emails keep coming for a type that is switched off.

### "I clicked unsubscribe and got a strange page."
**Answer for the user:** The unsubscribe worked even if the page looks plain; you can confirm in Account settings → Notifications.
**What to check:** that type's `email` = false in `user_notification_settings`.
**Escalate when:** the setting didn't change.

### "Why didn't I get an email when the commission was paid / my payment failed?"
**Answer for the user:** Payment updates are shown in the app's notification bell only. Please check the bell and More → Payments.
**What to check:** in-app notifications list.
**Escalate when:** no in-app notification either (AFS-10).

### "I don't get the weekly statistics email."
**Answer for the user:** It's sent on Mondays to accounts with "Seller Statistics" / "Affiliate Statistics" email switched on.
**What to check:** the statistics setting; the user isn't marked as imported/fake; has a primary email.
**Escalate when:** all fine and still nothing after two Mondays.

### "Browser notifications don't show."
**Answer for the user:** Please allow notifications for seller.ainfluencer.com / affiliate.ainfluencer.com in your browser settings and log in again.
**What to check:** a registered device token for the user (`user_fcm_tokens`).
**Escalate when:** permission is allowed, token exists, and nothing arrives.

## Known issues
- Emails are delayed up to ~12 hours by the twice-daily batching.
- Payment events never produce an email.
- Activity digest notifications and some old notification types are never sent.
- The daily "connect Stripe / add a card" reminder most likely never sends (configuration missing).
- Unsubscribe link shows a plain technical response instead of a page *(to verify)*.
- Monthly statistics period may be wrong in some months (date calculation runs on the last day of the month) *(to verify)*.
- Admin bulk messages and seller email campaigns are not sent automatically (AFS-08).

## Related topics
- AFS-06 / AFA-04 — partnership requests and decisions.
- AFS-10 / AFA-08 — payment notifications (in-app only).
- AFS-12 — downgrade clean-up notifications.
- AFS-13 — team invitation emails.
- AFC-01 — affiliate chat (not supported; weekly unread email).
- AFC-03 — ticket reply notifications.
- AFC-04 — other settings.

## Technical reference
- **Endpoints:** `{seller|affiliate}/v1/notifications` (list), `/suggests`, PATCH `/{id}/read`, PATCH `/read-all`, PATCH `/{id}/toggle-bookmark`; `{seller|affiliate}/v1/settings/notifications` (GET) and `/update` (PUT); POST `{seller|affiliate}/v1/user-fcm-tokens`; GET `client/v1/callback/email/unsubscribe/{token}` → `UnsubscribeController` (sets `email=false` for that setting).
- **Queue:** `app/Services/General/Notification/NotificationQueue.php` (`create`: web notification immediately via `WebNotificationManager`, email queued per user + class, merged), processor `notification:process-queue` (`ProcessNotificationQueueCommand`) at 06:00 and 18:00 UTC.
- **Other jobs:** `email:new-partnership-requests` 08:00; `statistics:send-seller weekly|monthly`, `statistics:send-affiliate weekly|monthly` (`SendSellerStatisticsCommand`, `SendAffiliateStatisticsCommand`); `chat:process-unread-messages` Monday 00:00; `aggregate:notification` hourly (no-op); `payment:send-stripe-connect-nudges` 10:00 (likely no-op).
- **Data:** `user_type_notification_queues` (`user_type_id`, `notification_class`, `data`, `status` pending/processed/cancelled/failed, `sent_at`, `unsubscribe_token`), `notifications` (in-app), `notification_setting_groups`, `notification_settings` (`key`, `is_locked`), `user_notification_settings` (`in_app`, `email`), `user_fcm_tokens`.
- **Settings seed:** `database/seeders/NotificationSettingTableSeeder.php`. Keys in `app/Enums/Notification/NotificationSettingEnum.php`.
- **UI:** bell `shared-components/store/notification.store.js`; settings `shared-components/algo-setting/tabs/Notification.vue`.
- **Ainfluencer brand digest:** core's daily brand digest pulls affiliate activity from `admin/v1/integrations/ainfluencer/digests*` (see NTF topics).
