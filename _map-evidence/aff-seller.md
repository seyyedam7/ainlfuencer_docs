# Ainfluencer Affiliate: Seller (brand/merchant) SPA, live-code inventory

**Scope:** production tag **v1.4.02** of `aff-frontend/spa-seller` plus the `aff-frontend/shared-components` it imports. Live URL: https://seller.ainfluencer.com.
**Backend reference:** affiliate backend route dump for v1.5.27 (`aff_routes.json`, 895 routes, 280 under `seller/v1`).
**Method:** static analysis only. Nothing was run in a browser.

Companion file: `aff-seller-calls.json`, with one row per backend call site and its live flag, routes, matched backend route and problems.

Headline numbers:

- **Files:** 838 JS/Vue files in scope (336 seller, 502 shared). **695 are reachable** from `main.js`: 278 seller, 416 shared, and 1 file pulled from spa-affiliate.
- **Routes:** 56 router records in total.
  - `/app` (layout) with **46 children**, including the dashboard, the logout pseudo-route and 3 parent routes (partnerships, reports, payment-history) with their 10 sub-routes.
  - `/user` (auth layout) with **6 children**.
  - `/` and the `*` catch-all.
  - No routes are commented out in `router/index.js`.
- **Backend calls:** 282 call sites extracted, **225 live**, covering 218 distinct method+path pairs.
  - **4 live calls have no matching backend route.** These are confirmed user-facing failures (see §4.2).
  - 57 call sites are dead: the function is never called, the file is unreachable, or the call is role-gated to Affiliate.

---

## 1. How the app is built

| Aspect | Detail |
|---|---|
| Stack | Vue **2.7** + vue-router **3** (history mode) + **Pinia** stores + BootstrapVue, built with Vue CLI 5 (`--modern`) plus the PWA plugin. The page title is "Ainfluencer \| Seller" and the PWA `start_url` is `/user/login`. |
| Monorepo | `aff-frontend/package.json` npm workspaces: `shared-components`, `spa-seller`, `spa-affiliate`, `spa-buyer`, `spa-admin`, `spa-landpage`. Imports `shared-components/...` resolve to the workspace package. `@/` → `spa-seller/src` (`jsconfig.json`). Shared code also imports `@/store/*.js`, which resolves to the **seller** stores in this build. One shared chat file imports `../../../spa-affiliate/src/store/seller.store`, so an affiliate store is bundled. |
| Entry | `src/main.js`. It first runs `shared-components/algo-auth/script/auth-init.script.js` (pre-router handling of `/auth` and `/impersonate`), then initializes Sentry (tracing 10%, replay 10%), i18n, Pinia, the permission mixin and directive (`$can`), and registers `utils/service-worker` (PWA + Firebase messaging SW). |
| Router | `src/router/index.js` builds routes with helpers from `shared-components/router/auth.router.js` (`getLayoutViewRoute`, `getAuthenticationRoute`, `getLoginRoute`, and so on). All dashboard pages are children of `/app` and are lazy-loaded (`() => import()`). |
| Guards | (a) `/app` `beforeEnter`: `GET seller/v1/auth/auth-status`. If not logged in → `/user/login`. If logged in but `register_completed=false` → `/user/register-steps`. Otherwise it loads `system/basic-info` and `settings/client`, and on the `dashboard` route consumes the stored SSO redirect path. (b) `/user` `beforeEnter`: a logged-in user is sent to the dashboard, except for `email-verified`, or `register-steps` while registration is incomplete. (c) Global `beforeEach`: if `meta.permission` is set, checks the team permission with `usePermissionsStore().hasPermission`; on failure it **silently redirects to `/app?redirect=…`**. Account owners (empty `member` array) pass every check. (d) `App.vue` also registers a `beforeEach` that only calls `next()` when `meta.progress` exists. Every route has it, but a route added without `routerOptions` would hang navigation. |
| HTTP | Each Pinia store creates its own `axios.create({ baseURL: VUE_APP_API_BASE_URL })` (`https://backstage.ainfluencer.com`) from `shared-components/algo-axios` (axios + an unused MockAdapter in passthrough mode). Endpoint paths come from `spa-seller/.env` (`VUE_APP_*`). Requests send `Authorization: Bearer <localStorage.auth_token>` and **`X-Platform: <selected store account>`**. The selected account comes from the header store switcher, is persisted in `localStorage.platform`, and drives Amazon vs Shopify wording and endpoints. |
| Errors / limits | `AuthorizationStore.handleHttpErrors`: **HTTP 402** opens the "Upgrade" limit modal; the modal's title and text are chosen from the `SellerFeatureNameEnum` type (see §3.15). **HTTP 499** hard-redirects to `/app/settings?tab=billing`. Other errors are rethrown as `AlgoError` (422 = field errors). |
| Realtime / push | Chat uses Pusher (`pusher-js`, key from env, cluster us3) with presence-channel auth at `POST /broadcasting/auth`. Push notifications use Firebase Messaging: after permission is granted, `POST seller/v1/user-fcm-tokens` (or `token=null` when denied). |
| Uploads | Brand and product assets upload in chunks through **resumablejs** to `POST seller/v1/products[/brands]/{id}/upload-assets`. Brand terms, avatars and ticket attachments use multipart axios posts. |
| Other external | Google reCAPTCHA v2 on login and register. Stripe (portal, checkout and wallet deposit) through backend-generated URLs. OAuth popups for Google, Facebook and Amazon, and a full-page redirect for Shopify, using `redirect_link` URLs returned by the backend. Outbound links go to the core app (`app.ainfluencer.com/search`, `/campaigns`). |
| Build / env | Jenkins builds tags with `npm run build` (production mode), which loads `.env` (+`.env.production`, not present). `.env.development` is **not** loaded in that build. Both `.env` and `.env.development` are **git-ignored**; the copies in the snapshot were used for path resolution here (see §6). |

---

## 2. Live routes

Columns: path, component, guard, inbound links, purpose. For the guard, "auth" means the `/app` `beforeEnter` described in §1, and `perm X:Y` means the global permission check. For inbound links, "sidebar", "footer" (mobile bottom bar) and "header" are links in the layout; anything else names the component that links to the route.

### 2.1 Entry, auth and onboarding (`/user`, auth layout `shared-components/algo-views/auth.layout.vue`)

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/` | redirect → `/app` | none | – | root |
| `*` | redirect → `/` | none | – | unknown URLs end at the dashboard or login |
| `/user` | auth.layout (redirects to `user/login`) | `/user` guard | – | auth shell |
| `/user/login` | `algo-auth/auth-login.vue` | `/user` guard | register/forgot/reset pages, logout | email+password+reCAPTCHA login, Google or Facebook popup login |
| `/user/register` | `algo-auth/auth-register/auth-register.vue` | `/user` guard | login page ("By creating an account…") | sign-up form, then "Check your Email" |
| `/user/forgot-password` | `algo-auth/auth-forgot-password.vue` | `/user` guard | login, reset page | request a reset email |
| `/user/reset-password/:token` | `algo-auth/auth-reset-password.vue` (route name `seller-reset-form`) | `/user` guard | **email link only** | set a new password |
| `/user/register-steps` | `algo-auth/auth-register-steps/auth-register-steps.vue` | `/user` guard + forced by the `/app` guard | **redirect only** (from the `/app` guard) | post-signup onboarding wizard |
| `/user/email-verified/:token` | `algo-auth/auth-email-verified.vue` | `/user` guard (allowed while logged in) | **email or SSO link only** | stores the token (and optional `?redirect=`), then goes to `/user` |
| `/auth?...`, `/impersonate?token=` | not routes; handled in `auth-init.script.js` before Vue mounts | – | OAuth callback, panel switch, admin impersonation | see §3.2 |

### 2.2 App shell and dashboard (`/app`, layout `spa-seller/src/views/seller-app.layout.vue`)

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/app` (name `dashboard`) | `components/dashboard/seller-dashboard.page.vue` | auth | sidebar, footer, product-list empty state, login/SSO landing | KPIs, recent partnerships, onboarding cards |
| `/app/logout` | no component; `beforeEnter` → `AuthorizationStore.logout()` | auth | user menu "Sign out", register-steps | clears localStorage, reloads, broadcasts logout to other tabs |
| `/app/profile` | `shared…/algo-components/profile/profile-tabs.page.vue` | auth | user menu "Edit profile", dashboard "Complete Profile", My Brands "Add Shopify/Amazon Store", header "Add New Store" (`?tab=manage_accounts`) | tabs: overview, edit_profile_info, get_better_profile, manage_accounts (store connections) |
| `/app/settings` | `shared…/algo-setting/algo-setting.page.vue` | auth (each tab checks its own permission) | user menu "Account settings", dashboard "Connect" (`?tab=billing`), payments "Manage Credit Card", role page back link, **HTTP 499 redirect** | tabs: team, messageTemplates, billing, notifications, security |
| `/app/manage-subscription` | `shared…/algo-manage-subscription/manage-subscription.page.vue` | auth | sidebar footer widget, limit modal "Upgrade Plan Now", profile overview, bulk-invite modal | plans, upgrade, downgrade, cancel, resume |

### 2.3 Affiliate discovery, expanded network, lists

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/app/discovery-affiliates` | `discovery/affiliates/affiliate-list.page.vue`. With `?grouped=true` it shows the landing view `landing/grouped-affiliate-list.page.vue` | auth (sidebar entry needs `affiliates.all_affiliates:view`; the route itself has no meta) | sidebar "Affiliates & Influencers → Affiliate Discovery" (`grouped=true`), footer, dashboard "See all Affiliates", list pages, post-login redirect target | find affiliates; invite, message, reveal email, add to list |
| `/app/affiliate/:id` | `discovery/affiliates/affiliate-details.page.vue` | auth | affiliate cards, partnership tables and modals, chat contact info, promotion details | affiliate profile, contracts, related affiliates |
| `/app/expanded-network` | `discovery/unlisted-affiliates/unlisted-affiliates.page.vue` | auth (the button needs `affiliates.unlisted_affiliates:view`) | **only** the grouped-landing "Invite via Email Campaigns / Expanded Network" button | scraped, non-registered creators: reveal emails, add to lists |
| `/app/discovery-lists` | `discovery/lists/lists.page.vue` | auth (sidebar needs `lists.affiliate_list` or `lists.unlisted_affiliate_list` view) | sidebar "More → My Lists", promote modal, email campaigns, list detail back links | "My Lists" (affiliate lists only; the Expanded Network tab is commented out) |
| `/app/lists/affiliates/:id` | `discovery/lists/affiliates/list-details.page.vue` | auth | list card, promote modal "View List" | list members; remove, reveal, message, invite |
| `/app/lists/unlisted-affiliates/:id` | `discovery/lists/unlisted-affiliates/list-details.page.vue` | auth | **only** the "View List" link in the expanded-network add-to-list modals | expanded-network list detail |

### 2.4 Partnerships

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/app/partnerships` | `partnership/partnership-tabs.page.vue` (parent only, no default child) | perm `partnership.partnership:view` | none directly | tab shell (Products / Brands switch; tabs Applications · Partnered · Invitations · Revoked · Rejected via `?tab=requests\|active\|invitations\|revoked\|rejected`) |
| `/app/partnerships/partnership-products` | `partnership/products/product-partnership.page.vue` | perm `partnership.partnership:view` | sidebar "More → Partnership" (`?tab=requests`, badge = new requests count), product list, affiliate details | product-level requests and contracts |
| `/app/partnerships/partnership-brands` | `partnership/brands/brand-partnership.page.vue` | perm `partnership.partnership:view` | dashboard "See Partnerships" / "See Contracts" (`?tab=active`), brand cards, affiliate details | brand (or store) level requests and contracts |
| `/app/my-affiliates` | `partnership/my-affiliates/my-affiliates.table.vue` | auth (**no permission meta**) | partnership tabs "My affiliate list", product "View affiliates" | list of partnered affiliates with sales stats |

### 2.5 Products, brands/stores, promotions

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/app/discovery-products` | `discovery/products/product-list.page.vue` → `product-list.table.vue` | auth (sidebar needs `products.all_products:view`) | sidebar "My Products", footer, dashboard | product catalogue (synced from Amazon or Shopify): status, commission, promote |
| `/app/product/:id` | `discovery/products/product-details/product-details.page.vue` | auth | many tables and cards | product detail: activate, commission, variations, assets |
| `/app/discovery-brands` | `discovery/brands/brand-list.page.vue` | auth | sidebar "My Brands"/"My Stores" (label follows the platform), footer, dashboard | brand or store list |
| `/app/brand/:id` | `discovery/brands/brand-details/brand-details.page.vue` | perm `brands.custom_brands:view` with **custom id = route `:id`** | brand cards, partnership tables, product page | brand detail: activation, auto-approval, landing page, social links, terms, Amazon link, assets |
| `/app/promotion-list` | `promotions/promotion-list/promotion-list.page.vue` | perm `promotions.all_promotions:view` | sidebar "More → My Promotions", dashboard | promotion (claim-code coupon) list |
| `/app/new-promotion` | `promotions/create-update-promotion/new-promotion.wrapper.vue` → `promotion-form.page.vue` | auth | "Add New Promotion" buttons (need `promotions…:create`) | 3-step create wizard |
| `/app/promotion-details/:id` | `promotions/promotion-details/promotion-detail.page.vue` | auth | promotion list items, edit pages | details, delete, edit entry points |
| `/app/promotion-edit/:id` | `promotion-edit.page.vue` | auth | promotion details "Edit Promotion" | edit duration and buyer visibility |
| `/app/promotion-edit-duration/:id` | `promotion-edit-duration.page.vue` | auth | promotion details "Edit Duration" | change from/to dates |
| `/app/promotion-edit-assigner/:id` | `promotion-edit-assigner.page.vue` | auth | promotion details "Edit Assigned" | private promotion: choose partnered affiliates and products |
| `/app/update-promotion/:id` | `update-promotion.wrapper.vue` → `promotion-form.page.vue` (edit mode) | auth | **no inbound link** (URL only) | legacy full-form edit |

### 2.6 Reports

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/app/reports` | `report/report-tabs.page.vue` (parent only; "Export Data As CSV" needs `reports.reports:export`) | auth (sidebar needs `reports.reports:view`; **no route meta**) | none directly | tab shell |
| `/app/reports/performance-report` | `report/performance/performance-report.page.vue` | auth | sidebar "Reports", footer, payments "Sale Report" | daily performance chart and table |
| `/app/reports/brands` (name `brand-report`) | `report/brand/brand-report.page.vue` | auth | report tabs, brand header | per-brand stats, compare |
| `/app/reports/product-report` (tab label "Product", breadcrumb "Source Report") | `report/product/product-report.page.vue` | auth | report tabs | per-product stats |
| `/app/reports/affiliate-report` | `report/affiliate/affiliate-report.page.vue` | auth | report tabs | per-affiliate stats |
| `/app/reports/brb-report` | `report/brb/brb-report.page.vue` | auth | report tabs (**tab shown only when the platform is Amazon**) | Amazon Brand Referral Bonus report |

