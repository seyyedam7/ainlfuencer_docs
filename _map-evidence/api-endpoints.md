# Backend endpoint inventory (core/backend/api)

Generated 2026-10-05 from a route dump of `core/backend/api` master (`old-origin/master` 99921fbce, 2026-10-04): 1,148 routes.

Each route is classified by who calls it. Client calls come from static analysis of the **reachable** code of the brand web app (master) and the influencer mobile app (develop, v32.2.2). See [web.md](web.md) and [mobile.md](mobile.md).

`{locale}` is the language segment; the clients use `en`.

| Class | Meaning | Routes |
|---|---|---|
| client | Called by live code in the web or mobile app | 365 |
| webhook | Called by a third party, the chat service, or a browser returning from a payment or OAuth provider | 33 |
| admin | Admin panel only (admin middleware or an `admin` path). Out of scope for this map | 188 |
| other-client | Called only by out-of-scope apps (core/frontend/admin, brand-mobile-app, affiliate). Out of scope for this map | 88 |
| web-page | Server-rendered site pages (SEO, sitemaps, downloads). Out of scope for this map | 61 |
| unused | Nothing in any client calls it. Excluded from the map (see [excluded-dead-code.md](excluded-dead-code.md)) | 413 |

## 1. Live endpoints by topic

Topic IDs match [../flow-map.md](../flow-map.md). The Caller column lists up to 4 web pages or mobile screens that call each endpoint.

### ACC-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/v3/mobile/register` | AuthController@register | mobile: SIGN_UP_SCREEN |
| POST | `/api/{locale}/v3/mobile/resend-email-verification` | AuthController@resendEmailVerification | mobile: CONFIRM_EMAIL_SCREEN |
| POST | `/api/{locale}/v3/mobile/verify-email` | AuthController@verifyEmail | mobile: CONFIRM_EMAIL_SCREEN |
| POST | `/api/v2/auth/email/resend` | AuthController@resendEmail | web: /affiliate/invitation, /gifting/invitation, /invitation, /verify |
| POST | `/api/v2/auth/email/verify` | AuthController@verifyEmail | web: /affiliate/invitation, /gifting/invitation, /invitation, /verify |
| POST | `/api/v2/auth/register` | AuthController@register | web: /register |
| POST | `/api/v2/email/confirm` | EmailVerificationController@verifyBrand | web: /auto-verify, /verifybylink/:token |
| POST | `/api/v2/guest/signup` | SignupController@signup | web: /affiliate/invitation, /gifting/invitation, /invitation |
| POST | `/api/v2/webhook/signup` | SignupController@signup | webhook / callback |

### ACC-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/v3/mobile/auth/social/{provider}` | SocialAuthController@store | mobile: LOGIN_SCREEN, SIGN_UP_SCREEN |
| POST | `/api/{locale}/v3/mobile/login` | AuthController@login | mobile: LOGIN_SCREEN |
| POST | `/api/{locale}/v3/mobile/refresh-token` | AuthController@refreshToken | mobile: (global) |
| GET | `/api/{locale}/v3/mobile/versions` | VersionController@versions | mobile: SPLASH_SCREEN |
| POST | `/api/tmp-link/verify` | TmpLinkController@verify | web: /tmp-link |
| POST | `/api/v2/3rd-party/verify-auth` | AuthController@verifyAuth | web: /login, /register |
| POST | `/api/v2/auth/login` | AuthController@login | web: /login |
| POST | `/api/v2/auth/refresh-token` | AuthController@refreshToken | web: /, /account/:id, /accountnew/:id, /affiliate/affiliate-campaign-list |

### ACC-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/v3/mobile/auth/forgot-password/forgot` | AuthController@sendForgotPasswordCode | mobile: CHANGE_EMAIL_SCREEN, CONFIRM_EMAIL_SCREEN, FORGET_PASSWORD_SCREEN |
| POST | `/api/{locale}/v3/mobile/auth/forgot-password/reset` | AuthController@resetPasswordWithCode | mobile: RESET_PASSWORD_SCREEN |
| POST | `/api/{locale}/v3/mobile/auth/forgot-password/verify` | AuthController@verifyResetPasswordCode | mobile: CONFIRM_EMAIL_SCREEN |
| POST | `/api/{locale}/v3/mobile/change-password` | AuthController@changePassword | mobile: CHANGE_PASSWORD |
| POST | `/api/v2/auth/forgot-password` | AuthController@forgot_password | web: /forgot-password |
| POST | `/api/v2/auth/reset-password` | AuthController@reset_password | web: /reset/password/:token |
| POST | `/api/v2/user/change-password` | InformationController@changePassword | web: /profile, /profile/change-password |
| POST | `/api/v2/user/set-password` | UserController@setPassword | web: /oauth/set-password |

### ACC-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/v3/mobile/change-email` | AuthController@changeEmail | mobile: CHANGE_EMAIL_SCREEN |
| PUT | `/api/v2/user/profile` | InformationController@update | web: /change-email |

### ACC-05

| Method | Path | Controller | Caller |
|---|---|---|---|
| PUT | `/api/{locale}/v3/mobile/user/profile` | UserController@updateProfile | mobile: EDIT_ACCOUNT_SCREEN |
| GET | `/api/{locale}/v3/mobile/user/profile` | UserController@showProfile | mobile: CHAT_SCREEN |
| DELETE | `/api/{locale}/v3/mobile/user/profile/remove` | UserController@removeProfile | mobile: SETTINGS_SCREEEN |
| GET | `/api/v2/user/current-user` | UserController@currentUser | web: /oauth/set-password |
| DELETE | `/api/v2/user/current-user` | UserController@deleteCurrentUser | web: /profile |

### BRD-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/v2/funnels` | FunnelsController@storeLog | web: /create-campaign, /onboarding |
| GET | `/api/v2/funnels/processes` | FunnelsController@processList | web: /create-campaign, /onboarding |
| POST | `/api/v2/phone-verification/attempt` | AuthController@phoneVerificationAttempt | web: /verify-phone-number |
| POST | `/api/v2/phone-verification/verify` | AuthController@finalizePhoneVerification | web: /verify-phone-number |
| PUT | `/api/v2/user/brand-information` | BrandInformationController@updateBrandInformation | web: /find/influencer-explorer |
| GET | `/api/v2/user/brand-information` | BrandInformationController@getBrandInformation | web: (global), /, /account/:id, /accountnew/:id, /affiliate/affiliate-campaign-list, /campaign/discover, /find/influencer-explorer |
| POST | `/api/v3/brand/on-boarding` | BrandOnBoardingController@store | web: /create-campaign, /onboarding |
| GET | `/api/v3/brand/on-boarding` | BrandOnBoardingController@index | web: /create-campaign, /login, /onboarding, /profile |

### BRD-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/v2/digital-assets` | DigitalAssetController@getUserFiles | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/v2/settings/brand/tutorial/videos` | SettingController@getBrandVideoTutorials | web: / |
| GET | `/api/v2/settings/brand/tutorial/videos/in-chat` | SettingController@getInChatBrandVideoTutorials | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/v3/brand/dashboard` | BrandDashboardController@allData | web: / |
| PUT | `/api/v3/brand/profile` | ProfileController@update | web: /profile |
| GET | `/api/v3/brand/profile/payment-information` | ProfileController@paymentInformation | web: /profile/transactions, /transactions2 |
| GET | `/api/v3/brand/profile/score-activities` | ProfileController@scoreActivities | web: /profile/points |
| GET | `/api/v3/brand/profile/total-point` | ProfileController@totalPoint | web: /profile/points |
| GET | `/api/v3/brand/profile/transactions` | ProfileController@transactions | web: /profile/transactions, /transactions2 |
| PATCH | `/api/v3/brand/profile/{brandIdentityId}/change-logo` | ProfileController@changeLogo | web: /profile |

### BRD-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/v2/teams/brand/members` | MemberController@index | web: /, /team-members |
| DELETE | `/api/v2/teams/brand/members/invitations/{invitationId}` | MemberInvitationController@deleteMemberInvitation | web: /team-members |
| POST | `/api/v2/teams/brand/members/invite` | MemberInvitationController@invite | web: /, /team-members |
| POST | `/api/v2/teams/brand/members/register` | MemberController@register | web: /teams/members |
| DELETE | `/api/v2/teams/brand/members/{memberId}` | MemberController@deleteMember | web: /team-members |

