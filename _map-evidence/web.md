# Brand web app (src/web): live-code inventory

Snapshot: production `master`, at `core/frontend/web (master snapshot)` (`src/` inside). All paths below are relative to `src/web/src/` unless they start with the snapshot.

## How this was built (and what it can't see)

- **Routes**: the single `RouteObject[]` in `lib/routes/index.tsx`, with comments stripped. It mounts `lib/routes/createAd.tsx` under `/campaign`. `/affiliate/*` goes to `components/pages/affiliate` → `AffiliateContent.tsx`, which switches on the first path segment. The router is set up by `routes.tsx` (`useRoutes(routes)`), which is rendered by `App.tsx`, which is rendered by `index.tsx`.
- **Reachability** (`(generator script)`): a walk of the import graph from `index.tsx`. It strips comments and resolves relative imports and the tsconfig `baseUrl: src`. It drops imports whose local name is never used outside comments and strings, and it resolves names re-exported through barrel files.
  - There is no `React.lazy` or dynamic page import. The only `import()` calls are for i18n JSON and web-vitals.
  - Result: **1250 of 2046 code files are reachable** and 796 are dead. The lists are in `(generator script)` (`reachable`, `unreachable`, plus the import edges and which symbols each file uses).
- **Calls** (`(generator script)` → `finalize.py`): every `API.<verb>(...)` in `lib/services/*`, plus direct `axios` / `fetch` calls elsewhere and chat-host `fetch` calls.
  - A service function counts as **live** only if a reachable file imports it *and* uses the name.
  - Pages are worked out by climbing the reverse import graph from each caller up to the routed page component.
  - Output: `(generator script)`.
- **Inbound links** (`(generator script)` → `links.json`): every string or template literal starting with `/` in reachable code, matched against the route table. Relative `navigate("pay?…")` calls and `${route_base}/…` strings were checked by hand.
- **Limits**:
  - This is static analysis. A component that is reachable but sits behind a runtime condition (a feature flag, plan or role) still counts as live.
  - "Pages" for a call are where the calling component is mounted. Shared modals inflate that list. For example, the influencer quick-view modal makes `/account/:id` calls appear under `/search` and `/favorite-influencers`.
  - Links that arrive from emails, the backend or other apps can't be seen in this code.

## 0. Global behaviour (every page under the logged-in layout)

- **`DashboardLayout`** (`components/layouts/dashboard/index.tsx`) wraps every route under `/` in `AuthLogin` (`components/hoc/auth`):
  - With no login token, it redirects to `/login`.
  - With a token but onboarding not complete (`brand_onboarding.is_complete` false and no `brand_information`), it redirects to `/onboarding`.
- **Header** (`layouts/dashboard/header`):
  - Notifications popover: `v2/notifications`, plus read, read-all, delete and delete-all.
  - Plan tracker (`subscriptions/getCurrentPlanInfo`), team switcher, tutorial videos, link to Help Desk.
  - Profile menu, which links to `/profile`, `/profile/points`, `/profile/transactions`, `/profile/translated-contents`, `/profile/email-notifications`, `/subscription-plans`, `/team-members`, `/favorite-influencers`, `/influencer-content` and `/packages`.
  - Top banner (rendered from the header): "add funds" → `/funds`.
- **Sidebar** (`layouts/dashboard/sidebar/model.ts`, the source of truth for the menu):
  - Items: `/` Dashboard, `/search` Find influencers, `/campaigns`, `/new-offersV2` Inbox, `/favorite-influencers`, `/influencer-content`, `/affiliate`, `/analytics`, `/scheduler`, `/our-apps`, `/packages`. The `/offers` and `/subscription-plans` entries are commented out.
  - "Upgrade plan" card → `/subscription-plans`, shown when the plan is not Ultimate and not Free.
  - "Book strategy call" → external bookme.brandid.app link.
  - Unread badge: polls `GET {CHAT}/message/unread-count`. The query runs every 1 s but is throttled inside to about one call every 50 s when nothing changes.
  - **Legacy Twilio client is still live**: the sidebar calls `useTwilioClient()` from `components/pages/singleOffer/model.ts`. That connects `@twilio/conversations`, listens for `messageAdded`, and refreshes its token through `POST v2/refresh-twilio-token`.
- **App level**:
  - `chatgram-widget` support chat is initialised in `App.tsx`.
  - Shopify-installed users are forced to `/oauth/set-password` (`isUserNeedToSetShopifyPassword`).
  - Sentry, GTM, Clarity and GA are loaded.
  - On any 401, axios refreshes the token directly with `POST v2/auth/refresh-token` (in the `_config.ts` interceptor) and then reloads the page.
  - **HTTP 402 = plan feature limit.** It opens `FeatureLimitModal` (feature keys are listed in section 6).

### Guards

| Guard | File | Behaviour |
|---|---|---|
| AuthLogin (layout) | `components/hoc/auth` | All routes under `/`: needs a token and completed onboarding (see above). |
| ProtectedRoute | `components/common/protectedRoute` | Team members (`currentTeam` set) are redirected to `/`. Used on `/profile` (index), `/profile/change-password`, `/profile/email-notifications` and `/influencer-content`. |
| AuthGuard | `components/common/authGuard` | Logged-in users are redirected to `/`. Used on `/login`, `/register`, `/forgot-password` and `/reset/password/:token`. |
| PhoneNumberVerificationMiddleware | `components/common/middlewares/phoneNumberVerification` | **Dead.** Imported in `routes.tsx` and `layouts/dashboard/index.tsx` but never rendered, so phone verification is **not enforced** anywhere in the UI. |

## 1. Live URLs by feature area

Legend for "Inbound links":
- **nav**: the sidebar or header menu links here.
- **in-app**: other pages link here through Link, navigate or href.
- **none (direct/email)**: routed but nothing in the code links to it. It is reachable only by typing the URL, from an email, or from a backend redirect.
- **self-only**: the only links come from inside the page itself.

All rows in 1.1 to 1.9 use the AuthLogin layout guard unless the row says otherwise.

### 1.1 Home and discovery

