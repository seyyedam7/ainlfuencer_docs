---
id: AFA-06
title: Landing pages and storefront
domain: Affiliate app (creators)
audience: [affiliate, buyer]
apps: [affiliate-app, buyer-site]
related: [AFA-05, AFA-09, AFA-10, AFB-01, AFB-02]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-06 · Landing pages and storefront

> Affiliates control how their links look to shoppers. **Landing templates** set the layout, colours and product sliders of the product page behind a link. The **branded affiliate page** (storefront) is their public shop page listing everything they promote.

## Who and where
- **Affiliates** in the affiliate app, sidebar "More" → **"Landing Templates"** (`/app/landpage-list`):
  - The card **"Your branded affiliate page"**, with "Assign Featured Products" and "Copy Link".
  - The list of landing templates, with create, edit, delete and set default.
  - Create: `/app/new-landing-page` ("Create New Landing Page"); edit: `/app/update-landpage/:id`.
- **Shoppers** see the results on the buyer site: product pages `/product/<link code>` (AFB-01) and the storefront `/user/<affiliate id>` (AFB-02).

## How it works
1. **A starting template exists.** Every new affiliate account gets a system landing page and a branded affiliate page automatically.
2. **Create a template** in 4 steps:
   1. "Choose Template Grid" (left, centre or right layout) and "Landing Name".
   2. Colours.
   3. Product sliders: "Up to 3 sliders", each with a "Slider Title" and up to 10 products picked from the affiliate's own links.
   4. Done.
3. **Use a template.** When creating a link, the affiliate picks a "Landing Page" (required, AFA-05). The shopper's product page then uses that template's layout, colours and sliders. "Set default" makes a template the default for new links and for the storefront's product sliders.
4. **Delete a template.** Links that used it move to the default template. If there is no default, they simply lose the assignment.
5. **Branded affiliate page.** "Assign Featured Products" picks **up to 3** featured products (from the affiliate's links). "Copy Link" copies the public storefront address. The storefront lists the affiliate's **default links** for active products (AFB-02).

## Rules and limits
- Up to **3 sliders** per template, **10 products** per slider; up to **3 featured products** on the branded page.
- Only products the affiliate has links for can be added.
- **Plan limit on templates** ("custom landing page": Free 1, Pro 5, Ultimate unlimited by default) is **not checked when creating** in the current version: the app check is switched off, and so is the server check. It is applied on a **downgrade or cancel**, when templates over the new limit are deleted automatically (AFA-09).
- A product appears on the storefront only if it has an **active default link** and the product is active.

## Statuses
Templates have no status beyond "default" or not.

## Timers and automatic actions
- On a plan downgrade or cancellation: extra templates are deleted (their links move to the default template).

## Notifications
None.

## Common questions

### "A product isn't showing on my storefront."
**Answer for the user:** The storefront shows products that have an active default link and are still active at the seller. If you made a new link, tick "Set this link as default link". If the partnership ended or the seller deactivated the product, it's removed from the page.
**What to check:** The link's `is_default` and `status`; product status; contract status.
**Escalate when:** An active default link for an active product still doesn't appear.

### "Some of my landing templates disappeared."
**Answer for the user:** When a plan is downgraded or cancelled, templates over the new plan's limit are removed automatically. Your links were moved to your default template.
**What to check:** Recent plan change (AFA-09).
**Escalate when:** No plan change happened.

### "Can I add more than 3 featured products / 3 sliders?"
**Answer for the user:** No. The branded page shows up to 3 featured products, and each template has up to 3 sliders of 10 products.
**What to check:** —
**Escalate when:** —

### "Where do I find my storefront link?"
**Answer for the user:** Landing Templates → "Your branded affiliate page" → "Copy Link".
**What to check:** —
**Escalate when:** —

## Known issues
- Template limits by plan aren't enforced at creation, but extra templates are deleted on downgrade. Users can lose templates they were allowed to create *(which templates count towards the limit, including system pages, is to verify)*.

## Related topics
- AFA-05 · Links and the "Landing Page" choice.
- AFA-09 · Plan limits and downgrade effects.
- AFA-10 · Landing-page report.
- AFB-01 · Product page that uses the template.
- AFB-02 · Public storefront.

## Technical reference
- **Endpoints:**
  - `GET/POST affiliate/v1/landing-pages`, `GET/PUT/DELETE affiliate/v1/landing-pages/{id}`, `PUT .../{id}/set-default`, `GET .../{id}/products`, `PUT .../{id}/user-type-collections` (featured products).
  - `GET affiliate/v1/landing-pages/template-configs`, `GET affiliate/v1/landing-pages/drop-down`, `GET affiliate/v1/products/links/drop-down`.
- **Validation:** `CreateProductLandingPageRequest` / `UpdateLandingPageRequest` (`sliders` max 3, products per slider max 10).
- **Logic:** `Services/General/LandingPage/LandingPage.php` (`delete` reassigns assignees to the default page; `setDefault`). Registration creates pages via jobs `CreateUserTypeLandingPage` and `CreateSystemLandingPage`.
- **Plan:** middleware `affiliate_access_feature:custom_landing_page` is commented out in `routes/affiliate/v1/landing_page.php`; downgrade `Plan/Downgrade/Affiliate/CustomLandingPageFeatureDowngrader` → `Cascade/Nodes/LandingPageNode`.
- **Data:** `landing_pages` (`user_type_id`, `type`, `is_default`), `landing_page_assignees` (link ↔ template), `landing_page_collections` (sliders).
- **Frontend:** `spa-affiliate/src/components/design-studio/**` (UI gate `hasProPlan || true`).