### BRD-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/v2/brand/suspension-appeal` | SuspensionAppealController@storeBrandAppeal | web: /forgot-password, /login, /register |

### CHAT-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/chats` | ChatConversationController@index | mobile: AFFILIATE_DETAIL, BRAND_PROFILE, CAMPAIGN_DETAILS, CHAT_SCREEN |
| POST | `/api/{locale}/v3/mobile/chats/affiliate/{adId}/send-address` | ChatConversationController@sendAffiliateAddress | mobile: CHAT_SCREEN, GIFTING_DETAIL, SETTINGS_SHIPPING_ADDRESS, SETTINGS_SHIPPING_ADDRESS_FORM |
| POST | `/api/{locale}/v3/mobile/chats/messages` | ChatConversationController@sendMessage | mobile: AFFILIATE_DETAIL, GIFTING_DETAIL |
| POST | `/api/{locale}/v3/mobile/chats/rooms/{room}/pin` | ChatConversationController@pinRoom | mobile: CAMPAIGN_DETAILS, CHAT_SCREEN, CHAT_THREADS_SCREEN, INBOX_SCREEN |
| POST | `/api/{locale}/v3/mobile/chats/rooms/{room}/unpin` | ChatConversationController@unpinRoom | mobile: CAMPAIGN_DETAILS, CHAT_SCREEN, CHAT_THREADS_SCREEN, INBOX_SCREEN |
| GET | `/api/{locale}/v3/mobile/chats/{conversationContext}` | ChatConversationController@show | mobile: AFFILIATE_DETAIL, CHAT_SCREEN, GIFTING_DETAIL |
| POST | `/api/{locale}/v3/mobile/chats/{conversationContext}/archive` | ChatConversationController@archiveThread | mobile: CAMPAIGN_DETAILS, CHAT_SCREEN, CHAT_THREADS_SCREEN, INBOX_SCREEN |
| POST | `/api/{locale}/v3/mobile/chats/{conversationContext}/pin` | ChatConversationController@pinThread | mobile: CAMPAIGN_DETAILS, CHAT_SCREEN, CHAT_THREADS_SCREEN, INBOX_SCREEN |
| POST | `/api/{locale}/v3/mobile/chats/{conversationContext}/unarchive` | ChatConversationController@unarchiveThread | mobile: CAMPAIGN_DETAILS, CHAT_SCREEN, CHAT_THREADS_SCREEN, INBOX_SCREEN |
| POST | `/api/{locale}/v3/mobile/chats/{conversationContext}/unpin` | ChatConversationController@unpinThread | mobile: CAMPAIGN_DETAILS, CHAT_SCREEN, CHAT_THREADS_SCREEN, INBOX_SCREEN |
| POST | `/api/{locale}/v3/mobile/files/upload` | FileController@upload | mobile: CHAT_SCREEN, MAKE_OFFER_SCREEN, MEDIA_KIT, PHOTO_GALLERY |
| GET | `/api/{locale}/v3/mobile/message-templates` | MessageTemplateController@index | mobile: CHAT_SCREEN |
| POST | `/api/{locale}/v3/mobile/message-templates` | MessageTemplateController@store | mobile: CHAT_SCREEN |
| PATCH | `/api/{locale}/v3/mobile/message-templates/{id}` | MessageTemplateController@update | mobile: CHAT_SCREEN |
| DELETE | `/api/{locale}/v3/mobile/message-templates/{id}` | MessageTemplateController@delete | mobile: CHAT_SCREEN |
| POST | `/api/{locale}/v3/mobile/offers/{id}/actions/{action}/toggle` | OfferController@toggleAttributeByAction | mobile: BRAND_PROFILE, CHAT_SCREEN |
| POST | `/api/{locale}/v3/mobile/offers/{id}/send-address` | OfferController@sendAddress | mobile: CHAT_SCREEN, GIFTING_DETAIL, SETTINGS_SHIPPING_ADDRESS, SETTINGS_SHIPPING_ADDRESS_FORM |
| PUT | `/api/{locale}/v3/mobile/offers/{id}/state` | OfferController@updateState | mobile: BRAND_PROFILE, CHAT_SCREEN |
| GET | `/api/v2/chatbox/chats` | ChatConversationController@index | web: /, /new-offersV2, /new-offersV2/:id |
| POST | `/api/v2/chatbox/chats/messages` | ChatConversationController@sendMessage | web: /affiliate/dashboard, /campaigns/affiliate-influencers/:id, /campaigns/seeding-influencers/:id |
| GET | `/api/v2/chatbox/chats/{conversationContext}` | ChatConversationController@show | web: /affiliate/dashboard, /campaigns/affiliate-influencers/:id, /campaigns/seeding-influencers/:id |
| POST | `/api/v2/message-templates` | MessageTemplateController@store | web: /new-offersV2, /new-offersV2/:id, /offers |
| PATCH | `/api/v2/message-templates/{id}` | MessageTemplateController@update | web: /new-offersV2, /new-offersV2/:id, /offers |
| DELETE | `/api/v2/message-templates/{id}` | MessageTemplateController@delete | web: /new-offersV2, /new-offersV2/:id, /offers |
| POST | `/api/v2/offers/send-bulk-message` | OfferController@sendBulkMessage | web: /new-offersV2, /new-offersV2/:id, /offers |
| PATCH | `/api/v2/offers/{offer}/state` | OfferController@setState | web: /new-offersV2, /new-offersV2/:id, /new-offersV2/deals, /offers |
| POST | `/api/v2/refresh-twilio-token` | TwilioTokenController@refreshTwilioToken | web: (global) |
| GET | `/api/v3/brand/inbox/campaigns` | InboxController@campaigns | web: /new-offersV2, /new-offersV2/:id |

### CHAT-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/v2/chatbox/webhook` | WebhookController@webhook | webhook / callback |

### CMP-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/files` | FileController@upload | web: /affiliate/affiliate_campaign_details, /affiliate/product_seeding_wizard, /campaign/post-content, /create-campaign |
| POST | `/api/files/batch` | FileController@batchUpload | web: /new-offersV2, /new-offersV2/:id |
| POST | `/api/user-third-party/shopify/v1/campaign/finish-process` | ShopifyController@finishShopifyCampaign | web: /create-campaign |
| POST | `/api/user-third-party/shopify/v1/campaign/save-product` | ShopifyController@saveProduct | web: /create-campaign |
| POST | `/api/user-third-party/shopify/v1/campaign/start-process` | ShopifyController@startCampaign | web: /create-campaign, /new-offersV2, /new-offersV2/:id |
| POST | `/api/user-third-party/shopify/v1/connect` | ShopifyController@connect | web: /create-campaign |
| GET | `/api/v2/ai/campaign-creation/{adId}/title-and-description` | CampaignCreationController@titleAndDescription | web: /create-campaign |
| GET | `/api/v2/all-medias` | MediaController@allMedias | web: /campaign/post-content, /create-campaign, /new-offersV2, /new-offersV2/:id |
| GET | `/api/v2/all-videos` | MediaController@allVideos | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /profile/contents |
| POST | `/api/v2/campaign/create/content` | ContentController@store | web: /campaign/post-content |
| POST | `/api/v2/campaign/create/step1` | AdController@storeStep1 | web: /campaign/post-content |
| GET | `/api/v2/campaign/goal-questions` | AdController@getGoals | web: /campaign/objectives, /onboarding |
| GET | `/api/v2/campaigns/find-influencer-dropdown` | AdController@findInfluencerDropdown | web: /search |
| POST | `/api/v2/campaigns/progress-step` | AdController@progressStep | web: /create-campaign |
| GET | `/api/v2/campaigns/templates` | TemplateController@index | web: /campaign/marketplace |
| GET | `/api/v2/campaigns/templates-list` | TemplateController@templatesList | web: /create-campaign |
| POST | `/api/v2/campaigns/{ad}/influencer-filters` | AdController@storeInfluencerFilters | web: /campaign/discover |
| POST | `/api/v2/campaigns/{ad}/publish-terms` | AdController@storePublishingTerms | web: /campaign/publish-terms |
| PUT | `/api/v2/campaigns/{ad}/publish-terms` | AdController@updatePublishingTerms | web: /find/influencer-explorer, /find/influencer-explorer/custom-search |
| GET | `/api/v2/campaigns/{ad}/related-influencers` | AdInfluencerController@index | web: /new-offersV2/:id/payment-successful, /offers/:id/payment-successful |
| POST | `/api/v2/campaigns/{ad}/settings` | AdController@storeSettings | web: /find/influencer-explorer, /find/influencer-explorer/custom-search, /new-offersV2/deals, /offers |
| GET | `/api/v2/medias` | MediaController@index | web: /content-library |
| DELETE | `/api/v2/medias/{id}` | MediaController@delete | web: /campaign/post-content, /content-library, /create-campaign |
| POST | `/api/v3/campaigns` | AdCreationProcessController@store | web: /create-campaign |

