# Buyer app (aff-buyer, Next.js): live-code inventory

**Scope.** This covers `aff-buyer` at production tag v1.3.04. Backend calls are checked against `aff_routes.json` (affiliate backend v1.5.27). The source was only read, never changed.

**Machine-readable companion:** `aff-buyer-calls.json`: 27 records. 26 are affiliate-backend calls (25 live, 1 dead: reset-password), and 1 is the internal NextAuth route.

## 0. Who the "buyer" is

The **buyer is the shopper or end visitor** who clicks an affiliate's link. This app is the **public, mostly anonymous storefront and landing-page site**. It renders:
1. The **product landing page** behind an affiliate product link: `/product/{productLinkId}`. It shows the product, the brand, an "affiliate" badge, the seller's promotion or coupon, and a **"Buy Now on Amazon/Shopify"** button. That button opens the affiliate-tagged marketplace URL (`amazon_link`) in a new tab. Product collections and sliders come from the affiliate's landing-page template: grid layout, colors, and up to 3 sliders.
2. The **affiliate storefront** ("Your branded affiliate page" in the affiliate app): `/user/{affiliateId}`. It shows the affiliate's profile, social links, featured products, and all promoted products with category, order and "Discounted" filters.
3. The **seller brand/store landing page**: `/seller/brand/{slug}` for an Amazon brand and `/seller/store/{slug}` for a Shopify store. These pages target **prospective affiliates**, not shoppers. They show brand analytics, average commission, featured products, partnered affiliates and an **"Apply for partnership"** form. Logged-out visitors are handed to the affiliate app to finish.

Nothing is bought in this app. Buyers can create an optional **buyer account** (email/password with an OTP email check, or Google/Facebook/Amazon). The only benefit in the UI is seeing **coupon codes** ("Login to See") and "future discounted offers". There are no orders, carts, chats or tickets in the Next app. Checkout happens on Amazon or Shopify, and attribution comes from the affiliate tag in the link.

**spa-buyer replaced?** Yes. `aff-frontend/spa-buyer` is a 10-file Vue stub (login, register steps, profile and tickets only) that uses `/buyer/v1/auth/*`. Jenkins does not build it. The Next app is the real buyer front end; it uses `client/buyer/v1/*` plus `buyer/v1/auth/auth-status` and `buyer/v1/profile`. `spa-landpage` is not built either.

## 1. How it is built and deployed

- **Stack:** Next.js 14.2.5 (App Router, React 18, TypeScript), Tailwind with shadcn/radix UI, NextAuth 4 (Credentials provider, JWT session), axios, Sentry (`@sentry/nextjs`, tunnel route `/monitoring`), and Hotjar (production only, injected in the root layout).
- **Build and deploy:**
  - Jenkinsfile: tag `v*` → agent `ainfluencer-affiliate-production`. It runs `npm ci && npm run build`, builds a Docker image `ainfluencer-buyer-frontend:<tag>`, and runs the container with `-p $NEXT_APP_PORT` (default 3000) and `--restart unless-stopped`.
  - The environment is passed as build args and `docker run -e` from the agent: `NEXT_PUBLIC_GATEWAY`, `NEXT_PUBLIC_AFFILIATE_APP_URL`, `NEXT_PUBLIC_GOOGLE_RECAPTCHA_KEY`, `NEXTAUTH_URL` and `NEXTAUTH_SECRET`.
  - A legacy `.gitlab-ci.yml` (ssh to `18.117.175.47`) also exists; it appears superseded.
- **Aliases:** `@/*` → repo root (tsconfig). There is no `middleware.ts`, so **no route guard**: every page is public, and auth only changes what is shown.
- **HTTP clients:**
  - Server components use `fetch(`${NEXT_PUBLIC_GATEWAY}/...`)`. The calls are not cached (`cache: "no-cache"`), and a Bearer token from `getServerSession` is attached when one exists.
  - Client code uses the `axiosInstance` (`lib/api/axios-instance.ts`, baseURL `NEXT_PUBLIC_GATEWAY`). An interceptor adds `Authorization: Bearer <session.accessToken>`.
  - `useAxios` is a separate hook with the same base URL.
  - The dev gateway is `https://backstage.ainfluencer.com` (`.env.develop`).
