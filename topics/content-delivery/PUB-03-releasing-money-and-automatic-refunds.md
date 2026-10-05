---
id: PUB-03
title: Releasing money and automatic refunds
domain: Content delivery and settlement
audience: [brand, influencer, admin, system]
apps: [background, web, mobile, email]
related: [PUB-01, PUB-02, OFR-03, OFR-04, OFR-05, PAY-02, PAY-03, PAY-04, PAY-05]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# PUB-03 · Releasing money and automatic refunds

> After a brand pays for an offer, the money is held by Ainfluencer (escrow). Background jobs that run every minute decide whether that money goes to the influencer's wallet or back to the brand's wallet. In today's flow **money is only released when the brand or an admin approves the delivery**; if nobody approves, the brand is refunded automatically.

## Who and where

- **Nobody clicks anything here.** This is done by scheduled jobs on the server (all times UTC).
- Brands see the result in their wallet and billing history (`/funds`, `/profile/transactions`, see PAY-01 and PAY-02) and in the inbox status of the deal.
- Influencers see the result in the mobile app wallet ("Payments" tab, see PAY-03) and on the deal card in chat ("Published & Approved", "Failed").
- Admins can approve or disapprove a delivery from the admin panel, which triggers the same release or refund.

## How it works

Every paid offer has a **publish job** (the delivery record, see PUB-01). Its **start** date is the agreed publish date. The money sits in the influencer's locked (escrow) wallet as an order item with status `transferred`.

1. **Brand or admin approves** the delivery (PUB-02). Within about a minute the release job moves the money from the influencer's locked wallet to their normal wallet ("available"). The influencer gets "Funds Added to Wallet!". Any open revision requests on the offer are closed automatically (OFR-05).
2. **Nobody approves.** The system treats the deal as failed and refunds the brand:
   - If the influencer **submitted a post link that Ainfluencer could read** (Instagram, TikTok or YouTube post that the scraper fetched), the start date is moved to "fetch time + 7 days". The deal fails **2 days after that** (≈ 9 days after the link was submitted) and the brand is refunded **3 days after that start** (≈ 10 days after submission).
   - If the influencer **uploaded content files** (content-only deal), the start date is moved to "upload time + 7 days". The deal expires 9 days after upload and the brand is refunded 10 days after upload.
   - If the influencer **submitted a link that could not be read** (for example an Instagram story, a private post, an unsupported site, or the scraper failed), the start date is **not** moved. The deal fails 2 days after the agreed publish date and is refunded 3 days after it, even if the link was submitted only hours before.
   - If the influencer **submitted nothing**, the deal expires 2 days after the publish date ("publishing failed") and the brand is refunded 3 days after the publish date.
3. **Brand disapproves** (PUB-02): the money stays in escrow and the brand is refunded **14 days after the job's start date** (if that date is already more than 14 days ago, within a minute).
4. **Admin disapproves**: the brand is refunded **immediately**.

The refund goes to the **brand's Ainfluencer wallet**, not back to the card. The coupon and credit share of the payment goes back to Ainfluencer, not to the brand, and the brand's platform fee is not refunded (see PAY-02 and PAY-05). After a refund the offer status becomes `refunded`.

## Rules and limits

- Release needs: the publish job is `healthy`, in state `completed` (set by approval), has an approver, `url_approval` is `brand:approved` or `admin:approved`, and the order item is still `transferred`.
- A brand can approve while the publish job is still in state `scheduled`, even if it already shows as Failed. **Approving after the refund re-charges the brand's wallet** for the offer price (the fee is not charged again); if the wallet doesn't cover it, the approval is refused with "You currently do not have enough funds in your wallet."
- A brand **cannot disapprove** once the money is released: "You cannot disapprove this offer, the money has been transferred to the influencer's wallet."
- If the influencer submits a new link after the deal failed, the deal becomes healthy again and the start date moves to "now + 7 days", which restarts the timers. This only saves the money if it happens **before** the refund.
- The amount the influencer sees as earned is shown in PAY-03 (price × 0.8). Withdrawal is manual (PAY-04).

## Statuses

| What the user sees | Internal values | What it means | What happens next |
|---|---|---|---|
| Brand: "Published" / Influencer: "Published & Waiting Approval" | job `scheduled` + `healthy`, `url_approval=pending` | Link or files submitted, not yet approved | Approve → release. No approval → fails and refunds (timers above). |
| Brand: "Published" / Influencer: "Published & Approved" | job `completed` + `healthy`, `url_approval=brand:approved` or `admin:approved` | Approved | Money released within ~1 minute; order item → `completed`. |
| Brand: "Published" (if content exists) or "Failed" / Influencer: "Published & Waiting Approval" or "Failed" | job status `failed` or `expired`, `url_approval` empty | Timer ran out, not yet refunded | Refund at start + 3 days. |
| "Failed" (both sides) | offer `refunded`, order item `refunded` | Brand refunded | Brand can still approve (re-charges wallet) or reschedule (OFR-04). |
| "Disapproved by Admin" / Influencer: "Rejected by Admin" | `url_approval=admin:disapproved` | Admin rejected the delivery | Already refunded. |
| Brand disapproved | `url_approval=brand:disapproved`, status `failed` | Brand rejected the delivery | Refund at start + 14 days unless the influencer resubmits or the brand approves. |

