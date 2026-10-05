---
id: PUB-02
title: Approving or disapproving content
domain: Content delivery and settlement
audience: [brand, admin, influencer]
apps: [web, mobile, email]
related: [PUB-01, PUB-03, PUB-05, PUB-06, PUBL-03, BRD-03, PAY-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# PUB-02 · Approving or disapproving content

> When an influencer delivers a post or files, the brand (or an Ainfluencer admin) approves or disapproves it. **Approval is what releases the influencer's payment.** This topic also covers where brands view, download and share delivered content.

## Who and where

- **Brand, web inbox** `/new-offersV2/:id`: the chat shows a "published content" message and the sidebar shows "Final Delivered Result" with a phone-style preview of the post ("View Post", "Download", "Preview", "Copy" share link). Below it, while nobody has reviewed the delivery yet: "Let our Admin team know if you approve this collaboration:" with **"Approve it"** and **"Disapprove it"**.
- **Brand, content library** `/influencer-content` ("Contents"): all delivered content with "View details", "Download", "View live post", "Translate" (PUB-06) and "Copy share link". Content status shows "Approved", "Pending" or "Draft". **There is no approve or disapprove button on this page today.** Team members can't open this page (they are sent to the dashboard).
- **Public share link** `/view-contents/:code`: anyone with the link can view the delivered content (PUBL-03).
- **Admin**: approves or disapproves from the admin panel.
- **Influencer, mobile**: sees the outcome on the deal card ("Published & Waiting Approval", "Published & Approved", "Failed", "Rejected by Admin") and receives a push or email.

## How it works

1. The influencer submits a link or files (PUB-01). The brand sees the delivery in the inbox.
2. The brand clicks **"Approve it"** or **"Disapprove it"**. A pop-up asks for an optional note: "Let our Admin (Ainfluencer's support team) know what you think about {influencer}" with "This is to be sent ONLY to our admin and NOT to the influencer." The disapprove pop-up adds: "Note: Continue communicating with influencer to ask them to correct the content and republish it as you wish." Then **"Submit"**.
3. **On approve:** the delivery is marked approved, the influencer gets "Campaign delivery was approved by {brand}." and the money is released within about a minute (PUB-03). If the offer had already been refunded to the brand, approving takes the offer price from the brand's wallet again first.
4. **On disapprove (brand):** the delivery is marked disapproved and failed. The money stays held; the brand is refunded automatically 14 days after the deal's start date (PUB-03). The influencer gets an email about the disapproval. If the influencer submits a new link before the refund, the deal goes back to "waiting approval".
5. **On disapprove (admin):** the brand is refunded **immediately**, the influencer gets a "disapproved by Ainfluencer Team" message in chat and an email, and the brand's chat shows "The delivery of the offer was disapproved by Ainfluencer support."

## Rules and limits

- Only the brand that owns the offer or an admin can approve/disapprove.
- The brand can approve or disapprove only while the deal is still in the "scheduled" stage (before approval). After approval: "You can't update an unscheduled offer."
- The brand **can approve without a delivery** (nothing forces a link first) and can approve a deal that already shows Failed.
- The brand can't disapprove after approving: "You can not disapprove an offer that is approved before." and can't disapprove once money is released: "You cannot disapprove this offer, the money has been transferred to the influencer's wallet."
- If an admin already approved or disapproved, the brand can't change it: "URL has been approved or disapproved by an administrator."
- Approving a refunded offer needs enough wallet balance for the offer price (the fee isn't charged again); otherwise the server answers "You currently do not have enough funds in your wallet."
- The note is optional and goes to Ainfluencer only.
- Once anyone has reviewed (approved or disapproved), the "Approve it / Disapprove it" box disappears from the brand's inbox.

## Statuses

| Brand sees | Influencer sees | Internal `publish_jobs.url_approval` | Meaning |
|---|---|---|---|
| "Published" | "Published & Waiting Approval" | `pending` | Delivered, waiting for review |
| "Published" | "Published & Approved" | `brand:approved` / `admin:approved` | Approved, money released |
| "Published" (if content exists) or "Failed" | "Published & Waiting Approval" or "Failed" | `brand:disapproved` | Brand disapproved; refund 14 days after start |
| "Disapproved by Admin" | "Rejected by Admin" | `admin:disapproved` | Admin disapproved; brand refunded |

The brand-side label is "Published" for both waiting and approved deliveries; check the approve box or ask support for the internal value.

## Timers and automatic actions

- There is **no automatic approval**. If nobody approves, the deal fails and the brand is refunded (about 10 days after a readable link or file upload; see PUB-03 for exact timers).
- Brand disapproval → refund at start + 14 days (`modules:publish_jobs:refund`).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Brand approves | Influencer: "Campaign delivery was approved by {brand}." / "Delivery of your campaign with {brand} was accepted by them." | push, in-app; chat event |
| Admin approves | Influencer (email "post approved by admin"); brand (approved-by-admin notification) | email, push, in-app; chat event |
| Brand disapproves | Influencer (disapproval email with the brand's reason) | email only |
| Admin disapproves | Influencer (email + chat event); brand (disapproval + refund notices) | email, push, in-app |
| Money released | See PUB-03 | |

## Common questions

### "Where do I approve the influencer's post?"
**Answer for the user:** Open the chat with the influencer in your inbox. Under "Final Delivered Result" you'll find "Approve it" and "Disapprove it". Approving releases the payment to the influencer.
**What to check:** the job has a submission and `reviewer_id` is empty.
**Escalate when:** the box doesn't appear although the delivery is unreviewed.

### "I approved by mistake. Can I undo it?"
**Answer for the user:** Approval releases the payment within minutes and can't be reversed by you. Please tell us what happened and we'll review it.
**Escalate when:** always (finance/admin decision).

### "I disapproved. What happens to my money?"
**Answer for the user:** The money stays on hold so the influencer can fix the post. If nothing changes, it's refunded to your Ainfluencer wallet 14 days after the deal's start date. Keep talking to the influencer in the chat.
**What to check:** `url_approval`, `disapproved_at`, `start`.
**Escalate when:** the brand needs the money back immediately (an admin disapproval refunds at once).

### "The brand disapproved my post. Will I lose the payment?" (influencer)
**Answer for the user:** Not yet. Talk to the brand in the chat, fix the post and submit the new link from the deal card. The deal will go back to waiting for approval.
**What to check:** refund not yet done (order item `transferred`).

### "Why can't I approve? It says I don't have enough funds."
**Answer for the user:** This deal was already refunded to your wallet, so approving it takes the offer price again. Add funds to your wallet and try again.
**What to check:** offer status `refunded`; brand wallet balance.

### "My team member can't see Contents."
**Answer for the user:** The Contents page is available to the account owner only. Team members can still see and approve deliveries in the inbox chat.

### "How do I share the delivered content with my team or agency?"
**Answer for the user:** Use "Copy share link" on the Contents page or "Copy" in the chat preview. Anyone with the link can view it.

## Known issues

- **No automatic approval**: deliveries not approved in time are refunded to the brand, and the brand-side status still says "Published" until then (PUB-03). Escalate to dev.
- The Contents page (`/influencer-content`) has no approve button although its "Approved" counter suggests otherwise.
- Brand disapproval only emails the influencer; no chat message is posted. *(to verify)*
- After a brand disapproval the review box disappears, so the brand can't switch to "Approve it" from the inbox even though the backend would allow it. Workaround: ask an admin to approve.
- The admin-disapproval refund notification reaches the brand twice.
- If an approval or disapproval is refused by the server (for example not enough wallet balance on a refunded deal), the inbox closes the pop-up and hides the box without showing the error; reloading the chat brings the box back. Check the job in data before telling the brand it worked.

## Related topics

- PUB-01 delivery; PUB-03 release and refund timers; PUB-05 review after the deal; PUB-06 translating delivered videos.
- PUBL-03 public content view; BRD-03 team members; PAY-02 refunds.

## Technical reference

- **Endpoints:**
  - `POST /api/{locale}/offer/v2/{offerId}/approve` and `/disapprove` (optional `reason`) → `app/Modules/Offer/Http/Controllers/OfferController@approve|disapprove`; jobs `app/Jobs/ApprovedBrand.php`, `ApprovedAdmin.php`, `DisApprovedBrand.php`, `DisApprovedAdmin.php`.
  - `GET /api/v2/published-contents`, `/published-contents/influencers/{integrationId}`, `/published-contents/offers/{encodedOfferId}` (also the public view), `/published-contents/download/{contentId}` (zip), `/published-contents/download/{contentId}/files/{fileId}`, `GET /api/v2/guest/published-contents` → `PublishedContentController`; `GET /api/v2/influencer-generated-contents` → `MediaController@influencerGeneratedContents`.
- **Web:** `components/common/mobileFrame` (approve box shown when `reviewer_id` is empty; `parts/modalApproved`), `components/pages/inboxV37/InboxV37StatusBlocks.tsx` (DeliveredBlock), `components/pages/contents2` (`handleApprove` exists but is not rendered), `components/pages/viewContentsNoAuth`.
- **Data:** `publish_jobs.url_approval`, `brand_url_approval`, `reviewer_id`, `approved_at`, `disapproved_at`, `feedback` (the note), `state`, `status`; `published_contents`.
- **Labels:** brand status from `app/Modules/OfferOverallStatus/OfferStatuses/Paid.php` and `Refunded.php`.
