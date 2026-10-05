---
id: AFA-08
title: Earnings and payouts (affiliate)
domain: Affiliate app (creators)
audience: [affiliate, seller, system]
apps: [affiliate-app, mobile, background]
related: [AFS-10, AFA-04, AFA-05, AFA-09, AFA-10, MKT-03, AFC-02, AFX-02]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-08 · Earnings and payouts (affiliate)

> How an affiliate gets paid commission on the affiliate platform: they connect a Stripe account, sales are counted from Amazon reports, and once a sale is at least 35 days old the seller's card is charged and the money goes straight to the affiliate's Stripe account, minus the platform fee.

## Who and where
- **Affiliate** (creator) in the affiliate app, affiliate.ainfluencer.com:
  - **Connect Stripe**: user menu → "Account Settings" → "Billing" tab (`/app/settings?tab=billing`). The box is titled "Connect Credit Card" and the button is "Let's Go". It really sets up a **Stripe Connect payout account**, not a card.
  - **See earnings**: sidebar "More" → "Payments" (`/app/payment-history/commission`). This page has two tables: "Upcoming Payments" and "Commission Payments History". The "Subscription" tab on the same page lists plan invoices (AFA-09).
  - **One statement**: click a row in the history → `/app/payment-history/commission-invoice/:id` ("Invoice Details", "Download Invoice").
  - **Dashboard card** "Upcoming Payments" on `/app` (AFA-10).
- **Influencers in the Ainfluencer mobile app** see the same payouts in the affiliate analytics screens (MKT-03). The mobile app gets them from the affiliate platform.
- **Seller** side of the same money (card charge, retries) is AFS-10.

## How it works
1. **The affiliate connects Stripe.** In "Billing" they click "Let's Go". The app shows "Redirecting to Stripe" and opens Stripe's onboarding. Ainfluencer creates a **Stripe Express** account for them. Its country is the country of the phone number they verified at sign-up. If there is none, it uses the country in their profile, and if that is empty too, it uses **US**.
2. **Stripe finishes the check.** The account counts as connected only when Stripe reports that it can **both** take charges and send payouts. Until then the box shows "Check Your Credit Card". After that it shows "Connected Credit Card" with a "Manage Credit Card" button. That button opens Stripe onboarding again so the affiliate can update their details.
3. **Sales are counted.** For Amazon, each night the platform pulls the seller's Amazon Attribution report for the last 60 days, with a fresh pull about every 72 hours per store. Each sale on a tracked link is matched to the affiliate's partnership contract. The commission is worked out from the contract's commission rate (AFA-04).
4. **The 35-day wait.** A sale can be billed only when its date is **35 or more days ago**. This window covers Amazon returns and late reports.
5. **The nightly billing run (03:00 UTC).** Every night the system takes **up to 10 sellers**, each at most once every 24 hours. For each seller it adds up all unbilled sales older than 35 days, per affiliate and per currency. It then creates one **commission statement** per affiliate and currency.
6. **One charge pays the affiliate.** The seller's saved card (or US bank account / Link) is charged the **full commission**. In the same Stripe payment, the commission **minus the platform fee** is sent to the affiliate's Stripe account, and Ainfluencer keeps the fee. Ainfluencer holds nothing in between: there is no wallet or balance for affiliates.
7. **Stripe pays out to the bank.** The money then reaches the affiliate's bank on **Stripe's own payout schedule** for their Stripe account. Ainfluencer doesn't control that timing.
8. **If the card fails**, the system tries the seller's other saved payment methods. If all fail, it retries **3 times, 60 seconds apart**. After that the statement stays "Failed" until the seller retries it (AFS-10). The affiliate gets nothing for that statement until then.

