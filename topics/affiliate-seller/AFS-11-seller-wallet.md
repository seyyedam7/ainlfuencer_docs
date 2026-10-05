---
id: AFS-11
title: Seller wallet
domain: Seller app (brands)
audience: [seller, system]
apps: [seller-app, background]
related: [AFS-05, AFS-10, AFS-12, AFS-13]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-11 · Seller wallet

> The seller wallet is a prepaid balance in US dollars inside the seller app. The seller tops it up by card through Stripe Checkout. Today it is used for one thing only: paying for affiliate email reveals beyond the plan's free quota (AFS-05). It is **not** used for commission charges, which always go to the card (AFS-10), and money can't be withdrawn.

## Who and where
- **Seller** in the seller app:
  - Header wallet badge (shows the balance) → **Wallet** page `/app/wallet`, with the "Ainfluencer WALLET" card ("Balance"), a **Deposit** box and a **Withdraw** box.
  - "Transactions History" at `/app/wallet/transaction-history`, tabs "Usage", "Deposit", "Withdraw".
  - The reveal pop-up "Pay for remaining emails using Wallet" also links here (AFS-05).
- **Team members** need the wallet permissions in their role (AFS-13) to see the badge, the history and the Deposit box.

## How it works
1. The seller types an amount in dollars under "Amount", ticks "I’ve read and agree to the Wallet Terms & Conditions" and clicks **"Deposit"**. ("Payments are securely processed via Stripe.")
2. The system records a **pending deposit** and opens Stripe Checkout for a "Wallet Top-Up" of that amount in USD.
3. The seller pays on Stripe.
   - Paid: Stripe sends the seller back to "Deposit Successful!" (buttons "Go To Dashboard", "Go to Wallet").
   - Cancelled: "Deposit Failed" ("Something went wrong during the payment process…").
4. The balance is increased **only when Stripe confirms the payment** to the platform (Stripe's "payment succeeded" message). That usually takes seconds; the success page itself doesn't add money.
5. When the seller reveals more emails than the plan quota allows, the price per email × number of emails is taken from the balance and shown under "Usage".

## Rules and limits
- Currency: **USD only**.
- **No minimum is checked in the app** (any number ≥ 0 is accepted); Stripe itself refuses very small amounts (below about $0.50), which shows an error.
- **Withdraw is not available** ("Coming soon..."). There is no refund-to-card button; a refund of wallet money is a manual finance action.
- **Uses:** extra email reveals only. Not used for commission, subscriptions or Ainfluencer (core) payments.
- Reveal price per email comes from the platform's settings (the app falls back to $0.10 when none is sent). The whole reveal is refused if the balance is too low ("Insufficient balance").
- The "Wallet Terms & Conditions" link goes nowhere.
- Permissions for team members: view wallet, view transaction history, deposit (AFS-13).

## Statuses
Deposit tab, "Status" column:

| UI | Meaning | Next |
|---|---|---|
| "Confirmed" | Stripe confirmed the payment; balance increased. | — |
| "Rejected" | Not confirmed: the checkout was abandoned or cancelled, **or Stripe's confirmation hasn't arrived yet**. | If the seller was charged, check Stripe (see questions). |

## Timers and automatic actions
- No scheduled jobs. The balance changes only on Stripe's payment confirmation (deposit) or on an email reveal (usage).

## Notifications
- None from the platform for deposits or usage (Stripe may send its own receipt email).

## Common questions

### "I paid but my wallet balance didn't go up."
**Answer for the user:** Your deposit is added once Stripe confirms the payment, usually within a minute. Please refresh the page. If it still isn't there, send us the date and amount and we'll check with Stripe.
**What to check:** `wallet_transactions` for the seller's wallet: type deposit, `amount`, `confirmed` (false = not confirmed); `stripe_payment_intents` row with the same amount (`status` succeeded, metadata source `wallet_checkout_create`).
**Escalate when:** Stripe shows the payment succeeded but `confirmed` is still false after 15 minutes (finance/dev).

### "My deposit shows 'Rejected'."
**Answer for the user:** "Rejected" means the payment wasn't completed (for example the Stripe page was closed). Nothing was charged in that case. If you see a charge on your card, send us the details.
**What to check:** whether a succeeded Stripe payment exists for that transaction.
**Escalate when:** the card was charged.

### "Can I withdraw my wallet money?"
**Answer for the user:** Withdrawals aren't available yet. Your balance stays in your wallet and is used for extra email reveals.
**What to check:** —
**Escalate when:** the seller asks for a refund (finance decides).

### "Can I pay affiliate commissions from my wallet?"
**Answer for the user:** No. Commissions are charged to the card saved in Account settings → Billing. The wallet is only for extra email reveals.
**What to check:** —
**Escalate when:** —

### "The reveal says I don't have enough balance but I do."
**Answer for the user:** The system compares the price with your balance rounded down to whole dollars, so cents can't be used. Top up a little more and try again.
**What to check:** `wallets.balance` vs price × number of emails selected.
**Escalate when:** the whole-dollar balance is clearly enough.

## Known issues
- **Pending deposits show as "Rejected"** in the history, even while Stripe's confirmation is on its way.
- **Cents in the balance are ignored** when paying for reveals (the balance is rounded down to whole dollars before comparing).
- **No amount validation** in the app; tiny or zero amounts fail at Stripe with an error.
- "Wallet Terms & Conditions" link is a placeholder.
- Withdraw is "Coming soon...".

## Related topics
- AFS-05 — revealing affiliate emails (the only use of the wallet).
- AFS-10 — commission billing (card, not wallet).
- AFS-12 — plan quota for free email reveals.
- AFS-13 — team permissions for the wallet.

## Technical reference
- **Endpoints:** POST `seller/v1/wallet/checkout/create` (`amount`) → `CreateWalletCheckoutController` → `Services/General/Wallet/Checkout/CreateWalletCheckoutService` (creates an unconfirmed deposit, Cashier `checkout()` with metadata `user_type_id`, `transaction_id`, `source=wallet_checkout_create`). GET `seller/v1/wallet/checkout/success` and `/cancel` are Stripe return URLs that only log activity and redirect to `/app/wallet/success-payment` or `/failed-payment`. GET `seller/v1/wallet/transactions/list` → `WalletManager::getTransactionsList`.
- **Confirmation:** Stripe `payment_intent.succeeded` on POST `stripe/webhook` → `Listeners/Payment/StripeWebhook` → `StripePaymentDriver::updateOrCreatePaymentIntent` → `handleWalletCheckoutSuccess` → `DepositWalletTransactionService::confirmDeposit` (sets `confirmed=true`, increments `wallets.balance`).
- **Usage:** `Services/General/RevealInformation/PaymentHandlers/WalletPayment` → `UsageWalletTransactionService` (action `reveal_information`, price from `purchasable_features`).
- **Data:** `wallets` (`user_type_id`, `balance`), `wallet_transactions` (`wallet_id`, `type` deposit/usage/withdraw, `amount`, `confirmed`, `action`, `gateway`, `uuid`, `meta`), `stripe_payment_intents`, `purchasable_features` (`price`).
- **Docs in repo:** `docs/Wallet.md`, `docs/RevealInformation.md` (affiliate backend).
- **UI:** `spa-seller/src/components/wallet/*`, `shared-components/algo-wallet/*`.
