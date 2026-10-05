---
id: PAY-03
title: Influencer wallet and earnings
domain: Money (wallets, refunds, payouts)
audience: [influencer, system]
apps: [mobile, background]
related: [PUB-03, PAY-04, PAY-02, OFR-03, OFR-05, MKT-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# PAY-03 · Influencer wallet and earnings

> The influencer's wallet in the mobile app shows money earned from paid collaborations: what is waiting in escrow, what has been released and can be withdrawn, and what was paid out. Affiliate earnings are **not** here; they come from the affiliate platform (MKT-03).

## Who and where
- **Influencer** in the mobile app, screen **WALLET_SCREEN** with three tabs: **"Wallet"**, **"Payments"**, **"Payouts"**.
- Reached from: the earnings card on Home and on Profile, the "See payout" button on a "Published & Approved" offer card in chat, the "cost transferred" push notification, and the `wallet` deep link.
- **"Transactions Before 9th August 2025"** (link at the bottom of the Wallet tab) opens the old wallet (WALLET_SCREEN_OLD), history only.
- "Questions? Chat with support" / "Contact support" opens the support chat.

## How it works
**Where the money is at each step**
1. The brand pays for the offer (OFR-03). The **full agreed offer price** goes into the influencer's **escrow (lock) wallet**. The brand's platform fee is paid by the brand on top and never touches the influencer's money. If the brand used a coupon, Ainfluencer pays the coupon part, so the influencer still gets the full price.
2. The influencer posts and submits the link (PUB-01).
3. The money is **released** from escrow to the influencer's wallet when the brand or an admin approves the post, or when the release timer runs out (PUB-03). The influencer gets a "cost transferred" push.
4. The influencer requests a payout (PAY-04). **Ainfluencer's fee (shown in the app as "Ainfluencer Fee (X%)", normally 20%) is taken at payout time**, not before. So the wallet shows the full price, and the payout is the price minus the fee.

**Wallet tab**
- A card **"Wallet Balance"**: money released to the influencer and not yet paid out (it still includes money in a withdrawal request that is being processed).
- Under it, two smaller figures: **available to withdraw** (balance minus requests in progress) and **pending withdrawal requests**. *(to verify on a device: in v32.2.2 these two figures appear without their labels.)*
- Payout method section and the **"Submit Withdrawal Request"** button (PAY-04).
- "How do Ainfluencer payouts work?" pop-up.

**Payments tab**
- One row per paid deal: brand, "Campaign:", the deal status (for example "Published & Approved") and the payment status:
  - "Pending Payment" – not paid by the brand yet.
  - "Payment Escrowed" – paid, held by Ainfluencer.
  - "Payment Released" – released to the wallet; can be withdrawn.
  - "Payout Made" – Ainfluencer has paid it out.
  - "Refunded Payment" – returned to the brand (PAY-02).
- The amount shown is the **full offer price** (before the payout fee).

**Payouts tab** – withdrawal requests and their status (PAY-04).

**Old wallet ("Transactions Before 9th August 2025")** – Payments and Payouts from the old payment system, with a detail screen (TRANSACTION_DETAILS: "Open Offer", "Post Review"). It can't be used to withdraw. Any balance left from the old system appears in the new withdrawal sheet as **"Legacy balance"** ("Funds from campaigns completed before the new payment system update").

## Rules and limits
- Money in escrow can't be withdrawn. Only released deals ("Payment Released") can be.
- The fee is a percentage set by Ainfluencer per payout method (the PayPal / crypto payout gateway); the app shows it at withdrawal. The legacy balance is withdrawn with **no fee**.
- If the brand revised the offer after payment and the influencer accepted a higher price (OFR-05), the extra amount is added to that deal; the payout uses the deal's current price.
- Currency: USD.
- Credit added by staff to an influencer is kept in a separate credit wallet and can't be withdrawn (withdrawals are checked against the released-money wallet only).

## Statuses
| Payments tab label | Internal (`order_items.status`) | Next |
|---|---|---|
| "Pending Payment" | `pending` | Brand pays |
| "Payment Escrowed" | `transferred` | Released (PUB-03) or refunded (PAY-02) |
| "Payment Released" | `completed` | Influencer requests payout (PAY-04) |
| "Payout Made" | `settled` | Final |
| "Refunded Payment" | `refunded` | Final |

## Timers and automatic actions
- Release from escrow is automatic every minute when the rules in PUB-03 are met (approved, or 14 days after start with a link, or 2 days after start without one).
- The pop-up "How do Ainfluencer payouts work?" says payments are held "for up to 7 days after campaign delivery" and that withdrawals "are processed on the 1st and 15th of each month". The 7 days is not what the code does (see PUB-03 for real timers); the 1st/15th is a team schedule for manual payouts (PAY-04).

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Money released to the wallet ("cost transferred") | Influencer | Push (opens the wallet) and in-app; details in PUB-03 |
| Deal refunded to the brand | Influencer | Email, in-app |
| Payout marked paid by Ainfluencer | Influencer | In-app only (PAY-04) |

## Common questions

### "The brand approved my post. Why can't I withdraw?"
**Answer for the user:** After approval the payment moves from escrow into your wallet within a minute or two, and the deal shows "Payment Released". Then choose a payout method and tap "Submit Withdrawal Request". If the button is grey, check that a payout method is selected and that the deal isn't already in a withdrawal request.
**What to check:** `order_items.status` for the deal (`completed` = withdrawable), any `payment_receipts` with `WITHDRAW_IN_PROGRESS` for it, the selected payout method.
**Escalate when:** the post is approved but the order item stays `transferred` for more than an hour.

### "Why does my payout show less than the offer price?"
**Answer for the user:** Your wallet shows the full offer price. Ainfluencer's fee (shown as "Ainfluencer Fee" with its percentage in the withdrawal screen, usually 20%) is deducted when the payout is made. The withdrawal screen shows "Offer Amount", the fee and "You Will Receive" before you confirm.
**What to check:** `payment_receipts.fee` on the request, gateway `fee_per_transaction`.
**Escalate when:** the fee charged differs from the percentage shown when the request was made.

### "My wallet balance shows money but 'available' is $0."
**Answer for the user:** That money is already part of a withdrawal request we're processing. Once it's paid, the balance goes down; if it's rejected, it becomes available again.
**What to check:** `GET finance/current-user/wallets` summary `pending_payout`; `payment_receipts` in `WITHDRAW_IN_PROGRESS`.
**Escalate when:** no request is in progress but available is still 0.

### "A deal disappeared / says Refunded Payment."
**Answer for the user:** The payment for that deal went back to the brand because the content wasn't delivered or wasn't approved in time. You received an email about it.
**What to check:** publish job status and feedback; order item `refunded` (PAY-02).
**Escalate when:** the influencer has proof the post was live and submitted on time.

### "Where is my money from before August 2025?"
**Answer for the user:** Tap "Transactions Before 9th August 2025" at the bottom of the Wallet tab to see your old history. Any old balance that can still be withdrawn appears as "Legacy balance" when you tap "Submit Withdrawal Request".
**What to check:** `legacy_payouts` row for the user (`balance`, `status` pending/in_progress/settled).
**Escalate when:** the old wallet shows money but there is no legacy balance row.

### "The chat card shows a different amount from my wallet."
**Answer for the user:** The offer card estimates your payout as 80% of the price. Your wallet shows the full price, and the exact fee is shown when you request a withdrawal.
**What to check:** offer price, gateway fee.
**Escalate when:** n/a (display difference only).

## Known issues
- The "Published & Approved" offer card always shows **price × 0.8**, even if the fee for the influencer's payout method is different.
- The two figures under "Wallet Balance" have no labels in v32.2.2 *(to verify)*.
- The payouts explainer pop-up mentions a 7-day escrow and PayPal only; real timers are in PUB-03, and crypto is also available.

## Related topics
- PUB-03: when escrow is released or refunded.
- PAY-04: payout methods and withdrawals.
- PAY-02: refunds back to the brand.
- MKT-03 / AFA-08: affiliate earnings (separate, via Stripe Connect).

## Technical reference
- **Endpoints:** `GET /api/{locale}/finance/current-user/wallets` (WalletController@index: `summary.balance`, `pending_payout`, `available_to_withdraw`, `reports.total_earn`); `GET /api/{locale}/finance/mobile/influencer-orders` (OrderController@influencerOrders → Payments tab); `GET /api/{locale}/finance/mobile/orders/payable` (PAY-04); legacy `GET /api/{locale}/v3/mobile/wallet`, `…/wallet/transactions/payments`, `…/wallet/transactions/payouts`.
- **Money movement:** `TransferOrderTransactionsListener` (brand → influencer **lock**, coupon share from system user → influencer lock); `OrderItemCompletedListener` (influencer lock → influencer default, includes approved revision amounts); `TransferRefundOrderTransactionsListener` (refund).
- **Data:** `wallets` (influencer's default/lock/credit), `order_items` (`influencer_id`, `status`, `pay_price`), `payment_receipts` (withdrawals, `is_income = 0`), `legacy_payouts` (`balance`, `status`), legacy `transactions`.
- **Fee:** `gateways.fee_per_transaction` (column default 20) copied to `payment_receipts.fee` when the payout is requested; applied in `AdminPayoutController@pay`.
- **Mobile:** `src/features/new-wallet/*` (WalletScreen, WalletBalance, PaymentList/PayoutItem, PayoutTutorialModal), `src/features/wallet/*` (old wallet), `src/features/chat/components/OfferCard.tsx` (price × 0.8), translations `src/localization/translations/en/wallet.js`.