### 2.7 Communication and support

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/app/messages` | `shared…/algo-chat/Chat.vue` | auth (sidebar needs `messages.messages:access`) | sidebar "Messages" (badge), header icon, every "Message" button (use-message mixin) | 1:1 chat with affiliates |
| `/app/email-campaigns` | `email-campaigns/index.page.vue` | auth (sidebar needs `email_campaigns…:view`; the Send tab needs `:create`) | sidebar "More → Email Campaigns", list card "Send Email" (`?listIds=&type=`) | SMTP-based email campaigns and history |
| `/app/support-tickets` | `shared…/algo-tickets/ticket-list.page.vue` | perm `help_center.help_center:view` | sidebar "More → Help Center", user menu | tickets list and FAQ |
| `/app/support-ticket-details/:id` | `shared…/algo-tickets/ticket-details/ticket-details.page.vue` | auth (**no permission meta**) | ticket list rows | ticket thread |

### 2.8 Money: payments, wallet

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/app/payment-history` | `payment-history/payment-history.page.vue` (parent) | auth | `profile-widget-items.js` (not rendered in the seller layout) | tabs for commission and subscription statements |
| `/app/payment-history/commission` | `commission-transactions.table.vue` | auth (tab needs `payments.commission_statements:view`; without it the page redirects to subscription) | sidebar "More → Payments", settings billing tab | commission charges to the seller (affiliate commission + Ainfluencer fee) |
| `/app/payment-history/commission-invoice/:id` | `commission-invoice/commission-invoice.page.vue` | auth | commission row "Invoice" (needs `…:export`) | invoice detail and download |
| `/app/payment-history/subscription` | `subscription-transactions.table.vue` | auth | settings billing, manage-subscription "View billing" | Stripe subscription charges and invoices |
| `/app/wallet` | `wallet/wallet-main.page.vue` | auth (header badge needs `wallet.agency_wallet:view`) | header wallet badge, reveal-wallet modal, status pages | balance and deposit (withdraw is "Coming soon...") |
| `/app/wallet/transaction-history` | `wallet/transaction-history/transaction-history.page.vue` | perm `wallet.agency_wallet:view` (the link itself needs `wallet.agency_transaction_history:view`) | wallet page | usage, deposit and withdraw transactions |
| `/app/wallet/success-payment` | `wallet/status-page/success-payment.page.vue` | auth | **no in-app link**: the Stripe return URL, presumably via backend `seller/v1/wallet/checkout/success` | "Deposit Successful!" |
| `/app/wallet/failed-payment` | `wallet/status-page/failed-payment.page.vue` | auth | **no in-app link**: Stripe cancel URL (`…/checkout/cancel`) | "Deposit Failed" |

### 2.9 Team roles

| Path | Component | Guard | Inbound | Purpose |
|---|---|---|---|---|
| `/app/role-permission/:teamId/create` | `role-permission/create/create-role-permission.page.vue` | perm `setting.team:create` | settings, team, roles "Add New Role" | define a role's permission matrix |
| `/app/role-permission/:teamId/edit/:roleId` | same component | perm `setting.team:update` | settings, team, roles row | edit a role |

Layout chrome, visible on every `/app` page:

- **Sidebar.** Dashboard. Affiliates & Influencers, which has Affiliate Discovery and an *external* link "Influencer Discovery" to app.ainfluencer.com/search. My Products. My Brands or Stores. Campaigns, an *external* link to app.ainfluencer.com/campaigns. More: My Promotions, My Lists, Partnership, Messages, Email Campaigns, Reports, Payments, Help Center. A child with a non-zero count badge (Partnership, Messages) is promoted to the top level.
- **Sidebar extras.** A "Switch Teams" box appears when the user belongs to more than one team. The sidebar footer has the Go-Pro / manage-subscription widget.
- **Header.**
  - Store-account switcher ("amazon-dropdown"). It lists connected Amazon and Shopify accounts plus "Add New Store", which goes to `/app/profile?tab=manage_accounts`.
  - Wallet badge.
  - Messages icon and Notifications bell (side drawer).
  - Onboarding help panel, which shows per-route guideline HTML from `seller/v1/guidelines/seller-<route>`.
  - User menu: daily limits, a panel switcher, Help center, Account settings, Edit profile and Logout.

---

## 3. User-facing flows and visible business rules

Labels are quoted from the UI. Enum labels (statuses, types) are **served by the backend** in `seller/v1/system/basic-info` → `enums.*`, so the front end only knows their keys.

### 3.1 Registration and email verification

1. Go to `/user/register`. Fields: **Full name** (required, 1–100 characters, valid name), **Email** (valid), **Password** (8–20 characters, at least one capital letter and one number). Google **reCAPTCHA** must be solved. The agreement text links to the Privacy Policy and Terms of Service.
2. `POST seller/v1/auth/register` sends `name, email, password, recaptcha_token`, plus:
   - `promoter_ref_code` from the FirstPromoter cookie `_fprom_ref`;
   - `partnership_request_token` from the `temp_partnership_token` cookie, which the buyer or landing app sets when someone applies to a brand.
3. The screen switches to "Check your Email … We sent you an email with instructions on how to set up your account." There is no in-app code entry: `verifyEmail`, `resendEmailVerifycode` and the OTP email UI are dead code.
4. The email link goes to `/user/email-verified/{token}`. The app stores the token, then `/user` → `/app`. The `/app` guard sees `register_completed=false` and sends the user to `/user/register-steps`.
5. Google or Facebook buttons on the login and register pages start social login (§3.2). The Amazon social login button is commented out.

### 3.2 Login, SSO from the core Ainfluencer app, impersonation, panel switch

**Email login.** `/user/login` takes email and password; reCAPTCHA is required (bypassed only in development). The "Remember for 30 days" checkbox is **not wired** to anything. The call is `POST seller/v1/auth/login` with `partnership_request_token` when the cookie exists. After login the app sets the setting `youtube_modal_show=true`, then does a hard redirect to **`/app/discovery-affiliates?grouped=true`**.

**Social login.** `POST seller/v1/auth/login/{google|facebook}` returns `redirect_link`, which opens in a popup. The backend callback lands on `/auth?status=success&token=…` (or `status=error&message=…`). `auth-init.script.js` stores the token and either notifies the opener through `window.opener[CLOSE_ALGO]` (Safari uses localStorage messaging) or reloads `/`. Errors arrive in the main window as `?auth-social-error=` or `?status=error&message=` and are shown as a red alert by `App.vue`.

**SSO / deep link from core or backend emails.** Backend links of the form `/user/email-verified/{sanctumToken}?redirect=<path>` work like this:

1. `setSsoRedirectPath()` normalizes `redirect` to `/app/<path>` (relative paths only) and stores it in **sessionStorage**.
2. After the guard passes, the `dashboard` route consumes the stored path and navigates there.
3. If onboarding is unfinished, the final "done" step uses it instead of the default discovery page.

The `?redirect=` that the permission guard adds is **not** consumed anywhere.

**Token handoff.** `/auth?source=local&token=X` stores X and goes to `/`.

**Admin impersonation.** `/impersonate?token=X` stores the token with `impersonate_mode=true`. "Logout" in impersonation mode clears the token and sends the browser to `https://admin.<domain>`.

**Panel switch** (user menu switch: Affiliate | Seller):

- If the user already has an affiliate account, the stored `affiliate_link` opens in a new tab.
- Otherwise a prompt "Create Affiliate account" appears. Confirming calls `GET affiliate/v1/auth/become-affiliate`, then opens `/auth?source=redirect&panel=affiliate`. That page fetches `GET affiliate/v1/auth/local-login` with the seller bearer token and follows `data.link`.
- Panel names come from `VUE_APP_*_PANEL_NAME`, which only exist in `.env.development` (see §6).

**Logout** broadcasts to other tabs (BroadcastChannel `algorift-broadcast-channel`) and clears all of localStorage. Logging in in one tab also reloads the other tabs.

### 3.3 Onboarding wizard (`/user/register-steps`)

The steps come from `GET seller/v1/profile/show-register-steps`, sorted by `sort_order`. The wizard starts at the first step with `is_passed=false`, and a synthetic "done" step is added at the end. A step whose `is_mandatory=false` can be skipped (`POST seller/v1/auth/skip-step {step}`).

| step_title | Screen and rules |
|---|---|
| `verify_mobile` | Phone input (country list, with ignored countries PK, IN, RU, KP, IQ, IR, CU, SD, SY, AF), then `POST auth/mobile-verification-code`. A **6-digit** OTP follows (`POST auth/mobile-verify`). Resend becomes available after a **60 s** timer. |
| `questions` | Questionnaire from `GET seller/v1/questions`. The answers are auto-submitted (`POST seller/v1/questions`) when changed. |
| `sync_social_account` | "Connect your stores": **Connect to Amazon** (Amazon Seller Central OAuth popup, `POST profile/sync/account/amazon`) or **Connect to Shopify** (modal asks for the shop domain, then `POST profile/sync/account/shopify {shop}`, then a full-page redirect to Shopify). "Next" is enabled only when at least one Amazon or Shopify account exists, or when the step is skippable. Code bugs in this step: (a) the Shopify list is never populated after reload (`shopifyAccounts` is never set from `synced-accounts`), so a Shopify-only seller cannot press "Next" unless the backend marks the step as passed or it is skippable; (b) the "Add Seller Central account" item under the Shopify list calls the Amazon sync. |
| `profile` | Avatar upload with crop (JPG/PNG only), bio, **gender** (male/female/other) and **country** are required. Saved with `POST seller/v1/profile` (multipart). |
| `done` | Button goes to the SSO redirect path or `/app/discovery-affiliates?grouped=true`. |

The header has Logout (or the impersonation logout).

### 3.4 Connecting and switching stores (Amazon / Shopify)

- **Where:**
  - Profile → "Manage accounts" (`/app/profile?tab=manage_accounts`; changing it needs `edit_profile.accounts:update`);
  - the onboarding step;
  - header switcher → "Add New Store";
  - My Brands → "Add Shopify/Amazon Store".
- **Listing:** `GET seller/v1/profile/platform-accounts` returns `{amazon[], shopify[]}`. Each account shows a connected/disconnected status dot.
- **Plan limit:** the badge "Seller Accounts Limit: spend / limit" comes from `profile/overview` → `plans.accounts`. When spend equals limit the UI shows "Account Limit Reached – Upgrade your plan to add more accounts". The backend may also answer 402 with type `seller_account` ("Seller Account Limit Reached").
- **Connect:**
  - "Connect New Amazon Store": OAuth popup through `POST profile/sync/account/amazon`.
  - "Connect another Shopify account": shop-domain modal, then `POST profile/sync/account/shopify`, then a full-page redirect.
  - Google and Facebook "quick login" connections use `profile/sync/account/{google|facebook}` (the UI block is mostly commented out).
- **Delete:**
  - Amazon: `DELETE seller/v1/profile/synced-amazon/{id}`.
  - Shopify: `DELETE seller/v1/profile/synced-shopify/{id}`. **No such backend route exists, so it returns 404** (see §4.2).
  - Social: `DELETE profile/synced-accounts/{id}`.
- **Switching:** the header dropdown lists the accounts from `GET seller/v1/profile/platform-accounts/dropdown`. Choosing one stores it and reloads the page; on a brand or product detail page it navigates to `/app` first. Every API call then carries `X-Platform`. If nothing has been chosen yet, the layout derives a default from auth-status `platform`; **the whole layout stays hidden until a platform is set**.
- **Wording:** Amazon shows "Brand", "AMZ" and ASIN. Shopify shows "Store" and "Products", with `GET seller/v1/stores` and `/stores/{id}` instead of `products/brands`.
- **Amazon Attribution notice:** My Brands shows an "Important notice" button opening "Important Notice from Ainfluencer". It says Ainfluencer creates ad groups in the seller's Amazon Ads account and that groups starting with "ainfluencer_" must not be deleted or renamed.

### 3.5 Dashboard (`/app`)

- **Cards:**
  - "Complete Profile" with a percentage from `profile/profile-completion-percentage` (`completed_count / required_count`). Shown when below 100% and the user has `edit_profile.edit_profile_information:view`.
  - "Connect Your Credit Card – We use Stripe to connect your card safely…". Shown when `payment_profile_status ≠ connected` and the user has `setting.stripe_account:view`. It links to Settings → Billing.
- **KPI boxes** for brands, products, affiliates, promotions and contracts (`dashboard/report/*`), with "See …" links.
- **Charts and tables:**
  - "Performance Report (Last Week)" (`reports/performance/chart`; needs `reports.reports:view`);
  - brand pie chart;
  - Top Products and Top Affiliates;
  - "Recent Partnerships" (`dashboard/contracts`) with Details and **Revoke**;
  - incoming and outgoing request tables (accept, reject, cancel), shown with `partnership.partnership:view`.
