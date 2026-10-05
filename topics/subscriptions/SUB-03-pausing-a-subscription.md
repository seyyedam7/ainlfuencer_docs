---
id: SUB-03
title: Pausing a subscription
domain: Subscriptions and plan limits
audience: [brand, admin, system]
apps: [background]
related: [SUB-01, SUB-02, SUB-04, CMP-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# SUB-03 · Pausing a subscription

> A pause stops subscription billing for a number of months instead of cancelling. While paused, the brand keeps its plan but **all its live campaigns are taken off the marketplace**. **Brands can't pause or unpause from the web app**: the backend supports it, but no screen calls it. A pause exists only if the Ainfluencer team set it (or it was set in Stripe).

## Who and where
- **Brand**: no pause button anywhere in the web app. The brand notices a pause only because its campaigns are paused and can't be switched back on.
- **Team (admin / finance / dev)**: sets or ends a pause on the brand's request, through Stripe or the backend.
- **System**: Stripe and an hourly job end the pause automatically on the resume date.
- The plans page's Annual banner advertises "Pause: Up to 3 months", and the cancel survey has "We are pausing marketing for a while", but there is no self-serve pause behind either.

## How it works
1. The team pauses the subscription for **1 to 3 months** (backend maximum). Stripe stops collecting payments until the resume date (pause date + N months).
2. Moments later, **every live campaign** of the brand is paused and removed from the influencer marketplace, so no new offers arrive. These campaigns are marked as paused by the subscription pause.
3. During the pause:
   - The plan itself stays: seats, inbox and chat, analytics, video translation, content library and score keep working. Nothing is deleted.
   - Existing paid deals continue normally (posting, approval, release of escrow).
   - Campaigns can be edited, renamed or paused, but **switching one back on is refused** with: "Your subscription is paused, so campaigns cannot be switched back on. Resume your subscription first."
4. The pause ends either:
   - **automatically on the resume date**: Stripe restarts billing; Ainfluencer is told by webhook, and an hourly safety job ends any pause whose date has passed (so at worst about an hour late); or
   - **early**, when the team unpauses it; billing restarts immediately.
5. When the pause ends, **exactly the campaigns that the pause froze are switched back on**. Campaigns the brand had paused itself stay paused.

## Rules and limits
- Pause length: whole months, **1–3**. (The backend guide `cancellation-flow-api.md` says 1–12; the code allows 3.)
- A **cancelled** subscription (cancellation scheduled) can't be paused: "Your subscription is already cancelled, so it cannot be paused."
- An already paused subscription can't be paused again: "Your subscription is already paused."
- Pausing **drops any queued downgrade** (SUB-02); the plan will no longer change by itself at renewal.
- **Cancelling while paused** lifts the pause first, then cancels at the end of the paid period (SUB-02).
- The refusal to reactivate campaigns is a 400 error (not the upgrade pop-up): buying a bigger plan doesn't help; only ending the pause does.

## Statuses
| What support may see | Internal | Meaning |
|---|---|---|
| Subscription paused | `subscriptions.paused_at` set and `pause_resumes_at` in the future (or empty = open-ended) | Billing on hold, campaigns frozen |
| Pause over | `paused_at` empty, or `pause_resumes_at` in the past | Billing running; frozen campaigns restored |
| Campaign frozen by the pause | `ads.paused_by = subscription_pause` | Will come back on when the pause ends |
| Campaign paused by the brand | `ads.paused_by = brand` | Stays paused after the pause |
| Plan info | `getCurrentPlanInfo` → `plan.is_paused`, `paused_at`, `pause_resumes_at` | Not displayed by the web app |

Stripe still reports the subscription as `active` while paused, so plan limits and features stay.

## Timers and automatic actions
- **Stripe** resumes collection on the resume date and sends a webhook; Ainfluencer clears the pause and un-freezes the campaigns.
- **`subscriptions:resume-due-pauses` – hourly** (UTC): ends any pause whose resume date has passed (Stripe first, then locally) and un-freezes the campaigns, in case the webhook was missed.
- Freezing and un-freezing run on a queue: campaigns change a few seconds after the pause/unpause.

## Notifications
None. Ainfluencer sends no email or notification when a pause starts or ends.

## Common questions

### "How do I pause my subscription?"
**Answer for the user:** Pausing isn't available as a button in the app yet. Tell us how many months you'd like to pause (1 to 3) and our team will set it up. While paused you won't be billed, your campaigns are taken off the marketplace, and everything else keeps working.
**What to check:** active subscription, not cancelled (`ends_at` empty), not already paused.
**Escalate when:** always → finance/dev team to apply the pause.

### "Why can't I turn my campaign back on? It says my subscription is paused."
**Answer for the user:** Your subscription is currently paused, so campaigns can't go live until it resumes. It resumes automatically on {date}; if you want to resume earlier, we can do that for you.
**What to check:** `subscriptions.paused_at`, `pause_resumes_at`.
**Escalate when:** the brand wants to resume early (team unpauses), or the resume date has passed by more than an hour and campaigns are still frozen.

### "My pause ended but my campaigns are still paused."
**Answer for the user:** Campaigns paused by the subscription pause come back automatically within about an hour of the resume date. Campaigns you paused yourself stay paused; you can switch those on from your Campaigns page.
**What to check:** `ads.paused_by` (`subscription_pause` vs `brand`/`system`), `pause_resumes_at`.
**Escalate when:** campaigns with `paused_by = subscription_pause` remain after the pause ended (+1 hour).

### "Will I lose anything while paused?"
**Answer for the user:** No. Your plan, team, chats, content and score stay as they are; nothing is deleted. Only your campaigns are hidden from influencers until the pause ends.
**What to check:** n/a.
**Escalate when:** n/a.

### "I was paused and then cancelled. When does my plan end?"
**Answer for the user:** Cancelling lifts the pause, and your plan then runs until the end of the period you've paid for, shown as "Cancels on" on the billing page.
**What to check:** `ends_at`, `paused_at` cleared.
**Escalate when:** the end date looks wrong.

## Known issues
- No self-serve pause or unpause in the web app, although the plans page advertises "Pause: Up to 3 months".
- The billing page doesn't show that the subscription is paused or when it resumes; the brand only sees the activation error on campaigns.

## Related topics
- SUB-02: cancelling (and how a pause interacts with it).
- SUB-01: plan status and billing.
- CMP-03: pausing and resuming campaigns.

## Technical reference
- **Endpoints** (backend only; no client calls them): `POST /api/subscriptions/pause` (`months` 1–`SubscriptionService::MAX_PAUSE_MONTHS` = 3, `PauseSubscriptionRequest`), `POST /api/subscriptions/unpause`. Service: `SubscriptionService::pause()` / `unpause()` / `releasePause()` / `isPaused()`; Stripe call via `SubscriptionPauseGateway` (`pause_collection`).
- **Campaign freeze:** `app/Services/Susbscription/Pause/CampaignFreezer.php` (`freeze` sets `ads.paused_by = subscription_pause`; `thaw` restores only those), run by `SyncSubscriptionPauseJob`.
- **Activation guard:** middleware `campaign.activation.quota` (`app/Modules/PlanEnforcement/Http/Middleware/EnforceCampaignActivationQuota.php`) on `PUT /api/v2/campaigns/{ad}` and `PATCH /api/v2/campaigns-affiliate/{id}`; message key `subscription.paused_cannot_activate_campaign` (400).
- **Webhook:** `CashierWebhookController::syncPauseCollectionFromStripePayload` mirrors a pause set or removed in Stripe.
- **Job:** `subscriptions:resume-due-pauses` (hourly, `app/Console/Commands/Subscription/ResumeDuePausesCommand.php`, supports `--dry-run`).
- **Data:** `subscriptions.paused_at`, `subscriptions.pause_resumes_at`, `ads.paused_by`.
- **Docs:** `core/backend/api/documentations/cancellation-flow-api.md` §3 (describes a "pause instead" option the current web app doesn't have).