## Rules and limits
- **Stripe Connect is required to be paid.** If the affiliate's Stripe account isn't connected, their sales are **skipped silently** in the billing run. Nothing is lost: the sales stay unbilled and are picked up on a later run once Stripe is connected.
- **No route in the affiliate app is blocked** for a missing Stripe account in the current version. The "HTTP 499 → go to Billing" redirect exists in the app, but only seller actions trigger it. Affiliates can partner and make links without Stripe. They just won't be paid.
- **The seller must have a Stripe customer and a card.** No card means no charge, so no payout. A **suspended seller** is skipped entirely.
- **Platform fee** = commission × fee %. The fee % comes from the **seller's** plan ("application fee"). If the plan sets none, or for Shopify sales, it is **5%**. The fee % is fixed per sale on the first day that sale is recorded.
- **Commission formula (Amazon):** percent commission = attributed sales × rate. The rate is the contract's rate at the time the report is synced, so a seller's rate change re-prices sales that haven't been billed yet (AFS-03).
- **Shopify sales are not paid.** No commission is ever calculated for Shopify sales in this version, so Shopify statements are $0 and are skipped (see Known issues).
- **No minimum payout.** Any amount above $0.00 (after rounding to cents) is billed.
- **Currency** is the Amazon marketplace's currency (or the Shopify store's currency). One statement per affiliate per currency.
- **Sales after a product is deactivated** earn nothing (AFS-03). Sales on a link whose contract item can't be found are skipped.
- **Team members** of an affiliate need the Stripe permission to use the "Billing" button.

## Statuses
Commission statements ("Commission Payments History", column "Status"):

| Status shown | Internal value | Meaning | What happens next |
|---|---|---|---|
| Pending | `pending` | Statement created, charge in progress or waiting for a retry | Usually resolves within minutes |
| Succeeded | `succeeded` | Seller charged; affiliate's share sent to their Stripe account | Stripe pays it out to the bank on its schedule |
| Failed | `failed` | All charge attempts on the seller's payment methods failed | Waits for the seller to retry (AFS-10) |
| Server error | `server_error` | Non-Stripe error during the charge; **not retried automatically** | Escalate to dev |
| Refunded / Partially refunded | `refunded` / `partially_refunded` | An admin refunded the seller | See Known issues |
| Canceled | `canceled` | Exists in the list, but nothing sets it today | — |

The mobile app shows these as payout statuses: processing (pending), paid (succeeded), failed (failed or server error), plus canceled/refunded.

## Timers and automatic actions
| When (UTC) | What | Affiliate notices |
|---|---|---|
| Daily 03:00 | Billing run: up to 10 sellers, sales ≥ 35 days old | New row in "Commission Payments History"; web notification on success |
| 60 s after a failed charge | Retry, up to 3 attempts total | Status stays "Pending" then "Failed" |
| Daily 06:00 | Amazon report pull for stores whose report is older than 72 h (last 60 days) | Sales and "Remaining" numbers change |
| On Stripe update | Stripe tells us the account status | Billing box changes to connected / not connected |
| Daily 10:00 | "Connect your Stripe" reminder email job | **Likely never sent** (see Known issues) |

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Commission charge succeeded | Affiliate ("💰 Commission Payment Received") and seller | In-app (web) only, **no email** |
| Charge failing / failed for good | Seller only | In-app (web) only |
| Stripe account status changed | Affiliate | Email, queued for the 06:00/18:00 UTC batch *(to verify that it is actually delivered)* |

## Common questions

### "When will I get paid for my sales?"
**Answer for the user:** Commission is paid once a sale is at least 35 days old. After that, the seller's card is charged in our nightly billing run, and your share goes straight to your connected Stripe account. Stripe then sends it to your bank on your Stripe payout schedule. In practice, expect roughly 5–7 weeks from the sale, sometimes more if the seller's card fails.
**What to check:** Sale date ≥ 35 days ago; the affiliate's Stripe status is connected; the seller has a card and isn't suspended; whether a statement exists (`commission_transactions` for this affiliate) and its status.
**Escalate when:** Sales are older than ~40 days, Stripe is connected, the seller has a card, and still no statement exists (possible billing-queue stall, see Known issues).

### "The 'How It Works' box says I'm paid on the 15th, 1.5 months after the sale month. Is that right?"
**Answer for the user:** That text is out of date. Sales are billed on a rolling basis: each sale becomes payable 35 days after its date, and billing runs every night. There's no fixed monthly payday.
**What to check:** Nothing. This is a wording issue in the app.
**Escalate when:** Never for the user. It's a known text issue.