- The YouTube welcome modal ("🚀 Welcome to Your Ainfluencer Dashboard!") exists, but its call is commented out, so it never shows.

### 3.6 Affiliate discovery, inviting affiliates (partnership requests)

- **Landing** (`?grouped=true`): "Find the Perfect Affiliates Effortlessly". Rows of grouped affiliates come from `discover/affiliates/landing`, with "View All" per group and "See All Affiliates". The page shows daily quotas from auth-status: **Daily Conversations, Daily Email Reveals, Daily Invites** ("unlimited" when the limit is −1). The "Invite via Email Campaigns / +72.5M Affiliates" button goes to Expanded Network (needs `affiliates.unlisted_affiliates:view`).
- **List view:**
  - Search, grid or table layout.
  - Filters: Platforms, Categories, Partnership status, Partners min/max, Location, Gender, Promotional methods, "Only Pro Affiliates", and follower min/max per network (YouTube, Instagram, Facebook, TikTok, Pinterest, X).
  - Each card offers "Invite affiliate", "Add to list" and "Send message".
- **Single invite** (`partnership-request-modal.vue`):
  1. Choose Brand or Product.
  2. Pick a brand or store. Only **active** brands without an existing partnership are listed (`products/brands/drop-down?has_partnership=false&status=active`). For a product invite, pick products of that brand (active and not partnered).
  3. Optionally "Propose custom commission": a percentage greater than 0 and at most 100. It defaults to the first product's commission.
  4. "Message for invitation" is required.
  5. Sent with `POST seller/v1/partnership/brand/request` or `/product/request` (`type: affiliate`, `assignee_ids`, `seller_brand_ids` or `seller_product_ids`, commission fields).
- **Bulk invite** (select cards, then "Request"):
  1. The summary shows "Total selected / Currently Partnered / Unpartnered" and "Daily limit left (<plan>)", with an "Upgrade Plan" link.
  2. The same form runs a pre-check: `POST partnership/{brand|product}/check-pre-request`. If `partnered_requests_count > 0`, a confirmation step appears before sending.
- **Limits:** the backend enforces daily invites (402 type `affiliate_invite`: "Daily Invitation Limit Reached") and active contracts (`active_contract`). Commission rules come back as 402 too:
  - `private_commission`: the **Pro plan only allows 20% or more**;
  - `commission`: customizable commission needs the **Ultimate** plan.
- **Permission:** the details page checks `partnership.partnership:request` ("You are not authorized to send partnership request.").
- **Affiliate detail page:**
  - "Show Affiliate Email" uses the reveal flow (§3.8).
  - Also available: Message, Add To List and "Request New Partnership".
  - Brand Contracts and Product Contracts tables.
  - "Affiliates in Related Categories / Promotional Methods".
  - "Affiliate Landing Page" opens `{BUYER_PANEL_URI}/user/{id}`.

### 3.7 Partnership management (`/app/partnerships/...`, `/app/my-affiliates`)

The Products ↔ Brands switch and the tabs map to these lists:

| Tab (`?tab=`) | Data |
|---|---|
| **Applications** (`requests`) | incoming: affiliates who applied (`partnership/requests` type=applied, status pending) |
| **Partnered** (`active`) | `partnership/contracts` with status active |
| **Invitations** (`invitations`) | outgoing invites the seller sent |
| **Revoked** | revoked contracts |
| **Rejected** | rejected requests |

The sidebar Partnership badge shows `new_partnership_requests_count`.

- **Accept** (`PATCH partnership/applied/{brand|product}/change-status/{id}` with status approved) shows "Partnership Successfully Initiated – You can now check your contract details in {Brand} active contracts section." and offers "Start Chat".
- **Reject** (status rejected) asks "Are You Sure to Reject Partnership? If you reject a partnership you cannot revert this action."
- **Cancel invitation** (`PATCH partnership/invited/{…}/change-status/{id}`) asks "Are You Sure You Want to Cancel This Invitation?"
- **Revoke contract** (`PUT partnership/contracts/{id}/revoke`; needs `partnership.partnership:revoke`) asks "Are You Sure You Want to Revoke This Partnership? Revoking this partnership will prevent your affiliate from promoting this {brand/product}, and all their efforts will be lost."
- **Details modals** show commission, profit, dates (requested, invited, partnered, revoked, rejected), dominant category, number of ASINs or products, terms of sales and message. The contract view (`partnership/contracts/{id}`) shows "Revoked by".
- **Auto-approval:** each brand has "Automatic Partnership Approval", a switch on the brand page (`PATCH products/brands/change-auto-partnership-approval`). When it is on, applications are accepted automatically by the backend.
- **My Affiliates** (`partnership/affiliate`) shows platforms, active products, total sales, total revenue and partnered-on date per affiliate, with a "Message" button.

### 3.8 Revealing affiliate emails (contact data)

There are two mechanisms in live code:

1. **Discovery bulk "Reveal Emails"** (`affiliate-reveal-email-modal.vue`). The modal shows "Total selected / Currently revealed / Unrevealed" and "Your daily email reveals left". It calls `POST seller/v1/reveal-emails/user-types {user_type_ids}`. **There is no backend route for this, so the action fails with 404** (see §4.2).
2. **Reveal-information flow** (`mixins/reveal-info.mixin.js`). Used by the affiliate detail page ("Show Affiliate Email"), the list "Emails Reveal" action, the expanded network page and detail drawer, and email campaign step 3.
   - If the selection fits within the remaining daily free reveals (`reveal_email_limit.limit − spend`), the **plan** modal appears.
   - Otherwise the **wallet** modal appears: "Pay for remaining emails using Wallet". Price per reveal comes from system-info (fallback **$0.10**), and the total is computed for **all** selected emails.
   - Both submit `POST seller/v1/reveal-information/process {item_ids, model_type (user_type | unlisted_affiliate), payment_method (plan | wallet)}`, then show "Emails revealed successfully".
   - Revealed lists: `reveal-information/list/{affiliates|unlisted-affiliates}`.

### 3.9 Expanded Network (non-registered creators)

- **Page:** "Your Next Top Affiliates Are Just a Click Away – Use your Email Campaigns to reach unlisted affiliates…". It shows "Daily free email reveals".
- **Data:** search and filters from `unlisted-affiliates/filter-options`; cards from `unlisted-affiliates`.
- **Detail drawer** (`unlisted-affiliates/{id}`) shows personal data, emails, phones and websites ("Reveal to access"), plus per-network stats for Instagram, YouTube, TikTok, Pinterest and websites.
- **Actions:** "Reveal Emails (n)" (§3.8), and "Add To List" or bulk "My Lists". The add-to-list modal can also create a list inline (`user-type-lists` POST, then `assign-items`). It confirms with "Affiliate added to list(s)" and a "View List" link to `/app/lists/unlisted-affiliates/:id`.

### 3.10 Lists (user-type lists)

- **Page:** "My Lists" shows affiliate lists (`user-type-lists/affiliates`) with search, filter and "Create New List".
- **Create** (`POST user-type-lists`):
  - **Manual List**: "you manually add affiliates".
  - **Smart List**: "automatically updates based on your filters". Smart lists take platform, profile (partnership status, number of partners, location, gender, is pro) and promotional filters. **"NOTE: After creation, Smart List filters will be permanent and cannot be changed."**
- **Card actions:**
  - Pin (`POST seller/v1/pins/toggle`);
  - Edit List Name (`PATCH user-type-lists/{id}`);
  - Duplicate as Manual (`POST …/{id}/duplicate`);
  - Delete (`DELETE …/{id}`);
  - Add Affiliate (goes to discovery);
  - **Send Email** (goes to `/app/email-campaigns?listIds=…&type=…`);
  - Message (`POST chats/messages/send-as-list`);
  - Emails Reveal;
  - **Send Partnership Request**, which uses `POST partnership/{brand|product}/request-as-list`. **There is no backend route for it, so it fails with 404** (see §4.2).
- **List detail:**
  - Members come from `user-type-lists/{id}/affiliates`, with revealed and unrevealed counts (`user-type-lists/revealed-stats`).
  - Remove selected members with `DELETE …/{id}/delete-items`. Removing from a smart list is blocked: "You can't remove affiliate from a smart list."
- **Add to list from cards:** `user-type-lists/drop-down-affiliates`, then `PATCH user-type-lists/assign-items`.
- **Promote from My Products:** "Promote Products To My Lists" sends `POST partnership/product/request` with `type: affiliate_list` and the chosen list ids. This works.

### 3.11 Products & commissions

- **Product list** (`GET seller/v1/products`):
  - Filters: status, brand, price from/to, commission from/to (from must be ≤ to), review count, availability.
  - An inactive product shows "This product is hidden for affiliates. Activate the product to unhide it."
- **Bulk actions:**
  - "Edit Products" opens the **Edit Commission** modal. It sends `PUT products/bulk-update-commissions {seller_product_ids, commission}` with commission greater than 0, and warns: "Lowering the commission rate will affect all existing brand partnerships and may lead to dissatisfaction among your affiliates."
  - "Promote" works as in §3.10.
- **Product detail** (`products/{id}`):
  - Banner "Product is Inactive … Activate Product Now".
  - Affiliation details: product status switch (`PUT products/change-status`; deactivating asks for confirmation), default commission edit (`PUT products/{id}/commission`; 0 ≤ value < 100; needs `products.all_products:update`).
  - Rating, number of affiliates ("View affiliates"), BSR, price, terms download.
  - Variations (`products/{id}/variations`).
  - Media assets: list, search and filter, add image, video or doc (chunked upload; MP4 for video; size limits from system config), delete.
- Commission plan rules come from the backend (402 `private_commission` and `commission`, see §3.6).

### 3.12 Brands / stores and the branded landing page

- **List** (`GET products/brands`, or `GET stores` for Shopify): cards show active contracts, average commission, status and average price. An inactive brand shows "This brand is hidden for affiliates. Activate the brand to unhide it."
- **Detail** (`products/brands/{id}`, or `stores/{id}`):
  - **Activation:** activate or deactivate with confirmation (`PUT products/brands/change-status`). The backend may answer 402 `brand`: "Brand Limit Reached – maximum number of active brands".
  - **Edit brand:** image (cropped) and bio via `POST products/brands/{id}/profile`.
  - **Social Links:** Instagram, Facebook, Pinterest, TikTok, X and website (`PUT products/brands/update/socials/{id}`).
  - **Automatic Partnership Approval** toggle.
  - **Amazon brand link** (Amazon only): "Set your Amazon brand link" → "Send for approval" (`PUT products/brands/update/amazon-link`) → "Pending approval" until approved, then "See on Amazon".
  - **Terms of Sales:** upload PDF, DOC or DOCX only ("Please Choose Correct File Format"), via `POST products/brands/update/terms-file`. Download or delete is also possible.
  - **BRANDED LANDING PAGE:** "Go to Landing", "Copy Link", "Share to..." and "Customize". These are disabled while the brand is inactive ("Activate your brand to use these").
    - Link = `{VUE_APP_BUYER_PANEL_URI}/seller/brand/{slug}` (Amazon) or `/seller/store/{slug}` (Shopify).
    - Customize modal: Landing URL slug ("must be unique"), "Hide Featured Products", choose up to **3** featured products, hide "Commission Paid, Revenue, Clicks, Sales Units & Conversions" analytics. Saved with `PUT products/brands/update/landing-page/{slug}`.
  - **Stats tiles:** clicks, conversions, sales, revenue, commission paid, page views, conversion rate.
  - **Tabs:** overview, products, assets.
- Brand-level edits need `brands.all_brands:update` or `brands.custom_brands:update` for that brand id.

### 3.13 Promotions (claim-code coupons)

- **List** (`GET promotions`):
  - Columns: Access (public or private), Status, Brand, Claim Code (copy), From, Affiliates, number of products.
  - Status badge labels used in code: **Active, Inactive, Expired, Waiting for approval**.
  - Creating needs `promotions.all_promotions:create`. The backend may answer 402 `promotion`: "Upgrade to the Ultimate Plan to create promotions".
- **Create wizard** ("All promotions will undergo a review by our team prior to approval"):
  1. "Select Product for Promotion". Requires a connected Amazon seller account ("Connect Your Amazon Seller Account: To View Active Brands & Products"; `profile/profiles`). Then choose one **active** brand ("Please activate at least one brand first…") and its products; each product shows its default commission.
  2. "Promotion Details":
     - **Promotion Code** is required, letters and numbers only, no spaces, **8–12 characters**, and must be re-entered to confirm ("Codes are not same").
     - **Availability** is Public or Private ("If choosing private, you must select affiliates in next step"; in practice assignment happens later through "Edit Assigned").
     - Name is required, plus "Buyer Visibility (only on public)" text and an optional internal note.
     - From and To dates are required, with from earlier than to.
     - **Discount Type** is percentage, fixed amount or free shipping. **Value** is required, and at most 100 for a percentage.
     - Saved with `POST promotions`.
  3. "Your promotion has been successfully created. It will be reviewed by our team and, if accepted, will go live soon. Your affiliates will receive a notification … after it is approved."
- **Details:** "This promotion is available to the public and can be accessed by all your partnered affiliates", or "No One Has Access To This Promotion".
  - **Edit Duration** (`PATCH promotions/{id}`).
  - **Edit Promotion** (dates and buyer visibility).
  - **Edit Assigned**: pick partnered affiliates and the products each can access (`GET/PATCH promotions/{id}/affiliates`).
  - **Delete** (`DELETE promotions/{id}`).

### 3.14 Reports

