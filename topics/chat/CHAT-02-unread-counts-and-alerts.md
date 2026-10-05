---
id: CHAT-02
title: Unread counts and new-message alerts
domain: Messaging
audience: [brand, influencer, system]
apps: [web, mobile, email, background]
related: [CHAT-01, CHAT-03, NTF-01, NTF-02, NTF-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/backend/chatbox (gateway, api, processor)@master, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# CHAT-02 · Unread counts and new-message alerts

> How users find out about new chat messages: the unread badges in both apps, push notifications on the phone, and an hourly "you have unread messages" email.

## Who and where
- **Brand**, web: number badge on the sidebar **"Inbox"** item, unread counts per conversation in the Inbox list, bell notifications (NTF-01), and email.
- **Influencer**, mobile: badge on the **Inbox** tab, unread markers in the Inbox list, push notifications, and email.

## How it works
1. **Someone sends a message.** The chat service stores it and tells Ainfluencer.
2. **Unread count.** A message counts as unread for the other person until they open the chat. **Opening a chat marks everything up to the newest message as read** (there are no per-message "seen" ticks for the sender).
3. **Push / in-app alert.** For normal messages typed by a person, the other side gets an in-app notification and a phone push:
   - Offer chats: "new message" alert; tapping the push opens that offer's chat.
   - Affiliate/gifting threads: "New message from &lt;sender&gt;" with a short preview ("You have a new message from a brand. Open the chat to view and reply.").
4. **No alert for automatic cards.** System messages and offer cards (offer updated, paid, publish status…) and **brand bulk messages** do **not** send a push (CHAT-03).
5. **Hourly email.** Every hour the system looks for users with unread messages and sends one email ("Messages from Brands - on Ainfluencer!" to influencers, "Messages from Influencers - on Ainfluencer!" to brands) with the latest unread messages per offer chat. After sending, those messages aren't emailed again.

## Rules and limits
- Web sidebar badge refreshes about every 50 seconds when nothing changes; the mobile badge every 60 seconds (and the Inbox list every 30 s while open).
- The mobile badge counts only messages from the other side.
- Push works only while the influencer is logged in on the phone (NTF-02).
- Archived/deleted users get no chat emails or pushes.
- The digest email covers **offer chats**; affiliate/gifting thread messages are not listed in it *(to verify)*.

## Statuses
Not applicable (counts only).

## Timers and automatic actions
| Job | When (UTC) | What |
|---|---|---|
| `conversations:aggregated-message-notifier` | Every hour | Unread-messages email per user, then clears the "to email" list |

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| New typed message in an offer chat | The other participant (both, if the sender role is unknown) | Push + in-app |
| New typed message in an affiliate/gifting thread | Other participant | Push + in-app |
| System card, bulk message | – | None (only the card in chat and the badge) |
| Unread messages waiting | Each user with unread messages | Email, hourly |

## Common questions

### "I didn't get a notification for a message."
**Answer for the user:** Push notifications are sent for messages typed by the other person, not for automatic updates or bulk messages. Make sure you're logged in on the app and notifications are allowed in your phone settings.
**What to check:** the message type (system card or bulk = no push); user logged in on mobile; notification record in `notifications`.
**Escalate when:** a normal typed message produced no notification record.

### "The unread badge won't go away."
**Answer for the user:** Open the chat and scroll to the newest message; that marks it as read. If it stays, close and reopen the app.
**What to check:** the influencer side marks seen via `PUT v3/mobile/offers/{id}/state`; the count comes from the chat service.
**Escalate when:** the count stays after opening every chat.

### "I keep getting 'Messages from Influencers' emails."
**Answer for the user:** We send a summary once an hour while you have unread messages. Reading your Inbox stops them.
**What to check:** unread chats for the user.
**Escalate when:** emails continue after all chats are read.

### "Tapping the notification opened Explore instead of the chat."
**Answer for the user:** For affiliate and gifting messages the app currently opens the main screen. Open the Inbox tab to see the message.
**What to check:** event `events:chat:message-received`.
**Escalate when:** offer-chat pushes also open Explore.

### "The influencer says they never got my bulk message alert."
**Answer for the user:** Bulk messages appear in each chat but don't send a phone notification; the influencer will see them in their Inbox and in the hourly email.
**Escalate when:** the message isn't in the chat at all.

## Known issues
- **Affiliate/gifting message pushes open Explore**, not the chat (the app doesn't recognise that push type). Workaround: open Inbox. Escalate to dev.
- **Bulk and system messages never push**, by design of the current rule (any typed card is treated as "system").
- Date-change messages (OFR-04) are sent as normal messages, so they do push.

## Related topics
- CHAT-01 the chat; CHAT-03 which messages are system cards.
- NTF-01 in-app notifications; NTF-02 push routing; NTF-03 email preferences.

## Technical reference
- **Webhook:** chat gateway → `POST /api/v2/chatbox/webhook` (`app/Modules/ChatBox/Http/Controllers/WebhookController.php`): offer chats update `offers.last_message`/`last_message_at` and send `App\Notifications\Message` (DATABASE + FIREBASE; event `events:mobile:offer:message-received`); threads send `app/Modules/ChatBox/Notifications/ConversationMessageNotification.php` (event `events:chat:message-received`). Skipped when `app/Modules/ChatBox/Support/SystemMessageDetector.php` says system (`attributes.type`, `data.batch`, `is_systematic`).
- **Unread counts:** chat service `GET /message/unread-count` (web sidebar); Laravel `GET v3/mobile/chats` → `total_unread_messages_count`; seen = socket `seen-message` / gateway `POST /delivery/seen/:id` (advances `Participant.lastReadIndex`).
- **Digest:** `app/Modules/ChatBox/Console/Commands/AggregatedConversationMessageNotifier.php` (hourly, `app/Console/Kernel.php`) → `Jobs/DispatchUnreadMessageFetcher.php` → `Jobs/FetchUnreadMessages.php` → `Notifications/ConversationAggregatedMessage.php` (EMAIL); cache cleared via `app/Listeners/NotificationSentListener.php` → gateway `GET /delivery/remove-cached-unread/{userId}`. Reads processor Redis keys `identities_with_unread`, `latest_unread:{user}:{conv}`.
- **Mobile:** `src/features/inbox/actions/useInboxUnreadBadgeCount.ts`, push routing `src/features/notification/utils/redirectTo.ts`. **Web:** `src/components/layouts/dashboard/sidebar/index.tsx`.
