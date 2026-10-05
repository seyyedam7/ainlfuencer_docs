---
id: PUB-05
title: Reviews and ratings
domain: Content delivery and settlement
audience: [brand, influencer, admin, system]
apps: [web, email, mobile, background]
related: [PUB-02, PUB-03, PUB-04, PUBL-02, DSC-01, BRD-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# PUB-05 · Reviews and ratings

> After a deal is delivered (or fails), the brand can rate the influencer on three criteria and write a public review. Reviews are checked by an Ainfluencer admin before they appear on the influencer's profile, and they feed the influencer's trust score.

## Who and where

- **Brand, web app:**
  - Inbox chat `/new-offersV2/:id`: **"Write a review"** button on the deal.
  - `/analytics`: **"Write a review for this Influencer"** under "Performance by Influencer" (PUB-04). The dashboard (`/`) also offers it.
  - Guest review page `/review/:code`, opened from the review email (no login needed).
  - `/new-offersV2/reviews` ("Ratings & Reviews"): list of reviews the brand has written, with "Your Rating and Review" details. Not linked from the menu; reachable by direct URL. `/profile/points` also lists reviews given.
- **Admin:** approves, rejects or resets reviews in the admin panel.
- **Influencer, mobile:** only a notification when a review is approved. Approved reviews show on the influencer's profile that brands see (`/account/:id`) and the public profile (PUBL-02).

## How it works

1. **When the button appears.** "Write a review" shows when the deal's status is "Published", "Failed" or "Disapproved by Admin" and the brand hasn't reviewed it yet.
2. **The form** ("Your Rating and Review") shows the influencer, campaign and published date, then asks for:
   - "Performance" (stars): "Was {name} useful in terms of your objectives and performance you liked to achieve?"
   - "Communication" (stars): "How clearly did {name} communicate their plans, questions, and collaborations?"
   - "Observance of agreed terms and rules" (stars): "Did {name} observe your terms and rules such as date to publish, using certain hashtags, or putting the link in Bio?"
   - "Would you work with {name} again?" Yes/No: "We won't publish your answer to this question anywhere, and {name} won't see it."
   - "Leave a public review" (text, up to 5,000 characters).
   Then "Submit". Success: "Your review has been submitted successfully".
3. **Admin moderation.** The review is saved as **pending**. An admin approves or rejects it. On approval the influencer is notified, the review appears on their profile with the average of the three star ratings, and the influencer's review statistics are recalculated.
4. **Reminder emails.** For finished deals the system sends up to three "Write a review for {influencer}" emails with a link to `/review/{code}` (see Timers, and Known issues).

## Rules and limits

- One review per deal. A second one is refused ("already submitted"); the guest link then says "You just created the review for the influencer {time}!".
- All three star ratings, the yes/no answer and the text are required (the form says "Please select a rate" / "Please select an option").
- Ratings are out of 5.
- Only approved reviews appear publicly. A rejected review stays hidden; an admin "reset" lets the brand submit again with the same link.
- The "work again" answer is never shown to the influencer.

## Statuses

| Status (`reviews.status`) | Meaning | Visible to |
|---|---|---|
| `pending` (no `code_used_at`) | Review placeholder created for the reminder email, not filled in | nobody |
| `pending` (with `code_used_at`) | Brand submitted, waiting for admin | brand (in "Ratings & Reviews") |
| `approved` | Admin approved | brand, influencer profile, public profile |
| `rejected` | Admin rejected | brand only |

Deal-level label (internal `review_status_string`): "Waiting Review" → button shown; "Reviewed" → button hidden.

## Timers and automatic actions

- `review-email:send` runs every minute (UTC). It picks up publish jobs that have been approved or disapproved and have no submitted review, and queues three reminders: now, +96 hours and +168 hours. Each reminder is sent only if the deal is at least 24 hours old, no review exists yet, and that reminder wasn't already sent.
- `integration:update-trust-metric` on the 1st and 16th of each month recalculates the influencer trust metric from offers, deliveries and reviews.
- On admin approval, the influencer's aggregated review data is refreshed straight away.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Deal approved/disapproved, no review yet | Intended for the brand: "Write a review for {influencer}" (3 times) | email (see Known issues) |
| Admin approves a review | Influencer (review submitted by brand) | email, push, in-app |

The influencer's push for an approved review opens the Explore tab in the app, not the review.

## Common questions

### "Where can I leave a review?" (brand)
**Answer for the user:** Open the deal in your inbox and click "Write a review". You can also review from the Analytics page under "Performance by Influencer".
**What to check:** the deal status is Published, Failed or Disapproved by Admin; no review with `code_used_at` exists for the offer.
**Escalate when:** the button is missing for a finished deal without a review.

### "I submitted a review but it's not on the influencer's profile."
**Answer for the user:** Reviews are checked by our team before they're published. It will appear once approved.
**What to check:** `reviews.status` for the offer (`pending`, `approved`, `rejected`), `approved_at`, `admin_note`.
**Escalate when:** pending for more than a few days (admin moderation queue).

### "Can I edit my review?"
**Answer for the user:** Reviews can't be edited after submitting. If something's wrong, tell us and our team can reset it so you can submit again.
**Escalate when:** the brand asks for a change (admin uses "reset").

### "I got a bad review. Can it be removed?" (influencer)
**Answer for the user:** Every review is checked before it's published. If you think a review breaks our rules or is unfair, send us the details and our team will look at it.
**Escalate when:** always (admin decision).

### "The review link says I already reviewed."
**Answer for the user:** A review for this deal was already submitted on the date shown. Each deal can be reviewed once.

### "Does the influencer see my 'work again' answer?"
**Answer for the user:** No. It's only used by Ainfluencer.

## Known issues

- **Review reminder emails go to the wrong person.** The email is written for the brand ("Write a review for {influencer}") but the code sends it to the user of the reviewed influencer account. Brands therefore likely never receive the reminder. **Escalate to dev.**
- **Some deals never get reminders.** The reminder job only moves forward through deals in ID order; a deal that is approved after a newer deal was already processed is skipped. *(to verify)*
- The influencer's "review approved" push opens Explore instead of anything review-related.
- `/new-offersV2/reviews` has no menu link; give brands the URL directly.

## Related topics

- PUB-02 approval/disapproval makes a deal reviewable; PUB-04 Analytics page entry point.
- PUBL-02 public influencer profile shows approved reviews; DSC-01 influencer search/profile; BRD-02 brand score (reviews given appear in `/profile/points`).

## Technical reference

- **Endpoints:**
  - `POST /api/v2/reviews` (`offer_id`, `review`, `performance`, `communication`, `rules_observation`, `collaborate_again`) and `GET /api/v2/reviews` → `app/Http/Controllers/v2/Frontend/ReviewController` → `app/Services/Implementation/ReviewService` (store creates the review through the guest service, then fills it).
  - `GET /api/v2/guest/reviews/{code}`, `PUT /api/v2/guest/reviews` (`code` + same fields), `GET /api/v2/guest/reviews/{code}/result` → `app/Http/Controllers/v2/Guest/ReviewController` → `app/Services/Implementation/Guest/ReviewService`.
  - Admin moderation: `PUT` admin reviews `{id}` (`action` = approve/reject/reset) → `app/Services/Implementation/Admin/ReviewService::update` (`UpdateReviewsData`, `ReviewSubmittedByBrandNotification`).
- **Reminders:** `app/Console/Commands/SendReviewEmail.php` (cursor cached under `send_review_emails_to_brand`) → `app/Jobs/PublishJob/SendReviewEmail.php` → `app/Mail/v2/ReviewCreatedEmail.php` (sent to `review->reviewee->user`); delays in `app/Helpers/Enums/Constants.php` (24 / 96 / 168 h); de-duplication via `EmailTracker`.
- **Trust metric:** `app/Console/Commands/Integration/UpdateInfluencerTrustMetric.php`.
- **Data:** `reviews` (`offer_id`, `reviewer_id` = brand identity, `reviewee_id` = influencer identity, `reviewee_integration_id`, `code`, `code_used_at`, `status`, `performance`, `communication`, `rules_observation`, `collaborate_again`, `review`, `approved_at`, `rejected_at`, `admin_note`).
- **Web:** `components/pages/inboxV37/InboxV37LeaveReviewDialog.tsx`, `components/pages/review`, `components/pages/reviewResult`, `components/pages/ReviewAndRating`; labels in `locales/en/singleOffer.json` (`submitReviewModal`).
