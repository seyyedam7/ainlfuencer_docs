---
id: AFA-05
title: Affiliate links, coupons and redirect settings
domain: Affiliate app (creators)
audience: [affiliate, buyer]
apps: [affiliate-app, buyer-site]
related: [AFA-04, AFA-06, AFB-01, AFB-02, AFS-07, AFA-08, AFA-10]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02, affiliate/ainflu-affiliate-buyer@v1.3.04
---

# AFA-05 · Affiliate links, coupons and redirect settings

> Once partnered, an affiliate shares tracked links. Each link opens a product page on the Ainfluencer shopper site, which sends the shopper on to Amazon or Shopify. The affiliate can attach a seller's coupon, a source (for reporting) and a landing template, and decide where shoppers go if the link or product stops working.

## Who and where
- **Affiliates** in the affiliate app:
  - **"My Links"** (sidebar) → `/app/active-management`: partnered products/brands, "Product Links", "Generate Link", "Copy AMZ Link", "Message Seller", and the **"Product Redirect Setting"** button.
  - "See Inactive Products" → `/app/inactive-products`; inactive brands at `/app/inactive-brands`.
  - Product detail `/app/product/:id` → "Products Links" tab → "Generate New Link" (only when partnered).
- **Shoppers** see the result on the buyer site (AFB-01).

## How it works
1. **Default links are automatic.** When a partnership is approved, the system creates a default link for each product (AFA-04). In lists it is shown as "Default" / "System Default Link".
2. **Make another link.** "Generate New Link" opens "Generate New Product Link":
   - "Title" (optional name),
   - **"Choose Source"**: pick an existing source, or create one with "Source Name", "Primary ID" and an optional "Sub ID". Sources group clicks and sales in the Source report (AFA-10).
   - **"Promotion"**: optional coupon from the seller,
   - **"Landing Page"** (required): which landing template the product page uses (AFA-06),
   - **"Set this link as default link"**.
   
   Then "Submit". Links can later be edited ("Edit Product Link") or deleted.
3. **Two kinds of link.**
   - The **Ainfluencer link** ("AinLink") opens the product page on the shopper site at `/product/<link code>`. This is the link the redirect settings and coupons work with.
   - **"Copy AMZ Link"** copies the direct Amazon link with tracking ("AmzLink"). Redirect settings do **not** apply to it.
4. **Coupons.** Sellers create promotions (AFS-07). The affiliate only attaches one to a link. Shoppers see the discount on the product page, and the code itself only when logged in to a shopper account (AFB-01, AFB-03). A **private** promotion can be attached only if the seller gave this affiliate access, and only while the affiliate has a contract for that product.
5. **Product Redirect Setting.** Four rules, each with radio options. "None" means the product page shows as normal.

   | Rule (exact label) | Amazon options | Shopify options | Default |
   |---|---|---|---|
   | "Redirect if product link is removed" (link deleted, revoked or inactive) | Amazon, Landing, Amazon store* | Shopify, Landing | Landing |
   | "Redirect if product is out of stock" | None, Amazon, Landing | None, Shopify, Landing | Landing |
   | "Redirect if product is removed or disabled" | Amazon, Landing, Amazon store* | Shopify, Landing | Landing |
   | "Redirect if promotion is no longer available to" | None, Amazon, Landing | None, Shopify, Landing | None |

   - "Landing" = the affiliate's storefront page (AFB-02).
   - "Amazon" = the product on Amazon with the affiliate's associate tag.
   - "Amazon store" = the affiliate's own Amazon store link. *It can only be chosen when an Amazon Store link is saved in the profile ("To enable this feature, please set your Amazon Store link in your Affiliate Profile.").
   - The rules are checked in this order: link removed → product removed/disabled → out of stock → promotion gone. The first rule that applies wins.

## Rules and limits
- Links appear only for products with an **active contract**. When a contract is revoked, all its links become revoked (AFA-04).
- **Inactive or removed products** move to the inactive lists. Sales after a product was deactivated earn nothing (AFS-03).
- Settings are kept separately for Amazon and Shopify. The app shows the set for the platform selected in the header.
- Deleting a landing template moves its links to the default template (AFA-06).

## Statuses
| Link status | Meaning | Shopper experience |
|---|---|---|
| `active` | Working link | Product page |
| `revoked` | Contract ended or plan downgrade | "Link removed" rule applies |
| `inactive` | Disabled | "Link removed" rule applies |
| deleted | Removed by the affiliate | "Link removed" rule applies |

## Timers and automatic actions
- Revoked at once when a contract ends, including the nightly campaign-end revoke at 02:00 UTC (AFX-02).
- Stock status comes from the store sync (Amazon roughly every 48 h).

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| A downgrade removes a promotion from a link | Affiliate | Email (batched) |
| Seller deactivates a product/brand | Affiliate | In-app (some also email) |