- **Auth:**
  - NextAuth `CredentialsProvider` stores the backend Sanctum token as `session.accessToken`.
  - Login is modal-based: the root layout mounts `AuthManager`, which renders a dialog for the `?modal=signin|signup|code|forgotpassword|completeprofile|share` query parameter.
  - The header shows Register/Sign-in buttons or a user avatar menu. It appears only on `/product/[id]` and `/user/[id]`.

## 2. Live routes

| Path | File | Guard | Inbound links | Purpose |
|---|---|---|---|---|
| `/` | app/page.tsx | none | — | Client-side `redirect("https://ainfluencer.com")` |
| `/product/[id]` (`id` = affiliate product-link id) | app/product/[id]/page.tsx (+ loading.tsx, error.tsx) | none (session optional) | Affiliate links shared by affiliates (backend-generated); product cards in collections, storefront and brand pages (`productData.link`); sibling variations | Product landing page. Template grid (left/center/right) and colors come from the affiliate's landing template. Shows Amazon/Shopify availability, brand badge, affiliate badge (→ `/user/{affiliate.id}`), promotion with coupon (code visible only when logged in), Buy/View on marketplace, about/description/highlights/details, sibling variations, collections (sliders), benefits, footer disclaimer. If the backend answers non-2xx with `data.redirect_to`, the page redirects (see 3.1) |
| `/user/[id]` (`id` = affiliate id, URL-decoded if it contains %) | app/user/[id]/page.tsx (+ error.tsx) → domains/user/AffiliatePage.tsx | none | AffiliateBadge on product pages; "Copy Link" in the affiliate app's landing list; product redirect `affiliate_landing_page` | Affiliate storefront: profile and social accounts, featured products, product grid with search, categories (`X-Platform`), order (Newest, Oldest, High/Low Price, Discounted), pagination |
| `/seller/brand/[slug]` | app/seller/brand/[slug]/page.tsx (+ error, not-found) → domains/seller-brand/SellerBrandPage.tsx | none | Shared by sellers (seller app brand landing link). No in-app links | Amazon brand landing for prospective affiliates: rating, description, product count, active contracts, category, average commission, featured products, partnered affiliates (links to their landing pages), analytics, products grid, "Apply for partnership" (`#partnership-request`), terms file download |
| `/seller/store/[slug]` | app/seller/store/[slug]/page.tsx (+ error, not-found) → domains/seller-brand/SellerStorePage.tsx | none | Shared by Shopify sellers | Shopify store landing page (same idea, slug = Shopify `shop_name`) |
| `/auth?token=…&is_registered=0\|1` | app/auth/page.tsx | none | Backend social-login callback (`web/v1/buyer/login/{google\|facebook\|amazon}/callback`) redirects the popup here | Validates the token with `buyer/v1/auth/auth-status`, calls NextAuth `signIn`, posts `loginComplete` or `signUpComplete` to the opener and closes. The opener reloads; after sign-up it should open `?modal=completeprofile`, but the reload runs before `setSearchParams` (see 6) |
| `/api/auth/[...nextauth]` | app/api/auth/[...nextauth]/route.ts | — | next-auth client | NextAuth endpoints (session, csrf, callback/credentials, signout) |
| `/monitoring` | (Sentry tunnelRoute) | — | Sentry SDK | Sentry event tunnel |
| `?modal=…` on any page | managers/AuthManager.tsx | — | Header buttons; Coupon "Login to See" → signup; WelcomeBanner → signup; ProductDataHeader share → share; modal cross-links | signin, signup, code (email OTP), forgotpassword, completeprofile, share (Facebook/X/WhatsApp/Telegram/copy of current URL) |

**Dead inbound links:** the avatar menu (`components/Header/UserProfile.tsx`) links to `/dashboard` and `/saved-products`. Neither page exists, so both give a Next 404.

## 3. Flows and business rules

### 3.1 Shopper clicks an affiliate link
- **Server render:** the server fetches `GET client/buyer/v1/products/links/{id}`. It is called twice, once for metadata and once for the page; the page call carries a Bearer token if a session exists. The backend uses that token to decide promotion and coupon visibility. Its example response for guests has a `promotion` without `code`.
- **Redirect rules** (non-2xx with `data.redirect_to`). These implement the affiliate's **Redirect Setting** (affiliate app → My Links → Redirect Setting):
  - `affiliate_landing_page` → `/user/{entity_id}`, the affiliate storefront.
  - `product_landing_page` → `/product/{entity_id}`.
  - `redirect_to_amazon_store` → `data.link`, the external Amazon store.
  - `redirect_to_amazon` → `redirect(window.location.href)`. This **throws on the server** (`window` is undefined), so the visitor sees error.tsx instead of being sent to Amazon. This is a likely bug.
  - Any other non-2xx → "Error Fetching Product" plus a toast.
