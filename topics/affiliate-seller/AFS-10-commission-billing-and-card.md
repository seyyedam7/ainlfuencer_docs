---
id: AFS-10
title: Commission billing and card
domain: Seller app (brands)
audience: [seller, affiliate, system]
apps: [seller-app, background]
related: [AFS-03, AFS-06, AFS-09, AFS-11, AFS-12, AFA-08, AFX-02, AFC-02]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-10 · Commission billing and card

> A seller on the affiliate platform pays affiliates a commission on the sales their links bring in. The platform charges the seller's saved card automatically, at night, once a sale is at least 35 days old, and sends the affiliate their share (commission minus the Ainfluencer fee) through Stripe.

## Who and where
- **Seller** (seller.ainfluencer.com):
  - Adds or changes the card in **Account settings → Billing** (`/app/settings?tab=billing`), box "Connect Credit Card" / "Connected Credit Card", button "Let's Go" or "Manage Credit Card". This opens the Stripe customer portal ("Redirecting to Stripe").
  - Sees charges in **More → Payments** (`/app/payment-history/commission`, "Commission Statements"). From Billing there are also the buttons "Commission Statements" and "Subscription Statements" (subscription charges are AFS-12).
  - Opens one charge's invoice at `/app/payment-history/commission-invoice/:id` ("Invoice Details", "Download Invoice").
  - The dashboard shows a "Connect Your Credit Card" card while no card is saved.
- **Ainfluencer brands** can also add the card from the Ainfluencer web app (affiliate campaign setup, see AFX-02); it is the same Stripe customer.
- **Affiliate**: receives the money in their Stripe Connect account (setup and payout view are AFA-08).
- **System**: the nightly billing job.

## How it works
1. **Card.** The seller clicks "Let's Go" / "Manage Credit Card" and adds a card (or US bank account / Link, whatever the Stripe portal offers) on Stripe. Stripe tells the platform, and the Billing box switches to "Connected Credit Card". "Connected" just means at least one payment method is saved.
2. **Sales come in.** Amazon sales arrive with the Amazon report sync (every 72 h, covering the last 60 days, AFS-09). Each sale row gets its commission worked out from the commission on the affiliate's contract for that product (AFS-03, AFS-06).
3. **Waiting period.** A sale can only be billed when its date is **35 days or more in the past** (this covers Amazon returns). Until then it shows to the affiliate as "upcoming" (AFA-08).
4. **Nightly run, 03:00 UTC.** The system picks **at most 10 sellers** per night (those whose last billing is oldest, and at most once per 24 h per seller). For each seller it adds up all unbilled, 35+ day-old sales **per affiliate and per currency**.
5. **Charge.** For each affiliate total it charges the seller's card **the full commission**. Stripe immediately routes the money to the affiliate's Stripe account, minus the **Ainfluencer fee**, which Ainfluencer keeps. There is no wallet or holding step: it is one Stripe charge. The affiliate's bank payout then follows Stripe's own payout schedule.
6. **If the card fails**, the system tries the seller's other saved payment methods, then tries again 60 seconds later, up to **3 attempts** in total (about 2 minutes). After the 2nd failure the seller gets "⚠️ Last Payment Attempt"; after the 3rd, "❌ Payment Transaction Failed". The row shows status "Failed" and a **"Retry"** button.
7. **Seller retries.** On the statement row the seller clicks "Retry". The charge is tried again with the current card(s). Success message: "The invoice payment has been retried successfully and processed".
8. **Success.** The row shows "Succeeded" with a "Get Invoice" button. Seller gets "✅ Commission Payment Sent", affiliate gets "💰 Commission Payment Received".

## Rules and limits
- **Delay:** sales are billed only when 35+ days old (`PAYMENT_TRANSACTION_PERIOD`, default 35).
- **Volume:** at most **10 sellers per night**, each seller at most once per 24 h (`SELLER_COMMISSION_CALCULATION_SYNC_COUNT` 10, `..._SYNC_PERIOD` 24 h). A busy night can push a seller to a later night.
- **Who is billed:** sellers who are **not suspended** and have at least one Amazon profile or Shopify store connected.
- **What is billed:** only sale rows that are linked to a partnership contract and not billed before. A sale on a product that was deactivated is never given a commission (sales dated after the deactivation are dropped at sync).
- **Commission formula (Amazon):**
  - Percentage commission: commission = the product's attributed sales (14-day Amazon attribution) × rate %.
  - Fixed-amount commission: the code does **sale amount − fixed amount**, not the fixed amount. See Known issues.
  - The rate comes from the contract **at the time the report is synced**. Amazon rows are re-synced for 60 days, so changing a product's commission also re-prices sales that are not yet billed (AFS-03).
