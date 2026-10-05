---
id: SUB-01
title: Plans and buying or upgrading
domain: Subscriptions and plan limits
audience: [brand, system]
apps: [web, background]
related: [SUB-02, SUB-03, SUB-04, PAY-01, PAY-05]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# SUB-01 · Plans and buying or upgrading

> Brands use Ainfluencer on a plan: **Free**, **Starter**, **Pro** (shown as "Professional") or **Ultimate**, billed monthly, quarterly or yearly through Stripe. The plan sets the brand's limits and its platform fee on offers (SUB-04). This page covers comparing plans, the first purchase, upgrading, and the billing history.

## Who and where
- **Brand** (account owner) in the web app:
  - **`/subscription-plans`**: plan comparison ("Pick the plan that fits how you grow."), reached from about 25 upgrade buttons, the plan tracker in the header, the sidebar "Upgrade plan" card and every limit pop-up (SUB-04).
  - **`/subscription-plans/payment-result`**: where Stripe sends the brand back after the first purchase.
  - **`/profile/transactions?tab=billing`** → tab "Subscription & billing": current plan, usage, plan switcher, "Billing history", cancel (SUB-02). Also linked from "Already subscribed?" → "Manage subscription & billing →".
- Custom/enterprise plans: "Meet our sales team" / "Book a demo →" links open a booking page; there is no custom plan in the product.

## How it works
**Comparing plans**
- Billing switch: "Monthly", "Quarterly", "Annual" (Annual is pre-selected). Each card shows the monthly equivalent ("/mo") and a note such as "$399 billed annually · save 30%".
- Card buttons: "Get started →", "Choose Starter →", "Get Ultimate →"; "You're on this plan" for the current plan; "Downgrade" for a lower plan (SUB-02). The Free card's button is always disabled.
- The "current plan" card shows usage this month ("Influencer invites", "Active campaigns", "Username reveals", …) and "Resets in N days".

