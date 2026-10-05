---
id: SUB-04
title: Plan limits ("upgrade" pop-ups)
domain: Subscriptions and plan limits
audience: [brand, system]
apps: [web, background]
related: [SUB-01, SUB-02, CMP-01, CMP-03, CMP-05, DSC-01, DSC-03, BRD-03, OFR-03, PUB-04, CMP-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# SUB-04 · Plan limits ("upgrade" pop-ups)

> Each plan allows a number of campaigns, influencer invites, team seats, username reveals and analytics reports, and sets the brand's platform fee on offers. When a brand tries to go over a limit, the action is refused and an upgrade pop-up opens. Some features listed in the plan comparison are display only and not enforced.

## Who and where
- **Brand** in the web app. The limit pop-up can appear on any page where a limited action happens: creating or duplicating a campaign, switching a campaign back on, inviting influencers (search, profiles, favourites), inviting team members, launching a gifting campaign, revealing usernames, requesting advanced analytics.
- Usage is shown on `/subscription-plans` (current plan card, "Resets in N days") and on `/profile/transactions?tab=billing` ("Active campaigns", "Invites this month", "Team seats").

## How it works
1. The brand does a limited action.
2. The system checks the brand's current plan (Free if there is no active paid subscription).
3. If the action fits, it goes ahead and the usage counter goes up (if the action then fails, the count is given back).
4. If not, the action is refused and the web app opens the **limit pop-up**, with a title such as "Campaign Limit Reached", "Influencer Invite Limit Reached", "Seats Limit Reached", "Username Reveal Limit Reached", "Analytics Limit Reached" or "Commission Limit Reached", a usage meter, the plan cards (Monthly / Quarterly / Yearly) and links to `/subscription-plans`. "Need a custom solution? Meet our sales team" opens a booking page.
5. For invites, a special pop-up "You've hit your monthly invite limit" says "You've selected X, but only Y invites are left this cycle." and offers "Upgrade and invite all X" or **"Invite Y now, stay on {plan}"**.

## Rules and limits
**Allowances per plan** (from the plan configuration seed data; the live values are in the `plan_price_feature` table and should match):

| | Free | Starter | Pro | Ultimate |
|---|---|---|---|---|
| Campaigns held at once | 1 | 2 | 3 | Unlimited |
| Influencer invites / month | 5 | 50 | 500 | 50,000 |
| Team seats (incl. owner) | 1 | 1 | 3 | Unlimited |
| Username reveals / month | 5 | 50 | 100 | 6,000 |
| Extra reveal price (from wallet) | $1.00 | $0.75 | $0.75 | $0.50 |
| Advanced analytics reports / month | 3 | 30 | 100 | 1,000 |
| Extra report price (from wallet) | $1.00 | $0.75 | $0.75 | $0.25 |
| Product seeding (gifting) launch | No* | Yes | Yes | Yes |
| **Platform fee on offers** (paid by the brand on top of the offer price) | **25%** | **20%** | **15%** | **3%** |
| Affiliate: minimum commission / connected stores | 25% / 1 | any / 2 | any / 5 | any / unlimited |

\* The Free plan card says "Product Seeding · basic access", but the configuration gives Free no gifting-launch access *(to verify against the live table)*.

**What counts**
- **Campaigns**: campaigns the brand is *holding*: live (active + approved) or submitted and waiting for review (all types: paid, affiliate, gifting). Drafts, rejected, expired and paused campaigns don't count. Checked when creating, submitting, duplicating and switching a campaign back on.
- **Invites**: each influencer invited to a campaign (inviting a whole favourites list counts each). If an admin rejects the campaign, its pending invites are given back (DSC-03).
- **Seats**: other people on the team (invited or joined); the owner is free, so Pro's "3 seats" = owner + 2. See BRD-03.
- **Username reveals**: each new reveal; already revealed usernames stay free. When the monthly allowance is used up, the brand can pay per reveal from the wallet (DSC-01).
- **Advanced analytics**: over the allowance, the report is charged from the wallet at the plan's price instead of being refused (PUB-04).
- **Platform fee**: applied at offer checkout on the offer amount after any coupon (OFR-03).
- **Affiliate commission and connected stores** are enforced by the affiliate platform; Ainfluencer shows its refusal as the same pop-up (CMP-04).

**When monthly allowances reset**
- Paid plans: every month on the anniversary of the subscription start date.
- Free: on the 1st of each calendar month.

**When the plan shrinks** (downgrade, cancellation or failed renewal): extra campaigns are paused and extra team members removed automatically (SUB-02). A paused campaign can only be switched back on when there is a free slot.

**Not enforced anywhere in Ainfluencer** (shown in the comparison only): "chats & collabs / month", affiliate email reveals, AI dubbing minutes, branded landing page, affiliate auto-payouts, premium affiliate tools, dedicated success manager, product-seeding creator outreach and shipment tracking. Some may be handled by the affiliate platform.

**Not available:** add-on packs (extra invites, seats or campaigns without changing plan). That part of the pop-up is switched off; all "Add…" buttons lead to `/subscription-plans`.

## Statuses
| What happens | Internal |
|---|---|
| Limit reached | HTTP **402** `feature_limit_exceeded` with the feature name, `max_quota`, `current_quota` |
| Campaign switch-on refused because the subscription is paused | HTTP 400 (not a limit; see SUB-03) |
| Campaign paused because of the plan | `ads.paused_by = system` |

## Timers and automatic actions
- Usage is counted per action; monthly counts start fresh at the reset date (no job needed).
- `plans:detect-downgrades` (hourly) applies smaller limits after a paid plan ends (SUB-02).

## Notifications
None. The only feedback is the pop-up in the web app.

## Common questions

### "I can't create a new campaign. It says I've reached my limit."
**Answer for the user:** Your plan lets you run {N} campaigns at the same time, and campaigns waiting for review count too. You can pause or end one of your running campaigns, or upgrade your plan for more slots. Drafts don't count.
**What to check:** held campaigns (`ads` with `status = active` and approval `approved`/`pending`, plus gifting in review), plan limit, `getCurrentPlanInfo` usage.
**Escalate when:** the brand holds fewer campaigns than the plan allows and is still refused.

### "I paused a campaign but I still can't create a new one / turn another on."
**Answer for the user:** Paused campaigns don't count, so this should free a slot. Please refresh and try again. Campaigns waiting for admin review do count toward your limit.
**What to check:** campaigns in review; `ads.status` after pausing.
**Escalate when:** usage still counts the paused campaign.

### "I removed a team member but I can't invite a new one."
**Answer for the user:** Let us check your team seats for you.
**What to check:** joined members and open invitations vs seat limit. *(to verify)* The seat counter used at invite time is a running total that is not reduced when a member is removed, so the brand can be blocked although a seat is free.
**Escalate when:** a seat is free but invites are refused → dev to correct the counter.

### "Why is my invite limit used up? I only invited a few people."
**Answer for the user:** Each influencer you invite counts, including every influencer in a favourites list you invite at once. Your invites reset on {reset date}. You can also invite only the remaining number with "Invite Y now".
**What to check:** `feature_usages` for `affiliates_invites` in the current period; recent invitations.
**Escalate when:** usage is higher than the invitations actually sent.

### "Why is the platform fee different from another brand's / from last month?"
**Answer for the user:** The platform fee depends on your plan: 25% on Free, 20% on Starter, 15% on Pro and 3% on Ultimate. It's added on top of the influencer's price at checkout and uses the plan you're on when you pay.
**What to check:** plan at payment time, `orders.fee_amount`.
**Escalate when:** the fee charged doesn't match the plan at the time of payment.

### "Can I buy extra invites or seats without upgrading?"
**Answer for the user:** Not at the moment; the way to get more is to move to a higher plan. For special needs, our sales team can discuss a custom plan.
**What to check:** n/a.
**Escalate when:** the brand wants a custom plan → sales.

### "My plan says 5 chats & collabs a month. Why could I do more?"
**Answer for the user:** That limit isn't applied at the moment, so you can keep chatting and collaborating.
**What to check:** n/a.
**Escalate when:** n/a.

## Known issues
- About ten features shown in the plan comparison are not enforced (list above).
- The add-on purchase section is switched off; its buttons only open the plans page.
- *(to verify)* Removing a team member may not free the seat for new invitations.
- *(to verify)* Free plan gifting access: the plan card says "basic access" but the configuration says no.
- The fee is calculated from the plan of the person paying; a team member paying on the brand's behalf may get a different fee *(to verify)*.

## Related topics
- SUB-01: plans and prices.
- SUB-02: what happens when the plan shrinks.
- CMP-01 / CMP-03 / CMP-05: campaign creation, pausing, gifting.
- DSC-01 / DSC-03: username reveals and invitations.
- BRD-03: team seats.
- OFR-03: platform fee at checkout.

## Technical reference
- **Request-time gate:** middleware `feature:<name>` → `app/Http/Middleware/CheckFeature.php` → `app/Services/Susbscription/FeatureGuardService.php::can()`; refusal `FeatureLimitExceededException` (402 `feature_limit_exceeded`, message "You do not have access to this feature."). Usage stored in `feature_usages`, rolled back on errors.
- **Routes with `feature:`:** `POST /api/v3/campaigns`, `POST /api/v3/campaigns/affiliate`, v2 campaign create and `POST /api/v2/campaigns/{ad}/duplicate` (campaigns); `v2/user/invitations`, `v2/temp-invitations` (affiliates_invites, counts `identities` + `favorite_list_id`); `teams/brand/members/invite|register` (seats, counts `members`); `product-seedings/handle` at `current_step = finish` (product_seeding_access); `api/v1/advance-analytics` (advanced_analytics, falls back to wallet payment).
- **Slot checks:** `app/Modules/PlanEnforcement` — `CampaignSlotGate` (wizard submits, affiliate/gifting), `campaign.activation.quota` middleware (re-activation), `CampaignQuotaPolicy::usage()` (single campaign count used everywhere), `ApprovedCampaignAdmission`.
- **Value settings:** `collab_transaction_fee_percent` → `app/Modules/Payment/Helpers/AmountHelper.php::get_fee_percentage()` (used by `OrderService::calculateOrderSummary`); `influencer_username_reveal(_price)` → `InfluencerRevealService`; `deep_analytics_price`; `min_required_payment_per_collabs` (read by `MakeOfferRequest`, not present in the seed data → fallback); affiliate features enforced by the affiliate service (`AinfluencerAffiliateService` maps its 402).
- **Periods:** `SubscriptionService::getSubscriptionPeriod()` (month from subscription `created_at` anniversary; calendar month without a subscription).
- **Data:** `features` (ids/names in `FeatureNameEnum`), `plan_price_feature` (`value` JSON: `value`, `has_recurring`, `recurring_period`; `-1` = unlimited), `feature_usages` (`feature_id`, `user_id`, `usage_date`, `usage_count`), `ads.paused_by`.
- **Seed data:** `database/seeders/Subscription/plan_price_feature_data_production.php`, `FeatureSeeder.php`.
- **Web:** `components/common/modals/limitModals/{FeatureLimitModal,InviteQuotaModal}`, global 402 handler opening `FeatureLimitModal`.
