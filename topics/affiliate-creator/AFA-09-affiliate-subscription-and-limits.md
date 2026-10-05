---
id: AFA-09
title: Affiliate subscription and limits
domain: Affiliate app (creators)
audience: [affiliate, admin, system]
apps: [affiliate-app, background]
related: [AFA-04, AFA-06, AFA-08, AFS-12, AFC-02]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-09 · Affiliate subscription and limits

> Affiliates can stay on the Free plan or pay monthly or yearly for Pro or Ultimate. Plans define limits on applications, active contracts, landing templates, chat and team members. Today those limits are mostly **not enforced** while using the app, but a **downgrade or cancellation does remove** extra contracts, links and templates.

## Who and where
- **Affiliates**: "Manage Subscription" (`/app/manage-subscription`), reached from the sidebar upgrade widget ("Upgrade to Premium") or the profile. The header badge shows Free or Pro.
- Plan invoices: "Payments" → "Subscription" tab (`/app/payment-history/subscription`).
- **Admins** can give an affiliate a custom plan.
- This is a separate subscription from the Ainfluencer brand plans (SUB-01) and seller plans (AFS-12).

## How it works
1. **Choose a plan.** The page lists Free, Pro and Ultimate with a "Monthly" / "Yearly" switch ("Join Pro", "Join Ultimate", "Upgrade to …", "Downgrade to …"). For a tailored plan there's "Book on Website" / "Get Your Custom Plan".
2. **Subscribe.** Picking a paid plan opens **Stripe Checkout**. When payment succeeds, Stripe tells us and the plan becomes active.
3. **Upgrade** (to a higher plan or from monthly to yearly): takes effect **immediately**, and the price difference is charged at once.
4. **Downgrade** to a lower paid plan: a confirmation ("Confirm Downgrade to …?") asks the user to remove extra team members and contracts first, with the checkbox "I confirm that I have removed my excess team members and reduced my active contracts." The change is **scheduled for the end of the paid period**.
5. **Cancel** ("Cancel Plan"): "Confirm Plan Cancellation?" explains the account will be **moved to the Free Plan**. The plan stays active until the period ends ("Active Until: …"). Until then "Resume Plan" undoes the cancellation.
6. **When the lower plan starts** (end of period after a downgrade or cancel), the system automatically:
   - revokes **active contracts above the new limit**, starting with the ones with the **lowest sales**. Their links stop working and both sides get an email;
   - deletes **landing templates above the new limit** (AFA-06).

## Rules and limits
Default plan values (from the setup data; live values are on the plans page):

| Feature | Free | Pro | Ultimate |
|---|---|---|---|
| Price | $0 | $49/month or $490/year | $99/month or $990/year |
| Partnership applications ("Daily Applications") | 10 / day | 30 / day | 50 / day |
| Active contracts | 100 | 300 | Unlimited |
| Custom landing templates | 1 | 5 | Unlimited |
| New chat conversations | 10 / day | 30 / day | 50 / day |
| Private (custom) commission | No | Yes | Yes |
| Team members | No | 10 | Unlimited |

- **Enforcement today:** the checks that should block applications, contracts, templates and chats over the limit are **switched off** in this version. The counters in the user menu are shown, but users aren't stopped. Only the **downgrade/cancel clean-up** applies limits.
- Custom plans are created by an admin and may have any values.
- The plan does **not** change the platform fee on commissions. That fee depends on the **seller's** plan (AFA-08).

## Statuses
| Shown | Meaning |
|---|---|
| Active | Paid plan running; "Renewal date: …" |
| Active Until: … | Cancelled, still active until that date; can be resumed |
| Free | No paid plan |

Internal subscription statuses (Stripe): active, trial, past_due, unpaid, incomplete, canceled, expired, and others.

## Timers and automatic actions
| When | What |
|---|---|
| Immediately on upgrade | Plan changes, difference charged |
| End of billing period | Scheduled downgrade or cancellation takes effect; clean-up of extra contracts, links and templates |

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Contract revoked by downgrade | Affiliate and seller | Email (batched 06:00/18:00 UTC) |
| Promotion removed from a link by downgrade | Affiliate | Email |
| Custom plan assigned by admin | Affiliate | Email |

