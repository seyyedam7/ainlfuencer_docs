---
id: AFS-03
title: Products, brands and commissions
domain: Seller app (brands)
audience: [seller, affiliate, admin]
apps: [seller-app, email, background]
related: [AFX-02, AFX-03, AFS-02, AFS-04, AFS-06, AFS-09, AFS-10, AFS-12, AFA-03, AFA-05]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-03 · Products, brands and commissions

> The seller's catalogue is imported from the connected store. The seller decides which brands (Amazon) or stores (Shopify) and which products affiliates can promote, and the commission paid per sale. Commission is set **per product**; changing it changes it on existing partnerships too.

## Who and where

- **Seller** (team members need the Products / Brands permissions, AFS-13), for the store selected in the header (AFS-02).
- "My Products" (`/app/discovery-products`) and the product page (`/app/product/<id>`).
- "My Brands" / "My Stores" (`/app/discovery-brands`) and the brand page (`/app/brand/<id>`).
- **Admins** approve Amazon brand links and can change products in the admin panel.
- Ainfluencer brands can change commission and status from the Ainfluencer web app too (AFX-02, CMP-04); it is the same data.

## How it works

**Catalogue**
1. Products and brands are imported when the store is connected and refreshed automatically (Amazon about every 48 h; Shopify whenever a product changes in Shopify) (AFX-03).
2. New products get the **default commission of 15%** (percentage). Amazon products start **active** only if they're in stock and eligible; Shopify products follow the Shopify product status (only "Active" Shopify products are active).
3. New products of a brand are added automatically to the seller's active **brand** partnerships.

**Activating / deactivating**
1. Product page → status switch (deactivating asks for confirmation). An inactive product shows "This product is hidden for affiliates. Activate the product to unhide it."
2. Brand page → activate/deactivate (with confirmation). Deactivating a brand (or a Shopify store) **also deactivates all its active products**. Re-activating the brand does **not** re-activate the products; turn them on again one by one.
3. Inactive products/brands are hidden from affiliates. Existing links of an inactive product show the shopper the affiliate's "product removed" behaviour (AFA-05), and **Amazon sales dated after deactivation earn no commission**.
4. Partnered affiliates are notified of status changes.

**Changing commission**
1. Single product: product page → default commission edit (value from 0 up to, but not including, 100%).
2. Several products: My Products (or brand page) → select → "Edit Products" → "Edit Commission" (more than 0; up to 100 products per save). The modal warns: "Lowering the commission rate will affect all existing brand partnerships and may lead to dissatisfaction among your affiliates."
3. The new rate is saved on the product **and copied to every partnership item for that product that doesn't have a custom commission**, including revoked ones. Sales that haven't been billed yet are re-priced at the new rate during the next Amazon report sync (sales are re-read for the last 60 days, AFS-10).
4. Partnered affiliates get a "commission updated" notification.

**Brand page extras**
- "Automatic Partnership Approval" switch: when on, affiliate applications without a custom commission are approved instantly (AFS-06). It is switched on/off for all brands when the subscription becomes active/inactive (AFS-12).
- Edit brand: image and bio. "Social Links": Instagram, Facebook, Pinterest, TikTok, X, website.
- "Terms of Sales": upload PDF, DOC or DOCX (max 4 MB); download or delete. Partnered affiliates are emailed when it changes.
- Amazon brand link (Amazon only): "Set your Amazon brand link" → "Send for approval" → "Pending approval" until an admin accepts → "See on Amazon".
- Media assets (brand and product): images, MP4 videos, documents for affiliates to use.
- "BRANDED LANDING PAGE" (AFS-04).

## Rules and limits

- **Commission type:** sellers set percentages. Brand invitations use each product's own commission; custom rates on invitations are in AFS-06.
- **Plan rules** (AFS-12), answered with an "upgrade" pop-up (HTTP 402):
  - minimum commission on some plans (currently 25% on Free per plan data *(to verify live)*) → pop-up "Upgrade to Ultimate";
  - number of active brands/stores → "Brand Limit Reached";
  - number of active products (ASIN limit).
  - Deactivating is never blocked by limits.
- Activating a product whose brand/store is inactive counts against the brand limit.
- Commission edits require the "Products → update" permission.

## Statuses

| Shown | Internal | Meaning |
|---|---|---|
| Active / Inactive product | `seller_products.status` active/inactive, `inactive_at` | Hidden from affiliates when inactive; Amazon sales after `inactive_at` earn nothing |
| In stock / Out of stock / Removed | `seller_products.platform_status` | From the store. "Removed" = no longer returned by Amazon |
| Active / Inactive brand | `seller_brands.status` (Amazon) / `shopify_account_stores.status` (Shopify) | Inactive hides the brand and switches its products off |
| Pending approval / accepted / rejected | Amazon brand link approval (`waiting_for_approval → accepted/rejected`) | Admin decision; seller notified |

## Timers and automatic actions

- Amazon product refresh every 48 h; Shopify on each product webhook (AFX-03).
- Amazon sales sync daily 06:00 UTC (last 60 days) re-prices unbilled sales at the current contract rate.
- Shopify store inactive (hourly check / uninstall) → all its products deactivated.
- A plan downgrade can deactivate brands/products and raise commissions to the plan minimum (AFS-12).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Product commission changed | Partnered affiliates | Web notification (+ email in the 06:00/18:00 digest) |
| Product or brand status changed | Partnered affiliates | Web notification |
| Product/brand deactivated | Affiliates | Email (digest) |
| Brand terms file changed | Partnered affiliates | Email |
| Amazon brand link approved/rejected | Seller | Web + email |

