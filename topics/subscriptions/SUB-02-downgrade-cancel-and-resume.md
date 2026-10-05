---
id: SUB-02
title: Downgrade, cancel and resume
domain: Subscriptions and plan limits
audience: [brand, system]
apps: [web, background]
related: [SUB-01, SUB-03, SUB-04, CMP-03, BRD-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# SUB-02 · Downgrade, cancel and resume

> A brand can move to a cheaper plan or cancel. Neither happens straight away: the brand keeps the current plan until the end of the period already paid for. When the paid plan finally ends, campaigns and team members over the new limits are switched off automatically.

## Who and where
- **Brand** (account owner) in the web app:
  - **Downgrade**: the "Downgrade" button on a lower plan's card at `/subscription-plans`, or "Move to Starter" inside the cancel window. (In the "Compare plans" grid on the billing tab, downgrade buttons are disabled.)
  - **Cancel / Resume**: `/profile/transactions?tab=billing` → box "Cancel your subscription" → **"Cancel subscription"**; after cancelling the box becomes "Cancellation scheduled" with **"Resume"**.
- **System**: Stripe webhooks and the hourly job apply the new limits when the paid plan ends.

## How it works
**Downgrading to a lower paid plan**
1. On `/subscription-plans` the brand clicks "Downgrade" on a lower plan (the Free card can't be chosen; to go to Free, cancel).
2. Confirmation: "Takes effect – You keep {current plan} until {date}, then {new plan} renews at {price}". Buttons "Keep {plan}" / **"Confirm downgrade"**; note "You can upgrade again anytime".
3. The change is **queued at Stripe for the end of the current period**. Nothing changes now: same plan, same limits, no charge or refund.
4. At the end of the period Stripe moves the subscription to the new plan and bills the new price. Ainfluencer then applies the new limits (see "When the paid plan ends").
- Upgrading, cancelling or pausing before that date **drops the queued downgrade**.

**Cancelling** (a short window with up to 3 steps)
1. **Step 1 – "Before you cancel"**: "Your {plan} plan stays active until {date}." It lists what the brand has (public score, deals closed, live campaigns, content library items, money held in escrow).
   - If money is held in escrow, the window says "{amount} is still held in escrow, so this subscription cannot be cancelled until those funds are approved or refunded." and "Continue cancelling" is disabled.
   - If the brand is above Starter, it offers **"Move to Starter"** ("Keep a lower plan instead of cancelling."), which is a downgrade as above.
   - If already cancelled, it offers "Resume" instead.
2. **Step 2 – survey** "What pushed you to cancel?": "It costs more than it returns right now", "We are not using it enough", "We did not get the results we hoped for", "A feature we need is missing", "We are pausing marketing for a while", "Something else". Answering is saved even if the brand then backs out. If the survey can't load, this step is skipped.
3. **Step 3 – "Confirm cancellation"**: "Your {plan} plan will end on {date}. You keep full access until then, and you will not be charged again." → **"Cancel my subscription"**.
4. Done: "Subscription cancelled – Active until {date}".
- The end date is the end of the current paid period, as reported by Stripe.
- Cancelling a paused subscription lifts the pause first (SUB-03).

**Resuming a cancellation**
- While the end date hasn't passed, "Resume" (billing tab or cancel window) undoes the cancellation; toast "Your subscription is active again." Billing continues as before.
- After the end date, resuming is impossible; the brand buys a plan again (SUB-01).

**When the paid plan ends** (cancellation ran out, a downgrade took effect, or a renewal payment failed)
- **Campaigns**: if the brand runs more campaigns than the new plan allows, the **oldest running campaigns are paused** (marked paused by the system); the newest ones stay live. Campaigns waiting for admin review are not paused, but if one is approved later while the brand is over the limit, that campaign is paused at approval (CMP-02).
- **Team members**: open team invitations are deleted first, then the **oldest team members** are removed until the brand fits its seats (BRD-03).
- Nothing is deleted: campaigns, content and data stay. The brand can switch a paused campaign back on only when it has a free campaign slot (otherwise the upgrade pop-up appears, SUB-04).
- No email or notification is sent about these changes.

## Rules and limits
- Only the account owner manages the subscription.
- No refund or credit is given for the rest of the period when downgrading or cancelling.
- Cancelling is blocked in the window while the escrow check shows money held (see Known issues).
- A cancelled subscription can't be paused.
- Downgrades to Free are done by cancelling.

## Statuses
| What the brand sees | Internal | Meaning |
|---|---|---|
| "Next charge {date}" | `subscriptions.ends_at` empty | Active, renews |
| "Cancellation scheduled" / "Cancels on {date}" | `ends_at` set, in the future | Cancelled, still active until then ("grace period") |
| Plan shows Free | no active row (`ends_at` passed, or `stripe_status` past_due/unpaid/canceled) | Ended |
| (not shown in the web app) | `getCurrentPlanInfo.plan.scheduled_change` | Downgrade queued at Stripe |
| Campaign "Paused" | `ads.paused_by = system` | Paused because of the plan limit |

## Timers and automatic actions
- **Stripe webhook** when the subscription is updated or deleted → re-check of the brand's limits right away.
- **`plans:detect-downgrades` – hourly** (UTC): finds brands whose last known plan was paid but who no longer have an active subscription (cancellation ran out, payment failed) and applies the new limits. Up to 2,000 brands per run. So the pause can happen up to about an hour after the end date.

## Notifications
None from Ainfluencer for downgrade, cancel, resume, or the automatic pausing of campaigns / removal of team members. Stripe may send its own emails.

## Common questions

### "I cancelled. Do I lose access now?"
**Answer for the user:** No. You keep your plan and everything in it until the end of the period you've paid for (shown as "Cancels on" on the billing page). You won't be charged again.
**What to check:** `subscriptions.ends_at`.
**Escalate when:** the brand lost features before `ends_at`.

### "I changed my mind. Can I undo the cancellation?"
**Answer for the user:** Yes, as long as the end date hasn't passed: go to Payments & Billing → "Subscription & billing" and click "Resume".
**What to check:** `ends_at` in the future, `stripe_status = active`.
**Escalate when:** "Resume" fails while `ends_at` is still in the future.

### "I downgraded but I'm still on the old plan."
**Answer for the user:** That's expected. A downgrade takes effect at the end of your current billing period, so you keep the higher plan you've already paid for until then.
**What to check:** `getCurrentPlanInfo` → `scheduled_change.effective_at`; `current_period_end`.
**Escalate when:** the effective date has passed and the plan didn't change.

### "My campaigns were paused and a team member lost access."
**Answer for the user:** Your paid plan ended (or moved to a smaller plan), so we paused the campaigns and removed the team members that no longer fit in it. Nothing was deleted. You can upgrade to restore the limits, or pause another campaign to switch one back on.
**What to check:** `ads.paused_by = system`, plan snapshot (`user_plan_snapshots`), subscription status, team members.
**Escalate when:** the brand still has an active paid plan with enough slots.

### "It says I can't cancel because money is in escrow."
**Answer for the user:** You have paid offers whose content hasn't been approved or refunded yet. Please approve the delivered content (or wait for the deal to finish), then try again. If you believe nothing is pending, contact us.
**What to check:** paid offers with escrow (PAY-01 "Held in escrow"), cancellation summary `escrow`.
**Escalate when:** nothing is held but the window still blocks → finance/dev can cancel in Stripe.

### "Can I get a refund for the rest of my plan?"
**Answer for the user:** Cancelling or downgrading doesn't refund the current period; you keep using the plan until it ends. If you have a special case, we'll pass it to our team.
**What to check:** invoice dates and cycle.
**Escalate when:** the brand asks for a refund → finance team (manual in Stripe).

## Known issues
- The plans-page downgrade success message says "Your plan has been changed and billing was processed automatically." This is wrong: nothing changes until the end of the period.
- The web app doesn't show that a downgrade is queued, so brands think it failed.
- The cancel window text says live campaigns "stop accepting offers immediately", the content library becomes read-only, and scores reset after 60 days. None of this is done by the system; campaigns keep running until the paid period ends.
- *(to verify)* The escrow check that blocks cancelling reads the **old** payment records only, so money held for offers paid since August 2025 may not block it (and old leftovers might block it wrongly).
- Campaigns paused by a failed renewal stay paused after the payment succeeds.

## Related topics
- SUB-01: plans, prices and failed renewals.
- SUB-03: pausing instead of cancelling.
- SUB-04: limits that decide what gets paused.
- CMP-03: pausing and resuming campaigns.
- BRD-03: team members and seats.

## Technical reference
- **Endpoints** (`/api/subscriptions`, `SubscriptionController` → `SubscriptionService`): `PUT update/{planPriceId}` (lower level → `PeriodEndPlanDowngrader::scheduleAtPeriodEnd`, a Stripe subscription schedule; returns `ScheduledPlanChange`), `GET cancellation-summary` (`CancellationSummaryService`: score, deals, live campaigns, content items, `escrow` from `User::dealsOnHold()` = legacy `transactions.status = locked`, `blockers.has_funds_in_escrow`, `subscription.access_until|already_cancelled|is_paused`), `POST cancel` (releases pause, cancels scheduled change, Cashier `cancel()`; returns `{cancelled, ends_at, on_grace_period}`), `POST resume` (only `onGracePeriod()`), survey `GET /api/v2/forms/positions` (slug `plan_cancellation_survey`), `GET/POST /api/v2/forms/{formId}`.
- **Enforcement:** `app/Modules/PlanEnforcement` (README): `SyncUserPlanJob` → `PlanChangeDetector` (compares with `user_plan_snapshots`; first sighting is never a change) → `PlanQuotaEnforcer` → `CampaignQuotaPolicy` (keep newest, pause oldest live; `paused_by = system`) and `SeatQuotaPolicy` (invitations first, then oldest members). `PlanQuotaEnforced` event has no listener (no notification). `ApprovedCampaignAdmission` pauses an over-quota campaign at approval.
- **Jobs:** `plans:detect-downgrades` (hourly, `Console/Commands/DetectPlanDowngradesCommand.php`; treats `incomplete`, `incomplete_expired`, `past_due`, `unpaid` as lapsed). Manual only: `plans:enforce-quotas`, `plans:repair-system-pauses`.
- **Data:** `subscriptions` (`ends_at`, `stripe_status`, `stripe_price`, `current_period_end`), `user_plan_snapshots` (`plan_level`), `ads.status|paused_by`, team membership tables, form submissions.
- **Docs:** `core/backend/api/documentations/cancellation-flow-api.md` (backend contract; some screen texts differ).
- **Web:** `components/pages/transactions2/parts/{BillingView,CancelSubscriptionModal}.tsx`, `components/pages/subscriptionPlans/index.tsx`.
