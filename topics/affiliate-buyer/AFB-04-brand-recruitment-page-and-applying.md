---
id: AFB-04
title: Brand recruitment page and applying
domain: Buyer site (shoppers)
audience: [affiliate, seller]
apps: [buyer-site, affiliate-app]
related: [AFS-04, AFA-04, AFA-01, AFS-12, CHAT-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-buyer@v1.3.04, affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFB-04 · Brand recruitment page and applying

> A seller's public page for recruiting creators. It shows the brand or store, its products, commission and stats, with an "Apply for Partnership" form. A visitor without an affiliate login can apply first and finish after signing up in the affiliate app.

## Who and where
- **Prospective affiliates** (usually not logged in) on the buyer site:
  - Amazon brand: `/seller/brand/<slug>`
  - Shopify store: `/seller/store/<slug>` (the slug is the Shopify shop name)
- **Sellers** create and share the page from the seller app ("Branded landing page", AFS-04).
- There are no links to these pages inside the site; sellers share them directly.

## How it works
1. **The page** shows the brand/store name, rating, description, number of products, active contracts, category, average commission, featured products (up to 3), partnered affiliates (linking to their storefronts), optional statistics ("Clicks", "Conversions", "Revenue", "Sales Units"), a product grid, benefits, and a terms file download if the seller uploaded one.
2. **Apply.** In "Apply For Brand Partnership to <brand>" ("By partnering with this brand, you will gain access to all its products along with their default commission."), the visitor writes an optional message ("Write a message to the brand...") and clicks "Apply for Partnership".
3. **Visitor not logged in (normal case):**
   1. The request is saved on the server under a random token, and the token is stored in a browser cookie shared with the affiliate app.
   2. A message appears: "Partnership request saved! Please sign in to complete the process."
   3. The visitor is sent to the affiliate app (`affiliate.<domain>/app?confirm-temp-partnership-requests=true`).
   4. When they **register or log in** there (email or Google/Facebook), or open the dashboard while logged in, the saved request is turned into a real brand partnership request (AFA-04).
4. **Visitor logged in on the shopper site:** the request is sent directly ("Partnership request sent successfully!"). This works only if that login also has an affiliate profile. A plain shopper account gets an "Access denied" error.
5. The seller then approves or rejects as usual (AFS-06). If the brand has automatic approval on, it's approved instantly.

## Rules and limits
- **Plan:** the page works only if the seller's plan includes the **brand landing page** feature. Otherwise visitors see an error page ("Oops!" / "Something went wrong."). An unknown address shows "Brand Not Found" / "Store Not Found".
- The page also stops working when the brand is deactivated (AFS-04).
- The saved request lives in a **browser cookie that lasts until the browser is closed**, on the same device. If the visitor signs up later on another device or browser, the request is lost and they must apply again from the app.
- Duplicate requests are ignored: if the affiliate already has a pending or approved request with that seller's brand/store, nothing new is created.
- Applying for a brand covers **all its products at the default commission**.
- Questions to the brand should go through the **Ainfluencer chat** (CHAT-01).

## Statuses
Once confirmed, the request follows the normal partnership statuses: pending → approved / rejected / canceled (AFA-04).

## Timers and automatic actions
- The saved request has no server-side expiry; only the browser cookie limits it.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Request confirmed and pending | Seller | Daily 08:00 UTC "new partnership requests" email (AFS-06) |
| Request approved / rejected | Affiliate | In-app, email in the 06:00/18:00 batch |

## Common questions

### "I applied on the brand's page and signed up, but I don't see the request." (affiliate)
**Answer for the user:** The request is completed automatically only if you sign up or log in on the same browser right after applying. If you closed the browser or switched device, please apply again from Discovery in the affiliate app.
**What to check:** Whether a partnership request exists for that affiliate and brand (AFA-04).
**Escalate when:** They signed up in the same browser session and nothing was created.

### "The brand page shows an error." (seller or visitor)
**Answer for the user (seller):** Your plan needs to include the branded landing page, and the brand must be active. Check your subscription and brand status.
**What to check:** The seller's plan feature `brand_landing_page` (AFS-12); brand status; the slug in the address.
**Escalate when:** Plan and brand are fine and the page still errors.

### "It says 'Access denied' when I click Apply." (visitor)
**Answer for the user:** You're signed in with a shopper account. Please sign out of the shopper site (or use a private window) and apply again; you'll then be guided to the affiliate sign-up.
**What to check:** Whether the email has an affiliate profile.
**Escalate when:** —

### "Can I propose a different commission from this page?"
**Answer for the user:** No. Applying here asks for the brand's default commission. You can propose a commission for individual products from the affiliate app.
**What to check:** —
**Escalate when:** —

## Known issues
- A visitor signed in to a **shopper** account gets "Access denied" when applying.
- The saved request is lost if the browser is closed before signing up.
- *(To verify)* For Shopify store pages, the logged-in path doesn't send the platform, so it may fail where the guest path works.

## Related topics
- AFS-04 · Seller creates the brand landing page.
- AFA-04 · Partnership requests and approval.
- AFA-01 · Affiliate sign-up that finishes the request.
- AFS-12 · Seller plan features.
- CHAT-01 · Ainfluencer chat.

## Technical reference
- **Endpoints:**
  - `GET client/affiliate/v1/products/brands/{sellerBrandSlug}`, `GET client/affiliate/v1/products/{sellerBrandSlug}` (Amazon).
  - `GET client/affiliate/v1/products/stores/{storeSlug}`, `.../products` (Shopify).
  - All of these go through middleware `check_brand_landing_page` (`Http/Middleware/CheckBrandLandingPageAccess`: 404 unknown slug, 403 plan lacks `brand_landing_page`).
  - Guest apply: `POST client/affiliate/v1/temp_partnership/brand/request` (`BrandTempPartnershipService` → `temp_partnership_requests`).
  - Confirm: `POST affiliate/v1/temp_partnership/brand/confirm-request`, or `partnership_request_token` on register/login (`ConfirmPartnershipRequestJob` → `Services/General/Partnership/ConfirmPartnershipService`).
  - Logged-in apply: `POST affiliate/v1/partnerships/brand/request` (needs affiliate access).
- **Buyer code:** `aff-buyer/app/seller/{brand,store}/[slug]/page.tsx`, `domains/seller-brand/components/ApplyBrandPartnership.tsx` (cookie `temp_partnership_token`, domain = parent domain, no expiry). Affiliate app: `spa-affiliate/src/components/dashboard/affiliate-dashboard.page.vue::confirmTempPartnershipRequest`.
- **Data:** `temp_partnership_requests` (`token`, `type`, `message`) + items; converted into `partnership_requests`.