### "I connected Stripe but I still haven't been paid."
**Answer for the user:** Connecting Stripe makes you eligible. Payment then follows the normal schedule: sales must be 35+ days old and the seller's card must be charged successfully. Any older sales that were waiting are paid in the next billing for that seller.
**What to check:** In order: (1) `stripe_accounts.status` = `connected` (Stripe must report both charges and payouts enabled; if the box says "Check Your Credit Card", Stripe still needs details). (2) Unbilled sales older than 35 days exist ("Upcoming Payments" table). (3) The seller has a Stripe customer and card. (4) The seller isn't suspended. (5) Any `failed` statements for that seller.
**Escalate when:** All of the above are fine and nothing was billed for 3+ days.

### "My payment shows Failed."
**Answer for the user:** The seller's card was declined. The seller needs to update their card and retry the payment. We've notified them in their dashboard. You'll be paid as soon as the charge succeeds.
**What to check:** `commission_transactions.status` and `attempts`; `commission_transaction_logs` for the decline message.
**Escalate when:** The status is "Server error" (not retried automatically), or a failed statement stays failed for more than 7 days. In that case, finance should contact the seller.

### "The amount I received in Stripe is less than the commission shown."
**Answer for the user:** The commission shown is the full amount charged to the seller. Ainfluencer's platform fee (usually 5%, depending on the seller's plan) is taken from it, so you receive the commission minus that fee.
**What to check:** The statement's amount and platform fee (`commission_transactions.amount`, `application_fee_amount`). Net = amount − fee.
**Escalate when:** Net received ≠ amount − fee.

### "The invoice says 'Total Commission' X, but I got much less."
**Answer for the user:** The "Total Commission" on the invoice page is shown incorrectly: it adds the platform fee instead of subtracting it. What you actually receive is the commission minus the fee.
**What to check:** Amount and fee on the statement; tell the user the net figure.
**Escalate when:** Never needed for the user; it's a known display bug.

### "I sell through a Shopify brand and see sales but no commission."
**Answer for the user:** Shopify commission isn't being calculated at the moment, so Shopify sales aren't paid yet. We've reported this to the team.
**What to check:** The product's platform is Shopify.
**Escalate when:** Always log it with the dev team, with the affiliate and seller names, so the sales can be paid later.

### "Why can't I connect Stripe / Stripe asks for the wrong country?"
**Answer for the user:** Your Stripe account is created in the country of the phone number you verified on Ainfluencer. Once a Stripe account exists, its country can't be changed.
**What to check:** The affiliate's verified phone country and profile country; whether a `stripe_accounts` row already exists.
**Escalate when:** The country is wrong and the affiliate needs a new Stripe account. Dev has to remove the old connection.

### "I want to message the seller about a payment."
**Answer for the user:** Please contact the brand through the Ainfluencer chat. The affiliate platform's own messages aren't a supported channel.
**What to check:** Nothing; point them to CHAT-01.
**Escalate when:** —

## Known issues
- **Shopify sales earn $0.** No code calculates commission for Shopify sales, so they are never charged or paid. Escalate to dev.
- **Fixed-amount commission is wrong** (Amazon): instead of paying a fixed amount, it calculates "sale − amount". Escalate to dev if a seller uses fixed commission.
- **Invoice page "Total Commission" is overstated**: it shows commission + fee instead of commission − fee. Use the statement amount minus the fee.
- **"Commission Payments History" → "Total Income"** shows the gross commission, not what the affiliate received.
- **Payment notification shows the gross amount and always uses "$"**, even for other currencies.
- **"Upcoming Payments" → "Due Date"** always shows the next Sunday. It isn't a real date; billing runs nightly.
- **"Upcoming Payments" only shows Amazon sales that are already 35+ days old** and waiting to be billed, before the platform fee. Newer sales don't appear there yet. They show in reports (AFA-10).
- **"How It Works" text is outdated**: it says "paid out 1.5 months following the sale month", with April sales paid on 15 June. The code bills each sale 35 days after its date. It also mentions deducting refunds, and no such logic exists.
- **Billing queue can stall** *(to verify)*: a seller who has nothing billable (for example, all their affiliates lack Stripe) keeps its old "last billed" date. That seller stays first in the queue of 10 per night, so other sellers may wait longer. Escalate to dev if billing seems stuck for several days.
- **Stripe "connect" reminder emails are likely never sent** (missing configuration).
- **Admin refunds don't take money back from the affiliate's Stripe account**, and a full refund is likely recorded as "Partially refunded". Card disputes have no effect on the affiliate. Finance should handle these by hand.
- **Affiliate payouts on Associates (AFA-07)** are not part of this flow.