- **Tabs:** Performance, Brand (or Store), Product, Affiliate, and **BRB** (Amazon only).
- **Each tab:** date range, filters (brands, products, affiliates), chart (`/chart`) and table, and "Select to compare" on the brand, product and affiliate tabs.
- **Export:** "Export Data As CSV" needs `reports.reports:export` ("You are not authorized to export reports."). It calls `POST reports/{…}/export`.
- **Demo data:** every report has a **"Demo Data:" switch** that replaces real data with client-side generated fake numbers. Support should be aware that screenshots may show demo data.
- **HTTP methods:** brand, product and affiliate list and chart calls use **POST**; performance and BRB use GET.

### 3.15 Subscription and plan limits

- **Page:** `/app/manage-subscription` lists plans from `GET seller/v1/plans` with a monthly, quarterly or yearly toggle. It also shows subscription status (`plans/subscriptions/status`) and active plan.
- **Changing plan:**
  - Free plan, or no active main plan: checkout. `GET plans/{stripePriceId}/subscribe` returns a URL, the UI says "redirecting to Stripe...", then redirects. If the plan has no Stripe price for the chosen billing mode: "This billing option is not available for checkout."
  - Same level: switch billing period (switch modal or upgrade modal).
  - Higher level: "Yes, Upgrade" (`PUT plans/{id}/update`).
  - Lower level: downgrade modal with the checkbox "I understand that my data will be permanently deleted 30 days after my billing period ends…".
- **Cancel / resume:**
  - "Cancel Plan" opens the modal "…losing access to exclusive seller features… data permanently deleted 30 days after my billing period ends" and calls `PUT plans/cancel`.
  - "Resume plan" calls `PUT plans/resume`.
- **Other entry points:** "Get your custom plan" points to booking on the website. "View billing" goes to subscription statements.
- **Limit modal (402) types** (`SellerFeatureNameEnum`):

  | Type | Title | Message |
  |---|---|---|
  | `brand` | Brand Limit Reached | – |
  | `seller_account` | Seller Account Limit Reached | – |
  | `affiliate_invite` | Daily Invitation Limit Reached | – |
  | `active_contract` | Active Contract Limit Reached | – |
  | `private_commission` | (commission rule) | Pro allows ≥20% only |
  | `commission` | Upgrade to Ultimate | – |
  | `promotion` | (promotion rule) | Ultimate only |
  | `chat_system` | Daily Limit Reached | new conversations |

  Every type shows "Upgrade Plan Now", which goes to manage-subscription, plus the current limit when the backend returns it.
- **User menu** shows Daily Invites, Daily Email Revealed and Daily New Conversations as spend/limit.

### 3.16 Payments, billing and cards

- **Settings → Billing:** "Manage Credit Cards". `GET payments/create-billing-portal` returns a Stripe Customer Portal URL, which opens after "Redirecting to Stripe". This needs `setting.stripe_account:update`. The status text depends on `payment_profile_status`. Without a payment profile, some backend calls answer **499**, which forces a redirect to Settings → Billing.
- **Commission statements** (`payments/commission-transactions`):
  - Columns: Affiliate, Total Sale, Due Date, Commission Paid, Ain Fee, Total Paid, Status, plus a "Sale Report" link.
  - Tooltip: the "Ainfluencer fee is based on **3.5%** of your sales generated by affiliates under the Ultimate Plan". The total excludes Stripe fees, taxes and FX.
  - Row action: a succeeded row opens the **invoice page** (`…/{id}`, `…/{id}/items`, "Download Invoice" = `…/{id}/export`). Any other status gets **Retry** (`POST …/{id}/retry`), which shows "The invoice payment has been retried successfully and processed". Both need `payments.commission_statements:export`.
- **Subscription statements:** Charges and Invoices tabs (`payments/charges`, `payments/invoices`) with a View or copy-link action.

### 3.17 Wallet

- **Wallet page:** balance comes from auth-status `wallet_balance`, which is also shown in the header badge.
- **Deposit** needs `wallet.agency_wallet:deposit`:
  1. Enter an amount in $. **There is no client-side validation.**
  2. Tick "I've read and agree to the Wallet Terms & Conditions". The link is `#`, a placeholder.
  3. `POST seller/v1/wallet/checkout/create` → "Redirecting to Stripe" → Stripe Checkout.
  4. Stripe returns to `/app/wallet/success-payment` ("Deposit Successful!") or `/app/wallet/failed-payment`, presumably through the backend `wallet/checkout/success|cancel`.
- **Withdraw** shows "Coming soon...".
- **Transactions History** (`wallet/transactions/list`): tabs Usage, Deposit, Withdraw, with "View Details" (Usage Details modal).
- **Use:** the wallet pays for email reveals beyond the daily free quota (§3.8).

### 3.18 Messages (chat)

- **Page:** `/app/messages` (`algo-chat`).
- **Contacts:** `chats/contacts`, with search and filters (type, partnership status, category, promotion method, social accounts, date range). Contacts can be tagged (`chats/contacts/tag`) and marked read (`chats/read-contact-messages`).
- **Online status:** Pusher presence channel.
- **Messages:** `chats/contacts/{id}/messages` and `POST chats/messages` (reply, emoji), saved messages (`chats/messages/saved`), find-page for jumping to a message.
- **Start a conversation:**
  - "Start new conversation" modal (affiliate list with a "Partnered" badge).
  - Any "Message" button: `chats/find-contact/{id}`; if no contact exists, an empty conversation is opened with the affiliate info.
- **Bulk messages:** "Message" with "Custom Value" variables sends `POST chats/messages/send-as-bulk` (selection) or `…/send-as-list` (list), then "Message sent successfully".
- **Limit:** the daily new-conversation limit comes back as 402 `chat_system`.
- **Templates and auto-reply:** managed in Settings → Message Templates (§3.21).

### 3.19 Email campaigns (SMTP)

The **Send** tab is a 3-step wizard (needs `email_campaigns.all_email_campaigns:create`; without it only History is shown).

1. **"Select an SMTP for This Campaign"**: the list comes from `smtp-connections/list`. Each connection can be tested (`POST smtp-connections/test`) or deleted. "Add New SMTP" asks for:
   - Name ("visible only to you");
   - SMTP Host, SMTP Port, Username, From Email;
   - Password or App Password;
   - Encryption Type.

   Saved with `POST smtp-connections/store` (or `PUT …/update/{id}`).
2. **"Select Recipients"** from Ainfluencer Affiliates or Expanded Network, picking either **My Lists** or **All Revealed** emails (`reveal-information/list/*`). Opening from a list card pre-selects that list.
3. **"Write & Finalize Your Message"**:
   - Shows list name, total in list, recipients revealed and "Need to Reveal", with a **Reveal Emails** button (§3.8).
   - Subject and Body are required. Body supports Custom Values and Message Templates.
   - There is a "Schedule Email" toggle and date-time picker.

**Send Email** calls `POST email-campaigns/store {smtp_connection_id, recipients, message, subject, name=subject}`. The success screen says: "Your email campaign has been successfully submitted. It will be sent **after approval by the Ainfluencer team** at the selected time…". **Code bug:** the schedule value is collected but **not sent** in the payload, so the "selected time" is ignored client-side.

The **History** tab lists campaigns (`email-campaigns/list`) with status (backend enum) and a message-body view. Actions: **Resend** (`GET email-campaigns/resend/{id}`) and **Cancel** (`GET email-campaigns/cancel/{id}`, needs `…:cancel`).

### 3.20 Teams and roles

- **Settings → Team:**
  - "Your Team Members": `teams/{id}/members/list`. Members can have their role changed (`PUT …/members/update/{memberId}`) or be removed.
  - "Invited People": "Send New Invitation" takes an email and a role ("member access limits"), via `POST teams/{id}/invitation`. Invitations can be cancelled.
  - "Roles": add, edit or delete roles (`teams/{id}/roles/*`).
  - "Invited Teams": accept or reject invitations from other sellers' teams (`PATCH teams/invitations/{id}/change-status`).
- **Role editor** (`/app/role-permission/...`): a permission matrix from `teams/permissions/list/{roleId}`.
  - Resources: brands (all or custom per-brand), products, affiliates, promotions, partnership, messages, reports, payments, email campaigns, help center, wallet, edit profile, settings, lists.
  - "Custom" brand access lets the owner pick specific brands.
  - Messages: "No changes to submit", "Role created successfully".
- **Switching teams:** the sidebar "Switch Teams" calls `GET teams/{teamId}/impersonate`, which returns a new token. The token is stored, other tabs are told to reload, and the page reloads. As a team member, `$can()` hides menu items and the route guard redirects to `/app` when access is missing.

### 3.21 Settings: notifications, security, message templates

- **Notifications:** an In-App / Email matrix (`settings/notifications`, update with `PUT settings/notifications/update`; needs `setting.notifications:update`).
- **Security:**
  - "Reset Password (Need to verify)" opens a confirmation, then `POST profile/request-change-password-by-email`, then "Check Your Inbox".
  - "Active Sessions" (`profile/sessions`) offers Terminate (`DELETE …/{id}`) and "Terminate All Except Me" (`DELETE …/all`).
- **Message Templates** (`message-templates/*`): each template has a Category ("Templates can be reused across Email, Partnership, and Chat pages"), a Shortcut (e.g. `/invite1`), the message with Custom Values (`message-variables/list`), and an **Auto Reply** keyword: "When someone sends you this word in the chat, your saved message will be sent automatically as a reply."

### 3.22 Notifications

The header bell opens a side drawer (`notifications`) with suggestions/filters (`notifications/suggests`), bookmarks (`PATCH notifications/{id}/toggle-bookmark`), mark-read (`PATCH notifications/{id}/read`) and "Mark all as read" (`PATCH notifications/read-all`). Browser push uses Firebase Cloud Messaging; the token is registered with `POST seller/v1/user-fcm-tokens`.

### 3.23 Help center (tickets)

`/app/support-tickets`:

- **List:** "n tickets", "Create new ticket", FAQ, and Close ticket.
- **New ticket:** Department (`ticket_category_id`), Impact of issue (`priority`), Title, Message, and Attachments (PDF, JPG, PNG or DOC). Sent with `POST seller/v1/tickets` (multipart).
- **Ticket detail:** thread (`GET tickets/{id}`), replies with attachments up to **5 MB** (`POST tickets/{id}/messages`), close (`PATCH tickets/{id}`) and "Rate this conversation".

### 3.24 Profile

Tabs are each gated by an `edit_profile.*` permission:

- **Overview:** current plan and usage (daily invites, daily conversations, seller accounts, active brands, daily applies, daily contracts), marketplace region and company data.
- **Edit profile information:** bio, full name, nickname, gender, birthday, residing country, company name, website and address. Changing the phone requires an OTP (`profile/request-change-mobile`, then `PUT profile/change-mobile`). Saved with `POST seller/v1/profile`.
- **Get better profile:** questions (`profile/questions`).
- **Manage accounts:** §3.4.

### 3.25 Not in the seller app

The seller SPA has **no screens** for these items from the brief:

- **Campaigns/programs.** The sidebar "Campaigns" item is an external link to app.ainfluencer.com/campaigns.
- **Associates.** `associate.enum.js` is unreachable.
- **Coupons**, except promotion claim codes (§3.13).
- **API keys** and **integrations management.** The backend has `seller/v1/api-keys` and `seller/v1/integrations/ainfluencer/*` (campaigns, coupons, cards, products, accounts, register), but no seller-SPA code calls them; they serve the core app.
- **Landing pages other than the brand landing page.** The affiliate custom landing page belongs to the affiliate app.

---

## 4. Backend calls

### 4.1 Summary per area (live calls only; full detail in `aff-seller-calls.json`)

Base for every call: `VUE_APP_API_BASE_URL` = `https://backstage.ainfluencer.com`; paths below are relative to it. "used on" lists the route(s) whose component tree reaches a caller. "(global: App/layout)" means the call is made from App.vue, the layout or a guard on every `/app` page.

#### Global bootstrap (5)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/guidelines/general` | getGlobalOnboarding (onboarding-store.js) | (global: App/layout) | GuidelinePageController |
| GET | `seller/v1/guidelines/{param}` | getOnBoardingForKey (onboarding-store.js) | (global: App/layout) | GuidelinePageController |
| POST | `seller/v1/pins/toggle` | togglePin (global.store.js) | /discovery-lists | ToggleUserTypeListController |
| GET | `seller/v1/profile/platform-accounts/dropdown` | getPlatformListDropdown (global.store.js) | (global: App/layout) | IndexPlatformAccountController |
| GET | `seller/v1/system/basic-info` | fetchSystemInfoFromServer (system-info.store.js) | (global: App/layout),  (dashboard), /affiliate/:id, /brand/:id +27 more | BasicInfoController |

#### Auth / onboarding (13)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `affiliate/v1/auth/become-affiliate` | sellerBecomesAffiliate (authentication.store.js) | (global: App/layout) | BecomeAffiliateController |
| GET | `seller/v1/auth/auth-status` | checkTokenValidation (authorization.store.js) | (global: App/layout),  (dashboard), /affiliate/:id, /brand/:id +44 more | AuthStatusController |
| POST | `seller/v1/auth/forget-password` | forgotPassword (authentication.store.js) | /user/forgot-password | ForgetPasswordController |
| POST | `seller/v1/auth/login` | login (authentication.store.js) | /user/login | LoginController |
| POST | `seller/v1/auth/mobile-verification-code` | verifyPhone (authentication.store.js) | /user/register-steps | SendMobileVerificationCodeController |
| POST | `seller/v1/auth/mobile-verify` | verifyOTP (authentication.store.js) | /user/register-steps | MobileVerifyController |
| POST | `seller/v1/auth/register` | register (authentication.store.js) | /user/register | RegisterController |
| POST | `seller/v1/auth/reset-password` | resetPassword (authentication.store.js) | /user/reset-password/:token | ResetPasswordController |
| POST | `seller/v1/auth/skip-step` | skipRegisterStep (authentication.store.js) | /user/register-steps | SkipStepController |
| GET | `seller/v1/profile/show-register-steps` | getRegisterStepsList (authentication.store.js) | /user/register-steps | ShowRegisterStepsController |
| GET | `seller/v1/questions` | getQuestionList (authentication.store.js) | /user/register-steps | RegisterQuestionListController |
| POST | `seller/v1/questions` | postQuestionList (authentication.store.js) | /user/register-steps | StoreRegisterQuestionController |
| GET | `{panel}/v1/auth/local-login` | redirect (auth-init.script.js) | (global: App/layout) | LocalLoginCallbackController |

