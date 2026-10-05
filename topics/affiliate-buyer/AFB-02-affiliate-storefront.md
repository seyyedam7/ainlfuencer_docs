---
id: AFB-02
title: Affiliate storefront
domain: Buyer site (shoppers)
audience: [buyer, affiliate]
apps: [buyer-site]
related: [AFA-06, AFA-05, AFB-01, AFA-02]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-buyer@v1.3.04, affiliate/ainflu-affiliate-backend@v1.5.27
---

# AFB-02 · Affiliate storefront

> The affiliate's public shop page on the shopper site: their profile and social accounts, a few featured products, and a searchable grid of everything they promote. Each product opens the affiliate's tracked product page (AFB-01).

## Who and where
- **Shoppers** (no login needed) at `/user/<affiliate id>` on the buyer site.
- How people get there:
  - The affiliate shares it ("Copy Link" on "Your branded affiliate page", AFA-06).
  - The affiliate badge on a product page links here (AFB-01).
  - A redirect rule set to "Landing" sends shoppers here when a link or product isn't available (AFA-05).

## How it works
1. The page shows the affiliate's name, photo, bio and **social accounts** (YouTube, Instagram, TikTok, Pinterest, Facebook, website). Only accounts that are available/active are listed.
2. **Featured products** come from "Assign Featured Products" (up to 3, AFA-06).
3. The **product grid** ("Products Of …") has search, "All Categories" filter and sort order (Newest, Oldest, High/Low Price, Discounted), with paging.
4. Clicking a product opens its tracked product page (AFB-01).
5. If there's nothing to show: "There Is No Item To Show".

## Rules and limits
- The grid lists only the affiliate's **default** links that are **active**, for products the seller still has **active**. A product drops off when the partnership ends, the link is revoked, or the seller deactivates the product.
- Category filters use the platform (Amazon marketplace / Shopify) of the request.
- No login, cart or checkout here. Purchases happen on Amazon or Shopify.

## Statuses
None.

## Timers and automatic actions
- Changes appear immediately when links or contracts change. Product data follows the seller's store sync.

## Notifications
None.

## Common questions

### "My storefront is empty / a product is missing." (affiliate)
**Answer for the user:** The storefront shows your active default links for products that are still active at the seller. Check that you're still partnered, and that the link is marked "Set this link as default link".
**What to check:** `affiliate_product_links.is_default`, `status`; product status; contract status (AFA-04).
**Escalate when:** An active default link for an active product isn't shown.

### "My social account isn't shown on my page." (affiliate)
**Answer for the user:** Only connected, active accounts appear. Check "Manage Platforms" in your profile; manual accounts may still be under review.
**What to check:** The account's status (AFA-02).
**Escalate when:** A verified, active account isn't shown.

### "I was sent to this page instead of the product I clicked." (shopper)
**Answer for the user:** That product isn't available through this creator's link any more, so you've been shown their other products.
**What to check:** AFA-05 redirect rules.
**Escalate when:** —

## Known issues
- The avatar menu on this site has "Dashboard" and "Saved products" entries that lead to missing pages (AFB-03).

## Related topics
- AFA-06 · Branded affiliate page and featured products.
- AFA-05 · Default links and redirect rules.
- AFB-01 · Product page.
- AFA-02 · Social accounts shown here.

## Technical reference
- **Endpoints:**
  - `GET client/buyer/v1/profile/affiliate/{affiliateId}` (`ShowAffiliateProfileLinkService`: profile, landing collections, available social channels).
  - `GET client/buyer/v1/profile/links/affiliate/{affiliateId}/products` (`AffiliateProfileProductsService`: `is_default = true`, link `status = active`, product `status = active`; filters `query`, `category_ids[]`, order, `has_promotion`).
  - `GET client/buyer/v1/categories` (header `X-Platform`).
- **Buyer code:** `aff-buyer/app/user/[id]/page.tsx`, `domains/user/AffiliatePage.tsx`, `domains/user/components/AffiliateProducts/*`.
- **Data:** `affiliate_product_links`, `landing_pages` / collections, `user_social_account_channels`.