## Related topics
- AFS-10 · Seller side: card, nightly charge, retries.
- AFA-04 · Partnerships: the contract and commission rate a sale is paid at.
- AFA-05 · Links: only sales on tracked links count.
- AFA-10 · Reports: sales and commission before billing.
- MKT-03 · The same payouts in the Ainfluencer mobile app (net of fee).
- AFC-02 · Notification settings and email batching.
- CHAT-01 · Ainfluencer chat, the channel for talking to the seller.

## Technical reference
- **Endpoints (affiliate app):**
  - `GET affiliate/v1/payments/onboarding-request` → `Affiliate\V1\Payment\OnboardingRequestController` → `Services/Actions/Affiliate/Payment/OnboardingRequestService` (Stripe Express account + account link).
  - `GET affiliate/v1/payments/upcoming-commission-transactions` → `IndexUpcomingCommissionTransactionService` (Amazon rows ≥ 35 days old, unbilled, grouped by seller).
  - `GET affiliate/v1/payments/commission-transactions[/{id}[/items|/export]]` → `IndexCommissionTransactionService`, `ShowCommissionTransactionService` (resource `CommissionTransactionResource`: `total` = amount + fee, the bug).
  - `GET affiliate/v1/payments/invoices`, `.../charges` (subscription, AFA-09).
  - `GET affiliate/v1/dashboard/report/upcoming-payments` → `ShowUpcomingPaymentService`.
  - Core/mobile: `GET affiliate/v1/integrations/ainfluencer/payouts` → `IndexPayoutIntegrationService` / `PayoutIntegrationResource` (net = amount − application_fee_amount), plus `reports/summary|timeline|products`.
- **Billing:** command `payment:create-transaction` (`Console/Commands/Payment/CreatePaymentTransaction`, daily 03:00 UTC, production only) → `Services/General/Calculation/Calculation::calculateAllSellerCost` → `StripePaymentDriver::createCommissionTransaction` (skips if seller has no `user_types.stripe_id` or the receiver has no `stripe_accounts` row with `status=connected`) → job `Jobs/PayCommissionTransaction` → `StripePaymentDriver::payCommissionTransaction` (PaymentIntent with `confirm=true`, `transfer_data.destination` = affiliate account, `on_behalf_of` = affiliate, `application_fee_amount` = fee).
- **Config:** `payment.transaction_period` = 35 days, `payment.max_attempts` = 3, `payment.attempts_delay` = 60 s, `ainfluencer.seller.commission.calculation_sync_count` = 10, `...calculation_sync_period` = 24 h, `ainfluencer.seller.default_application_fee` = 5.
- **Stripe webhooks:** `POST client/v1/callback/stripe` (Connect events) and `POST stripe/webhook` → `Listeners/Payment/StripeWebhook`. `account.updated` sets `stripe_accounts.status` = `connected` only when `charges_enabled && payouts_enabled`.
- **Data:** `stripe_accounts` (`user_type_id`, `account_id`, `status`, `charges_enabled`, `payouts_enabled`); `amazon_reports` (`date`, `affiliate_id`, `seller_id`, `partnership_contract_id`, `commission`, `application_fee`); `shopify_reports` (`total_commission` always 0); `commission_transactions` (`payer_user_type_id`, `receiver_user_type_id`, `amount`, `application_fee_amount`, `currency`, `status`, `attempts`); `commission_transaction_items` (links to report rows; a row with an item is billed); `commission_transaction_logs` (one per charge attempt, decline text); `payment_transactions` (`tracking_code`, `status`); `user_types.report_synced_at` (seller's last billing).
- **Commission calculation:** `Services/General/AmazonReport/SaveAmazonReport::calculateCommission`.
- **Notifications:** `AffiliateCommissionPaymentSuccessNotification` (web only), `StripeAccountStatusUpdatedNotification` (mail only, queued), nudge command `payment:send-stripe-connect-nudges` (daily 10:00).
- **Frontend:** `aff-frontend/shared-components/algo-setting/tabs/Billing.vue`, `spa-affiliate/src/components/payment-history/commission/*` (incl. `modals/how-it-works.modal.vue`), `.../commission-invoice/commission-invoice.page.vue`.
