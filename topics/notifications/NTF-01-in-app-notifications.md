---
id: NTF-01
title: In-app notifications
domain: Notifications and emails
audience: [brand, influencer]
apps: [web, mobile]
related: [NTF-02, NTF-03, NTF-04, BRD-03, CHAT-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# NTF-01 · In-app notifications

> The notification bell in both apps: a list of events (offers, campaigns, deliveries, payments, account changes) that the user can open, mark as read and (on the web) delete. **Notifications older than 30 days are deleted automatically every day.**

## Who and where

- **Brands, web**: the bell in the header opens a pop-up "Notifications" with the latest **unread** items, "Mark all as read", "Delete", "Delete All" and "See All ({n} Unread)". "See All" opens `/notifications`, the full list.
- **Influencers, mobile**: the bell at the top of the main tabs (and the wallet) opens the "Notifications" screen (NOTIFICATION_SCREEN). A dot on the bell means there are unread items.
- Chat messages have their own unread badges (CHAT-02); they are not part of this bell, though some message events may also appear here.

## How it works

**Web**
1. The header loads the 10 most recent unread notifications when a page opens and each time the bell is clicked. The red badge shows the unread count ("10+" above 10). It doesn't refresh by itself while the page stays open.
2. Clicking an item marks it read and opens the related page:
   - campaign events → `/campaigns`;
   - offer events → the inbox chat for that offer `/new-offersV2/{id}` (or the old `/offers` list if there is no offer id);
   - publishing events → `/scheduler`;
   - anything else → stays on the current page.
3. "Mark all as read" marks every notification read. "Delete" removes one. **"Delete All" removes every notification of the account, read or unread**, not only those shown.
4. `/notifications` lists all notifications (35 at a time, more on scroll). Unread items are highlighted with a blue left border. Clicking marks read and opens the related page; the "x" deletes.

**Mobile**
1. The "Notifications" screen loads 20 at a time (more on scroll). Pull down to refresh.
2. Items are marked read automatically **when they appear on screen** (about 1 second later). There is no "mark all as read" or delete in the app.
3. Tapping an item opens the related screen: offer and chat events → the offer chat; payment released → Wallet; published or failed posts → Scheduler (always on the "Scheduled" tab, see known issues); invitations → the invitations list; account status changes → Profile; anything else → Explore. Details in NTF-02.

## Rules and limits

- **Retention: 30 days.** Every day at 14:00 UTC, notifications older than 30 days are removed for everyone, read or not.
- Notifications are personal. A **team member working inside another brand's team sees that brand's notifications** (and marking them read or deleting them affects the brand's list) (BRD-03).
- Whether a notification is created in the bell at all depends on the user's settings: turning off "in-app" for a category (on the web the switch is labelled "Push"; see NTF-03) stops new bell notifications of that category. Notifications that aren't linked to a category are always created.
- Deleted or archived accounts receive no notifications.
- Deleting is permanent; deleted notifications can't be restored.

## Statuses

| What the user sees | Meaning (`notifications.read_at`) |
|---|---|
| Highlighted / counted in the badge | Unread (`read_at` empty) |
| Normal | Read |

## Timers and automatic actions

- Daily 14:00 UTC: `notifications:clear` keeps only the last 30 days.

## Notifications

This topic is the in-app channel itself. What triggers each notification is described in the topic of the event (offers, deliveries, payments, invitations, accounts). Which ones can be turned off: NTF-03.

## Common questions

### "My old notifications disappeared."
**Answer for the user:** Notifications are kept for 30 days and then removed automatically. Your offers, chats and payments are not affected; you can still find them in the inbox, scheduler and wallet.
**What to check:** Age of the missing items (more than 30 days → expected).
**Escalate when:** Notifications younger than 30 days disappear and the user didn't use "Delete All".

### "The bell number doesn't go away."
**Answer for the user:** Open the bell and click "Mark all as read", or refresh the page. The counter only updates when the page loads or when you open the bell.
**What to check:** Count of unread notifications for the user.
**Escalate when:** "Mark all as read" succeeds but the count stays after a refresh.

### "I clicked a notification and nothing happened."
**Answer for the user:** Some notifications are just information and don't open a page. You can find the details in your Campaigns, Inbox or Scheduler.
**What to check:** The notification type (only campaign, offer and publishing notifications have a link on the web).
**Escalate when:** An offer notification doesn't open the chat.

### "I'm not getting any notifications in the bell."
**Answer for the user:** Please check your notification settings (web: profile menu → "Notifications & Emails"; app: Profile → gear → "Notifications & Emails") and make sure the categories you want are turned on.
**What to check:** The user's saved settings (`notification_settings`); whether the user unsubscribed from all emails (that also turns off in-app for most categories, see NTF-03).
**Escalate when:** Settings are on and events still produce no notification.

### "I deleted all my notifications by mistake."
**Answer for the user:** Deleted notifications can't be restored, but all the underlying information is still in your inbox, campaigns, scheduler and wallet.
**What to check:** Nothing.
**Escalate when:** Never.

### "Why do I see another brand's notifications?" (team member)
**Answer for the user:** You're currently working inside that brand's team, so you see its notifications. Switch back to your own brand with the team switcher to see yours.
**What to check:** The user's current team (BRD-03).
**Escalate when:** Never.

## Known issues

- **Mobile: publishing notifications always open the "Scheduled" tab**, even for published or failed posts. The user must switch tabs manually.
- **Mobile: unknown or unmapped notifications open Explore** instead of a related screen (for example gifting and chat-message notifications).
- **Web: the bell doesn't update live**; new notifications appear only after a page load or a click on the bell.
- **"Delete All" deletes everything**, not just the unread items shown in the pop-up.

## Related topics

- NTF-02: push notifications and which screen they open.
- NTF-03: turning categories on or off.
- NTF-04: digest and reminder emails (not in the bell).
- CHAT-02: unread chat badges.
- BRD-03: team members see the team's notifications.

## Technical reference

- **Endpoints**
  - Web: `GET /api/v2/notifications?page=&limit=&state=unread|read`, `POST /api/v2/notifications/read` (`ids[]`), `PATCH /api/v2/notifications/read-all`, `DELETE /api/v2/notifications/{id}`, `DELETE /api/v2/notifications/delete/all` → `v2/Frontend/NotificationController` → `Services/Implementation/NotificationService` (owner = authenticated user; in team context the team owner).
  - Mobile: `GET /api/en/v3/mobile/notifications`, `POST …/notifications/read` → `Mobile/.../NotificationController` (same service). `POST …/notifications/read/all` exists but the app doesn't call it.
- **Data**: `notifications` (Laravel database notifications: `id` uuid, `type` = class, `notifiable_id`, `data` JSON with `event`, `message`, `id`/`model`, `offer_id`, `read_at`, `created_at`).
- **Job**: `notifications:clear` → `app/Console/Commands/Notifications/DropNotificationTableCommand.php` (daily 14:00 UTC; rebuilds the table with the last 30 days).
- **Web**: header `src/components/layouts/dashboard/header/parts/notification` (`model.ts` `getRedirectUrl`), page `src/components/pages/notifications`. Strings `locales/en/header.json` (`notification`).
- **Mobile**: `src/features/notification` (`NotificationScreen.tsx`, `hooks/useNotificationScreen.ts`, `components/RenderItem.tsx`, `utils/redirectTo.ts`), bell `src/commonComponents/NotificationIcon`.
