---
id: OFR-06
title: Admin changes to offers
domain: Offers and negotiation
audience: [admin, brand, influencer]
apps: [web, mobile, email]
related: [OFR-02, OFR-03, OFR-04, OFR-05, PUB-02, PUB-03, CHAT-03, CHAT-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# OFR-06 · Admin changes to offers

> Ainfluencer staff can change an offer from the admin panel: edit its terms, put it back to "Waiting Payment", send it back for approval, re-charge a refunded deal, or delete an unpaid offer. Users see these as **admin messages in the chat** and get an email or push. This explains offers that "changed by themselves".

## Who and where
- **Admin**, in the admin panel (out of scope for this knowledge base; only the effects are described here).
- **Brand** sees an admin card in the web Inbox chat (messages from staff show "Admin:") and gets an email.
- **Influencer** sees an admin card in the mobile chat and gets a notification ("updated by admin"); the push opens the chat.

## How it works
**1. Edit offer terms** (price, delivery date, duration, bio link)
- Before payment: the change is logged and the offer goes into the chosen waiting state (for example "Waiting Payment"). A chat card lists the changes; the **influencer can accept or reject** the admin edit from the offer card (the same Accept/Reject as a counter, OFR-02).
- On a scheduled (paid) deal: date, duration and bio link can change; price can't (it's ignored once paid, except in a few pre-payment states). The post is rescheduled.
- On a refunded deal: if the brand's wallet holds enough money, the admin can re-activate it (offer back to paid, post rescheduled, money taken from the wallet); otherwise the admin can set it to "awaiting payment".
- The admin joins the chat for the message and leaves afterwards.

**2. Reset to "Waiting Payment"** — terms (and optionally price) updated, offer set back to **awaiting payment** so the brand can pay. Not possible if money is already in escrow or released.

**3. Send back for approval ("waiting for influencer")** — offer set to **pending** with the edited terms; not possible once money is in escrow or released.

**4. Re-pay (re-charge)** — for refunded or rejected offers (or paid ones whose job expired): the offer is paid again from the **brand's wallet balance** (no new card charge, no platform fee), with an optional new date. Refused when the wallet is short ("The brand`s wallet balance is insufficient to complete this payment.").

**5. Delete** — only for unpaid offers in "Waiting Payment" with no transactions. The chat is closed for both users (read-only).

**6. Approve or disapprove content** on the brand's behalf — see PUB-02 (admin disapproval refunds the brand immediately).

**What happens to a pending admin edit**
| Event | Admin card becomes |
|---|---|
| Influencer accepts | accepted |
| Influencer rejects | rejected |
| Brand counters ("Edit") before the influencer answers | **discarded** |
| A newer admin edit | old pending card is removed |
| Edit that needs no answer (e.g. reschedule after refund) | "ignored" (informational) |

## Rules and limits
- Paid offers: price can't be changed by the admin edit (use a revision, OFR-05).
- Reset / send back are blocked once escrow exists ("The amount of this offer is already locked or has been transferred to the influencer…").
- Re-pay uses only the brand's wallet; card is not charged.
- Delete only for unpaid "Waiting Payment" offers; a deleted offer can't be re-made by the influencer for the same campaign (OFR-01).

## Statuses
Admin actions move offers between the normal statuses (OFR-02, OFR-03): `pending`, `awaiting-payment`, `paid`, `refunded`. The admin message itself has its own status: `pending`, `accepted`, `rejected`, `discarded`, `ignored`.

## Timers and automatic actions
None specific. Re-activated deals follow the normal post timers (PUB-01, PUB-03).

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Any admin edit / reset / send back / re-pay | Brand | Email ("admin edited offer" mail with link to the offer) |
| Same | Influencer | Push, in-app (and email per template); push opens the chat |
| Same | Both | Admin card in chat (no chat push) |

## Common questions

### "My offer changed and I didn't do anything."
**Answer for the user:** Our team adjusted this offer (you'll see an admin message in the chat explaining the new terms). If the change needs your approval you'll see Accept/Reject on the offer card.
**What to check:** `edit_offer_messages` for the offer's chat and the latest `change_histories` rows; who made the change.
**Escalate when:** there's no admin message and no change history (unexplained change).

### "The admin's change disappeared after I edited the offer."
**Answer for the user:** When you send your own changes, any pending change from our team is discarded so only one proposal is open at a time. Contact us if you still want the admin's version.
**What to check:** `edit_offer_messages.status = discarded`.
**Escalate when:** the brand wants the admin edit re-applied (admin team).

### "Money was taken from my wallet for an old deal."
**Answer for the user:** Our team re-activated a deal that had been refunded, using your wallet balance, so the influencer can deliver. Details are in the chat.
**What to check:** wallet transaction for the re-pay order ("Repay offer by admin."), offer status `paid`.
**Escalate when:** the brand didn't agree to the re-activation (admin/finance).

### "My offer was deleted."
**Answer for the user:** Our team removed this unpaid offer. The chat stays visible but is closed.
**What to check:** `offers.deleted_at`, `offers.extra.deletion_reason`.
**Escalate when:** the offer was paid.

### "The influencer can't answer the admin's change."
**Answer for the user:** They need an active account to accept or reject. Once it's active they'll see the buttons on the offer card.
**What to check:** influencer `qc_status`; offer `status = pending`.
**Escalate when:** account active and buttons missing.

## Known issues
- "Send back for approval" records the change as the influencer's, so the **brand** sees "Waiting your Approval" and an "Accept" button that doesn't work (OFR-02 Known issues). Workaround: admin uses "Reset to Waiting Payment" instead.
- The mobile app doesn't live-update edited admin cards; the influencer must reopen the chat (CHAT-03).

## Related topics
- OFR-02 statuses and counters; OFR-03 payment; OFR-04 dates; OFR-05 revisions.
- PUB-02 admin approve/disapprove; PUB-03 refunds.
- CHAT-03 admin cards in chat; CHAT-04 flagged offers reviewed by admins.

## Technical reference
- **Admin edit:** `app/Services/Implementation/Admin/OfferService::update()` (+ `change_offer_history`, `join_chat`/`leave_chat` via gateway participant API) → `ChatboxMessageSenderService::sendOfferUpdatedByAdminMessage()` (`admin.offer:pending|ignored`), mail `AdminEditOfferToBrand`, notification `NewAdminEditOfferToInfluencerNotification`.
- **Admin actions** (`app/Modules/Offer/Routes/api.php`, prefix `offer/admin`, admin middleware) → `app/Modules/Offer/Http/Controllers/OfferController`: `POST {offerId}/repay` (wallet-funded order, `OrderPaidEvent($order, false)` = no platform fee), `POST {offerId}/reset-to-awaiting-payment`, `POST {offerId}/waiting-for-influencer`, `POST {offerId}/approve|disapprove`.
- **Delete:** `Admin/OfferService::delete()` (only "Waiting Payment" without transactions; conversation → `SYSTEM_ONLY`).
- **Edit resolution:** `OfferService::discardAdminEditMessages()` (brand counter), `OfferHistoryService` accept/reject → `acceptAdminEditMessages` / `rejectAdminEditMessages` → `ChatboxMessageSenderService::updateOfferUpdatedByAdminMessage()` (edits the chat message in place).
- **Data:** `edit_offer_messages` (`conversation_id`, `message_id`, `offer_id`, `status`, `sender_id`, `updater_id`), `change_histories`, `offers.status`, `offers.deleted_at`, `offers.extra.deletion_reason`.
- Messages: `resources/lang/en/modules/offer.php`.