Note: the brand-side label stays "Published" whenever any content was submitted, even after the job failed, so a brand may not notice the deal is about to be refunded.

## Timers and automatic actions

All run every minute (UTC). Only the first three affect current deals:

| Job | Condition (verified in code) | Effect |
|---|---|---|
| `publish:jobs:expiry-check` | job `scheduled` + `healthy`, manual, start + 2 days passed, old `published_url` field empty, not admin-approved | Status → `expired`; "publishing failed" notification. **Current submissions never fill `published_url`, so submitted deals are caught too.** |
| `publish:jobs` (pending-approval part) | `url_approval=pending`, `published_url` empty, state not `published`/`completed`, start + 48 hours passed | Status → `failed`, `url_approval` cleared. **This is what fails submitted-but-unapproved deals.** |
| `modules:publish_jobs:refund` | status `expired`/`failed`, state `scheduled`/`published`, offer `paid`, order item `transferred`, and either (`url_approval` contains "disapproved" and start + 14 days) or (`url_approval` empty and start + 3 days) | Refunds the brand, offer → `refunded`, notifications. |
| `modules:publish_jobs:finalize` | job `healthy`, state `published`/`completed`, order item `transferred`, and approved; *or* legacy auto-release branches (see below) | Releases the money to the influencer. |

**Legacy auto-release branches (not reachable for current deals).** The release job also has branches for "link submitted, no approval, 14 days after start", "no link, no approval, 2 days after start" and "pending approval 14 days after start". They all require the job to be in state `published` or `completed`, and the first two also look at the old `published_url` field. Current deals stay in state `scheduled` until approved and store links in a separate table, so **these branches never release money for current deals**. They only applied to the old automatic publisher (pre-2025 jobs).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Money released | Influencer: "Funds Added to Wallet!" / "Your Funds for campaign with {brand} are now in your wallet!" | push, in-app, email ("Your post was successfully published!") |
| Money released | Brand: `brand.publish-job:transferred` template, plus a chat update | email, push, in-app (if the template exists) |
| Deal expired by timer | Influencer: "Content failed to be published!" | push, in-app, email ("Your post failed to deliver.") |
| Refund | Influencer: "Funds will be refunded to {brand}!" | email, in-app |
| Refund | Brand: `brand.publish-job:refunded` template, plus a chat update | email, push, in-app (if the template exists) |
| Admin disapproval | Brand gets the refund notice twice (once from the refund, once from the disapproval); influencer gets a disapproval email and the refund notice | email, push, in-app |

## Common questions

### "I posted and submitted my link. Why did my deal fail and the brand get refunded?"
**Answer for the user:** Payment is released only after the brand or the Ainfluencer team approves the delivery. If no one approved it in time, the system marks the deal as failed and returns the money to the brand. We'll review your delivery.
**What to check:** the publish job's `start`, `status`, `url_approval`, `approved_at`; the `published_contents` rows (URL, `content_id`, `meta.error`); the order item status (`transferred` or `refunded`); the offer status.
**Escalate when:** the link was valid and delivered on time. An admin can approve the delivery (this re-charges the brand wallet if already refunded) or decide on a manual payout. Escalate to finance/dev if the brand wallet can't cover it.

### "The brand approved, when do I get paid?"
**Answer for the user:** Within a few minutes of approval the money moves to your wallet as available balance. Withdrawals are then requested from the wallet and paid by our team.
**What to check:** `url_approval` is `brand:approved`/`admin:approved`, state `completed`, status `healthy`, `busy=0`; order item status (`completed` means released).
**Escalate when:** approved more than 15 minutes ago and the order item is still `transferred`.

### "Why was I (the brand) refunded? I didn't ask for it."
**Answer for the user:** The delivery wasn't approved in time (or the influencer didn't deliver), so the payment was automatically returned to your Ainfluencer wallet. If you're happy with the post, you can still approve it from the inbox; the offer price will be taken from your wallet again.
**What to check:** timeline of `start`, submissions, approval; wallet transaction "refund".
**Escalate when:** the brand says they approved before the refund time.

