---
id: NTF-03
title: Email and notification preferences
domain: Notifications and emails
audience: [brand, influencer]
apps: [web, mobile, email]
related: [NTF-01, NTF-02, NTF-04, BRD-03, ACC-01, ACC-05]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# NTF-03 · Email and notification preferences

> Users can switch groups ("categories") of notifications on or off per channel. There is also an "Unsubscribe" link in emails that stops most emails at once. Notifications that aren't linked to any category can't be switched off.

## Who and where

- **Brands, web**: profile menu → "Notifications & Emails" (also the "Emails & Notifications" tab on the profile pages) → `/profile/email-notifications`, titled "Manage Your Email and Notification Preferences". Team members working inside another brand's team can't open it.
- **Influencers, mobile**: Profile → gear → "Account Settings" → "Notifications & Emails" (title "Manage Your Email And Notification Preferences").
- **Everyone, email**: the "Unsubscribe" button at the bottom of most emails.

## How it works

**Web (brands)**
1. The page lists 7 categories, each with three switches: **"Email"**, **"Push"** and **"SMS"**:
   - "Account Notifications" ("Always Enabled", can't be changed)
   - "Campaign Updates"
   - "Offer Notifications" ("Suggested")
   - "Deal Completion"
   - "Payment Notifications"
   - "Admin Actions or Alerts" ("Suggested")
   - "Marketing Updates"
2. The brand changes switches and clicks **"Save Preferences"**. If everything except Account is turned off, a confirmation appears ("Stay Informed with Notifications!" → "Confirm Mute" or "Cancel").
3. On the web, the **"Push" switch controls the in-app bell notifications** (NTF-01); brands have no phone push.

**Mobile (influencers)**
1. The screen lists the categories from the server, each with **"Email"** and **"Push"** switches (no SMS). Account Notifications are always on. Influencer categories include "Campaign Invitations", "Offer Notifications", "Post Publishing & Deal Updates", "Payment & Wallet Notifications", "Admin Requests & Actions", "Marketing Updates", and newer ones such as "Gifting Campaigns" and "Chat Messages".
2. The "Push" switch controls **both** phone push (NTF-02) and the in-app list (NTF-01).
3. Tap **"Save Preferences"**.

**Unsubscribe link in emails**
- Clicking it shows "You've been unsubscribed!" ("We've stopped sending you emails, just as you asked..."). From then on:
  - most emails stop (essential emails such as verification codes, password resets, security alerts, suspension and some account emails still arrive);
  - for categories the user had already saved, in-app, push, email and SMS are all switched off (except Account).
- In campaign-invitation emails sent to people **without** an account, the unsubscribe link removes their pending external invitations instead.

## Rules and limits

- New users start with everything on.
- **Only notifications linked to a category can be switched off.** Anything not linked is always sent ("fail open"). Which notifications are linked is set up in the database by the team, so a switch may not cover every email of that topic *(to verify for a specific email before promising it will stop)*.
- "Account Notifications" can never be switched off.
- Settings are per person. Team members manage their own settings while in their own brand (BRD-03).
- Some categories exist only on mobile or aren't shown on the web page: brands' "Gifting Campaigns", "Chat Messages" and "Daily Digest" categories are not displayed on the web, so brands can't change them there; they stay as they were (on by default).
- **Turning switches back on doesn't undo "Unsubscribe".** After unsubscribing from emails, emails stay off until staff re-subscribe the user.

## Statuses

| What support sees | Meaning |
|---|---|
| No `notification_settings` row for a category | Everything on (defaults) |
| `notification_settings.receive_email / receive_in_app / receive_firebase / receive_sms` = 0 | That channel is off for that category |
| `users.unsubscribe_at` set | Unsubscribed from emails (link in email, or account deleted) |

## Timers and automatic actions

None. Changes apply to notifications created after saving.

## Notifications

None for changing settings.

## Common questions

### "How do I stop marketing or digest emails?"
**Answer for the user:** Open "Notifications & Emails" (web: profile menu; app: Profile → settings) and turn off "Email" for "Marketing Updates", then click "Save Preferences". You can also click "Unsubscribe" at the bottom of the email to stop most emails at once.
**What to check:** In the original configuration, weekly digest emails are linked to the marketing category *(to verify in production)*; the brand daily digest has its own "Daily Digest" category that isn't shown on the web page (NTF-04).
**Escalate when:** A brand wants to stop the daily digest (the switch isn't on the web page) or emails continue after switching off.

### "I turned notifications off but I still get them."
**Answer for the user:** Some essential notifications can't be turned off (for example account, security and verification emails). For others, please make sure you clicked "Save Preferences".
**What to check:** Saved rows in `notification_settings`; whether the specific notification is linked to the category (some aren't and are always sent).
**Escalate when:** The notification belongs to a switched-off category and still arrives (configuration issue for dev).

### "I unsubscribed by mistake and now I get no emails."
**Answer for the user:** We'll turn your emails back on. Turning the switches on in settings isn't enough after using the Unsubscribe link.
**What to check:** `users.unsubscribe_at` is set.
**Escalate when:** Always: staff must clear the unsubscribe and the user should then review the switches.

### "I don't get notifications in the bell anymore." (brand)
**Answer for the user:** On the website, the "Push" switch controls the notification bell. Please turn "Push" back on for the categories you want and save.
**What to check:** `receive_in_app` for the brand's categories; `unsubscribe_at`.
**Escalate when:** Settings are on and the bell stays empty for new events.

### "Where do I turn off SMS?" (brand)
**Answer for the user:** Use the "SMS" switches on the "Notifications & Emails" page and save.
**What to check:** SMS is used only for a few brand messages (for example abandoned checkout reminders).
**Escalate when:** SMS continue after switching off.

### "My teammate can't open the notification settings."
**Answer for the user:** Settings are personal. Your teammate should switch to their own brand in the team switcher and open "Notifications & Emails" there.
**What to check:** BRD-03.
**Escalate when:** Never.

## Known issues

- **"Unsubscribe" can't be undone by the user.** Escalate to staff.
- **Web "Push" means in-app (bell), not phone push.** Explain this if brands are confused.
- **Brands can't see or change "Gifting Campaigns", "Chat Messages" or "Daily Digest" on the web.** Escalate to dev if a brand needs one turned off.
- Which emails each switch really stops depends on database configuration that support can't see; don't promise an email will stop unless verified.

## Related topics

- NTF-01: in-app notifications (bell).
- NTF-02: push notifications.
- NTF-04: digest and reminder emails.
- BRD-03: team members and personal settings.
- ACC-01: verification emails (always sent).
- ACC-05: deleting an account also unsubscribes it.

## Technical reference

- **Endpoints**: `GET /api/v2/notify-settings/categories` (scope `brand`/`influencer` derived from the user) and `PUT /api/v2/notify-settings/categories` (`categories[].category_id`, `settings.in_app|email|sms|firebase`) → `Modules/NotificationSetting/Http/Controllers/API/NotificationCategoryController` → `Services/Implementation/NotificationCategoryService` (non-switchable channels are forced to the category default).
- **Sending check**: `Notifications/Core/QueueableNotification::shouldSend` and `Mail/Core/BaseMailable::shouldSendByEmail` → `NotificationSettingRepository::isChannelEnabledFor` (no matching setting row → send). `BaseNotification::via` sends only database + fcm when `users.unsubscribe_at` is set. Mailables with `$shouldNotUnsubscribe = true` (verification, password reset, suspension, violation, etc.) ignore unsubscribe.
- **Unsubscribe**: `GET /emails/unsubscribe?subscription=…` → `EmailsUnsubscribeController` → `Services/Implementation/EmailsUnsubscribeService` (sets `unsubscribe_at`, switches off saved non-account categories); `GET /invitations/unsubscribe` (external invitees).
- **Data**: `notification_categories` (`slug`, `scope`, `is_*_switchable`, `default_*`), `notification_items` (`class_fqn`), `notification_category_item`, `notification_settings` (`user_id`, `category_id`, `receive_in_app`, `receive_email`, `receive_firebase`, `receive_sms`), `users.unsubscribe_at`. Category registration: `Modules/NotificationSetting/Support/NotificationCatalog.php`; migrations `2026_09_16_…_register_gifting_and_chat_notifications`, `2026_09_21_…_register_gifting_moderation_notifications`, `2026_09_29_…_register_brand_daily_digest_notification`, `2025_02_02_…_add_login_security_notification_to_account_category`.
- **Web**: `src/components/pages/emailNotificationSettings` (`notificationSettingsModel.ts`: "push" ↔ `in_app`), strings `locales/en/emailNotifications.json`. **Mobile**: `src/features/profile/screens/NotificationsEmails.tsx`, `hooks/useEmailsAndNotifications.ts` (push → `in_app` + `firebase`).