#### Store connections (Amazon/Shopify) & social login (11)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| POST | `seller/v1/auth/login/amazon` | loginWithAmazon (social-service.store.js) | /user/login, /user/register | AmazonLoginController |
| POST | `seller/v1/auth/login/facebook` | loginWithFacebook (social-service.store.js) | /user/login, /user/register | FacebookLoginController |
| POST | `seller/v1/auth/login/google` | loginWithGoogle (social-service.store.js) | /user/login, /user/register | GoogleLoginController |
| POST | `seller/v1/profile/sync/account/amazon` | syncAmazonAccount (social-service.store.js) | /profile, /user/register-steps | SyncAmazonAccountController |
| POST | `seller/v1/profile/sync/account/facebook` | syncFacebookAccount (social-service.store.js) | /profile, /user/register-steps | SyncFacebookAccountController |
| POST | `seller/v1/profile/sync/account/google` | syncGoogleAccount (social-service.store.js) | /profile, /user/register-steps | SyncGoogleAccountController |
| POST | `seller/v1/profile/sync/account/shopify` | syncShopifyAccount (social-service.store.js) | /profile, /user/register-steps | SyncShopifyAccountController |
| GET | `seller/v1/profile/synced-accounts` | getSyncedSocialAccountsList (social-service.store.js) | /profile, /user/register-steps | SyncedAccountListController |
| DELETE | `seller/v1/profile/synced-accounts/{param}` | deleteSocialAccount (social-service.store.js) | /profile, /user/register-steps | DeleteSyncedAccountController |
| DELETE | `seller/v1/profile/synced-amazon/{param}` | deleteAmazonAccount (social-service.store.js) | /profile, /user/register-steps | DeletePlatformAccountController |
| DELETE | `seller/v1/profile/synced-shopify/{param}` | deleteShopifyAccount (social-service.store.js) | /profile, /user/register-steps | **NO BACKEND ROUTE** |

#### Profile (11)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/profile` | getProfileInfo (profile.store.js) | /profile | ShowProfileController |
| POST | `seller/v1/profile` | updateProfileInfo (profile.store.js) | /profile, /user/register-steps | UpdateProfileController |
| PUT | `seller/v1/profile/change-mobile` | verifyProfileMobileOtp (profile.store.js) | /profile | ChangeMobileNumberController |
| GET | `seller/v1/profile/overview` | getProfileOverView (profile.store.js) | /profile | ProfileOverviewController |
| GET | `seller/v1/profile/platform-accounts` | getPlatformAccounts (profile.store.js) | /profile | SellerPlatformAccountController |
| GET | `seller/v1/profile/questions` | getQuestionList (profile.store.js) | /profile | ProfileQuestionListController |
| POST | `seller/v1/profile/questions` | postQuestionList (profile.store.js) | /profile | StoreProfileQuestionController |
| POST | `seller/v1/profile/request-change-mobile` | verifyProfileMobileNumber (profile.store.js) | /profile | RequestChangeMobileNumberController |
| POST | `seller/v1/profile/request-change-password-by-email` | resetPassword (profile.store.js) | /settings | RequestChangePasswordByEmailController |
| POST | `seller/v1/profile/sync/account/amazon/profile/{param}` | syncAmazonProfile (profile.store.js) | /profile | SyncAmazonAccountProfileController |
| POST | `seller/v1/profile/sync/account/amazon/profile/{param}` | syncProductWithAmazon (profile.store.js) | /profile | SyncAmazonAccountProfileController |

#### Dashboard (11)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/dashboard/contracts` | getRecentPartnership (dashboard.store.js) |  (dashboard) | IndexPartnershipContractController |
| GET | `seller/v1/dashboard/report/affiliate-partnerships` | getAffiliatePartnershipInfo (dashboard.store.js) |  (dashboard) | ShowAffiliatePartnershipReportController |
| GET | `seller/v1/dashboard/report/brands` | getBrandInfo (dashboard.store.js) |  (dashboard) | ShowBrandReportController |
| GET | `seller/v1/dashboard/report/brands/chart` | getBrandChart (dashboard.store.js) |  (dashboard) | BrandChartReportController |
| GET | `seller/v1/dashboard/report/contracts` | getContractInfo (dashboard.store.js) |  (dashboard) | ShowContractReportController |
| GET | `seller/v1/dashboard/report/products` | getProductInfo (dashboard.store.js) |  (dashboard) | ShowProductReportController |
| GET | `seller/v1/dashboard/report/promotions` | getPromotionInfo (dashboard.store.js) |  (dashboard) | ShowPromotionReportController |
| GET | `seller/v1/dashboard/report/statistics` | getReportStatistics (dashboard.store.js) |  (dashboard) | ShowStatisticsReportController |
| GET | `seller/v1/dashboard/report/top-affiliates` | getTopAffiliateReport (dashboard.store.js) |  (dashboard) | TopAffiliateReportController |
| GET | `seller/v1/dashboard/report/top-products` | getTopProductReport (dashboard.store.js) |  (dashboard) | TopProductReportController |
| GET | `seller/v1/profile/profile-completion-percentage` | getCompleteProfilePercentage (dashboard.store.js) |  (dashboard) | ShowProfileCompletionPercentageController |

#### Affiliate discovery, reveal emails (8)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/discover/affiliates` | getAffiliateList (affiliate.store.js) | /discovery-affiliates, /messages, /my-affiliates, /partnerships/partnership-brands +6 more | IndexAffiliateController |
| GET | `seller/v1/discover/affiliates/landing` | getAffiliateListLanding (affiliate.store.js) | /discovery-affiliates | LandingDiscoverAffiliateController |
| GET | `seller/v1/partnership/affiliate/drop-down` | getAllAffiliatesOfTheSellerRequest (affiliate.store.js) | /partnerships/partnership-brands, /partnerships/partnership-products, /reports/affiliate-report | AffiliateDropDownListController |
| GET | `seller/v1/profile/affiliate/{param}` | getAffiliateInfo (affiliate.store.js) |  (dashboard), /affiliate/:id, /discovery-affiliates, /lists/affiliates/:id +4 more | ShowAffiliateDetailController |
| POST | `seller/v1/reveal-emails/user-types` | revealAffiliateEmail (affiliate.store.js) | /discovery-affiliates | **NO BACKEND ROUTE** |
| GET | `seller/v1/reveal-information/list/affiliates` | revealInformationAffiliates (affiliate.store.js) | /email-campaigns | ListAffiliateRevealInformationController |
| GET | `seller/v1/reveal-information/list/unlisted-affiliates` | revealInformationUnlistedAffiliates (affiliate.store.js) | /email-campaigns | ListUnlistedAffiliateRevealInformationController |
| POST | `seller/v1/reveal-information/process` | revealInformationProcess (affiliate.store.js) | /affiliate/:id, /discovery-lists, /email-campaigns, /expanded-network +2 more | ProcessRevealInformationController |

#### Expanded network (3)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/unlisted-affiliates` | getUnlistedList (unlisted.store.js) | /expanded-network | IndexUnlistedAffiliateController |
| GET | `seller/v1/unlisted-affiliates/filter-options` | getUnlistedFilterOptions (unlisted.store.js) | /expanded-network | ShowUnlistedAffiliateFilterOptionsController |
| GET | `seller/v1/unlisted-affiliates/{param}` | getUnlistedDetails (unlisted.store.js) | /expanded-network, /lists/unlisted-affiliates/:id | ShowUnlistedAffiliateController |

#### Lists (user-type lists) (13)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| POST | `seller/v1/user-type-lists` | createList (lists.store.js) | /discovery-lists, /expanded-network | CreateUserTypeListController |
| GET | `seller/v1/user-type-lists/affiliates` | getAffiliatesUserTypeLists (lists.store.js) | /discovery-lists, /email-campaigns | IndexAffiliateListsController |
| PATCH | `seller/v1/user-type-lists/assign-items` | assignItemToList (lists.store.js) | /affiliate/:id, /discovery-affiliates, /expanded-network, /lists/affiliates/:id | AssignUserTypeListItemController |
| GET | `seller/v1/user-type-lists/drop-down-affiliates` | getMyAffiliatesListDropdown (lists.store.js) | /affiliate/:id, /brand/:id, /discovery-affiliates, /discovery-products +1 more | UserTypeListAffiliateDropDownListController |
| POST | `seller/v1/user-type-lists/revealed-stats` | getListAffiliatesRevealedStats (lists.store.js) | /email-campaigns, /lists/affiliates/:id | RevealedStatsUserTypeListsController |
| GET | `seller/v1/user-type-lists/unlisted-affiliates` | getUnlistedAffiliatesUserTypeLists (lists.store.js) | /email-campaigns, /expanded-network | IndexUnlistedAffiliateListsController |
| DELETE | `seller/v1/user-type-lists/{param}` | deleteList (lists.store.js) | /discovery-lists | DeleteUserTypeListController |
| PATCH | `seller/v1/user-type-lists/{param}` | editList (lists.store.js) | /discovery-lists | UpdateUserTypeListController |
| GET | `seller/v1/user-type-lists/{param}/affiliates` | getListAffiliates (lists.store.js) | /lists/affiliates/:id | ShowUserTypeListAffiliateController |
| DELETE | `seller/v1/user-type-lists/{param}/delete-items` | removeItemFromList (lists.store.js) | /lists/affiliates/:id, /lists/unlisted-affiliates/:id | DeleteUserTypeListItemController |
| POST | `seller/v1/user-type-lists/{param}/duplicate` | duplicateList (lists.store.js) | /discovery-lists | DuplicateUserTypeListController |
| GET | `seller/v1/user-type-lists/{param}/unlisted-affiliate-membership` | getUnlistedAffiliatesMembershipList (lists.store.js) | /expanded-network | UserTypeListUnlistedAffiliateMembershipController |
| GET | `seller/v1/user-type-lists/{param}/unlisted-affiliates` | getListUnlistedAffiliates (lists.store.js) | /lists/unlisted-affiliates/:id | ShowUserTypeListUnlistedAffiliateController |

#### Partnerships (15)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/partnership/affiliate` | getPartnershipAffiliates (partnership.store.js) | /my-affiliates | AffiliateContractListController |
| PATCH | `seller/v1/partnership/applied/brand/change-status/{param}` | updateBrandPartnershipSentStatus (partnership.store.js) |  (dashboard), /partnerships/partnership-brands | ChangeStatusAppliedBrandRequestController |
| PATCH | `seller/v1/partnership/applied/product/change-status/{param}` | updateProductPartnershipSentStatus (partnership.store.js) |  (dashboard), /partnerships/partnership-products | ChangeStatusAppliedProductRequestController |
| POST | `seller/v1/partnership/brand/check-pre-request` | checkPartnershipBrandPreRequest (partnership.store.js) | /discovery-affiliates | BrandRequestCheckController |
| POST | `seller/v1/partnership/brand/request` | sendPartnershipBrandRequestToAffiliate (partnership.store.js) | /affiliate/:id, /discovery-affiliates, /lists/affiliates/:id | BrandRequestController |
| POST | `seller/v1/partnership/brand/request-as-list` | sendPartnershipBulkBrandRequestToList (partnership.store.js) | /discovery-lists, /lists/affiliates/:id | **NO BACKEND ROUTE** |
| GET | `seller/v1/partnership/contracts` | getPartnershipContracts (partnership.store.js) | /partnerships/partnership-brands, /partnerships/partnership-products | IndexPartnershipContractController |
| GET | `seller/v1/partnership/contracts/{param}` | getPartnershipContract (partnership.store.js) |  (dashboard) | ShowPartnershipContractController |
| PUT | `seller/v1/partnership/contracts/{param}/revoke` | revokeContract (partnership.store.js) |  (dashboard), /partnerships/partnership-brands, /partnerships/partnership-products | PartnershipContractRevokeController |
| PATCH | `seller/v1/partnership/invited/brand/change-status/{param}` | updateBrandPartnershipReceivedStatus (partnership.store.js) |  (dashboard), /partnerships/partnership-brands | ChangeStatusInvitedBrandRequestController |
| PATCH | `seller/v1/partnership/invited/product/change-status/{param}` | updateProductPartnershipReceivedStatus (partnership.store.js) |  (dashboard), /partnerships/partnership-products | ChangeStatusInvitedProductRequestController |
| POST | `seller/v1/partnership/product/check-pre-request` | checkPartnershipProductPreRequest (partnership.store.js) | /brand/:id, /discovery-affiliates, /discovery-products | ProductRequestCheckController |
| POST | `seller/v1/partnership/product/request` | sendPartnershipProductRequestToAffiliate (partnership.store.js) | /affiliate/:id, /brand/:id, /discovery-affiliates, /discovery-products +1 more | ProductRequestController |
| POST | `seller/v1/partnership/product/request-as-list` | sendPartnershipBulkProductRequestToList (partnership.store.js) | /discovery-lists, /lists/affiliates/:id | **NO BACKEND ROUTE** |
| GET | `seller/v1/partnership/requests` | getPartnershipRequests (partnership.store.js) |  (dashboard), /partnerships/partnership-brands, /partnerships/partnership-products | IndexPartnershipRequestController |