- **Buy button:** if `amazon_status === "in_stock"` it shows "Buy Now On {Amazon|Shopify}"; otherwise "View Product On …". Both open `amazon_link` (the affiliate-tagged URL) in a new tab. The marketplace (Amazon vs Shopify and the Amazon country) is detected from the link.
- **Promotions/coupons:** a promotion box shows the discount (percent or fixed) and "provided by {seller}". The **coupon code is shown only to authenticated buyers**; guests see a masked placeholder and "Login to See", which opens sign-up. Guests without a promotion see the WelcomeBanner ("By logging in… future discounted offers").
- **Variations:** `GET client/buyer/v1/products/links/{id}/variations` lists siblings; clicking one opens its link.

### 3.2 Affiliate storefront (`/user/[id]`)
- `GET client/buyer/v1/profile/affiliate/{id}` loads the profile, socials and featured products.
- `GET client/buyer/v1/profile/links/affiliate/{id}/products` loads the product grid with `page`, `per_page`, `category_ids[]`, `query`, and the order or `has_promotion` filter.
- Categories come from `GET client/buyer/v1/categories` with `X-Platform` and `user_type_id`.
- Product cards link to each product's `link`, i.e. the affiliate's tracked product landing.

### 3.3 Buyer account
- **Sign in:** `POST client/buyer/v1/auth/login {username,password,recaptcha_token}` → `GET buyer/v1/auth/auth-status` (Bearer) → NextAuth `signIn("credentials", {token,name,email,avatar})` → refresh.
- **Sign up:**
  1. `POST client/buyer/v1/auth/register {email,password,name,recaptcha_token}`, after the terms checkbox is ticked.
  2. `POST client/buyer/v1/auth/verification-code {username}` sends the code.
  3. `?modal=code&email=` → `POST client/buyer/v1/auth/email-verify` → auth-status → signIn. Resend posts verification-code again.
- **Social login** (Google, Facebook, Amazon): `POST client/buyer/v1/auth/login/{provider}` returns a `redirect_link`, opened in a 600x600 popup. The callback lands on `/auth?token=`, which completes NextAuth and messages the opener.
- **Complete profile:** `POST buyer/v1/profile {name}`, the username modal after social sign-up.
- **Forgot password:** `POST client/buyer/v1/auth/forget-password {email}`. **Completing the reset is broken:** the backend emails `{BUYER_BASE_URL}/user/reset-password/{token}` (config `sanctum.buyer.reset_password.url`). This app has no such route (it would 404), and the `ResetPassword` modal, which would call `client/buyer/v1/auth/reset-password`, is never imported or registered as a modal type.
- **Logout:** NextAuth `signOut` from the avatar menu. No backend call is made, so the Sanctum token is not revoked.

### 3.4 Prospective affiliate applies on a seller brand or store page
- **Data:**
  - Brand: `GET client/affiliate/v1/products/brands/{slug}`, then products from `GET client/affiliate/v1/products/{brandSlug}`.
  - Store: `GET client/affiliate/v1/products/stores/{slug}` and `.../products`.
  - Backend middleware `check_brand_landing_page` returns 404 for an unknown slug (→ not-found.tsx) and 403 if the seller's plan lacks the landing-page feature (→ error.tsx).
- **Apply for partnership** (with a message):
  - **Guest** (the normal case; these pages have no login UI):
    1. A uuid `temp_partnership_token` is kept in a cookie on the parent domain.
    2. `POST client/affiliate/v1/temp_partnership/brand/request {partnership_request_token, seller_brand_id, message, platform}`.
    3. Toast "Please sign in to complete", then redirect to `{NEXT_PUBLIC_AFFILIATE_APP_URL or https://affiliate.<parent>/app}?confirm-temp-partnership-requests=true`.
    4. In the affiliate app, login or register sends the token, and the dashboard confirms the request (`POST affiliate/v1/temp_partnership/brand/confirm-request`).
  - **Has a NextAuth session:** `POST affiliate/v1/partnerships/brand/request {message, seller_brand_ids:[id]}` with the **buyer** token. `AffiliateAccessMiddleware` returns 403 unless the same user also has an affiliate user-type.

