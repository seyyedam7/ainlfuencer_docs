---
id: PAY-05
title: Coupons and credits
domain: Money (wallets, refunds, payouts)
audience: [brand, admin]
apps: [web, email]
related: [OFR-03, PAY-01, PAY-02, PAY-03, SUB-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# PAY-05 · Coupons and credits

> Two ways Ainfluencer gives a brand money off: a **coupon code** typed at offer checkout, and **credit** that staff add to the brand's account. Both are paid for by Ainfluencer: the influencer still receives the full offer price, and if the deal is refunded, the coupon or credit part goes back to Ainfluencer, not to the brand.

## Who and where
- **Brand** in the web app: the coupon field on the offer checkout (`/new-offersV2/:id/pay`, see OFR-03). Credit shows as part of the "Wallet balance" and in the "Credits Received" box on `/profile/transactions` (PAY-01).
- **Admin** in the admin panel: creates coupons, assigns them to specific brands, and adds or removes wallet credit.
- Ainfluencer coupons are **not** used for subscriptions. The Stripe subscription checkout has its own Stripe promotion-code box (SUB-01).

## How it works
**Coupons**
1. The brand adds offers to the checkout and types a coupon code.
2. The system checks the code (rules below). If valid, the coupon is **reserved** for this brand and the discount is shown in the order summary.
3. The brand pays. The coupon use is recorded and the reservation ends.
4. If the brand doesn't pay within **30 minutes** of starting checkout, the order expires and the coupon is released (OFR-03).
5. The brand can remove the coupon only before starting payment (while the order is still a basket).

**How the discount is worked out**
- **Fixed** coupons take off a dollar amount; **percentage** coupons take off a % of the order (offers + content insurance).
- A coupon may have a **maximum discount** per order; the discount is never more than the order total.
- The platform fee is calculated on the offer amount **after** the coupon.
- The discount is split equally across the offers in the order. For each offer, Ainfluencer pays the coupon part into escrow, so the influencer gets the full price.

**Credits**
- An admin "modifies the wallet" of a brand:
  - **Positive amount** → goes into the brand's **credit wallet**. The brand gets an email "You Received a Credit of $X from Ainfluencer!" and an in-app notification. It shows in "Credits Received" ("Bonus credits added to your account.").
  - **Negative amount** → taken from the brand's cash (default) wallet.
- At checkout, **credit is used before cash**.
- Credit can't be withdrawn or refunded to a card.

**Assigned coupons**
- An admin can make a coupon for **specific users**. When the coupon is set to "sent", each added brand gets an email "Your Ainfluencer account credit" with the coupon.

## Rules and limits
A coupon is refused with these messages:
- Unknown code: "Invalid coupon code. Check the code and try again."
- Expired: "The coupon is no longer valid."
- Not assigned to this brand: "This coupon isn't valid for your account."
- Brand already used it the allowed number of times (reservations count too): "You've used this coupon the maximum number of times."
- Total uses reached: "This coupon has reached its maximum usage limit."
- Order too small / too big: "A minimum order of $X is required to use this coupon." / "This coupon can only be applied to orders up to $X."
- Already a coupon on the order: "You can only apply one coupon per order."

Other rules:
- One coupon per order.
- The minimum checkout amount ($10 for offers) applies to the offer + insurance amount, not after the coupon (OFR-03).
- If the order changes so it no longer fits the coupon's minimum/maximum, the coupon is removed automatically.
- **In a refund** (PAY-02) the brand gets back the offer price **minus** the coupon part and **minus** the credit part. Those parts return to Ainfluencer.
- Influencers: staff can also adjust an influencer's credit; a negative influencer credit is a debt the finance team can take out of a later payout (PAY-04).

## Statuses
| Item | Values |
|---|---|
| Coupon type | `fixed`, `percentage` |
| Who can use it | `all_users`, `specific_users` |
| Assignment email | `distribution_status`: `pending` (no email), `sent` (email to added users) |
| Credit adjustment record | `payment_receipts` with gateway `direct_deposit`: `DEPOSIT_PAID` (added) or `WITHDRAW_COMPLETED` (removed) |

## Timers and automatic actions
- `orders:expire-unpaid` (every minute): orders unpaid 30 minutes after checkout started → `expired`, coupon reservation released.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Credit added by staff ("You Received a Credit of $X from Ainfluencer!") | Brand | Email, in-app |
| Brand added to a "sent" specific-users coupon ("Your Ainfluencer account credit") | Brand | Email |

## Common questions

### "My coupon code doesn't work."
**Answer for the user:** Coupons can have an expiry date, a minimum or maximum order amount, a limit on how many times they can be used, or be reserved for specific accounts. The message under the field tells you which one applies. If you think it should work, send us the code and we'll check it.
**What to check:** `coupons` row by `code`: `expired_at`, `usage_type`, `allowed_users`, `limit_usage_per_user`, `limit_usage_total`, `total_usage_count`, `min_order_value`, `max_order_value`; the brand's `coupon_usages` and open reservations.
**Escalate when:** the coupon looks valid for this brand and order but is still refused.

### "It says I've already used the coupon, but I didn't finish paying."
**Answer for the user:** When you apply a coupon it's held for you for 30 minutes during checkout. If the checkout wasn't finished, it's released automatically after that; please try again in a little while.
**What to check:** pending orders for the brand with `coupon_id`, `expired_at`; reservation rows.
**Escalate when:** more than 30 minutes have passed and it's still blocked.

### "Can I remove the coupon or use a different one?"
**Answer for the user:** You can remove it before you start paying. Once payment has started, the order keeps that coupon until it's paid or expires (30 minutes).
**What to check:** `orders.status` (`pre_factor` = can remove).
**Escalate when:** n/a.

### "I got a refund but the coupon amount wasn't included."
**Answer for the user:** The coupon discount was paid by Ainfluencer, so in a refund that part goes back to Ainfluencer and your wallet receives what you actually paid for the offer.
**What to check:** `orders.coupon_discount`, number of offers in the order, refunded amount.
**Escalate when:** the refund is less than what the brand paid for that offer (excluding fee and insurance).

### "I received credit. How do I use it?"
**Answer for the user:** Credit is already part of your wallet balance and is used automatically before your own funds the next time you pay for an offer or other purchase. It can't be withdrawn.
**What to check:** brand's credit wallet balance; "Credits Received".
**Escalate when:** credit was announced but the balance didn't change.

### "Does the influencer get less if I use a coupon?"
**Answer for the user:** No. The influencer receives the full agreed price; Ainfluencer covers the discount.
**What to check:** n/a.
**Escalate when:** n/a.

## Known issues
- The coupon's internal "total discount given" counter is calculated wrongly (it adds the use count instead of the previous total). It affects admin reporting only, not what brands pay. *(to verify in admin panel)*
- Credit used on a deal is lost to the brand if the deal is refunded; brands often expect it back. This is by design.

## Related topics
- OFR-03: checkout where coupons are applied.
- PAY-01: wallet balance and "Credits Received".
- PAY-02: refund amounts.
- PAY-04: influencer debt offset at payout.

## Technical reference
- **Brand endpoints:** `POST /api/{locale}/finance/current-user/coupon` (CouponController@store, `code`), `DELETE /api/{locale}/finance/current-user/coupon/{orderId}` (only `pre_factor` orders).
- **Admin endpoints:** `finance/admin/coupon` resource (AdminCouponController), `POST finance/admin/modify-wallet` (AdminChargeController: `userId`, `amount` ±USD, permission `admin-charge-credit`; positive → credit wallet + `CreditReceivedNotification`; negative → brand default wallet / influencer credit wallet via `PayoutCompletedEvent`).
- **Logic:** `app/Modules/Finance/Services/CouponService.php` (`validateCoupon`, `getDiscountPrice`, `applyCoupon`, reservations), `OrderService::calculateOrderSummary` (fee on subtotal − coupon; auto-remove coupon outside min/max), `TransferOrderTransactionsListener` (coupon share: system user → influencer lock; credit spent first by `WalletService::transfer`), `TransferRefundOrderTransactionsListener` (coupon/credit shares → system), `Listeners/ApplyCouponListener.php`.
- **Data:** `coupons` (`code`, `type`, `value`, `max_discount_per_transaction`, `min_order_value`, `max_order_value`, `expired_at`, `usage_type`, `allowed_users`, `limit_usage_per_user`, `limit_usage_total`, `total_usage_count`, `distribution_status`), `coupon_usages`, coupon reservations, `orders.coupon_id|coupon_discount`, `wallets` (name `credit`), `transaction_reports.credit|discount_by_system`.
- **Job:** `orders:expire-unpaid` (every minute, `Console/ExpireUnpaidOrdersCommand.php`).