#### Products & brands (32)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/categories` | getCategoryList (product.store.js) | /discovery-affiliates, /discovery-brands, /lists/affiliates/:id, /partnerships/partnership-products +1 more | IndexCategoryController |
| GET | `seller/v1/products` | getProductList (brand.store.js) | /affiliate/:id, /brand/:id, /discovery-affiliates, /discovery-lists +6 more | IndexProductController |
| GET | `seller/v1/products` | getAllProductsOfTheSeller (product.store.js) | /my-affiliates | IndexProductController |
| GET | `seller/v1/products` | getBrandProductList (product.store.js) | /new-promotion, /update-promotion/:id | IndexProductController |
| GET | `seller/v1/products` | getProductList (product.store.js) | /affiliate/:id, /brand/:id, /discovery-affiliates, /discovery-lists +8 more | IndexProductController |
| GET | `seller/v1/products/brands` | getBrandList (brand.store.js) | /brand/:id, /discovery-brands, /discovery-products, /reports/brands +2 more | IndexBrandController |
| GET | `seller/v1/products/brands` | getBrandList (product.store.js) | /brand/:id, /discovery-brands, /discovery-products, /reports/brb-report | IndexBrandController |
| PATCH | `seller/v1/products/brands/change-auto-partnership-approval` | updateBrandAutoPartnershipApproval (brand.store.js) | /brand/:id | ChangeAutoPartnershipApprovalController |
| PUT | `seller/v1/products/brands/change-status` | updateBrandStatus (brand.store.js) | /brand/:id | UpdateSellerBrandStatusController |
| GET | `seller/v1/products/brands/drop-down` | getAllBrandsOfTheSeller (brand.store.js) | /affiliate/:id, /discovery-affiliates, /discovery-lists, /lists/affiliates/:id +10 more | BrandDropDownListController |
| PUT | `seller/v1/products/brands/update/amazon-link` | updateBrandAmazonLink (brand.store.js) | /brand/:id | UpdateSellerBrandAmazonLinkController |
| PUT | `seller/v1/products/brands/update/landing-page/{param}` | updateBrandLanding (brand.store.js) | /brand/:id | UpdateSellerBrandLandingPageController |
| PUT | `seller/v1/products/brands/update/socials/{param}` | updateBrandSocials (brand.store.js) | /brand/:id | UpdateSellerBrandSocialsController |
| POST | `seller/v1/products/brands/update/terms-file` | updateTermFile (brand.store.js) | /brand/:id | UpdateBrandTermsFileController |
| GET | `seller/v1/products/brands/{param}` | getBrandDetail (brand.store.js) | /brand/:id | ShowBrandController |
| DELETE | `seller/v1/products/brands/{param}/assets` | deleteBrandAsset (product.store.js) | /brand/:id | DeleteBrandAssetController |
| GET | `seller/v1/products/brands/{param}/assets` | getBrandAssets (product.store.js) | /brand/:id | ShowBrandAssetController |
| GET | `seller/v1/products/brands/{param}/assets/{param}` | getBrandAssetsDetail (product.store.js) | /brand/:id | ShowBrandAssetDetailController |
| POST | `seller/v1/products/brands/{param}/profile` | updateBrandProfile (brand.store.js) | /brand/:id | UpdateSellerBrandProfileController |
| POST | `seller/v1/products/brands/{param}/upload-assets` | storeMedia via image/video/doc-asset-uploa… (rsumable.store.js) | /brand/:id | UploadBrandAssetController |
| PUT | `seller/v1/products/bulk-update-commissions` | updateBulkCommissionProducts (product.store.js) | /brand/:id, /discovery-products | BulkUpdateCommissionsProductController |
| PUT | `seller/v1/products/change-status` | updateProductStatus (product.store.js) | /product/:id | UpdateSellerProductStatusController |
| GET | `seller/v1/products/drop-down` | getAllProductsOfTheSellerRequest (product.store.js) | /partnerships/partnership-products, /promotion-list, /reports/product-report | ProductDropDownListController |
| GET | `seller/v1/products/{param}` | getProductDetail (product.store.js) | /product/:id | ShowProductController |
| DELETE | `seller/v1/products/{param}/assets` | deleteProductAsset (product.store.js) | /product/:id | DeleteProductAssetController |
| GET | `seller/v1/products/{param}/assets` | getProductAssets (product.store.js) | /product/:id | ShowProductAssetController |
| GET | `seller/v1/products/{param}/assets/{param}` | getProductAssetsDetail (product.store.js) | /product/:id | ShowProductAssetDetailController |
| PUT | `seller/v1/products/{param}/commission` | updateProductCommission (product.store.js) | /product/:id | UpdateCommissionProductController |
| POST | `seller/v1/products/{param}/upload-assets` | storeMedia via image/video/doc-asset-uploa… (rsumable.store.js) | /product/:id | UploadProductAssetController |
| GET | `seller/v1/products/{param}/variations` | getProductVariation (product.store.js) | /product/:id | IndexProductVariationController |
| GET | `seller/v1/stores` | getBrandList (brand.store.js) | /brand/:id, /discovery-brands, /discovery-products, /reports/brands +2 more | IndexStoreController |
| GET | `seller/v1/stores/{param}` | getBrandDetail (brand.store.js) | /brand/:id | ShowStoreController |

#### Promotions (coupon codes) (8)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/profile/profiles` | getSellerAmazonAccounts (promotions.store.js) | /new-promotion, /update-promotion/:id | AmazonAccountProfileDropDownListController |
| GET | `seller/v1/promotions` | getPromotionList (promotions.store.js) | /promotion-list | IndexPromotionController |
| POST | `seller/v1/promotions` | savePromotion (promotions.store.js) | /new-promotion, /update-promotion/:id | CreatePromotionController |
| DELETE | `seller/v1/promotions/{param}` | deletePromotion (promotions.store.js) | /promotion-details/:id, /promotion-list | DeletePromotionController |
| GET | `seller/v1/promotions/{param}` | getPromotion (promotions.store.js) | /new-promotion, /promotion-details/:id, /promotion-edit-assigner/:id, /promotion-edit-duration/:id +2 more | ShowPromotionController |
| PATCH | `seller/v1/promotions/{param}` | updatePromotion (promotions.store.js) | /promotion-edit-duration/:id, /promotion-edit/:id | UpdatePromotionController |
| GET | `seller/v1/promotions/{param}/affiliates` | getPromotionAffiliates (promotions.store.js) | /promotion-details/:id, /promotion-edit-assigner/:id | IndexPromotionAffiliateController |
| PATCH | `seller/v1/promotions/{param}/affiliates` | updateAffilialtedPromotionStatus (promotions.store.js) | /promotion-details/:id, /promotion-edit-assigner/:id | AssignPromotionAffiliateController |

#### Reports (16)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/dashboard/report/products` | getProductInfo (report.store.js) |  (dashboard) | ShowProductReportController |
| POST | `seller/v1/reports/affiliate` | getAffiliateList (report.store.js) | /reports/affiliate-report | AffiliateReportListController |
| POST | `seller/v1/reports/affiliate/chart` | getAffiliateReportChart (report.store.js) | /reports/affiliate-report | AffiliateReportChartController |
| POST | `seller/v1/reports/affiliate/export` | getAffiliateExportFile (report.store.js) | /reports, /reports/affiliate-report | AffiliateReportExportController |
| POST | `seller/v1/reports/brand` | getBrandList (report.store.js) | /reports/brands, /reports/brb-report | BrandReportListController |
| POST | `seller/v1/reports/brand/chart` | getBrandReportChart (report.store.js) | /reports/brands | BrandReportChartController |
| POST | `seller/v1/reports/brand/export` | getBrandExportFile (report.store.js) | /reports, /reports/brands | BrandReportExportController |
| GET | `seller/v1/reports/brb` | getBrbList (report.store.js) | /reports/brb-report | BrbReportListController |
| GET | `seller/v1/reports/brb/chart` | getBRBReportChart (report.store.js) | /reports/brb-report | BrbReportChartController |
| POST | `seller/v1/reports/brb/export` | getBRBExportFile (report.store.js) | /reports, /reports/brb-report | BrbReportExportController |
| GET | `seller/v1/reports/performance` | getPerformanceList (report.store.js) | /reports/performance-report | PerformanceReportListController |
| GET | `seller/v1/reports/performance/chart` | getPerformanceChart (report.store.js) |  (dashboard), /reports/performance-report | PerformanceReportChartController |
| POST | `seller/v1/reports/performance/export` | getPerformanceExportFile (report.store.js) | /reports, /reports/performance-report | PerformanceReportExportController |
| POST | `seller/v1/reports/product` | getProductList (report.store.js) | /reports/brb-report, /reports/product-report | ProductReportListController |
| POST | `seller/v1/reports/product/chart` | getProductReportChart (report.store.js) | /reports/product-report | ProductReportChartController |
| POST | `seller/v1/reports/product/export` | getProductExportFile (report.store.js) | /reports, /reports/product-report | ProductReportExportController |

#### Email campaigns & SMTP (9)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/email-campaigns/cancel/{param}` | cancelCampaign (email-campaign.store.js) | /email-campaigns | CancelEmailCampaignController |
| GET | `seller/v1/email-campaigns/list` | getCampaignList (email-campaign.store.js) | /email-campaigns | ListEmailCampaignController |
| GET | `seller/v1/email-campaigns/resend/{param}` | resendCampaign (email-campaign.store.js) | /email-campaigns | ResendEmailCampaignController |
| POST | `seller/v1/email-campaigns/store` | createCampaign (email-campaign.store.js) | /email-campaigns | StoreEmailCampaignController |
| DELETE | `seller/v1/smtp-connections/delete/{param}` | deleteSmtpConnection (email-campaign.store.js) | /email-campaigns | DeleteSmtpConnectionController |
| GET | `seller/v1/smtp-connections/list` | getSmtpConnectionList (email-campaign.store.js) | /email-campaigns | ListSmtpConnectionController |
| POST | `seller/v1/smtp-connections/store` | createSmtpConnection (email-campaign.store.js) | /email-campaigns | StoreSmtpConnectionController |
| POST | `seller/v1/smtp-connections/test` | testSmtpConnection (email-campaign.store.js) | /email-campaigns | TestSmtpConnectionController |
| PUT | `seller/v1/smtp-connections/update/{param}` | updateSmtpConnection (email-campaign.store.js) | /email-campaigns | UpdateSmtpConnectionController |

#### Messages (chat) (15)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| POST | `broadcasting/auth` | presenceChannelSetup (chat.store.js) | /messages | BroadcastController@authenticate |
| GET | `seller/v1/chats/contacts` | getContacts (chat.store.js) | /messages | IndexContactController |
| GET | `seller/v1/chats/contacts` | searchContact (chat.store.js) | /messages | IndexContactController |
| DELETE | `seller/v1/chats/contacts/tag` | deleteContactTag (chat.store.js) | /messages | DeleteContactTagController |
| POST | `seller/v1/chats/contacts/tag` | createContactTag (chat.store.js) | /messages | CreateContactTagController |
| GET | `seller/v1/chats/contacts/{param}/messages` | getAllMessages (chat.store.js) | /messages | HistoryMessageController |
| GET | `seller/v1/chats/find-contact/{param}` | findContact (chat.store.js) |  (dashboard), /affiliate/:id, /discovery-affiliates, /lists/affiliates/:id +3 more | FindContactController |
| GET | `seller/v1/chats/messages` | searchMessage (chat.store.js) | /messages | SearchMessageController |
| POST | `seller/v1/chats/messages` | sendMessage (chat.store.js) | /messages | CreateMessageController |
| POST | `seller/v1/chats/messages/find-page` | findPage (chat.store.js) | /messages | FindMessagePageController |
| GET | `seller/v1/chats/messages/saved` | getSavedMessages (chat.store.js) | /messages | IndexSavedMessageController |
| POST | `seller/v1/chats/messages/saved` | createSaveMessage (chat.store.js) | /messages | CreateSavedMessageController |
| POST | `seller/v1/chats/messages/send-as-bulk` | sendBulkMessage (chat.store.js) | /discovery-affiliates | SendBulkMessageController |
| POST | `seller/v1/chats/messages/send-as-list` | sendBulkMessageToList (chat.store.js) | /discovery-lists | SendListMessageController |
| PUT | `seller/v1/chats/read-contact-messages` | updateReadCount (chat.store.js) | /messages | ReadContactMessagesController |

#### Payments / billing (8)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/payments/charges` | getSubscriptionCharges (billing.store.js) | /payment-history/subscription | IndexChargeController |
| GET | `seller/v1/payments/commission-transactions` | getCommissionTransactions (billing.store.js) | /payment-history/commission | IndexCommissionTransactionController |
| GET | `seller/v1/payments/commission-transactions/{param}` | showCommissionTransaction (billing.store.js) | /payment-history/commission-invoice/:id | ShowCommissionTransactionController |
| GET | `seller/v1/payments/commission-transactions/{param}/export` | exportCommissionTransactionInvoice (billing.store.js) | /payment-history/commission-invoice/:id | CommissionTransactionExportController |
| GET | `seller/v1/payments/commission-transactions/{param}/items` | showCommissionTransactionInvoice (billing.store.js) | /payment-history/commission-invoice/:id | IndexCommissionTransactionItemsController |
| POST | `seller/v1/payments/commission-transactions/{param}/retry` | retryCommissionTransactions (billing.store.js) | /payment-history/commission | RetryCommissionTransactionController |
| GET | `seller/v1/payments/create-billing-portal` | createStripPortal (billing.store.js) | /settings | CreateBillingPortalController |
| GET | `seller/v1/payments/invoices` | getSubscriptionInvoices (billing.store.js) | /payment-history/subscription | IndexInvoiceController |

