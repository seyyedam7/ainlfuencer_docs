---
id: OFR-03
title: Paying for an offer
domain: Offers and negotiation
audience: [brand, influencer, system]
apps: [web, email, background]
related: [OFR-02, OFR-04, OFR-05, PUB-01, PUB-03, PAY-01, PAY-02, PAY-05, SUB-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# OFR-03 · Paying for an offer

> When an offer shows "Waiting Payment", the brand pays it from the Inbox. The money is **held in escrow** by Ainfluencer (not yet given to the influencer), the offer becomes **Paid & Scheduled**, and a publishing job is created so the influencer can deliver (PUB-01).

## Who and where
- **Brand**, web app only. Inbox `/new-offersV2/:id` → header button **"Pay $&lt;price&gt;"** (only when the status is "Waiting Payment") → checkout page `/new-offersV2/:id/pay` (title "Checkout") → success page `/new-offersV2/:id/payment-successful`.
- **Influencer** sees the result in the mobile chat: card "The Brand has just paid for this offer" and status "Paid & Scheduled".

## How it works
1. **Brand clicks "Pay $…"**. The web app first checks the agreed delivery date:
   - **Less than 24 hours away** → pop-up "Update Date & Make Payment" ("Less than 24 hours left until the publication date…"). The brand picks a new date and clicks "Update & Pay".
   - **Already passed** → pop-up "Update Date and Make Payment" ("The delivery date has expired. In order to make payment, please update the delivery date.").
   - The new date must be at least about 24 hours ahead ("You can't expect influencers to publish your content today. Select any date in 24 hours (tomorrow onwards)…"). See OFR-04.
2. The app empties the brand's basket, puts this one offer in it and opens the checkout.
3. **Checkout page** ("Pay safely with Escrow"):
   - **"What you're paying"**: offer price + **platform fee** + optional **insurance** − coupon − wallet. The tooltip "HOW YOUR TOTAL IS CALCULATED" lists "Offer price", "Platform fee", "Insurance (optional)", "Discount", "Wallet credit", "Total due".
   - **Platform fee** depends on the brand's plan. The page copy says "Free: 25% · Starter: 20% · Pro: 15% · Ultimate: 3%"; the real rate is the plan's "collab transaction fee" setting (SUB-04). An upgrade banner shows how much Ultimate would save.
   - **"Have a coupon?"**: one coupon per order (PAY-05).
   - **Insurance** (toggle): for physical products shipped to the creator. The brand enters the product value; the fee is a percentage of that value ("Insurance fee (10%)" on screen; the server uses its configured percentage, default 20% *(to verify which applies in production)*). Covers "Product not returned" and "No-show creators". Not part of the fee base.
   - **"Pay from wallet"** toggle: uses wallet balance first (credit, then cash). If the wallet covers everything, the button becomes **"Confirm & Pay with Wallet"**.
   - **"Payment Method"** ("CHOOSE ONE"): **"Credit Card"** (saved card), **"PayPal"**, **"Stripe"** (hosted checkout), **"Crypto"**. Which ones actually work depends on which payment gateways the team has switched on.
   - Tick the terms box ("By clicking this, I agree to Ainfluencer Terms of Use and Privacy Policy"), then **"Pay $&lt;total&gt;"**.
4. **What happens next:**
   - **Wallet or saved card:** charged immediately. Toast "The payment was made successfully. Continue communicating with the influencer regarding the post and your expectations." and the success page, which also suggests other influencers waiting for payment.
   - **PayPal / Stripe / Crypto:** the brand is sent to the provider, then back to an Ainfluencer result page "Payment successful" / "Payment failed" with "Return to Dashboard". The offer is marked paid when this return page confirms the payment.
5. **When payment succeeds** the system: moves the offer price into the **influencer's locked (escrow) wallet**, takes the platform fee and any insurance, marks the order paid, sets the offer to **paid** (applying the new date if one was chosen), creates the **publish job** (scheduled, manual posting), posts the "paid" card in chat, and notifies both sides. Paying also unlocks the influencer's username for the brand (DSC-01).

## Rules and limits
- Only offers in **"Waiting Payment"** (`awaiting-payment`) can be paid.
- **Minimum payment $10** (goods total incl. insurance): "Minimum payment amount is $10. Please adjust your order."
- The checkout is valid for **30 minutes** after the brand first clicks pay; after that: "This order has expired and cannot be paid." Start again from the Inbox.
- Card payments that need extra bank verification (3-D Secure) are **not supported** inside checkout: "Additional verification required. Please complete authentication to proceed." Use another card, the wallet or Stripe/PayPal.
- Paying part by wallet and part by card requires a card to be selected: "Insufficient wallet balance. Please select PayPal or a credit card to complete the payment."
- Influencer payout share: the influencer later receives the price minus Ainfluencer's cut (PAY-03). The brand's platform fee is separate.

## Statuses
| What the user sees | Internal | Meaning |
|---|---|---|
| Brand "Waiting Payment" / influencer "Waiting Brand Approval" | offer `awaiting-payment`; order `pre_factor` (basket) | Not paid yet |
| (checkout open) | order `pending`, `expired_at` = +30 min | Payment in progress |
| Brand "Scheduled" / influencer "Paid & Scheduled" | offer `paid`; order `paid`; order item `transferred`; publish job `scheduled`/`healthy` | Money in escrow, influencer must deliver |
| – | order `expired` | Checkout abandoned after 30 min; coupon released |

## Timers and automatic actions
- **Every minute** (`orders:expire-unpaid`): pending orders past their 30 minutes become expired and their coupon reservation is released.
- **Abandoned checkout emails:** when the brand opens checkout, two reminder emails are scheduled: after **3 hours** ("Payment Required for Your Deal with …") and after **24 hours**, each sent only if the offer is still awaiting payment and only once per offer. (The abandoned-checkout **SMS is switched off** in the current code.)
- Awaiting-payment reminders 1 and 3 days after the offer: see OFR-01.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Offer paid | Influencer | Email, push, in-app ("offer paid"); push opens the chat |
| Offer paid | Brand | Email/in-app per template |
| Influencer has no PayPal payout set up when paid | Influencer | "Connect PayPal" email |
| Checkout abandoned | Brand | Email after 3 h and 24 h |
| Paid card in chat | Both | Chat card (no chat push) |

## Common questions

### "Why do I have to change the date before I can pay?"
**Answer for the user:** The influencer needs at least a day to prepare. If the delivery date is less than 24 hours away or has passed, please choose a new date; you'll go straight to checkout afterwards.
**What to check:** `offers.start` vs now.
**Escalate when:** the date is days away and the pop-up still appears.

### "My payment went through but the offer still says Waiting Payment."
**Answer for the user:** Sorry about that. Your money is safe in your Ainfluencer wallet. You can pay the offer from your wallet balance now ("Pay from wallet"), or we can check it for you.
**What to check:** the brand's wallet top-ups/payment receipts (a `deposit_paid` receipt for the order) vs the order status (`pending`/`expired`). This happens when a PayPal/Stripe/crypto payment finished but the return page didn't confirm it (user closed the tab), or Stripe confirmed it first.
**Escalate when:** the money is not in the wallet either, or the brand was charged twice. Escalate to finance/dev.

### "It says the order expired."
**Answer for the user:** A checkout stays open for 30 minutes. Please go back to the chat and click Pay again.
**What to check:** order `expired_at`, status `expired`.
**Escalate when:** the brand was charged for the expired order.

### "My card is declined with 'Additional verification required'."
**Answer for the user:** Your bank wants to confirm the payment, which our card checkout can't do yet. Please use another card, pay with Stripe or PayPal, or add funds to your wallet first.
**What to check:** the failed payment receipt message.
**Escalate when:** several different cards fail with other errors.

### "What is the platform fee and why is it so high?"
**Answer for the user:** It covers escrow, payment processing and platform services, and depends on your plan (Free plans pay the most, Ultimate the least). It's shown before you pay.
**What to check:** the brand's plan (SUB-01) and `orders.fee_amount`.
**Escalate when:** the fee shown doesn't match the plan.

### "Do I need insurance?"
**Answer for the user:** Only if you ship a physical product to the influencer. It reimburses the product if the creator keeps it without delivering. Digital collaborations don't need it.
**What to check:** `orders.insurance_amount`, `insurance_base_amount`.
**Escalate when:** an insurance claim is made (finance team).

### "Can I get my money back if the influencer doesn't deliver?"
**Answer for the user:** Yes. Your payment is held in escrow; if the influencer doesn't deliver, it is refunded to your Ainfluencer wallet (PAY-02).
**What to check:** publish job status and timers (PUB-03).
**Escalate when:** a refund is overdue.

## Known issues
- **Redirect payments not always confirmed.** For PayPal, Stripe hosted checkout and crypto, the offer is marked paid only when the brand comes back to the result page. If they close the tab, or if Stripe's confirmation reaches us before the return page, the money is credited to the **wallet** but the **offer stays "Waiting Payment"**. Workaround: pay again from the wallet. Escalate to dev if the money isn't in the wallet.
- If moving the money fails after a successful charge, the order is marked refunded, but the offer may still be set to paid *(to verify)*. Escalate to dev.
- Insurance percentage: the screen says 10%, the server default is 20% *(to verify the production setting)*.

## Related topics
- OFR-02 negotiation before payment; OFR-04 changing the date; OFR-05 paying a revision difference.
- PUB-01 what the influencer does next; PUB-03 release and refund of escrow.
- PAY-01 brand wallet and cards; PAY-02 refunds; PAY-05 coupons; SUB-04 plan-based fee.

## Technical reference
- **Endpoints** (Finance module, `app/Modules/Finance/Routes`): `GET/POST /api/{locale}/finance/current-user/basket`, `POST …/basket/empty`, `POST …/basket/insurance` (`BasketController::setInsurance`, `config('finance.order.insurance_percentage')`, default 20), `POST/DELETE …/finance/current-user/coupon`, `POST /api/{locale}/finance/payment` (`PaymentController::handle`), `GET /api/v3/offers/{offer}/abandoned-tracking` (`app/Services/Implementation/v3/PaymentTrackingService.php`), browser returns `finance/payment/callback/{success|failed}/{tracking}` (`PaymentCallbackController`), Stripe webhook `POST /api/en/finance/stripe/webhook` (`StripeWebhookController`, credits the receipt only; no `OrderPaidEvent`). Date change: `PUT /api/v2/offers/{offer}/update-start` (OFR-04).
- **Money logic:** `OrderService::calculateOrderSummary()` (fee = plan `collab_transaction_fee_percent` on subtotal − coupon via `app/Modules/Payment/Helpers/AmountHelper.php`; insurance excluded from fee base); `PaymentController::MINIMUM_PAYMENT_AMOUNT` = 10; order lifetime `config('finance.order.lifetime')` = 30 min; card charge `Gateway/StripeGateway::recharge()` (`requires_action` → error).
- **On paid:** `OrderPaidEvent` → `TransferOrderTransactionsListener` (brand → influencer LOCK wallet, fee, insurance; order `paid`), `ApplyCouponListener`, `app/Modules/Offer/Listeners/OfferPaidListener.php` (offer `paid`, start from order item `metadata.start`, publish job create/update, `ConversationSendOfferUpdateJob` `offer:paid`, notifications, `ConnectPaypalEmail`).
- **Data:** `orders` (`status` pre_factor/pending/paid/expired/refunded, `expired_at`, `fee_amount`, `insurance_amount`, `coupon_id`, `total_pay_price`; amounts in cents), `order_items` (`item_type = Offer`, `status`, `metadata.start`), `payment_receipts` (`status`, `gateway_id`, `creator_type`), `offers.status`, `publish_jobs`.
- **Jobs:** `orders:expire-unpaid` (every minute); abandoned emails `FirstAbandonedNotification` (+3 h) / `SecondAbandonedNotification` (+24 h), SMS lines commented out.
- **Web:** `src/components/pages/inboxV37/InboxV37ChatArea.tsx` (`makePaymentHandler`), `src/components/pages/checkoutV2/*`, `src/components/pages/paymentSuccessful`, copy in `src/lib/localization/locales/en/pay.json`.