| URL | Page component | Extra guard | Inbound links | Purpose |
|---|---|---|---|---|
| `/` | `pages/dashboardV3` | – | nav | Brand home: KPIs, recent inbox messages (`v2/chatbox/chats`), scheduler preview, team, tutorial videos, published content, CTAs to create a campaign (`/affiliate/select_campaign_type`), seeding and packages. |
| `/search` | `hoc/findInfluencers` → `pages/searchV3` | – | nav, in-app (many) | **Find influencers** (`POST v2/elastic/extended/search`). Invite to a campaign (`v2/user/invitations`, `v2/temp-invitations`), add to a seeding campaign (`product-seedings/{id}/influencers`), favourites, quick-view profile, reveal username (quota or wallet). |
| `/find/influencer-explorer` | `hoc/findInfluencers` → `pages/influencer-explorer` | – | in-app: only the "back" link in `influencerProfileNew/dossier/DossierProfile.tsx` | Older explorer on `POST v2/elastic/search`. Effectively superseded by `/search`. |
| `/find/influencer-explorer/custom-search` | `pages/custom-search` | – | **none (direct)** | Older custom search on `v2/elastic/search`. |
| `/account/:id` | `pages/influencerProfile` | – | in-app (cards, popovers, favourites, reviews; about 20 sites) | Influencer profile as the brand sees it: all-data, accounts, published content, add to favourites, invite to a campaign, reveal username, like feedback, hit profile views. |
| `/accountnew/:id` | `pages/influencerProfileNew` | – | self-only (similar-creator links inside the dossier) | New "dossier" design of the influencer profile. Not linked from anywhere else. |
| `/favorite-influencers` | `pages/favoriteInfluencersV2` | – | nav, in-app | Favourite lists: create, rename and delete lists, add or remove influencers, invite to a campaign. |

### 1.2 Campaigns

| URL | Page component | Extra guard | Inbound links | Purpose |
|---|---|---|---|---|
| `/campaigns` | `pages/campaignsV2` | – | nav, in-app (notification redirect for `ad*` events, campaign flows) | List of all campaigns, filterable by type (paid collaboration, seeding, affiliate). Actions: delete, duplicate, edit, copy public link (`/view-campaign/:id` or `/view-campaign-affiliate/:id`), open the influencer pipeline. Edit seeding → `/affiliate/product_seeding_wizard?edit=`. New campaign → `/affiliate/select_campaign_type`. |
| `/campaigns/affiliate-influencers/:id` | `pages/affiliateInfluencers` | – | in-app (`campaignsV2`, `affiliateDashboard2/CampaignsTab`) | Influencers in an affiliate campaign. Chat drawer (`v2/chatbox/chats/{id}`, `POST v2/chatbox/chats/messages`, chat-host conversation history), products modal. Also embedded in `/affiliate/dashboard?tab=affiliates`. |
| `/campaigns/seeding-influencers/:id` | `pages/seedingInfluencers` | – | in-app (`campaignsV2`) | Pipeline for a product-seeding (gifting) campaign: list, move stage (`PUT product-seedings/{id}/influencers/{inf}/status`), outreach message, CSV export, chat drawer. |
| `/create-campaign` | `pages/createAd` | – | in-app (`campaignsV2`, `selectCampaignType` "Paid Collaboration", the select-campaign modal on profiles) | **Paid-collaboration campaign wizard.** See section 2.2. |
| `/campaign/*` | `common/createAd/layout` (`route_base=/campaign`, `mode=create`) | – | in-app: only `/campaign/objectives/:id` from the "Edit ad" link in `influencerProfile(/New)/parts/modal/selectCampaignModal/.../buttons` | **Legacy campaign wizard** (objectives → marketplace → post-content → publish-terms → confirm → discover). Used today only to edit an existing ad from an influencer profile. See section 2.3. |
| `/analytics` | `pages/analytics` | – | nav | Analytics on published content per campaign and influencer (`v2/campaigns/published/analysis*`), recommended influencers, leave a review. |
| `/publish-jobs/:id/analytics` | `pages/publishJobs` | – | in-app (scheduler grid "analytics" action) | Analytics for a single published post (`v2/publish-jobs/{id}`). |
| `/scheduler` | `pages/scheduler` | – | nav, notification redirect (`publish*` events), dashboard | Calendar and list of scheduled and published posts (`v2/publish-jobs`, `v2/schedules`). |
| `/content-library` | `pages/contentLibrary` | – | in-app (scheduler) | Brand media library (`v2/medias`, delete). |

### 1.3 Inbox, offers and payment of offers