#### Subscription / plans (7)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/plans` | getPaymentPlans (manage-subscription.store.js) | /manage-subscription | IndexDefaultPlanController |
| GET | `seller/v1/plans/active` | getActivePlan (manage-subscription.store.js) | /manage-subscription | ShowActivePlanController |
| PUT | `seller/v1/plans/cancel` | cancelPlan (manage-subscription.store.js) | /manage-subscription | CancelPlanSubscriptionController |
| PUT | `seller/v1/plans/resume` | resumeSubscription (manage-subscription.store.js) | /manage-subscription | ResumePlanSubscriptionController |
| GET | `seller/v1/plans/subscriptions/status` | getSubscriptionStatus (manage-subscription.store.js) | /manage-subscription | SubscriptionStatusController |
| GET | `seller/v1/plans/{param}/subscribe` | requestPaymentLink (manage-subscription.store.js) | /manage-subscription | PlanSubscribeController |
| PUT | `seller/v1/plans/{param}/update` | updatePlan (manage-subscription.store.js) | /manage-subscription | UpdatePlanSubscriptionController |

#### Wallet (2)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| POST | `seller/v1/wallet/checkout/create` | createWalletTransaction (wallet.store.js) | /wallet | CreateWalletCheckoutController |
| GET | `seller/v1/wallet/transactions/list` | getWalletTransactionList (wallet.store.js) | /wallet/transaction-history | GetWalletTransactionsListController |

#### Settings, teams & roles, message templates (27)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| DELETE | `seller/v1/message-templates/delete/{param}` | deleteSavedMessage (settings.store.js) | /settings | DeleteMessageTemplateController |
| GET | `seller/v1/message-templates/list` | getSavedMessages (settings.store.js) | /email-campaigns, /settings | ListMessageTemplateController |
| POST | `seller/v1/message-templates/store` | createSavedMessage (settings.store.js) | /settings | StoreMessageTemplateController |
| PUT | `seller/v1/message-templates/update/{param}` | updateSavedMessage (settings.store.js) | /settings | UpdateMessageTemplateController |
| GET | `seller/v1/message-variables/list` | getMessageVariables (settings.store.js) | /discovery-affiliates, /discovery-lists, /email-campaigns, /settings | ListMessageVariableController |
| GET | `seller/v1/profile/sessions` | dataProvider (Security.vue) | /settings | SessionListController |
| DELETE | `seller/v1/profile/sessions/all` | terminateAll (Security.vue) | /settings | DeleteAllSessionsController |
| DELETE | `seller/v1/profile/sessions/{param}` | terminate (Security.vue) | /settings | DeleteSessionController |
| GET | `seller/v1/settings/client` | fetchSettings (settings.store.js) | (global: App/layout),  (dashboard), /affiliate/:id, /brand/:id +31 more | GetUserSettingController |
| POST | `seller/v1/settings/client/store` | updateSettings (settings.store.js) | (global: App/layout),  (dashboard), /affiliate/:id, /brand/:id +31 more | StoreUserSettingController |
| GET | `seller/v1/settings/notifications` | loadSettingNotification (settings.store.js) | /settings | GetUserNotificationSettingController |
| PUT | `seller/v1/settings/notifications/update` | changeSettingNotification (settings.store.js) | /settings | StoreUserNotificationSettingController |
| GET | `seller/v1/teams/invitations/list` | getInvitedTeamRequests (settings.store.js) | /settings | ListTeamInvitationController |
| PATCH | `seller/v1/teams/invitations/{param}/change-status` | changeTeamInvitationStatus (settings.store.js) | /settings | ChangeTeamInvitationStatusController |
| GET | `seller/v1/teams/my` | getMyTeam (settings.store.js) | /settings | MyTeamController |
| GET | `seller/v1/teams/permissions/list/{param}` | getTeamPermissions (settings.store.js) | /role-permission/:teamId/create|edit/:roleId | ListTeamPermissionController |
| GET | `seller/v1/teams/{param}/impersonate` | switchTeam (settings.store.js) | (global: App/layout) | GetImpersonationTokenController |
| POST | `seller/v1/teams/{param}/invitation` | sendInvitation (settings.store.js) | /settings | SendTeamInvitationController |
| POST | `seller/v1/teams/{param}/members/cancel/{param}` | cancelInvitation (settings.store.js) | /settings | CancelTeamMemberController |
| DELETE | `seller/v1/teams/{param}/members/delete/{param}` | removeTeamMember (settings.store.js) | /settings | DeleteTeamMemberController |
| GET | `seller/v1/teams/{param}/members/list` | getTeamMembers (settings.store.js) | /settings | ListTeamMemberController |
| PUT | `seller/v1/teams/{param}/members/update/{param}` | updateRole (settings.store.js) | /settings | UpdateTeamMemberController |
| POST | `seller/v1/teams/{param}/roles/create` | createTeamRole (settings.store.js) | /role-permission/:teamId/create|edit/:roleId | CreateTeamRoleController |
| DELETE | `seller/v1/teams/{param}/roles/delete/{param}` | deleteTeamRole (settings.store.js) | /settings | DeleteTeamRoleController |
| GET | `seller/v1/teams/{param}/roles/detail/{param}` | getTeamRoleDetail (settings.store.js) | /role-permission/:teamId/create|edit/:roleId | ShowTeamRoleController |
| GET | `seller/v1/teams/{param}/roles/list` | getTeamRoles (settings.store.js) | /settings | ListTeamRoleController |
| PUT | `seller/v1/teams/{param}/roles/update/{param}` | updateTeamRole (settings.store.js) | /role-permission/:teamId/create|edit/:roleId | UpdateTeamRoleController |

#### Notifications (6)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/notifications` | getNotificationList (notification.store.js) | (global: App/layout) | NotificationListController |
| PATCH | `seller/v1/notifications/read-all` | markAllNotificationAsRead (notification.store.js) | (global: App/layout) | ReadAllNotificationController |
| GET | `seller/v1/notifications/suggests` | getSuggestionList (notification.store.js) | (global: App/layout) | SearchSuggestionNotificationController |
| PATCH | `seller/v1/notifications/{param}/read` | markNotificationAsRead (notification.store.js) | (global: App/layout) | ReadNotificationController |
| PATCH | `seller/v1/notifications/{param}/toggle-bookmark` | toggleBookmarkStatus (notification.store.js) | (global: App/layout) | ToggleBookmarkNotificationController |
| POST | `seller/v1/user-fcm-tokens` | sendTokenToServer (pushNotificationService.js) | (global: App/layout) | CreateUserFcmTokenController |

#### Help center (tickets) (5)

| method | path | fn (file) | used on (route) | backend controller |
|---|---|---|---|---|
| GET | `seller/v1/tickets` | getTicketsList (ticket.store.js) | /support-tickets | IndexTicketController |
| POST | `seller/v1/tickets` | createTicket (ticket.store.js) | /support-tickets | CreateTicketController |
| GET | `seller/v1/tickets/{param}` | getTicketDetails (ticket.store.js) | /support-ticket-details/:id | ShowTicketController |
| PATCH | `seller/v1/tickets/{param}` | closeTicket (ticket.store.js) | /support-ticket-details/:id, /support-tickets | CloseTicketController |
| POST | `seller/v1/tickets/{param}/messages` | sendTicketMessage (ticket.store.js) | /support-ticket-details/:id | CreateTicketMessageController |

### 4.2 Live calls with no matching backend route (user-facing failures)

| # | Call | Where in the UI | Effect |
|---|---|---|---|
| 1 | `POST seller/v1/reveal-emails/user-types` (`affiliate.store.js › revealAffiliateEmail`) | Affiliate Discovery bulk bar → **Reveal Emails** → modal "Affiliate Email Reveals" → reveal (`affiliate-reveal-email-modal.vue`) | 404. The backend has no `reveal-emails/*` routes at all; the working flow is `reveal-information/process`. The other three `reveal-emails/*` functions are already dead code. |
| 2 | `POST seller/v1/partnership/brand/request-as-list` | My Lists card or list detail → **Send Partnership Request** → Brand tab (`partnership-bulk-request-to-list-modal.vue`) | 404. The backend only has `partnership/brand/request`; promote-to-list uses that endpoint with `type: affiliate_list`. |
| 3 | `POST seller/v1/partnership/product/request-as-list` | the same modal, Product tab | 404 |
| 4 | `DELETE seller/v1/profile/synced-shopify/{id}` (`social-service.store.js › deleteShopifyAccount`) | Profile → Manage accounts → Shopify account → Delete, and the onboarding step | 404. The backend only has `synced-amazon/{id}`, `synced-accounts/{id}` and the core integration `integrations/ainfluencer/shopify-account/delete`. |

No live call has a method mismatch. The dead-code mismatches are `GET user-type-lists` (backend is POST only) and `PATCH products/brands/{id}` (backend has GET and POST).

Other observations:

- Env vars referenced by live or reachable code but **missing from `spa-seller/.env`**:
  - `VUE_APP_ALGO_AUTH_BECOME_SELLER`, `VUE_APP_IMPERSONATE_USER`, `VUE_APP_AMAZON_MARKETPLACE_*`, `VUE_APP_AFFILIATE_SOCIAL_MEDIA_OAUTH_URL_API`, and the spa-affiliate `seller.store` vars. All of these sit on dead or affiliate-only paths.
  - `VUE_APP_*_PANEL_URI`, `*_PANEL_NAME` and `VUE_APP_S3_URL` exist only in `.env.development`; see §6.
- Backend `seller/v1` routes with **no live caller** in this SPA: 70 of 280. They include `api-keys/*`, 32 `integrations/ainfluencer/*` (core app), `reports/sale*`, `reports/performance/detail/{date}`, `promotions/{id}/update-status` and `/accessibility`, `partnership/*/count`, `products/{id}/sync-request`, `wallet/checkout/success|cancel` (Stripe return URLs), `auth/become-seller`, `teams/{id}` and `profile/payments/accounts`.

---

## 5. Dead code

### 5.1 Routes

- **Commented-out routes:** none in `spa-seller/src/router/index.js`.
- **Routes with no in-app inbound link**, reachable only by URL or redirect:
  - `/app/update-promotion/:id`, a legacy edit wizard;
  - `/app/partnerships`, `/app/reports` and `/app/payment-history` as parents (no default child; `payment-history` is only linked from the unused `profile-widget-items.js`);
  - `/app/wallet/success-payment|failed-payment` (Stripe return);
  - `/user/reset-password/:token` and `/user/email-verified/:token` (email links);
  - `/user/register-steps` (guard redirect).
- **Unused default route component:** `getLayoutViewRoute`'s default `shared-components/algo-views/app.layout.vue` is never used, because the seller passes its own layout. That also takes `widget-user-button/*` and `panel-selector.vue` out of the build in practice.

### 5.2 Unreachable seller files (25 + 22 mocks)

- `_mock_/` (11 scenario files) and `store/_mock_/**` (22 files). The mock import in `main.js` is commented out.
- `store/profile.store.js`: superseded by the shared `store/profile.store.js`.
- `utils/request-notification.js`.
- `components/dashboard/sections/dashboard-footer.section.vue`.
- `components/discovery/affiliates/affiliate-details-chart/{brand-chart,product-chart}.vue`.
- `components/discovery/lists/unlisted-affiliates/{unlisted-affiliates-lists,list-item.component,list-item-empty-state.component}.vue`: the Expanded Network tab of My Lists is commented out in `lists.page.vue`.
- `components/discovery/products/utility/product-card-template.component.vue`.
- `components/email-campaigns/modals/reveal-affiliate-email.modal.vue`: its import is commented out in `lists/modals/email-reveal/item.vue`.
- `components/partnership/products/contracts/partnership-product-contracts-table.vue` (a duplicate of the `.table.vue` file) and `components/partnership/utils/partnership-contracts-table-tab.component.vue`.
- `components/payment-history/invoice-modal/{invoice-modal.vue,invoice-modal-controller.js}`: replaced by the invoice page.
- `components/profile/profile-billing-tab.vue`.
- `components/promotions/create-update-promotion/{product-search.modal.vue, promotion-steps/promotion-step-4.vue}`.
- `components/promotions/utility/{checkbox-item.utility,checkbox-item.skeleton,new-promotion.skeleton,product-pill.utility.skeleton,search-panel.utility}.vue`.
- `components/report/utility/{bar-chart.panel,line-chart.panel,single-report-table}.vue`.

### 5.3 Shared code not used by the seller build

- **86 shared files are unreachable.** They include `algo-messenger` UI (6), `algo-components/profile/profile-old/*` (8), several `algo-form-ui` inputs (dropzone, editor, password, range slider, tree select, and others), `algo-notification/index.js`, `router/guards/permission.js`, `facebook/fAuth.js`, `amazon/aAuth.js`, `google/*.old.js`, `algo-manage-subscription/sections/*` and `Modals/admin-plan-creation.modal.vue`, and `algo-views/app.layout.vue`.
- **Bundled but never executed for a Seller** (role-gated at runtime):
  - `algo-components/profile/profile-manage-social-accounts/affiliate-profile-manage-social-accounts.component.vue` → `platforms-v2/*` (`AppInfo.isAffiliate`);
  - `algo-chat/components/ContactInfoView/DetailSectionSeller.vue`;
  - the `isAffiliate(userRole)` branches in chat;
  - `spa-affiliate/src/store/seller.store.js`, imported by the shared chat;
  - the affiliate branches of `Billing.vue` (`stripeOnboardRequest`), `report-page.vue` and `auth.layout.vue`.
- `algo-messenger/messenger.store.js` (legacy messenger) is instantiated by the `use-message` mixin, which creates an extra Pusher websocket connection. Only `onSendMessageToUser` is used; all 11 of its API functions are dead.

### 5.4 Unused API functions (dead call sites)

