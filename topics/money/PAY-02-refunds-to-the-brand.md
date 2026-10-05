---
id: PAY-02
title: Refunds to the brand
domain: Money (wallets, refunds, payouts)
audience: [brand, influencer, admin, system]
apps: [web, email, background]
related: [PUB-02, PUB-03, OFR-05, OFR-06, PAY-01, PAY-05, PAY-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# PAY-02 · Refunds to the brand

> When a paid deal is not delivered, or Ainfluencer staff decide the brand should get its money back, the money held in escrow goes back to the **brand's Ainfluencer wallet** (never to the card). Refunds happen only through Ainfluencer staff or automatic timers; neither the brand nor the influencer has a refund or cancel button.

## Who and where
- **System**: the automatic refund timer (every minute).
- **Admin**: disapproves delivered content (refund is immediate) or refunds a deal from the admin panel.
- **Brand**: sees the result only, in the web app at `/profile/transactions` → "Wallet & transactions": the deal shows "Refunded To Brand Wallet" and the "Refunded to you" box goes up (PAY-01). The money is immediately usable for other offers.
- **Influencer**: is told by email and in-app notification; the deal shows "Refunded Payment" in the mobile wallet (PAY-03).

## How it works
A refund is only possible while the money is **still in escrow** (the deal is "On Hold in Escrow"). Once it has been released to the influencer, it can't be refunded through the product.

The ways money comes back:
1. **Admin disapproves the content** (PUB-02): the refund happens **immediately**.
2. **Brand disapproves the content** (PUB-02): nothing is refunded right away. The automatic timer refunds it later (see below), unless an admin steps in first.
3. **Automatic refund timer**: every minute the system refunds deals whose publish job has **failed or expired** and that have either
   - been **disapproved** (by brand or admin) and are **14 days past the publish start date**, or
   - **no approval decision at all** and are **3 days past the publish start date**.
   (Exact rules and the release side are in PUB-03.)
4. **Admin refund from the admin panel**: an admin with the refund permission refunds one deal; only possible while the deal is in escrow.
5. **Revisions after payment** (OFR-05):
   - the influencer **rejects** a paid revision → what the brand paid for that revision goes back to the brand's wallet;
   - the brand sends a **newer revision** that replaces an older paid one → the older payment is refunded;
   - an approved revision **lowers the price** → the difference is refunded.

**How much comes back**
- The brand gets back **the offer price minus the coupon share and minus any credit share** used for that deal.
- **Not refunded:** the **platform fee** and the **content insurance** charged at checkout (OFR-03).
- The **coupon share** and the **credit share** go back to Ainfluencer, not to the brand (they were paid by Ainfluencer in the first place, PAY-05).
- Example: offer $100, platform fee 15% ($15), brand paid $115 in cash → refund $100 to the wallet; the $15 fee stays. If a $20 coupon had been used, the brand paid $80 + fee and gets $80 back.

## Rules and limits
- Refunds always go to the **brand's default wallet**. There is no automatic refund to a card or PayPal. Wallet money is not refundable to the card (PAY-01).
- **Brands can't request a refund** in the app, and **influencers can't cancel** a paid deal from the mobile app.
- A brand **can't disapprove** once the money has been released ("unable to disapprove delivered offer") or once an admin has approved it.
- An admin refund needs the deal to be in escrow (`order_items.status = transferred`). After release (`completed`) or payout (`settled`) there is no refund path in the product.
- One refund per deal: once `refunded`, the timer and admins skip it.

## Statuses
| What people see | Internal | Meaning |
|---|---|---|
| Brand: "On Hold in Escrow" | `order_items.status = transferred` | Refundable |
| Brand: "Refunded To Brand Wallet"; influencer: "Refunded Payment" | `order_items.status = refunded` | Money back in the brand wallet |
| Brand: "Transferred To Influencer Wallet" / "Paid Out to Influencer" | `completed` / `settled` | Too late to refund in the product |
| Offer | `offers.status = refunded` (set by the refund flow) | Shown in chat/inbox as refunded / failed |

## Timers and automatic actions
- `modules:publish_jobs:refund` — **every minute** (UTC). Finds failed/expired publish jobs on paid offers whose money is still in escrow and that are disapproved + 14 days after start, or undecided + 3 days after start; queues one refund per job (max 1,000 per run).
- Revision refunds happen at the moment the influencer rejects, or the brand replaces/approves the revision.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Any deal refund ("Funds will be refunded to {brand}!") | Influencer | Email, in-app |
| Publish job refunded (job status message) | Brand and influencer | Email, push, in-app (PUB-03) |
| Revision rejected ("…$X is refunded to your wallet…") | Brand | Chat system message (CHAT-03) |

## Common questions

### "I disapproved the post. Where is my refund?"
**Answer for the user:** When you disapprove a post, the refund isn't instant. If the influencer doesn't fix it, the money returns to your Ainfluencer wallet automatically 14 days after the publish start date. Our team can also review it sooner.
**What to check:** `publish_jobs.url_approval` (`brand:disapproved`), `publish_jobs.status`, `start`; `order_items.status` for the offer (`transferred` = still in escrow).
**Escalate when:** more than 14 days have passed since `start`, the job is failed/disapproved and the order item is still `transferred` → dev (timer didn't pick it up).

### "Why didn't I get the full amount back?"
**Answer for the user:** Refunds return the offer price to your wallet. The platform fee and the optional content insurance aren't refundable, and any part paid by a coupon or by Ainfluencer credit goes back to Ainfluencer rather than to your wallet.
**What to check:** the transaction row breakdown ("Amount", "Platform fee", "Content insurance", "Coupon discount"), `orders.fee_amount`, `orders.insurance_amount`, `orders.coupon_discount`, credit used.
**Escalate when:** the refunded amount is less than offer price − coupon share − credit share.

### "Can I get the refund on my card instead of the wallet?"
**Answer for the user:** Refunds always go to your Ainfluencer wallet so you can use them on your next collaboration. Cash refunds to a card aren't automatic; we can pass your request to our finance team.
**What to check:** wallet balance and refund history.
**Escalate when:** the brand asks for a cash refund → finance team.

### "The influencer never posted. Can I cancel and get my money back?"
**Answer for the user:** If no post link is submitted within 2 days of the publish start date, the deal fails automatically and your money is returned to your wallet a few days later. If you need it sooner, we can review it.
**What to check:** `publish_jobs.status` (`expired`/`failed`), `start`, `url_approval`; order item status.
**Escalate when:** the job is still `scheduled` long after the start date, or 3+ days after start with no approval and no refund.

### "The money was already released to the influencer, but the content was bad."
**Answer for the user:** Once the payment has been released, we can't reverse it from your account. Please send us the details and our team will review the case with the influencer.
**What to check:** order item `completed` or `settled`, approval history (`publish_jobs.url_approval`, `approved_at`).
**Escalate when:** always (finance/admin decision; no product refund path).

### (Influencer) "I got an email saying funds will be refunded to the brand. Why?"
**Answer for the user:** The deal was cancelled because the content wasn't delivered or wasn't approved in time, so the payment held in escrow went back to the brand. You can check the deal in the chat and in the Scheduler's Failed tab.
**What to check:** publish job status and disapproval reason (`publish_jobs.feedback`).
**Escalate when:** the influencer says they delivered on time and the post was live → admin review (PUB-02).

## Known issues
- *(to verify)* The automatic refund timer and an older timer check a post-link field that the current submit flow doesn't always fill. A real post could be treated as "no link" and refunded after 3 days. If an influencer delivered but the brand got refunded, escalate to dev (see PUB-03).
- *(to verify)* If the transfer into escrow fails at payment time, the order is marked `refunded` even though no money moved, while the offer can still show as paid. Escalate to dev if an offer is "paid" but its order shows refunded with no wallet movement.
- The backend has a brand self-refund endpoint, but the web app doesn't use it; if it were used it would leave the offer marked paid. Not reachable by users today.

## Related topics
- PUB-02: approving or disapproving content.
- PUB-03: release and refund timers.
- OFR-05: revisions after payment (revision refunds).
- PAY-01: the brand wallet and transaction list.
- PAY-05: who pays for coupons and credit.

## Technical reference
- **Refund engine:** `OrderService::refund($orderItemId)` (`app/Modules/Finance/Services/OrderService.php`) → `OrderItemRefundedEvent` → `TransferRefundOrderTransactionsListener`: from influencer **lock** wallet → coupon share and credit share to the system user's default wallet; `pay_price − credit − coupon` to the brand's default wallet; order item `refunded`; `InfluencerRefundNotification` (mail + database, event `influencer.deal:refunded`). Fee and insurance are not reversed.
- **Callers:** admin disapprove `POST /api/{locale}/offer/v2/{offerId}/disapprove` (`app/Modules/Offer/Http/Controllers/OfferController.php@disapprove`, immediate for admins); `RefundPublishJobJob` (queue `publish-jobs:refund`); admin `POST /api/{locale}/finance/admin/refund` (AdminRefundController, `orderItemId` must be `transferred`, permission `order-refund`); unused `POST finance/current-user/orders/refund`.
- **Revisions:** `app/Modules/Offer/Listeners/RevisionOfferRejectedListener.php` (brand lock → brand default), `RevisionOfferApprovedListener.php` (negative price → influencer lock → brand default), `Services/RevisionOfferService.php` + `Jobs/TransferMoneyToBrandDefaultWallet.php` (replaced revision).
- **Data:** `order_items.status` (pending/transferred/completed/settled/refunded), `orders.status`, `orders.fee_amount`, `orders.insurance_amount`, `orders.coupon_discount`, `offers.status`, `publish_jobs.status|state|url_approval|start|disapproved_at`, `wallet_transactions`, `transaction_reports.refunded`.
- **Scheduled job:** `modules:publish_jobs:refund` (`app/Modules/PublishJob/Console/Commands/RefundPublishedJob.php`, every minute, `withoutOverlapping`).
