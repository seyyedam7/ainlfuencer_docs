---
id: PUB-04
title: Insights and post analytics
domain: Content delivery and settlement
audience: [brand, influencer, system]
apps: [web, mobile, background]
related: [PUB-01, PUB-02, PUB-05, INF-02, INF-06, CHAT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# PUB-04 · Insights and post analytics

> Two different things live here: **insights** are screenshots of an influencer's own audience statistics (from Instagram Insights, TikTok Analytics or YouTube Analytics) that a brand can request in the chat; **post analytics** are likes, comments, views and saves that Ainfluencer collects automatically from delivered posts and shows on the brand's Analytics pages.

## Who and where

- **Brand, inbox chat** `/new-offersV2/:id`: a chart button in the message box, tooltip "Request Influencer insights". The first click sends the request; later clicks open the influencer's insights.
- **Influencer, mobile deal chat**: an "Insight Request" bubble with "Upload & Submit Insights" (or "Upload & Submit Analytics" for TikTok/YouTube) and step-by-step help ("Here are the steps to submit the screenshots of your Instagram insights:"). Insights can also be attached from the message box.
- **Brand, Analytics** `/analytics` (left menu): "Campaign Performance at a Glance" per campaign: number of influencers who delivered, "Total Likes", "Total Comments", "Total Views", "Total Saves", "Performance by Influencer" (with "Write a review for this Influencer", PUB-05), and "Recommended Influencer for Maximum Impact" with "Invite".
- **Brand, single post** `/publish-jobs/:id/analytics` ("Post Analytics"), opened from the "Analytics" action in `/scheduler`: Likes, Comments, Engagement rate and Bio Link Clicks over time.

## How it works

**Insights (screenshots)**
1. The brand clicks "Request Influencer insights" in the chat. A notice confirms "Insights for {name} was requested." and a request bubble appears in the chat.
2. The influencer gets an "Insights Requested" push/email ("Submission of {platform} Insights for Collaboration with {brand} - a Request from the brand"). The bubble says: "The brand {brand} is requesting you to submit your audience`s insights for the last 90 days."
3. The influencer uploads screenshots and taps "Upload & Submit Insights". Success: "The insight data sent successfully."
4. The brand clicks the same button again to see the screenshots. If none were uploaded yet: "The Influencer has not yet submitted the Insights for {name}" with the note that the insights appear "either here on the right panel or by clicking on the same button".

**Post analytics (automatic)**
1. When the influencer submits a post link (PUB-01), Ainfluencer schedules automatic snapshots of that post: about **1 hour, 1 day, 3 days and 1 week** after submission (plus one around the deal's start date). For posts with a limited duration, snapshots are spread over that duration.
2. Each snapshot reads the public numbers (likes, comments, views, shares, saves, caption) from Instagram, TikTok or YouTube and stores them.
3. `/analytics` adds these up per campaign and per influencer and draws simple charts.

## Rules and limits

- **One insight request per offer.** A second request returns "You have already sent this request."
- Insights belong to the **influencer's account, not to the deal**: uploads are saved on the influencer's profile and every brand that requests them sees the same, latest set. New uploads are added to the existing ones; the influencer can delete individual files ("The Insight will be deleted!").
- Post analytics exist only for **links** on Instagram, TikTok or YouTube. Uploaded files and unreadable links (stories, private posts) have no numbers.
- `/analytics` lists a campaign only when it has a **paid** deal whose delivery was **approved** (by brand or admin), has at least one submitted link and at least one snapshot. Unapproved or failed deliveries don't appear.
- Numbers are only as fresh as the last snapshot; there's no live refresh button.

## Statuses

| What the brand sees | Meaning |
|---|---|
| Insight button sends the request | No request sent yet for this offer |
| "Insights requested" notice | Request sent, influencer notified |
| Screenshots shown | Influencer has uploaded insights (could be from an earlier request) |
| "No Published Campaigns Yet!" on `/analytics` | No approved, link-based deliveries with data yet |
| "No Data" on Post Analytics | No snapshots stored for that deal |

Internal: `insight_requests.status` (`requested` → `sent`), `insights` (per influencer), `publish_job_metrics` (one row per snapshot).

## Timers and automatic actions

- Snapshots at +1 h, +1 day, +3 days, +1 week after link submission (queue jobs, not a cron). If the scraper is temporarily down, the snapshot is retried later; if the post is deleted or private, that snapshot is skipped.
- Influencer profile data refreshes (followers, engagement) are separate (INF-06).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Brand requests insights | Influencer ("Insights Requested") | push, in-app, email (email skipped if unsubscribed); chat bubble |
| Influencer uploads insights | Brand | none from the server; the brand checks via the button *(to verify)* |

## Common questions

### "I asked for insights but nothing happened." (brand)
**Answer for the user:** The influencer has been notified and needs to upload screenshots from their app. Click the insights button again later to check, or remind them in the chat.
**What to check:** `insight_requests` row for the offer (`status`, `sent_at`); `insights` files for the influencer.
**Escalate when:** the status is `sent` but the brand sees nothing.

### "Can I request insights again?" (brand)
**Answer for the user:** Each deal allows one request. You can ask the influencer in the chat to upload newer screenshots; you'll see the latest ones with the same button.

### "Which screenshots should I upload?" (influencer)
**Answer for the user:** Screenshots of your audience statistics for the last 90 days (countries, cities, age, gender, reach). You need a creator or business account on Instagram to see insights. Follow the steps shown in the request bubble.

### "Why is my campaign not on the Analytics page?" (brand)
**Answer for the user:** Analytics appear after an influencer's post has been approved and we've collected its numbers. Deliveries that are still waiting for approval, failed, or delivered only as files don't show there.
**What to check:** `publish_jobs.url_approval` contains `:approved`; `published_contents.url` set; `publish_job_metrics` rows exist.
**Escalate when:** approved link deliveries with metrics still don't show.

### "The numbers look wrong or stopped updating."
**Answer for the user:** We take snapshots of public numbers during the first week after the post is submitted. If the post was made private, deleted, or is a story, we can't read it.
**What to check:** latest `publish_job_metrics.created_at` for the post; `published_contents.meta.error`.

### "Bio Link Clicks shows nothing."
**Answer for the user:** Link clicks aren't available for most posts. *(to verify)*

## Known issues

- Tapping the "Insights Requested" push in the mobile app opens Explore instead of the deal chat (the event isn't mapped). Workaround: open the chat from the inbox.
- Insights are shared across all brands and deals for that influencer; a brand may see screenshots uploaded for another brand's request.
- `/analytics` hides deliveries that were never approved, which (because of PUB-03) are also the ones refunded.

## Related topics

- PUB-01 link submission starts the snapshots; PUB-02 approval makes campaigns appear in `/analytics`; PUB-05 reviews from the Analytics page.
- INF-02 / INF-06 influencer profile data; CHAT-01 deal chat.

## Technical reference

- **Endpoints:**
  - `POST /api/v2/insight/request` (`offer_id`) and `GET /api/v2/insight/{offer_id}` → `app/Http/Controllers/v2/Frontend/InsightController` → `app/Services/Implementation/InsightService` (`InsightCreatedEvent` → `Listeners/InsightCreatedListener` → `app/Notifications/Insight.php`, event `events:mobile:offer:insight-requested`).
  - `POST /api/{locale}/v3/mobile/offers/{id}/insights` (`file_ids[]`, `offer_id`) and `DELETE /api/{locale}/v3/mobile/insights/{insightId}/files/{fileId}` → `app/Modules/Mobile/Http/Controllers/API/InsightController` → `app/Services/Implementation/Mobile/InsightService`.
  - `GET /api/v2/campaigns/published/analysis`, `…/{adId}/publishers`, `…/{adId}/publishers/{integrationId}`, `…/{adId}/influencers/recommended` → `app/Http/Controllers/v2/Frontend/PublishedCampaignAnalysisController` → `app/Services/Implementation/PublishedCampaignAnalysisService`, filter `app/Repositories/v2/Ad/AdRepository::getPublishedCampaignAnalysisBaseQuery`.
  - `GET /api/v2/publish-jobs/{id}` → single post analytics.
- **Snapshots:** `app/Modules/PublishJob/Jobs/SchedulePublishedContentAnalyticsJobs.php` → `StorePublishedContentAnalyticsJob.php`.
- **Data:** `insight_requests` (`offer_id`, `status`, `insight_id`, `sent_at`), `insights` (`identity_id`, `integration_id`) with files, `publish_job_metrics` (`publish_job_id`, `published_content_id`, `likes`, `comments`, `views`, `shares`, `saves`, `created_at`).
- **Web:** `components/pages/inboxV37/chat/InboxV37ChatSender.tsx`, `InboxV37InsightNoticeModal.tsx`, `components/pages/analytics`, `components/pages/publishJobs`. **Mobile:** chat insight bubble, labels in `localization/translations/en/chat.js` (`requestInsight`, `uploadInsightModal`).