- **Ainfluencer fee** = commission × fee %. The fee % is the seller plan's "Application fee" value, fixed on the sale row the first time it is synced. On the Free plan (or no paid plan) the platform default is used (`SELLER_DEFAULT_APPLICATION_FEE`, 5% unless changed). The seeded production plan values are Starter 20%, Pro 15%, Ultimate 3%; check the plan in the admin panel for the real value. Shopify rows always use the default.
- **Skipped silently (sale stays unbilled and is tried again on a later night):**
  - the seller has no Stripe customer, or
  - the affiliate has not finished connecting Stripe (status not "connected", AFA-08).
- **Minimum:** totals that round to $0.00 are skipped. There is no other minimum.
- **Retry:** only allowed when the row is "Failed". Any other status gets the error "You can not retry a payment transaction when the status is not failed". Needs the team permission to view commission statements.
- **No card:** creating an Ainfluencer affiliate campaign is refused (HTTP 499, the seller app jumps to Settings → Billing), and active campaigns are paused every night at 02:30 UTC and not un-paused automatically (AFX-02).
- **Shopify sales are never charged** (commission is never calculated). See Known issues.

## Statuses
Shown in the "Status" column of Commission Statements (labels come from the backend).

| Status (UI) | Meaning | Next |
|---|---|---|
| "Pending" (`pending`) | Charge created, Stripe not finished yet. | Normally becomes Succeeded or Failed within minutes. |
| "Succeeded" (`succeeded`) | Card charged; affiliate received commission minus fee. | "Get Invoice". |
| "Failed" (`failed`) | Card declined on all saved methods after 3 attempts. | Seller fixes card, clicks "Retry". |
| "Server error" (`server_error`) | A non-card error (Stripe or system). | Seller can't retry it. Escalate to dev. |
| "Refunded" / "Partially refunded" (`refunded`, `partially_refunded`) | An admin refunded the charge. | None. |

The affiliate sees the same charge as a payout: pending → "processing", succeeded → "paid", failed/server error → "failed" (AFA-08).

## Timers and automatic actions
| When (UTC) | What | User notices |
|---|---|---|
| 02:30 daily | Campaigns of sellers with no saved payment method are paused (AFX-02). | Campaign paused on the affiliate platform (Ainfluencer is not told); never auto-resumed. |
| 03:00 daily | Billing run: up to 10 sellers, sales 35+ days old. | New rows in Commission Statements. |
| +60 s, +120 s | Automatic re-attempts after a card failure (3 attempts total). | Notifications after attempt 2 and 3. |
| 06:00 daily | Amazon report sync (profiles older than 72 h, last 60 days). | New sales; re-priced unbilled sales. |

## Notifications
All payment notifications are **in-app only** (bell). No email is sent for any of them.

| Trigger | Who | Channel |
|---|---|---|
| Charge succeeded: "✅ Commission Payment Sent" | Seller | In-app |
| Charge succeeded: "💰 Commission Payment Received" | Affiliate | In-app |
| 2nd failed attempt: "⚠️ Last Payment Attempt" | Seller | In-app |
| 3rd failed attempt (and every failed manual retry after that): "❌ Payment Transaction Failed" | Seller | In-app |
| Affiliate's Stripe account status changes | Affiliate | Email (AFA-08) |

## Common questions

### "Why was I charged? I don't remember buying anything."
**Answer for the user:** This is the commission for sales your affiliates generated on Amazon at least 35 days ago. Each line in Commission Statements is one affiliate; open "Get Invoice" to see the products and units behind it.
**What to check:** the row in `payment_transactions` (by `tracking_code`) → its `commission_transactions` row → `commission_transaction_items` → the linked `amazon_reports` rows (dates, `commission`).
**Escalate when:** the sales or commission don't match the seller's Amazon data, or the commission type is a fixed amount (known bug below).