### CMP-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/v2/campaigns` | AdController@index | web: /new-offersV2, /new-offersV2/:id, /offers |
| GET | `/api/v2/campaigns/{ad}` | AdController@show | web: /account/:id, /accountnew/:id, /affiliate/affiliate-campaign-list, /affiliate/dashboard |
| PUT | `/api/v2/campaigns/{ad}` | AdController@update | web: /account/:id, /accountnew/:id, /analytics, /campaign/discover |
| DELETE | `/api/v2/campaigns/{ad}` | AdController@destroy | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /analytics |
| POST | `/api/v2/campaigns/{ad}/duplicate` | AdController@duplicate | web: /campaigns |

### CMP-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| DELETE | `/api/user-third-party/ainfluencer-affiliate/v1/account/delete/{account_id}` | AinfluencerAffiliateProgramController@deleteAccount | web: /affiliate/dashboard, /affiliate/sync_account |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/affiliate-programs/list` | AinfluencerAffiliateProgramController@listAffiliateProgram | web: /affiliate/affiliate_campaign_products, /affiliate/affiliate_campaign_setup, /affiliate/dashboard, /affiliate/product_list |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/amazon-account/profiles/check/{account_id}` | AinfluencerAffiliateAmazonAccountController@checkAmazonAccountProfile | web: /affiliate/dashboard, /affiliate/sync_account |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/amazon-account/sync` | AinfluencerAffiliateAmazonAccountController@syncAmazonAccount | web: /affiliate/dashboard, /affiliate/product_seeding_wizard, /affiliate/sync_account |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/brands/amazon/drop-down` | AinfluencerAffiliateBrandController@listAmazonBrandDropdown | web: /affiliate/affiliate_campaign_products, /affiliate/product_list |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/brands/shopify/drop-down` | AinfluencerAffiliateBrandController@listShopifyBrandDropdown | web: /affiliate/affiliate_campaign_products, /affiliate/product_list |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/campaigns/assign-items` | AinfluencerAffiliateCampaignController@assignCampaignItems | web: /affiliate/affiliate_campaign_products, /affiliate/product_list |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/cards/list` | AinfluencerAffiliateCardController@ListCard | web: /affiliate/dashboard, /affiliate/epty_state|<unknown step> (legacy DashboardAffiliate), /affiliate/sync_account, /affiliate/sync_payment |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/cards/sync` | AinfluencerAffiliateCardController@syncCard | web: /affiliate/dashboard, /affiliate/epty_state|<unknown step> (legacy DashboardAffiliate), /affiliate/sync_account, /affiliate/sync_payment |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/products/amazon/bulk-generate-link` | AinfluencerAffiliateProductController@bulkGenerateLinkAmazonProduct | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/products/amazon/list` | AinfluencerAffiliateProductController@ListAmazonProduct | web: /affiliate/affiliate_campaign_products, /affiliate/product_list, /affiliate/product_seeding_wizard, /new-offersV2 |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/products/bulk-change-status` | AinfluencerAffiliateProductController@bulkChangeStatusProduct | web: /affiliate/affiliate_campaign_products, /affiliate/product_list |
| PUT | `/api/user-third-party/ainfluencer-affiliate/v1/products/bulk-update-commission` | AinfluencerAffiliateProductController@bulkUpdateCommissionProduct | web: /affiliate/affiliate_campaign_products, /affiliate/product_list |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/products/shopify/bulk-generate-link` | AinfluencerAffiliateProductController@bulkGenerateLinkShopifyProduct | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/products/shopify/list` | AinfluencerAffiliateProductController@ListShopifyProduct | web: /affiliate/affiliate_campaign_products, /affiliate/product_list, /affiliate/product_seeding_wizard, /new-offersV2 |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/shopify-account/stores/check/{account_id}` | AinfluencerAffiliateShopifyAccountController@checkShopifyAccountStore | web: /affiliate/dashboard, /affiliate/sync_account, /create-campaign, /new-offersV2 |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/shopify-account/stores/list` | AinfluencerAffiliateShopifyAccountController@listShopifyAccountStore | web: /affiliate/affiliate_campaign_setup, /create-campaign, /new-offersV2, /new-offersV2/:id |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/shopify-account/sync` | AinfluencerAffiliateShopifyAccountController@syncShopifyAccount | web: /affiliate/dashboard, /affiliate/product_seeding_wizard, /affiliate/sync_account, /create-campaign |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/sso/authorize-url` | AinfluencerAffiliateSsoController@authorizeUrl | web: /affiliate/dashboard, /affiliate/epty_state|<unknown step> (legacy DashboardAffiliate), /affiliate/sync_account |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/sso/callbacks/social-media-sync` | AinfluencerAffiliateSsoController@socialMediaSync | webhook / callback |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/sso/callbacks/user-registered` | AinfluencerAffiliateSsoController@userRegistered | webhook / callback |
| GET | `/api/user-third-party/ainfluencer-affiliate/v1/status` | AinfluencerAffiliateProgramController@status | web: /, /affiliate/dashboard, /affiliate/epty_state|<unknown step> (legacy DashboardAffiliate), /affiliate/select_campaign_type |
| POST | `/api/user-third-party/ainfluencer-affiliate/v1/webhook` | AinfluencerAffiliateWebhookController@handle | webhook / callback |
| GET | `/api/v1/custom-forms/{slug}` | CustomFormController@show | web: /affiliate/dashboard, /affiliate/product_seeding_wizard, /affiliate/sync_account |
| POST | `/api/v1/custom-forms/{slug}/submit` | CustomFormController@submit | web: /affiliate/dashboard, /affiliate/product_seeding_wizard, /affiliate/sync_account |
| GET | `/api/v2/campaigns-affiliate/{id}` | AdAffiliateController@show | web: /account/:id, /accountnew/:id, /affiliate/affiliate-campaign-list, /affiliate/affiliate_campaign_details, /affiliate/dashboard, /campaigns, /campaigns/affiliate-influencers/:id |
| PATCH | `/api/v2/campaigns-affiliate/{id}` | AdAffiliateController@update | web: /affiliate/affiliate-campaign-list, /affiliate/affiliate_campaign_details, /affiliate/dashboard |
| DELETE | `/api/v2/campaigns-affiliate/{id}` | AdAffiliateController@destroy | web: /affiliate/affiliate-campaign-list |
| GET | `/api/v2/campaigns-affiliate/{id}/influencers` | AdAffiliateController@influencers | web: /affiliate/dashboard, /campaigns/affiliate-influencers/:id |
| GET | `/api/v2/campaigns-affiliate/{id}/products` | AdAffiliateController@products | web: /affiliate/affiliate_campaign_products, /affiliate/product_list |
| POST | `/api/v3/campaigns/affiliate` | AdCreationProcessAffiliateController@store | web: /affiliate/affiliate_campaign_details, /affiliate/affiliate_campaign_products |