| URL | Page component | Extra guard | Inbound links | Purpose |
|---|---|---|---|---|
| `/new-offersV2` | `hoc/newOffersV2` → `pages/inboxV37` | – | nav (labelled "Inbox", with unread badge) | **Current inbox.** Room list (`v2/chatbox/chats`), campaign filter (`v3/brand/inbox/campaigns`), bulk message. |
| `/new-offersV2/:id` | `pages/inboxV37` (same component, so the room list is not remounted) | – | in-app (notifications, dashboard, content, scheduler, reviews, payment pages, `/offers` rows, …) | Chat with one influencer over socket.io (section 3). Offer actions: accept, reject (`v2/offers/{id}/change-status`), counter (`…/counter`), change publish date (`…/update-start`), revision (`v2/offer-revision/`), approve or disapprove content (`offer/v2/{id}/approve`, `…/disapprove`), insight request, Shopify fulfilment and tracking, affiliate link generation, gifting tab, collaboration form (`v2/forms/*`), report influencer, leave review. |
| `/new-offersV2/:id/pay` | `pages/checkoutV2` | – | in-app: relative `navigate("pay?BasketData=…")` from the inbox "Pay" action (`InboxV37ChatArea.makePaymentHandler`) and from the publish-date modals | **Offer checkout.** Basket (`finance/current-user/basket*`), card or wallet or PayPal, coupon, insurance add-on, `POST finance/payment` and `finance/payment/entity`, abandoned-checkout tracking. |
| `/new-offersV2/:id/payment-successful` | `pages/paymentSuccessful` | – | in-app (`checkoutV2` and `payment` after success) | Payment confirmation. Shows other influencers awaiting payment and related influencers. |
| `/new-offersV2/deals` | `pages/deals/indexV2` | – | self-only (tab inside the deals page's own `newOffersV2/parts/customHoc`) | Closed-deals grid (`v2/offers?action=deals_closed`). Nothing in the nav or inbox links here. |
| `/new-offersV2/reviews` | `pages/ReviewAndRating` | – | **none (direct)** | Ratings and reviews grid (`v2/reviews`). |
| `/new-offersV2/favorite` | `pages/favoriteInfluencerOffer/indexV2` | – | **none (direct)** | Old favourites grid. |
| `/offers` | `hoc/offerV2` → `pages/offers` | – | in-app: `href="/offers"` in "invite sent" modals (`favoriteInfluencerTabs/inviteInfluencerAd`, `influencerProfile(/New)/parts/modal/inviteInfluencerAd`, `createAd/.../afterCreateCampaignModal/pendingInvite`), `paymentSuccessful/paymentInfluencers`, and the notification fallback when an offer event has no id (`header/notification/model.ts`). Not in the sidebar (commented out). | **Legacy offers list** (`v2/offers` datagrid, bulk message). Rows link to `/new-offersV2/:id`. Still live and still linked. |
| `/offers/deals`, `/offers/favorite` | `pages/deals`, `pages/favoriteInfluencerOffer` | – | self-only (tabs of `/offers`) | Legacy closed-deals and favourites grids. |
| `/offers/reviews` | `pages/ReviewAndRating` | – | **none** | Same as `/new-offersV2/reviews`. |
| `/offers/:id` | `pages/singleOffer` | – | in-app (`transactions2/WalletView` order link) | **Redirect only.** Immediately does `navigate('/new-offersV2/:id', {replace:true})`. The old UI is commented out and would show "Under Maintenance". |
| `/offers/:id/pay` | `pages/payment` | – | **none** (the old relative "pay" links came from the dead singleOffer UI) | Old checkout. The component is still embedded as payment options inside some inbox modals. |
| `/offers/:id/pay-v2` | `pages/checkoutV2` | – | **none** | Same checkout as `/new-offersV2/:id/pay`. |
| `/offers/:id/payment-successful` | `pages/paymentSuccessful` | – | **none** | Old success URL. |
| `/inbox-v37-demo`, `/inbox-v37-demo/:id` | `RedirectInboxV37Demo` (inline in `lib/routes/index.tsx`) | – | **none** | Redirects to `/new-offersV2(/:id)`. |
| `/influencer-content` | `pages/contents2` | **ProtectedRoute** (team members bounced to `/`) | nav, header menu, dashboard | **Content delivered by influencers**: view, approve or disapprove, download (axios blob `v2/published-contents/download/{id}`), request a translation, copy the public link `/view-contents/:code`. |

### 1.4 Affiliate and seeding (see section 2.1 for steps)

| URL | Page component | Extra guard | Inbound links | Purpose |
|---|---|---|---|---|
| `/affiliate/dashboard` | `pages/affiliateDashboard2` | – | nav (`/affiliate` → AffiliateContent redirects to `/affiliate/dashboard`), many in-app | **Affiliate dashboard**, with tabs via `?tab=`:<br>• programs: embeds `affiliate/parts/syncAffiliate` (connect Amazon or Shopify)<br>• campaigns<br>• affiliates: embeds `affiliateInfluencers`<br>• performance<br>• payouts: card sync<br>"Advanced dashboard" opens `seller.ainfluencer.com/app/...` through SSO (`user-third-party/ainfluencer-affiliate/v1/sso/authorize-url`). |
| `/affiliate/dashboard2` | redirect | – | none | Redirects to `/affiliate/dashboard`. |
| `/affiliate`, `/affiliate/<step>` | `pages/affiliate` → `AffiliateContent` | – | nav and in-app (see section 2.1) | Affiliate and seeding setup steps. This component is also the OAuth popup landing page: it reads `account_id`, `status` and `error` from the query and posts `AMAZON_AFFILIATE_SYNC_COMPLETE` to the window that opened it. |

### 1.5 Money: wallet, cards, transactions, subscription

| URL | Page component | Extra guard | Inbound links | Purpose |
|---|---|---|---|---|
| `/funds` | `pages/newFunds` | – | in-app (top banner, reveal-username modal, the campaign-created modal `?amount=`, cards, wallet view, payment success) | **Add funds to wallet**: card (`POST finance/current-user/wallets`) or PayPal (`v2/paypal/charge`). Minimum amount is 1. Can return to `/new-offersV2/:offerId`. |
| `/cards` | `pages/newCards` | – | in-app (`transactions2` billing and wallet views, `creditCardNew`) | Saved cards (`finance/current-user/cards`): delete, make default. |
| `/cards/create` | `pages/creditCardNew` | – | in-app (`/cards`) | Add a card with Stripe Elements (`POST finance/current-user/cards`). |
| `/profile/transactions` | `hoc/profile` → `pages/transactions2` | – | header menu, profile tab, plan-change modal (`?tab=billing`) | Wallet, orders and transactions, billing and **subscription management**: change plan, cancel (with a cancellation-survey form `v2/forms/*`), resume, receipts. |
| `/transactions2` | `pages/transactions2` | – | **none** | Same page outside the profile shell. |
| `/subscription-plans` | `pages/subscriptionPlans` | – | in-app (≈25 upgrade CTAs, header plan tracker, sidebar card) | Plan comparison and purchase (`subscriptions/plans`, `subscriptions/subscribe/{plan}` returns a payment link, `subscriptions/update/{id}`). |
| `/subscription-plans/payment-result` | `pages/subscriptionPlans/paymentResult` | – | **none (return URL from the payment provider)** | Result page after the external subscription checkout. |
| `/packages` | `pages/packages` | – | nav, header, dashboard | "Fully managed" agency packages. A static marketing page with booking links. |
| `/our-apps` | `pages/ourApps` | – | nav | Static "brandID family" page (sister apps). |

### 1.6 Profile and account

| URL | Page component | Extra guard | Inbound links | Purpose |
|---|---|---|---|---|
| `/profile` | `hoc/profile` → `pages/userInfo2` | **ProtectedRoute** | header (avatar and menu), profile tab | Brand and company profile (`PUT v3/brand/profile`, logo upload), change password, delete account (`DELETE v2/user/current-user`). |
| `/profile/points` | `pages/points2` | – | header menu, profile tab | Brand score and tier (Bronze, Silver, Gold, Platinum), score activities, reviews given. |
| `/profile/translated-contents` | `pages/translation2` | – | header menu, profile tab, content modals, payment success | Translation requests and downloads (axios blob `v2/translation-requests/{id}/…/download`). |
| `/profile/email-notifications` | `pages/emailNotificationSettings` | **ProtectedRoute** | header menu, profile tab | Email notification categories (`v2/notify-settings/categories`). |
| `/profile/change-password` | `pages/changePassword` | **ProtectedRoute** | **none** (tab commented out in `hoc/profile/tabs`) | Change password. The same action is also available on `/profile`. |
| `/profile/contents` | `pages/contents2` | – | **none** (tab commented out) | Same as `/influencer-content` but **without** ProtectedRoute. |
| `/profile/favorites` | `pages/favoritesInfluencers` | – | **none** | Old favourites page. |
| `/profile/reviews` | `pages/ReviewAndRating` | – | **none** (tab commented out) | Reviews grid. |
| `/team-members` | `pages/teamMembersManagement` | – | header menu and team switcher | Invite and remove team members (`v2/teams/brand/members*`). Gated by the `seats` limit. |
| `/notifications` | `pages/notifications` | – | header "see all" | Notification list. Deleting an integration is possible from here (`DELETE v2/integrations/{id}`). |
| `/instagram-account/:id` | `pages/connectAccount` | – | **none** | **Stub.** The form only does `console.log` and calls no API. Effectively dead. |
| `*` | `pages/404` | – | – | Not found (only for logged-in users; logged-out users are redirected to `/login` first). |

### 1.7 Auth, sign-up and onboarding (public, no layout)

| URL | Page component | Guard | Inbound links | Purpose |
|---|---|---|---|---|
| `/login` | `pages/login2` | AuthGuard | many (logout, layout redirect) | Email or password with reCAPTCHA, or Google (`v2/3rd-party/verify-auth`). Routing after login:<br>• influencer accounts → `/influencer-landing`<br>• unverified email → `/verify`<br>• suspended account → appeal modal (`v2/brand/suspension-appeal`)<br>• otherwise the onboarding check (`v3/brand/on-boarding`), then `/` or `/onboarding` |
| `/register` | `pages/register2` | AuthGuard | in-app (login, verify pages) | Brand sign-up (`v2/auth/register`). **Phone number is required** (E.164 format). UTM params are captured. Then → `/verify`. |
| `/verify` | `pages/verifyEmail2` | – | in-app (`navigate` with router state from login and register) | Enter the email code (`v2/auth/email/verify`), resend. Brand → `/onboarding`; influencer → `/influencer-landing`. |
| `/verifybylink/:token` | `pages/verifyByLink2` | – | **none (email link)** | Brand email-confirm link (`v2/email/confirm`) → `/onboarding`. |
| `/auto-verify?token=` | `pages/verifyInfluencerByLink` | – | **none (email link)** | Influencer email-confirm link → `/influencer-landing`. |
| `/verify-phone-number` | `pages/verifyPhoneNumber2` | – | **none**, and it requires router state (otherwise it redirects to `/register`) | OTP page (`v2/phone-verification/attempt`, `/verify`) with a "Skip" button. **Effectively unreachable**: nothing navigates here. |
| `/forgot-password` | `pages/forgotPassword2` | AuthGuard | in-app (login form) | Request a reset email. |
| `/reset/password/:token` | `pages/resetPassword2` | AuthGuard | **none (email link)** | Set a new password. |
| `/change-email` | `pages/changeEmail` | – | **none** | Change the email address of an unverified account (`PUT v2/user/profile` with a bearer token). Nothing links here. |
| `/oauth/set-password` | `pages/shopifyCreatePassowrd` | – | App-level forced redirect for Shopify-installed users | Shopify app users set a password (`v2/user/set-password`), then → `/onboarding`. |
| `/onboarding` | `pages/onboarding` | – (AuthLogin sends users here) | in-app (AuthLogin, verify pages) | Brand onboarding. **Only two steps are active** (business name, then location); the rest are commented out in `onboarding/model.ts`. Uses `v3/brand/on-boarding` and `v2/funnels`. The finish CTA goes to `/affiliate/select_campaign_type`. |
| `/tmp-link?token=` | `pages/tempLogin` | – | **none (admin or email magic link)** | Token login (`POST tmp-link/verify`), including team-context tokens. |
| `/teams/members` | `pages/memberRegisteration` | – | **none (team-invite email)** | An invited team member registers (`v2/teams/brand/members/register`). |
| `/influencer-landing` | `pages/InfluencerLanding` | – | in-app (login, register, verify) | Shown to *influencer* accounts that land on the brand app. Points them to the mobile app. |
| `/app-download` | `pages/downloadApp` | – | **none** | Page to download the mobile app. |

### 1.8 Influencer-facing invitations and guest pages (public)

| URL | Page component | Inbound links | Purpose |
|---|---|---|---|
| `/invitation?ad_id=&email=` | `pages/invitation` | "Join" CTA on `/view-campaign/:id` (paid campaigns), invitation emails | An influencer invited to a campaign looks it up (`v2/invitations/find`), signs up (`v2/guest/signup`) and verifies by email. |
| `/affiliate/invitation` | `pages/invitation` | "Join" CTA on `/view-campaign-affiliate/:id` | Same flow for affiliate campaigns. |
| `/gifting/invitation` | `pages/invitation` | **none in code** | Same flow. Note that the seeding CTA links to `/seeding/invitation` instead (see section 5). |
| `/view-campaign/:id` | `pages/viewCampaignNoAuth` | site URL copied in `campaignsV2` (copy link) | Public campaign preview (`v2/guest/campaigns/{id}`), content download (`v2/guest/campaigns/contents/download/{id}`), "Join" CTA. |
| `/view-campaign-affiliate/:id` | `pages/viewCampaignAffiliateNoAuth` | copied from `campaignsV2`, `affiliateCampaignList`, `affiliateDashboard2/CampaignsTab` | Public preview of an affiliate campaign. |
| `/view-campaign-seeding/:id` | `pages/viewCampaignSeedingNoAuth` | **none in code** (`campaignsV2` copies `/view-campaign/:id` for non-affiliate campaigns) | Public preview of a seeding campaign. |
| `/view-contents/:code?content_id=` | `pages/viewContentsNoAuth` | copied from `contents2` and `mobileFrame` (share link) | Public view of delivered content (`v2/published-contents/offers/{code}`). |
| `/public/profile/:id` | `pages/influencerProfile` (public mode) | built in `influencerCard/popoverProfile` (share or copy) | Public influencer profile (`v3/influencer/profile/public/*`). |
| `/review/:code` | `pages/review` | **none (email link)** | Guest review form after a collaboration (`GET/PUT v2/guest/reviews/{code}`). |
| `/review/:code/result` | `pages/reviewResult` | **none (email link)**. The review form does not navigate here. | Guest review result (`v2/guest/reviews/{code}/result`). |

**Count**: the route table has 94 URL patterns plus the `*` 404 route. That counts `/campaign/<step>` and `/campaign/<step>/:adID` separately, and `/affiliate/*` once. Treating the `/campaign` step variants as one pattern each leaves **88 patterns**:
- 4 are redirect-only: `/offers/:id`, `/inbox-v37-demo`, `/inbox-v37-demo/:id`, `/affiliate/dashboard2`.
- 2 are a stub or effectively unreachable: `/instagram-account/:id`, `/verify-phone-number`.
- **82 are live pages.**

Of the 82 live pages:
- **23 have no inbound link in code.** They are reached by email or magic link, as a payment return URL, or by typing the URL. The rows say **none**.
- **4 are self-linked only**: `/accountnew/:id`, `/new-offersV2/deals`, `/offers/deals`, `/offers/favorite`.

The 14 `/affiliate/<step>` values from section 2.1 are not included in these counts.

## 2. Step flows

### 2.1 Affiliate steps (`/affiliate/<step>`, `pages/affiliate/AffiliateContent.tsx`)

Step navigation uses `goTo()`, which builds `/affiliate/<step>?accountId&accountType&profileId&campaignId&accountName`.

| Step URL | Component (`pages/affiliate/parts/…`) | What happens / API |
|---|---|---|
| `/affiliate` | – | Redirects to `/affiliate/dashboard`. The real `/affiliate/dashboard` route goes to `affiliateDashboard2`, which wins over the catch-all. |
| `/affiliate/epty_state` or any unknown segment | `dashboardAffiliate` (legacy) | Old affiliate dashboard (`…/status`, cards, SSO link). Only reachable by typing a URL. |
| `/affiliate/sync_account` | `syncAffiliate` (also embedded as the Programs tab of the dashboard) | Connect stores:<br>• Amazon: `…/amazon-account/sync`, popup OAuth, `…/amazon-account/profiles/check/{id}`<br>• Shopify: `…/shopify-account/sync`, `…/shopify-account/stores/check/{id}`<br>• List or delete accounts: `…/affiliate-programs/list`, `…/account/delete/{id}`<br>• "Other platform" request forms: `v1/custom-forms/{ecommerce-platform\|affiliate-platform}`<br>• Grow-program modal → `/subscription-plans` |
| `/affiliate/sync_payment` | `payment` | Add a payout or billing card (`…/cards/sync` with `return_url`, `…/cards/list`). |
| `/affiliate/loading_store` | `step3` | 3-second fake loading screen, then `product_list`. No API call. |
| `/affiliate/product_list` | `productList` | Store products: `…/products/amazon/list` or `…/products/shopify/list`, brand dropdowns. Bulk commission (`…/products/bulk-update-commission`, gated by the `affiliate_commission_percent` limit), bulk activate or deactivate, assign to a campaign (`…/campaigns/assign-items`), campaign products (`v2/campaigns-affiliate/{id}/products`). |
| `/affiliate/affiliate_finalize` | `affiliateFinalize` | Done screen: view the dashboard or set up payment. |
| `/affiliate/coupons` | `couponManagement` | Coupon management UI. No service call was found in this component. |
| `/affiliate/select_campaign_type` | `selectCampaignType` | **Campaign type picker**, the main "create campaign" entry from about 20 CTAs. See the business rules in section 6. |
| `/affiliate/affiliate_campaign_setup` | `affiliateCampaignSetup` | Pick a connected store or account (`…/affiliate-programs/list`, `…/shopify-account/stores/list`). |
| `/affiliate/affiliate_campaign_details` | `affiliateCampaignDetails` | Campaign details. Create with `POST v3/campaigns/affiliate`; edit with `GET/PATCH v2/campaigns-affiliate/{id}`. |
| `/affiliate/affiliate_campaign_products` | `affiliateCampaignProducts` | Choose products. Its `onContinue` is a no-op. |
| `/affiliate/product_seeding_wizard[?edit=id]` | `productSeedingWizard` | **Product seeding (gifting) wizard**, in self-serve or managed mode: products from Amazon or Shopify, create or update with `POST product-seedings/handle`, load with `GET product-seedings/{id}`. Gated by `product_seeding_access`. |
| `/affiliate/affiliate-campaign-list` | `affiliateCampaignList` | List of affiliate campaigns: edit, delete, copy link, `/subscription-plans` CTA. |
| `/affiliate/programs`, `/affiliate/settings`, `/affiliate/review` | – | Accepted by the switch but **render nothing** (blank page). Nothing links to them. |

### 2.2 `/create-campaign` (paid collaboration), `pages/createAd`

The page shows one stage at a time, with no URL change:

1. **Start screen** (`BeforeCreateCampaign`): "Create new campaign" creates a draft (`POST v3/campaigns`).
2. **Store choice** (`CreateCampaignThirdParty`): connect Shopify (`user-third-party/shopify/v1/connect`, `.../products`, `campaign/start-process`, `save-product`, `finish-process`) or skip.
3. **Campaign forms**: the normal set (`CreateCampaignFormsNormal`) or the Shopify set (`CreateCampaignFormsShopify`).
   - Sections: detail, about brand, bio, category, country, shipping, date range, expectations, gender, hashtags, influencer type, language, post, receive offer, remaining posts, Shopify list, upload content.
   - Uploads use `files` and the media library; stock photos come from Pexels, Pixabay and Unsplash.
   - AI title and description: `v2/ai/campaign-creation/{id}/title-and-description`.
   - Progress is saved with `v2/campaigns/progress-step`; templates come from `v2/campaigns/templates-list`.
4. **After-create modal**:
   - add funds → `/funds?amount=…`
   - pending invites → `href="/offers"` (legacy list)

### 2.3 `/campaign/*` legacy wizard (`lib/routes/createAd.tsx`, `common/createAd/parts/*`)

| Step | Component | API |
|---|---|---|
| `/campaign` and `/campaign/objectives[/:adID]` | `objectives` | `v2/campaign/goal-questions`, `PUT v2/campaigns/{id}`. Maximum two objectives. |
| `/campaign/marketplace[/:adID]` | `marketplace` | `v2/campaigns/templates` |
| `/campaign/post-content[/:adID]` | `post` | `POST v2/campaign/create/step1`, `…/create/content`, `files`, `v2/all-medias` |
| `/campaign/publish-terms[/:adID]` | `publishTerms` | `POST/PUT v2/campaigns/{id}/publish-terms` |
| `/campaign/confirm[/:adID]` | `confirm` | – |
| `/campaign/discover[/:adID]` | `discover` | `POST v2/campaigns/{id}/influencer-filters`, `v2/categories/hashtags` |

The only inbound link is "Edit ad" (`/campaign/objectives/:id`). The layout's "onboard" mode (sidebar steps under `/welcome/*`) is never used, because the route always passes `mode="create"`.

## 3. Real-time chat (chat host `CHAT_BASE_URL`, socket.io)

The connection is `io(CHAT_BASE_URL, {transports:['websocket'], auth:{'x-chat-auth':'Bearer <chatbox_token>','x-public-key':…}})`. It is created by `useWebSocketClient` in `components/pages/newSingleOfferV2/model.ts`, which the inbox uses through `pages/inboxV37/hooks/useInboxOfferChatSession.ts`. It is live on `/new-offersV2` and `/new-offersV2/:id`.

| Direction | Event | Live? | File |
|---|---|---|---|
| on | `connect`, `disconnect`, `connect_error`, `onAny` (logs) | live | `newSingleOfferV2/model.ts` |
| on | `new-message`, `delivered-message`, `edit-message`, `delete-message`, `error-event` | live | `newSingleOfferV2/model.ts` (log only); `inboxV37/chat/InboxV37ChatMessages.tsx` (real handlers) |
| on | `seen-message`, `unread_count_updated`, `channel_updated` | live (log-only listeners) | `newSingleOfferV2/model.ts` |
| on | `message_received` | live (no-op listener) | `inboxV37/chat/messagesModel.ts` `joinChannel` |
| emit | `new-message` `{conversationId, body \| attachment, attributes:{role:'brand'}}` | live | `newSingleOfferV2/model.ts` (`useMessageService`), `newSingleOfferV2/parts/newChat/model.ts` (`sendMessageWebSocket`), both called from `inboxV37/chat/InboxV37ChatSender.tsx` |
| emit | `seen-message` `{messageId}` | live | `inboxV37/chat/InboxV37ChatMessages.tsx` |
| emit | `subscribe_channel` `{chatId}` | live | `newSingleOfferV2/model.ts` (`getChannelWebSocket`), `inboxV37/hooks/useInboxOfferChatSession.ts` |
| emit | `unsubscribe_channel` `{chatId}` | live | `inboxV37/hooks/useInboxOfferChatSession.ts` |
| emit | `join_channel` `{chatId, perPage}` | live | `inboxV37/chat/messagesModel.ts` `joinChannel` |
| emit | `update_channel` (and on `channel_updated`) | **dead** (`updateChannel` never called) | `inboxV37/chat/messagesModel.ts` |
| emit | `subscribe_all_conversations` | **dead** | `newSingleOfferV2/parts/newChat/model.ts` (`getSubscribedConversations` is never called) |
| emit | `unsubscribe_channel`, `seen-message`, `join_channel` | **dead** (unreachable files) | `newSingleOfferV2/index.tsx`, `newSingleOfferV2/parts/newRoom/index.tsx`, `newSingleOfferV2/parts/newChat/parts/messages/model.ts` |

Chat-host REST calls (headers `x-chat-auth` and `x-public-key`):

| Call | Live? | Where |
|---|---|---|
| `GET {CHAT}/message/unread-count` | live | Sidebar badge (global) and `useUnreadMessageService` (inbox) |
| `GET {CHAT}/message/conversation/{id}?page&pageSize&order=desc[&startIndex]` | live | `inboxV37/chat/preloadConversationPage.ts` and `messagesModel.fetchConversationPage`, used by the inbox and by the chat drawer on the campaign influencer pages |
| `GET {CHAT}/message/unread/id` | dead | Exported in `newSingleOfferV2/model.ts` but never called |

Media is shown from `CHAT_IMAGE_BASE_URL`; that is a URL prefix, not an API call.

**Legacy Twilio** (`@twilio/conversations` in `singleOffer/model.ts`) is still initialised by the sidebar on every logged-in page, along with `POST v2/refresh-twilio-token`.

Other third parties called from live code:
- OpenCage geocoding (`common/customMap`) and Mapbox static images (inbox location messages)
- Stripe Elements (adding cards), PayPal (funds and checkout)
- Google sign-in (`gapi-script`), reCAPTCHA
- chatgram-widget

## 4. Dead code

`(generator script)` has the complete file list: 796 unreachable code files.

### 4.1 Dead routes (only in comments in `lib/routes/index.tsx`)

- The old `/search` → `SearchPageV2` (and an older `SearchPage`).
- The old `/campaigns` tree under `AdsHoc`: `Ads`, `/campaigns/invited-influencers` → `InvitedInfluencers`, `/campaigns/type-affiliate` → `AdsAffiliate`, `/campaigns/seeding` → `AdsSeeding`. It was replaced by `CampaignsV2Page`.

**Unused imports in `lib/routes/index.tsx`** (imported but never rendered): `Skip`, `AdsHoc`, `Ads`, `Company`, `Contents`, `DashboardV2`, `ForgotPassword`, `InvitedInfluencers`, `Login`, `Points`, `Register`, `ResetPassword`, `SearchPageV2`, `PayOffer` (`newPayOffer`), `Transactions` (`newTransactions`), `AdsAffiliate`, `AdsSeeding`.

Also unused: `PhoneNumberVerificationMiddleware` in both `routes.tsx` and `layouts/dashboard/index.tsx`.

### 4.2 Page folders

**Fully dead** (0 reachable files):

| Folder | Why |
|---|---|
| `pages/ads` | Old campaigns list. Only referenced in the commented-out route. |
| `pages/adsAffiliate` | Old affiliate campaigns list. Only referenced in the commented-out route. |
| `pages/adsSeeding` | Old seeding campaigns list. Only referenced in the commented-out route. |
| `pages/cards` | Old cards page; `newCards` is routed. |
| `pages/company` | Unused import (uses `createCompanyService`). |
| `pages/contentsTranslated` | Never imported; `translation2` replaced it. |
| `pages/editAccount` | Never imported; `userInfo2` replaced it. |
| `pages/forgotPassword` | Unused import; `forgotPassword2` is routed. |
| `pages/funds` | Old; `newFunds` is routed (used `v2/user/wallet/charge`). |
| `pages/invitedInfluencers` | Unused import (`v2/user/invitations` list and cancel). |
| `pages/login` | Unused import; `login2` is routed. |
| `pages/newPayOffer` | Unused import (`v3/offers/{id}/pay`, PayPal pay-offer). |
| `pages/payOffer` | Never imported. |
| `pages/points` | Unused import; `points2` is routed. |
| `pages/register` | Unused import; `register2` is routed. |
| `pages/resetPassword` | Unused import; `resetPassword2` is routed. |
| `pages/verifiyPhoneNumber` | Never imported; `verifyPhoneNumber2` is routed. |
| `pages/verifyByLink` | Never imported; `verifyByLink2` is routed. |
| `pages/verifyEmail` | Never imported; `verifyEmail2` is routed. |

**Mostly dead** (only fragments reused):

| Folder | Reachable / total files | What survives |
|---|---|---|
| `pages/dashboardV2` | 1/50 | Only `parts/youtube-video`. |
| `pages/newSingleOfferV2` | 40/133 | `context`, `model`, and modals reused by `inboxV37`. The page itself, `newRoom`, and the old chat message model are dead. |
| `pages/singleOffer` | 14/114 | The page only redirects. `model.ts` (Twilio) is used by the sidebar; a few modals are reused. |
| `pages/searchV2` | 5/41 | `context` and `model` used by `searchV3` and `influencer-explorer`. |
| `pages/newOffersV2` | 11/37 | Datagrid column components reused by `deals/indexV2`. **This is not a routed page.** The `/new-offersV2` URL is served by `inboxV37`. |
| `pages/newTransactions` | 5/17 | Pieces reused by `transactions2`. |
| `pages/userInfo` | 3/12 | Category selector and model reused. |
| `pages/influencerProfileNew` | 22/56 | |
| `pages/contents` | 1/3 | Only its modal (used by `dashboardV3`). |

The suspected-dead list from the brief checks out as follows:
- `cards`, `contentsTranslated`, `editAccount`, `verifiyPhoneNumber`, `verifyByLink` and `verifyEmail` are **confirmed dead**.
- `newOffersV2` the *folder* is dead as a page.
- The old **`/offers` inbox is still live**: routed, linked from invite modals and the notification fallback, not in the sidebar.

### 4.3 Other dead components and modules

- Fully dead `components/common/*`: `creditCard`, `dataGridV3`, `socialMediaPreview2NewProfile`, `endOfSummerModal`, `modalV2`, `middlewares`, `baseLoading`, `chart`, `countdown`, `customHtmlTooltip`, `dataGridRange`, `dataGridSearchV3`, `datapicker3`, `datepicker2`, `dialog`, `favoriteButton`, `languageSwitcher`, `letterAvatar`, `scrollbar`, `settings`, `tabPanel`, `translationBoxGuest`, `trustPay`.
- Fully dead HOCs: `components/hoc/ads`, `components/hoc/offers`.
- Fully dead hooks and lib files: `hooks/custom`, `hooks/shopify`, `lib/redux/actions/twilio_change_refetch_flag.ts`, `lib/jsons/carrierCompanies.ts`, `lib/constant/initialLocalLang.ts`, `lib/utils/checkDevice copy.ts`.

### 4.4 Service files and functions

**Whole service files that are dead:**

| File | Why |
|---|---|
| `lib/services/newOffersV2.ts` | Never imported by any file; a stale copy of `offer.ts`. |
| `lib/services/wishlist.tsx` | Only imported by `offers/.../influencer` grid columns, and the import is unused. |
| `lib/services/analytics.ts` | `v1/advance-analytics`; only used by the dead `newSingleOfferV2/.../tabAnalytics`. |
| `lib/services/company.ts` | Only used by the dead `pages/company`. |
| `lib/services/invitedInfluencer.ts` | Only used by the dead `pages/invitedInfluencers` and `pages/points`. |

**Dead functions in live service files.** There are 101 dead call entries in total, each with a specific reason in `web-calls.json`. Notable ones:
- `ad.ts`:
  - `getActiveCampaignsService` and `UpdateAdSettingService`: imported but never called.
  - `shortLinkService` and `CampaignAggregated`: only used by dead code.
  - `getSeedingListService`: only used by dead `adsSeeding`.
  - `getAdAffiliateProductsService`: imported but never called.
- `affiliate.ts`: `getAmazonAccountsList` (the call is commented out).
- `auth.ts`:
  - `refreshTokenService`: imported in `_config.ts` but never called; the interceptor uses `axios.post` directly.
  - `influencerRegisterService`, `simulate401`.
- `chat.ts`:
  - `chatRoomsService` (`v2/chats`): only used by the old `newRoom` components.
  - `unreadChatsService`: only used by the dead `header/parts/unreadMessages`.
- `dashboard.ts`: `lastMessagesDataService`.
- `file.ts`: `AdFileSort`.
- `influencer.ts`: `getInfluencerService`, `deleteInfluencerFromFavoritesListSerivce`.
- `integration.ts`: `getIntegrationsService`, `addIntegrationsService`, `getDeteilAccountService`.
- `invitation.ts`: `getInvitationService` (commented out).
- `offer.ts`: `approveStatusOfferService`, `disapproveStatusOfferService`, `updatePublishDateAfterFailureService`, `EditOfferAfterPaymentService`, `chageStateOfferService`, `getChatTemplateService`, `mostUsedHashtagService`, `recentAdsService`, `influencerGoalAnswerService`, `ratingAndReviewsSidebarService`, `messageReactionService`, `offerInformationService`.
- `payments.ts`:
  - The old `v2/cards*` endpoints (`cardsService`, `cardsCreateService`, `cardDeleteService`, `cardUpdateService`) were replaced by `finance/current-user/cards*`.
  - `transActionsService`, `walletChargeService`, `offerChargeService`, `offerPaymentServiceV3`, `paypalPayOfferService`.
- `profile.ts`: `getUserService`.
- `publishJobs.ts`: `downloadZipPublishedContents`, `putAnalyticsData`.
- `search.ts`: `elasticSearchByIDService`.
- `translateContent.ts`: duplicate `finishShopifyProcess` and `userShopifyConnect`. The live copies are in `thirdparty.ts`.

## 5. Broken or suspicious links

1. **`/seeding/invitation?ad_id=…` is not routed.** It is the "Join campaign" CTA on the public seeding campaign preview (`pages/viewCampaignNoAuth/desktop/index.tsx:93`, `mobile/index.tsx:93`, used by `/view-campaign-seeding/:id`).
   - It falls through to the logged-in layout's `*` route, so a logged-out influencer is sent to `/login` and a logged-in user sees the 404.
   - `/gifting/invitation` *is* routed but nothing links to it, so it is probably the intended target.
   - The invitation page detects seeding with `pathname.includes("/seeding/invitation")` (`invitation/parts/howItWorks`, `stepper/parts/createAnAccount`, `stepper/parts/welcome`, `viewCampaignNoAuth/campaignPreviewModel.ts`). So `/gifting/invitation` renders the non-seeding copy.
2. `/affiliate/programs`, `/affiliate/settings` and `/affiliate/review` are accepted by the `AffiliateContent` switch but render a **blank page**. Nothing links to them.
3. `/welcome`, `/welcome/<step>` and `/welcome/skip` are built in `common/createAd/layout/parts/onboard/*` and are not routed. They are latent only, because the onboard mode is never mounted.
4. `/view-campaign-seeding/:id` is never produced by the copy-link logic: `campaignsV2.handleCopy` uses `/view-campaign/:id` for every type except affiliate. Seeding previews may therefore be opened through the generic viewer.
5. Several flows still point at the **legacy `/offers` list** instead of the inbox, using a full-reload `href="/offers"`:
   - invite-success modals (`favoriteInfluencerTabs/parts/inviteInfluencerAd`, `influencerProfile(/New)/parts/modal/inviteInfluencerAd`, `searchV2/parts/inviteInfluencerPendingAd`)
   - the after-create pending-invite modal
   - the payment-success influencer list
   - the notification fallback `getRedirectUrl` (offer event with no id → `/offers`)
6. `changePublishDataModal` has a relative `navigate("pay/<offerId>")` branch that would resolve to the unrouted `/new-offersV2/:id/pay/:offerId`. It is never triggered today, because the inbox does not pass an `offerId` prop.
7. `/instagram-account/:id` is routed to a non-functional stub.

## 6. Business rules visible in the UI (hints for flows)

- **Access and onboarding**:
  - Every dashboard page needs a login, and brands who haven't finished onboarding are forced to `/onboarding`.
  - Onboarding now asks only for **business name and location**. The other steps (company size, category, hashtags, platforms, target influencers, objectives) are commented out.
  - Influencer accounts that log in on the brand app are sent to `/influencer-landing`.
  - Suspended brands get an appeal modal at login (`is_appealable`).
  - Shopify-installed brands must set a password first.
- **Phone**:
  - The phone number is **required at registration** (E.164).
  - The OTP page `/verify-phone-number` exists but nothing navigates to it. The phone-verification middleware and modal are dead, so phone verification is not enforced in the web app.
- **Team members** (a brand member acting inside another brand's team, `currentTeam`):
  - They are blocked from `/profile`, `/profile/change-password`, `/profile/email-notifications` and `/influencer-content` (redirected to `/`).
  - The sidebar still shows "Influencer content" to them.
  - `X-Team-Id` is sent on every API call.
- **Plan limits**: the backend answers **HTTP 402 `feature_limit_exceeded`**, which opens `FeatureLimitModal` (`components/common/modals/limitModals`) with an upgrade CTA to `/subscription-plans`. Feature keys:
  - `campaigns`
  - `influencer_invites` (also add-on invite packs: +100 for $29, +250 for $59, +500 for $99)
  - `analytics`
  - `seats` (extra seat: $19 on Pro, $29 on Ultimate)
  - `affiliate_commission_percent` (lower commission needs a higher plan)
  - `influencer_username_reveal` (monthly reveal credits; can pay from the wallet instead)
  - `product_seeding_access`
  - Fallback plan prices in the modal: Starter $49/mo, Pro $149/mo, Ultimate $699/mo, with quarterly and yearly discounts. Durations 1, 3 (or 4) and 12 months come from `subscriptions/plans`.
- **Campaign types** offered at `/affiliate/select_campaign_type`:
  - **Affiliate Campaign**: commission per sale. Disabled until `…/status` reports a **card on file and a connected Amazon or Shopify store**; otherwise it routes to `/affiliate/dashboard`. A modal asks "track through Ainfluencer" or "I already have an affiliate platform" (that choice means: run a Paid Collaboration with your own links, $10 minimum per influencer).
  - **Paid Collaboration** → `/create-campaign`. "$10 minimum payment per influencer."
  - **Paid collab + Affiliate**: *coming soon*.
  - **Product Gifting** (featured) → `/affiliate/product_seeding_wizard`, in **self-serve** ("~15 minutes", you ship) or **managed** ("1 brief call", the team runs it) mode.
- **Offer payment**:
  - Pay is triggered in the inbox.
  - If the agreed publish date is less than 24 h away, or already past, the brand must first change the publish date (the waiting-payment and change-date modals).
  - Checkout uses a basket with an **optional insurance add-on**, coupons, and card, wallet or PayPal.
  - After payment, other influencers awaiting payment and related influencers are suggested.
- **Wallet**: add funds by card or PayPal, minimum amount 1. The campaign-created modal suggests funding (`/funds?amount=`). The "Fully managed" packages are paid only through wallet funds ("no extra fees").
- **Brand score**: tiers Bronze, Silver, Gold and Platinum on `/profile/points`. Higher tiers "rank higher in influencer search and carry a verified badge".
- **Subscription management** (on `/profile/transactions?tab=billing`):
  - Cancelling requires a survey form (`v2/forms/positions` → `v2/forms/{id}`).
  - Cancelled plans can be resumed.
  - Receipts are listed.
- **Contact sharing in chat**: the inbox sender warns about "risk of account suspension" when contact details are shared, and has a "continue on free" acknowledgement that links to `/subscription-plans`.
- **Affiliate store connect** happens in a popup (Amazon or Shopify OAuth). The callback lands on `/affiliate/...?account_id=&status=&error=&error_type=` and posts `AMAZON_AFFILIATE_SYNC_COMPLETE` to the window that opened it, with a localStorage fallback.

## 7. Files

- `(generator script)`: 358 entries (257 live, 101 dead; 237 unique live method+path pairs).
- `(generator script)`: reachable and unreachable files, import edges, symbol usage.
- `(generator script)`: every in-code link literal with the route it matches.
- Scripts: `(generator script)`, `links.py`, `api.py`, `finalize.py` (re-run them in that order).