### 3.5 Relationship to affiliates and sellers
- **Sellers** own the products, brands and stores and create promotions, which carry the coupon codes.
- **Affiliates** partner with sellers in the affiliate app, generate product links (optionally with a promotion, a source and a landing template), and choose redirect behaviour.
- **This app** renders those links and storefronts for shoppers, and renders seller landing pages that recruit affiliates.
- **Click/conversion tracking** is not done in this Next code. It presumably happens server-side when `client/buyer/v1/products/links/{id}` is fetched (backend), plus the affiliate tag in `amazon_link`.

## 4. Backend calls (live)

| Method | Path | Base / auth | File : fn | Pages / modal | Backend route | Problem |
|---|---|---|---|---|---|---|
| GET | `client/buyer/v1/products/links/{id}` | NEXT_PUBLIC_GATEWAY (server fetch, no auth) | app/product/[id]/page.tsx:37 : generateMetadata | /product/[id] | GET client/buyer/v1/products/links/{productLinkId} -> ShowProductLinkController [api,decode_hash_id] |  |
| GET | `client/buyer/v1/products/links/{id}` | NEXT_PUBLIC_GATEWAY (server fetch, Bearer from NextAuth session if logged in) | app/product/[id]/page.tsx:88 : ProductPage | /product/[id] | GET client/buyer/v1/products/links/{productLinkId} -> ShowProductLinkController [api,decode_hash_id] |  |
| GET | `client/buyer/v1/products/links/{id}/variations` | NEXT_PUBLIC_GATEWAY (client useAxios hook) | app/product/components/ProductSiblings/ProductSiblings.tsx:35 : ProductSiblings | /product/[id] | GET client/buyer/v1/products/links/{productLinkId}/variations -> ShowProductLinkVariationsController [api,decode_hash_id] |  |
| GET | `client/buyer/v1/profile/affiliate/{affiliateId}` | NEXT_PUBLIC_GATEWAY (server fetch, no auth) | app/user/[id]/page.tsx:55 : AffiliateServerPage | /user/[id] | GET client/buyer/v1/profile/affiliate/{affiliateId} -> ShowAffiliateProfileLinkController [api,decode_hash_id] |  |
| GET | `client/buyer/v1/profile/links/affiliate/{affiliateId}/products` | axiosInstance (NEXT_PUBLIC_GATEWAY) | domains/user/components/AffiliateProducts/AffiliateProducts.tsx:41 : getProducts | /user/[id] | GET client/buyer/v1/profile/links/affiliate/{affiliateId}/products -> AffiliateProfileProductsController [api,decode_hash_id] |  |
| GET | `client/buyer/v1/categories` | axiosInstance (NEXT_PUBLIC_GATEWAY) | domains/user/components/AffiliateProducts/CategoriesSelect.tsx:40 : loadCategories | /user/[id], /seller/brand/[slug], /seller/store/[slug] | GET client/buyer/v1/categories -> IndexCategoryController [api,decode_hash_id,scope_by_affiliate_or_platform] |  |
| GET | `client/affiliate/v1/products/brands/{slug}` | NEXT_PUBLIC_GATEWAY (server fetch, Bearer if session) | app/seller/brand/[slug]/page.tsx:16 : BrandPage | /seller/brand/[slug] | GET client/affiliate/v1/products/brands/{sellerBrandSlug} -> ShowBrandController [api,decode_hash_id,check_brand_landing_page:0] |  |
| GET | `client/affiliate/v1/products/{brandSlug}` | axiosInstance (NEXT_PUBLIC_GATEWAY) | domains/seller-brand/components/Products/BrandProducts.tsx:45 : getProducts | /seller/brand/[slug] | GET client/affiliate/v1/products/{sellerBrandSlug} -> ProductListController [api,decode_hash_id,check_brand_landing_page:0] |  |
| GET | `client/affiliate/v1/products/stores/{slug}` | NEXT_PUBLIC_GATEWAY (server fetch, Bearer if session) | app/seller/store/[slug]/page.tsx:16 : StorePage | /seller/store/[slug] | GET client/affiliate/v1/products/stores/{storeSlug} -> ShowStoreController [api,decode_hash_id,check_brand_landing_page:1] |  |
| GET | `client/affiliate/v1/products/stores/{storeSlug}/products` | axiosInstance (NEXT_PUBLIC_GATEWAY) | domains/seller-brand/components/Products/StoreProducts.tsx:42 : getProducts | /seller/store/[slug] | GET client/affiliate/v1/products/stores/{storeSlug}/products -> StoreProductListController [api,decode_hash_id,check_brand_landing_page:1] |  |
| POST | `affiliate/v1/partnerships/brand/request` | axiosInstance (NEXT_PUBLIC_GATEWAY, Bearer = NextAuth buyer session token) | domains/seller-brand/components/ApplyBrandPartnership.tsx:47 : handleApplyPartnership (logged-in branch) | /seller/brand/[slug], /seller/store/[slug] | POST affiliate/v1/partnerships/brand/request -> BrandRequestController [api,decode_hash_id,auth_or_api_key,affiliate_access,validate_multiple_platforms] | role mismatch risk: token is the buyer session token; AffiliateAccessMiddleware returns 403 unless the same user also owns an affiliate user-type. Brand/store pages render no login UI, so this branch only runs if a buyer session cookie already exists on this domain |
| POST | `client/affiliate/v1/temp_partnership/brand/request` | axiosInstance (NEXT_PUBLIC_GATEWAY) | domains/seller-brand/components/ApplyBrandPartnership.tsx:77 : handleApplyPartnership (guest branch) | /seller/brand/[slug], /seller/store/[slug] | POST client/affiliate/v1/temp_partnership/brand/request -> BrandRequestController [api,decode_hash_id] |  |
| POST | `client/buyer/v1/auth/login` | axiosInstance (NEXT_PUBLIC_GATEWAY) | modals/SignInModal.tsx:40 : handleSubmit | ?modal=signin (any page) | POST client/buyer/v1/auth/login -> LoginController [api,decode_hash_id] |  |
| GET | `buyer/v1/auth/auth-status` | axiosInstance (NEXT_PUBLIC_GATEWAY, explicit Bearer) | modals/SignInModal.tsx:53 : handleSubmit | ?modal=signin (any page) | GET buyer/v1/auth/auth-status -> AuthStatusController [api,decode_hash_id,auth:sanctum] |  |
| GET | `buyer/v1/auth/auth-status` | axiosInstance (NEXT_PUBLIC_GATEWAY, explicit Bearer) | modals/EmailConfirmation.tsx:48 : verify | ?modal=code (any page) | GET buyer/v1/auth/auth-status -> AuthStatusController [api,decode_hash_id,auth:sanctum] |  |
| GET | `buyer/v1/auth/auth-status` | axiosInstance (NEXT_PUBLIC_GATEWAY, explicit Bearer from ?token) | app/auth/page.tsx:29 : handleAuthorization | /auth?token=..&is_registered=.. | GET buyer/v1/auth/auth-status -> AuthStatusController [api,decode_hash_id,auth:sanctum] |  |
| POST | `client/buyer/v1/auth/register` | axiosInstance (NEXT_PUBLIC_GATEWAY) | modals/SignUpModal.tsx:35 : handleSubmit | ?modal=signup (any page) | POST client/buyer/v1/auth/register -> RegisterController [api,decode_hash_id] |  |
| POST | `client/buyer/v1/auth/verification-code` | axiosInstance (NEXT_PUBLIC_GATEWAY) | modals/SignUpModal.tsx:46 : handleSubmit | ?modal=signup (any page) | POST client/buyer/v1/auth/verification-code -> SendVerificationCodeController [api,decode_hash_id] |  |
| POST | `client/buyer/v1/auth/verification-code` | axiosInstance (NEXT_PUBLIC_GATEWAY) | modals/EmailConfirmation.tsx:103 : resend | ?modal=code (any page) | POST client/buyer/v1/auth/verification-code -> SendVerificationCodeController [api,decode_hash_id] |  |
| POST | `client/buyer/v1/auth/email-verify` | axiosInstance (NEXT_PUBLIC_GATEWAY) | modals/EmailConfirmation.tsx:37 : verify | ?modal=code (any page) | POST client/buyer/v1/auth/email-verify -> EmailVerifyController [api,decode_hash_id] |  |
| POST | `buyer/v1/profile` | axiosInstance (NEXT_PUBLIC_GATEWAY, Bearer from session) | modals/CompleteProfile.tsx:25 : saveUsername | ?modal=completeprofile (any page) | POST buyer/v1/profile -> UpdateProfileController [api,decode_hash_id,auth:sanctum,buyer_access] |  |
| POST | `client/buyer/v1/auth/forget-password` | axiosInstance (NEXT_PUBLIC_GATEWAY) | modals/ForgotPasswordModal.tsx:34 : handleSubmit | ?modal=forgotpassword (any page) | POST client/buyer/v1/auth/forget-password -> ForgetPasswordController [api,decode_hash_id] |  |
| POST | `client/buyer/v1/auth/login/google` | axiosInstance (NEXT_PUBLIC_GATEWAY) | lib/api/socailAuthHelpers.ts:5 : googleLogin | ?modal=signin, ?modal=signup | POST client/buyer/v1/auth/login/google -> GoogleLoginController [api,decode_hash_id] |  |
| POST | `client/buyer/v1/auth/login/facebook` | axiosInstance (NEXT_PUBLIC_GATEWAY) | lib/api/socailAuthHelpers.ts:30 : facebookLogin | ?modal=signin, ?modal=signup | POST client/buyer/v1/auth/login/facebook -> FacebookLoginController [api,decode_hash_id] |  |
| POST | `client/buyer/v1/auth/login/amazon` | axiosInstance (NEXT_PUBLIC_GATEWAY) | lib/api/socailAuthHelpers.ts:55 : amazonLogin | ?modal=signin, ?modal=signup | POST client/buyer/v1/auth/login/amazon -> AmazonLoginController [api,decode_hash_id] |  |
| GET/POST | `/api/auth/* (session, csrf, providers, callback/credentials, signout)` | Next.js route handler (same origin) | app/api/auth/[...nextauth]/route.ts : NextAuth handler | all pages (useSession/getServerSession/signIn/signOut) | — |  |