### CMP-05

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/product-seedings/handle` | ProductSeedingController@handle | web: /affiliate/product_seeding_wizard |
| GET | `/api/product-seedings/list` | ProductSeedingController@list | web: /affiliate/invitation, /campaigns, /gifting/invitation, /invitation |
| GET | `/api/product-seedings/{id}` | ProductSeedingController@get | web: /affiliate/invitation, /affiliate/product_seeding_wizard, /gifting/invitation, /invitation |
| DELETE | `/api/product-seedings/{id}` | ProductSeedingController@destroy | web: /campaigns |
| GET | `/api/product-seedings/{id}/influencers` | ProductSeedingInfluencerController@list | web: /campaigns/seeding-influencers/:id |
| POST | `/api/product-seedings/{id}/influencers` | ProductSeedingInfluencerController@sendRequestByBrand | web: /search |
| GET | `/api/product-seedings/{id}/influencers/export` | ProductSeedingInfluencerController@export | web: /campaigns/seeding-influencers/:id |
| PUT | `/api/product-seedings/{id}/influencers/{influencerId}/status` | ProductSeedingInfluencerController@changeStatusByBrand | web: /campaigns/seeding-influencers/:id |

### CMP-06

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/user-third-party/aftership/webhook` | AftershipWebhookController@trackingWebhook | webhook / callback |
| POST | `/api/user-third-party/shopify/v1/add-tracking-info` | ShopifyFulfillmentController@importTrackingInfo | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/user-third-party/shopify/v1/carriers` | TrackingServiceController@getAllCarriers | web: /new-offersV2, /new-offersV2/:id |
| POST | `/api/user-third-party/shopify/v1/edit-tracking-info` | TrackingServiceController@edit | web: /new-offersV2, /new-offersV2/:id |
| POST | `/api/user-third-party/shopify/v1/fulfill-to-influencer` | ShopifyFulfillmentController@sendToInfluencer | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/user-third-party/shopify/v1/has-fulfillment/{offer_id}` | ShopifyFulfillmentController@hasFulfillment | web: /new-offersV2, /new-offersV2/:id |
| POST | `/api/user-third-party/track123/webhook` | Track123WebhookController@trackingWebhook | webhook / callback |

### DSC-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/integrations/{integrationId}/reveal-username` | UsernameRevealController@reveal | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id |
| POST | `/api/integrations/{integrationId}/reveal-username-by-wallet` | UsernameRevealController@revealByWallet | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id |
| GET | `/api/v2/categories` | CategoryController@index | web: /account/:id, /accountnew/:id, /affiliate/affiliate-campaign-list, /affiliate/affiliate_campaign_details |
| GET | `/api/v2/categories/hashtags` | CategoryController@search | web: /campaign/discover, /campaign/post-content, /create-campaign, /find/influencer-explorer |
| POST | `/api/v2/elastic/extended/search` | ExtendedElasticController@search | web: /search |
| POST | `/api/v2/elastic/search` | ElasticController@search | web: /find/influencer-explorer, /find/influencer-explorer/custom-search |
| GET | `/api/v2/platforms` | PlatformController@index | web: /create-campaign, /onboarding, /profile |
| GET | `/api/v3/influencer/profile/all-data/{influencerIntegrationId}` | ProfileController@allData | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id, /influencer-content, /profile/contents |
| GET | `/api/v3/influencer/profile/elastic-data/{influencerIntegrationId}` | ProfileController@elasticData | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id |
| POST | `/api/v3/influencer/profile/feedbacks/like` | ProfileController@submitReviewLike | web: /account/:id, /accountnew/:id, /new-offersV2, /new-offersV2/:id |
| PATCH | `/api/v3/influencer/profile/hit-profile-views/{influencerIntegrationId}` | ProfileController@hitProfileViews | web: /account/:id, /accountnew/:id, /new-offersV2, /new-offersV2/:id |
| GET | `/api/v3/influencer/profile/preview-data/{influencerIntegrationId}` | ProfileController@previewData | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /analytics |
| GET | `/api/v3/influencer/profile/user/{influencerUserId}/all-data` | ProfileController@allDataByUser | web: /affiliate/dashboard, /campaigns/affiliate-influencers/:id, /campaigns/seeding-influencers/:id, /favorite-influencers |
| GET | `/api/v3/influencer/profile/{influencerIntegrationId}/all-accounts` | ProfileController@getAllProfiles | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id |

### DSC-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/v2/favorite-influencers` | BrandFavoriteInfluencerController@index | web: /favorite-influencers, /new-offersV2, /new-offersV2/:id, /new-offersV2/favorite |
| POST | `/api/v2/favorite-influencers/add` | BrandFavoriteInfluencerController@add | web: /account/:id, /accountnew/:id, /analytics, /favorite-influencers |
| POST | `/api/v2/favorite-influencers/remove` | BrandFavoriteInfluencerController@remove | web: /account/:id, /accountnew/:id, /analytics, /favorite-influencers |
| GET | `/api/v2/favorite-lists` | FavoriteListController@index | web: /account/:id, /accountnew/:id, /analytics, /favorite-influencers |
| POST | `/api/v2/favorite-lists` | FavoriteListController@store | web: /account/:id, /accountnew/:id, /analytics, /favorite-influencers |
| PUT | `/api/v2/favorite-lists/{id}` | FavoriteListController@update | web: /new-offersV2/favorite, /offers/favorite, /profile/favorites |
| DELETE | `/api/v2/favorite-lists/{id}` | FavoriteListController@destroy | web: /favorite-influencers, /new-offersV2/favorite, /offers/favorite, /profile/favorites |

### DSC-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/ads/invited/summary` | AdController@getInvitedCampaignsStatistics | mobile: HOME_SCREEN |
| GET | `/api/{locale}/v3/mobile/invitations` | InvitationController@list | mobile: CHAT_SCREEN, GIFTING_DETAIL, GiftingInvitationRouter, INVITED_CAMPAIGN_DETAILS |
| POST | `/api/{locale}/v3/mobile/invitations/hide` | InvitationController@hide | mobile: INVITED_CAMPAIGN_DETAILS |
| POST | `/api/{locale}/v3/mobile/invitations/see` | InvitationController@see | mobile: AFFILIATE_DETAIL, CAMPAIGN_DETAILS, INVITED_CAMPAIGN_DETAILS |
| GET | `/api/v2/invitations/find` | InvitationController@findByDetail | web: /affiliate/invitation, /gifting/invitation, /invitation |
| POST | `/api/v2/temp-invitations` | TemporaryInvitationController@store | web: /create-campaign, /search |
| POST | `/api/v2/user/invitations` | InvitationController@store | web: /account/:id, /accountnew/:id, /analytics, /favorite-influencers |
| POST | `/api/webhook/insta-dm/{id}` | WebhookController@instaDM | webhook / callback |

### INF-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/v3/mobile/influencer/integration-verifications/manual` | IntegrationVerificationStatusController@requestManualVerification | mobile: VERIFY_INSTAGRAM_SCREEN |
| GET | `/api/{locale}/v3/mobile/influencer/integration-verifications/status` | IntegrationVerificationStatusController@status | mobile: VERIFY_INSTAGRAM_SCREEN |
| POST | `/api/{locale}/v3/mobile/influencer/on-boarding/profile/lookup` | OnboardingController@store | mobile: GET_USER_NAME_SCREEN |
| POST | `/api/{locale}/v3/mobile/influencer/on-boarding/profile/verification` | OnboardingController@integrationAccountVerification | mobile: GET_USER_NAME_SCREEN, VERIFY_INSTAGRAM_SCREEN |
| GET | `/api/{locale}/v3/mobile/influencer/youtube/callback` | YouTubeOAuthController@callback | webhook / callback |
| GET | `/api/{locale}/v3/mobile/influencer/youtube/consent-screen` | YouTubeOAuthController@consentScreen | mobile: GET_USER_NAME_SCREEN |
| POST | `/api/v3/webhooks/instagram/dm-verification` | DmVerificationWebhookController | webhook / callback |
| POST | `/api/v3/webhooks/tiktok/dm-verification` | DmVerificationWebhookController | webhook / callback |

