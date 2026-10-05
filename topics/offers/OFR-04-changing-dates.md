---
id: OFR-04
title: "Changing dates: publish date and deadline extension"
domain: Offers and negotiation
audience: [brand, influencer]
apps: [web, mobile]
related: [OFR-02, OFR-03, OFR-05, PUB-01, PUB-03, CHAT-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# OFR-04 · Changing dates: publish date and deadline extension

> The "delivery date" (publish date) is when the influencer must post. Before payment, the brand changes it while paying if it's too close. After payment, the influencer can push it back once with **"Extend Deadline"**, and the brand can change it through a revision (OFR-05) or, after a failed delivery, through **"Restart the Deal by Extending the Date"**.

## Who and where
- **Brand**, web Inbox `/new-offersV2/:id`:
  - Pop-ups shown when clicking **"Pay $…"**: "Update Date & Make Payment" (less than 24 h left) or "Update Date and Make Payment" (date passed). Button **"Update & Pay"**.
  - On a **"Failed"** deal, the **"Edit"** button opens **"Restart the Deal by Extending the Date"** ("The deal failed to be delivered on time by the influencer but don't worry you can save it by extending the date of delivery."). Button **"Update"** (money still in escrow) or **"Update & Pay $…"** (money already refunded).
  - On a "Scheduled" deal, **"Edit"** is a revision request (OFR-05).
- **Influencer**, mobile deal chat (CHAT_SCREEN): **"Extend Deadline"** on the offer card (status "Paid & Scheduled", or "Failed" while the money is still in escrow) and in the post-step bubble → screen **"Extend Deadline"** (EXTEND_DEADLINE) with a "Delivery date" picker and **"Submit"** ("Choose a later date for delivering this campaign.").

## How it works
**Brand, before payment (date too close or passed)**
1. Brand clicks "Pay". If the date is less than 24 h away or already passed, a pop-up asks for a "New Delivery Date:" (pre-filled 15 days ahead).
2. The new date must be at least ~24–25 hours ahead ("You can't expect influencers to publish your content today… We suggest give it at least 5-7 days for the best results.").
3. "Update & Pay" saves the date **immediately, without asking the influencer**, keeps the offer "Waiting Payment", posts a chat message from the brand ("Hi &lt;name&gt;, I just extended the date of delivery. Hope this works out.") and opens the checkout (OFR-03).

**Influencer, after payment ("Extend Deadline")**
1. Influencer taps "Extend Deadline", picks a later date/time and taps "Submit".
2. The date changes **immediately** (the brand is not asked). A chat message from the influencer says "I just extended the date of delivery. Hope this works out."
3. If the job had already **failed/expired** (no link within 2 days of the date, PUB-01) but the money is still in escrow, extending **revives the deal**: the post is scheduled again and the offer goes back to "Paid & Scheduled".

**Brand, after a failed delivery ("Restart the Deal by Extending the Date")**
- **Money still in escrow** (status still paid): "Update" saves the new date on the offer.
- **Money already refunded** to the brand: "Update & Pay $…" puts the offer in the basket with the new date; after paying, the deal is scheduled again with the new date (like OFR-03).

## Rules and limits
- Influencer: **one extension** per scheduled period. The counter resets when the job expires, so after a failure the influencer can extend once more. Only possible while the post is still scheduled (not after it's published, approved or refunded). The new date must be in the future (the app also blocks dates not later than the current one).
- Influencers can't extend when the deal was refunded (the card shows "The offer failed to be delivered." with no buttons).
- Brand before payment: new date at least 1 day ahead (server rule) / 25 hours (web check).
- Brand after payment on a scheduled deal: use a revision (OFR-05); the influencer must accept it.

## Statuses
| What the user sees | Meaning | Effect of a date change |
|---|---|---|
| "Waiting Payment" | Not paid | Brand sets date while paying |
| "Paid & Scheduled" / "Scheduled" | Paid, waiting for the post | Influencer can extend once; brand uses a revision |
| "Failed" (influencer card: "You are late but you can still publish the post & submit it`s URL!") | Job expired, money still in escrow | Influencer extends → back to scheduled; brand "Update" |
| "Failed" after refund | Money back with the brand | Brand "Update & Pay" re-pays with a new date |

## Timers and automatic actions
- A scheduled job with **no link 2 days after the date** expires (PUB-01); money is refunded **3 days** after that if nothing else happens (PUB-03). Extending in time stops this.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Brand changes date before payment | Influencer | Chat message from the brand (counts as a normal message, so a "new message" push) |
| Influencer extends | Brand | Chat message from the influencer (normal message alert) |
| Brand restarts a refunded deal | Influencer | "Offer paid" notifications (OFR-03) |

## Common questions

### "The influencer changed the delivery date without asking me."
**Answer for the user:** Influencers can extend their deadline once after payment if they need more time; your money stays in escrow until the content is delivered and approved. If the new date doesn't work for you, discuss it in the chat or request a change with "Edit".
**What to check:** `publish_jobs.start`, `offers.start`, `publish_jobs.extra.dead_line_extend_requests`.
**Escalate when:** the date changed more than once in the same scheduled period.

### "I can't extend my deadline again."
**Answer for the user:** You can extend only once per scheduled period. Please publish and submit your link, or ask the brand to change the date through the chat.
**What to check:** `dead_line_extend_requests` = 1; job state/status.
**Escalate when:** the counter is 0 and it still fails.

### "My deal failed. Can I still save it?"
**Answer for the user (influencer):** If the brand's money is still in escrow, yes: tap "Extend Deadline" or publish and tap "Submit Post URL". Once the money has been refunded to the brand, they need to restart the deal.
**What to check:** offer `status` (paid vs refunded), order item status.
**Escalate when:** status is paid, job expired, and the extend still fails.

### "Why must I change the date before paying?"
See OFR-03. The date must be at least a day ahead.

### "I restarted a failed deal but it still shows Failed."
**Answer for the user:** We're checking it for you.
**What to check:** see Known issues; compare `offers.start` with `publish_jobs.start/status`.
**Escalate when:** always (dev).

## Known issues
- **Brand "Update" on a failed but still-paid deal** saves the new date on the offer, but the post job may stay expired, so the deal can still show "Failed" and the refund timer keeps running *(to verify)*. Workaround: ask the influencer to use "Extend Deadline", or escalate to dev/admin (admin "repay/reschedule", OFR-06).
- Date-change chat messages are sent as normal messages, so both sides may get a "new message" push for them.

## Related topics
- OFR-03 paying (date check before checkout); OFR-05 revisions after payment.
- PUB-01 posting and the 2-day expiry; PUB-03 refunds.

## Technical reference
- **Brand:** `PUT /api/v2/offers/{offer}/update-start` → `app/Http/Controllers/v2/Frontend/OfferController::updateStartBeforePayment` → `OfferService::updateStartBeforePayment()` (sets `start`, status `awaiting-payment`, `ChatboxMessageSenderService::sendOfferStartChangeMessage`). Request `app/Http/Requests/v2/Offer/UpdateStartDate.php` (`after_or_equal` now + 1 day). Failed-deal "Update" uses `PUT /api/v2/offers/{offer}/counter` (amend mode on paid offers); refunded deals go through basket + `finance/payment` with order item `metadata.start`. `PUT v2/offers/{offer}/failure/update-start` exists but the web app doesn't call it.
- **Influencer:** `PUT /api/{locale}/v3/mobile/deals/{id}/deadline/extend` → `app/Modules/Mobile/Http/Controllers/API/PublishJobController::extendDeadline` → `app/Modules/Mobile/Services/Implementation/PublishJobService::extendDeadline()`; request `app/Modules/Mobile/Http/Requests/PublishJob/ExtendDeadlineRequest.php` (`REQUEST_LIMIT = 1`, owner check, `start after:now`).
- **Data:** `offers.start`, `publish_jobs.start/state/status`, `publish_jobs.extra.dead_line_extend_requests` (reset by `app/Jobs/PublishJob/ExpirePublishJobJob.php`).
- **Web:** `newSingleOfferV2/parts/changePublishDataModal`, `changePublishDateWaitingPaymentModal`, `changePublishDateAfterFailure`. **Mobile:** `src/features/chat/screens/ExtendDeadline/index.tsx`, `components/FailedButtonContent.tsx`.
