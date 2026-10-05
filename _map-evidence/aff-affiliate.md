# Affiliate app (spa-affiliate): live-code inventory

**Scope.** This covers `aff-frontend/spa-affiliate` plus the `shared-components` it imports, at production tag v1.4.02 (production URL https://affiliate.ainfluencer.com). Backend calls are checked against the affiliate backend route dump `aff_routes.json` (v1.5.27). The source was only read, never changed.

**Machine-readable companion:** `aff-affiliate-calls.json`: 287 call records. 201 are live, covering 195 distinct method+path pairs. 86 are dead or never triggered by the UI.

## 1. How it is built and deployed

- **Stack:** Vue 2.7, vue-router 3 in history mode, Pinia, bootstrap-vue and vue-i18n. Built with Vue CLI 5 (`vue-cli-service build --modern`) and shipped as a PWA (workbox). Firebase Cloud Messaging provides web push (`public/firebase-messaging-sw.js`), Pusher provides realtime chat, and Sentry is enabled with replay at 10%.
- **Monorepo:** npm workspaces under `aff-frontend`: shared-components, spa-seller, spa-affiliate, spa-buyer, spa-admin and spa-landpage.
  - The Jenkinsfile builds only seller, affiliate and admin. A `v*` tag builds with `NAME=production`, runs `npm install && npm run build` in `spa-affiliate`, and moves `dist/*` to `~/sources/front/affiliate` on agent `ainfluencer-affiliate-production`.
- **Import aliases:**
  - `@/` → `spa-affiliate/src` (jsconfig and Vue CLI default).
  - `shared-components/...` → the workspace package.
  - Some shared files import the affiliate stores through the workspace path `spa-affiliate/src/store/...`.
- **Entry:** `src/main.js`.
  - It first imports `shared-components/algo-auth/script/auth-init.script.js`, an IIFE that handles the `/auth` and `/impersonate` paths before the router loads.
  - It then sets up Sentry, Pinia, i18n, global mixins (`use-image`, the permission mixin `$can`), the service worker, and `App.vue`.
  - `App.vue` adds a progress bar, global modal and alert hosts, and asks for push permission on first load when logged in. That flow ends in POST `affiliate/v1/user-fcm-tokens`.
- **Environment:** all API paths are `VUE_APP_*` variables in `spa-affiliate/.env` (228 lines). The HTTP client is axios with `baseURL: process.env.VUE_APP_API_BASE_URL`. Every authenticated call sends `Authorization: Bearer <localStorage.auth_token>` and `X-Platform: <current platform id>`.
  - **`VUE_APP_API_BASE_URL` is defined only in `.env.development`** (`https://backstage.ainfluencer.com`), plus the panel URLs and S3 URL. A production-mode build does not load `.env.development`, so the production base URL must come from the build agent's environment or an untracked `.env.*.local`. This could not be verified from the repo.
- **Global error handling** (`AuthorizationStore.handleHttpErrors`):
  - "Upgrade required" errors open a limit modal. Affiliate limit types are `seller_invite` (daily partnership requests), `active_contract`, `custom_landing_page`, `chat_system` and `default`.
  - **HTTP 499** sends the user to `/app/settings?tab=billing`, i.e. Stripe is required.
  - Everything else is thrown to the caller and shown as a red alert.
- **Platform context:** after login, the layout calls `GET affiliate/v1/profile/platform-accounts/dropdown`. The user then picks a platform in the header (`amazon-…` or `shopify-…`), stored in `localStorage.platform`.
  - The choice changes wording (Brand vs Store) and some endpoints (`/products/brands` vs `/stores`), and it is sent as the `X-Platform` header.
  - Switching platform reloads the page, or goes to `/app` when on a brand or product detail page.

## 2. Live routes

**Guards:**
- `/app`: the `getLayoutViewRoute` `beforeEnter` calls `GET affiliate/v1/auth/auth-status` (once per page load), `GET affiliate/v1/system/basic-info` and `GET affiliate/v1/settings/client`.
  - Not logged in → `/user/login`.
  - Logged in but registration incomplete → `/user/register-steps`.
  - On `dashboard`, a pending SSO `redirect` path stored in sessionStorage is consumed and followed.
- `/user`: logged-in users are bounced to the dashboard, except on `email-verified` and on `register-steps` while registration is incomplete.
- **Permission gating inside pages:** the `$can(permission, action)` mixin covers team-member impersonation permissions, and `custom_permission` gates Associates.
- **Other routes:**
  - `/` redirects to `/app`, and `*` redirects to `/` (unknown URLs land on the dashboard or login).
  - `/app/logout` clears localStorage and reloads.
  - `/user` redirects to `user/login`.
- **Pre-router paths** handled by `auth-init.script.js`:
  - `/auth?source=local&token=` saves the token and goes to `/`.
  - `/auth?status=success&token=` is the social-login popup result. It notifies the opener, saves the token and goes to `/`.
  - `/auth?status=error&message=` puts `auth-social-error` on the opener's URL.
  - `/auth?source=redirect&panel=seller` is the panel switch. It calls `GET {panel}/v1/auth/local-login` and goes to `data.link`.
  - `/impersonate?token=` is admin impersonation. It sets `impersonate_mode`, and logout then returns to `admin.<domain>`.
- **Navigation:**
  - Sidebar: Dashboard, Discovery (`products-discovery`), Associate (Explore / My Tags / Analytics, only if `custom_permission` includes `associate_access`), My Links (`active-management`), and More (Partnership with a request badge, My Lists, Landing Templates, Messages with an unread badge, Reports → performance-report, Payments → payment-history/commission).
  - Mobile footer: Dashboard, Discovery, Partnership, Management, Reports.
  - User menu: Profile, Settings, Help center (support-tickets), panel switch Affiliate ↔ Seller, daily-limit counters, Logout.
  - Teams: the sidebar also lists teams the user belongs to, and clicking one switches team (`GET affiliate/v1/teams/{id}/impersonate`, which returns a new token, then reload).


**Dashboard**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app` (dashboard) | @/components/dashboard/affiliate-dashboard.page.vue | auth (/app beforeEnter) | affiliate-app.layout.vue, app.layout.vue | Dashboard: upcoming payment (sales/revenue/remaining commission), brand & product partnership counts, last-week performance chart, top products, recent payments, recent partnerships, profile completion %. Also confirms temp partnership requests when ?confirm-temp-partnership-requests=true. |

**Profile**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/profile` (profile) | @/components/profile/profile-tabs.page.vue | auth (/app beforeEnter) | affiliate-dashboard.page.vue, user-menu-dropdown.component.vue | Tabs overview / edit_profile_info / get_better_profile (questions) / manage_platforms (connect social & websites). ?tab= |

**Discovery**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/products-discovery` (discovery-products) | @/components/discovery/products/merge/product-merge.page.vue | auth (/app beforeEnter) | affiliate-app.layout.vue, list-item-empty-state.component.vue, list-item.component.vue, product-list-show.section.vue, product-title.component.vue | Product discovery grid/list with filters; "Generate link" (pre-check -> partnership request -> links), add to list, pin. |
| `/app/product/:id` (product-details) | @/components/discovery/products/product-details/product-details.page.vue | auth (/app beforeEnter) | apply-partnership-modal.vue, asset-detail.modal.vue, commission-invoice.page.vue, get-promo-link.vue, inactive-product.table.vue, management-active-product-item | Product details: variations, assets, promotions, partnership status, apply/cancel, create/delete links, message seller. |
| `/app/brands-discovery` (discovery-brands) | @/components/discovery/brands/brand-list.page.vue | auth (/app beforeEnter) | list-item-empty-state.component.vue, list-item.component.vue | Brand (Amazon) / Store (Shopify) discovery list; apply for brand partnership; add to list. |
| `/app/brand/:id` (brand-details) | @/components/discovery/brands/brand-details.page.vue | auth (/app beforeEnter) | brand-application.table.vue, brand-card.component.vue, brand-card.vue, brand-details.component.vue, brand-list.table.vue, brand-partnership-detail.vue, brand-pa | Brand details, assets, products, apply, link to brand report. |
| `/app/seller/:id` (seller-details) | @/components/discovery/seller/seller-details.page.vue | auth (/app beforeEnter) | DetailSectionSeller.vue, brand-cancel-application.modal.vue, brand-card.component.vue, brand-card.vue, brand-details.page.vue, brand-list.table.vue, brand-partn | Seller profile: stats (commission paid, revenue, clicks, units, conversions), brands/products, storefront link, Send Message, Apply For Partnership. |
| `/app/discovery-promote` (discovery-promote) | @/components/discovery/promote/one-click-promote.page.vue | auth (/app beforeEnter) | (none found) | "One Click Promote": pick a product, auto-join at default commission (daily application limit shown). Orphan: sidebar entry commented out. |
| `/app/discovery-promote-all/:brandId?` (discovery-promote-all) | @/components/discovery/promote/promote-all/promote-all.page.vue | auth (/app beforeEnter) | brand-card.vue, one-click-promote.page.vue, product-card.vue, product-landing-row.vue | Promote all products (optionally of one brand) in one click. |

**My Lists**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/discovery-my-lists` (my-lists-discovery) | @/components/discovery/lists/lists.page.vue | auth (/app beforeEnter) | affiliate-app.layout.vue | Saved lists of brands/products: create, rename, duplicate, delete, pin. |
| `/app/lists/brands/:id` (lists-brands-details) | @/components/discovery/lists/brands/list-details.page.vue | auth (/app beforeEnter) | list-item.component.vue | Brands in a list; remove items. |
| `/app/lists/products/:id` (lists-products-details) | @/components/discovery/lists/products/list-details.page.vue | auth (/app beforeEnter) | list-item.component.vue | Products in a list; remove items. |

**My Links (management)**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/active-management` (active-management) | @/components/management/management-index.page.vue | auth (/app beforeEnter) | StepFour.vue, affiliate-app.layout.vue, apply-partnership.vue, get-product-link.vue | "My Links": active products/brands with link counts, Generate New Link, Message Seller, Redirect Setting modal. |
| `/app/inactive-products` (inactive-products) | @/components/management/inactive-product.table.vue | auth (/app beforeEnter) | management-index.page.vue | Inactive (no longer partnered/out of program) products. |
| `/app/inactive-brands` (inactive-brands) | @/components/management/inactive-brand.table.vue | auth (/app beforeEnter) | management-index.page.vue | Inactive brands/stores. |

**Partnerships**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/partnerships` (partnerships) | @/components/partnership/partnership-tabs.page.vue | auth (/app beforeEnter) | (none found) | Tab shell (Products / Brands) + sub-tabs Invitations, Partnered, Applications, Revoked, Rejected; "Auto System-Generated Partnership Requests" switch (UI only). |
| `/app/partnerships/products` (product-partnerships) | @/components/partnership/products/product-partnership.page.vue | auth (/app beforeEnter) | affiliate-app.layout.vue, partnership-tables.section.vue, partnership-tabs.page.vue | Product partnership requests/contracts; accept/reject invitations, cancel applications, revoke contracts. |
| `/app/partnerships/brands` (brand-partnerships) | @/components/partnership/brands/brand-partnership.page.vue | auth (/app beforeEnter) | partnership-tables.section.vue, partnership-tabs.page.vue | Brand partnership requests/contracts (same actions). |
| `/app/my-sellers` (my-sellers-partnerships) | @/components/partnership/my-sellers/my-sellers.table.vue | auth (/app beforeEnter) | partnership-tabs.page.vue | Sellers the affiliate is partnered with. |

**Landing templates**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/landpage-list` (landpage-list) | @/components/design-studio/landpage-list/landpage-list.page.vue | auth (/app beforeEnter) | StepFour.vue, affiliate-app.layout.vue | Landing templates list + "affiliate landing" (branded affiliate page, copy link, featured products); set default, edit, delete, assign slider products. |
| `/app/new-landing-page` (new-landpage) | @/components/design-studio/create-update-landing-page/create-landpage.wrapper.vue | auth (/app beforeEnter) | (none found) | 4-step wizard: grid + name, colors, up to 3 sliders x 10 products, done. |
| `/app/update-landpage/:id` (update-landpage) | @/components/design-studio/create-update-landing-page/update-landpage.wrapper.vue | auth (/app beforeEnter) | landpage-card.component.vue | Same wizard in update mode. |

**Associates**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/associates/explore` (associates-explore) | @/components/associate/explore/explore-associate.page.vue | auth (/app beforeEnter) | affiliate-app.layout.vue | Amazon Associates via Ainfluencer: browse associate products/brands, scrape a product URL, get promo link with your tag ("Get Extra Commission via Ainfluencer"). Visible only with custom_permission associate_access. |
| `/app/associates/my-tags` (associates-tags) | @/components/associate/my-tags/associate-tags.page.vue | auth (/app beforeEnter) | affiliate-app.layout.vue | Associate promote tags (set default) and associate social media list/add. |
| `/app/associates/analytics` (associates-analytics) | @/components/associate/analytics/associate-analytics.page.vue | auth (/app beforeEnter) | affiliate-app.layout.vue, tags-table.vue | Associate sale report chart + table. |

**Reports**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/reports` (reports) | @/components/report/report-tabs.page.vue | auth (/app beforeEnter) | (none found) | Report tab shell + "Export Data As CSV" (downloads file). |
| `/app/reports/performance-report` (performance-report) | @/components/report/performance/performance-report.page.vue | auth (/app beforeEnter) | affiliate-app.layout.vue, commission-item-card.component.vue, commission-transactions.table.vue, report-tabs.page.vue, upcoming-commission-item-card.component.v | Performance list + chart. |
| `/app/reports/brand-report` (brand-report) | @/components/report/brand/brand-report.page.vue | auth (/app beforeEnter) | brand-details.page.vue, brand-section.vue, report-tabs.page.vue | Per-brand report. |
| `/app/reports/product-report` (product-report) | @/components/report/product/product-report.page.vue | auth (/app beforeEnter) | report-tabs.page.vue | Per-product report. |
| `/app/reports/source-report` (source-report) | @/components/report/source/source-report.page.vue | auth (/app beforeEnter) | report-tabs.page.vue | Per link-source report. |
| `/app/reports/landingpage-report` (landingpage-report) | @/components/report/landpage/landingpage-report.page.vue | auth (/app beforeEnter) | report-tabs.page.vue | Per landing page report. |

**Payments & subscription**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/payment-history` (payment-history) | @/components/payment-history/payment-history.page.vue | auth (/app beforeEnter) | profile-widget-items.js | Payments tab shell (Commission / Subscription). |
| `/app/payment-history/commission` (payment-history-commission) | @/components/payment-history/commission/commission-index.vue | auth (/app beforeEnter) | Billing.vue, affiliate-app.layout.vue, payment-history.page.vue, recent-payment.vue | Commission transactions (paid statements) + upcoming commission transactions. |
| `/app/payment-history/commission-invoice/:id` (payment-history-commission-invoice) | @/components/payment-history/commission/commission-invoice/commission-invoice.page.vue | auth (/app beforeEnter) | commission-transactions.table.vue | Commission statement detail: items + export. |
| `/app/payment-history/subscription` (payment-history-subscription) | @/components/payment-history/subscription-transactions.table.vue | auth (/app beforeEnter) | Billing.vue, manage-subscription.page.vue, payment-history.page.vue | Subscription invoices and charges. |
| `/app/manage-subscription` (manage-subscription) | @/components/manage-subscription/manage-subscription.page.vue | auth (/app beforeEnter) | limit-modal.modal.vue, manage-subscription-sidebar-widget.component.vue | Affiliate plans: subscribe (payment link), upgrade/downgrade, cancel, resume, subscription status. |

**Messages**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/messages` (messages) | sc/algo-chat/Chat.vue | auth (/app beforeEnter) | affiliate-app.layout.vue, layout-header.layout.vue, use-message.js | Chat with sellers (contacts, messages, saved messages/templates, tags, search, realtime via Pusher). |

**Support**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/support-tickets` (support-tickets) | sc/algo-tickets/ticket-list.page.vue | auth (/app beforeEnter) | user-menu-dropdown.component.vue | Ticket list + new ticket. |
| `/app/support-ticket-details/:id` (support-ticket-details) | sc/algo-tickets/ticket-details/ticket-details.page.vue | auth (/app beforeEnter) | ticket-item.component.vue, ticket-list.page.vue | Ticket thread: reply, close. |

**Settings**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/app/settings` (settings) | sc/algo-setting/algo-setting.page.vue | auth (/app beforeEnter) | payment-history.page.vue, user-menu-dropdown.component.vue | Tabs: Message templates, Billing (Stripe Connect onboarding / manage; links to statements), Notifications, Security (sessions, reset password). Team tab is seller-only. ?tab= |

**Auth (/user)**

| Path | Component | Guard | Inbound links (files) | Purpose |
|---|---|---|---|---|
| `/user/login` (login) | sc/algo-auth/auth-login.vue | guest-only (/user beforeEnter -> dashboard if logged in) | auth-forgot-password.vue, auth-register-email-verify.component.vue, auth-register-form.component.vue, auth-reset-password.vue | Email+password+reCAPTCHA; Google/Facebook/Amazon popup login; attaches temp_partnership_token cookie. Success -> /app/products-discovery. |
| `/user/register` (register) | sc/algo-auth/auth-register/auth-register.vue | guest-only (/user beforeEnter -> dashboard if logged in) | auth-login.vue | Register form (+reCAPTCHA, temp partnership token) -> "Check your email" screen. |
| `/user/forgot-password` (forgot-password) | sc/algo-auth/auth-forgot-password.vue | guest-only (/user beforeEnter -> dashboard if logged in) | auth-login.vue, auth-reset-password.vue | Request reset email. |
| `/user/reset-password/:token` (seller-reset-form) | sc/algo-auth/auth-reset-password.vue | guest-only (/user beforeEnter -> dashboard if logged in) | (none found) | Set new password from emailed link. |
| `/user/email-verified/:token` (email-verified) | sc/algo-auth/auth-email-verified.vue | none (allowed when logged in) | (none found) | Token landing for email-verify link and core-Ainfluencer SSO; stores token (+?redirect) then -> /user. |
| `/user/register-steps` (register-steps) | @/components/auth/register-steps.page.vue | logged-in & register incomplete | (none found) | Server-driven onboarding steps: verify_mobile, questions, sync_social_account (Google/Facebook), social_media (connect platforms), profile, done; optional steps skippable. |
## 3. User flows and business rules (as the UI implements them)

### 3.1 Registration, login and email verification
- **Register** (`/user/register`): form with reCAPTCHA v2 → `POST affiliate/v1/auth/register`. If a `temp_partnership_token` cookie exists (set by the buyer app's brand page), it is sent as `partnership_request_token`. Next comes a "Check your email" screen. The email link goes through the backend (`web/v1/affiliate/email/verify/{id}/{hash}`) and arrives at `/user/email-verified/{token}`, which stores the token and goes to `/user`. The guard then sends the user to `/user/register-steps`.
- **Login** (`/user/login`): email, password and reCAPTCHA → `POST affiliate/v1/auth/login`. The partnership token is attached here too. On success the app does a hard redirect to `/app/products-discovery`. In development builds, reCAPTCHA is skipped.
- **Social login** (Google, Facebook, Amazon): `POST affiliate/v1/auth/login/{google|facebook|amazon}` returns a `redirect_link`, which opens in a popup. The backend callback (`web/v1/affiliate/login/*/callback`) redirects the popup to `/auth?status=success&token=`. The popup hands the result to the opener through `window.opener[CLOSE_ALGO]`, or through localStorage on Safari.
- **Forgot/reset password:**
  - `POST affiliate/v1/auth/forget-password` emails `{AFFILIATE_BASE_URL}/user/reset-password/{token}`, and that page calls `POST affiliate/v1/auth/reset-password`.
  - From Settings → Security, the "reset password" modal calls `POST affiliate/v1/profile/request-change-password-by-email`.
- **Register steps** (`/user/register-steps`): `GET affiliate/v1/profile/show-register-steps` returns `[{step_title, is_passed, is_mandatory, sort_order}]`. The UI shows the first step that is not yet passed. Optional steps can be skipped (`POST affiliate/v1/auth/skip-step {step}`). The steps are:
  - `verify_mobile`: SMS OTP. `POST auth/mobile-verification-code`, then `POST auth/mobile-verify`. The OTP is 6 digits with a 60-second resend timer. Some countries cannot be selected (`VUE_APP_IGNORED_COUNTRIES`: PK, IN, RU, PRK, IQ, IR, CU, SD, SY).
  - `questions`: `GET` then `POST affiliate/v1/questions`.
  - `sync_social_account`: link a Google or Facebook identity (`POST profile/sync/account/{google|facebook}` returns a popup).
  - `social_media`: connect promotion platforms (see 3.3).
  - `profile`: `POST affiliate/v1/profile`.
  - `done`.
- **Logout** clears all localStorage (token, platform, settings) and tells other tabs through a BroadcastChannel. A login in another tab reloads this one.

### 3.2 SSO from core Ainfluencer and the panel switch
- **Core → affiliate SSO:**
  1. Core sends the user to backend `GET web/v1/sso/login?token=<JWT>&redirect=<path>`.
  2. `SsoLoginController` → `Sso::login` decodes the HS256 JWT (`issuer` selects the source handler) and creates or finds the user.
  3. It issues a Sanctum token and redirects to `{seller|affiliate base}/user/email-verified/{token}?redirect=<path>`. Seller-type users go to the seller panel.
  4. The SPA saves the token and normalizes `redirect` to an in-app `/app/...` path. Absolute and `//` URLs are rejected. The path is held in sessionStorage.
  5. After the auth and register-steps checks pass, the dashboard route redirects to that path.
  - If the JWT is invalid or expired, the backend shows the view `sso.login_error`.
- **Affiliate → Seller panel:**
  - If `auth-status` reports `seller_account`, the app opens `seller_link` in a new tab.
  - Otherwise it asks for confirmation ("create Seller account") and calls `GET seller/v1/auth/become-seller`. It then opens `/auth?source=redirect&panel=seller`, which fetches `GET seller/v1/auth/local-login` (Bearer) and goes to the returned link.

### 3.3 Connecting promotion platforms (Profile → Manage platforms, and register step `social_media`)
- **OAuth-connected platforms (v2):**
  - `GET affiliate/v2/social-medias` lists them.
  - Connect: `GET affiliate/v2/social-medias/oauth/{youtube|tiktok|instagram|pinterest|facebook}?action=social` returns a `redirect_link`, opened in a popup. The callback is `web/v2/affiliate/social-medias/login/{platform}/callback`.
  - Disconnect: `DELETE affiliate/v2/social-medias/{id}`.
  - Websites use a token instead: `POST affiliate/v2/social-medias/websites/social-verification-token`, then `POST .../websites/verify`.
- **Manual verification platforms (v1):** the UI warns "may take up to 12 hours".
  - Add: `POST affiliate/v1/social-medias`. Edit: `PUT affiliate/v1/social-medias/{id}`. Delete: `POST affiliate/v1/social-medias/delete {social_media_ids}`. Verify: `PUT affiliate/v1/social-medias/verify`.
  - `POST affiliate/v1/social-medias/check-website` checks a website URL.
  - The type list comes from `GET affiliate/v1/social-medias/types`.
- **Categories:** each platform must have a category. The page warns "Choose platform category" and saves them with `PUT affiliate/v2/social-medias/category {social_medias:[{id, category_id}]}`. Categories come from `GET affiliate/v1/categories`.
- **Linked login identities** (Google, Facebook): `GET` and `DELETE affiliate/v1/profile/synced-accounts[/{id}]`.
- **Hidden in this app:** the Amazon/Shopify seller-account connection block in the same shared component is seller-only.

### 3.4 Finding products and brands (Discovery)
- **Products** (`/app/products-discovery`):
  - `GET affiliate/v1/products` with filters, plus categories, brand dropdown and seller dropdown.
  - Actions: pin, add to list, generate link, message seller.
- **Brands** (`/app/brands-discovery`): Amazon uses `GET affiliate/v1/products/brands` and Shopify uses `GET affiliate/v1/stores`.
- **Detail pages:**
  - Brand detail: `GET products/brands/{id}`. Brand assets: `GET products/brands/{id}/assets[/{assetId}]`.
  - Product detail: `GET products/{id}` and `GET products/{id}/variations`. Product assets: `GET products/{id}/assets[/{assetId}]`.
  - Seller page: `GET profile/seller/{id}`, showing commission paid, revenue, clicks, sales units, conversions, active contracts and storefront link.
- **My Lists:**
  - `affiliate/v1/user-type-lists*`: create, rename (PATCH), delete, duplicate, assign items, delete items, and brand/product lists with details.
  - Pin and unpin: `POST affiliate/v1/pins/toggle`.
- **One Click Promote** (`/app/discovery-promote`, orphaned: its sidebar entry is commented out) and **Promote All** (`/app/discovery-promote-all/:brandId?`):
  - `GET affiliate/v1/products/landing-page/index` lists products. `POST affiliate/v1/products/landing-page/promote/{id}` auto-joins at the default commission.
  - The page shows the daily application limit from `invites_limit` (spend / limit, where -1 means unlimited).

### 3.5 Partnerships with sellers
- **Model:** an affiliate partners with a seller per **product** or per **brand/store**. Each partnership is a contract with a commission.
  - Seller → affiliate requests are **invitations** ("received"). Affiliate → seller requests are **applications** ("sent").
  - The UI tabs are Invitations, Partnered (active), Applications, Revoked and Rejected, for both the Products and Brands sides.
- **Apply for a brand:** `POST affiliate/v1/partnerships/brand/request {seller_brand_ids, message}`.
- **Apply for products / "Generate link"** from Discovery:
  1. `POST affiliate/v1/partnerships/product/check-pre-request {seller_product_ids}` returns `custom_commission_products`, `total_requests` and `duplicate_requests`.
  2. If no product needs a custom commission and the affiliate is not already partnered with all of them, the request is sent at once. Otherwise the affiliate may enter a commission % for products that need seller approval.
  3. `POST affiliate/v1/partnerships/product/request {seller_product_ids, commission_type:"percent", commission}` returns product links ready to copy.
  4. If already partnered with all, the modal offers "Go to Management".
- **Answering and withdrawing:**
  - Accept/reject a received invitation: `PATCH partnerships/invited/{brand|product}/change-status/{id}`.
  - Cancel a sent application: `PATCH partnerships/applied/{brand|product}/change-status/{id}`.
  - Revoke an active contract: `PUT partnerships/contracts/{id}/revoke`.
  - Lists: `GET partnerships/requests`, `GET partnerships/contracts`, `GET partnerships/contracts/{id}`, `GET partnerships/sellers` (My Sellers).
- **Limits** (backend "upgrade required" modal): daily partnership requests (`seller_invite`) and the active contract limit. The user menu shows the daily counters (`chat_limit`, `invites_limit`, `reveal_email_limit`).
- **Partnership started from the buyer app's brand page while logged out:**
  1. The brand page stores a `temp_partnership_token` cookie on the parent domain and records the request (`POST client/affiliate/v1/temp_partnership/brand/request`).
  2. It redirects to `affiliate.<domain>/app?confirm-temp-partnership-requests=true`.
  3. After login or register, which also send the token, the dashboard calls `POST affiliate/v1/temp_partnership/brand/confirm-request {partnership_request_token}`.
- **Not persisted:** the "Auto System-Generated Partnership Requests" switch on the Partnership page is local state only. No API call is made.

### 3.6 Affiliate links, coupons/promotions and redirects ("My Links")
- **Create, edit or delete a link** (product detail or management):
  - Create: `POST affiliate/v1/products/links`. Edit: `PUT affiliate/v1/products/links/{id}`. Delete: `DELETE affiliate/v1/products/links/{id}`.
  - Optional settings: a **source** (`GET/POST affiliate/v1/products/links/sources`, plus a drop-down), a **landing page** (`GET affiliate/v1/landing-pages/drop-down`), a **promotion/coupon** (`GET products/promotions/drop-down` or `GET products/{id}/promotions/drop-down`), and "Set this link as default".
- **Coupons:** promotions are created by sellers. The affiliate only attaches one to a link. The buyer then sees the coupon on the product landing page.
- **Management lists:** `/app/active-management` (`GET managements/products`, `GET managements/brands`) and the inactive lists.
- **Redirect Setting** (`GET/POST affiliate/v1/settings`) controls where a visitor goes in three cases: the link is removed, the product is out of stock, or the product is removed or disabled. The options are Amazon with the affiliate's associate tag, the affiliate landing page, the Amazon store (only if the profile has an Amazon store link), or none.

### 3.7 Landing pages and storefront
- **Templates:**
  - List: `GET affiliate/v1/landing-pages`. Detail: `GET landing-pages/{id}`. Delete: `DELETE`. Set default: `PUT landing-pages/{id}/set-default`.
  - Create: `POST affiliate/v1/landing-pages`. Update: `PUT landing-pages/{id}`. Template configs: `GET landing-pages/template-configs`.
  - The wizard has 4 steps: grid template + name, colors, up to 3 product sliders of 10 products each (picked from your links via `GET products/links/drop-down`), and done.
- **Affiliate landing ("Your branded affiliate page"):** copy the link, and assign featured products with `PUT landing-pages/{id}/user-type-collections`. `GET landing-pages/{id}/products` lists them.
  - The public page is served by the buyer app (`/user/{id}`, and `/product/{linkId}` for product landings).
- **Plan gate:** the UI gate is disabled (`hasProPlan || true`). The backend can still answer with the `custom_landing_page` upgrade modal.

### 3.8 Associates (Amazon Associates through Ainfluencer)
- **Visibility:** only shown when `custom_permission` includes `associate_access`.
- **Explore:** associate products and brands (`associates/associate-products/list`, `show/{id}`, `associate-brands/list`). "Find products to request" scrapes an Amazon URL (`POST associate-products/scrape`). "Get promo link" uses the affiliate's associate promote tag. The banner reads "Get Extra Commission via Ainfluencer".
- **My Tags:** `associate-promote-tags/list` and setting a default (`PATCH .../{id}/default`), plus associate social media list and add.
- **Analytics:** `associate-sale-reports/chart` and `/list`.

### 3.9 Teams
- **Switching only:** affiliates can switch into teams they belong to from the sidebar (`GET affiliate/v1/teams/{id}/impersonate` returns a new token). Inside a team, actions are limited by team permissions (`$can`).
- **Management is not available:** the team management tab in Settings is seller-only (`canView` requires `AppInfo.isSeller`). Its affiliate env URLs are also stale (`/teams/members/{ID}` does not match the backend `teams/{teamId}/members/list`), so it would 404 if it were ever shown.

### 3.10 Messages (chat with sellers)
- **Endpoints:**
  - Contacts: `GET affiliate/v1/chats/contacts`. Messages: `GET .../contacts/{id}/messages`, `POST chats/messages`. Search: `GET chats/messages`.
  - Saved messages: `GET/POST chats/messages/saved`. Tags: `POST/DELETE chats/contacts/tag`. Mark read: `PUT chats/read-contact-messages`. Find page: `POST chats/messages/find-page`.
- **Message seller** from any page: `GET chats/find-contact/{sellerId}`. For a new contact it also calls `GET profile/seller/{id}`, then opens `/app/messages`.
- **Realtime:** Pusher (key in `.env`, cluster us3), with channel auth at `POST /broadcasting/auth` (`user_type=affiliate`).
- **Daily limit:** new conversations are capped (`chat_limit`, upgrade modal type `chat_system`).
- **Message templates:** Settings → Message templates (`affiliate/v1/message-templates/*`, `message-variables/list`).

### 3.11 Reports, dashboard and earnings
- **Dashboard:** `affiliate/v1/dashboard/report/{upcoming-payments,brands-partnership,products-partnership,top-products,recent-payments,statistics}`, plus `GET dashboard/contracts`, `GET profile/profile-completion-percentage`, and the last-week performance chart.
- **Reports:** performance (GET list and chart), and brand, product, source and landing-page (POST list and chart). "Export Data As CSV" posts to `.../export` and downloads the file.

### 3.12 Payouts, payout method and subscription
- **Payout method = Stripe Connect.** Settings → Billing → "Connect" or "Manage Credit Card" calls `GET affiliate/v1/payments/onboarding-request` and opens the returned `url`.
  - `auth-status.payment_profile_status` is `connected`, `not_connected` or another value (shown as "Check your credit card").
  - Many backend actions answer HTTP 499 until Stripe is connected, which sends the user to `/app/settings?tab=billing`.
- **Earnings:**
  - Commission statements: `GET payments/commission-transactions`, `GET .../{id}`, `.../{id}/items`, `.../{id}/export`.
  - Upcoming commission: `GET payments/upcoming-commission-transactions`.
  - Subscription invoices and charges: `GET payments/invoices`, `GET payments/charges`.
- **Subscription** (`/app/manage-subscription`): `GET affiliate/v1/plans`, `GET plans/{id}/subscribe` (payment link), `PUT plans/{id}/update` (upgrade/downgrade), `PUT plans/cancel`, `PUT plans/resume`, `GET plans/subscriptions/status`, `GET plans/active`. The header badge shows Free or Pro (`plan.isPlanFree`).

### 3.13 Notifications, settings, support
- **Notifications:** sidebar panel (`GET affiliate/v1/notifications`, `/suggests`, `PATCH {id}/read`, `PATCH read-all`, `PATCH {id}/toggle-bookmark`) and web push through FCM (`POST affiliate/v1/user-fcm-tokens`; on denial it sends `token:null`). Settings → Notifications: `GET/PUT affiliate/v1/settings/notifications[/update]`.
- **Security:** sessions list and terminate (`GET/DELETE affiliate/v1/profile/sessions[/{id}]`). "Terminate all" is broken because its env variable is missing.
- **Client settings** (UI flags such as `youtube_modal_show`): `GET affiliate/v1/settings/client`, `POST affiliate/v1/settings/client/store`.
- **Onboarding guidelines:** `GET affiliate/v1/guidelines/{key}` and `/general`.
- **Support tickets:** `GET/POST affiliate/v1/tickets`, `GET tickets/{id}`, `POST tickets/{id}/messages`, `PATCH tickets/{id}` (close).

## 4. Backend calls by area (live)

Paths are shown as the frontend builds them; a leading slash is optional. All live calls match a backend route with the right method, with one exception, listed after the tables. Full details (file:line, env variable, callers) are in the JSON.

#### Panel switching / SSO

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/seller/v1/auth/become-seller` | affiliateBecomesSeller (sc/store/authentication.store.js:239) | (app layout: all /app/*) | BecomeSellerController |  |
| GET | `/{PANEL}/v1/auth/local-login` | redirect (sc/algo-auth/script/auth-init.script.js:125) | /auth?source=redirect&panel=X (pre-router script; opened by panel switcher in user menu) | LocalLoginCallbackController \| GET buyer/v1/auth/local-login |  |

#### Auth & registration

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/auth/auth-status` | checkTokenValidation (sc/store/authorization.store.js:344) | (router guard on /app/* and /user/*) | AuthStatusController |  |
| POST | `/affiliate/v1/auth/forget-password` | forgotPassword (sc/store/authentication.store.js:72) | /user/forgot-password | ForgetPasswordController |  |
| POST | `/affiliate/v1/auth/login` | login (sc/store/authentication.store.js:20) | /user/forgot-password, /user/login, /user/register, /user/reset-password/:token | LoginController |  |
| POST | `/affiliate/v1/auth/login/amazon` | loginWithAmazon (sc/store/social-service.store.js:397) | /user/login, /user/register | AmazonLoginController |  |
| POST | `/affiliate/v1/auth/login/facebook` | loginWithFacebook (sc/store/social-service.store.js:397) | /user/login, /user/register | FacebookLoginController |  |
| POST | `/affiliate/v1/auth/login/google` | loginWithGoogle (sc/store/social-service.store.js:397) | /user/login, /user/register | GoogleLoginController |  |
| POST | `/affiliate/v1/auth/mobile-verification-code` | verifyPhone (sc/store/authentication.store.js:126) | /user/register-steps | SendMobileVerificationCodeController |  |
| POST | `/affiliate/v1/auth/mobile-verify` | verifyOTP (sc/store/authentication.store.js:148) | /user/register-steps | MobileVerifyController |  |
| POST | `/affiliate/v1/auth/register` | register (sc/store/authentication.store.js:59) | /user/login, /user/register, /user/register-steps | RegisterController |  |
| POST | `/affiliate/v1/auth/reset-password` | resetPassword (sc/store/authentication.store.js:84) | /user/reset-password/:token | ResetPasswordController |  |
| POST | `/affiliate/v1/auth/skip-step` | skipRegisterStep (sc/store/authentication.store.js:211) | /user/register-steps | SkipStepController |  |
| GET | `/affiliate/v1/profile/questions` | getQuestionList (sc/store/profile.store.js:112) | profile | ProfileQuestionListController |  |
| POST | `/affiliate/v1/profile/questions` | postQuestionList (sc/store/profile.store.js:125) | profile | StoreProfileQuestionController |  |
| GET | `/affiliate/v1/profile/show-register-steps` | getRegisterStepsList (sc/store/authentication.store.js:199) | /user/register-steps | ShowRegisterStepsController |  |
| GET | `/affiliate/v1/questions` | getQuestionList (sc/store/authentication.store.js:172) | /user/register-steps | RegisterQuestionListController |  |
| POST | `/affiliate/v1/questions` | postQuestionList (sc/store/authentication.store.js:184) | /user/register-steps | StoreRegisterQuestionController |  |

#### Teams

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/teams/{ID}/impersonate` | switchTeam (sc/store/settings.store.js:240) | (app layout: all /app/*) | GetImpersonationTokenController |  |

#### Profile & connected platforms

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/associates/associate-social-medias/list` | getAssociateSocialMediaList (@/store/associate.store.js:145) | associates/my-tags | IndexAssociateSocialMediaController |  |
| POST | `/affiliate/v1/associates/associate-social-medias/store` | storeAssociateSocialMedia (@/store/associate.store.js:175) | associates/explore, associates/my-tags | StoreAssociateSocialMediaController |  |
| GET | `/affiliate/v1/profile` | getProfileInfo (sc/store/profile.store.js:18) | profile | ShowProfileController |  |
| POST | `/affiliate/v1/profile` | updateProfileInfo (sc/store/profile.store.js:61) | profile, /user/register-steps | UpdateProfileController |  |
| PUT | `/affiliate/v1/profile/change-mobile` | verifyProfileMobileOtp (sc/store/profile.store.js:185) | profile | ChangeMobileNumberController |  |
| GET | `/affiliate/v1/profile/overview` | getProfileOverView (sc/store/profile.store.js:47) | profile | ProfileOverviewController |  |
| GET | `/affiliate/v1/profile/platform-accounts/dropdown` | getPlatformListDropdown (@/store/global.store.js:34) | (app layout: all /app/*) | IndexPlatformAccountController |  |
| GET | `/affiliate/v1/profile/profile-completion-percentage` | getCompleteProfilePercentage (@/store/dashboard.store.js:32) | /app | ShowProfileCompletionPercentageController |  |
| POST | `/affiliate/v1/profile/request-change-mobile` | verifyProfileMobileNumber (sc/store/profile.store.js:174) | profile | RequestChangeMobileNumberController |  |
| GET | `/affiliate/v1/profile/seller/{id}` | getSellerInfo (@/store/seller.store.js:81) | /app, active-management, brand/:id, brands-discovery, discovery-promote, discovery-promote-all/:brandId?, inac | ShowSellerDetailController |  |
| GET | `/affiliate/v1/profile/sessions` | dataProvider (sc/algo-setting/tabs/Security.vue:158) | settings | SessionListController |  |
| DELETE | `/affiliate/v1/profile/sessions/{ID}` | terminate (sc/algo-setting/tabs/Security.vue:181) | settings | DeleteSessionController |  |
| POST | `/affiliate/v1/profile/sync/account/facebook` | syncFacebookAccount (sc/store/social-service.store.js:397) | profile, /user/register-steps | SyncFacebookAccountController |  |
| POST | `/affiliate/v1/profile/sync/account/google` | syncGoogleAccount (sc/store/social-service.store.js:397) | profile, /user/register-steps | SyncGoogleAccountController |  |
| GET | `/affiliate/v1/profile/synced-accounts` | getSyncedSocialAccountsList (sc/store/social-service.store.js:215) | profile, /user/register-steps | SyncedAccountListController |  |
| DELETE | `/affiliate/v1/profile/synced-accounts/{id}` | deleteSocialAccount (sc/store/social-service.store.js:229) | profile, /user/register-steps | DeleteSyncedAccountController |  |
| GET | `/affiliate/v1/social-medias` | getAffiliateSocialAccountsList (@/store/affiliate.store.js:121) | profile | IndexSocialMediaController |  |
| POST | `/affiliate/v1/social-medias` | addNewAffiliateSocialAccount (@/store/affiliate.store.js:68) | profile, /user/register-steps | CreateSocialMediaController |  |
| POST | `/affiliate/v1/social-medias/check-website` | checkWebsiteSocialMedia (@/store/affiliate.store.js:33) | profile | CheckWebsiteSocialMediaController |  |
| POST | `/affiliate/v1/social-medias/delete` | deleteAffiliateSocialAccount (@/store/affiliate.store.js:165) | profile, /user/register-steps | DeleteSocialMediaController |  |
| GET | `/affiliate/v1/social-medias/types` | getSelectOptionListForSocialType (@/store/affiliate.store.js:16) | profile, /user/register-steps | IndexSocialMediaTypeController |  |
| PUT | `/affiliate/v1/social-medias/verify` | verifyAffiliateSocialAccounts (@/store/affiliate.store.js:198) | profile, /user/register-steps | VerifySocialMediaController |  |
| PUT | `/affiliate/v1/social-medias/{id}` | updateAffiliateSocialAccount (@/store/affiliate.store.js:86) | profile | UpdateSocialMediaController |  |
| GET | `/affiliate/v2/social-medias` | getAffiliateSocialAccountsListV2 (@/store/affiliate.store.js:145) | profile, /user/register-steps | IndexSocialMediaController |  |
| PUT | `/affiliate/v2/social-medias/category` | updateAffiliateSocialAccountV2 (@/store/affiliate.store.js:103) | profile, /user/register-steps | UpdateSocialMediaCategoryController |  |
| GET | `/affiliate/v2/social-medias/oauth/{SOCIAL_MEDIA_NAME}` | syncSocialMediaAccount (sc/store/social-service.store.js:124) | profile, /user/register-steps | YoutubeLoginController \| GET affiliate/v2/social-medias/oauth/tiktok |  |
| DELETE | `/affiliate/v2/social-medias/{ID}` | deleteAffiliateSocialMediaV2 (@/store/affiliate.store.js:184) | profile, /user/register-steps | DeleteSocialMediaController |  |
| POST | `affiliate/v1/profile/request-change-password-by-email` | resetPassword (sc/store/profile.store.js:137) | settings | RequestChangePasswordByEmailController |  |
| POST | `affiliate/v2/social-medias/websites/social-verification-token` | getWebsiteVerificationToken (@/store/affiliate.store.js:218) | profile, /user/register-steps | GetWebsiteSocialMediaVerificationTokenController |  |
| POST | `affiliate/v2/social-medias/websites/verify` | verifyWebSiteSocialMedia (@/store/affiliate.store.js:235) | profile, /user/register-steps | VerifyWebsiteSocialController |  |

#### Partnerships

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| PATCH | `/affiliate/v1/partnerships/applied/brand/change-status/{ID}` | updateBrandPartnershipSentStatus (@/store/partnership.store.js:140) | brand/:id, partnerships/brands | ChangeStatusAppliedBrandRequestController |  |
| PATCH | `/affiliate/v1/partnerships/applied/product/change-status/{ID}` | updateProductPartnershipSentStatus (@/store/partnership.store.js:178) | /app, partnerships/products, product/:id | ChangeStatusAppliedProductRequestController |  |
| POST | `/affiliate/v1/partnerships/brand/request` | sendPartnershipBrandRequestToSeller (@/store/partnership.store.js:221) | associates/explore, brand/:id, brands-discovery, product/:id, seller/:id | BrandRequestController |  |
| GET | `/affiliate/v1/partnerships/contracts` | getPartnershipContracts (@/store/partnership.store.js:196) | partnerships/brands, partnerships/products | IndexPartnershipContractController |  |
| GET | `/affiliate/v1/partnerships/contracts/{ID}` | getPartnershipContract (@/store/partnership.store.js:47) | /app | ShowPartnershipContractController |  |
| PUT | `/affiliate/v1/partnerships/contracts/{ID}/revoke` | revokeContract (@/store/partnership.store.js:253) | /app, partnerships/brands, partnerships/products | PartnershipContractRevokeController |  |
| PATCH | `/affiliate/v1/partnerships/invited/brand/change-status/{ID}` | updateBrandPartnershipReceivedStatus (@/store/partnership.store.js:121) | /app, partnerships/brands | ChangeStatusInvitedBrandRequestController |  |
| PATCH | `/affiliate/v1/partnerships/invited/product/change-status/{ID}` | updateProductPartnershipReceivedStatus (@/store/partnership.store.js:159) | /app, partnerships/products | ChangeStatusInvitedProductRequestController |  |
| POST | `/affiliate/v1/partnerships/product/check-pre-request` | checkPartnershipProductPreRequest (@/store/partnership.store.js:29) | brand/:id, products-discovery | ProductBulkRequestCheckController |  |
| POST | `/affiliate/v1/partnerships/product/request` | sendPartnershipProductRequestToSeller (@/store/partnership.store.js:235) | associates/explore, brand/:id, product/:id, products-discovery, seller/:id | ProductRequestController |  |
| GET | `/affiliate/v1/partnerships/requests` | getPartnershipRequests (@/store/partnership.store.js:97) | /app, partnerships/brands, partnerships/products | IndexPartnershipRequestController |  |
| GET | `/affiliate/v1/partnerships/sellers` | getPartnershipSellers (@/store/partnership.store.js:81) | my-sellers | SellerListController |  |
| POST | `/affiliate/v1/temp_partnership/brand/confirm-request` | confirmTempPartnershipRequests (@/store/partnership.store.js:64) | /app | BrandConfirmRequestController |  |

#### Links & management (My Links)

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/managements/brands` | getManagementBrandList (@/store/brand.store.js:177) | active-management, inactive-brands | BrandListController |  |
| GET | `/affiliate/v1/managements/products` | getManagementProductList (@/store/product.store.js:366) | active-management, inactive-products | ProductListController |  |
| GET | `/affiliate/v1/products/landing-page/index` | getLandingProductList (@/store/product.store.js:56) | discovery-promote | LandingPageListController |  |
| POST | `/affiliate/v1/products/landing-page/promote/{ID}` | promoteProduct (@/store/product.store.js:76) | discovery-promote, discovery-promote-all/:brandId? | LandingPagePromoteController |  |
| POST | `/affiliate/v1/products/links` | createLink (@/store/links.store.js:16) | active-management, product/:id | CreateProductLinkController |  |
| GET | `/affiliate/v1/products/links/drop-down` | getLandingPageProductsLinks (@/store/landpage.store.js:65) | landpage-list | ProductLinkDropDownListController |  |
| GET | `/affiliate/v1/products/links/sources` | getSourceLinkList (@/store/links.store.js:119) | active-management, product/:id, reports/brand-report, reports/landingpage-report, reports/product-report | IndexProductLinkSourceController |  |
| POST | `/affiliate/v1/products/links/sources` | createSource (@/store/links.store.js:51) | active-management, product/:id | CreateProductLinkSourceController |  |
| GET | `/affiliate/v1/products/links/sources/drop-down` | getSourceLinkListDropDown (@/store/links.store.js:135) | active-management, product/:id, reports/source-report | ProductLinkSourceDropDownListController |  |
| DELETE | `/affiliate/v1/products/links/{id}` | deleteProductLink (@/store/product.store.js:193) | product/:id | DeleteProductLinkController |  |
| PUT | `/affiliate/v1/products/links/{id}` | updateLink (@/store/links.store.js:34) | active-management, product/:id | UpdateProductLinkController |  |
| GET | `/affiliate/v1/settings` | getManagementRedirectSetting (@/store/product.store.js:26) | active-management | ShowUserTypeSettingController |  |
| POST | `/affiliate/v1/settings` | updateManagementRedirectSetting (@/store/product.store.js:39) | active-management | UpdateUserTypeSettingController |  |

#### Landing pages

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/landing-pages` | getLandpagesList (@/store/landpage.store.js:100) | landpage-list | IndexLandingPageController |  |
| POST | `/affiliate/v1/landing-pages` | saveLandpage (@/store/landpage.store.js:82) | new-landing-page, update-landpage/:id | CreateProductLandingPageController |  |
| GET | `/affiliate/v1/landing-pages/drop-down` | getLandingPageList (@/store/product.store.js:297) | active-management, product/:id | LandingPageDropDownListController |  |
| GET | `/affiliate/v1/landing-pages/drop-down` | getLandpageTemplates (@/store/landpage.store.js:304) | reports/landingpage-report | LandingPageDropDownListController |  |
| GET | `/affiliate/v1/landing-pages/template-configs` | getLandpageTemplateConfigs (@/store/landpage.store.js:224) | new-landing-page, update-landpage/:id | IndexLandingPageTemplateConfigsController |  |
| GET | `/affiliate/v1/landing-pages/{id}` | getLandingPage (@/store/landpage.store.js:273) | landpage-list, new-landing-page, update-landpage/:id | ShowLandingPageController |  |
| PUT | `/affiliate/v1/landing-pages/{id}` | updateLandpage (@/store/landpage.store.js:338) | new-landing-page, update-landpage/:id | UpdateLandingPageController |  |
| DELETE | `/affiliate/v1/landing-pages/{landpageID}` | deleteLandingpage (@/store/landpage.store.js:238) | landpage-list | DeleteLandingPageController |  |
| GET | `affiliate/v1/landing-pages/{landpageId}/products` | getLandingPageProductsList (@/store/landpage.store.js:30) | landpage-list | IndexLandingPageProductController |  |
| PUT | `affiliate/v1/landing-pages/{landpageId}/set-default` | setDefaultLandpage (@/store/landpage.store.js:46) | landpage-list | SetDefaultLandingPageController |  |
| PUT | `affiliate/v1/landing-pages/{landpageId}/user-type-collections` | assignLandpageFeaturedProducts (@/store/landpage.store.js:17) | landpage-list | UpdateUserTypeLandingPageCollectionController |  |

#### Associates

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/associates/associate-brands/list` | getAssociateBrandList (@/store/associate.store.js:224) | associates/explore | IndexAssociateBrandController |  |
| GET | `/affiliate/v1/associates/associate-products/list` | getAssociateProducts (@/store/associate.store.js:16) | associates/explore | IndexAssociateProductController |  |
| POST | `/affiliate/v1/associates/associate-products/scrape` | sendAssociateProductLinkScrape (@/store/associate.store.js:48) | associates/explore | ScrapeAssociateProductController |  |
| GET | `/affiliate/v1/associates/associate-products/show/{ID}` | getAssociateProductDetails (@/store/associate.store.js:35) | associates/explore | ShowAssociateProductController |  |
| GET | `/affiliate/v1/associates/associate-promote-tags/list` | getAssociatePromoteTags (@/store/associate.store.js:65) | associates/analytics, associates/explore, associates/my-tags | ListAssociatePromoteTagController |  |
| PATCH | `/affiliate/v1/associates/associate-promote-tags/{ID}/default` | setDefaultAssociatePromoteTag (@/store/associate.store.js:82) | associates/my-tags | SetDefaultAssociatePromoteTagController |  |
| GET | `/affiliate/v1/associates/associate-sale-reports/chart` | getAssociateSaleReportChart (@/store/associate.store.js:129) | associates/analytics | ChartAssociateSaleReportController |  |
| GET | `/affiliate/v1/associates/associate-sale-reports/list` | getAssociateSaleReportList (@/store/associate.store.js:113) | associates/analytics | IndexAssociateSaleReportController |  |

#### Dashboard & reports

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/dashboard/contracts` | getRecentPartnership (@/store/dashboard.store.js:19) | /app | IndexPartnershipContractController |  |
| GET | `/affiliate/v1/dashboard/report/brands-partnership` | getBrandInfo (@/store/dashboard.store.js:73) | /app | ShowBrandPartnershipReportController |  |
| GET | `/affiliate/v1/dashboard/report/products-partnership` | getProductInfo (@/store/dashboard.store.js:86) | /app | ShowProductPartnershipReportController |  |
| GET | `/affiliate/v1/dashboard/report/products-partnership` | getProductInfo (@/store/report.store.js:19) | /app | ShowProductPartnershipReportController |  |
| GET | `/affiliate/v1/dashboard/report/recent-payments` | getRecentPaymentReport (@/store/dashboard.store.js:145) | /app | RecentPaymentsReportController |  |
| GET | `/affiliate/v1/dashboard/report/statistics` | getReportStatistics (@/store/dashboard.store.js:45) | /app | ShowStatisticsReportController |  |
| GET | `/affiliate/v1/dashboard/report/top-products` | getTopProductReport (@/store/dashboard.store.js:130) | /app | TopProductReportController |  |
| GET | `/affiliate/v1/dashboard/report/upcoming-payments` | getUpcomingPaymentInfo (@/store/dashboard.store.js:60) | /app | ShowUpcomingPaymentReportController |  |
| POST | `/affiliate/v1/reports/brand` | getBrandList (@/store/report.store.js:85) | reports/brand-report | BrandReportListController |  |
| POST | `/affiliate/v1/reports/brand/chart` | getBrandReportChart (@/store/report.store.js:103) | reports/brand-report | BrandReportChartController |  |
| POST | `/affiliate/v1/reports/brand/export` | getBrandExportFile (@/store/report.store.js:121) | reports, reports/brand-report | BrandReportExportController |  |
| POST | `/affiliate/v1/reports/landing-page` | getLandingPageList (@/store/report.store.js:258) | reports/landingpage-report | LandingPageReportListController |  |
| POST | `/affiliate/v1/reports/landing-page/chart` | getLandingPageReportChart (@/store/report.store.js:276) | reports/landingpage-report | LandingPageReportChartController |  |
| POST | `/affiliate/v1/reports/landing-page/export` | getLandingPageExportFile (@/store/report.store.js:294) | reports, reports/landingpage-report | LandingPageReportExportController |  |
| GET | `/affiliate/v1/reports/performance` | getPerformanceList (@/store/report.store.js:33) | reports/performance-report | PerformanceReportListController |  |
| GET | `/affiliate/v1/reports/performance/chart` | getPerformanceChart (@/store/report.store.js:50) | /app, reports/performance-report | PerformanceReportChartController |  |
| POST | `/affiliate/v1/reports/performance/export` | getPerformanceExportFile (@/store/report.store.js:66) | reports, reports/performance-report | PerformanceReportExportController |  |
| POST | `/affiliate/v1/reports/product` | getProductList (@/store/report.store.js:140) | reports/product-report | ProductReportListController |  |
| POST | `/affiliate/v1/reports/product/chart` | getProductReportChart (@/store/report.store.js:158) | reports/product-report | ProductReportChartController |  |
| POST | `/affiliate/v1/reports/product/export` | getProductExportFile (@/store/report.store.js:177) | reports, reports/product-report | ProductReportExportController |  |
| POST | `/affiliate/v1/reports/source` | getSourceList (@/store/report.store.js:203) | reports/source-report | SourceReportListController |  |
| POST | `/affiliate/v1/reports/source/chart` | getSourceReportChart (@/store/report.store.js:221) | reports/source-report | SourceReportChartController |  |
| POST | `/affiliate/v1/reports/source/export` | getSourceExportFile (@/store/report.store.js:239) | reports, reports/source-report | SourceReportExportController |  |

#### Payments, payouts & subscription

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/payments/charges` | getSubscriptionCharges (@/store/billing.store.js:153) | payment-history/subscription | IndexChargeController |  |
| GET | `/affiliate/v1/payments/commission-transactions` | getCommissionTransactions (@/store/billing.store.js:81) | payment-history/commission | IndexCommissionTransactionController |  |
| GET | `/affiliate/v1/payments/commission-transactions/{ID}` | showCommissionTransaction (@/store/billing.store.js:48) | payment-history/commission, payment-history/commission-invoice/:id | ShowCommissionTransactionController |  |
| GET | `/affiliate/v1/payments/commission-transactions/{ID}/export` | exportCommissionTransactionInvoice (@/store/billing.store.js:111) | payment-history/commission-invoice/:id | CommissionTransactionExportController |  |
| GET | `/affiliate/v1/payments/commission-transactions/{ID}/items` | showCommissionTransactionInvoice (@/store/billing.store.js:93) | payment-history/commission-invoice/:id | IndexCommissionTransactionItemsController |  |
| GET | `/affiliate/v1/payments/invoices` | getSubscriptionInvoices (@/store/billing.store.js:136) | payment-history/subscription | IndexInvoiceController |  |
| GET | `/affiliate/v1/payments/onboarding-request` | stripeOnboardRequest (@/store/billing.store.js:18) | settings | OnboardingRequestController |  |
| GET | `/affiliate/v1/payments/upcoming-commission-transactions` | getUpcomingCommissionTransactions (@/store/billing.store.js:63) | payment-history/commission | IndexUpcomingCommissionTransactionController |  |
| GET | `/affiliate/v1/plans` | getPaymentPlans (sc/store/manage-subscription.store.js:16) | manage-subscription | IndexDefaultPlanController |  |
| GET | `/affiliate/v1/plans/active` | getActivePlan (sc/store/manage-subscription.store.js:110) | manage-subscription | ShowActivePlanController |  |
| PUT | `/affiliate/v1/plans/cancel` | cancelPlan (sc/store/manage-subscription.store.js:48) | manage-subscription | CancelPlanSubscriptionController |  |
| PUT | `/affiliate/v1/plans/resume` | resumeSubscription (sc/store/manage-subscription.store.js:63) | manage-subscription | ResumePlanSubscriptionController |  |
| GET | `/affiliate/v1/plans/subscriptions/status` | getSubscriptionStatus (sc/store/manage-subscription.store.js:97) | manage-subscription | SubscriptionStatusController |  |
| GET | `/affiliate/v1/plans/{ID}/subscribe` | requestPaymentLink (sc/store/manage-subscription.store.js:33) | manage-subscription | PlanSubscribeController |  |
| PUT | `/affiliate/v1/plans/{ID}/update` | updatePlan (sc/store/manage-subscription.store.js:82) | manage-subscription | UpdatePlanSubscriptionController |  |

#### Messages

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/chats/contacts` | getContacts (sc/algo-chat/store/chat.store.js:179) | messages | IndexContactController |  |
| GET | `/affiliate/v1/chats/contacts` | searchContact (sc/algo-chat/store/chat.store.js:194) | messages | IndexContactController |  |
| DELETE | `/affiliate/v1/chats/contacts/tag` | deleteContactTag (sc/algo-chat/store/chat.store.js:346) | messages | DeleteContactTagController |  |
| POST | `/affiliate/v1/chats/contacts/tag` | createContactTag (sc/algo-chat/store/chat.store.js:331) | messages | CreateContactTagController |  |
| GET | `/affiliate/v1/chats/contacts/{PARAM1}/messages` | getAllMessages (sc/algo-chat/store/chat.store.js:229) | messages | HistoryMessageController |  |
| GET | `/affiliate/v1/chats/find-contact/{contactId}` | findContact (sc/algo-chat/store/chat.store.js:306) | /app, active-management, brand/:id, brands-discovery, discovery-promote, discovery-promote-all/:brandId?, inac | FindContactController |  |
| GET | `/affiliate/v1/chats/messages` | searchMessage (sc/algo-chat/store/chat.store.js:259) | messages | SearchMessageController |  |
| POST | `/affiliate/v1/chats/messages` | sendMessage (sc/algo-chat/store/chat.store.js:243) | messages | CreateMessageController |  |
| POST | `/affiliate/v1/chats/messages/find-page` | findPage (sc/algo-chat/store/chat.store.js:375) | messages | FindMessagePageController |  |
| GET | `/affiliate/v1/chats/messages/saved` | getSavedMessages (sc/algo-chat/store/chat.store.js:289) | messages | IndexSavedMessageController |  |
| POST | `/affiliate/v1/chats/messages/saved` | createSaveMessage (sc/algo-chat/store/chat.store.js:273) | messages | CreateSavedMessageController |  |
| PUT | `/affiliate/v1/chats/read-contact-messages` | updateReadCount (sc/algo-chat/store/chat.store.js:360) | messages | ReadContactMessagesController |  |
| DELETE | `/affiliate/v1/message-templates/delete/{ID}` | deleteSavedMessage (sc/store/settings.store.js:470) | settings | DeleteMessageTemplateController |  |
| GET | `/affiliate/v1/message-templates/list` | getSavedMessages (sc/store/settings.store.js:408) | settings | ListMessageTemplateController |  |
| POST | `/affiliate/v1/message-templates/store` | createSavedMessage (sc/store/settings.store.js:436) | settings | StoreMessageTemplateController |  |
| PUT | `/affiliate/v1/message-templates/update/{ID}` | updateSavedMessage (sc/store/settings.store.js:453) | settings | UpdateMessageTemplateController |  |
| GET | `/affiliate/v1/message-variables/list` | getMessageVariables (sc/store/settings.store.js:485) | settings | ListMessageVariableController |  |
| POST | `/broadcasting/auth` | subscribePresence (sc/algo-chat/store/chat.store.js:41) | (any page that instantiates the chat store), messages | BroadcastController@authenticate |  |

#### Notifications

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/notifications` | getNotificationList (sc/store/notification.store.js:23) | (app layout: all /app/*) | NotificationListController |  |
| PATCH | `/affiliate/v1/notifications/read-all` | markAllNotificationAsRead (sc/store/notification.store.js:74) | (app layout: all /app/*) | ReadAllNotificationController |  |
| GET | `/affiliate/v1/notifications/suggests` | getSuggestionList (sc/store/notification.store.js:43) | (app layout: all /app/*) | SearchSuggestionNotificationController |  |
| PATCH | `/affiliate/v1/notifications/{NOTIF_ID}/read` | markNotificationAsRead (sc/store/notification.store.js:93) | (app layout: all /app/*) | ReadNotificationController |  |
| PATCH | `/affiliate/v1/notifications/{NOTIF_ID}/toggle-bookmark` | toggleBookmarkStatus (sc/store/notification.store.js:62) | (app layout: all /app/*) | ToggleBookmarkNotificationController |  |
| GET | `/affiliate/v1/settings/notifications` | loadSettingNotification (sc/store/settings.store.js:42) | settings | GetUserNotificationSettingController |  |
| PUT | `/affiliate/v1/settings/notifications/update` | changeSettingNotification (sc/store/settings.store.js:58) | settings | StoreUserNotificationSettingController |  |
| POST | `/affiliate/v1/user-fcm-tokens` | sendTokenToServer (sc/services/pushNotificationService.js:56) | (App.vue created: after login, push-permission prompt) | CreateUserFcmTokenController |  |

#### Support tickets

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/tickets` | getTicketsList (sc/store/ticket.store.js:56) | support-tickets | IndexTicketController |  |
| POST | `/affiliate/v1/tickets` | createTicket (sc/store/ticket.store.js:16) | support-tickets | CreateTicketController |  |
| GET | `/affiliate/v1/tickets/{id}` | getTicketDetails (sc/store/ticket.store.js:76) | support-ticket-details/:id | ShowTicketController |  |
| PATCH | `/affiliate/v1/tickets/{id}` | closeTicket (sc/store/ticket.store.js:93) | support-ticket-details/:id, support-tickets | CloseTicketController |  |
| POST | `/affiliate/v1/tickets/{id}/messages` | sendTicketMessage (sc/store/ticket.store.js:36) | support-ticket-details/:id | CreateTicketMessageController |  |

#### System / settings

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/settings/client` | fetchSettings (sc/store/settings.store.js:27) | (app layout: all /app/*), /app, active-management, associates/analytics, associates/explore, associates/my-tags, br | GetUserSettingController |  |
| POST | `/affiliate/v1/settings/client/store` | updateSettings (sc/store/settings.store.js:114) | (app layout: all /app/*), /app, active-management, associates/analytics, associates/explore, associates/my-tags, br | StoreUserSettingController |  |
| GET | `/affiliate/v1/system/basic-info` | fetchSystemInfoFromServer (sc/store/system-info.store.js:24) | (router guard on /app/* and /user/*) | BasicInfoController |  |
| GET | `affiliate/v1/guidelines/general` | getGlobalOnboarding (sc/algo-onboarding/onboarding-store.js:29) | (app layout: all /app/*) | GuidelinePageController |  |
| GET | `affiliate/v1/guidelines/{key}` | getOnBoardingForKey (sc/algo-onboarding/onboarding-store.js:17) | (app layout: all /app/*) | GuidelinePageController |  |

#### Discovery & lists

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/categories` | getCategoryList (@/store/product.store.js:159) | active-management, associates/explore, brand/:id, brands-discovery, discovery-my-lists, discovery-promote, dis | IndexCategoryController |  |
| GET | `/affiliate/v1/categories` | getCategoryList (@/store/myList.store.js:260) | brand/:id, brands-discovery, lists/brands/:id, lists/products/:id, products-discovery | IndexCategoryController |  |
| GET | `/affiliate/v1/products` | getProductList (@/store/brand.store.js:137) | active-management, brand/:id, discovery-promote-all/:brandId?, inactive-brands, inactive-products, partnership | IndexProductController |  |
| GET | `/affiliate/v1/products` | getProductList (@/store/product.store.js:123) | active-management, brand/:id, discovery-promote-all/:brandId?, inactive-products, partnerships/brands, partner | IndexProductController |  |
| GET | `/affiliate/v1/products/brands` | getAllBrandsOfTheAffiliate (@/store/brand.store.js:43) | brand/:id, discovery-my-lists, discovery-promote-all/:brandId?, inactive-brands, inactive-products, my-sellers | IndexBrandController |  |
| GET | `/affiliate/v1/products/brands` | getBrandList (@/store/brand.store.js:61) | active-management, brands-discovery, discovery-promote, lists/products/:id, reports/brand-report, reports/prod | IndexBrandController |  |
| GET | `/affiliate/v1/products/brands/drop-down` | getAllBrandsOfTheSeller (@/store/brand.store.js:26) | partnerships/brands, partnerships/products, reports/brand-report, reports/performance-report, seller/:id | BrandDropDownListController |  |
| GET | `/affiliate/v1/products/brands/{ID}/assets` | getBrandAssets (@/store/product.store.js:451) | brand/:id | ShowBrandAssetController |  |
| GET | `/affiliate/v1/products/brands/{id}` | getBrandDetail (@/store/brand.store.js:98) | brand/:id, discovery-promote-all/:brandId? | ShowBrandController |  |
| GET | `/affiliate/v1/products/drop-down` | getAllProductsOfTheAffiliate (@/store/product.store.js:89) | partnerships/products, reports/product-report | ProductDropDownListController |  |
| GET | `/affiliate/v1/products/promotions/drop-down` | getPromotionList (@/store/product.store.js:331) | active-management, product/:id | PromotionDropDownListController |  |
| GET | `/affiliate/v1/products/sellers/drop-down` | getSellerList (@/store/seller.store.js:16) | active-management, brands-discovery, discovery-my-lists, discovery-promote-all/:brandId?, inactive-brands, ina | SellerDropDownListController |  |
| GET | `/affiliate/v1/products/{ID}/promotions/drop-down` | getPromotionListForProduct (@/store/product.store.js:351) | active-management, product/:id | ProductPromotionDropDownListController |  |
| GET | `/affiliate/v1/products/{value}` | getProductDetail (@/store/product.store.js:142) | product/:id | ShowProductController |  |
| GET | `/affiliate/v1/stores` | getBrandList (@/store/brand.store.js:61) | active-management, brands-discovery, discovery-promote, lists/products/:id, reports/brand-report, reports/prod | IndexStoreController |  |
| POST | `/affiliate/v1/user-type-lists` | createMyList (@/store/myList.store.js:45) | discovery-my-lists | CreateUserTypeListController |  |
| PATCH | `/affiliate/v1/user-type-lists/assign-items` | assignItemToList (@/store/myList.store.js:140) | brand/:id, brands-discovery, discovery-promote, discovery-promote-all/:brandId?, product/:id, products-discove | AssignUserTypeListItemController |  |
| GET | `/affiliate/v1/user-type-lists/brands` | getBrands (@/store/myList.store.js:196) | discovery-my-lists | IndexBrandListsController |  |
| GET | `/affiliate/v1/user-type-lists/drop-down-brands` | getMyBrandsListDropdown (@/store/myList.store.js:100) | brands-discovery, discovery-promote | UserTypeListBrandDropDownListController |  |
| GET | `/affiliate/v1/user-type-lists/drop-down-products` | getMyProductsListDropdown (@/store/myList.store.js:120) | brand/:id, discovery-promote, discovery-promote-all/:brandId?, product/:id, products-discovery | UserTypeListProductDropDownListController |  |
| GET | `/affiliate/v1/user-type-lists/products` | getProducts (@/store/myList.store.js:212) | discovery-my-lists | IndexProductListsController |  |
| GET | `/affiliate/v1/user-type-lists/{ID}/brands` | getMyListBrands (@/store/myList.store.js:279) | lists/brands/:id | ShowUserTypeListBrandController |  |
| DELETE | `/affiliate/v1/user-type-lists/{ID}/delete-items` | removeItemFromList (@/store/myList.store.js:160) | lists/brands/:id, lists/products/:id | DeleteUserTypeListItemController |  |
| GET | `/affiliate/v1/user-type-lists/{ID}/products` | getMyListProducts (@/store/myList.store.js:231) | lists/products/:id | ShowUserTypeListProductController |  |
| DELETE | `/affiliate/v1/user-type-lists/{id}` | deleteMyList (@/store/myList.store.js:84) | discovery-my-lists | DeleteUserTypeListController |  |
| PATCH | `/affiliate/v1/user-type-lists/{id}` | editMyList (@/store/myList.store.js:67) | discovery-my-lists | UpdateUserTypeListController |  |
| POST | `affiliate/v1/pins/toggle` | togglePin (@/store/global.store.js:17) | discovery-my-lists | ToggleUserTypeListController |  |
| GET | `affiliate/v1/products/brands/{ID}/assets/{ASSET_ID}` | getBrandAssetsDetail (@/store/product.store.js:473) | brand/:id | ShowBrandAssetDetailController |  |
| GET | `affiliate/v1/products/{ID}/assets` | getProductAssets (@/store/product.store.js:390) | product/:id | ShowProductAssetController |  |
| GET | `affiliate/v1/products/{ID}/assets/{ASSET_ID}` | getProductAssetsDetail (@/store/product.store.js:412) | product/:id | ShowProductAssetDetailController |  |
| GET | `affiliate/v1/products/{id}/variations` | getProductVariation (@/store/product.store.js:311) | product/:id | IndexProductVariationController |  |
| POST | `affiliate/v1/user-type-lists/{ID}/duplicate` | duplicateList (@/store/myList.store.js:179) | discovery-my-lists | DuplicateUserTypeListController |  |

#### Other

| Method | Path | fn (file) | Pages | Backend route | Problem |
|---|---|---|---|---|---|
| DELETE | `<UNSET:VUE_APP_ALGO_PROFILE_DELETE_ALL_SESSIONS>` | terminateAll (sc/algo-setting/tabs/Security.vue:195) | settings | — | unresolved path (env var not defined in spa-affiliate/.env) |
### Live calls with no route or a wrong method
- `DELETE <VUE_APP_ALGO_PROFILE_DELETE_ALL_SESSIONS>` (Settings → Security → "Terminate all"). The env variable is missing from `spa-affiliate/.env`, so axios sends the request to the base URL root. The backend route `DELETE affiliate/v1/profile/sessions/all` exists, so only the env value is missing.
- Two calls match only through a literal segment, which is expected:
  - `GET affiliate/v2/social-medias/oauth/{SOCIAL_MEDIA_NAME}` matches the five literal routes for youtube, tiktok, instagram, pinterest and facebook.
  - `GET /{PANEL}/v1/auth/local-login` exists for the seller, affiliate and buyer panels.

### Calls that would fail but are not reachable from the affiliate UI
These sit behind seller-only UI or are never called. The JSON marks them `live:false` with a reason.
- **Team management** (`settings.store.js`): `GET/PUT/DELETE /affiliate/v1/teams/members/{ID}` and `POST teams/members/cancel/{ID}` have no matching route; the backend uses `teams/{teamId}/members/{list|update|delete|cancel}/{memberId}`. Role and invitation env variables are missing. The tab is hidden for affiliates.
- **Shopify/Amazon seller-account sync:** `POST affiliate/v1/profile/sync/account/shopify` and `DELETE affiliate/v1/profile/synced-shopify/{id}` have no route. The Amazon/platform env variables are missing. The block is seller-only UI.
- **Product status and commission edit** (`partnership-details.component.vue`): `PUT <unset VUE_APP_ALGO_PRODUCT_CHANGE_STATUS_API>` and `PUT /affiliate/v1/products/{id}/commission` have no route. This is copied seller code whose handler is never bound.
- **Other dead store functions with route problems:**
  - Method mismatch, the backend allows GET only: `DELETE products/{id}/assets`, `DELETE products/brands/{id}/assets`, `PATCH products/brands/{id}`.
  - Method mismatch, the backend allows POST only: `GET user-type-lists`.
  - No route at all: `GET payments/transactions`, `GET dashboard/report/brands/chart`, `GET associates/associate-sale-reports/show/{id}`, `PUT products/brands/update/amazon-link`.

## 5. Dead code

### 5.1 Files in `spa-affiliate/src` never imported from `main.js` (44 of 331)

The import walk resolves `@/`, `shared-components/` and the workspace path `spa-affiliate/`. It follows lazy `import()` calls in the router, and it ignores commented-out code.

- `spa-affiliate/src/components/_utils/algorift-notice/algorift-notice.component.vue`
- `spa-affiliate/src/components/_utils/algorift-notice/algorift-notice.i18n.js`
- `spa-affiliate/src/components/associate/explore/filters/associate-region.vue`
- `spa-affiliate/src/components/associate/explore/filters/sort-box.vue`
- `spa-affiliate/src/components/auth/auth-register-steps-platforms.component.vue`
- `spa-affiliate/src/components/design-studio/components/product-link/double-search-panel.utility.vue`
- `spa-affiliate/src/components/design-studio/components/product-link/dump/double-search-panel.utility copy.vue`
- `spa-affiliate/src/components/design-studio/create-update-assignee/create-update-assignee.modal.vue`
- `spa-affiliate/src/components/design-studio/create-update-assignee/create-update-assignee.section.vue`
- `spa-affiliate/src/components/design-studio/create-update-collection/create-update-collection.modal.vue`
- `spa-affiliate/src/components/design-studio/create-update-collection/create-update-collection.section.vue`
- `spa-affiliate/src/components/design-studio/create-update-landing-page/select-product-modal.vue`
- `spa-affiliate/src/components/design-studio/utility/checkbox-item.skeleton.vue`
- `spa-affiliate/src/components/design-studio/utility/checkbox-item.utility.vue`
- `spa-affiliate/src/components/design-studio/utility/checkbox-status.class.js`
- `spa-affiliate/src/components/design-studio/utility/image-name-description.utility.vue`
- `spa-affiliate/src/components/design-studio/utility/search-panel.utility.vue`
- `spa-affiliate/src/components/design-studio/utility/theme-select-input.utility.vue`
- `spa-affiliate/src/components/discovery/brands/utility/brand.helper.js`
- `spa-affiliate/src/components/discovery/products/brand-id/product-brand-id-grid.vue`
- `spa-affiliate/src/components/discovery/products/brand-id/product-brand-id-list.vue`
- `spa-affiliate/src/components/discovery/products/brand-id/product-brand-id.js`
- `spa-affiliate/src/components/discovery/products/brand-id/product-brand-id.page.vue`
- `spa-affiliate/src/components/discovery/products/product-details/sections/link-list/link-list.controller.js`
- `spa-affiliate/src/components/discovery/products/product-details/sections/link-list/links-list.section.vue`
- `spa-affiliate/src/components/discovery/products/product-details/sections/product-description.section.vue`
- `spa-affiliate/src/components/discovery/products/product-details/sections/product-details.section.vue`
- `spa-affiliate/src/components/discovery/products/product-details/sections/product-info.section.vue`
- `spa-affiliate/src/components/discovery/products/product-details/sections/product-rating.section.vue`
- `spa-affiliate/src/components/discovery/products/product-details/sections/promotion-list/promotion-list.controller.js`
- `spa-affiliate/src/components/discovery/products/product-details/sections/promotion-list/promotion-list.section.vue`
- `spa-affiliate/src/components/discovery/products/product-details/sections/terms-of-sale/terms-of-sale.section.vue`
- `spa-affiliate/src/components/discovery/products/product-details/utility/info-panel.utility.vue`
- `spa-affiliate/src/components/discovery/products/product-list/bulk-actions/product-bulk-actions.vue`
- `spa-affiliate/src/components/discovery/products/product-list/product-list.controller.js`
- `spa-affiliate/src/components/discovery/products/product-list/product-list.page.vue`
- `spa-affiliate/src/components/discovery/products/product-list/product-list.table.vue`
- `spa-affiliate/src/components/discovery/utility/user-pill.utility.vue`
- `spa-affiliate/src/components/partnership/products/contracts/partnership-product-contracts-table.vue`
- `spa-affiliate/src/components/partnership/utils/partnership-contracts-table-tab.component.vue`
- `spa-affiliate/src/components/report/utility/bar-chart.panel.vue`
- `spa-affiliate/src/components/report/utility/line-chart.panel.vue`
- `spa-affiliate/src/mixins/reveal-info.mixin.js`
- `spa-affiliate/src/utils/request-notification.js`

Notable groups among these files:
- The old product list (`discovery/products/product-list/*`) and `discovery/products/brand-id/*`. Both were replaced by `products/merge/*`.
- The old product-details `sections/*` (description, info, rating, links-list, promotion-list, terms-of-sale section).
- The design-studio assignee and collection editors (`create-update-assignee/*`, `create-update-collection/*`) and the design-studio utility components.
- `auth/auth-register-steps-platforms.component.vue`, the v1 platforms step. `-v2` is the live one.
- `mixins/reveal-info.mixin.js`, `utils/request-notification.js`, `_utils/algorift-notice/*`, `report/utility/{bar,line}-chart.panel.vue`, and a duplicate copy `product-link/dump/double-search-panel.utility copy.vue`.

### 5.2 Shared files that are reachable only in theory
`shared-components/router/auth.router.js` has default-parameter imports. The affiliate router always passes its own components, so the defaults are bundled as lazy chunks but never rendered: `algo-views/app.layout.vue`, which pulls in `widget-user-button/{index,panel-selector}.vue`, `algo-notification/{WidgetNotification,notification-icon}.vue` and `algo-auth/auth-register-steps/{auth-register-steps,-done,-questions}.vue`. Also dead: the commented-out messenger route `algo-messenger/main-layout.vue`. From `algo-messenger/messenger.store.js`, only `onSendMessageToUser` is used.

### 5.3 Store functions never invoked, or invoked only by hidden UI (86 call records)
| Method | Path | fn | File | Why dead | Route check |
|---|---|---|---|---|---|
| GET | `/affiliate/v1/chats/contacts/{userId}` | showContact | sc/algo-chat/store/chat.store.js:208 | never referenced by reachable code | ok |
| DELETE | `/affiliate/v1/chats/messages/{messageId}` | deleteMessage | sc/algo-chat/store/chat.store.js:317 | never referenced by reachable code | ok |
| POST | `<UNSET:VUE_APP_ALGO_MESSENGER_SEND_BULK>` | sendBulkMessage | sc/algo-chat/store/chat.store.js:390 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| POST | `<UNSET:VUE_APP_ALGO_MESSENGER_SEND_LIST>` | sendBulkMessageToList | sc/algo-chat/store/chat.store.js:405 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| GET | `/affiliate/v1/chats/contacts` | searchContactList | sc/algo-messenger/messenger.store.js:102 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/chats/contacts/{PARAM1}/messages` | getMessageHistory | sc/algo-messenger/messenger.store.js:126 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/chats/contacts/{id}` | searchMessageHistory | sc/algo-messenger/messenger.store.js:144 | never referenced by reachable code | ok |
| POST | `/affiliate/v1/chats/messages` | sendMessage | sc/algo-messenger/messenger.store.js:163 | never referenced by reachable code | ok |
| POST | `/affiliate/v1/chats/messages/saved` | saveMessage | sc/algo-messenger/messenger.store.js:180 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/chats/messages` | generalSearch | sc/algo-messenger/messenger.store.js:200 | never referenced by reachable code | ok |
| POST | `/affiliate/v1/chats/messages/find-page` | findPageOfMessage | sc/algo-messenger/messenger.store.js:216 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/chats/contacts/{id}` | getContactshow | sc/algo-messenger/messenger.store.js:235 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/chats/messages/saved` | getSaveMessages | sc/algo-messenger/messenger.store.js:253 | never referenced by reachable code | ok |
| DELETE | `/affiliate/v1/chats/messages/{id}` | deleteMessage | sc/algo-messenger/messenger.store.js:270 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/chats/contacts` | getContactList | sc/algo-messenger/messenger.store.js:84 | never referenced by reachable code | ok |
| POST | `/affiliate/v1/auth/verification-code` | resendEmailVerifycode | sc/store/authentication.store.js:114 | never referenced by reachable code | ok |
| POST | `/affiliate/v1/auth/mobile-verification-code` | verifyMobileNumber | sc/store/authentication.store.js:137 | never referenced by reachable code | ok |
| PUT | `/affiliate/v1/profile/change-mobile` | verifyOTPChangePhone | sc/store/authentication.store.js:159 | never referenced by reachable code | ok |
| GET | `<UNSET:VUE_APP_ALGO_AUTH_BECOME_AFFILIATE>` | sellerBecomesAffiliate | sc/store/authentication.store.js:225 | UI-dead: Panel switcher: in affiliate panel the Affiliate entry is the active panel and goToOtherPanel() returns early; only meaningful in seller app | unresolved path (env var not defined in spa-affiliate/.env) |
| POST | `/affiliate/v1/auth/email-verify` | verifyEmail | sc/store/authentication.store.js:96 | never referenced by reachable code | ok |
| POST | `<UNSET:VUE_APP_IMPERSONATE_USER>` | impersonateUser | sc/store/authorization.store.js:243 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| POST | `/affiliate/v1/profile/request-change-password` | requestChangePassword | sc/store/profile.store.js:150 | never referenced by reachable code | ok |
| PUT | `/affiliate/v1/profile/change-password` | changePassword | sc/store/profile.store.js:162 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/amazon-marketplaces/list` | getAmazonMarketplaceList | sc/store/profile.store.js:196 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/amazon-marketplaces/show/{ID}` | getAmazonMarketplaceDetails | sc/store/profile.store.js:213 | never referenced by reachable code | ok |
| GET | `<UNSET:VUE_APP_SELLER_PLATFORM>` | getPlatformAccounts | sc/store/profile.store.js:33 | UI-dead: Seller-only UI (component branch v-if="AppInfo.isSeller"); not rendered in affiliate | unresolved path (env var not defined in spa-affiliate/.env) |
| POST | `<UNSET:VUE_APP_ALGO_SYNC_AMAZON_PROFILE_API>` | syncAmazonProfile | sc/store/profile.store.js:79 | UI-dead: Seller-only UI (component branch v-if="AppInfo.isSeller"); not rendered in affiliate | unresolved path (env var not defined in spa-affiliate/.env) |
| POST | `<UNSET:VUE_APP_ALGO_SYNC_PRODUCT_WITH_AMAZON_API>` | syncProductWithAmazon | sc/store/profile.store.js:96 | UI-dead: Seller-only UI (component branch v-if="AppInfo.isSeller"); not rendered in affiliate | unresolved path (env var not defined in spa-affiliate/.env) |
| GET | `/affiliate/v1/teams/my` | getMyTeam | sc/store/settings.store.js:138 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | ok |
| GET | `/affiliate/v1/teams/members/{ID}` | getTeamMembers | sc/store/settings.store.js:155 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | weak match: only resolves if path param equals literal segment(s): impersonate |
| POST | `/affiliate/v1/teams/{ID}/invitation` | sendInvitation | sc/store/settings.store.js:171 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | ok |
| PUT | `/affiliate/v1/teams/members/{ID}` | updateRole | sc/store/settings.store.js:188 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | wrong method (route allows GET,POST) |
| POST | `/affiliate/v1/teams/members/cancel/{ID}` | cancelInvitation | sc/store/settings.store.js:205 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | no backend route |
| DELETE | `/affiliate/v1/teams/members/{ID}` | removeTeamMember | sc/store/settings.store.js:223 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | wrong method (route allows GET,POST) |
| GET | `<UNSET:VUE_APP_MY_TEAMS_INVITATION_LIST>` | getInvitedTeamRequests | sc/store/settings.store.js:253 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | unresolved path (env var not defined in spa-affiliate/.env) |
| PATCH | `<UNSET:VUE_APP_MY_TEAMS_MEMBER_CHANGE_STATUS>` | changeTeamInvitationStatus | sc/store/settings.store.js:269 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | unresolved path (env var not defined in spa-affiliate/.env) |
| GET | `<UNSET:VUE_APP_MY_TEAMS_ROLE_LIST>` | getTeamRoles | sc/store/settings.store.js:286 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | unresolved path (env var not defined in spa-affiliate/.env) |
| GET | `<UNSET:VUE_APP_MY_TEAMS_ROLE_DETAIL>` | getTeamRoleDetail | sc/store/settings.store.js:303 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| POST | `<UNSET:VUE_APP_MY_TEAMS_ROLE_CREATE>` | createTeamRole | sc/store/settings.store.js:319 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| PUT | `<UNSET:VUE_APP_MY_TEAMS_ROLE_UPDATE>` | updateTeamRole | sc/store/settings.store.js:337 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| DELETE | `<UNSET:VUE_APP_MY_TEAMS_ROLE_DELETE>` | deleteTeamRole | sc/store/settings.store.js:355 | UI-dead: Team tab hidden in affiliate: algo-setting.page.vue Team tab canView requires AppInfo.isSeller | unresolved path (env var not defined in spa-affiliate/.env) |
| POST | `<UNSET:VUE_APP_MY_TEAMS_ROLE_ASSIGN>` | assignTeamRole | sc/store/settings.store.js:372 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| GET | `<UNSET:VUE_APP_MY_TEAMS_PERMISSION_LIST>` | getTeamPermissions | sc/store/settings.store.js:393 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| GET | `/affiliate/v1/message-templates/detail/{ID}` | getSavedMessage | sc/store/settings.store.js:424 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/message-variables/detail/{ID}` | getMessageVariable | sc/store/settings.store.js:501 | never referenced by reachable code | ok |
| DELETE | `<UNSET:VUE_APP_SOCIAL_SYNCED_AMAZON>/{id}` | deleteAmazonAccount | sc/store/social-service.store.js:242 | UI-dead: Seller-only UI (component branch v-if="AppInfo.isSeller"); not rendered in affiliate | unresolved path (env var not defined in spa-affiliate/.env) |
| DELETE | `/affiliate/v1/profile/synced-shopify/{id}` | deleteShopifyAccount | sc/store/social-service.store.js:255 | UI-dead: Seller-only UI (component branch v-if="AppInfo.isSeller"); not rendered in affiliate | no backend route |
| POST | `/affiliate/v1/profile/sync/account/amazon` | syncAmazonAccount | sc/store/social-service.store.js:397 | UI-dead: Seller-only UI (component branch v-if="AppInfo.isSeller"); not rendered in affiliate | ok |
| POST | `/affiliate/v1/profile/sync/account/shopify` | syncShopifyAccount | sc/store/social-service.store.js:397 | UI-dead: Seller-only UI (component branch v-if="AppInfo.isSeller"); not rendered in affiliate | no backend route |
| GET | `/affiliate/v1/social-medias/type-categories` | getSelectOptionListForSocialCategory | @/store/affiliate.store.js:52 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/associates/associate-sale-reports/show/{ID}` | getAssociateSaleReportDetails | @/store/associate.store.js:100 | never referenced by reachable code | no backend route |
| GET | `/affiliate/v1/associates/associate-social-medias/show/{ID}` | getAssociateSocialMediaDetails | @/store/associate.store.js:162 | never referenced by reachable code | ok |
| PUT | `/affiliate/v1/associates/associate-social-medias/update/{ID}` | updateAssociateSocialMedia | @/store/associate.store.js:193 | never referenced by reachable code | ok |
| DELETE | `/affiliate/v1/associates/associate-social-medias/delete/{ID}` | deleteAssociateSocialMedia | @/store/associate.store.js:211 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/associates/associate-brands/show/{ID}` | getAssociateBrandDetails | @/store/associate.store.js:241 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/payments/subscription-transactions` | getSubscriptionTransactions | @/store/billing.store.js:125 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/payments/transactions` | getTransationsList | @/store/billing.store.js:33 | never referenced by reachable code | no backend route |
| POST | `<UNSET:VUE_APP_ALGO_BRAND_UPDATE_TERMS_FILE_API>` | updateTermFile | @/store/brand.store.js:119 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| PUT | `<UNSET:VUE_APP_ALGO_BRAND_CHANGE_STATUS_API>` | updateBrandStatus | @/store/brand.store.js:160 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| PUT | `/affiliate/v1/products/brands/update/amazon-link` | updateBrandAmazonLink | @/store/brand.store.js:80 | never referenced by reachable code | no backend route |
| GET | `/affiliate/v1/reports/performance/chart` | getLinearChart | @/store/dashboard.store.js:113 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/dashboard/report/brands/chart` | getBrandChart | @/store/dashboard.store.js:99 | never referenced by reachable code | no backend route |
| DELETE | `/affiliate/v1/landing-pages/{ID}/collections/{collectionID}` | deleteCollection | @/store/landpage.store.js:121 | never referenced by reachable code | ok |
| PUT | `/affiliate/v1/landing-pages/{ID}/collections/{collectionID}` | updateCollection | @/store/landpage.store.js:142 | never referenced by reachable code | ok |
| POST | `/affiliate/v1/landing-pages/{ID}/collections` | createCollection | @/store/landpage.store.js:163 | never referenced by reachable code | ok |
| DELETE | `affiliate/v1/landing-pages/{ID}/assignees/{assigneeID}` | deleteLandpageAssignee | @/store/landpage.store.js:185 | never referenced by reachable code | ok |
| POST | `affiliate/v1/landing-pages/{ID}/assignees` | createLandpageAssignee | @/store/landpage.store.js:206 | never referenced by reachable code | ok |
| PUT | `/affiliate/v1/landing-pages/{id}` | updateLandingpage | @/store/landpage.store.js:255 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/landing-pages/templates` | getSelectedLandpageTemplates | @/store/landpage.store.js:290 | never referenced by reachable code | ok |
| POST | `/affiliate/v1/landing-pages` | createLandpage | @/store/landpage.store.js:321 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/products/links` | getPartnershipLinks | @/store/links.store.js:102 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/products/links/{id}` | showProductLink | @/store/links.store.js:71 | never referenced by reachable code | ok |
| DELETE | `/affiliate/v1/products/links/{id}` | deleteLink | @/store/links.store.js:88 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/user-type-lists` | getAllMyListOfTheAffiliate | @/store/myList.store.js:29 | never referenced by reachable code | wrong method (route allows POST) |
| GET | `<UNSET:VUE_APP_ALGO_ALL_SELLER_PRODUCT_LIST>` | getAllProductsOfTheSeller | @/store/product.store.js:105 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| GET | `/affiliate/v1/products/links` | getLinkList | @/store/product.store.js:176 | never referenced by reachable code | ok |
| GET | `/affiliate/v1/products` | searchProductList | @/store/product.store.js:208 | never referenced by reachable code | ok |
| PATCH | `/affiliate/v1/products/brands/{id}` | updateBrandDetail | @/store/product.store.js:225 | never referenced by reachable code | wrong method (route allows GET) |
| PUT | `<UNSET:VUE_APP_ALGO_PRODUCT_CHANGE_STATUS_API>` | updateProductStatus | @/store/product.store.js:246 | UI-dead: Only called from partnership-details.component.vue updateData()/updateProductStatus(), which no template element triggers (copied seller edit code) | unresolved path (env var not defined in spa-affiliate/.env) |
| PUT | `/affiliate/v1/products/{id}/commission` | updateProductCommission | @/store/product.store.js:260 | UI-dead: Only called from partnership-details.component.vue updateData()/updateProductStatus(), which no template element triggers (copied seller edit code) | weak match: only resolves if path param equals literal segment(s): links |
| GET | `<UNSET:VUE_APP_ALGO_PRODUCT_LINK_LIST_API>` | getProductLinkList | @/store/product.store.js:279 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| DELETE | `affiliate/v1/products/{ID}/assets` | deleteProductAsset | @/store/product.store.js:431 | never referenced by reachable code | weak match: only resolves if path param equals literal segment(s): links |
| DELETE | `/affiliate/v1/products/brands/{ID}/assets` | deleteBrandAsset | @/store/product.store.js:492 | never referenced by reachable code | wrong method (route allows GET) |
| GET | `<UNSET:VUE_APP_AFFILIATE_ACTIVE_PARTNERSHIPS_PRODUCTS_LIST_API>` | getSellerActiveProductPartnershipList | @/store/seller.store.js:34 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| GET | `<UNSET:VUE_APP_AFFILIATE_ACTIVE_PARTNERSHIPS_BRANDS_LIST_API>` | getSellerActiveBrandPartnershipList | @/store/seller.store.js:51 | never referenced by reachable code | unresolved path (env var not defined in spa-affiliate/.env) |
| POST | `/affiliate/v1/partnerships/brand/request` | sendPartnershipBrandRequestToSeller | @/store/seller.store.js:69 | never referenced by reachable code | ok |

### 5.4 Other dead or orphaned UI
- `/app/discovery-promote` (One Click Promote): its sidebar item is commented out and no component links to it. It is reachable only by typing the URL. `/app/discovery-promote-all` is still linked from cards.
- `/app/partnerships`, `/app/reports` and `/app/payment-history` are tab shells with child routes. The nav always links to a child route.
- The "Auto System-Generated Partnership Requests" switch is never persisted.
- `showYoutubeModal()` on the dashboard is commented out, but login still sets `youtube_modal_show`.
- `profile-widget-items.js` links to route name `overview`, which does not exist. It is used only by the unused shared `app.layout.vue`.
- The shared seller register-done step pushes `/app/discovery-affiliates?grouped=true`, a seller route. It is not used in the affiliate app.
- App.vue `beforeEach` calls `next()` only when `to.meta.progress` is defined. Every route defines it today, but a new route without `meta.progress` would hang navigation.
- `getAuthenticationRoute` checks route name `impersonate`, but the affiliate router has no such route. Impersonation is handled by the pre-router script instead.

## 6. Uncertainties
- **Production API base URL:** `VUE_APP_API_BASE_URL` exists only in `.env.development` (`https://backstage.ainfluencer.com`). The production build must get it from the Jenkins agent environment or an untracked `.env.production(.local)`. This could not be verified.
- **Caller resolution:** live/dead is decided at function level. A store function counts as live if a reachable file imports that store and references `.fn` or `"fn"`. UI hidden by runtime conditions was checked by hand only where it mattered (team tab, seller-only blocks, product edit, panel switcher), so other functions may be "live" in the code but sit behind permission (`$can`) or role conditions.
- **`routes` per call:** computed from the import closure of each route component. Shared mixins (`use-message.js`) and global stores make some lists broad, for example "Message seller" appears on many pages.
- **SSO:** the `issuer` values and source handlers (`SsoSourceEnum`) were not traced. Which core Ainfluencer screens link to `web/v1/sso/login` is outside this repo.
- **Backend sanitization** of product descriptions and notification content was not checked; see the security notes.
- **Team endpoints:** the backend team routes carry `impersonation_permission:seller.setting.team.*` even under `affiliate/v1`. Whether affiliates own teams at all is a product question.