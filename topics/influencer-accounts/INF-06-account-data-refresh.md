---
id: INF-06
title: Account data refresh
domain: Influencer social accounts
audience: [system, influencer, brand]
apps: [background]
related: [INF-01, INF-03, INF-04, INF-05, DSC-01, PUB-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce
---

# INF-06 · Account data refresh

> Follower counts, posts and profile details of connected accounts are refreshed by background jobs, not live. Instagram accounts are refreshed in bulk once a month; influencers can also refresh one account themselves with "Sync" (INF-05). This explains "my numbers are out of date" and "my account was archived".

## Who and where

- Nobody clicks anything; results show on the influencer's Profile cards, in brand search and influencer profiles (DSC-01).

## How it works

1. **Monthly Instagram refresh** (production only):
   - Starts at **02:00 UTC from the 2nd of each month** (it checks every day and starts on the first free day if last month's run was still going).
   - First a trial batch of **500** random accounts is refreshed. Only if at least **70%** come back usable is the full run sent. If the trial fails, nothing more is refreshed until the dev team decides.
   - All live (not archived) Instagram accounts are then refreshed in batches of up to 5,000: followers, following, posts count, picture and the **latest 10 posts**. Accounts are looked up by their Instagram id, so a renamed account is found and its new username saved.
   - Accounts not found by id are retried by username. Accounts still not found are **archived** (removed from brand search). Safety valve: if more than **30%** of the run is "not found", nothing is archived and the dev team reviews.
   - Progress is checked every **15 minutes**. A full run takes days.
2. **Pending manual-verification requests** (Instagram): every **30 minutes** missing followers and profile id are filled in so admins can review them (INF-03).
3. **Other platforms**: TikTok and YouTube are not part of the monthly Instagram run. A slower monthly re-index (from the 1st) re-fetches profile info for accounts across platforms, throttled *(to verify how often each account is actually reached)*. YouTube is otherwise refreshed when the influencer taps Sync.
4. **Post data for deals**: a scraper calls back with TikTok video reports and post updates, which fill publish dates and metrics on delivered content (PUB-01).
5. **Manual sync** by the influencer: once every 24 h for Instagram/TikTok (INF-05).

## Rules and limits

- Archived accounts are skipped by later monthly runs; only an admin can restore them (INF-04).
- An account refreshed by the monthly run also starts the 24-hour manual sync cooldown.

## Statuses

Internal run statuses only (`crawlzo_runs`, `crawlzo_jobs`: pending, running, interrupted, processing, done, failed). Influencers see no status.

## Timers and automatic actions

| Job | When (UTC) | Effect |
|---|---|---|
| `crawlzo:refresh-instagram --scheduled` | Daily 02:00, acts once a month from the 2nd (production only) | Starts monthly Instagram refresh |
| `crawlzo:poll-jobs` | Every 15 min | Downloads results, applies updates, archives not-found accounts at the end |
| `integration:fill-temporary-from-scraper` | Every 30 min | Fills data for pending Instagram manual requests |
| `elastic:manual-index-integrations` | Monthly (1st) | Slow re-index of account info |
| `mongo:prune-temporary-posts` | Hourly at :25 | Keeps the newest 10 scraped posts per account |

## Notifications

None to users. Archiving sends nothing; un-archiving by an admin sends an email and push (INF-04).

## Common questions

### "My follower count hasn't updated."
**Answer for the user:** We refresh Instagram accounts automatically once a month. To update now, open Profile → ⋯ on your account → "Sync with my Instagram" (once every 24 hours).
**What to check:** `integrations.last_ig_update_date`.
**Escalate when:** last update older than ~45 days for an active Instagram account (monthly run may have stalled).

### "My account was archived without warning."
**Answer for the user:** Our monthly check couldn't find this Instagram profile (it may have been renamed, deactivated or made unavailable). Please send us your current username and we'll restore it.
**What to check:** `integrations.archived_at` date vs. the latest run; whether the username changed.
**Escalate when:** the profile exists and is public (an admin restores it).

### "Brands see old posts on my profile."
**Answer for the user:** We show your latest 10 posts from the last refresh. Use Sync to refresh now.

## Known issues

- On 2026-09-02 a monthly run was skipped because the previous one overran; the scheduler now retries daily. If many users report stale data at the start of a month, the run may be blocked (canary failed or a run still open) — escalate to dev.

## Related topics

- INF-04 archived accounts, INF-05 manual sync, DSC-01 brand search, PUB-01 post tracking.

## Technical reference

- `app/Modules/Crawlzo` (README is authoritative): `Console/Commands/StartMonthlyRefreshCommand` (`EARLIEST_DAY_OF_MONTH = 2`), `PollJobsCommand`, `FinalizeRunCommand`, `CanaryCommand`, `RecoverInterruptedJobsCommand`; `Jobs/DownloadChunkJob`; `Services/CrawlzoRunCoordinator`; config `Config/crawlzo.php` (`CRAWLZO_CANARY_SIZE` 500, `CRAWLZO_CANARY_MIN_USABLE_RATIO` 0.7, `CRAWLZO_MAX_ARCHIVE_RATIO` 0.3, bulk size 5000). Scheduled in `app/Console/Kernel.php` (refresh only when `isProduction()`).
- `Console/Commands/Integration/FillTemporaryIntegrationsFromScraperCommand` (every 30 min); automatic rejection of not-found requests is commented out in `Services/Implementation/Integration/TemporaryIntegrationScraperService`.
- `Console/Commands/Elastic/Integration/ManualIndexIntegration` (monthly).
- Webhooks: `POST api/v2/webhooks/scraper/notify`, `POST api/v2/webhooks/scraper/tiktok/reports/video` → `Http/Controllers/v2/Webhook/ScraperController`.
- Data: `integrations` (followers, followings, medias, username, external_id, last_ig_update_date, archived_at), `crawlzo_runs`, `crawlzo_jobs`, `crawlzo_unresolved_accounts`, Mongo `temporary_instagram_posts`, Elastic `integration` index (`deleted_at` for archived).