## Common questions

### "My link sends people to my storefront instead of the product."
**Answer for the user:** That's your Redirect Setting at work. The link has been removed or revoked, or the product is out of stock or no longer available, and your setting sends shoppers to your storefront ("Landing"). Check your partnership and the product's status, or change the setting under My Links → Product Redirect Setting.
**What to check:** Link status; contract status; product `status` / `platform_status` (out of stock); the affiliate's settings.
**Escalate when:** Link, contract and product are all active and in stock, but the redirect still happens.

### "I set 'Amazon' and now shoppers see 'Error Fetching Product' / an error page."
**Answer for the user:** Sorry, the "Amazon" redirect option doesn't work on the shopper site right now. Please choose "Landing" (your storefront) for now. For Shopify products, the "Shopify" option has the same problem.
**What to check:** Which setting is "Amazon" or "Shopify" and which condition is active.
**Escalate when:** Report to dev (known issue, AFB-01).

### "Why can't I pick 'Amazon store'?"
**Answer for the user:** Add your Amazon Store link in your profile first. The option unlocks after that.
**What to check:** The profile's Amazon store link field.
**Escalate when:** The link is saved and the option is still disabled.

### "I can't add a coupon to my link."
**Answer for the user:** Coupons come from the seller. You'll only see promotions that the seller made public or shared with you, and only for products you're partnered on. Ask the seller through the Ainfluencer chat if you expected one.
**What to check:** The promotion is active and approved (AFS-07); public or assigned to this affiliate; active contract for the product.
**Escalate when:** The promotion is assigned to the affiliate and active but still not offered.

### "Which link should I share, the Ainfluencer link or the AMZ link?"
**Answer for the user:** The Ainfluencer link gives you the product page with coupons, your branding and redirect protection if the product disappears. The AMZ link goes straight to Amazon and skips those features. Both are tracked for commission.
**What to check:** —
**Escalate when:** —

## Known issues
- **"Amazon" redirect option** (any rule) makes the shopper site show an error instead of redirecting. **"Shopify" redirect option** shows "Error Fetching Product". Workaround: use "Landing". Escalate to dev.
- Tooltips still use older names ("AlgoLink", "AinLinks", "AmzLinks").

## Related topics
- AFA-04 · Partnerships that create and revoke links.
- AFA-06 · Landing templates used by links.
- AFB-01 · The product page shoppers see.
- AFB-02 · The storefront ("Landing" redirect target).
- AFS-07 · Seller promotions (coupon codes).
- AFA-10 · Source and landing-page reports.

## Technical reference
- **Endpoints:**
  - `POST affiliate/v1/products/links` (`Services/Actions/Affiliate/Product/Link/CreateProductLinkService`; validation: `landing_page_id` required, `source_id` or `source{name,primary_id,sub_id}`).
  - `PUT/DELETE/GET affiliate/v1/products/links/{id}`, `GET affiliate/v1/products/links/drop-down`.
  - Sources: `GET/POST affiliate/v1/products/links/sources[/drop-down|/{id}]`.
  - Promotions: `GET affiliate/v1/products/promotions/drop-down`, `GET affiliate/v1/products/{id}/promotions/drop-down`.
  - Management lists: `GET affiliate/v1/managements/products`, `.../brands`.
  - Settings: `GET/POST affiliate/v1/settings` (`ShowUserTypeSettingService`, `UpdateUserTypeSettingService`).
  - Public resolution: `GET client/buyer/v1/products/links/{id}` → `ShowProductLinkController` → `ShowProductLinkService` → `CheckProductLinkSetting` (conditions `ProductLinkRemovedCondition`, `ProductDisabledCondition`, `StockAvailabilityCondition`, `PromotionAvailabilityCondition`; strategies in `.../Product/Link/Strategies`). It returns HTTP 301 with `data.redirect_to` (`affiliate_landing_page`, `product_landing_page`, `redirect_to_amazon`, `redirect_to_amazon_store`, `redirect_to_shopify`).
- **Data:** `affiliate_product_links` (`user_type_id`, `seller_product_id`, `status`, `is_default`, `is_system`, `link`, `platform_link`, `seller_promotion_id`, `source_id`, `deleted_at`); `affiliate_product_link_sources`; `user_type_settings` (groups `product` / `shopify_product`, keys `redirect_if_product_link_removed`, `redirect_if_product_out_of_stock_to`, `redirect_if_product_removed_or_disabled`, `redirect_if_promotion_no_longer_available`; values `none|landing|amazon|amazon_store|shopify`). Defaults: `database/seeders/AffiliateUserTypeSettingTableSeeder.php`.
- **Frontend:** `spa-affiliate/src/components/management/**` (`Modals/management-redirect-setting.vue`), `discovery/products/product-details/sections/link-list/create-link.modal.vue`.
