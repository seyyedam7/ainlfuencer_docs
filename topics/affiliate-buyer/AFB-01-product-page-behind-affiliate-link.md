---
id: AFB-01
title: Product page behind an affiliate link
domain: Buyer site (shoppers)
audience: [buyer, affiliate]
apps: [buyer-site]
related: [AFA-05, AFA-06, AFB-02, AFB-03, AFS-07, AFA-08]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-buyer@v1.3.04, affiliate/ainflu-affiliate-backend@v1.5.27
---

# AFB-01 · Product page behind an affiliate link

> When a shopper clicks an affiliate's Ainfluencer link, they land on a product page on the Ainfluencer shopper site. It shows the product, the brand, who recommended it and any coupon. The "Buy Now" button sends them to Amazon or Shopify, where the purchase actually happens.

## Who and where
- **Shoppers** (no account needed) on the buyer site, address `/product/<link code>`. The domain is configured on the server; the default is buy.ainfluencer.com *(to verify for production)*.
- The site's home page (`/`) just redirects to ainfluencer.com.
- **Affiliates** shape this page through their link settings, landing template and redirect rules (AFA-05, AFA-06).

## How it works
1. **The shopper opens the link.** The site asks the affiliate platform for the link. The platform first checks the affiliate's **redirect rules** (AFA-05):
   - The link was deleted, revoked or deactivated → "Redirect if product link is removed".
   - The product was removed or disabled → "Redirect if product is removed or disabled".
   - The product is out of stock → "Redirect if product is out of stock".
   - The attached promotion is no longer available → "Redirect if promotion is no longer available to".

   If a rule applies, the shopper is sent to the affiliate's **storefront** (AFB-02) or to the affiliate's **Amazon store**. If the rule is set to Amazon or Shopify, see Known issues.
2. **Otherwise the page shows** (layout and colours from the affiliate's landing template):
   - Product images, price, availability ("In Stock" / "Out Of Stock", "Available on Amazon marketplace" / "Available on Shopify store").
   - The brand badge, and an affiliate badge that links to the affiliate's storefront.
   - "About This Item", "Description", "Details", "Product Variations" (other sizes/colours, each opening its own link).
   - Product sliders from the template, and an affiliate disclosure ("Affiliate Disclosure - Federal Trade Commission (FTC)").
3. **Coupon.** If the link has a promotion, a box shows "Get Extra" with the percent or amount and "This promotion is provided by <name>.". Guests see a hidden code with **"Login to See"** and "Please log in or register to claim it."; logged-in shoppers see the code and can copy it ("Coupon copied successfully"). With no promotion, guests see "To Get Related Promotions … Login/Register".
4. **Buy.** "Buy Now On Amazon" (or Shopify) when in stock, otherwise "View Product On Amazon". It opens the store in a new tab with the affiliate's tracking, so the affiliate is credited for the sale.
5. **Share.** The share button offers Facebook, X, WhatsApp, Telegram and copy link.

## Rules and limits
- **Nothing is bought on Ainfluencer.** Orders, payment, delivery and returns are handled by Amazon or the Shopify store. Ainfluencer can't look up or change orders.
- The coupon **code is only shown to shoppers logged in** to a shopper account (AFB-03). The code is used at checkout on the store.
- Commission for the affiliate is worked out later from Amazon reports. Shopify commission isn't calculated today (AFA-08).
- Every page visit is recorded as a page view for the affiliate's statistics.

## Statuses
None for shoppers. The link's state (active / revoked / inactive / deleted) decides whether the page shows or redirects (AFA-05).

## Timers and automatic actions
- Stock and price come from the seller's store sync (Amazon about every 48 hours), so the page can lag behind Amazon.

## Notifications
None.

## Common questions

### "I clicked a link and got 'Error Fetching Product' / an error page."
**Answer for the user (affiliate):** The product or link has a redirect rule set to "Amazon" or "Shopify", and that option is currently broken on the shopper site. Please change your Redirect Settings to "Landing" so shoppers go to your storefront instead.
**What to check:** The link, contract and product status, and the affiliate's redirect settings (AFA-05). It can also be a link code that doesn't exist.
**Escalate when:** The link is active, the product is active and in stock, and the page still errors.

### "The link takes me to the affiliate's page instead of the product."
**Answer for the user:** The product is no longer available through this link (partnership ended, product removed or out of stock), so you were sent to the creator's storefront with their other products.
**What to check:** As above.
**Escalate when:** —

### "Where is the coupon code?"
**Answer for the user:** Sign in or create a free shopper account ("Login to See"); the code then appears. Use it at checkout on Amazon or the store.
**What to check:** Whether the promotion is still active (AFS-07).
**Escalate when:** A logged-in shopper still can't see the code.

### "The coupon didn't work on Amazon."
**Answer for the user:** Coupons are created by the brand and applied on Amazon or the store. They may have expired or have conditions. Please contact the store; we can't change orders.
**What to check:** The promotion's end date and status.
**Escalate when:** The promotion is active and approved but the brand says the code is wrong (forward to the seller team).

### "I have a problem with my order."
**Answer for the user:** Your order was placed with Amazon or the brand's Shopify store, so please contact them directly. Ainfluencer only showed you the product.
**What to check:** —
**Escalate when:** —

### "The price or stock on the page is different from Amazon."
**Answer for the user:** Prices and stock are updated periodically, so Amazon's page is always the final word.
**What to check:** —
**Escalate when:** —

## Known issues
- **Redirect to "Amazon"** (any rule) crashes the page instead of redirecting. **Redirect to "Shopify"** shows "Error Fetching Product". Workaround: the affiliate sets the rule to "Landing". Escalate to dev.
- The page loads the link twice on the server, so page views may be counted twice *(to verify)*.
- Shared-link previews (canonical/OG addresses) may show "undefined" in the URL *(to verify)*.

## Related topics
- AFA-05 · Links, coupons and redirect rules.
- AFA-06 · Landing templates (layout and sliders).
- AFB-02 · Affiliate storefront.
- AFB-03 · Shopper account (needed to see codes).
- AFS-07 · Seller promotions.
- AFA-08 · How the affiliate is paid.

## Technical reference
- **Endpoints:**
  - `GET client/buyer/v1/products/links/{productLinkId}` → `Client\Buyer\V1\Product\Link\ShowProductLinkController` → `ShowProductLinkService`. Redirect = HTTP 301 with `data.redirect_to` and `entity_id` / `link`. Logs a `PageView` per call.
  - `GET client/buyer/v1/products/links/{id}/variations`.
- **Coupon visibility:** `Http/Resources/Client/V1/Product/Link/AffiliateProductLinkPromotionResource` (`code` only when a Sanctum user is authenticated).
- **Buyer code:** `aff-buyer/app/product/[id]/page.tsx` (redirect handling: `affiliate_landing_page`, `product_landing_page`, `redirect_to_amazon_store` work; `redirect_to_amazon` uses `window` on the server; `redirect_to_shopify` isn't handled), `app/product/components/productData/{BuyButton,Coupon,ProductPromotion,WelcomeBanner}.tsx`.
- **Data:** `affiliate_product_links` (`status`, `deleted_at`, `seller_promotion_id`, `platform_link`), `seller_products` (`status`, `platform_status`), `seller_promotions`, page views.
- **Config:** `ainfluencer.client.product_link` (env `CLIENT_PRODUCT_LINK`, default `https://buy.ainfluencer.com`).
