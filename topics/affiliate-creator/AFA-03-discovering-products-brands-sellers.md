---
id: AFA-03
title: Discovering products, brands and sellers
domain: Affiliate app (creators)
audience: [affiliate]
apps: [affiliate-app]
related: [AFA-04, AFA-05, AFS-03, AFB-04, CHAT-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-03 · Discovering products, brands and sellers

> Where affiliates browse the products and brands that sellers offer, see commissions, open seller profiles, save items to lists, and start a partnership ("Generate Link", "Apply For Partnership", "Promote").

## Who and where
- **Affiliates** in the affiliate app, sidebar **"Discovery"**:
  - Products `/app/products-discovery` (the landing page after login).
  - Brands `/app/brands-discovery` (called "Stores" when a Shopify platform is selected).
  - Product detail `/app/product/:id`, brand detail `/app/brand/:id`, seller profile `/app/seller/:id`.
- **My Lists**: sidebar "More" → "My Lists" (`/app/discovery-my-lists`, list pages `/app/lists/brands/:id`, `/app/lists/products/:id`).
- **Promote all of a brand's products**: `/app/discovery-promote-all/:brandId`, opened from brand and product cards ("Promote").
- **The platform picker in the header** decides what is shown: an Amazon marketplace (for example `amazon-US`) or Shopify. Switching it reloads the page. On a brand or product page, switching sends the user to the dashboard.

## How it works
1. **Browse products.** The grid shows only products that are active, not removed and have a price above zero. In-stock products come first, then the newest. If several sellers list the same product, it appears once. Filters include category, brand, seller and price; each card shows commission, partnership status ("Partnered" / "Not partnered") and promotions.
2. **Open a product** to see variations, media assets ("No Media Yet" if none), the seller's promotions, partnership status and, once partnered, "Products Links" with "Generate New Link" (AFA-05).
3. **Browse brands/stores.** Cards show "Avg Commission", "Active Contracts", "Number of Products" and so on. A brand page shows the seller's brand assets and terms ("View Terms"), "My Statistics" vs "Total", and "Apply For Partnership".
4. **Seller profile** ("Seller Profile") shows "Commission Paid", "Revenue", "Clicks", "Sales Units", "Conversions", active contracts, brands and products, a storefront link, "Apply For Partnership" and a message button.
5. **Start a partnership** from here: "Generate Link" on a product (one or many selected), "Apply For Partnership" on a brand, or "Promote" for a whole brand. What happens next (instant approval or seller review) is described in AFA-04.
6. **Lists.** "Add To My Lists" saves brands or products into a list. On "My Lists" the affiliate can "Create New List", "Edit List Name", "Duplicate as Manual", "Delete" and pin lists. There are also Ainfluencer system lists ("Ain List (System)"), which can be duplicated but not edited.

## Rules and limits
- **Inactive products and brands are hidden.** When a seller deactivates them, they disappear from Discovery (AFS-03). A deactivated brand shows "Brand Deactivation Notice".
- **"Promote" works only for brands with automatic partnership approval.** It joins at the product's default commission and returns a link straight away. For other brands it fails, and the affiliate must use "Apply For Partnership" instead.
- The **daily applications** counter is shown in the user menu ("Daily Applications:"). The backend doesn't block requests over that number in this version (AFA-09).
- **Messaging sellers**: the "Message Seller" / "Send Message" buttons open the affiliate platform's own chat, which is **not a supported channel**. Users should contact brands through the **Ainfluencer chat** (CHAT-01).

## Statuses
Partnership status on cards and detail pages (worked out from requests and contracts, see AFA-04):

| Shown | Meaning |
|---|---|
| Not partnered | No request or contract |
| Pending / "Applied To … Partnership" | Request sent, waiting for the seller |
| Partnered | Active contract; links can be made |
| Revoked / Rejected / Expired | Past contract or request; see AFA-04 |

## Timers and automatic actions
- Product data (price, stock) comes from the seller's store sync: Amazon about every 48 hours (AFX-03). Shopify updates arrive when the store changes.

## Notifications
None specific to browsing. Partnership notifications are in AFA-04.

## Common questions

### "A product I was promoting disappeared from Discovery."
**Answer for the user:** The seller has deactivated the product or brand, or it was removed from their store. Inactive items are hidden, and sales on them after deactivation don't earn commission.
**What to check:** `seller_products.status` / `inactive_at`, and the brand status (AFS-03).
**Escalate when:** The product is active in the seller's catalogue but still hidden.

### "Why do I see different products from my friend?"
**Answer for the user:** Discovery shows products for the platform and Amazon marketplace selected in the header (for example Amazon US vs Shopify). Switch the platform at the top to see others.
**What to check:** The selected platform.
**Escalate when:** —

### "'Promote' gives an error."
**Answer for the user:** One-click promotion only works for brands that approve partners automatically. For this brand, please use "Apply For Partnership" and the seller will review your request.
**What to check:** The brand's auto-approval setting (AFS-03).
**Escalate when:** The brand has auto-approval on and it still fails.

### "How do I contact the seller?"
**Answer for the user:** Please message the brand through the Ainfluencer chat. The affiliate platform's message button isn't a supported channel.
**What to check:** —
**Escalate when:** —

### "Can I edit an Ainfluencer list?"
**Answer for the user:** System lists can't be edited, but you can use "Duplicate as Manual" to make your own copy and change that.
**What to check:** —
**Escalate when:** —

## Known issues
- The "One Click Promote" page (`/app/discovery-promote`) still works if opened directly, but no menu links to it.
- Message buttons open the unsupported affiliate chat (AFC-01). Point users to the Ainfluencer chat.

## Related topics
- AFA-04 · What happens after "Generate Link" / "Apply For Partnership".
- AFA-05 · Creating and managing links.
- AFS-03 · How sellers activate products and set commissions.
- AFB-04 · Brand recruitment pages outside the app.
- CHAT-01 · Ainfluencer chat.

## Technical reference
- **Endpoints:**
  - `GET affiliate/v1/products` (`IndexProductService`: `available()`, `notRemoved()`, non-zero price, one per master product).
  - `GET affiliate/v1/products/{id}`, `.../variations`, `.../assets[/{mediaId}]`.
  - `GET affiliate/v1/products/brands`, `.../brands/{id}`, `.../brands/{id}/assets`; `GET affiliate/v1/stores` (Shopify).
  - `GET affiliate/v1/profile/seller/{id}`.
  - Drop-downs: `products/brands/drop-down`, `products/sellers/drop-down`, `products/drop-down`.
  - Promote: `GET affiliate/v1/products/landing-page/index`, `POST affiliate/v1/products/landing-page/promote/{id}` (`LandingPagePromoteService`, requires `seller_brands.auto_partnership_approval = true`).
  - Lists: `affiliate/v1/user-type-lists*`, `POST affiliate/v1/pins/toggle`.
- **Data:** `seller_products` (`status`, `inactive_at`, `platform_status`, `commission`, `commission_type`), `seller_brands` (`status`, `auto_partnership_approval`), `user_type_lists` / items.
- **Header:** `X-Platform` = `amazon-<marketplace>` or `shopify` (`validate_multiple_platforms`).
- **Frontend:** `spa-affiliate/src/components/discovery/**`.