## Common questions

### "I cancelled. When do I lose my features?"
**Answer for the user:** Your paid plan stays active until the end of the period you paid for. After that you move to the Free plan. You can click "Resume Plan" any time before then.
**What to check:** The subscription's end date.
**Escalate when:** The plan ended before the paid period.

### "After my plan ended, some partnerships and links stopped working."
**Answer for the user:** The Free plan allows fewer active contracts, so the system ended the extra ones automatically, starting with those with the lowest sales. You can upgrade again, or re-apply to the brands you want to keep.
**What to check:** Contracts revoked around the plan end date; the active-contract count vs. the new limit.
**Escalate when:** Contracts were revoked while the user was within the new plan's limit.

### "I was charged straight away when I changed plan."
**Answer for the user:** Upgrades take effect immediately and the difference is charged straight away. Downgrades only take effect at the end of your current period.
**What to check:** Stripe invoices on the "Subscription" tab.
**Escalate when:** A downgrade was charged immediately.

### "The menu says I've used my daily applications, but I can still apply."
**Answer for the user:** The counter is informational. You can continue applying.
**What to check:** —
**Escalate when:** —

### "Does a paid plan lower the fee on my commissions?"
**Answer for the user:** No. The platform fee on commissions depends on the brand's plan, not yours.
**What to check:** —
**Escalate when:** —

## Known issues
- Plan limits are not enforced during use, but the downgrade clean-up still applies them. Users can build up more than the Free plan allows and then lose the extra at the end of a paid period.
- *(To verify)* When a user who is also a **seller** starts or ends an affiliate subscription, the code also switches automatic partnership approval on or off for that user's **seller brands**.
- The limit pop-up ("upgrade required") rarely appears for affiliates because the checks are off.

## Related topics
- AFA-04 · Contracts affected by limits.
- AFA-06 · Landing templates affected by limits.
- AFA-08 · Commission payouts (fee set by the seller's plan).
- AFS-12 · Seller plans.
- AFC-02 · Email batching.

## Technical reference
- **Endpoints:** `GET affiliate/v1/plans`, `GET affiliate/v1/plans/active`, `GET affiliate/v1/plans/subscriptions/status`, `GET affiliate/v1/plans/{id}/subscribe` (Stripe Checkout URL), `PUT affiliate/v1/plans/{id}/update`, `PUT affiliate/v1/plans/cancel`, `PUT affiliate/v1/plans/resume`; invoices `GET affiliate/v1/payments/invoices`, `.../charges`.
- **Logic:** `Services/General/Plan/PlanManager::updatePlanSubscription` (upgrade = `swapAndInvoice`; downgrade = `PlanDowngrader::scheduleAtPeriodEnd`), `cancelPlanSubscription` (Cashier cancel at period end), `resumePlanSubscription`.
- **Clean-up:** `Observers/Handlers/Subscription/UpdatedSubscriptionObserver` (on `stripe_status → canceled` or `stripe_price` change) → `Services/General/Plan/Downgrade/AffiliatePlanDowngradeService` → `ActiveContractFeatureDowngrader` (lowest `attributed_sales_14d` first), `CustomLandingPageFeatureDowngrader`, cascade nodes `PartnershipContractNode`, `AffiliateProductLinkNode`, `LandingPageNode`.
- **Limits not enforced:** `affiliate_access_feature:*` middleware commented out in `routes/affiliate/v1/{partnership,landing_page,chat}.php`. Checkers exist in `Services/General/Plan/PlanAccess/Affiliate/Checkers/*`.
- **Data:** `subscriptions` (`stripe_status`, `stripe_price`, `ends_at`), `plans`/`plan_prices`/plan-price features (seed: `database/seeders/AffiliatePlanTableSeeder.php`, `AffiliatePlanPriceFeatureTableSeeder.php`).
- **Frontend:** `shared-components/algo-manage-subscription/*`, `spa-affiliate/src/components/manage-subscription/Modals/*`.