### INF-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/locations/countries` | LocationController@countries | mobile: SHIPPING_ADDRESS_SCREEN |
| GET | `/api/{locale}/v3/mobile/categories` | CategoryController@index | mobile: EDIT_CATEGORY, NICHE, TIKTOK_VERIFIED |
| POST | `/api/{locale}/v3/mobile/funnels` | FunnelsController@storeLog | mobile: GOALS, MEDIA_KIT, NICHE, PHOTO_GALLERY |
| GET | `/api/{locale}/v3/mobile/funnels/processes` | FunnelsController@processList | mobile: PROFILE_SCREEN |
| GET | `/api/{locale}/v3/mobile/goal/questions` | GoalQuestionController@index | mobile: GOALS |
| POST | `/api/{locale}/v3/mobile/influencer/on-boarding` | OnboardingController@createInfluencerOnboarding | mobile: EDIT_CATEGORY, GOALS, MEDIA_KIT, NICHE |
| POST | `/api/{locale}/v3/mobile/influencer/on-boarding/finished` | OnboardingController@setOnboarded | mobile: GOALS |
| GET | `/api/{locale}/v3/mobile/platforms/{platform}/accounts/data` | PlatformController@getAccount | mobile: MEDIA_KIT |
| GET | `/api/{locale}/v3/mobile/social/hashtags/search` | CategoryController@search | mobile: NICHE |
| GET | `/api/{locale}/v3/mobile/social/medias/feed` | FeedController@showFeed | mobile: PHOTO_GALLERY |

### INF-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/v3/mobile/temporary-integrations/{id}/resubmit` | TemporaryIntegrationController@resubmit | mobile: GET_USER_NAME_SCREEN, VERIFY_INSTAGRAM_SCREEN |

### INF-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/v2/influencer/suspension-appeal` | SuspensionAppealController@storeInfluencerAppeal | mobile: EXPLORE_SCREEN, PROFILE_SCREEN |
| GET | `/api/v2/settings/influencer/suspension/appeal` | SettingController@getInfluencerSuspensionAppealNotes | mobile: EXPLORE_SCREEN, PROFILE_SCREEN |
| GET | `/api/v2/settings/influencer/suspension/reasons` | SettingController@getInfluencerSuspensionReasons | mobile: EXPLORE_SCREEN, PROFILE_SCREEN |

### INF-05

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/addresses` | AddressController@index | mobile: CHAT_SCREEN, GIFTING_DETAIL, HOME_SCREEN, PROFILE_SCREEN |
| POST | `/api/{locale}/v3/mobile/addresses` | AddressController@store | mobile: CHAT_SCREEN, GIFTING_DETAIL, SETTINGS_SHIPPING_ADDRESS_FORM |
| PUT | `/api/{locale}/v3/mobile/addresses/{address}` | AddressController@update | mobile: CHAT_SCREEN, GIFTING_DETAIL, SETTINGS_SHIPPING_ADDRESS, SETTINGS_SHIPPING_ADDRESS_FORM |
| DELETE | `/api/{locale}/v3/mobile/addresses/{address}` | AddressController@destroy | mobile: CHAT_SCREEN, GIFTING_DETAIL, SETTINGS_SHIPPING_ADDRESS |
| GET | `/api/{locale}/v3/mobile/influencer/accounts` | AccountController@getAccounts | mobile: AFFILIATE_DETAIL, CHAT_SCREEN, CONGRATULATIONS, EXPLORE_SCREEN |
| GET | `/api/{locale}/v3/mobile/influencer/accounts/{integrationId}` | AccountController@getAccount | mobile: CAMPAIGN_DETAILS, EXPLORE_SCREEN, GET_USER_NAME_SCREEN, MAKE_OFFER_SCREEN |
| PUT | `/api/{locale}/v3/mobile/influencer/accounts/{integrationId}` | AccountController@updateAccount | mobile: EDIT_NAME, MEDIA_KIT, PROFILE_SCREEN |
| GET | `/api/{locale}/v3/mobile/influencer/accounts/{integrationId}/insights` | InsightController@show | mobile: CHAT_SCREEN, YOUR_AUDIENCE |
| DELETE | `/api/{locale}/v3/mobile/influencer/accounts/{integrationId}/picture` | AccountController@deleteAccountPicture | mobile: PROFILE_SCREEN |
| GET | `/api/{locale}/v3/mobile/influencer/accounts/{integrationId}/pricing` | AccountController@getAccountPricing | mobile: EDIT_OFFER_SCREEN, MAKE_OFFER_SCREEN |
| GET | `/api/{locale}/v3/mobile/influencer/accounts/{integrationId}/settings` | SettingController@getSettingByIg | mobile: MAKE_OFFER_SCREEN |
| POST | `/api/{locale}/v3/mobile/influencer/accounts/{integrationId}/sync` | AccountController@syncAccount | mobile: PROFILE_SCREEN |
| DELETE | `/api/v2/integrations/{id}` | IntegrationController@destroy | web: /notifications |

### INF-06

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/v2/webhooks/scraper/notify` | ScraperController@notify | webhook / callback |
| POST | `/api/v2/webhooks/scraper/tiktok/reports/video` | ScraperController@videoReport | webhook / callback |

### INT-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/user-third-party/shopify/v1/products` | ShopifyController@getProducts | web: /create-campaign, /new-offersV2, /new-offersV2/:id |
| POST | `/api/user-third-party/shopify/webhook/app-uninstall` | ShopifyWebhookController@appUninstalled | webhook / callback |
| POST | `/api/user-third-party/shopify/webhook/erase-customer` | ShopifyWebhookController@eraseCustomer | webhook / callback |
| POST | `/api/user-third-party/shopify/webhook/get-customer` | ShopifyWebhookController@getCustomer | webhook / callback |

### INT-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/user-third-party/brandid/v1/auth-by-ain` | BrandIDIntegrationController@authByAin | webhook / callback |
| GET | `/api/user-third-party/brandid/v1/link` | BrandIDIntegrationController@generateLink | mobile: CHAT_SCREEN |
| POST | `/api/user-third-party/brandid/v1/user-created` | BrandIDIntegrationController@registeredInBrandidWebhook | webhook / callback |
| POST | `/api/user-third-party/brandid/v1/username` | BrandIDIntegrationController@storeBrandidUsernameWebhook | webhook / callback |

### MKT-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/ads/all` | AdController@getAllAds | mobile: AllAds, SEARCH_ADS_SCREEN |
| GET | `/api/{locale}/v3/mobile/ads/all-add-once` | AdController@allAdsOnce | mobile: AllAds |
| GET | `/api/{locale}/v3/mobile/ads/for-you` | AdController@getForYou | mobile: AllAds, Invited, affiliate, category |
| GET | `/api/{locale}/v3/mobile/ads/from-brand` | AdController@getAdsByBrand | mobile: BRAND_PROFILE |
| GET | `/api/{locale}/v3/mobile/ads/most-paid` | AdController@mostPaid | mobile: AllAds, Invited, affiliate, category |
| GET | `/api/{locale}/v3/mobile/ads/my-categories` | AdController@getAdsByCategory | mobile: AllAds, Invited, TIKTOK_VERIFIED, affiliate |
| GET | `/api/{locale}/v3/mobile/ads/my-country` | AdController@getAdsByCountry | mobile: AllAds, Invited, affiliate, category |
| GET | `/api/{locale}/v3/mobile/ads/{id}` | AdController@show | mobile: CAMPAIGN_DETAILS, InvitationRouter |
| GET | `/api/{locale}/v3/mobile/ads/{id}/published-contents` | AdController@getAllPublishedContents | mobile: InfluencersContent |
| GET | `/api/{locale}/v3/mobile/brands/{id}/profile` | BrandProfileController@showProfile | mobile: BRAND_PROFILE |
| GET | `/api/{locale}/v3/mobile/offers/with-brand` | OfferController@listByBrand | mobile: BRAND_PROFILE |
| POST | `/api/{locale}/v3/mobile/privacy/brand/block` | BlockController@store | mobile: CAMPAIGN_DETAILS |
| POST | `/api/{locale}/v3/mobile/privacy/reports` | ReportController@store | mobile: CAMPAIGN_DETAILS, CHAT_SCREEN |
| GET | `/api/{locale}/v3/mobile/privacy/reports/items` | ReportController@getItems | mobile: CAMPAIGN_DETAILS, CHAT_SCREEN |
| POST | `/api/v2/reports` | ReportController@store | web: /new-offersV2, /new-offersV2/:id, /new-offersV2/deals, /offers |
| GET | `/api/v2/reports/items` | ReportController@getItems | web: /new-offersV2, /new-offersV2/:id, /new-offersV2/deals, /offers |
| POST | `/api/v3/mobile/ads/{ad}/view` | AdController@view | mobile: CAMPAIGN_DETAILS |

