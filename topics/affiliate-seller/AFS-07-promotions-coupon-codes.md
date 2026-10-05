---
id: AFS-07
title: Promotions (coupon codes)
domain: Seller app (brands)
audience: [seller, affiliate, admin]
apps: [seller-app, buyer-site, background]
related: [AFS-03, AFS-06, AFS-12, AFA-05, AFB-01, AFX-03]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02, core/backend/api@99921fbce
---

# AFS-07 · Promotions (coupon codes)

> A **promotion** is a discount code the seller offers through affiliates. Affiliates attach it to their links (AFA-05) and logged-in shoppers see it on the product page (AFB-01). For **Amazon**, Ainfluencer only stores and shares the code: the seller must create the same code in Amazon Seller Central. For **Shopify**, discount codes created in the Shopify admin appear here automatically.

## Who and where

- **Seller** (team permission "Promotions": view, create, update, delete).
- Sidebar "More → My Promotions" (`/app/promotion-list`), "Add New Promotion" (`/app/new-promotion`), details `/app/promotion-details/<id>` with "Edit Duration", "Edit Promotion", "Edit Assigned", "Delete".
- **Admins** review Shopify promotions created through the API/admin.

## How it works

**Creating (Amazon store selected)**
1. "Add New Promotion" → step "Select Product for Promotion". It needs a connected Amazon seller account and at least one **active** brand ("Please activate at least one brand first…"). Choose the brand and products; each shows its default commission.
2. Step "Promotion Details":
   - "Promotion Code": letters and numbers only, no spaces, **8–12 characters**, typed twice ("Codes are not same" if they differ);
   - Availability: **Public** (all partnered affiliates on those products) or **Private** (only affiliates the seller assigns later with "Edit Assigned");
   - Name, "Buyer Visibility (only on public)", optional note;
   - From and To dates (From before To);
   - Discount type: percentage (max 100), fixed amount, or free shipping, and the value.
3. Confirm "Submit Promotion for Review?" ("…you can only extend the duration later; no other edits will be possible.").
4. The success screen says the promotion "will be reviewed by our team". In reality **Amazon promotions are active and accepted immediately**.
5. **The seller must create the same code with the same discount in Amazon Seller Central.** Ainfluencer cannot create, change or delete Amazon promotions.

**Shopify**
- Basic discount codes created and active in the Shopify admin are imported automatically as **public, active** promotions and kept in sync (changes and deletions in Shopify are mirrored).
- Creating a promotion from the seller app with a Shopify store selected doesn't work (the form doesn't send the Shopify fields).

**After creation**
- "Edit Duration": change From/To. "Edit Promotion": dates and buyer visibility. "Edit Assigned" (private): choose partnered affiliates and which of their contracted products the code applies to.
- "Delete" removes it from Ainfluencer (not from Amazon).
- Affiliates see available codes and can attach one to a link; shoppers who are logged in on the product page see the code (AFB-01).

## Rules and limits

- **Plan:** creating needs a plan that includes promotions; otherwise "Promotion Creation Limited – Upgrade to the Ultimate Plan to create promotions for Affiliates." Private promotions need the private-promotion feature. On plans with a numeric promotion limit, a product that already has an active promotion can't get another (AFS-12; current paid plan data may not include promotions *(to verify live)*).
- Code: 8–12 letters/numbers (Amazon).
- Dates: end on or after start.
- Private promotion assignment: only to affiliates with a contract covering those products.
- When a contract is revoked, that affiliate loses access to private promotions.
- Promotions are per store: they belong to the Amazon profile or Shopify store selected.

## Statuses

| Shown | Internal (`seller_promotions.status` / `approval_status`) | Meaning |
|---|---|---|
| Active | `active` + `accepted` | Usable by affiliates |
| Waiting for approval | `approval_status` = `waiting_for_approval` | Only for Shopify promotions created outside the seller app; admin decides |
| Inactive | `inactive` | Switched off |
| Expired | `expired` | End date passed (set hourly) |
| Rejected / Deleted | `rejected` / `deleted` | Admin rejection or Shopify deletion |

Affiliate access to a promotion: `seller_promotion_affiliates.status` active/inactive.

## Timers and automatic actions

| Job | When (UTC) | Effect |
|---|---|---|
| `promotion:update-status` | hourly | Every promotion whose end date is in the past → `expired` (any status). Dates are saved without a time, so a promotion normally expires at about 00:00 UTC on its end date *(to verify)*. |
| Shopify discount webhooks | immediately | Create/update/delete in Shopify → promotion created/updated/deactivated |

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Admin approves/rejects a promotion | Seller | Web + email |
| Promotion becomes available | Affiliates | Mentioned on the success screen *(to verify; no sending code found for seller-app Amazon promotions)* |
| Plan downgrade unlinks a promotion from a link | Affiliate | Email |