| method | path | fn | file | reason | backend |
|---|---|---|---|---|---|
| DELETE | `seller/v1/chats/messages/{param}` | deleteMessage | sc/algo-chat/store/chat.store.js | function never called from reachable code | DELETE seller/v1/chats/messages/{messageId} |
| GET | `seller/v1/chats/contacts/{param}` | showContact | sc/algo-chat/store/chat.store.js | function never called from reachable code | GET seller/v1/chats/contacts/{userTypeId} |
| POST | `broadcasting/auth` | <pusher-init> | sc/algo-messenger/messenger.store.js | role-gated: Pusher client created but no channel subscription -> no /broadcasting/auth call | GET|POST broadcasting/auth |
| DELETE | `seller/v1/chats/messages/{param}` | deleteMessage | sc/algo-messenger/messenger.store.js | function never called from reachable code | DELETE seller/v1/chats/messages/{messageId} |
| POST | `seller/v1/chats/messages/find-page` | findPageOfMessage | sc/algo-messenger/messenger.store.js | function never called from reachable code | POST seller/v1/chats/messages/find-page |
| GET | `seller/v1/chats/messages` | generalSearch | sc/algo-messenger/messenger.store.js | function never called from reachable code | GET seller/v1/chats/messages |
| GET | `seller/v1/chats/contacts` | getContactList | sc/algo-messenger/messenger.store.js | function never called from reachable code | GET seller/v1/chats/contacts |
| GET | `seller/v1/chats/contacts/{param}` | getContactshow | sc/algo-messenger/messenger.store.js | function never called from reachable code | GET seller/v1/chats/contacts/{userTypeId} |
| GET | `seller/v1/chats/contacts/{param}/messages` | getMessageHistory | sc/algo-messenger/messenger.store.js | function never called from reachable code | GET seller/v1/chats/contacts/{userTypeId}/messages |
| GET | `seller/v1/chats/messages/saved` | getSaveMessages | sc/algo-messenger/messenger.store.js | function never called from reachable code | GET seller/v1/chats/messages/saved |
| POST | `seller/v1/chats/messages/saved` | saveMessage | sc/algo-messenger/messenger.store.js | function never called from reachable code | POST seller/v1/chats/messages/saved |
| GET | `seller/v1/chats/contacts` | searchContactList | sc/algo-messenger/messenger.store.js | function never called from reachable code | GET seller/v1/chats/contacts |
| GET | `seller/v1/chats/contacts/{param}` | searchMessageHistory | sc/algo-messenger/messenger.store.js | function never called from reachable code | GET seller/v1/chats/contacts/{userTypeId} |
| POST | `seller/v1/chats/messages` | sendMessage | sc/algo-messenger/messenger.store.js | function never called from reachable code | POST seller/v1/chats/messages |
| GET | `<UNDEFINED:VUE_APP_ALGO_AUTH_BECOME_SELLER>` | affiliateBecomesSeller | sc/store/authentication.store.js | role-gated: panel switcher calls it only when switching TO the Seller panel from another app; in seller app the Seller panel is the active one (returns early) | no route |
| POST | `seller/v1/auth/verification-code` | resendEmailVerifycode | sc/store/authentication.store.js | function never called from reachable code | POST seller/v1/auth/verification-code |
| POST | `seller/v1/auth/email-verify` | verifyEmail | sc/store/authentication.store.js | function never called from reachable code | POST seller/v1/auth/email-verify |
| POST | `seller/v1/auth/mobile-verification-code` | verifyMobileNumber | sc/store/authentication.store.js | function never called from reachable code | POST seller/v1/auth/mobile-verification-code |
| PUT | `seller/v1/profile/change-mobile` | verifyOTPChangePhone | sc/store/authentication.store.js | function never called from reachable code | PUT seller/v1/profile/change-mobile |
| POST | `<UNDEFINED:VUE_APP_IMPERSONATE_USER>` | impersonateUser | sc/store/authorization.store.js | function never called from reachable code | no route |
| PUT | `seller/v1/profile/change-password` | changePassword | sc/store/profile.store.js | function never called from reachable code | PUT seller/v1/profile/change-password |
| GET | `<UNDEFINED:VUE_APP_AMAZON_MARKETPLACE_SHOW>` | getAmazonMarketplaceDetails | sc/store/profile.store.js | function never called from reachable code | no route |
| GET | `<UNDEFINED:VUE_APP_AMAZON_MARKETPLACE_LIST>` | getAmazonMarketplaceList | sc/store/profile.store.js | function never called from reachable code | no route |
| POST | `seller/v1/profile/request-change-password` | requestChangePassword | sc/store/profile.store.js | function never called from reachable code | POST seller/v1/profile/request-change-password |
| POST | `seller/v1/teams/{param}/roles/assign/{param}/{param}` | assignTeamRole | sc/store/settings.store.js | function never called from reachable code | POST seller/v1/teams/{teamId}/roles/assign/{roleId}/{teamMemberId} |
| GET | `seller/v1/message-variables/detail/{param}` | getMessageVariable | sc/store/settings.store.js | function never called from reachable code | GET seller/v1/message-variables/detail/{id} |
| GET | `seller/v1/message-templates/detail/{param}` | getSavedMessage | sc/store/settings.store.js | function never called from reachable code | GET seller/v1/message-templates/detail/{id} |
| GET | `<UNDEFINED:VUE_APP_AFFILIATE_SOCIAL_MEDIA_OAUTH_URL_API>` | syncSocialMediaAccount | sc/store/social-service.store.js | role-gated: only from platforms-v2/platform.component.vue, rendered only when AppInfo.isAffiliate | no route |
| GET | `<UNDEFINED:VUE_APP_AFFILIATE_ACTIVE_PARTNERSHIPS_BRANDS_LIST_API>` | getSellerActiveBrandPartnershipList | spa-affiliate/src/store/seller.store.js | function never called from reachable code | no route |
| GET | `<UNDEFINED:VUE_APP_AFFILIATE_ACTIVE_PARTNERSHIPS_PRODUCTS_LIST_API>` | getSellerActiveProductPartnershipList | spa-affiliate/src/store/seller.store.js | function never called from reachable code | no route |
| GET | `<UNDEFINED:VUE_APP_ALGO_SELLER_INFO_API>/{param}` | getSellerInfo | spa-affiliate/src/store/seller.store.js | role-gated: spa-affiliate store bundled via shared chat; only called in isAffiliate(userRole) branches -> never for Seller; env vars also undefined in seller build | no route |
| GET | `<UNDEFINED:VUE_APP_ALGO_SELLER_USER_LIST_API>` | getSellerList | spa-affiliate/src/store/seller.store.js | role-gated: spa-affiliate store bundled via shared chat; only called in isAffiliate(userRole) branches -> never for Seller; env vars also undefined in seller build | no route |
| POST | `seller/v1/partnership/brand/request` | sendPartnershipBrandRequestToSeller | spa-affiliate/src/store/seller.store.js | function never called from reachable code | POST seller/v1/partnership/brand/request |
| GET | `seller/v1/reveal-emails/my` | getMyRevealedEmails | seller/store/affiliate.store.js | function never called from reachable code | no route |
| POST | `seller/v1/reveal-emails/user-type-list/{param}` | revealAffiliateListEmail | seller/store/affiliate.store.js | function never called from reachable code | no route |
| POST | `seller/v1/reveal-emails/wallet/user-type-list/{param}` | revealWalletAffiliateListEmail | seller/store/affiliate.store.js | function never called from reachable code | no route |
| GET | `seller/v1/profile/payments/accounts` | getConnectedPaymentAccounts | seller/store/billing.store.js | function never called from reachable code | GET seller/v1/profile/payments/accounts |
| GET | `seller/v1/plans` | getPaymentPlans | seller/store/billing.store.js | function never called from reachable code | GET seller/v1/plans |
| GET | `seller/v1/payments/transactions` | getTransationsList | seller/store/billing.store.js | function never called from reachable code | no route |
| GET | `seller/v1/plans/{param}/subscribe` | requestPaymentLink | seller/store/billing.store.js | function never called from reachable code | GET seller/v1/plans/{id}/subscribe |
| GET | `seller/v1/reports/performance/chart` | getLinearChart | seller/store/dashboard.store.js | function never called from reachable code | GET seller/v1/reports/performance/chart |
| GET | `seller/v1/email-campaigns/show/{param}` | getCampaign | seller/store/email-campaign.store.js | function never called from reachable code | GET seller/v1/email-campaigns/show/{id} |
| GET | `seller/v1/smtp-connections/detail/{param}` | getSmtpConnection | seller/store/email-campaign.store.js | function never called from reachable code | GET seller/v1/smtp-connections/detail/{id} |
| GET | `seller/v1/user-type-lists/{param}/membership` | getAffiliateMembershipList | seller/store/lists.store.js | function never called from reachable code | no route |
| GET | `seller/v1/user-type-lists` | getAllLists | seller/store/lists.store.js | function never called from reachable code | no route |
| GET | `seller/v1/user-type-lists/drop-down-unlisted-affiliates` | getMyUnlistedAffiliatesListDropdown | seller/store/lists.store.js | function never called from reachable code | GET seller/v1/user-type-lists/drop-down-unlisted-affiliates |
| POST | `seller/v1/partnership/brand/bulk-request` | sendPartnershipBulkBrandRequest | seller/store/partnership.store.js | function never called from reachable code | no route |
| POST | `seller/v1/partnership/product/bulk-request` | sendPartnershipBulkProductRequest | seller/store/partnership.store.js | function never called from reachable code | no route |
| GET | `seller/v1/products` | searchProductList | seller/store/product.store.js | function never called from reachable code | GET seller/v1/products |
| PATCH | `seller/v1/products/brands/{param}` | updateBrandDetail | seller/store/product.store.js | function never called from reachable code | no route |
| PUT | `seller/v1/profile/change-password` | changePassword | seller/store/profile.store.js | defining file not reachable from main.js | PUT seller/v1/profile/change-password |
| GET | `seller/v1/profile` | getProfileInfo | seller/store/profile.store.js | defining file not reachable from main.js | GET seller/v1/profile |
| GET | `seller/v1/profile/questions` | getQuestionList | seller/store/profile.store.js | defining file not reachable from main.js | GET seller/v1/profile/questions |
| POST | `seller/v1/profile/questions` | postQuestionList | seller/store/profile.store.js | defining file not reachable from main.js | POST seller/v1/profile/questions |
| POST | `seller/v1/profile/request-change-password` | requestChangePassword | seller/store/profile.store.js | defining file not reachable from main.js | POST seller/v1/profile/request-change-password |
| POST | `seller/v1/profile` | updateProfileInfo | seller/store/profile.store.js | defining file not reachable from main.js | POST seller/v1/profile |
| GET | `seller/v1/promotions/{param}` | getPromotionDetail | seller/store/promotions.store.js | function never called from reachable code | GET seller/v1/promotions/{promotionId} |

### 5.5 Dead or inert UI inside live components

- The "Remember for 30 days" checkbox on login is unbound.
- The Amazon social-login button is commented out on login and register.
- The dashboard YouTube welcome modal call is commented out.
- Wallet "Withdraw" shows "Coming soon...".
- The wallet T&C link is `#`.
- Google and Facebook account blocks in Manage accounts are mostly commented out.
- The Expanded Network tab in My Lists is commented out.
- The promotion step-2 hint "If choosing private, you must select affiliates in next step" refers to a step that no longer exists; the next step is the success screen.

---

## 6. Uncertainties and caveats

1. **Environment values.** `.env` and `.env.development` are git-ignored, yet present in the snapshot. Endpoints were resolved from the snapshot `.env`, as a production build would load it. The production agent's actual files are unknown.
   - Values defined **only** in `.env.development` (not loaded by `npm run build`): `VUE_APP_SELLER/AFFILIATE/BUYER/ADMIN_PANEL_URI|NAME`, `VUE_APP_S3_URL`, `VUE_APP_GOOGLE_RECAPTCHA_V2` (which has a hard-coded fallback) and `VUE_APP_API_BASE_URL` (also in `.env`).
   - If production really lacks them:
     - the panel switcher breaks, because `getPanelObject` calls `panel.name.toLowerCase()` on `undefined`;
     - brand landing links become `undefined/seller/brand/<slug>`;
     - the affiliate landing-page link breaks;
     - S3 placeholder avatars break.
   - The agent may provide these values in another way. Verify against the live site.
2. **Reachability is static.** The import graph was walked with Vue-aware pruning of imported-but-unused components; template usage and string-based `:is` were both checked. Runtime conditions are only partly modelled: the role gates were reviewed by hand, but plan, feature and permission (`$can`) checks were not.
   - "Live" means the function is reachable and has at least one call site in reachable code. It does not mean every user can trigger it.
   - Route attribution (`routes` in the JSON) counts any route whose component tree contains a caller. Shared modals can therefore over-attribute.
3. **Backend matching** used the v1.5.27 route dump, treating `{x}` as any single segment. Middleware and permission rules were not evaluated. A matched route can still reject the request (403 for team permissions, 402 for plan limits, 499 for missing payment profile).
4. **Enum labels** (statuses, discount types, commission-transaction statuses, email campaign statuses, plan features) come from `system/basic-info` at runtime and are not in the front-end source. The promotion badge strings "Active / Inactive / Expired / Waiting for approval" are hard-coded comparisons in `promotion-detail.panel.vue`.
5. **Plan rules** quoted in §3.15 come from front-end copy, such as Pro ≥20% commission, Ultimate for promotions and custom commission, and the 3.5% fee. The backend may differ.
6. **The onboarding step list is backend-driven.** Whether `verify_mobile`, `questions` and `profile` are enabled for sellers, and in what order, depends on `show-register-steps`. The Shopify-only blocker in §3.3 depends on how the backend sets `is_passed` after a Shopify connect.
7. **Wallet success and failure pages** are assumed to be Stripe return targets set by the backend (`wallet/checkout/success|cancel`). The front end never links to them.
8. **The email campaign "schedule" field** is not sent by the client. The backend could still schedule by another rule, for example approval time.
9. **Amazon vs Shopify:** the promotion wizard asks for an Amazon seller account (`profile/profiles`). Whether promotions work for Shopify-only sellers was not verified.
