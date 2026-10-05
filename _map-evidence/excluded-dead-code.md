# Code left out of the flow map

The flow map covers only code a real user can reach today. This file lists what was dropped and why, so the exclusions can be checked. Everything here comes from static analysis of the same snapshots as the map (see the header of [../flow-map.md](../flow-map.md)).

If something here is actually live (for example a page that is linked only from an email the code doesn't show), tell the map owner so it can be moved back in.

## 1. Brand web app (core/frontend/web, master)

The full lists are in [web.md](web.md) §4. Summary:

- **796 of 2,046 code files can't be reached** from the app entry point.
- **Routes that exist only in comments**: the old `/search` (SearchPageV2) and the old `/campaigns` tree (Ads, invited-influencers, type-affiliate, seeding).
- **19 page folders are fully dead**: `ads`, `adsAffiliate`, `adsSeeding`, `cards`, `company`, `contentsTranslated`, `editAccount`, `forgotPassword`, `funds`, `invitedInfluencers`, `login`, `newPayOffer`, `payOffer`, `points`, `register`, `resetPassword`, `verifiyPhoneNumber`, `verifyByLink`, `verifyEmail`. In most cases a `…2`/`new…` version of the page is the live one.
- **Pages that are routed but not usable**:
  - `/instagram-account/:id`: a stub that calls no API.
  - `/verify-phone-number`: nothing navigates to it. Phone OTP is **not enforced** in the web app; the phone-verification middleware is imported but never rendered.
  - `/affiliate/programs`, `/affiliate/settings`, `/affiliate/review`: render a blank page and nothing links to them.
- **Redirect-only URLs**: `/offers/:id` (→ `/new-offersV2/:id`), `/inbox-v37-demo[/:id]`, `/affiliate/dashboard2`.
- **5 service files are dead**: `newOffersV2.ts`, `wishlist.tsx`, `analytics.ts` (advanced analytics is not reachable from the web app), `company.ts`, `invitedInfluencer.ts`. A further 101 individual call sites are dead, each with a reason in the generator output.
- **Chat socket events with no server counterpart**: `subscribe_channel`, `unsubscribe_channel`, `join_channel`, `update_channel`, `subscribe_all_conversations`. The client sends them but the chat gateway ignores them.
- **Legacy Twilio chat**: still started by the sidebar on every logged-in page (`POST v2/refresh-twilio-token`), but nothing the user sees depends on it. The unread badge comes from the current chat service.

## 2. Influencer mobile app (Mobile/mobile-app, develop v32.2.2)

The full lists are in [mobile.md](mobile.md) §5. Summary:

- **275 code files can't be reached.** `features/userProfile` is fully dead.
- **26 of 90 registered screens have no way in**:
  - 6 orphan routes: `NOT_VERIFY_SCREEN`, `ADD_SOCIAL_SCREEN`, `CREATE_MSG_TEMPLATE`, `EDI_CONTENT_SCREEN`, `USER_TIKTOK_CONTENT_SHARE`, `BRAND_ID_LANDING`.
  - 15 legacy onboarding screens and modals (the old step-by-step chain). The live onboarding is the 6-step media-kit flow.
  - `CREATE_OFFER_SCREEN`: a stub.
  - 4 old-wallet screens: `WITHDRAW_SCREEN`, `PAYOUT_SETUP`, `PAYOUT_METHOD_LIST`, `CHECKOUT_LIST`. Their buttons are commented out, so the **old withdraw flow (`v3/mobile/wallet/checkout-requests`) is dead**. Only the old wallet history is still viewable.
- **Disabled in the UI**: Stripe payouts (only PayPal and crypto can be added), the Rates and Supercharge onboarding steps, and the language switcher.
- **Not called**: backend logout (logout is local only), REST message reactions, socket `send-message`, the cities lookup, and the offers-based inbox (the Inbox tab uses `v3/mobile/chats`).
- **Twilio chat path**: present but never runs, because the chat provider is hard-coded to the socket service.

## 3. Chat services (core/backend/chatbox)

See [chat.md](chat.md) §4.

- **Gateway features no client uses**: edit message, delete message, message search, online presence and read-receipt broadcast. The gateway supports them; neither app offers them.
- **Client listeners for events the gateway never sends**: `delivered-message`, `unread_count_updated`, `channel_updated`, `message_received`. `seen-message` arrives only as an error payload.
- **Calls to paths that don't exist**: web `GET /message/unread/id` and `v2/offers/list-by-conversation-ids`; Laravel's `SeenEndpoint` (`/messages/:id/seen`), which nothing calls.

## 4. Backend (core/backend/api, master)

### 4.1 Features that are wired but do nothing

| Item | Why it's excluded |
|---|---|
| ActiveCampaign integration | Every trigger is commented out and its config file is missing. |
| AfterShip tracking | Not bound. Track123 is the live tracking driver; only the AfterShip webhook route remains. |
| HeyGen video translation | Inactive. The default translation driver is BrandID. |
| Public API outbound webhooks | Infrastructure exists (webhook CRUD, logs, prune job), but no business code emits events. |
| Pusher / broadcasting, Slack notifications | Configured but unused. |
| `StripeGateway::payout`, `PaypalGateway::payout` | Never called. Influencer payouts are paid by hand by an admin. |
| Shopify GDPR webhooks (`get-customer`, `erase-customer`) | Stubs that do nothing. |
| `POST /v3/webhooks/stripe` | Broken route definition; it can't resolve a controller. |
| `POST /stripe/webhook` (Cashier default) | Registered by accident. The real subscription webhook is `/api/subscriptions/webhook`. |
| Offer statuses `expired`, `suspended`; order status `cancelled`; CheckoutRequest `in_progress`/`rejected`; Campaign `state` | Defined but never written. |
| Legacy revision expiry for new revisions (`offer:expire-revisions`) | Command exists but is not scheduled. |

### 4.2 Scheduled jobs that don't do what their name says

| Command | Issue |
|---|---|
| `offers:frozen-awaiting-payment` | Its SQL can never match, so it does nothing. |
| `ads:calculate-scores` | Cron `0 */72 * * *` actually runs daily at 00:00, not every 3 days. |
| `AddMultipleIntegration`, `AggregatedTwilioMessageNotifier` | Imported in the Kernel but not scheduled (the Twilio class file doesn't exist). |
| `commentedCommands()` block | Never called: waiting-payment emails, identity email update, file pruning, integration suspension/restoration checkers. |
| `GiftingRequestsDigestCommand` | Replaced by the brand daily digest, no longer scheduled. |
| Automatic integration suspension / restoration jobs | Their scheduler entries are commented out. Automatic suspension rules run only when an integration is created. |

### 4.3 Notifications and emails that are never sent

From the backend notification catalogue: `AdminEditOfferToInfluencerNotification`, `InfluencerPostApprovedByBrand`, `InfluencerPostFailedToPublish`, `InfluencerPostSuccessfullyPublished`, `InfluencerUnarchivedNotification`, `UserQualityControlNotification`, `TemporaryIntegration/Account{Verified,Rejected}Notification`, `OfferStartChangeNotification`, `PostReminderNotification`, `VerificationNotification`, `v2/ReviewSubmittedNotification`, `v2/ViolationSentNotification`, `InvitationCreated` (and its mobile copy), `PendingRevisionNotification`, `PublishApprovedByAdminNotification`, `PublishDisapprovedByAdminNotification` (mobile).

Mailables never sent: `AfterPaymentMade`, `ExternalInfluencerInvitation`, `InfluencerPostApprovedByBrand`, `LegacyAggregatedInvitation`, `v2/Admin/UserSuspended`, `v2/Offer/BrandWalletChargedEmail`, `WelcomeEmail`, `Team/Brand/Email/MemberJoinedTeam`, `AdminAddMessageToBrand`, `AdminAddMessageToInfluencer`.

Sent only by a manual command (not scheduled): `AwaitingPaymentOffer`, `IgNotConnect`, `ConnectPaypalEmail` reminder (the email is also sent once when an offer is paid and the influencer has no payout method).

The `Illuminate\Auth\Events\Registered` → `SendEmailVerificationNotification` listener is dead: nothing dispatches `Registered`. Verification emails are sent directly.

### 4.4 Routes nothing calls

**413 routes** are not called by the web app, the mobile app, the chat service, a third party, the admin panel or any other in-house client. The largest groups:

- `api/v2/mobile/*` (42 routes): the **old influencer app API**. The current app uses `v3/mobile`, and every store update is forced, so old app versions should no longer be in use.
- `api/{locale}/v3/mobile/*` (41 routes): v3 endpoints the current app doesn't call (for example the old checkout-request withdraw, PayPal connect, logout, unblock brand).
- Legacy v1 REST resources (`identities/{identity}`, `wallets/{wallet}`, `offers/{offer}`, `integrations/{integration}`, `contents`, `publish-jobs`, `campaigns`, `ads`, `methods`, `tasks`, …): the original API from before v2.
- `v2/twilio-migration/*`, `v2/extensions/*`, `v3/kpi/*`, `v1/elastic/*`, and old v2 offer/content/chat endpoints that were replaced by newer ones.

Some of these might still be called by tools outside the scanned repos (for example internal scripts or the Node/AI services). The full list is below.


<details><summary>Full list of unused routes, grouped by prefix (413)</summary>


**v2/mobile** (42)

- `DELETE v2/mobile/auth/account/remove`
- `DELETE v2/mobile/insight/{insight_id}/file/{file_id}`
- `DELETE v2/mobile/message-templates/{id}`
- `DELETE v2/mobile/published-contents/files`
- `GET v2/mobile/ads`
- `GET v2/mobile/ads/all`
- `GET v2/mobile/ads/for-you`
- `GET v2/mobile/ads/invited`
- `GET v2/mobile/ads/most-paid`
- `GET v2/mobile/ads/my-categories`
- `GET v2/mobile/ads/my-country`
- `GET v2/mobile/ads/{id}`
- `GET v2/mobile/boot`
- `GET v2/mobile/configs`
- `GET v2/mobile/funnels/processes`
- `GET v2/mobile/goal-questions`
- `GET v2/mobile/goal-questions/answers`
- `GET v2/mobile/initialize`
- `GET v2/mobile/insight/{identity_id}`
- `GET v2/mobile/invitations`
- `GET v2/mobile/message-templates`
- `GET v2/mobile/platform/influencer-data`
- `GET v2/mobile/reports/items`
- `GET v2/mobile/temporary-integrations`
- `GET v2/mobile/temporary-integrations/{id}`
- `PATCH v2/mobile/message-templates/{id}`
- `PATCH v2/mobile/publish-jobs/{id}/extend-deadline`
- `POST v2/mobile/auth/email/resend`
- `POST v2/mobile/block`
- `POST v2/mobile/block/retrive`
- `POST v2/mobile/funnels`
- `POST v2/mobile/goal-questions`
- `POST v2/mobile/insight`
- `POST v2/mobile/invitations/hide`
- `POST v2/mobile/invitations/see`
- `POST v2/mobile/message-templates`
- `POST v2/mobile/offers/make`
- `POST v2/mobile/published-contents/files`
- `POST v2/mobile/reports`
- `POST v2/mobile/temporary-integrations/verify/manually`
- `POST v2/mobile/temporary-integrations/{id}/resubmit`
- `PUT v2/mobile/tokens/{user_id}`

**v3/mobile** (41)

- `DELETE v3/mobile/deals/{id}`
- `DELETE v3/mobile/wallet/payout-methods/{method}`
- `GET v3/mobile/addresses/{address}`
- `GET v3/mobile/ads`
- `GET v3/mobile/ads/invited`
- `GET v3/mobile/ainfluencer-affiliate/sso/authorize-url`
- `GET v3/mobile/influencer/profile/aggregated-reviews/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/all-data/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/dealt-brands/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/details/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/elastic-data/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/feedbacks/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/galleries/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/integration/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/past-collaborations/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/recommended-influencers/{influencerIntegrationId}`
- `GET v3/mobile/influencer/profile/{influencerIntegrationId}/all-accounts`
- `GET v3/mobile/message-templates/default-chat-templates`
- `GET v3/mobile/offers/{id}/history`
- `GET v3/mobile/offers/{offer_id}/revise/{revision_id}/show`
- `GET v3/mobile/platforms`
- `GET v3/mobile/wallet/checkout-requests`
- `GET v3/mobile/wallet/payout-methods`
- `PATCH v3/mobile/influencer/profile/hit-profile-views/{influencerIntegrationId}`
- `POST v3/mobile/delete`
- `POST v3/mobile/firebase-token`
- `POST v3/mobile/forgot-password`
- `POST v3/mobile/influencer/profile/feedbacks/like`
- `POST v3/mobile/influencer/youtube/auth`
- `POST v3/mobile/logout`
- `POST v3/mobile/notifications/read/all`
- `POST v3/mobile/offers/{chatId}/messages/{messageId}/react`
- `POST v3/mobile/offers/{offer_id}/revise/{revision_id}/accept`
- `POST v3/mobile/offers/{offer_id}/revise/{revision_id}/reject`
- `POST v3/mobile/privacy/brand/unblock`
- `POST v3/mobile/reset-password`
- `POST v3/mobile/wallet/checkout-requests`
- `POST v3/mobile/wallet/payout-methods/paypal/connect`
- `POST v3/mobile/wallet/payout-methods/{method}/restore`
- `PUT v3/mobile/product-seedings/{id}/respond`
- `PUT v3/mobile/wallet/payout-methods/{method}`

**identities/{identity}** (13)

- `DELETE identities/{identity}`
- `DELETE identities/{identity}/hashtags/{hashtag}`
- `GET identities/{identity}`
- `GET identities/{identity}/campaigns`
- `GET identities/{identity}/hashtags`
- `GET identities/{identity}/hashtags/{hashtag}`
- `GET identities/{identity}/profile`
- `PATCH identities/{identity}/hashtags`
- `POST identities/{identity}/campaigns`
- `POST identities/{identity}/hashtags`
- `POST identities/{identity}/upload`
- `PUT/PATCH identities/{identity}`
- `PUT/PATCH identities/{identity}/hashtags/{hashtag}`

**v3/influencer** (12)

- `GET v3/influencer/on-boarding`
- `GET v3/influencer/profile/aggregated-reviews/{influencerIntegrationId}`
- `GET v3/influencer/profile/dealt-brands/{influencerIntegrationId}`
- `GET v3/influencer/profile/details/{influencerIntegrationId}`
- `GET v3/influencer/profile/feedbacks/{influencerIntegrationId}`
- `GET v3/influencer/profile/galleries/{influencerIntegrationId}`
- `GET v3/influencer/profile/integration/{influencerIntegrationId}`
- `GET v3/influencer/profile/past-collaborations/{influencerIntegrationId}`
- `GET v3/influencer/profile/recommended-influencers/{influencerIntegrationId}`
- `POST v3/influencer/on-boarding`
- `POST v3/influencer/on-boarding/profile/lookup`
- `POST v3/influencer/on-boarding/profile/verification`

**v3/brand** (11)

- `GET v3/brand/influencer/waiting-reviews`
- `GET v3/brand/mobile/account`
- `GET v3/brand/mobile/campaigns/{id}`
- `GET v3/brand/mobile/influencers/search`
- `GET v3/brand/mobile/offers/{id}`
- `GET v3/brand/mobile/third-party/{driver}/verify`
- `GET v3/brand/score`
- `POST v3/brand/mobile/auth/login/otp/{channel}`
- `POST v3/brand/mobile/auth/login/otp/{channel}/send`
- `POST v3/brand/mobile/influencers/invite`
- `POST v3/brand/mobile/third-party/login`

**v2/user** (10)

- `DELETE v2/user/brand-information/logo`
- `DELETE v2/user/payout-methods/{method}`
- `GET v2/user/me`
- `GET v2/user/payments`
- `GET v2/user/payout-methods`
- `GET v2/user/payouts`
- `GET v2/user/wallet/checkouts`
- `POST v2/user/payout-methods/{method}/restore`
- `POST v2/user/wallet/checkouts`
- `PUT v2/user/payout-methods/{method}`

**wallets/{wallet}** (9)

- `GET wallets/{wallet}/funds`
- `GET wallets/{wallet}/methods`
- `GET wallets/{wallet}/stripe/connect`
- `GET wallets/{wallet}/transactions`
- `PATCH wallets/{wallet}`
- `PATCH wallets/{wallet}/paypal`
- `POST wallets/{wallet}/charges`
- `POST wallets/{wallet}/methods`
- `POST wallets/{wallet}/payouts`

**user-third-party/ainfluencer-affiliate** (9)

- `GET user-third-party/ainfluencer-affiliate/v1/account/list`
- `GET user-third-party/ainfluencer-affiliate/v1/amazon-account/callbacks/sync`
- `GET user-third-party/ainfluencer-affiliate/v1/amazon-account/profiles/list`
- `GET user-third-party/ainfluencer-affiliate/v1/campaigns/amazon-products/items`
- `GET user-third-party/ainfluencer-affiliate/v1/coupons/list`
- `GET user-third-party/ainfluencer-affiliate/v1/coupons/show/{coupon_id}`
- `GET user-third-party/ainfluencer-affiliate/v1/shopify-account/callbacks/sync`
- `POST user-third-party/ainfluencer-affiliate/v1/campaigns/amazon-products/assign`
- `POST user-third-party/ainfluencer-affiliate/v1/coupons/create`

**finance/mobile** (9)

- `DELETE finance/mobile/payout-methods/{payout_method}`
- `DELETE finance/mobile/payouts/{payout}`
- `GET finance/mobile/payouts/create`
- `GET finance/mobile/payouts/{payout}`
- `GET finance/mobile/payouts/{payout}/edit`
- `GET finance/mobile/wallets`
- `GET finance/mobile/wallets/{wallet}`
- `POST finance/mobile/wallets`
- `PUT/PATCH finance/mobile/payouts/{payout}`

**integrations/{integration}** (8)

- `DELETE integrations/{integration}`
- `GET integrations/{integration}`
- `GET integrations/{integration}/pricings`
- `POST integrations/{integration}/checkpoint`
- `POST integrations/{integration}/phone-number`
- `POST integrations/{integration}/pricings`
- `POST integrations/{integration}/two-factor`
- `PUT/PATCH integrations/{integration}`

**offers/{offer}** (8)

- `DELETE offers/{offer}`
- `GET offers/{offer}`
- `GET offers/{offer}/history/{start?}/{end?}`
- `POST offers/{offer}/approve-post`
- `POST offers/{offer}/history/status`
- `POST offers/{offer}/nullify`
- `POST offers/{offer}/pay`
- `PUT/PATCH offers/{offer}`

**v2/chatbox** (7)

- `GET v2/chatbox/unread-messages-count`
- `POST v2/chatbox/chats/rooms/{room}/pin`
- `POST v2/chatbox/chats/rooms/{room}/unpin`
- `POST v2/chatbox/chats/{conversationContext}/archive`
- `POST v2/chatbox/chats/{conversationContext}/pin`
- `POST v2/chatbox/chats/{conversationContext}/unarchive`
- `POST v2/chatbox/chats/{conversationContext}/unpin`

**v3/kpi** (7)

- `GET v3/kpi/cohort/invite-payment`
- `GET v3/kpi/cohort/invite-payment/excel`
- `GET v3/kpi/cohort/payments`
- `GET v3/kpi/cohort/payments/excel`
- `GET v3/kpi/cohort/signups`
- `GET v3/kpi/cohort/signups/excel`
- `GET v3/kpi/reports/metrics`

**contents/{content}** (6)

- `DELETE contents/{content}`
- `GET contents/{content}`
- `GET contents/{content}/medias`
- `POST contents/{content}/medias`
- `POST contents/{content}/publish-jobs`
- `PUT/PATCH contents/{content}`

**publish-jobs/{publish_job}** (6)

- `DELETE publish-jobs/{publish_job}`
- `GET publish-jobs/{publish_job}`
- `GET publish-jobs/{publish_job}/metrics`
- `POST publish-jobs/{publish_job}/complete`
- `POST publish-jobs/{publish_job}/unpublish`
- `PUT/PATCH publish-jobs/{publish_job}`

**campaigns/{campaign}** (6)

- `DELETE campaigns/{campaign}`
- `GET campaigns/{campaign}`
- `GET campaigns/{campaign}/metrics`
- `POST campaigns/{campaign}/ads`
- `POST campaigns/{campaign}/close`
- `PUT/PATCH campaigns/{campaign}`

**ads/{ad}** (6)

- `DELETE ads/{ad}`
- `GET ads/{ad}`
- `GET ads/{ad}/metrics`
- `GET ads/{ad}/time-tables`
- `POST ads/{ad}/time-tables`
- `PUT/PATCH ads/{ad}`

**v1/elastic** (6)

- `GET v1/elastic/active`
- `POST v1/elastic/find-integration/{id}`
- `POST v1/elastic/search`
- `POST v1/elastic/search-influencer`
- `POST v1/elastic/test`
- `POST v1/elastic/update-language`

**v2/contents** (6)

- `DELETE v2/contents/{content}`
- `GET v2/contents`
- `GET v2/contents/created`
- `GET v2/contents/{content}`
- `POST v2/contents`
- `PUT v2/contents/{content}`

**v2/offers** (6)

- `GET v2/offers/revise/{offer_id}/show/{revision_id}`
- `GET v2/offers/{offer}/questionnaire`
- `GET v2/offers/{state}/count`
- `POST v2/offers/revise/{offer_id}/accept/{revision_id}`
- `POST v2/offers/revise/{offer_id}/reject/{revision_id}`
- `POST v2/offers/{offer}/questionnaire`

**v2/twilio-migration** (5)

- `GET v2/twilio-migration/authors/{id}`
- `GET v2/twilio-migration/offers`
- `GET v2/twilio-migration/offers/{chatId}/exists`
- `GET v2/twilio-migration/temporary-offers`
- `GET v2/twilio-migration/temporary-offers/{chatId}/exists`

**v2/integrations** (5)

- `GET v2/integrations/{id}/pricing`
- `GET v2/integrations/{id}/profile`
- `PATCH v2/integrations/{id}/onboarded`
- `POST v2/integrations/{id}/sync`
- `POST v2/integrations/{id}/update-elastic`

**v1/offer** (4)

- `GET v1/offer/index`
- `GET v1/offer/single`
- `POST v1/offer/latest`
- `POST v1/offer/most-used-hashtags`

**v2/users** (4)

- `GET v2/users/{user}/checkouts`
- `GET v2/users/{user}/transaction-history`
- `POST v2/users/{user}/payouts`
- `POST v2/users/{wallet}/charges`

**v2/extensions** (4)

- `GET v2/extensions/chrome/campaigns`
- `POST v2/extensions/chrome/auth/login`
- `POST v2/extensions/chrome/auth/uploader/login`
- `POST v2/extensions/chrome/publish-jobs/{id}/content`

**v2/ai** (3)

- `GET v2/ai/offers/{offerId}/flag-check`
- `POST v2/ai/offers`
- `PUT v2/ai/influencers/{id}`

**auth/password** (3)

- `POST auth/password/change`
- `POST auth/password/forgot`
- `POST auth/password/reset`

**users/{user}** (3)

- `GET users/{user}`
- `GET users/{user}/wallet`
- `PUT/PATCH users/{user}`

**categories/{category}** (3)

- `DELETE categories/{category}`
- `GET categories/{category}`
- `PUT/PATCH categories/{category}`

**medias/{media}** (3)

- `DELETE medias/{media}`
- `GET medias/{media}`
- `PUT/PATCH medias/{media}`

**integrations/lookup** (3)

- `GET integrations/lookup/{type}/hashtag/{hashtag}`
- `GET integrations/lookup/{type}/profile/{username}`
- `POST integrations/lookup/{type}`

**integration_pricing/{integration_pricing}** (3)

- `DELETE integration_pricing/{integration_pricing}`
- `GET integration_pricing/{integration_pricing}`
- `PUT/PATCH integration_pricing/{integration_pricing}`

**time-tables/{time_table}** (3)

- `DELETE time-tables/{time_table}`
- `GET time-tables/{time_table}`
- `PUT/PATCH time-tables/{time_table}`

**invitations/{invitation}** (3)

- `DELETE invitations/{invitation}`
- `GET invitations/{invitation}`
- `POST invitations/{invitation}/see`

**methods/{method}** (3)

- `DELETE methods/{method}`
- `GET methods/{method}`
- `PUT/PATCH methods/{method}`

**tasks/{task}** (3)

- `DELETE tasks/{task}`
- `GET tasks/{task}`
- `PUT/PATCH tasks/{task}`

**v2/categories** (3)

- `DELETE v2/categories/{category}`
- `GET v2/categories/{category}`
- `PUT v2/categories/{category}`

**v2/paypal** (3)

- `GET v2/paypal/{wallet}/connect`
- `POST v2/paypal/connect/directly`
- `POST v2/paypal/redirect`

**v2/settings** (3)

- `GET v2/settings/influencer-collaboration-types`
- `GET v2/settings/{name}`
- `PUT v2/settings/{name}`

**user-third-party/shopify** (3)

- `GET user-third-party/shopify/v1/products/{id}`
- `POST user-third-party/shopify/v1/complete-fulfill`
- `POST user-third-party/shopify/v1/delete-tracking-info/{id}`

**auth/email** (2)

- `POST auth/email/resend`
- `POST auth/email/verify`

**users/instagram** (2)

- `POST users/instagram/{username}/request-verification`
- `POST users/instagram/{username}/verify-token`

**categories** (2)

- `GET categories`
- `POST categories`

**countries/{country}** (2)

- `GET countries/{country}`
- `GET countries/{country}/states`

**states/{state}** (2)

- `GET states/{state}`
- `GET states/{state}/cities`

**identities** (2)

- `GET identities`
- `POST identities`

**contents** (2)

- `GET contents`
- `POST contents`

**integrations** (2)

- `GET integrations`
- `POST integrations`

**invitations/statics** (2)

- `POST invitations/statics`
- `POST invitations/statics/day`

**offers** (2)

- `GET offers`
- `POST offers`

**tasks** (2)

- `GET tasks`
- `POST tasks`

**notifications/{notification}** (2)

- `DELETE notifications/{notification}`
- `GET notifications/{notification}`

**v1/user** (2)

- `POST v1/user/integrations`
- `POST v1/user/wallet`

**v1/advertising** (2)

- `GET v1/advertising/index`
- `GET v1/advertising/last-front-ad`

**v2/invitations** (2)

- `POST v2/invitations/failed/export`
- `POST v2/invitations/statistics/export`

**v2/campaigns-affiliate** (2)

- `GET v2/campaigns-affiliate`
- `GET v2/campaigns-affiliate/aggregated`

**v2/mobile-app** (2)

- `GET v2/mobile-app/reviews/can-submit`
- `POST v2/mobile-app/reviews`

**v2/guest** (2)

- `GET v2/guest/campaigns/contents/download/{contentId}/files/{fileId}`
- `GET v2/guest/{code}`

**v2/3rd-party** (2)

- `GET v2/3rd-party/{driver}/verify`
- `POST v2/3rd-party/login`

**product-seedings/{id}** (2)

- `GET product-seedings/{id}/influencers/{influencerId}/timeline`
- `PUT product-seedings/{id}`

**user-third-party/brandid** (2)

- `GET user-third-party/brandid/v1/billboard`
- `GET user-third-party/brandid/v1/tips`

**finance/current-user** (2)

- `GET finance/current-user/orders/{order}`
- `POST finance/current-user/orders/refund`

**v2/translations** (2)

- `POST v2/translations/contents/create`
- `POST v2/translations/{id}/retry`

**api** (1)

- `GET api`

**auth/register** (1)

- `POST auth/register`

**auth/login** (1)

- `POST auth/login`

**auth/refresh** (1)

- `POST auth/refresh`

**auth/firebase** (1)

- `POST auth/firebase`

**auth/logout** (1)

- `POST auth/logout`

**batch** (1)

- `POST batch`

**categories/hashtag** (1)

- `GET categories/hashtag/{hashtag}`

**countries** (1)

- `GET countries`

**city/{city}** (1)

- `GET city/{city}`

**influencers** (1)

- `GET influencers`

**influencers/add-external** (1)

- `POST influencers/add-external`

**files** (1)

- `DELETE files`

**files/{file_name}** (1)

- `GET files/{file_name}`

**medias** (1)

- `GET medias`

**contents/created** (1)

- `GET contents/created`

**publish-jobs** (1)

- `GET publish-jobs`

**ads/created** (1)

- `GET ads/created`

**ads** (1)

- `GET ads`

**ads/{brand_identity_id}** (1)

- `GET ads/{brand_identity_id}/compaigns_ads`

**ads/update-filters** (1)

- `PATCH ads/update-filters/{id}`

**l/{shortened}** (1)

- `GET l/{shortened}`

**invitations/created** (1)

- `GET invitations/created`

**invitations/received** (1)

- `GET invitations/received`

**invitations** (1)

- `POST invitations`

**invitations/public** (1)

- `GET invitations/public/{invitation}`

**invitations/changestatus** (1)

- `GET invitations/changestatus/{inviteid}/{status}`

**invitations/set_invitationid_user** (1)

- `GET invitations/set_invitationid_user/{inviteid}/{userid}`

**offers/{brand_id}** (1)

- `GET offers/{brand_id}/ongoing-offers`

**offers/{offer_id}** (1)

- `PATCH offers/{offer_id}/set_state`

**notifications/read** (1)

- `POST notifications/read`

**notifications** (1)

- `GET notifications`

**wp/{query}** (1)

- `GET wp/{query}`

**v1/ads** (1)

- `POST v1/ads/get-ads`

**v1/offers** (1)

- `POST v1/offers/get-offer`

**v1/contents** (1)

- `POST v1/contents/get-data`

**v1/nodejs** (1)

- `POST v1/nodejs/command`

**testing/cicd** (1)

- `GET testing/cicd/test`

**testing/flag** (1)

- `POST testing/flag/custom`

**mobile/startup** (1)

- `GET mobile/startup/requirements`

**subscriptions/pause** (1)

- `POST subscriptions/pause`

**subscriptions/unpause** (1)

- `POST subscriptions/unpause`

**integrations/my-unlocked-usernames** (1)

- `GET integrations/my-unlocked-usernames`

**integrations/reveal-charges** (1)

- `GET integrations/reveal-charges`

**v2/kpi** (1)

- `POST v2/kpi/data/refactor`

**v2/transactions** (1)

- `GET v2/transactions`

**v2/payments** (1)

- `GET v2/payments`

**v2/payouts** (1)

- `GET v2/payouts`

**v2/auth** (1)

- `POST v2/auth/logout`

**v2/dashboard** (1)

- `POST v2/dashboard/data`

**v2/campaigns** (1)

- `POST v2/campaigns/{ad}/restore`

**v2/notifications** (1)

- `GET v2/notifications/{notification}`

**v2/payment** (1)

- `GET v2/payment/test`

**v2/reviews** (1)

- `GET v2/reviews/{id}`

**v2/influencer** (1)

- `POST v2/influencer/email/confirm`

**v2/favorite-lists** (1)

- `GET v2/favorite-lists/{id}`

**v2/refresh-conversation-token** (1)

- `POST v2/refresh-conversation-token`

**v2/aigrow** (1)

- `POST v2/aigrow/ig/feed`

**v2/platforms** (1)

- `GET v2/platforms/{id}`

**v2/data** (1)

- `GET v2/data/show/{slug}`

**v2/influencers** (1)

- `POST v2/influencers/explore`

**v2/user-settings** (1)

- `GET v2/user-settings/integrations/{integration}`

**v3/wallet** (1)

- `POST v3/wallet/{gateway}/charge`

**v3/dashboard** (1)

- `GET v3/dashboard/overview`

**v1/tracker** (1)

- `GET v1/tracker/{action}/{uuid}`

**finance/stripe** (1)

- `GET finance/stripe/connect`

**finance/gateways** (1)

- `GET finance/gateways/directs`

**v2/translation-requests** (1)

- `GET v2/translation-requests/{id}/original/{fileId}/download`

**v3/locations** (1)

- `GET v3/locations/cities`

**internal/v1** (1)

- `POST internal/v1/influencers/search`

**v1/advance-analytics** (1)

- `GET v1/advance-analytics`

**v1/public** (1)

- `GET v1/public/advance-analytics`

</details>

## 5. Affiliate platform (affiliate/*)

Full lists: [aff-seller.md](aff-seller.md) §5, [aff-affiliate.md](aff-affiliate.md) §5, [aff-buyer.md](aff-buyer.md) §5, [aff-backend.md](aff-backend.md) §3 and §6.

- **Not deployed**: `spa-buyer` (replaced by the Next.js buyer site) and `spa-landpage`. Jenkins builds only the seller, affiliate and admin SPAs.
- **Seller app**: 143 of 838 files unreachable; 57 dead call sites. "Remember for 30 days" does nothing; the email-campaign schedule time is never sent; the YouTube welcome modal never shows.
- **Affiliate app**: 44 of 331 files unreachable; 86 dead call records, including the team-management tab (seller-only) and the Amazon/Shopify account block. The "Auto System-Generated Partnership Requests" switch saves nothing. One Click Promote has no inbound link.
- **Buyer site**: the reset-password dialog is never wired in; `/dashboard` and `/saved-products` in the avatar menu don't exist.
- **Backend jobs that do nothing**: `aggregate:notification` (body commented out), `payment:send-stripe-connect-nudges` (config keys missing). Admin bulk messages and seller email campaigns are never sent, because their processing commands are commented out of the scheduler.
- **Backend notifications never sent**: `AffiliateDailyActivityNotification`, `SellerDailyActivityNotification`, `ApprovedBrandPartnershipRequestNotification`, `RejectedBrandPartnershipRequestNotification`, `RejectedProductPartnershipRequestNotification`, `Seller\ApprovedPartnershipRequestNotification`, `Seller\RejectedPartnershipRequestNotification`, `VerifyMessageNotification`. Payment success/failure produce no email (web notification only).
- **Shopify `orders/create` webhook**: log only.

**177 affiliate backend routes are called by nothing.** The largest groups: `buyer/v1/*` (the old Vue buyer app's account API: profile, chats, tickets, notifications, settings), `affiliate/v1/teams/*` (team management is seller-only), `*/api-keys`, and older partnership, product and landing-page endpoints.

<details><summary>Full list of unused affiliate routes, grouped by prefix (177)</summary>


**affiliate/v1/teams** (17)

- `DELETE affiliate/v1/teams/{teamId}/members/delete/{memberId}`
- `DELETE affiliate/v1/teams/{teamId}/roles/delete/{roleId}`
- `GET affiliate/v1/teams/invitations/list`
- `GET affiliate/v1/teams/my`
- `GET affiliate/v1/teams/permissions/list/{roleId?}`
- `GET affiliate/v1/teams/{teamId}`
- `GET affiliate/v1/teams/{teamId}/members/detail/{memberId}`
- `GET affiliate/v1/teams/{teamId}/members/list`
- `GET affiliate/v1/teams/{teamId}/roles/detail/{roleId}`
- `GET affiliate/v1/teams/{teamId}/roles/list`
- `PATCH affiliate/v1/teams/invitations/{invitationId}/change-status`
- `POST affiliate/v1/teams/{teamId}/invitation`
- `POST affiliate/v1/teams/{teamId}/members/cancel/{memberId}`
- `POST affiliate/v1/teams/{teamId}/roles/assign/{roleId}/{teamMemberId}`
- `POST affiliate/v1/teams/{teamId}/roles/create`
- `PUT affiliate/v1/teams/{teamId}/members/update/{memberId}`
- `PUT affiliate/v1/teams/{teamId}/roles/update/{roleId}`

**buyer/v1/profile** (16)

- `DELETE buyer/v1/profile/sessions/all`
- `DELETE buyer/v1/profile/sessions/{sessionId}`
- `DELETE buyer/v1/profile/synced-accounts/{id}`
- `GET buyer/v1/profile`
- `GET buyer/v1/profile/questions`
- `GET buyer/v1/profile/sessions`
- `GET buyer/v1/profile/show-register-steps`
- `GET buyer/v1/profile/synced-accounts`
- `POST buyer/v1/profile/questions`
- `POST buyer/v1/profile/request-change-mobile`
- `POST buyer/v1/profile/request-change-password`
- `POST buyer/v1/profile/sync/account/amazon`
- `POST buyer/v1/profile/sync/account/facebook`
- `POST buyer/v1/profile/sync/account/google`
- `PUT buyer/v1/profile/change-mobile`
- `PUT buyer/v1/profile/change-password`

**buyer/v1/chats** (13)

- `DELETE buyer/v1/chats/contacts/tag`
- `DELETE buyer/v1/chats/messages/{messageId}`
- `GET buyer/v1/chats/contacts`
- `GET buyer/v1/chats/contacts/{userTypeId}`
- `GET buyer/v1/chats/contacts/{userTypeId}/messages`
- `GET buyer/v1/chats/find-contact/{contactId}`
- `GET buyer/v1/chats/messages`
- `GET buyer/v1/chats/messages/saved`
- `POST buyer/v1/chats/contacts/tag`
- `POST buyer/v1/chats/messages`
- `POST buyer/v1/chats/messages/find-page`
- `POST buyer/v1/chats/messages/saved`
- `PUT buyer/v1/chats/read-contact-messages`

**buyer/v1/auth** (12)

- `POST buyer/v1/auth/email-verify`
- `POST buyer/v1/auth/forget-password`
- `POST buyer/v1/auth/login`
- `POST buyer/v1/auth/login/amazon`
- `POST buyer/v1/auth/login/facebook`
- `POST buyer/v1/auth/login/google`
- `POST buyer/v1/auth/mobile-verification-code`
- `POST buyer/v1/auth/mobile-verify`
- `POST buyer/v1/auth/register`
- `POST buyer/v1/auth/reset-password`
- `POST buyer/v1/auth/skip-step`
- `POST buyer/v1/auth/verification-code`

**affiliate/v1/partnerships** (11)

- `GET affiliate/v1/partnerships/applied/brand`
- `GET affiliate/v1/partnerships/applied/product`
- `GET affiliate/v1/partnerships/brands/list`
- `GET affiliate/v1/partnerships/contract/brand`
- `GET affiliate/v1/partnerships/contract/product`
- `GET affiliate/v1/partnerships/contracts/count`
- `GET affiliate/v1/partnerships/invited/brand`
- `GET affiliate/v1/partnerships/invited/product`
- `GET affiliate/v1/partnerships/products/list`
- `GET affiliate/v1/partnerships/requests/count`
- `POST affiliate/v1/partnerships/brand/check-pre-request`

**affiliate/v1/products** (8)

- `DELETE affiliate/v1/products/links/sources/{productLinkSourceId}`
- `GET affiliate/v1/products/brands/drop-down/partnered`
- `GET affiliate/v1/products/brands/{id}/sale-detail`
- `GET affiliate/v1/products/brands/{id}/sale-detail/export`
- `GET affiliate/v1/products/drop-down/partnered`
- `GET affiliate/v1/products/links`
- `POST affiliate/v1/products/links/set-default/{productLinkId}`
- `PUT affiliate/v1/products/links/sources/{productLinkSourceId}`

**affiliate/v1/landing-pages** (6)

- `DELETE affiliate/v1/landing-pages/{landingPageId}/assignees/{landingPageAssigneeId}`
- `DELETE affiliate/v1/landing-pages/{landingPageId}/collections/{landingPageCollectionId}`
- `GET affiliate/v1/landing-pages/templates`
- `POST affiliate/v1/landing-pages/{landingPageId}/assignees`
- `POST affiliate/v1/landing-pages/{landingPageId}/collections`
- `PUT affiliate/v1/landing-pages/{landingPageId}/collections/{landingPageCollectionId}`

**affiliate/v1/api-keys** (5)

- `DELETE affiliate/v1/api-keys/{id}`
- `GET affiliate/v1/api-keys`
- `GET affiliate/v1/api-keys/{id}`
- `POST affiliate/v1/api-keys`
- `PUT affiliate/v1/api-keys/{id}`

**buyer/v1/notifications** (5)

- `GET buyer/v1/notifications`
- `GET buyer/v1/notifications/suggests`
- `PATCH buyer/v1/notifications/read-all`
- `PATCH buyer/v1/notifications/{id}/read`
- `PATCH buyer/v1/notifications/{id}/toggle-bookmark`

**buyer/v1/tickets** (5)

- `GET buyer/v1/tickets`
- `GET buyer/v1/tickets/{ticket}`
- `PATCH buyer/v1/tickets/{ticket}`
- `POST buyer/v1/tickets`
- `POST buyer/v1/tickets/{ticket}/messages`

**seller/v1/reports** (5)

- `GET seller/v1/reports/performance/detail/{date}`
- `GET seller/v1/reports/performance/detail/{date}/export`
- `GET seller/v1/reports/sale`
- `GET seller/v1/reports/sale/chart`
- `POST seller/v1/reports/sale/export`

**seller/v1/api-keys** (5)

- `DELETE seller/v1/api-keys/{id}`
- `GET seller/v1/api-keys`
- `GET seller/v1/api-keys/{id}`
- `POST seller/v1/api-keys`
- `PUT seller/v1/api-keys/{id}`

**affiliate/v1/profile** (4)

- `DELETE affiliate/v1/profile/sessions/all`
- `POST affiliate/v1/profile/request-change-password`
- `POST affiliate/v1/profile/sync/account/amazon`
- `PUT affiliate/v1/profile/change-password`

**affiliate/v1/associates** (4)

- `DELETE affiliate/v1/associates/associate-social-medias/delete/{id}`
- `GET affiliate/v1/associates/associate-brands/show/{id}`
- `GET affiliate/v1/associates/associate-social-medias/show/{id}`
- `PUT affiliate/v1/associates/associate-social-medias/update/{id}`

**buyer/v1/settings** (4)

- `GET buyer/v1/settings/client`
- `GET buyer/v1/settings/notifications`
- `POST buyer/v1/settings/client/store`
- `PUT buyer/v1/settings/notifications/update`

**seller/v1/partnership** (4)

- `GET seller/v1/partnership/applied/product`
- `GET seller/v1/partnership/contracts/count`
- `GET seller/v1/partnership/invited/product`
- `GET seller/v1/partnership/requests/count`

**client/buyer/v1** (4)

- `POST client/buyer/v1/auth/mobile-verification-code`
- `POST client/buyer/v1/auth/mobile-verify`
- `POST client/buyer/v1/auth/reset-password`
- `POST client/buyer/v1/auth/skip-step`

**affiliate/v1/chats** (3)

- `DELETE affiliate/v1/chats/messages/{messageId}`
- `GET affiliate/v1/chats/contacts/{userTypeId}`
- `POST affiliate/v1/chats/messages/send-as-bulk`

**affiliate/v1/dashboard** (3)

- `GET affiliate/v1/dashboard/report/brands`
- `GET affiliate/v1/dashboard/report/contracts`
- `GET affiliate/v1/dashboard/report/products`

**seller/v1/profile** (3)

- `GET seller/v1/profile/payments/accounts`
- `POST seller/v1/profile/request-change-password`
- `PUT seller/v1/profile/change-password`

**affiliate/v1/auth** (2)

- `POST affiliate/v1/auth/email-verify`
- `POST affiliate/v1/auth/verification-code`

**affiliate/v1/user-type-lists** (2)

- `GET affiliate/v1/user-type-lists/{brandId}/brand-membership`
- `GET affiliate/v1/user-type-lists/{productId}/product-membership`

**affiliate/v1/amazon-marketplaces** (2)

- `GET affiliate/v1/amazon-marketplaces/list`
- `GET affiliate/v1/amazon-marketplaces/show/{id}`

**buyer/v1/questions** (2)

- `GET buyer/v1/questions`
- `POST buyer/v1/questions`

**seller/v1/auth** (2)

- `POST seller/v1/auth/email-verify`
- `POST seller/v1/auth/verification-code`

**seller/v1/chats** (2)

- `DELETE seller/v1/chats/messages/{messageId}`
- `GET seller/v1/chats/contacts/{userTypeId}`

**seller/v1/products** (2)

- `POST seller/v1/products/brands/{id}`
- `POST seller/v1/products/{id}/sync-request`

**seller/v1/promotions** (2)

- `PATCH seller/v1/promotions/{promotionId}/accessibility`
- `PATCH seller/v1/promotions/{promotionId}/update-status`

**seller/v1/user-type-lists** (2)

- `GET seller/v1/user-type-lists/drop-down-unlisted-affiliates`
- `GET seller/v1/user-type-lists/{affiliateId}/affiliate-membership`

**seller/v1/teams** (2)

- `GET seller/v1/teams/{teamId}/members/detail/{memberId}`
- `POST seller/v1/teams/{teamId}/roles/assign/{roleId}/{teamMemberId}`

**seller/v1/wallet** (2)

- `GET seller/v1/wallet/checkout/cancel`
- `GET seller/v1/wallet/checkout/success`

**affiliate/v1/social-medias** (1)

- `GET affiliate/v1/social-medias/type-categories`

**affiliate/v1/stores** (1)

- `GET affiliate/v1/stores/{id}`

**affiliate/v1/reports** (1)

- `POST affiliate/v1/reports`

**affiliate/v1/promotions** (1)

- `GET affiliate/v1/promotions`

**affiliate/v1/payments** (1)

- `GET affiliate/v1/payments/subscription-transactions`

**affiliate/v1/message-templates** (1)

- `GET affiliate/v1/message-templates/detail/{id}`

**affiliate/v1/message-variables** (1)

- `GET affiliate/v1/message-variables/detail/{id}`

**buyer/v1/categories** (1)

- `GET buyer/v1/categories`

**buyer/v1/products** (1)

- `GET buyer/v1/products/{id}`

**buyer/v1/system** (1)

- `GET buyer/v1/system/basic-info`

**buyer/v1/guidelines** (1)

- `GET buyer/v1/guidelines/{key}`

**seller/v1/discover** (1)

- `GET seller/v1/discover/drop-down-affiliates`

**seller/v1/payments** (1)

- `GET seller/v1/payments/subscription-transactions`

**seller/v1/message-templates** (1)

- `GET seller/v1/message-templates/detail/{id}`

**seller/v1/message-variables** (1)

- `GET seller/v1/message-variables/detail/{id}`

**seller/v1/smtp-connections** (1)

- `GET seller/v1/smtp-connections/detail/{id}`

**seller/v1/email-campaigns** (1)

- `GET seller/v1/email-campaigns/show/{id}`

**seller/v1/integrations** (1)

- `GET seller/v1/integrations/ainfluencer/campaigns/list`

</details>
