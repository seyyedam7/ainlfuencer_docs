---
id: AFC-01
title: Seller–affiliate chat (not supported, use the Ainfluencer chat)
domain: Shared by seller and affiliate apps
audience: [seller, affiliate]
apps: [seller-app, affiliate-app]
related: [CHAT-01, AFS-12, AFA-09, AFC-02]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFC-01 · Seller–affiliate chat (not supported)

> The affiliate platform has its own built-in chat ("Messages" in the seller and affiliate apps). **It is not a supported support channel.** Brands/sellers and influencers/affiliates must talk to each other through the **Ainfluencer chat** (CHAT-01).

## Who and where
- Seller app and affiliate app: sidebar "Messages" (`/app/messages`) and "Message" buttons on affiliate, brand and product pages.

## How it works
Support policy:
1. Do **not** troubleshoot the affiliate-platform chat (missing messages, unread counts, online status, templates, auto-replies, bulk messages).
2. Tell the user that messaging between brands and influencers happens in the **Ainfluencer chat** (CHAT-01) and help them there.

## Rules and limits
For reference only (so support recognises them):
- Sellers and affiliates have a **daily limit on new conversations** set by their plan; reaching it shows "Daily Limit Reached" (AFS-12, AFA-09).
- A weekly "unread messages" email is sent on Mondays at 00:00 UTC (AFC-02).

## Common questions

### "My message to the affiliate/seller wasn't delivered in Messages."
**Answer for the user:** The affiliate platform's Messages isn't a supported channel. Please contact them through the Ainfluencer chat, where we can help if anything goes wrong.
**What to check:** nothing in the affiliate chat; check the conversation in the Ainfluencer chat (CHAT-01).
**Escalate when:** a problem in the Ainfluencer chat (follow CHAT-01).

### "It says 'Daily Limit Reached' when I start a conversation."
**Answer for the user:** That limit belongs to the affiliate-platform chat, which isn't our supported channel. Please use the Ainfluencer chat to reach this person.
**What to check:** —
**Escalate when:** —

## Related topics
- CHAT-01 — the Ainfluencer chat: the supported channel for brand ↔ influencer messages.
- AFS-12 / AFA-09 — plan limits (daily new conversations).
- AFC-02 — notifications and emails.

## Technical reference
- Endpoints: `seller/v1/chats*`, `affiliate/v1/chats*`, `*/v1/message-templates*`, `*/v1/message-variables/list`, `broadcasting/auth` (realtime).
- Weekly email: `chat:process-unread-messages` (Mondays 00:00 UTC) → `UnreadChatMessagesNotification`.
- Plan gate: `seller_access_feature:chat_system` on starting conversations and bulk/list messages.
- Data: `chat_messages` (`from_id`, `to_id`, `created_at`).
