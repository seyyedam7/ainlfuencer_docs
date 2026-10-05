---
id: CHAT-03
title: System messages and offer cards in chat
domain: Messaging
audience: [system, brand, influencer]
apps: [web, mobile]
related: [CHAT-01, CHAT-02, OFR-01, OFR-02, OFR-03, OFR-04, OFR-05, OFR-06, PUB-01, PUB-02, PUB-04, CMP-06]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/backend/chatbox (gateway)@master, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# CHAT-03 · System messages and offer cards in chat

> Besides people's messages, Ainfluencer automatically posts **cards** into the chat: offer created, terms changed, paid, date changed, admin edits, revision requests, publishing updates, insight requests, address cards and Shopify shipping steps. They are the deal's timeline. On mobile, the **offer card at the top** of the chat is where most influencer actions happen.

## Who and where
- Both chats: web Inbox `/new-offersV2/:id` and mobile CHAT_SCREEN.
- **Mobile offer card** (top of the chat): status, campaign, "Fee", "Delivery Date", "Duration", and the action buttons per status (Accept/Edit/Reject, Edit Offer, Submit Post URL, Extend Deadline, See Payout). "Tap to see offer details"; "Offer progress" explains the steps.
- **Web chat header** shows status, "Delivery Date:", "Paid:", action buttons, and "Click to See Offer Details" (Duration, Content Type, Bio Link, Offer ID).

## How it works
The backend posts a card at each step. Main card types and what users see:

| When | Card | Posted as |
|---|---|---|
| Offer sent (OFR-01) | "Offer created", the influencer's **collaboration terms** ("My Main Niches", "My Audience", collab types), "awaiting payment" | System / influencer |
| Counter, edit, accept, reject (OFR-02) | "&lt;actor&gt; changed the offer." with old → new values (fee, duration, delivery date, bio link); accepted/rejected | System |
| Offer paid (OFR-03) | "The Brand has just paid for this offer" / web "You paid the offer" with next-step tips | System |
| Status changes | rejected, failed, scheduled, published, completed | System |
| Date changed (OFR-04) | "Hi &lt;name&gt;, I just extended the date of delivery…" (brand) / "I just extended the date of delivery. Hope this works out." (influencer) | **Normal message** from that person |
| Admin edit (OFR-06) | Admin card listing changes; later updated to accepted/rejected/discarded | Admin (edited in place) |
| Revision (OFR-05) | "Brand … has requested to change some terms of offer", then accepted/declined | System |
| Publishing (PUB-01, PUB-03) | "The content just published", "Failed to publish the content", "Post was published successfully. Payment transferred to your wallet", approved/disapproved by brand or Ainfluencer Team, refunded | System |
| Published content (PUB-02) | Card with the submitted post/media (replaces the previous one) | System |
| Insights (PUB-04) | "Insight Request" / "Analytics Request" with steps; "You submitted your Insights/Analytics" | System |
| Address | **Address card** with the influencer's shipping address (offer, affiliate, gifting) | Influencer |
| Shopify fulfilment (CMP-06) | Request to confirm the shipping address; "address completed by influencer" | System / brand |
| Bulk message | Brand's bulk text, marked as bulk | Brand |
| Affiliate link (web) | Affiliate link message from the brand's affiliate tab | Brand |

**What cards don't do:** they don't send a push or change the "new message" alert (CHAT-02), and in the inbox preview an updated offer shows "Offer updated." instead of the card text.

## Rules and limits
- Cards are created by the system; users can't edit or delete them.
- Only one published-content card is shown (a new one replaces the old).
- Only one pending admin-edit card at a time (a newer edit removes the older pending one).
- Opening revision cards or admin cards requires the influencer to answer on the offer card (Accept/Reject).

## Statuses
The offer card shows the offer status as each side sees it (OFR-02): influencer "Waiting Your Approval", "Waiting Brand Approval", "Paid & Scheduled", "Published & Waiting Approval", "Published & Approved", "Rejected by Brand/Admin/You", "Failed"; brand "Waiting Payment", "Waiting Influencer Approval", "Scheduled", "Published", "Disapproved by Admin", "Failed".