## Common questions

### "I changed the commission. Does it affect my current affiliates?"
**Answer for the user:** Yes. The product's commission applies to all your partnerships on that product (except partnerships where you agreed a custom commission). Sales not yet billed are also calculated at the new rate.
**What to check:** `partnership_contract_item_products.commission` and `is_custom_commission` for that product.
**Escalate when:** the seller disputes an already-billed charge (finance, AFS-10).

### "I get 'Upgrade to Ultimate' when I set a lower commission."
**Answer for the user:** Your plan has a minimum commission. Choose a rate at or above it, or upgrade (Manage Subscription).
**What to check:** the seller's plan and its commission minimum (AFS-12).
**Escalate when:** the plan has no minimum and the pop-up still appears.

### "I re-activated my brand but its products are still hidden."
**Answer for the user:** Turning a brand off switches its products off, but turning it back on doesn't switch them on. Please activate the products you want on My Products.
**What to check:** product `status`/`inactive_at` vs. brand change time.
**Escalate when:** products can't be activated (no limit pop-up, no error).

### "An affiliate says my product disappeared."
**Answer for the user:** The product is inactive or no longer available in your store, so affiliates can't see or promote it. Activate it on the product page if it's still for sale.
**What to check:** `status`, `platform_status` (removed/out of stock), brand/store status, Shopify store status (AFX-03).
**Escalate when:** the product is active and in stock but still hidden.

### "My commission went back to 15%." / "A product I turned off is active again."
**Answer for the user:** We're checking this. Product data is refreshed from your store regularly and in some cases the refresh may reset these settings. Please send us the product and the time you noticed.
**What to check:** product `updated_at` vs. the last Amazon profile `synced_at` or a Shopify product update.
**Escalate when:** always (see Known issues).

### "How long does Amazon brand link approval take?"
**Answer for the user:** It's reviewed by our team; you'll get a notification when it's accepted or rejected.
**What to check:** the request in admin "requests".
**Escalate when:** pending for more than a few working days.

## Known issues

- **Re-imports may overwrite seller settings** *(to verify with data)*. The import code writes the default 15% commission and the store's status onto existing products: Amazon at each 48-hour product refresh, Shopify on every product update from Shopify. Existing partnerships keep their own rate, but the product (and new partnerships) show 15%, and deactivated products may come back. Escalate to dev with examples.
- **Commission changes also rewrite revoked contracts' items** (harmless for billing, but visible in contract details).
- **Re-activating a brand doesn't re-activate its products** (by design in code; users don't expect it).

## Related topics

- AFS-02 / AFX-03 — Where products come from; store status.
- AFS-04 — Branded landing page on the brand page.
- AFS-06 — Invitations, applications, auto-approval, custom commission.
- AFS-10 — How commission is charged; re-pricing of unbilled sales.
- AFS-12 — Plan minimum commission, brand/product limits.
- AFA-03 / AFA-05 — What affiliates see; link behaviour when a product is removed.

## Technical reference

- **Endpoints:** `GET seller/v1/products`, `GET seller/v1/products/{id}`, `PUT seller/v1/products/change-status` (gates `brand`, `asin`), `PUT seller/v1/products/{productId}/commission` (gate `commission`), `PUT seller/v1/products/bulk-update-commissions` (max 100 ids, gate `commission`), `GET seller/v1/products/{id}/variations`, product assets `…/{id}/upload-assets`, `…/{id}/assets`; brands `GET seller/v1/products/brands`, `GET …/brands/{id}`, `PUT …/brands/change-status` (gate `brand`), `PATCH …/brands/change-auto-partnership-approval`, `POST …/brands/update/terms-file` (pdf/doc/docx, 4096 KB), `PUT …/brands/update/amazon-link`, `PUT …/brands/update/socials/{id}`, `POST …/brands/{id}/profile`, brand assets; Shopify stores `GET seller/v1/stores`, `GET seller/v1/stores/{id}`.
- **Services:** `Services/General/Product/Product.php` (`updateCommissions` → events `ChangeDataOfProducts`, `ProductCommissionsUpdated`), listener `UpdateBrandContractProductCommissions` (all non-custom contract item products, no status filter), `Services/General/Product/Brand/UpdateBrandStatus.php` (cascades inactive to products only), `Listeners/AutoPartnershipApproval.php`. Imports: `Services/General/ProductSync/ProductSync.php::syncProducts` (Amazon, writes `commission` = default and status), `Services/General/Shopify/ProductSync/ShopifyProductSync.php::syncSellerProductPivotForVariant`.
- **Config:** `ainfluencer.seller.default_commission` (env `SELLER_DEFAULT_COMMISSION`, 15).
- **Data:** `seller_products` (`status`, `inactive_at`, `platform_status`, `commission`, `commission_type`, `seller_brand_id`, `accountable_type/id`), `seller_brands` (`status`, `inactive_at`, `auto_partnership_approval`, `slug`), `shopify_account_stores` (`status`, `auto_partnership_approval`), `seller_brand_amazon_links` (approval status), `partnership_contract_item_products` (`commission`, `is_custom_commission`), `amazon_reports` (`commission`, date vs `inactive_at`).
- **Frontend:** `spa-seller/src/components/discovery/products/*` (list, "Edit Commission" modal, detail), `spa-seller/src/components/discovery/brands/*` (list, detail, terms, Amazon link, assets).
