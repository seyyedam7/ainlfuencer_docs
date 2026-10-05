---
id: PAY-01
title: Brand wallet, cards and billing history
domain: Money (wallets, refunds, payouts)
audience: [brand, system]
apps: [web, email, background]
related: [OFR-03, PAY-02, PAY-05, SUB-01, PUB-03, DSC-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# PAY-01 · Brand wallet, cards and billing history

> Every brand has an Ainfluencer wallet in US dollars. The brand can top it up (card, Stripe, PayPal or crypto), save cards for later, and see where its money is: in the wallet, held in escrow for an offer, paid to influencers, or refunded. This page covers topping up, saved cards and the "Payments & Billing" page.

## Who and where
- **Brand** in the web app:
  - **Add funds**: `/funds`, page title "Add Funds to Your Wallet". It is reached from the top banner ("add funds"), from the wallet page ("Add funds"), from the cards page ("Add Funds"), from the "campaign created" pop-up and from the username-reveal pop-up (these last ones open `/funds?amount=…` with the amount already filled).
  - **Saved cards**: `/cards` ("Your Card List") and `/cards/create` ("Add Credit Card").
  - **Payments & Billing**: `/profile/transactions` (profile menu). Two tabs: "Wallet & transactions" (this topic) and "Subscription & billing" (SUB-01, SUB-02).
- Influencers have their own wallet in the mobile app (PAY-03).

## How it works
**The three wallets.** Behind the scenes each user has three wallets:
- **Default wallet**: the brand's own money (top-ups and refunds).
- **Credit wallet**: bonus credit added by Ainfluencer staff (PAY-05). It can't be topped up or withdrawn.
- **Lock (escrow) wallet**: money held for a deal. When a brand pays an offer, the money moves into the *influencer's* lock wallet and waits there until the content is approved or a timer releases it (PUB-03).

The "Wallet balance" the brand sees is **default + credit together**. When the brand pays, **credit is used first**, then cash.

**Adding funds (`/funds`)**
1. Pick an amount: preset buttons **$500**, **$1,000** ("Recommended"), **$2,000** ("Best Value"), **$5,000**, or "Enter custom amount" (whole dollars). $1,000 is pre-selected. If the page was opened with an amount (`?amount=`), only that amount is shown.
2. Pick a payment method:
   - **"Stripe"** ("Popular" – Apple Pay, Google Pay, cards & more): opens Stripe's hosted checkout.
   - **"Credit Card"** (Visa, Mastercard, American Express): choose a saved card or "+ New Card". The card is charged **immediately**.
   - **"PayPal"**: opens PayPal.
   - **"Crypto"** (Bitcoin, Ethereum, Solana, USDT, USDC): opens the crypto payment provider (NOWPayments).
3. Tick **"I agree that wallet deposits are non-refundable if not used on campaigns."** and click **"Add $X to Wallet"**.
4. Result:
   - **Saved card**: the money is in the wallet straight away. Toast "Funds were added to your wallet successfully." and the brand lands on `/profile/transactions` (or back on the offer they came from).
   - **Stripe / PayPal / Crypto**: after paying, the provider sends the brand to an Ainfluencer result page: "Payment successful" or "Payment failed", with "Tracking code", "Method", "Amount", "Date" and a "Return to Dashboard" button. The wallet is credited when this page confirms the payment with the provider (for Stripe, Stripe's own confirmation message also credits it).
5. No fee is taken on top-ups: the full amount goes into the wallet. The brand gets a "Deposit Confirmation" email.

**Saved cards (`/cards`)**
- "Add New Card" opens `/cards/create`: card number, expiry, CVC and billing address (Stripe form), with "Set as default card". The **first card is always made default**.
- Each card has a menu: **"Make Default"** and **"Remove This Card"** → "Delete Card" / "Yes, Delete it".
- Cards are stored at Stripe; Ainfluencer keeps only the masked number. Adding the same card twice doesn't create a duplicate.
- The default card is also the one Stripe uses for subscription charges (the billing tab's "Change payment method" links here). *(to verify: a subscription bought through Stripe Checkout may keep its own card)*.

**Payments & Billing → "Wallet & transactions"**
- **"Wallet balance"** with "Add funds" and **"Top-up history"** (pop-up "Wallet top-ups": card deposits and other top-ups, "Total shown", "Current balance"). Also "Default card" with "Add" / "Manage".
- **"Where your money is"**: "Total spent" (with "See breakdown": "Influencer deals", "Content translation", "Insurance"), "Held in escrow", "Paid to influencers", "Refunded to you", "Credits Received".
- If influencers have delivered content waiting for approval, a banner says "{amount} is waiting on your approval" with "Review escrow".
- **"Transactions"** table (influencer, campaign, "Paid by you", date, type, method, status). Filter chips: "All", "Transferred To Influencer Wallet", "Refunded To Brand Wallet", "Paid Out to Influencer", "On Hold in Escrow". Clicking a row shows the cost breakdown ("Amount", "Platform fee", "Content insurance", "Coupon discount", "Charged to", "Total paid by you") and, for money in escrow, "Review and release {amount}" and "View offer".
- **"Older transactions"** opens the history from the old payment system (before August 2025).

## Rules and limits
- Minimum top-up: **$1** ("Minimum is $1"). No maximum in the app.
- Currency: **USD only**.
- Wallet deposits are **non-refundable to the card** if not used. There is no withdraw button for brands; any cash-out is a manual finance decision.
- Each payment method can be switched on or off by the team. If a method is off, the brand sees "Payment gateway unavailable. Please try again later or select another option." The page still shows all four buttons.
- Cards that need extra bank verification (3-D Secure) can fail with "Additional verification required. Please complete authentication to proceed." *(to verify how often this happens)*.
- Paying offers from the wallet, with coupons and insurance, is covered in OFR-03.

## Statuses
Status of a deal in the "Transactions" table:

| What the brand sees | Meaning | What happens next |
|---|---|---|
| "On Hold in Escrow" (`transferred`) | Paid; money is held for the influencer | Released when content is approved or the timer passes (PUB-03), or refunded (PAY-02) |
| "Transferred To Influencer Wallet" (`completed`) | Released to the influencer's wallet | Influencer can request a payout (PAY-04) |
| "Paid Out to Influencer" (`settled`) | Ainfluencer has paid the influencer | Final |
| "Refunded To Brand Wallet" (`refunded`) | Money returned to the brand's wallet | Final (PAY-02) |

Top-up records (`payment_receipts`): `DEPOSIT_REQUESTED`/`DEPOSIT_PENDING` → `DEPOSIT_PAID`, or `DEPOSIT_FAILED` / `DEPOSIT_CANCELED` (or `user_cancelled` when the brand cancelled on the provider's page).

## Timers and automatic actions
- **Hourly**: `finance:create-missed-wallets` creates any missing wallet (default, lock, credit) and registers the user with Stripe. A brand whose wallet "doesn't exist" is usually fixed within the hour.
- Offer checkouts that are not paid within 30 minutes expire (OFR-03).

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Wallet top-up confirmed ("Deposit Confirmation - Thank you for choosing Ainfluencer!") | Brand | Email, in-app |
| Credit added by staff (PAY-05) | Brand | Email, in-app |

## Common questions

### "I paid with PayPal / crypto but my wallet balance didn't go up."
**Answer for the user:** Payments through PayPal and crypto are added to your wallet once the payment is confirmed when you come back to Ainfluencer after paying. If the page closed early, or the crypto payment was still confirming, it can be missed. Please send us the tracking code or the date and amount, and we'll check it for you.
**What to check:** the brand's `payment_receipts` rows (gateway, amount, status, `tracking_code`, `ref_id`). `DEPOSIT_REQUESTED` with no `DEPOSIT_PAID` = not confirmed. Compare with the provider's records.
**Escalate when:** the provider shows the payment as completed but the receipt is not `DEPOSIT_PAID` → finance/dev to credit it by hand.

### "Can I get my wallet money back on my card?"
**Answer for the user:** Wallet deposits can be used for offers, translations and other platform purchases, but they're non-refundable to your card if unused (you agreed to this when adding funds). If you have a special case, we'll pass it to our finance team.
**What to check:** balance and whether the money came from a top-up, a refund or credit (credit can never be cashed out).
**Escalate when:** the brand insists on a cash refund → finance team decision.

### "My card payment failed."
**Answer for the user:** Your bank declined the charge or asked for extra verification that our card form can't complete. Please try another card, or use "Stripe" (Apple Pay / Google Pay / card) or PayPal on the Add Funds page.
**What to check:** the error shown ("Additional verification required…", "Missing payment details…", "Payment could not be processed…"); whether the card is the default and not expired.
**Escalate when:** every card fails for this brand, or the card was charged but the wallet wasn't credited.

### "Why is my balance different from what I added?"
**Answer for the user:** Your balance goes down when you pay an offer (that money moves to escrow), and goes up again if an offer is refunded. The "Where your money is" section shows how much is held in escrow, paid to influencers and refunded.
**What to check:** "Held in escrow", recent payments, refunds, and credits used (credit is spent before cash).
**Escalate when:** the numbers don't add up after checking the transaction list.

### "I can't delete my card / I want a different default card."
**Answer for the user:** On the Cards page, open the card's menu and choose "Make Default" for the card you want, or "Remove This Card" to delete one.
**What to check:** the card exists in the list; the brand is logged in as the account owner.
**Escalate when:** the action shows an error.

### "Where are my old transactions?"
**Answer for the user:** Payments made before our new payment system (August 2025) are under "Older transactions" on the Payments & Billing page.
**What to check:** legacy `transactions` table for the brand.
**Escalate when:** an old payment is missing from both lists.

## Known issues
- **PayPal and crypto top-ups depend on the return page.** There is no server-to-server confirmation from PayPal or the crypto provider; if the brand never reaches the result page (or crypto is still unconfirmed at that moment), the wallet is not credited. Workaround: none for the user → escalate to dev/finance with the tracking code.
- The Add Funds page always shows all four methods, even if one is switched off; the error only appears after clicking.
- The wallet page text says the balance can pay "your subscription", but subscriptions are charged to the card at Stripe, not to the wallet (SUB-01).

## Related topics
- OFR-03: paying for an offer (checkout, insurance, platform fee).
- PAY-02: refunds back to the brand wallet.
- PAY-05: coupons and staff credits.
- SUB-01: subscription charges and billing history (other tab of the same page).
- PUB-03: when escrow is released or refunded.

## Technical reference
- **Endpoints** (all `/api/{locale}/finance/...`, `app/Modules/Finance/Routes/api.php`):
  - `GET current-user/wallets` (WalletController@index: summary = default + credit; reports from `transaction_reports`), `GET current-user/wallets/{wallet}` (wallet transactions), `POST current-user/wallets` (WalletController@store: top-up; `amount` min 1, `gateway` in card|stripe|paypal|nowpayments, `walletSettingId` for card).
  - `GET/POST current-user/cards`, `DELETE current-user/cards/{id}`, `POST current-user/cards/{walletSettingId}/make-default` (CardController).
  - `GET current-user/orders` (OrderController@index) for the transactions table; `GET /api/v2/user/wallet` for older transactions.
  - Returns: `GET payment/callback/success|failed/{tracking}` (PaymentCallbackController, renders `finance::payment-callback`); webhook `POST stripe/webhook` (StripeWebhookController: `payment_intent.succeeded` → PaymentReceiptPaidEvent).
- **Note:** the web `/funds` page sends every method (including PayPal) to `POST current-user/wallets`; `v2/paypal/charge` is not called.
- **Data:** `wallets` (`holder_id`, `holder_type`, `name` default|lock|credit, `balance` in cents), `wallet_transactions`, `wallet_settings` (`key` = card/customer_id/payout, `provider`, `is_default`, `extra.token`), `payment_receipts` (`status`, `gateway_id`, `amount` cents, `tracking_code`, `ref_id`, `is_income`), `payment_receipt_logs`, `orders`/`order_items` (`status`), `transaction_reports` (per-user totals), `gateways` (`active`, `fee_per_transaction`).
- **Listeners:** PaymentReceiptPaidListener (marks DEPOSIT_PAID, sends ChargeWalletNotification for top-ups), TransferFromGatewayToOwnerListener (moves the money from the gateway wallet to the user's wallet).
- **Jobs:** `finance:create-missed-wallets` (hourly, `Console/CreateUserWalletsIfNotExists.php`).
- **Key files:** `app/Modules/Finance/docs.md` (wallet model), `Gateway/StripeGateway.php` (`recharge`), `Gateway/NowPaymentsGateway.php`, `Gateway/PaypalGateway.php`, `Gateway/GatewayFactory.php` (inactive gateway → error); web `components/pages/newFunds`, `newCards`, `creditCardNew`, `transactions2`.
