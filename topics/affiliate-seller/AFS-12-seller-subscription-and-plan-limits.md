---
id: AFS-12
title: Seller subscription and plan limits
domain: Seller app (brands)
audience: [seller, admin, system]
apps: [seller-app, background]
related: [AFS-02, AFS-03, AFS-05, AFS-06, AFS-07, AFS-08, AFS-10, AFS-13, AFA-09, AFC-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-12 · Seller subscription and plan limits

> Sellers on the affiliate platform are on a plan: Free, Starter, Pro (shown as "Professional"/"Pro"), Ultimate, or a Custom plan made by an admin. The plan sets daily and total limits and the Ainfluencer fee on commissions. Paid plans are billed by Stripe. Hitting a limit opens an "upgrade" pop-up; moving to a lower plan or cancelling automatically switches off whatever is over the new limits.

## Who and where
- **Seller**, in the seller app:
  - **Manage Subscription** page (`/app/manage-subscription`). Reached from the sidebar "Upgrade to Premium" widget, the "Upgrade Plan Now" button in any limit pop-up, the profile overview, and the bulk-invite pop-up.
  - Toggle at the top: "Monthly", "3 months", "Yearly".
  - When on a paid plan: "View Billing" (subscription statements), "Cancel Plan" or "Resume Plan".
  - "Get Your Custom Plan" opens a note to "schedule a meeting" with a "Book on Website" button (no self-service custom plan).
  - Subscription charges: **More → Payments → Subscription Statements** (`/app/payment-history/subscription`), tabs for charges and invoices (columns "ID", "Date", "Card Details", "Amount", "Status").
  - User menu shows "Daily Invites", "Daily Email Revealed", "Daily New Conversations" as used / limit.
- **Admin**: creates custom plans in the admin panel; the seller gets an email.
- **System**: Stripe webhooks keep the subscription in sync and trigger the automatic downgrade clean-up.

## How it works
1. **Free → paid.** The seller clicks the plan's button ("Join Pro", "Join Ultimate", "Upgrade to Starter"). The app shows "redirecting to Stripe..." and opens Stripe Checkout (promotion codes allowed). After payment Stripe returns the seller to the app and the plan becomes active. If the plan has no price for the chosen period: "This billing option is not available for checkout."
2. **Upgrade (higher plan).** Pop-up "Upgrade to {plan}?", "Yes, Upgrade" / "No". The change is **immediate** and Stripe **charges the difference right away**. Any downgrade that was scheduled is cancelled.
3. **Same plan, different period.** Monthly → longer period uses the same upgrade pop-up. Annual or quarterly → monthly shows "Switch to {plan} Monthly Plan?" ("Switch to Monthly" / "Stay on Annual Plan"). This switch is **immediate**; the unused part of the longer plan stays as a credit at Stripe for future payments.
4. **Downgrade (lower paid plan).** Pop-up "Confirm Downgrade to {plan}?" with a required tick box and "Proceed with Downgrade". The change is **scheduled for the end of the current paid period**; until then the seller keeps the current plan.
5. **Cancel.** "Cancel Plan" → "Plan Cancellation" pop-up with a required tick box → "Cancel Plan". The plan stays active until the period ends; the card on the active plan then reads "Active Until: {date}". While that date hasn't passed, "Resume Plan" undoes the cancellation. There is no button to move to Free directly: for paid sellers the Free card shows "Expired".
6. **When the new plan takes effect** (period end for downgrade/cancel, immediately for upgrade), the system runs the **automatic clean-up** (see Rules) and notifies affected affiliates.
7. **Hitting a limit.** Anywhere in the app, when an action is over the plan limit the backend answers "payment required" and the app opens a pop-up with a title such as "Brand Limit Reached", the current limit, and "Upgrade Plan Now".

## Rules and limits
- **What is limited** and how it's counted:
  - Affiliate invitations: invitations sent **today** (UTC day). Pop-up "Daily Invitation Limit Reached".
  - New conversations (affiliate chat, AFC-01): distinct people first messaged **today** (UTC). "Daily Limit Reached".
  - Active contracts: all currently active partnerships. "Active Contract Limit Reached".
  - Active brands: active brands **plus** active Shopify stores. "Brand Limit Reached".
  - Seller (Amazon) accounts: all connected Amazon accounts. "Seller Account Limit Reached".
  - Minimum commission: the lowest commission % allowed. Pop-up "Upgrade to Ultimate".
  - Custom (private) commission on an invitation: "Private Commission Feature Limit".
  - Promotions: "Promotion Creation Limited".
  - Email campaigns (AFS-08), email reveals (AFS-05), Ainfluencer fee % (AFS-10), campaigns (AFX-02).
- **Seeded plan values** (from the production seed file; live values are in the admin panel and may differ):

  | | Free | Starter | Pro | Ultimate |
  |---|---|---|---|---|
  | Price (month / 3 months / year) | $0 | $49 / $129 / $399 | $149 / $399 / $1,249 | $699 / $1,799 / $5,999 |
  | Active brands | 1 | 2 | 5 | Unlimited |
  | Affiliate invites per day | 1 | 50 | 500 | 50,000 |
  | New conversations per day | 5 | 25 | 100 | Unlimited |
  | Active contracts | 5 | 25 | 200 | Unlimited |
  | Team members | 0 | 2 | 3 | Unlimited |
  | Minimum commission | 25% | none | none | none |
  | Ainfluencer fee (plan value) | 25%* | 20% | 15% | 3% |

  \*On Free the code ignores the plan value and uses the platform default (`SELLER_DEFAULT_APPLICATION_FEE`, 5% unless set otherwise). Some paid-plan limits are stored "per month" but the code counts them **per UTC day**.
- **A limit that is not set on a plan counts as "not included"** (pop-up). In the seed file, the current paid plans don't list promotions, private commission, email campaigns, ASIN or seller-account limits *(to verify against the live plans)*.
- **Team-member limit is not enforced** by the backend (AFS-13).
- **Auto-approval follows the subscription:** when a paid subscription becomes active, "automatic partnership approval" is switched **on for all the seller's brands**; when there's no active subscription it's switched **off** for all (AFS-06). This overrides the seller's own choice.
- **Failed renewal:** while Stripe shows the subscription as past due or unpaid, the seller is treated as Free for limits and fee, and auto-approval is switched off.
- **Data deletion:** the pop-ups say data is "permanently deleted 30 days after my billing period ends". No code that deletes data after 30 days was found; only the clean-up below happens.
- **Custom plans** are created only by an admin (seller sees "Get Your Custom Plan" → "Book on Website").

## Automatic clean-up on downgrade or cancellation
Runs whenever the seller's subscription price changes or the subscription ends. For each limit of the **new** plan, the **lowest-selling** items above the limit are switched off:
- **Active contracts** over the limit → revoked; the affiliate's links stop working; seller and affiliate are notified.
- **Brands** over the limit → set inactive; their products too; affiliates partnered on them get "brand deactivated" / "product deactivated" emails.
- **Products** (ASIN limit) over the limit → set inactive (sales after that earn no commission).
- **Amazon accounts** over the limit → disconnected, their profiles and brands inactive.
- **Promotions** → deleted if the new plan doesn't include promotions (or private promotions); affiliates' links lose the coupon (they're notified).
- **Contracts with a custom commission** → revoked if the new plan doesn't include private commission.
- **Commission rates below the new plan's minimum** → raised to the minimum on the products **and on every contract** for those products (for example, moving to Free with a 25% minimum raises a 15% commission to 25%). This raises future commission charges (AFS-10).
- Nothing is switched back on automatically after an upgrade; the seller re-activates items by hand.

## Statuses
| What the seller sees | Meaning (`subscriptions.stripe_status`) | Next |
|---|---|---|
| "Active" + "Renewal date: …" | `active`, renews automatically. | — |
| "Active Until: {date}" + "Resume Plan" | `active` with an end date (cancelled, still in paid period). | Ends on that date unless resumed. |
| Free plan shown as active | No active subscription (`canceled`, or none). | Free limits apply. |
| Treated as Free | `past_due`, `unpaid`, `incomplete`, `incomplete_expired`. | Seller updates the card in Stripe. |

## Timers and automatic actions
| When | What | User notices |
|---|---|---|
| Immediately on upgrade / switch to monthly | Stripe swaps the plan and invoices. | New charge in Subscription Statements. |
| End of paid period | Scheduled downgrade or cancellation takes effect, then the clean-up runs. | Items switched off; affiliates notified. |
| Each renewal | Stripe charges the card. | Charge in Subscription Statements. |
| Midnight UTC | Daily invite and new-conversation counters reset. | Counters in the user menu go back to 0. |
| Rolling 24 h | Free email-reveal quota counts reveals in the last 24 hours (not per calendar day, AFS-05). | Quota frees up 24 h after each reveal. |

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Admin creates a custom plan | Seller | Email |
| Clean-up revokes a contract | Seller and affiliate | Email (queued, sent at 06:00/18:00 UTC) |
| Clean-up deactivates a brand / product | Partnered affiliates | Email (queued) |
| Clean-up removes a coupon from a link | Affiliate | Email (queued) |
| Subscription payment receipts | Seller | Stripe's own emails (if enabled in Stripe) |

## Common questions

### "I upgraded but I'm still blocked by a limit."
**Answer for the user:** Upgrades apply right away. Please refresh the page; if the pop-up still appears, tell us which action you were doing.
**What to check:** `subscriptions` row for the seller: `stripe_status` = active, `stripe_price` = the new price, `ends_at`; then the limit for that feature on the plan price (`plan_price_feature`).
**Escalate when:** the subscription is active on the new price and the feature is missing or 0 on that plan.

### "I downgraded but I still have my old plan."
**Answer for the user:** Downgrades take effect at the end of the period you've already paid for, so you keep your current features until then.
**What to check:** Stripe subscription schedule; `subscriptions.stripe_price` unchanged until period end.
**Escalate when:** the period has ended and the plan didn't change.

### "After cancelling, my brands/products/contracts were switched off."
**Answer for the user:** When a plan ends, anything above the new plan's limits is switched off automatically, starting with the items with the lowest sales. You can switch items back on within your current limits, or upgrade.
**What to check:** `seller_brands.status`/`inactive_at`, `seller_products.status`/`inactive_at`, `partnership_contracts.status` = revoked with `revoked_at` matching the subscription end.
**Escalate when:** more items were switched off than the new limits require.

### "Why did my commission rates go up?"
**Answer for the user:** Your new plan has a minimum commission, so rates below it were raised automatically on your products and existing partnerships.
**What to check:** product `commission` values equal the plan minimum; timing matches the plan change.
**Escalate when:** the seller disputes commission charged at the raised rate (finance), or rates were raised on a paid plan without a minimum.

### "Will my data be deleted 30 days after I cancel?"
**Answer for the user:** Your account and data stay; items above the Free plan limits are switched off when the paid period ends.
**What to check:** nothing to check; no automatic deletion exists.
**Escalate when:** the user reports data actually missing.

### "Why do I keep getting 'Daily Invitation Limit Reached'?"
**Answer for the user:** Your plan allows a set number of invitations per day (resets at midnight UTC). Upgrade for more, or continue tomorrow.
**What to check:** invitations created today (`partnership_requests`, type seller-to-affiliate, `created_at` today UTC) vs the plan's `affiliate_invite` value.
**Escalate when:** the count is below the limit and the pop-up still appears.

### "How do I get a custom plan?"
**Answer for the user:** Click "Get Your Custom Plan" → "Book on Website" to schedule a call; our team sets it up and you'll get an email.
**What to check:** —
**Escalate when:** sales/admin team needs to create it.

## Known issues
- **Plan card text and enforced limits can differ.** The marketing text on the plan cards (for example "/ month") is stored separately from the limits the system enforces; the code counts invites and conversations per UTC day.
- **Limits missing on a plan are treated as "not included"** *(to verify against live plan data)*: a paid seller may get "Upgrade Needed" or "Promotion Creation Limited" pop-ups even on a high plan, and a plan change may switch off items for a limit that is simply missing. Escalate to dev with the plan name.
- **Clean-up also runs on upgrades** (any price change); with higher limits it normally does nothing, but see the item above.
- **Commission raise ignores commission type** *(to verify)*: a fixed-amount commission below the minimum is set to the minimum number (for example $5 → 25). Escalate to dev.
- **Auto-approval is overwritten** for all brands on every subscription start/end.
- **The tick box text promises deletion after 30 days**; this does not happen.
- **Fee on Free plan** uses the platform default, not the 25% stored on the Free plan (the commission statement tooltip also quotes 3.5% for Ultimate). See AFS-10.

## Related topics
- AFS-10 — commission billing; the plan sets the Ainfluencer fee.
- AFS-06 — partnerships, contract limits, auto-approval.
- AFS-03 — products, brands, commission (brand/ASIN/commission limits).
- AFS-02 — store connections (seller-account limit).
- AFS-05 — email reveals (daily quota, then wallet).
- AFS-07 — promotions.
- AFS-08 — email campaigns (plan feature).
- AFS-13 — team members.
- AFA-09 — the affiliate's own plans.

## Technical reference
- **Endpoints:** `seller/v1/plans` (GET list, levels ≤ 4), `plans/active`, `plans/subscriptions/status`, GET `plans/{stripePriceId}/subscribe` (Checkout URL; refused if already subscribed), PUT `plans/{stripePriceId}/update`, PUT `plans/cancel`, PUT `plans/resume` → `app/Http/Controllers/Seller/V1/Plan/*` → `app/Services/General/Plan/PlanManager.php` (upgrade/same level: `swapAndInvoice`; downgrade: `PeriodEndPlanDowngrader::scheduleAtPeriodEnd` via Stripe subscription schedules; cancel/resume: Cashier `cancel()`/`resume()`).
- **Subscription statements:** `seller/v1/payments/charges`, `seller/v1/payments/invoices` (from `stripe_charges`, `stripe_invoices`).
- **Limit checks (HTTP 402):** middleware `seller_access_feature:<feature>` (`app/Http/Middleware/SellerAccessFeatureMiddleware.php`, skipped when `IS_CHECK_PLAN_FEATURES=false` or the request sets `status=inactive`) → `PlanAccessManager::handleForSeller` → checkers in `app/Services/General/Plan/PlanAccess/Seller/Checkers/`. Used on partnership invites (`affiliate_invite`, `active_contract`, `private_commission`), product/brand status (`brand`, `asin`), commission edits (`commission`), promotions (`promotion`, `private_promotion`), chat (`chat_system`), Amazon sync (`seller_account`), email campaigns (`email_campaign`). Missing feature value → `null` → denied.
- **Clean-up:** `app/Observers/Handlers/Subscription/UpdatedSubscriptionObserver.php` (on `stripe_status` → `canceled` or `stripe_price` change) → `app/Services/General/Plan/Downgrade/SellerPlanDowngradeService.php` → `Downgrade/Seller/*FeatureDowngrader.php` and `Downgrade/Seller/Cascade/Nodes/*`. Auto-approval toggling in the same observer and in `Created/DeletedSubscriptionObserver`.
- **Custom plan:** admin `admin/v1/plans/{userTypeId}/custom` → `ProcessCustomPlanJob`, `CustomPlanNotification` (email).
- **Data:** `subscriptions` (Cashier: `user_type_id`, `stripe_status`, `stripe_price`, `ends_at`), `plans` (`id`: seller Free 3, Starter 12, Pro 4, Ultimate 5, Custom 10; `level`), `plan_prices` (`stripe_product_price_id`, `duration`, `is_active`), `plan_price_feature` (`value`, scalar or JSON `{"value":…}`; -1 = unlimited, 0 = not included), `features` (`name`, `user_type`), `seller_brands.auto_partnership_approval`.
- **Seed data:** `database/seeders/data/seller_plan_prices_data_production.php`, `seller_plan_price_feature_data_production.php`.
- **UI:** `shared-components/algo-manage-subscription/*`, limit pop-up texts in `shared-components/store/authorization.store.js` (`SELLER_DAILY_LIMIT_TYPE`).
