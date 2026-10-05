---
id: CHAT-01
title: Brand–influencer chat
domain: Messaging
audience: [brand, influencer]
apps: [web, mobile]
related: [CHAT-02, CHAT-03, CHAT-04, OFR-01, OFR-02, CMP-04, CMP-05, MKT-03, MKT-04, AFC-01, INF-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/backend/chatbox (gateway, api, processor)@master, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# CHAT-01 · Brand–influencer chat

> The **Ainfluencer chat is THE channel for all brand ↔ influencer messaging**: paid collaboration offers, **affiliate campaigns** and **gifting (product seeding) campaigns**. Brands use the Inbox in the web app; influencers use the Inbox in the mobile app. The affiliate platform's own chat (seller.ainfluencer.com / affiliate.ainfluencer.com "Messages", AFC-01) is **not supported**: always direct users here.

## Who and where
- **Brand (and team members)**, web app:
  - **Inbox** `/new-offersV2` (sidebar "Inbox" with unread badge) and one chat at `/new-offersV2/:id`.
  - **Chat drawer** on campaign pages: affiliate dashboard `/affiliate/dashboard`, `/campaigns/affiliate-influencers/:id`, gifting pipeline `/campaigns/seeding-influencers/:id`.
  - Dashboard `/` shows recent messages.
- **Influencer**, mobile app: **Inbox** tab (INBOX_SCREEN), a brand's list of threads (CHAT_THREADS_SCREEN), and the chat (CHAT_SCREEN). "Message brand" buttons on affiliate and gifting campaign screens.
- **Not this topic:** the yellow "support chat" bubble / "Get help" (Chatgram) is Ainfluencer customer support, not brand ↔ influencer chat.

## How it works
**Rooms and threads.** The inbox shows one **room per brand–influencer pair**. Inside a room there is one **thread** per collaboration:
- **Collab / "UGC Collab" thread** (paid offer): created automatically when the influencer sends the offer (OFR-01). One per offer.
- **Affiliate thread**: one per affiliate campaign and influencer, created on the **first message** from either side (brand from the affiliate pages, influencer with "Message brand" on the affiliate campaign).
- **Gifting / "Seeding" thread**: one per gifting campaign and influencer, created on the first message or when an address card is sent.
If a room has several threads, the brand sees a thread switcher at the top of the chat; the influencer sees CHAT_THREADS_SCREEN.

**Sending messages**
1. Type and send. Messages travel live (real-time) to the other side if they are online. Long messages are split automatically (about 2,200 characters per part on mobile).
2. **Attachments**: images, videos and files. Web limits: max 10 files at a time; videos 200 MB, images 20 MB, PDFs 50 MB. Mobile upload limit 50 MB.
3. **Message templates**: both sides can save and reuse templates (web "Templates" icon; mobile "Templated Responses", "Create New Template").
4. **Shipping address** (influencer): the location pin in the chat input sends a saved address as an **address card** (paid offers, affiliate and gifting threads). "Set it once and use it with all brands."
5. **Insights** (influencer): attach insight screenshots to answer the brand's "Insight Request" (PUB-04).
6. **Reactions** (mobile only): influencers can react to a brand's message with an emoji.

**Contact details are hidden or blocked before payment** (CHAT-04): unless the brand is on the **Ultimate** plan, or the offer is paid, emails/phones/links/handles are hidden from view ("Contact hidden · tap to learn why") and a brand trying to send them sees an "Unlock full sharing with &lt;name&gt;" pop-up instead of sending.

**Inbox tools**
- **Brand (web):** campaign filter and "Filters" dialog (thread and offer status), **"Send Bulk Messages"** to many influencers (offer and existing affiliate/gifting threads), and per-offer **"More actions"** (⋮) menu: "Reject Offer", **"Archive"**, "Delete", "Spam User", "Block User", "Report Influencer".
- **Influencer (mobile):** search ("Search for a campaign or brand ..."), account selector, status filter ("Paid & Scheduled", "Waiting Your Approval", "Waiting Brand Approval", "Published", "Failed"), **Archive/Unarchive** a thread, **pin** up to **5** rooms ("Pin Limit Reached!").

## Rules and limits
- **Only the two participants** (plus Ainfluencer admins who join temporarily) can read or write a thread. Brand team members see the brand owner's chats.
- **When the chat is closed (read-only).** Nobody can send messages in an offer chat when:
  - the brand **archived, deleted or spammed/blocked** the offer from the ⋮ menu (un-doing it with "Unarchive"/"Retrieve"/"Unspam User"/"Unblock User" re-opens the chat);
  - the influencer's social account is **not active** (pending review, draft, suspended, archived): all its offer chats become read-only until it's active again (INF-03, INF-04); the mobile input is hidden while suspended;
  - an admin deleted the offer (OFR-06).
- Archive/pin on mobile is **per viewer**: it doesn't change what the brand sees. The brand's ⋮ "Archive" on the web is different: it also closes the chat for both.
- **Bulk messages:** default **5 sends per brand per day** (UTC day) with a **5-minute cooldown** between sends (team-adjustable); message box limit 200 characters in the web form.
- **Not available:** editing or deleting sent messages, read receipts ("seen" ticks), searching messages, online status, archive/pin on the web inbox list.

## Statuses
| Thread | Shown as | Notes |
|---|---|---|
| Paid offer | "Collab" (web) / "UGC Collab" (mobile) with the offer status (OFR-02) | Created with the offer |
| Affiliate | "Affiliate" | Created on first message |
| Gifting | "Seeding" (web) / "Gifting" (mobile) with the gifting stage | Created on first message or address card |
| Conversation status (internal) | `ACTIVE` / `SYSTEM_ONLY` (read-only) / `CLOSED` | See Rules |

## Timers and automatic actions
- Hourly unread-messages email, push alerts and badges: CHAT-02.
- Daily AI check of offer chats for contact sharing at 03:30 UTC: CHAT-04.
- Pending offers pruned after 6 months delete their chat too (OFR-02).

## Notifications
See CHAT-02 (push and email for new messages). System cards: CHAT-03.

## Common questions

### "Can I message the influencer/brand on the affiliate platform instead?"
**Answer for the user:** Please use the Ainfluencer chat (Inbox in the web app for brands, Inbox in the mobile app for influencers). It's the only supported way to talk about paid, affiliate and gifting collaborations; the affiliate platform's messages aren't monitored by support.
**What to check:** nothing.
**Escalate when:** never; this is policy (AFC-01).

### "How do I message an influencer in my affiliate / gifting campaign?"
**Answer for the user:** Open the campaign's influencer list (Affiliate dashboard → Affiliates, or the gifting campaign pipeline) and click the chat icon next to the influencer. Your first message creates the conversation, which then also appears in your Inbox.
**What to check:** the influencer has joined/requested the campaign.
**Escalate when:** the chat drawer shows an error for a valid influencer.

### "The influencer can't reply / my message box is disabled."
**Answer for the user:** The chat is closed right now. This happens if the offer was archived, deleted or blocked, or if the influencer's account is under review or suspended. Unarchive the offer from the ⋮ menu, or wait until the account is active.
**What to check:** `offers.state` (archive/delete/spam/block), the influencer integration `qc_status`, offer `deleted_at`.
**Escalate when:** state is normal, account active, and sending still fails.

### "My message shows 'Contact hidden'."
**Answer for the user:** To protect both sides, contact details and links are hidden until the offer is paid (or when chatting with a brand on the Ultimate plan). Keep the conversation on Ainfluencer.
**What to check:** offer status, brand plan. See CHAT-04.
**Escalate when:** the offer is paid and contacts are still hidden.

### "I can't find a chat in my inbox."
**Answer for the user (influencer):** Check the "Archived" view and the account selector (choose "All accounts"). **(brand):** check filters/campaign filter; archived offers stay in the list but are closed.
**What to check:** per-viewer archive state, filters, whether the offer was deleted/pruned.
**Escalate when:** the offer exists and is active but the thread doesn't appear.

### "Can I delete or edit a message I sent?"
**Answer for the user:** Not at the moment. If you sent something by mistake, send a correction.
**Escalate when:** a message must be removed for legal/safety reasons (admin).

### "Bulk message says I have to wait."
**Answer for the user:** You can send up to 5 bulk messages per day, with 5 minutes between them. Try again after the cooldown or tomorrow (UTC).
**What to check:** brand `meta.bulk_message_daily_count` for today.
**Escalate when:** the brand hasn't sent any today and is blocked.

## Known issues
- **Brand ⋮ "Archive" closes the chat for both sides** (not just hides it): the influencer can no longer send messages until the brand unarchives. Workaround: unarchive.
- **Reactions** (mobile) may disappear after reopening the chat; the brand side doesn't show them.
- New affiliate/gifting threads created while the brand's Inbox is open appear only after a refresh.
- Pushes for affiliate/gifting thread messages open the Explore tab, not the chat (CHAT-02).

## Related topics
- CHAT-02 unread badges and alerts; CHAT-03 system cards; CHAT-04 contact-sharing rules.
- OFR-01/OFR-02 offers; CMP-04/MKT-03 affiliate campaigns; CMP-05/MKT-04 gifting.
- AFC-01 the unsupported affiliate-platform chat; INF-04 suspensions.

## Technical reference
- **Laravel endpoints** (shared controller `app/Modules/ChatBox/Http/Controllers/ChatConversationController.php`): brand `GET /api/v2/chatbox/chats` (rooms + threads + unread + `bulk_message_stats`; filters in `ChatListRequest`), `GET /api/v2/chatbox/chats/{conversationContext}`, `POST /api/v2/chatbox/chats/messages` (`subject_type` affiliate|product_seeding, `subject_id`, `influencer_user_id`, `message`); mobile `GET/POST /api/{locale}/v3/mobile/chats*`, `POST …/chats/affiliate/{adId}/send-address`, archive/unarchive/pin/unpin thread, `rooms/{room}/pin|unpin`. Offer endpoints: `POST v3/mobile/offers/{id}/send-address`, `PUT v3/mobile/offers/{id}/state` (marks seen), brand `PATCH /api/v2/offers/{offer}/state` (archive/delete/spam/block → `app/Classes/Offer/StateAction.php` → `UpdateConversationStatus`). Templates: `v2/message-templates*`, `v3/mobile/message-templates*`. Bulk: `POST /api/v2/offers/send-bulk-message` (`app/Services/Implementation/BulkMessageService.php`, defaults 5/day, 5 min; setting `twilio_bulk_message_limit_settings`). Campaign filter: `GET /api/v3/brand/inbox/campaigns`.
- **Chat service:** clients connect by socket to `chatbox-gateway` (events `new-message`, `seen-message`, `react-message`); history `GET /message/conversation/:id`; writes via Kafka to `chatbox-processor` (Mongo). Laravel creates conversations (`OfferCreatorService::setupChatBoxConversation`; lazy threads via `app/Modules/ChatBox/Services/ConversableMessageSender.php`, `ConversationContextService`). A non-`ACTIVE` conversation makes participants `RESTRICTED` (no sending).
- **Data:** MySQL `offers.chatbox_conversation_id` (= `chat_id`), `offers.state`, `rooms`, `room_participants`, `room_user_states` (room pins), `conversation_contexts` (`provider_conversation_id`, `context_type` sponsored/affiliate/product_seeding/hybrid, `ad_id`), `conversation_context_user_states` (per-viewer archive/pin); Mongo `Conversation.status`, `Participant.status/lastReadIndex`, `Message`.
- **Status triggers:** `app/Listeners/v2/Integration/UserQualityControlChangeListener.php` (non-active account → `SYSTEM_ONLY`), `app/Listeners/v2/Offer/ConversationStateChangeListener.php`.
- **Web:** `src/components/pages/inboxV37/*`, `InboxV37ChatMenu.tsx`, `chat/InboxV37ChatSender.tsx`. **Mobile:** `src/features/inbox/*` (`MAX_PINNED_ROOMS = 5`), `src/features/chat/screens/ChatScreen.tsx`, `hooks/useChat.tsx`.
