---
id: PUB-01
title: Scheduling and submitting the post
domain: Content delivery and settlement
audience: [influencer, brand, system]
apps: [mobile, web, background]
related: [OFR-03, OFR-04, OFR-05, PUB-02, PUB-03, PUB-04, CHAT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# PUB-01 · Scheduling and submitting the post

> Once a brand pays for an offer, a delivery record (a "publish job") is created for the agreed publish date. The influencer posts on their own social account and then submits the post link(s) and/or uploads the content in the app, so the brand and Ainfluencer can check and approve it.

## Who and where

- **Influencer (mobile app):**
  - Deal chat (CHAT_SCREEN) offer card with a countdown ("Due to publish" / "Past Due"), the buttons "Submit Post URL" and "Extend Deadline", and "Campaign Delivery Terms". "Submit post URL" is also in the chat menu.
  - Scheduler tab (SCHEDULER_SCREEN) with tabs "Scheduled", "Published", "Failed". Card menu: "View in Instagram" / "View in TikTok", "View Offer", "View Campaign".
- **Brand (web app):**
  - `/scheduler` ("Scheduler"): a calendar of deals per day (counts of posts, stories, reels) and a list of upcoming deals; each row has "View Offer", "Preview Campaign", "View Feed" and "Analytics" (opens `/publish-jobs/:id/analytics`, see PUB-04).
  - Delivered content appears in the inbox chat (`/new-offersV2/:id`) as "Final Delivered Result" (PUB-02).

## How it works

1. **Brand pays** (OFR-03). The system creates the publish job with the agreed publish date as its start date and holds the money in escrow (PUB-03). The influencer gets an "offer paid" email/push and the deal shows "Paid & Scheduled" in chat.
2. **Reminders.** At payment time the system schedules reminder pushes at 72, 24, 12, 6 and 3 hours before the publish date (only the ones still in the future): "Reminder: {N} hours left to deliver the campaign for {brand}."
3. **Influencer creates and posts** the content on their own account, following the brand's terms (caption, link in bio, date). The chat card's "Steps to Follow" explain this; it ends with: "Note: Ainfluencer admin will check all the details and verify them. The funds then will be released if you followed the requirements."
4. **Influencer submits the delivery** from the card:
   - "Submit Instagram Post" / "Submit TikTok Post" / "Submit YouTube Post" / "Submit Post URL": paste one or more post links, then "Submit URL". Success toast: "Your Post URL submitted successfully".
   - "Upload Content": for content-only deals, upload images/videos from the phone.
   - The influencer can later edit links ("Edit URL") or add/delete files ("Add More Images/Videos"), but at least one item must stay: "You can't delete all content from this offer. At least one published content must remain available…"
5. **What the system does after submission:**
   - Saves each link, marks the delivery "waiting approval" and closes any open revision request on the offer (OFR-05).
   - Posts a "published content" message in the deal chat so the brand sees it.
   - Tries to fetch the post (Instagram, TikTok, YouTube) to save a copy of the media and caption. If it works, **the deal's start date moves to 7 days from now** (this gives time for approval, see PUB-03). If it fails, the link is kept with an error note ("The influencer link submitted is invalid. Please review and edit it.").
   - Schedules post-performance snapshots (PUB-04).
   - For file uploads, the start date also moves to 7 days from now.
6. **Approval** by the brand or admin follows (PUB-02), and the money is released (PUB-03).

## Rules and limits

- Up to **5 links** per submission. Each must be a reachable URL; Instagram, TikTok and YouTube links must match the platform's post format, otherwise "The URL format is invalid for Instagram." (or TikTok / YouTube). The app also blocks duplicates ("Duplicate URL. Please enter a unique URL.").
- Links can't be changed after the delivery is approved: "You can't change the URL while it's approved."
- Submitting is possible while the deal is paid, including after it failed (the card says "You are late but you can still publish the post & submit it's URL!"). Submitting after a failure revives the deal and moves the start date 7 days ahead.
- No link or files **2 days after the publish date** → the deal expires ("publishing failed") and the brand is refunded at 3 days (PUB-03).
- The influencer can ask for a later date with "Extend Deadline" (OFR-04). The new date must be in the future.
- There is no cancel/refund button for influencers.

## Statuses

| Influencer sees (chat card) | Scheduler tab | Brand sees | Meaning |
|---|---|---|---|
| "Paid & Scheduled" | Scheduled | "Scheduled" | Paid, nothing submitted yet |
| "Published & Waiting Approval" | **none** (see Known issues) | "Published" | Link or files submitted, waiting for approval |
| "Published & Approved" (shows paid amount, "See Payout") | Published | "Published" | Approved; money released |
| "Failed" (with "Extend Deadline" and "Submit Post URL" while still paid) | Failed | "Failed" (or "Published" if content was submitted) | Expired or failed by timer, or disapproved |

Internal: `publish_jobs.state` (`scheduled`, `completed`), `status` (`healthy`, `failed`, `expired`), `url_approval` (`pending` after submission).

## Timers and automatic actions

- Reminder pushes: 72/24/12/6/3 hours before the publish date, only while the job is scheduled and healthy.
- Expiry: 2 days after the start date with no delivery → expired, "Content failed to be published!" (`publish:jobs:expiry-check`, every minute, UTC).
- After a delivery is submitted, the approval window and the automatic refund are described in PUB-03.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Offer paid | Influencer ("Your offer was paid for by {brand}!") | email, push, in-app |
| 72/24/12/6/3 h before publish date | Influencer ("Reminder: {N} hours left to deliver the campaign for {brand}.") | push |
| Link/files submitted | Brand (published-content chat message) | in-app chat |
| No delivery by start + 2 days | Influencer ("Content failed to be published!"); brand (publishing-failed template) | push, in-app, email |

Push taps: `publish-job:published` and `publish-job:failed` open the Scheduler; reminders open the deal chat.

## Common questions

### "I submitted my link but the deal disappeared from my Scheduler."
**Answer for the user:** Your link was received and is waiting for approval. Deals waiting for approval don't appear in the Scheduler tabs; you'll find the deal in your inbox chat with the status "Published & Waiting Approval".
**What to check:** `url_approval=pending`, `state=scheduled`, `status=healthy`, and a `published_contents` row with the URL.
**Escalate when:** no `published_contents` row exists although the user saw the success toast.

### "The app says my URL format is invalid."
**Answer for the user:** Please paste the direct link to the post (for example the link from "Share → Copy link" on the post itself), not your profile link.
**What to check:** the URL against the platform; Instagram stories and profile links are common mistakes.
**Escalate when:** a normal post link is rejected.

### "I'm late. Can I still deliver?"
**Answer for the user:** Yes, as long as the deal is still paid. Publish the post and tap "Submit Post URL", or use "Extend Deadline" to set a new date first.
**What to check:** offer status `paid` (not `refunded`); order item `transferred`.
**Escalate when:** the deal was already refunded to the brand (the influencer can't get paid without brand/admin action).

### "I keep getting reminders although I already submitted."
**Answer for the user:** The reminders were scheduled when the brand paid and may still arrive. Your submission is saved; you can ignore them.
**What to check:** that the link is in `published_contents`.

### "The brand says they can't see my post."
**Answer for the user:** We'll check your link. Please also share it in the chat with the brand.
**What to check:** `published_contents.meta.error` (fetch failed), the URL itself; whether the post is private or deleted.
**Escalate when:** the link is valid but the brand's inbox shows no "Final Delivered Result".

### "Can I deliver content without posting it?"
**Answer for the user:** If your deal is only for creating content, use "Upload Content" to send the files instead of a link.

## Known issues

- **Deals waiting for approval vanish from the influencer's Scheduler.** "Scheduled" only lists deals with nothing submitted, "Published" only approved deals. Workaround: open the deal from the inbox.
- **Reminder pushes keep coming after submission.** The reminder check looks at an old field that current submissions don't fill. Workaround: ignore.
- **Reminders depend on email.** They are scheduled while building the "offer paid" email, so an influencer who unsubscribed from emails may get no reminders. *(to verify)*
- Tapping a publish-job notification from the in-app notification list always opens the "Scheduled" tab, not the right tab.
- After submitting, the influencer's approval window is limited; unapproved deliveries are refunded to the brand (PUB-03).
- Submitting a link whose post can't be fetched (stories, private posts) doesn't move the start date, so the deal can fail within hours if submitted late. *(see PUB-03)*

## Related topics

- OFR-03 payment creates the publish job; OFR-04 extend deadline / change date; OFR-05 revisions closed on submission.
- PUB-02 approval; PUB-03 release and refunds; PUB-04 post analytics.
- CHAT-01 messaging in the deal chat.

## Technical reference

- **Endpoints:**
  - `POST /api/{locale}/v3/mobile/deals/{publishJob}/url` (`published_urls[]`, max 5) → `app/Modules/Mobile/Http/Controllers/API/PublishJobController@submitPublishUrl` → `app/Modules/PublishJob/Services/PublishedContentService::submitPublishedUrl` → `PublishedUrlUpdated` → `Listeners/PublishedUrlUpdatedListener` (chain: `SendPublishedContentMessage`, `StorePublishedContent` per URL, `SchedulePublishedContentAnalyticsJobs`).
  - `POST|DELETE /api/{locale}/v3/mobile/deals/{id}/published-contents/files` → `app/Modules/Mobile/Http/Controllers/API/PublishedContentController` → `app/Services/Implementation/PublishedContentService::storeFiles / deleteFiles`.
  - `GET /api/{locale}/v3/mobile/deals/{scheduled|published|failed}` → `PublishJobController@index` → `app/Repositories/v2/PublishJob/PublishJobRepository::getScheduledDeals`.
  - `GET /api/v2/publish-jobs` (web always sends `status=pending` = scheduled + healthy), `GET /api/v2/publish-jobs/{id}`, `GET /api/v2/schedules` → `app/Http/Controllers/v2/Frontend/PublishJobController` → `app/Services/Implementation/PublishJobService`.
  - Legacy publisher webhooks `POST /api/webhook`, `/api/webhook/publish-job/{id}`, `/api/webhook/instagram/publish-job` (old automatic publisher only).
- **Creation:** `app/Modules/Offer/Listeners/OfferPaidListener::getPublishJobDto` (start = offer start, `manual=1`, `scheduled`/`healthy`).
- **Reminders:** `app/Modules/Mobile/Notifications/Offer/OfferPaidNotification::toMail` → `app/Jobs/PostReminderJob` (delays from `app/Modules/Payment/Traits/PublishPostReminder`).
- **Data:** `publish_jobs` (`start`, `state`, `status`, `url_approval`, `extra.dead_line_extend_requests`), `published_contents` (`publish_job_id`, `url`, `content_id`, `meta.error`, `published_at`), `contents`/`medias` for uploaded files.
- **Mobile:** `features/chat/components/modals/OverlayPostUrl`, `features/scheduler/SchedulerScreen.tsx`, labels in `localization/translations/en/chat.js`.
- **Web:** `components/pages/scheduler`, `lib/services/scheduler.ts`.