### "My affiliate says they weren't paid."
**Answer for the user:** Commission is paid only when the sale is 35+ days old, the seller has a card on file, and the affiliate has fully connected Stripe. After that, Stripe pays out on its own schedule.
**What to check, in order:** (1) sale date at least 35 days ago; (2) seller has a `stripe_id` and a row in `stripe_customer_payment_methods`; (3) affiliate's `stripe_accounts.status` = `connected`; (4) seller `user_types.status` is not suspended (5); (5) `amazon_reports.partnership_contract_id` is set; (6) seller's `user_types.report_synced_at` (was the seller processed recently?); (7) is it a Shopify sale (never billed).
**Escalate when:** all checks pass and the sale is more than a few days past day 35 without a commission row.

### "My payment failed. How do I fix it?"
**Answer for the user:** Update your card in Account settings → Billing → "Manage Credit Card", then go to More → Payments, find the failed line and click "Retry".
**What to check:** `commission_transactions.status`, `attempts`, and the latest `commission_transaction_logs.description` (Stripe's decline reason).
**Escalate when:** Retry shows "You can not retry…" on a row that is not Succeeded, or the status is "Server error".

### "What is the Ainfluencer fee and why is it this amount?"
**Answer for the user:** It's Ainfluencer's share, taken out of the commission you pay; your affiliate receives the commission minus this fee. The percentage depends on your plan.
**What to check:** `amazon_reports.application_fee` on the billed rows (the % locked at first sync) and the seller's plan (AFS-12).
**Escalate when:** the fee % doesn't match the plan the seller had when the sales were first synced.

### "The statement total is higher than what my bank shows."
**Answer for the user:** Your card is charged the "Commission Paid" amount. The "Total Fee" column adds the Ainfluencer fee on top, but that fee is already included in the commission you paid, so the bank amount is the correct one.
**What to check:** Stripe charge amount = `commission_transactions.amount`.
**Escalate when:** the bank amount is different from `amount`.

### "I removed my card. Will I still be charged?"
**Answer for the user:** Without a card, commission can't be collected, and your affiliate campaigns are paused every night until a card is added. Unpaid commission stays owed and will be charged once a card is added.
**What to check:** `stripe_customer_payment_methods` for the seller's Stripe customer.
**Escalate when:** the seller disputes owed commission (finance).

### "My Shopify affiliate sales show $0 commission."
**Answer for the user:** Commission for Shopify sales isn't calculated automatically at the moment, so nothing has been charged for them. We've passed this to the team.
**What to check:** `shopify_reports.total_commission` is 0 for the rows.
**Escalate when:** always (known gap, finance/dev decide how to settle).

## Known issues
- **Shopify sales are never charged.** No production code fills `shopify_reports.total_commission`, so Shopify sales produce $0 commission and no charge. Escalate to dev/finance.
- **Fixed-amount commission is calculated as "sale minus amount".** A $100 sale with a $5 fixed commission is charged $95 as commission; a sale below the fixed amount gives a negative commission. Escalate to dev; refunds go to finance.
- **"Total Fee" / "Total Paid" overstates the charge.** The statement and invoice add the fee to the commission, but the card was charged only the commission. Use "Commission Paid" as the real amount.
- **"Total Sale" column shows units sold, formatted as money** (for example 12 units shows as "$12"). The invoice lines are correct.
- **Fee tooltip is wrong.** It says the fee is "%3.5 of your sales… under the Ultimate Plan". In the code the fee is a % of the commission, and the % comes from the plan (seeded Ultimate value is 3%).
- **A seller with sales but no saved card can stop the whole nightly run** *(to verify in logs)*. The run crashes on such a seller before charging the others in that batch, and the same batch is picked again the next night. Symptom: many sellers not billed for weeks. Escalate to dev.
- **Sellers can be stuck at the back of the queue** *(to verify)*. A seller's "last billed" time is updated only when a charge is created, and sellers never billed sort last. Sellers with nothing to bill can keep taking the 10 nightly slots. Escalate to dev if billing is late for many sellers.
- **Bank-account (ACH) payments may be recorded as "Server error"** *(to verify)*. Stripe returns "processing" for bank debits; the code doesn't handle that status, and the job is retried, which could create a second charge attempt. Escalate to dev and check Stripe before any refund.
- **"Server error" rows can't be retried** by the seller (the UI shows "Retry" but the backend refuses). Escalate to dev.
- **Disputes and Stripe refunds have no effect** on the platform records (the affiliate transfer is not reversed). Admin refunds may be recorded as "Partially refunded" even when full *(to verify)*. Escalate to finance.
- The daily "Stripe connect nudge" reminder (seller without card) most likely never sends.

## Related topics
- AFS-03 — product commission rates and how changes re-price unbilled sales.
- AFS-06 — partnership contracts, which carry the commission used for billing.
- AFS-09 — reports and the Amazon sync that brings in sales.
- AFS-11 — the seller wallet (not used for commission).
- AFS-12 — plans, which set the Ainfluencer fee %.
- AFA-08 — affiliate side: Stripe Connect, upcoming commission, payout status.
- AFX-02 — card requirement and nightly pause for Ainfluencer affiliate campaigns.
- AFC-02 — notifications.

## Technical reference
- **Endpoints (seller app):**
  - GET `seller/v1/payments/create-billing-portal` → `CreateBillingPortalController` → `Services/Actions/Seller/Payment/CreateBillingPortal` (creates the Stripe customer if missing, returns the portal URL).
  - GET `seller/v1/payments/commission-transactions` (list), `/{paymentTransactionId}`, `/{id}/items`, `/{id}/export` (CSV `commission_transactions.csv`), POST `/{id}/retry` → `RetryCommissionTransactionController` → `Services/General/Payment/CommissionTransaction::retry`.
  - Core: `seller/v1/integrations/ainfluencer/cards/list` and `cards/sync` (AFX-02).
- **Webhooks:** POST `stripe/webhook` (Cashier, platform account) and POST `client/v1/callback/stripe` (Connect) → `app/Listeners/Payment/StripeWebhook.php`. `payment_method.attached/detached` → `stripe_customer_payment_methods`; `customer.updated` → default method; `account.updated` → `stripe_accounts`; `payment_intent.*` → `stripe_payment_intents` only (does **not** update commission rows); `charge.dispute.*` → `stripe_charges` only. Test-mode events are ignored in production.
- **Scheduled job:** `payment:create-transaction` (`app/Console/Commands/Payment/CreatePaymentTransaction.php`), daily 03:00 UTC → `Services/General/Calculation/Calculation::calculateAllSellerCost` → `Services/General/Payment/Driver/StripePaymentDriver::createCommissionTransaction` → job `app/Jobs/PayCommissionTransaction.php` → `StripePaymentDriver::payCommissionTransaction` (PaymentIntent: `amount` = commission, `application_fee_amount` = fee, `transfer_data.destination` and `on_behalf_of` = affiliate Connect account, `confirm` = true).
- **Formula:** `Services/General/AmazonReport/SaveAmazonReport::calculateCommission` (percent: `attributedSales14d × rate/100`; amount: `attributedSales14d − value`). Fee % from `Services/General/AmazonReport/AmazonReport::getApplicationFee` (plan feature `application_fee`, else `config('ainfluencer.seller.default_application_fee')`); stored in `amazon_reports.application_fee` and not overwritten on re-sync.
- **Config:** `config/payment.php` (`max_attempts` 3, `attempts_delay` 60 s, `transaction_period` 35), `config/ainfluencer.php` (`seller.commission.calculation_sync_count` 10, `calculation_sync_period` 24, `seller.default_application_fee` 5). Horizon `tries` = 3 on the default queue.
- **Data:**
  - `user_types.stripe_id`, `user_types.status` (5 = suspended), `user_types.report_synced_at` (last billing run that created a charge).
  - `stripe_customer_payment_methods` (`stripe_customer_id`, `payment_method_id`, `type`, `is_default`).
  - `stripe_accounts` (`user_type_id`, `account_id`, `status` connected/not_connected, `charges_enabled`, `payouts_enabled`).
  - `amazon_reports` (`seller_id`, `affiliate_id`, `date`, `partnership_contract_id`, `commission`, `application_fee`, `attributed_sales_14d`, `units_sold_14d`); `shopify_reports` (`total_revenue`, `units_sold`, `total_commission` always 0).
  - `commission_transactions` (`payer_user_type_id`, `receiver_user_type_id`, `amount`, `application_fee_amount`, `currency`, `status`, `attempts`); `commission_transaction_items` (`model_type`/`model_id` → report rows); `commission_transaction_logs` (one per PaymentIntent try: `status`, `description`, `payment_id`).
  - `payment_transactions` (`tracking_code`, `transactionable_*`, `status`, `payment_intent_id`) — the row the seller UI lists and retries.
  - `partnership_contracts.total_commission` / `total_buy_amount` are overwritten with the last batch's sums only (not lifetime totals).
- **UI:** `shared-components/algo-setting/tabs/Billing.vue`, `spa-seller/src/components/payment-history/commission-transactions.*`, `.../commission-invoice/*`; 499 handling in `shared-components/store/authorization.store.js`.