### "I disapproved the post. When do I get my money back?"
**Answer for the user:** Brand disapprovals are refunded automatically 14 days after the deal's start date, giving the influencer a chance to fix the post. If the influencer submits a new link before then, the deal goes back to waiting for approval.
**What to check:** `url_approval=brand:disapproved`, `start` (it may have been pushed by 7 days when content was submitted).
**Escalate when:** more than 14 days after start and no refund, or the brand needs an immediate refund (admin disapproval refunds immediately).

### "The refund went to my wallet, can I get it on my card?"
**Answer for the user:** Refunds go to your Ainfluencer wallet and can be used for new offers. See PAY-02 for card refunds.
**Escalate when:** the brand insists on a card refund (finance team).

### "Why is the refund smaller than what I paid?"
**Answer for the user:** The platform fee isn't refunded, and any coupon or credit you used goes back to Ainfluencer rather than to your wallet.
**What to check:** the order's coupon and credit amounts (PAY-05).

## Known issues

- **Unapproved real posts are refunded.** Confirmed in code: the old `publish:jobs` timer and the expiry check look at `publish_jobs.published_url`, which the current flow never fills (links live in `published_contents`). So a submitted-but-unapproved deal is marked failed 48 hours after its start date and the brand is refunded 3 days after start. Because successful link fetching and file uploads push the start date by 7 days, the effective approval window is about 9 days after submission; for links that can't be fetched it can be under 2 days. The release job's "auto-release after 14 days" rule never applies to current deals. Workaround: admin or brand approval before the refund, or approval after the refund (re-charges the brand wallet). **Escalate to dev** for a fix; production impact not checked against data.
- The brand still sees "Published" and the influencer "Published & Waiting Approval" after the job has failed, so neither side is warned before the refund.
- When both expiry timers fire together the influencer gets a "Content failed to be published!" notification even though they submitted content.
- Influencer extend-deadline (OFR-04) or brand "reschedule after failure" on an already refunded deal can set the offer back to Paid/Scheduled without a new escrow payment, so the timers can never release or refund it; only approval (which re-charges) settles it. *(to verify)*

## Related topics

- PUB-01: submitting the link or files (starts the timers).
- PUB-02: approve and disapprove (the only release trigger today).
- OFR-03: payment that creates the escrow; OFR-04: extending or rescheduling dates; OFR-05: revisions closed at release/refund.
- PAY-02: refunds to the brand; PAY-03: influencer wallet; PAY-04: payouts; PAY-05: coupons and credits.

## Technical reference

- **Approve / disapprove:** `POST /api/{locale}/offer/v2/{offerId}/approve` and `/disapprove` → `app/Modules/Offer/Http/Controllers/OfferController.php` (approve re-orders a refunded offer via `OrderPaidEvent`; admin disapprove calls `OrderService::refund` immediately).
- **Release:** `app/Modules/PublishJob/Console/Commands/FinalizePublishedJob.php` (max 2,000 per run) → `Jobs/FinalizePublishJob.php` → `OrderItemCompletedEvent` → `app/Modules/Finance/Listeners/OrderItemCompletedListener.php` (influencer lock → default wallet, order item `completed`) + `PublishJobFinalized` → `Listeners/PublishJobFinalizedListener.php` (rejects open revisions, `publish-job:transferred` notification).
- **Refund:** `app/Modules/PublishJob/Console/Commands/RefundPublishedJob.php` (max 1,000 per run) → `Jobs/RefundPublishJobJob.php` → `OrderService::refund` → `app/Modules/Finance/Listeners/TransferRefundOrderTransactionsListener.php` (coupon/credit to system user, rest to brand default wallet, `InfluencerRefundNotification`) + `Listeners/PublishJobRefundedListener.php` (offer → `refunded`).
- **Failure timers:** `app/Console/Commands/PublishJobExpiryCheck.php` → `app/Jobs/PublishJob/ExpirePublishJobJob.php`; `app/Console/Commands/PublishJobs.php::handlePendingPublishJobs` → `app/Jobs/ProcessPendingPublishJob.php`. Scheduled in `app/Console/Kernel.php` (every minute, UTC).
- **Start-date pushes:** `app/Services/Implementation/Admin/PublishedContentService::store` (after a link is fetched) and `app/Services/Implementation/PublishedContentService::storeFiles` (file upload) set `start = now + 7 days`; `app/Modules/PublishJob/Services/PublishedContentService::submitPublishedUrl` sets it only when the job was failed/expired.
- **Data:** `publish_jobs` (`start`, `state`, `status`, `url_approval`, `approved_at`, `disapproved_at`, `reviewer_id`, `busy`, `order_item_id`, legacy `published_url`/`published_content_id`); `published_contents` (`url`, `content_id`, `meta`); `order_items` (`status`: transferred → completed/refunded); `offers.status` (`paid`, `refunded`).
