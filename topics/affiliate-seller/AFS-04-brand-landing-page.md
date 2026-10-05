---
id: AFS-04
title: Brand landing page
domain: Seller app (brands)
audience: [seller, affiliate]
apps: [seller-app, buyer-site]
related: [AFB-04, AFS-03, AFS-06, AFS-12, AFA-04]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-04 · Brand landing page

> Every brand (Amazon) or store (Shopify) automatically gets a public recruitment page that the seller can share to attract affiliates. Visitors see the brand, a few featured products and optional stats, and can "Apply for partnership" (the visitor side is AFB-04).

## Who and where

- **Seller**: brand page (`/app/brand/<id>`) → box **"BRANDED LANDING PAGE"** with "Go to Landing", "Copy Link", "Share to..." and "Customize".
- **Visitors / affiliates**: the public page on the buyer site, `/seller/brand/<slug>` (Amazon) or `/seller/store/<shop>` (Shopify). See AFB-04.

## How it works

1. The page is created automatically when the brand or Shopify store is created.
2. "Go to Landing" opens it; "Copy Link" copies it; "Share to..." opens the share options.
3. "Customize" opens "Customize Brand Landing" (or "Store"):
   - "Hide Featured Products" switch, and up to **3 featured products** chosen from the brand's products ("Featured products stand out and can attract more affiliates.");
   - "Hide Brand Analytics" switch (hides "Commission Paid, Revenue, Clicks, Sales Units & Conversions");
   - "Landing URL" slug ("You can change the default URL, but it must be unique.") — Amazon brands only (see Known issues).
4. Saving updates the public page immediately.
5. A visitor who applies is handled as in AFB-04/AFS-06: logged-in affiliates create an application; guests finish it after signing up in the affiliate app.

## Rules and limits

- All four buttons are disabled while the brand is inactive ("Activate your brand to use these").
- "Customize" needs the brand "update" permission (AFS-13).
- Featured products: maximum 3. Slug must be unique among brands.
- The public page shows only an active brand (master brand active) and counts only active products and contracts.
- **Plan:** the landing page is listed as a plan feature, and all current plans include it. The plan check on the public page does not block any seller in practice *(to verify)*.

## Statuses

No statuses of its own. The page depends on the brand being active (AFS-03).

## Timers and automatic actions

None. Stats on the page are computed when it loads.

## Notifications

None for the landing page itself. Applications made from it notify the seller like any application (AFS-06).

## Common questions

### "Where do I find my brand's recruitment link?"
**Answer for the user:** Open My Brands → your brand → "BRANDED LANDING PAGE" → "Copy Link".
**What to check:** the brand is active (buttons are disabled otherwise).
**Escalate when:** the link opens an error for an active brand.

### "The landing page buttons are greyed out."
**Answer for the user:** Your brand is inactive. Activate it on the brand page and the buttons become available.
**What to check:** `seller_brands.status` / Shopify store status.
**Escalate when:** the brand is active and buttons are still disabled.

### "I changed the URL of my Shopify store page but it still uses the old address."
**Answer for the user:** For Shopify stores the page address is always your store's Shopify address; it can't be changed at the moment.
**What to check:** platform = Shopify.
**Escalate when:** n/a (known limitation).

### "I want to hide my sales numbers from the public page."
**Answer for the user:** Use "Customize" → turn on "Hide Brand Analytics" and save.
**What to check:** `landing_page_assignees.config.hide_brand_analytics`.
**Escalate when:** numbers still show after saving.

### "The public page shows an error / 'access denied'."
**Answer for the user:** We'll check the page for you; please send us the link.
**What to check:** the slug exists (Amazon `seller_brands.slug`, Shopify `shop_name`), brand active, seller found.
**Escalate when:** brand active and slug correct.

## Known issues

- **Shopify slug is not saved.** The modal shows "Landing URL" for stores too, but only Amazon brand slugs are stored; Shopify pages always use the shop's `.myshopify.com` name.
- **Wrong prefix in the modal**: the slug field shows "https://app.ainfluencer.com/" in front, but the real address is the buyer-site link copied by "Copy Link".
- **Success message after saving says "Social links updated successfully"** (wrong text; the landing page was saved).
- If a brand has no landing page record, saving fails with "Landing page assignee not found" *(to verify; rare)*. Escalate to dev.

## Related topics

- AFB-04 — The public page and applying from it.
- AFS-03 — Brand activation, products.
- AFS-06 — Handling the applications that arrive.
- AFS-12 — Plan features.

## Technical reference

- **Endpoints:** `PUT seller/v1/products/brands/update/landing-page/{id}` (`UpdateSellerBrandLandingPageController` → `UpdateSellerBrandLandingPageService`; rules: `featured_product_ids` max 3, `hide_featured_products`, `hide_brand_analytics`, `slug` required + unique for Amazon only). Public: `GET client/affiliate/v1/products/brands/{sellerBrandSlug}`, `GET client/affiliate/v1/products/{sellerBrandSlug}`, `GET client/affiliate/v1/products/stores/{storeSlug}`, `GET …/stores/{storeSlug}/products` — middleware `check_brand_landing_page` (`Http/Middleware/CheckBrandLandingPageAccess.php`, feature `brand_landing_page`, checker `BrandLandingPageFeatureChecker` compares against an enum object so it never denies).
- **Data:** `seller_brands.slug`, `shopify_account_stores.shop_name`, `landing_page_assignees.config` (`hide_featured_products`, `featured_product_ids`, `hide_brand_analytics`). Created by job `CreateBrandLandingPage`.
- **Frontend:** `spa-seller/src/components/discovery/brands/brand-details/brand-details.page.vue` (box, link `VUE_APP_BUYER_PANEL_URI/seller/{brand|store}/{slug}`), `brand-detail-header/brand-landing-config.modal.vue`.