## Timers and automatic actions
Cards are posted by the jobs of each flow (e.g. PUB-03 release/refund every minute, PUB-01 expiry), so a card can appear without anyone acting.

## Notifications
Cards themselves send no push. The underlying events send their own notifications (see each OFR/PUB topic).

## Common questions

### "The offer card on my phone shows old terms."
**Answer for the user:** Close the chat and open it again; the card refreshes when the chat reopens.
**What to check:** the offer's current values via the offer; mobile doesn't live-update edited cards.
**Escalate when:** reopening still shows wrong terms.

### "I see a message saying the offer changed, but I didn't change it."
**Answer for the user:** The card shows who made the change (the other side or our team). If it was our team, there's an admin card explaining it.
**What to check:** `change_histories.user_id` for the batch; `edit_offer_messages`.
**Escalate when:** the change has no author (OFR-06).

### "Why didn't I get a notification for 'offer paid' in the chat?"
**Answer for the user:** Chat cards don't send a phone alert, but the payment itself sends its own "offer paid" notification and email.
**What to check:** OFR-03 notifications sent.
**Escalate when:** neither the card nor the paid notification exists.

### "My shipping address card didn't send."
**Answer for the user:** Make sure you picked a saved address and that the chat is open (not closed). Try again from the location pin.
**What to check:** gifting send-address by campaign only is broken (MKT-04); chat status (CHAT-01).
**Escalate when:** the address card fails on an active chat.

## Known issues
- **Mobile doesn't live-update edited cards** (admin edits, brand edits, reactions); they refresh when the chat is reopened.
- In the inbox list, any card except the first three creation cards shows the preview "Offer updated."
- Date-change messages are sent as normal messages, so they trigger "new message" pushes (CHAT-02).

## Related topics
- CHAT-01 chat; CHAT-02 alerts.
- OFR-01…OFR-06 offer steps; PUB-01, PUB-02, PUB-04 delivery and insights; CMP-06 Shopify shipping.

## Technical reference
- **Sender:** `app/Modules/ChatBox/Services/ChatboxMessageSenderService.php` (`sendOfferUpdate`, `sendChangeStatusMessage`, `sendOfferStartChangeMessage[ByInfluencer]`, `sendOfferUpdatedByAdminMessage`/`updateOfferUpdatedByAdminMessage`, `sendRevise`, `sendPublishJobUpdate`, `sendPublishJobEvent`, `sendPublishedContent`, `sendInsight`, `sendInfluencerAddressMessage`, `sendCustomMessage`), all via gateway `POST /message/send-message` (edits via `PATCH /message/edit-message`).
- **Callers:** `app/Jobs/Offer/SendOfferInitialMessagesJob.php`, `app/Modules/Offer/Jobs/ConversationSendOfferUpdateJob.php`, `app/Listeners/v2/Offer/OfferStatusChangedEventListener.php`, `app/Modules/Offer/Listeners/SendRevisionOfferMessageListener.php`, `app/Notifications/PublishJobNotification.php`, `app/Modules/PublishJob/Jobs/SendPublishedContentMessage.php`, `app/Services/Implementation/InsightService.php`, `app/Modules/Mobile/Jobs/SendInfluencerAddressJob.php`, `app/Jobs/v2/Offer/ProcessBulkMessage.php`, `app/Modules/ChatBox/Jobs/ProcessBulkThreadMessage.php`.
- **Message attributes:** `attributes.type` values `offer:*`, `admin.offer:*`, `publish-job:*`, `revise:*`, `meta:influencer-address`, `meta:affiliate_link_message`, `brand:bulk-message`, `offer:fulfillment_complete_address_by_influencer`; `data.batch` links a card to a `change_histories` batch. Inbox preview rule in `WebhookController`.
- **Mobile:** `src/features/chat/components/OfferCard.tsx`, `BubbleStatusUpdate`, `AdminMessage`, translations `src/localization/translations/en/chat.js`. **Web:** `inboxV37/chat/parts/*`, `newSingleOfferV2/parts/newChat/parts/messages/parts/modes/*`.