### MKT-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/ads/affiliate` | AdAffiliateController@index | mobile: AllAds, Invited, affiliate, category |
| GET | `/api/{locale}/v3/mobile/ads/affiliate/from-brand` | AdAffiliateController@getByBrand | mobile: BRAND_PROFILE |
| GET | `/api/{locale}/v3/mobile/ads/affiliate/{id}` | AdAffiliateController@show | mobile: AffiliateInvitationRouter, CHAT_SCREEN |
| POST | `/api/{locale}/v3/mobile/ads/affiliate/{id}/assign-influencer` | AdAffiliateController@assignToAd | mobile: AFFILIATE_DETAIL |
| GET | `/api/{locale}/v3/mobile/ads/affiliate/{id}/products` | AdAffiliateController@productsList | mobile: AFFILIATE_DETAIL |
| GET | `/api/{locale}/v3/mobile/ainfluencer-affiliate/payouts` | AffiliateReportController@payouts | mobile: AFFILIATE_ANALYTICS, HOME_SCREEN, affiliate |
| GET | `/api/{locale}/v3/mobile/ainfluencer-affiliate/reports/products` | AffiliateReportController@products | mobile: AFFILIATE_ANALYTICS, HOME_SCREEN, affiliate |
| GET | `/api/{locale}/v3/mobile/ainfluencer-affiliate/reports/summary` | AffiliateReportController@summary | mobile: AFFILIATE_ANALYTICS, HOME_SCREEN, affiliate |
| GET | `/api/{locale}/v3/mobile/ainfluencer-affiliate/reports/timeline` | AffiliateReportController@timeline | mobile: AFFILIATE_ANALYTICS, HOME_SCREEN, affiliate |

### MKT-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/product-seedings` | ProductSeedingController@index | mobile: AllAds, BRAND_PROFILE, CHAT_SCREEN, GIFTING_CAMPAIGNS |
| GET | `/api/{locale}/v3/mobile/product-seedings/requests` | ProductSeedingController@receivedRequests | mobile: CHAT_SCREEN, GIFTING_CAMPAIGNS, GIFTING_DETAIL, GiftingInvitationRouter |
| PUT | `/api/{locale}/v3/mobile/product-seedings/requests/{requestId}` | ProductSeedingController@updateRequest | mobile: GIFTING_DETAIL |
| POST | `/api/{locale}/v3/mobile/product-seedings/requests/{requestId}/posts` | ProductSeedingController@sharePosts | mobile: CHAT_SCREEN, GIFTING_DETAIL |
| DELETE | `/api/{locale}/v3/mobile/product-seedings/requests/{requestId}/posts/{postId}` | ProductSeedingController@removePost | mobile: CHAT_SCREEN, GIFTING_DETAIL |
| PUT | `/api/{locale}/v3/mobile/product-seedings/requests/{requestId}/respond` | ProductSeedingController@respondToRequest | mobile: CHAT_SCREEN, GIFTING_DETAIL |
| POST | `/api/{locale}/v3/mobile/product-seedings/requests/{requestId}/send-address` | ProductSeedingController@sendAddress | mobile: CHAT_SCREEN, GIFTING_DETAIL, SETTINGS_SHIPPING_ADDRESS, SETTINGS_SHIPPING_ADDRESS_FORM |
| PUT | `/api/{locale}/v3/mobile/product-seedings/requests/{requestId}/status` | ProductSeedingController@changeStatus | mobile: CHAT_SCREEN, GIFTING_DETAIL |
| GET | `/api/{locale}/v3/mobile/product-seedings/{id}` | ProductSeedingController@show | mobile: CHAT_SCREEN, GIFTING_DETAIL |
| POST | `/api/{locale}/v3/mobile/product-seedings/{id}/apply` | ProductSeedingController@apply | mobile: CHAT_SCREEN, GIFTING_DETAIL |

### NTF-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/notifications` | NotificationController@index | mobile: EXPLORE_SCREEN, HOME_SCREEN, INBOX_SCREEN, INVITED_CAMPAIGN_DETAILS |
| POST | `/api/{locale}/v3/mobile/notifications/read` | NotificationController@read | mobile: EXPLORE_SCREEN, HOME_SCREEN, INBOX_SCREEN, INVITED_CAMPAIGN_DETAILS |
| GET | `/api/v2/notifications` | NotificationController@index | web: /notifications |
| DELETE | `/api/v2/notifications/delete/all` | NotificationController@deleteAll | web: (global) |
| POST | `/api/v2/notifications/read` | NotificationController@read | web: /notifications |
| PATCH | `/api/v2/notifications/read-all` | NotificationController@readAll | web: (global) |
| DELETE | `/api/v2/notifications/{notification}` | NotificationController@destroy | web: /notifications |

### NTF-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/v2/notify-settings/categories` | NotificationCategoryController@listCategories | web: /profile/email-notifications; mobile: NOTIFICATIONS_EMAILS |
| PUT | `/api/v2/notify-settings/categories` | NotificationCategoryController@bulkUpdateSettings | web: /profile/email-notifications; mobile: NOTIFICATIONS_EMAILS |

### OFR-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/offers` | OfferController@index | mobile: CHAT_SCREEN, HOME_SCREEN |
| POST | `/api/{locale}/v3/mobile/offers` | OfferController@store | mobile: MAKE_OFFER_SCREEN |
| POST | `/api/{locale}/v3/mobile/offers/temporary` | OfferController@storeTemporaryOffer | mobile: MAKE_OFFER_SCREEN |
| GET | `/api/{locale}/v3/mobile/settings/collaboration-types` | SettingController@getInfluencerCollaborationTypes | mobile: MAKE_OFFER_SCREEN |
| GET | `/api/{locale}/v3/mobile/settings/offers/pricing` | SettingController@getMakeOfferPricing | mobile: EDIT_OFFER_SCREEN |
| GET | `/api/v2/settings/offers/pricing` | SettingController@getMakeOfferPricing | web: /new-offersV2, /new-offersV2/:id |