**Calls with no route or a wrong method:** none. All 25 live backend call records (21 distinct method+path pairs) match a route with the right method. The one functional problem is the role mismatch on `POST affiliate/v1/partnerships/brand/request` from a buyer session (3.4).

## 5. Dead code

The import walk starts from every `app/**/{page,layout,error,loading,not-found,global-error,route}.tsx` plus the instrumentation and Sentry configs. 171 of 174 code files are reachable. Unreached:
- `modals/ResetPassword.tsx`: the reset-password completion flow (`POST client/buyer/v1/auth/reset-password`). Not imported, and `AuthManager` has no modal type for it.
- `components/ui/card.tsx`, `components/ui/label.tsx`: unused shadcn components.
- `icons/global/WhatsappIcon.tsx`: unused.
- `shared/hooks/useScreenSizeBelow.ts`: unused.
- `app/types/next-auth.d.ts` is a type declaration, which is fine. Config files are not counted.

Also dead or unused:
- Avatar menu links `/dashboard` and `/saved-products` point to pages that do not exist.
- The `redirect_to_amazon` branch on the product page cannot work: `window` is used in a server component.
- `domains/seller-brand` and `app/product/components/Benefits/AlgoRiftBnefits.tsx` are duplicated (two copies of the benefits components).
- `app/seller/ARCHITECTURE.md` is documentation, not code.
- `NEXT_PUBLIC_BASE_URL` and `NEXT_PUBLIC_SITE_URL` are used in metadata (canonical and OG URLs) but are not passed by Jenkins or the Dockerfile, so canonical URLs render as `undefined/...`. This is likely, but not confirmed for production.

## 6. Uncertainties

- **Production hostname** of this app: not in the repo. Backend links use `BUYER_BASE_URL`; `.env.example` shows a stage host `stage.ainfluencerapp.com`, and the affiliate `.env` names `VUE_APP_BUYER_PANEL_URI=https://app.ainfluencerapp.com`, which may be stale. The cookie logic assumes the buyer app and the affiliate app share a parent domain (`affiliate.<parent>`).
- **Click tracking:** whether opening `/product/[id]` records a click (backend side) was not traced. The two server fetches per view (metadata plus page) could double-count if the backend counts on GET.
- **Post sign-up modal:** after social sign-up, `AuthManager` calls `window.location.reload()` before `setSearchParams("completeprofile")`, so the complete-profile modal probably never opens.
- **Coupon visibility** is decided by the backend response for authenticated users. The UI also hides the code for guests. Not verified against live API responses.
- **`spa-buyer`:** judged not deployed from the Jenkinsfile; whether an old deployment still serves it somewhere is unknown.