## Common questions

### "My Amazon promotion is active here, but the code doesn't work on Amazon."
**Answer for the user:** Ainfluencer can't create codes in Amazon. Please create the same code, with the same discount and dates, in Amazon Seller Central → Promotions. Once it exists there, shoppers can use it.
**What to check:** the code in `seller_promotions.platform_code`; whether the seller created it on Amazon.
**Escalate when:** the code exists on Amazon and still fails (Amazon-side issue, not ours).

### "The screen said my promotion will be reviewed. How long does it take?"
**Answer for the user:** Amazon promotions don't need our review; it's already active. Shopify codes created in your Shopify admin are also active right away.
**What to check:** `approval_status`.
**Escalate when:** a promotion shows "Waiting for approval" for more than a few working days.

### "I extended my promotion but it still shows Expired."
**Answer for the user:** We're aware of this. Changing the dates of an expired promotion doesn't reactivate it. Please create a new promotion for now.
**What to check:** `status` = `expired`, new `end_date` in the future.
**Escalate when:** the seller needs that promotion restored (dev can change the status).

### "I get 'Promotion Creation Limited'."
**Answer for the user:** Promotions are included only in some plans. Check Manage Subscription for the plans that include them.
**What to check:** the seller's plan features `promotion` / `private_promotion` (AFS-12).
**Escalate when:** the plan includes promotions and the pop-up still appears.

### "I use Shopify. How do I create a promotion?"
**Answer for the user:** Create a discount code in your Shopify admin. It appears in My Promotions automatically as a public promotion for your partnered affiliates.
**What to check:** `shopify_discounts` row for the code; store connection active (AFX-03).
**Escalate when:** an active basic code in Shopify doesn't appear within an hour.

### "An affiliate can't see my private promotion."
**Answer for the user:** Private promotions are visible only to affiliates you assign in "Edit Assigned", and only for products in their partnership.
**What to check:** `seller_promotion_affiliates` rows and items; the affiliate's contract is active.
**Escalate when:** assigned with an active contract and still not visible.

## Known issues

- **"Will be reviewed by our team" is misleading for Amazon**: they're accepted instantly.
- **Amazon promotions aren't created on Amazon**; sellers often miss this step.
- **Shopify sellers can't create promotions in the seller app** (validation error); use Shopify admin.
- **Extending an expired promotion doesn't reactivate it.**
- **Deleting a promotion doesn't remove it from Amazon** (seller must remove it there).
- **Coupons from Ainfluencer campaigns are not available**: the affiliate platform has an API for them, but no live Ainfluencer screen uses it (the brand app's `/affiliate/coupons` page makes no calls).

## Related topics

- AFA-05 — Affiliates attaching codes to links.
- AFB-01 — Shoppers see the code only when logged in.
- AFS-06 — Contracts required for private promotions.
- AFS-12 — Plan features for promotions.
- AFX-03 — Shopify store connection and webhooks.

## Technical reference

- **Endpoints:** `GET/POST seller/v1/promotions` (POST gates `promotion`, `private_promotion`; rules by platform in `Http/Requests/Seller/V1/Promotion/CreatePromotionRequest.php`: Amazon `amazon_account_profile_id`, `platform_code` 8–12 alphanumeric; Shopify needs `shopify_account_store_id`, `shopify_discount_type`), `GET/PATCH/DELETE seller/v1/promotions/{id}`, `PATCH …/{id}/update-status`, `PATCH …/{id}/accessibility`, `GET/PATCH …/{id}/affiliates` (gate `private_promotion`). Integration (not used by live core screens): `seller/v1/integrations/ainfluencer/coupons/{create,list,show}`.
- **Services:** `Services/General/Promotion/CreatePromotion.php`, strategies `AmazonPromotionStrategy` (`approval_status` accepted, no Amazon API calls) and `ShopifyPromotionStrategy` (`waiting_for_approval`, pushes to Shopify), `UpdatePromotion.php` (status only changes if sent), Shopify webhook handlers `Services/General/Shopify/Webhook/Handlers/Discount{Create,Update,Delete}Handler.php` (imported codes: active, accepted, public). Expiry `UpdatePromotionStatusCommand` (hourly). Admin approval `admin/v1/requests/*`.
- **Data:** `seller_promotions` (`status`, `approval_status`, `platform_code`, `is_public`, `buyer_visibility`, `start_date`, `end_date`, `accountable_type/id`), `seller_promotionables`, `seller_promotion_affiliates` (+ items), `shopify_discounts` (`status` 0 active/1 expired/2 deleted, `created_via`).
- **Frontend:** `spa-seller/src/components/promotions/*` (`promotion-form.page.vue`, edit pages), `spa-seller/src/store/promotions.store.js`.
