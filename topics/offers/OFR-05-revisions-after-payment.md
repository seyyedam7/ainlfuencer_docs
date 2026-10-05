---
id: OFR-05
title: Revisions after payment
domain: Offers and negotiation
audience: [brand, influencer, system]
apps: [web, mobile, email]
related: [OFR-03, OFR-04, PUB-01, PUB-03, PAY-02, CHAT-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# OFR-05 · Revisions after payment

> After paying, while the post is still scheduled, the brand can ask to change the deal: price, delivery date, duration or bio link. If the new price is higher, the brand pays the difference first. The influencer then accepts (new terms apply) or rejects (old terms stay, extra money goes back to the brand).

## Who and where
- **Brand**, web Inbox `/new-offersV2/:id`: on a **"Scheduled"** deal, the **"Edit"** button opens the same "Edit Offer" pop-up as before payment, but submitting creates a **revision request**. If extra money is needed, a **"Payment Options"** step appears in the pop-up.
- **Influencer**, mobile deal chat: card "Brand &lt;name&gt; has requested to change some terms of offer" and, on the offer card, **"Accept"** / **"Reject"** buttons in place of "Submit Post URL" / "Extend Deadline". The changed values show as "(Changed to …)".

## How it works
1. Brand clicks "Edit" on a Scheduled deal, changes the terms and clicks **"Send Edit for Approval"**. Unchanged fields (same duration, same bio link, a date on the same day) are ignored.
2. The system works out the **price difference** (new price − current price):
   - **Higher price** → the request waits for payment. The pop-up switches to "Payment Options" and the brand pays the difference (wallet, card, PayPal…, same methods as OFR-03). That money is held for the revision.
   - **Same or lower price** → no payment; the request is sent at once.
3. When the request is "sent" (paid or free), the influencer gets a push and a chat card, and the web chat header shows the pending changes in brackets (e.g. "Delivery Date: … (You requested date change to … Waiting influencer acceptance)", "Paid: $100 (Changed to $150)").
4. **Influencer accepts** ("You accepted changes the Brand … requested. Your offer should be delivered according to the new terms."):
   - The offer's price/date/duration/bio link are updated and the post is (re)scheduled with the new date.
   - A higher price: the extra money moves into the influencer's escrow. A lower price: the difference goes back to the brand's wallet.
   - The brand is notified ("Deal Change Request Accepted").
5. **Influencer rejects** ("You declined changes the Brand … requested. The offer terms stay the same as they were prior to this request."):
   - Old terms stay. Any extra money the brand paid goes back to the **brand's wallet** (not the card); any platform credit used goes back to the platform.
   - The brand is notified ("Deal Change Request Rejected").

## Rules and limits
- Only after payment and only while the post is **scheduled** ("Can not change offer that`s not Scheduled."). Not before payment ("You can only submit a revision after the offer has been paid."). On the web, "Edit" becomes a revision only when the status is "Scheduled".
- **Maximum revisions per offer:** default **2** accepted or still-open requests (team setting). The error text says "Your revision requests have reached the maximum limit (4 requests)" even when the limit is 2.
- New price must be at least the minimum offer price (default $10); lowering below $10 is refused ("The offer price cannot be less than $10.").
- New date must be in the future; the bio link must be a working URL.
- **One open request at a time:** a new request **replaces** the previous one. A previous request that was already paid is refunded to the brand's wallet; one still waiting for payment is cancelled.
- The influencer must answer before posting: while a request is open the card shows Accept/Reject instead of Submit/Extend.

## Statuses
| Status (internal `revision_offers.status`) | Meaning | Next |
|---|---|---|
| `pending` | Waiting for the brand to pay the difference; influencer sees nothing yet | Brand pays → `paid` |
| `paid` | Sent to the influencer (also used when no payment was needed) | Accept → `approved`, Reject → `rejected` |
| `approved` | Accepted; new terms in force | – |
| `rejected` | Declined or closed automatically; extra money refunded | – |
| `overwritten` / `expired` | Replaced by a newer request | – |

## Timers and automatic actions
- Open requests are **rejected automatically** (with refund) when the influencer submits the post link, when the money is released to the influencer, or when the deal is refunded (PUB-01, PUB-03).
- Current revisions **don't expire** on their own. (A 2-day expiry job still runs, but only for revisions from the old system.)

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Request sent (paid or free) | Influencer | Push, in-app; chat card "Brand … has requested to change some terms of offer" |
| Influencer accepts | Brand | In-app, email; chat card |
| Influencer rejects | Brand | In-app, email; chat card |

## Common questions

### "I asked for a change but the influencer doesn't see it."
**Answer for the user:** If your change raises the price, the request is only sent after you pay the difference in the "Payment Options" step. Please open "Edit" again and complete the payment.
**What to check:** `revision_offers` row for the offer with status `pending` (unpaid) vs `paid`.
**Escalate when:** the revision is `paid` and the influencer still has no Accept/Reject.

### "The influencer rejected my change. Where is my money?"
**Answer for the user:** The extra amount you paid for the change was returned to your Ainfluencer wallet. You can use it for any payment.
**What to check:** revision `rejected`, wallet transaction "Transfer fo Revise Id: …" back to the brand default wallet.
**Escalate when:** no refund transaction exists.

### "I can't request another change."
**Answer for the user:** Each deal allows a limited number of change requests (2 by default). If you really need another change, agree it in the chat or contact support.
**What to check:** count of `approved` + `paid` revisions vs the `backend_revision.request_limit` setting.
**Escalate when:** the count is below the limit.

### "Can I change the deal after the influencer posted?"
**Answer for the user:** No. Changes are only possible while the post is still scheduled. Once a link is submitted, open requests are closed.
**What to check:** publish job state and `published_url`.
**Escalate when:** never; explain the rule.

### "What happens if the influencer never answers?"
**Answer for the user:** The request stays open until they answer, post, or the deal is settled; then it's closed automatically and any extra payment is refunded to your wallet.
**What to check:** revision status and publish job timers.
**Escalate when:** a deal was settled but the revision is still `paid`.

## Known issues
- The limit error text always says "(4 requests)", whatever the real limit is.
- Unanswered revisions don't expire, so a brand's extra money can sit on hold for weeks until the deal is settled.
- The mobile card doesn't live-update when a request arrives; reopening the chat shows it (CHAT-03).

## Related topics
- OFR-03 payment methods (also used for the difference); OFR-04 date changes.
- PUB-01 submitting the link closes requests; PUB-03 settlement closes requests; PAY-02 refunds.

## Technical reference
- **Endpoints:** `POST /api/{locale}/v2/offer-revision` → `app/Modules/Offer/Http/Controllers/RevisionOfferController::store` (limit from `settings` `integration_platform = backend_revision`, `name = request_limit`, default 2); `POST /api/{locale}/finance/payment/entity` (`itemType = RevisionOffer`, `PaymentController::entity`) → `EntityPaidEvent` → `app/Modules/Offer/Listeners/RevisionOfferPaidListener.php`; `POST /api/v3/mobile/offer/{offerId}/revise/accept|reject` → `RevisionOfferController::accept|reject`.
- **Service:** `app/Modules/Offer/Services/RevisionOfferService.php` (`create`: amount = new − current price, ≤ 0 → `paid` immediately; refunds/overwrites older paid revisions via `TransferMoneyToBrandDefaultWallet`; `rejectOfferOpenRevisions`). Listeners: `RevisionOfferApprovedListener` (updates offer + publish job, order item "Revise", money move), `RevisionOfferRejectedListener` (refund to brand default wallet), `SendRevisionOfferNotificationListener`, `SendRevisionOfferMessageListener`.
- **Auto-close callers:** `app/Modules/Mobile/Http/Controllers/API/PublishJobController::submitPublishUrl`, `app/Modules/PublishJob/Listeners/PublishJobFinalizedListener.php`, `PublishJobRefundedListener.php`.
- **Data:** `revision_offers` (`offer_id`, `brand_id`, `influencer_id`, `status`, `amount` in cents, `metadata` = requested changes), `offers.extra.revision_changes` (pending changes shown in headers), `offers.extra.revised`.
- **Legacy:** `app/Modules/ReviseOffer` (`revision:expire`, every minute, 2 days) applies only to old-system revisions.
- **Web:** `newSingleOfferV2/parts/modal/index.tsx` (`offerReviseService` when status is "Scheduled", "Payment Options" step). **Mobile:** `src/features/chat/components/DoubleButtonsContent.tsx`.