### OFR-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/campaigns/statistics` | AdController@statistics | mobile: HOME_SCREEN, WELCOME_SCREEN |
| GET | `/api/{locale}/v3/mobile/offers/{id}` | OfferController@show | mobile: CHAT_SCREEN, EDIT_OFFER_SCREEN |
| PUT | `/api/{locale}/v3/mobile/offers/{id}` | OfferController@update | mobile: EDIT_OFFER_SCREEN |
| PUT | `/api/{locale}/v3/mobile/offers/{id}/history` | OfferHistoryController@acceptOrRejectHistory | mobile: CHAT_SCREEN |
| GET | `/api/v2/awaiting-payment` | OfferController@awaitingPaymentList | web: /new-offersV2/:id/payment-successful, /offers/:id/payment-successful |
| GET | `/api/v2/offers` | OfferController@index | web: /new-offersV2/deals, /offers, /offers/deals |
| GET | `/api/v2/offers/{offer}` | OfferController@show | web: /analytics, /influencer-content, /new-offersV2, /new-offersV2/:id |
| POST | `/api/v2/offers/{offer}/change-status` | OfferController@changeStatus | web: /new-offersV2, /new-offersV2/:id |
| PUT | `/api/v2/offers/{offer}/counter` | OfferController@counter | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/v2/offers/{offer}/history` | OfferController@history | web: /new-offersV2, /new-offersV2/:id |

### OFR-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/finance/current-user/basket` | BasketController@index | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id |
| POST | `/api/{locale}/finance/current-user/basket` | BasketController@updateItem | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id |
| POST | `/api/{locale}/finance/current-user/basket/empty` | BasketController@empty | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id |
| POST | `/api/{locale}/finance/current-user/basket/insurance` | BasketController@setInsurance | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /new-offersV2/:id/pay |
| POST | `/api/{locale}/finance/current-user/coupon` | CouponController@store | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /new-offersV2/:id/pay |
| DELETE | `/api/{locale}/finance/current-user/coupon/{coupon}` | CouponController@destroy | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /new-offersV2/:id/pay |
| POST | `/api/{locale}/finance/payment` | PaymentController@handle | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /new-offersV2/:id/pay |
| GET | `/api/v3/offers/{offer}/abandoned-tracking` | PaymentTrackingController@abandonedTracking | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /new-offersV2/:id/pay |

### OFR-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| PUT | `/api/{locale}/v3/mobile/deals/{id}/deadline/extend` | PublishJobController@extendDeadline | mobile: EXTEND_DEADLINE |
| PUT | `/api/v2/offers/{offer}/update-start` | OfferController@updateStartBeforePayment | web: /new-offersV2, /new-offersV2/:id |

### OFR-05

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/finance/payment/entity` | PaymentController@entity | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /new-offersV2/:id/pay |
| POST | `/api/{locale}/v2/offer-revision` | RevisionOfferController@store | web: /new-offersV2, /new-offersV2/:id |
| POST | `/api/v3/mobile/offer/{offerId}/revise/accept` | RevisionOfferController@accept | mobile: CHAT_SCREEN |
| POST | `/api/v3/mobile/offer/{offerId}/revise/reject` | RevisionOfferController@reject | mobile: CHAT_SCREEN |

### PAY-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/finance/current-user/cards` | CardController@index | web: /cards, /funds, /influencer-content, /new-offersV2 |
| POST | `/api/{locale}/finance/current-user/cards` | CardController@store | web: /cards/create, /funds, /influencer-content, /new-offersV2 |
| DELETE | `/api/{locale}/finance/current-user/cards/{card}` | CardController@destroy | web: /cards, /funds, /influencer-content, /new-offersV2 |
| POST | `/api/{locale}/finance/current-user/cards/{walletSettingId}/make-default` | CardController@makeDefault | web: /cards, /funds, /influencer-content, /new-offersV2 |
| GET | `/api/{locale}/finance/current-user/orders` | OrderController@index | web: /profile/transactions, /transactions2 |
| GET | `/api/{locale}/finance/current-user/wallets` | WalletController@index | web: /account/:id, /accountnew/:id, /affiliate/dashboard, /campaigns/affiliate-influencers/:id, /profile/transactions, /transactions2; mobile: HOME_SCREEN, PROFILE_SCREEN, WALLET_SCREEN |
| POST | `/api/{locale}/finance/current-user/wallets` | WalletController@store | web: /funds |
| GET | `/api/{locale}/finance/current-user/wallets/{wallet}` | WalletController@show | web: /profile/transactions, /transactions2 |
| GET | `/api/{locale}/finance/payment/callback/failed/{payment_receipt_tracking}` | PaymentCallbackController@failed | webhook / callback |
| GET | `/api/{locale}/finance/payment/callback/success/{payment_receipt_tracking}` | PaymentCallbackController@success | webhook / callback |
| POST | `/api/{locale}/finance/stripe/webhook` | StripeWebhookController@handleWebhook | webhook / callback |
| POST | `/api/v2/paypal/charge` | ChargeController@charge | web: /funds |
| GET | `/api/v2/user/wallet` | WalletController@info | web: /profile/transactions, /transactions2 |
| POST | `/api/webhooks/stripe` | StripeController | webhook / callback |
| POST | `/v3/webhooks/stripe` | Closure | webhook / callback |

### PAY-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/v3/mobile/wallet` | WalletController@getWalletInformation | mobile: WALLET_SCREEN_OLD |
| GET | `/api/{locale}/v3/mobile/wallet/transactions/payments` | TransactionController@payments | mobile: WALLET_SCREEN_OLD |
| GET | `/api/{locale}/v3/mobile/wallet/transactions/payouts` | TransactionController@payouts | mobile: WALLET_SCREEN_OLD |
| GET | `/api/{locale}/finance/mobile/influencer-orders` | OrderController@influencerOrders | mobile: WALLET_SCREEN |

### PAY-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/{locale}/finance/mobile/orders/payable` | PayoutController@payableOrderList | mobile: HOME_SCREEN, PROFILE_SCREEN, WALLET_SCREEN |
| GET | `/api/{locale}/finance/mobile/payout-methods` | PayoutMethodController@index | mobile: PROFILE_SCREEN, WALLET_SCREEN |
| POST | `/api/{locale}/finance/mobile/payout-methods` | PayoutMethodController@store | mobile: WALLET_SCREEN |
| GET | `/api/{locale}/finance/mobile/payout-methods/crypto-options` | PayoutMethodController@cryptoOptions | mobile: WALLET_SCREEN |
| GET | `/api/{locale}/finance/mobile/payouts` | PayoutController@index | mobile: WALLET_SCREEN |
| POST | `/api/{locale}/finance/mobile/payouts` | PayoutController@store | mobile: PROFILE_SCREEN, WALLET_SCREEN |
| GET | `/api/{locale}/finance/stripe/callback` | StripeController@callback | webhook / callback |
| GET | `/api/{locale}/finance/stripe/refresh` | StripeController@refresh | webhook / callback |
| GET | `/api/payment/stripe/connect/callback` | PaymentController@stripe_connect_callback | webhook / callback |

### PUB-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/v3/mobile/deals/{id}/published-contents/files` | PublishedContentController@storeFiles | mobile: CHAT_SCREEN |
| DELETE | `/api/{locale}/v3/mobile/deals/{id}/published-contents/files` | PublishedContentController@deleteFiles | mobile: CHAT_SCREEN |
| POST | `/api/{locale}/v3/mobile/deals/{publishJob}/url` | PublishJobController@submitPublishUrl | mobile: CHAT_SCREEN |
| GET | `/api/{locale}/v3/mobile/deals/{type}` | PublishJobController@index | mobile: HOME_SCREEN, SCHEDULER_SCREEN |
| GET | `/api/v2/publish-jobs` | PublishJobController@index | web: /, /scheduler |
| GET | `/api/v2/publish-jobs/{publish_job}` | PublishJobController@show | web: /publish-jobs/:id/analytics |
| GET | `/api/v2/schedules` | PublishJobController@schedules | web: /, /scheduler |
| POST | `/api/webhook` | WebhookController | webhook / callback |
| POST | `/api/webhook/instagram/publish-job` | WebhookController@forward_publishjob | webhook / callback |
| POST | `/api/webhook/publish-job/{id}` | WebhookController@publishJob | webhook / callback |