**First purchase (brand is on Free)**
1. The brand clicks a plan button.
2. The system opens **Stripe Checkout** (Stripe's hosted page). Stripe promotion codes can be entered there.
3. After paying, Stripe sends the brand to `/subscription-plans/payment-result`: "Payment successful – Your subscription is now active.", "Payment failed – Your payment could not be completed. Please try again." or "Payment status pending – We are still waiting for confirmation from Stripe. Please refresh this page in a moment." Buttons "Back to Subscription Plans" / "Try Again".
4. Stripe tells Ainfluencer about the new subscription (webhook) and the new limits apply within moments.

**Upgrading (brand already pays)**
1. The brand picks a higher plan (on either page). A confirmation shows "Current" → "New plan", **"Charged today"** – "Prorated for the rest of this cycle, then {price}", and "Takes effect instantly. Your card on file is charged…". Buttons "Not now" / **"Confirm & upgrade"**.
2. The plan changes **immediately** and Stripe invoices the prorated difference straight away, charging the card on file at Stripe.
3. Success: "Upgrade successful – Your plan has been upgraded and billing was processed automatically." (plans page) or toast "Upgraded to {plan}" (billing tab).
- Switching to another billing cycle of the **same** plan is treated like an upgrade (immediate, prorated).
- An upgrade cancels any downgrade that was waiting for the end of the period (SUB-02).

**Billing history** (tab "Subscription & billing")
- "Current plan" with price, "Next charge" date (or "Cancels on"), and the card ("Card •••• 1234" / "No default card"; "Change payment method" → `/cards`).
- Usage meters: "Active campaigns", "Invites this month", "Team seats".
- "Compare plans" grid ("Changes take effect immediately, and we prorate the difference").
- **"Billing history"**: every Stripe invoice with "Date", "Description" ("{plan} plan · monthly / yearly / every 3 months"), "Amount", "Status" ("Paid", "Open", "Void", "Uncollectible") and "PDF"/"View"; "Download all".

## Rules and limits
**Prices** (from the plan price list; live prices are in the `plan_prices` table):

| Plan | Monthly | Quarterly | Annual |
|---|---|---|---|
| Free | $0 | – | – |
| Starter | $49 | $129 ($43/mo) | $399 ($33.25/mo) |
| Pro ("Professional") | $149 | $399 ($133/mo) | $1,249 ($104.08/mo) |
| Ultimate | $699 | $1,799 ($599.67/mo) | $5,999 ($499.92/mo) |

- One active subscription per brand. Starting a second one is refused: "You currently have an active plan on your account."
- Subscriptions are paid **by card through Stripe**, not from the Ainfluencer wallet.
- There is **no free trial** in our code.
- A brand with no active subscription is on Free automatically.
- If a renewal payment fails, Stripe marks the subscription "past due" and the brand is **treated as Free straight away**; within the hour campaigns over the Free limit are paused and extra team members removed (SUB-02). Paying the invoice restores the plan, but paused campaigns must be resumed by hand.
- The Annual banner on the plans page promises "Cancel: Anytime", "Refund: 7-day full" and "Pause: Up to 3 months". None of these is automated: refunds and pauses are done by the team (SUB-03).
- Limits per plan: SUB-04.

## Statuses
| Status (Stripe, `subscriptions.stripe_status`) | Meaning for the brand |
|---|---|
| `active` | Plan works (also while cancelled-but-not-ended, and while paused) |
| `past_due`, `unpaid`, `incomplete`, `incomplete_expired` | Treated as **Free** |
| `canceled` | Ended; Free |

Other fields: `ends_at` (set = cancellation scheduled), `paused_at` / `pause_resumes_at` (SUB-03), `stripe_price` (which plan price).

## Timers and automatic actions
- Stripe webhook (`/api/subscriptions/webhook`) on create/update/delete/invoice paid: updates the local subscription, stores invoices for the billing history, and re-checks the brand's plan limits.
- `plans:detect-downgrades` (hourly): catches brands whose paid plan lapsed without a webhook (SUB-02).
- Monthly allowances reset on the monthly anniversary of the subscription start (Free: on the 1st of each calendar month) (SUB-04).

## Notifications
Ainfluencer itself sends no subscription emails. Receipts and payment emails come from Stripe (if enabled in Stripe). Invoices are always available in "Billing history".

## Common questions

### "I paid but I'm still on Free."
**Answer for the user:** Please refresh the page after a minute; the plan switches as soon as Stripe confirms the payment. If it still shows Free, send us the date and amount and we'll check it.
**What to check:** `subscriptions` row for the user (`stripe_status`, `stripe_price`, `created_at`), `stripe_invoices`.
**Escalate when:** Stripe shows a paid subscription but there is no active row → dev (webhook problem).

### "How much will I be charged if I upgrade now?"
**Answer for the user:** You pay only the difference for the rest of your current billing period, charged right away, and then the new plan's normal price at renewal. The confirmation window shows this before you confirm.
**What to check:** current plan/price and period end (`current_period_end`).
**Escalate when:** the charged amount doesn't match Stripe's proration.

### "Where can I download my invoices?"
**Answer for the user:** Go to Payments & Billing → "Subscription & billing" → "Billing history" and click "PDF" or "View" on any line, or "Download all".
**What to check:** `stripe_invoices` for the user.
**Escalate when:** an invoice exists in Stripe but not in the list.

### "My plan dropped to Free and my campaigns were paused."
**Answer for the user:** This happens when a renewal payment fails. Please update your card and pay the open invoice; your plan comes back once payment succeeds. Campaigns that were paused need to be switched back on from your Campaigns page.
**What to check:** `stripe_status` (`past_due`/`unpaid`), open invoices, `ads.paused_by = system`.
**Escalate when:** the subscription is `active` but the brand is still treated as Free.

### "Can I pay my subscription from my wallet balance?"
**Answer for the user:** Not at the moment. Subscriptions are charged to your card through Stripe; the wallet is used for offers and other purchases.
**What to check:** n/a.
**Escalate when:** n/a.

### "Do you offer a free trial / a refund?"
**Answer for the user:** There's no free trial; the Free plan lets you try the platform. For a refund on a subscription charge, we'll pass your request to our team.
**What to check:** invoice date and plan cycle.
**Escalate when:** any refund request → finance team (manual in Stripe).

## Known issues
- The plans page shows refund and pause promises ("7-day full", "Up to 3 months") that have no self-serve flow.
- The wallet page says the wallet can pay "your subscription", which isn't true (PAY-01).
- A failed renewal pauses campaigns within the hour and does not un-pause them when the payment later succeeds.
- *(to verify)* "Change payment method" changes the default card on the brand's Stripe customer; a subscription bought through Stripe Checkout may still charge the card used at checkout.

## Related topics
- SUB-02: downgrading, cancelling, resuming.
- SUB-03: pausing.
- SUB-04: what each plan allows (limits, fee %).
- PAY-01: cards and the wallet.

## Technical reference
- **Endpoints** (`routes/api.php`, prefix `/api/subscriptions`, `auth:api2`; `app/Http/Controllers/Subscription/SubscriptionController.php` → `app/Services/Susbscription/SubscriptionService.php`):
  - `GET plans` (PlanController@getDefaultPlans), `GET getCurrentPlanInfo` (plan, period dates, `is_paused`, `scheduled_change`, feature usage), `GET subscribe/{planPriceId}` (Cashier `newSubscription()->allowPromotionCodes()->checkout()` → Stripe Checkout URL; refused if already subscribed), `PUT update/{planPriceId}` (level ≥ current → `swapAndInvoice`; lower → scheduled downgrade, SUB-02), `GET receipts` (from `stripe_invoices`).
  - Webhook `POST /api/subscriptions/webhook` (`CashierWebhookController`: subscription created/updated/deleted, pause sync, downgrade reconcile, `invoice.payment_succeeded`, dispatches `SyncUserPlanJob`). `POST /stripe/webhook` is Cashier's default and skips this logic.
- **Data:** `plans` (`name`, `level`: Free 1, Starter 2, Pro 3, Ultimate 4, `slug`), `plan_prices` (`price`, `duration` months, `stripe_product_price_id`, `is_active`, `display_features`), `subscriptions` (Cashier: `stripe_status`, `stripe_price`, `ends_at`, `current_period_start|end`, `paused_at`, `pause_resumes_at`), `stripe_invoices`, `user_plan_snapshots`.
- **Active subscription:** `User::activeSubscription()` (Cashier `active()`; excludes incomplete/past_due/unpaid); no subscription → first Free plan price.
- **Seed data:** `database/seeders/Subscription/plan_prices_data_production.php`.
- **Web:** `components/pages/subscriptionPlans/*`, `components/pages/transactions2/parts/BillingView.tsx`, `components/common/modals/planChangeConfirm/*`, `lib/services/subscriptions.ts`.