### PUB-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/{locale}/offer/v2/{offerId}/approve` | OfferController@approve | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /profile/contents |
| POST | `/api/{locale}/offer/v2/{offerId}/disapprove` | OfferController@disapprove | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/v2/guest/published-contents` | PublishedContentController@index | web: /account/:id, /public/profile/:id |
| GET | `/api/v2/influencer-generated-contents` | MediaController@influencerGeneratedContents | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/v2/published-contents` | PublishedContentController@index | web: /, /affiliate/dashboard, /campaigns/affiliate-influencers/:id, /campaigns/seeding-influencers/:id |
| GET | `/api/v2/published-contents/download/{contentId}` | PublishedContentController@downloadAll | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /profile/contents |
| GET | `/api/v2/published-contents/download/{contentId}/files/{fileId}` | PublishedContentController@downloadSingle | web: /, /affiliate/invitation, /gifting/invitation, /invitation, /view-campaign-affiliate/:id |
| GET | `/api/v2/published-contents/influencers/{integrationId}` | PublishedContentController@showInfluencer | web: /account/:id, /accountnew/:id, /new-offersV2, /new-offersV2/:id |
| GET | `/api/v2/published-contents/offers/{encodedOfferId}` | PublishedContentController@showByOffer | web: /analytics, /new-offersV2, /new-offersV2/:id, /view-contents/:code |

### PUB-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| DELETE | `/api/{locale}/v3/mobile/insights/{insightId}/files/{fileId}` | InsightController@delete | mobile: CHAT_SCREEN |
| POST | `/api/{locale}/v3/mobile/offers/{id}/insights` | InsightController@store | mobile: CHAT_SCREEN |
| GET | `/api/v2/campaigns/published/analysis` | PublishedCampaignAnalysisController@index | web: /analytics |
| GET | `/api/v2/campaigns/published/analysis/{adId}/influencers/recommended` | PublishedCampaignAnalysisController@recommendedInfluencers | web: /analytics, /new-offersV2/favorite, /offers/favorite, /profile/favorites |
| GET | `/api/v2/campaigns/published/analysis/{adId}/publishers` | PublishedCampaignAnalysisController@show | web: /analytics |
| GET | `/api/v2/campaigns/published/analysis/{adId}/publishers/{integrationId}` | PublishedCampaignAnalysisController@analytics | web: /analytics |
| POST | `/api/v2/insight/request` | InsightController@store | web: /new-offersV2, /new-offersV2/:id |
| GET | `/api/v2/insight/{offer_id}` | InsightController@show | web: /new-offersV2, /new-offersV2/:id |

### PUB-05

| Method | Path | Controller | Caller |
|---|---|---|---|
| PUT | `/api/v2/guest/reviews` | ReviewController@update | web: /review/:code |
| GET | `/api/v2/guest/reviews/{id}` | ReviewController@show | web: /review/:code |
| GET | `/api/v2/guest/reviews/{id}/result` | ReviewController@result | web: /review/:code/result |
| POST | `/api/v2/reviews` | ReviewController@store | web: /, /analytics, /new-offersV2, /new-offersV2/:id |
| GET | `/api/v2/reviews` | ReviewController@index | web: /new-offersV2/reviews, /offers/reviews, /profile/points, /profile/reviews |

### PUB-06

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/v2/translation-requests` | TranslationRequestController@index | web: /profile/translated-contents |
| GET | `/api/v2/translation-requests/{id}/download/all` | TranslationRequestController@downloadAll | web: /new-offersV2, /new-offersV2/:id, /profile/translated-contents |
| GET | `/api/v2/translation-requests/{id}/translated/{fileId}/download` | TranslationRequestController@downloadTranslatedFile | web: /new-offersV2, /new-offersV2/:id, /profile/translated-contents |
| GET | `/api/v2/translations` | TranslationController@showTranslateable | web: /new-offersV2, /new-offersV2/:id, /profile/translated-contents |
| POST | `/api/v2/translations` | TranslationRequestController@store | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /profile/contents |
| GET | `/api/v2/translations/languages` | TranslationRequestController@supportedLanguages | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /profile/contents |
| DELETE | `/api/v2/translations/{id}` | TranslationRequestController@destroy | web: /influencer-content, /new-offersV2, /new-offersV2/:id, /profile/contents |
| POST | `/api/v2/webhooks/translations/callback` | TranslationCallbackController | webhook / callback |

### PUBL-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/v2/guest/campaigns/contents/download/{contentId}` | CampaignContentController@downloadAll | web: /affiliate/invitation, /gifting/invitation, /invitation, /view-campaign-affiliate/:id |
| GET | `/api/v2/guest/campaigns/{id}` | CampaignController@show | web: /affiliate/invitation, /gifting/invitation, /invitation, /view-campaign-affiliate/:id, /view-campaign-seeding/:id, /view-campaign/:id |

### PUBL-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/v3/influencer/profile/public/all-data/{influencerIntegrationId}` | PublicProfileController@allData | web: /account/:id, /accountnew/:id, /public/profile/:id |
| GET | `/api/v3/influencer/profile/public/elastic-data/{influencerIntegrationId}` | PublicProfileController@elasticData | web: /account/:id, /accountnew/:id, /public/profile/:id |
| PATCH | `/api/v3/influencer/profile/public/hit-profile-views/{influencerIntegrationId}` | PublicProfileController@hitProfileViews | web: /account/:id, /accountnew/:id, /public/profile/:id |
| GET | `/api/v3/influencer/profile/public/preview-data/{influencerIntegrationId}` | PublicProfileController@previewData | web: /account/:id, /analytics, /find/influencer-explorer, /find/influencer-explorer/custom-search |
| GET | `/api/v3/influencer/profile/public/published-contents/{integrationId}` | PublicPublishedContentController@listPublishedContents | web: /account/:id, /accountnew/:id, /public/profile/:id |
| GET | `/api/v3/influencer/profile/public/{influencerIntegrationId}/all-accounts` | PublicProfileController@getAllProfiles | web: /account/:id, /accountnew/:id, /public/profile/:id |

### SUB-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/api/subscriptions/getCurrentPlanInfo` | SubscriptionController@getCurrentPlanInfo | web: /, /account/:id, /accountnew/:id, /affiliate/affiliate-campaign-list |
| GET | `/api/subscriptions/plans` | PlanController@getDefaultPlans | web: /, /account/:id, /accountnew/:id, /affiliate/affiliate-campaign-list |
| GET | `/api/subscriptions/receipts` | SubscriptionController@receipts | web: /profile/transactions, /transactions2 |
| GET | `/api/subscriptions/subscribe/{planPriceId}` | SubscriptionController@subscribe | web: /profile/transactions, /subscription-plans, /transactions2 |
| PUT | `/api/subscriptions/update/{planPriceId}` | SubscriptionController@update | web: /profile/transactions, /subscription-plans, /transactions2 |
| POST | `/api/subscriptions/webhook` | CashierWebhookController@handleWebhook | webhook / callback |
| POST | `/stripe/webhook` | WebhookController@handleWebhook | webhook / callback |

### SUB-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/api/subscriptions/cancel` | SubscriptionController@cancel | web: /profile/transactions, /transactions2 |
| GET | `/api/subscriptions/cancellation-summary` | SubscriptionController@cancellationSummary | web: /profile/transactions, /transactions2 |
| POST | `/api/subscriptions/resume` | SubscriptionController@resume | web: /profile/transactions, /transactions2 |
| GET | `/api/v2/forms/positions` | FormController@formPositions | web: /new-offersV2, /new-offersV2/:id, /profile/transactions, /transactions2 |
| GET | `/api/v2/forms/{formId}` | FormController@show | web: /new-offersV2, /new-offersV2/:id, /profile/transactions, /transactions2 |
| POST | `/api/v2/forms/{formId}` | FormController@submit | web: /new-offersV2, /new-offersV2/:id, /profile/transactions, /transactions2 |

## 2. Client calls with no working backend route

These calls are in live client code, but no route in core master accepts them, so the action fails for users.

| App | Method | Path | Problem |
|---|---|---|---|
| web | DELETE | `v2/favorite-influencers/delete-all/{param}` | no route |
| web | POST | `v2/favorite-influencers/toggle` | no route |
| mobile | PUT | `v3/mobile/product-seedings/requests/{requestId}/posts/{postId}` | method not allowed (route has DELETE) |
| mobile | POST | `v3/mobile/product-seedings/{id}/send-address` | no route |
| mobile | HEAD | `v3/mobile/health` | no route |
