# Affiliate backend endpoint inventory (affiliate/ainflu-affiliate-backend)

Generated 2026-10-05 from a route dump of the affiliate backend at production tag `v1.5.27` (543264a76, 2026-10-04): 895 routes.

Client calls come from static analysis of the reachable code of the seller app and affiliate app (`ainflu-affiliate-frontend` v1.4.02), the buyer site (`ainflu-affiliate-buyer` v1.3.04), and the server-to-server calls made by core `core/backend/api`. See [aff-seller.md](aff-seller.md), [aff-affiliate.md](aff-affiliate.md), [aff-buyer.md](aff-buyer.md) and [aff-backend.md](aff-backend.md).

| Class | Meaning | Routes |
|---|---|---|
| client | Called by live code in the seller, affiliate or buyer app, or by core | 479 |
| webhook | Called by a third party (Stripe, Shopify, OAuth providers) or by email links | 24 |
| admin | Affiliate admin panel only. Out of scope for this map | 180 |
| other | Infrastructure (Horizon, API docs, Sanctum, debug) | 35 |
| unused | Nothing calls it. Excluded from the map (see [excluded-dead-code.md](excluded-dead-code.md)) | 177 |

## 1. Live endpoints by topic

Topic IDs match [../flow-map.md](../flow-map.md). The Caller column lists up to 4 pages per app, or the core class that makes the call.

### AFA-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/auth/auth-status` | AuthStatusController | affiliate: (router guard on /app/* and /user/*) |
| POST | `/affiliate/v1/auth/forget-password` | ForgetPasswordController | affiliate: /user/forgot-password |
| POST | `/affiliate/v1/auth/login` | LoginController | affiliate: /user/forgot-password, /user/login, /user/register, /user/reset-password/:token |
| POST | `/affiliate/v1/auth/login/amazon` | AmazonLoginController | affiliate: /user/login, /user/register |
| POST | `/affiliate/v1/auth/login/facebook` | FacebookLoginController | affiliate: /user/login, /user/register |
| POST | `/affiliate/v1/auth/login/google` | GoogleLoginController | affiliate: /user/login, /user/register |
| POST | `/affiliate/v1/auth/mobile-verification-code` | SendMobileVerificationCodeController | affiliate: /user/register-steps |
| POST | `/affiliate/v1/auth/mobile-verify` | MobileVerifyController | affiliate: /user/register-steps |
| POST | `/affiliate/v1/auth/register` | RegisterController | affiliate: /user/login, /user/register, /user/register-steps |
| POST | `/affiliate/v1/auth/reset-password` | ResetPasswordController | affiliate: /user/reset-password/:token |
| POST | `/affiliate/v1/auth/skip-step` | SkipStepController | affiliate: /user/register-steps |
| POST | `/affiliate/v1/profile` | UpdateProfileController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v1/profile` | ShowProfileController | affiliate: /app/profile |
| PUT | `/affiliate/v1/profile/change-mobile` | ChangeMobileNumberController | affiliate: /app/profile |
| GET | `/affiliate/v1/profile/overview` | ProfileOverviewController | affiliate: /app/profile |
| GET | `/affiliate/v1/profile/platform-accounts/dropdown` | IndexPlatformAccountController | affiliate: (app layout: all /app/*) |
| GET | `/affiliate/v1/profile/profile-completion-percentage` | ShowProfileCompletionPercentageController | affiliate: /app |
| GET | `/affiliate/v1/profile/questions` | ProfileQuestionListController | affiliate: /app/profile |
| POST | `/affiliate/v1/profile/questions` | StoreProfileQuestionController | affiliate: /app/profile |
| POST | `/affiliate/v1/profile/request-change-mobile` | RequestChangeMobileNumberController | affiliate: /app/profile |
| GET | `/affiliate/v1/profile/show-register-steps` | ShowRegisterStepsController | affiliate: /user/register-steps |
| POST | `/affiliate/v1/profile/sync/account/facebook` | SyncFacebookAccountController | affiliate: /app/profile, /user/register-steps |
| POST | `/affiliate/v1/profile/sync/account/google` | SyncGoogleAccountController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v1/profile/synced-accounts` | SyncedAccountListController | affiliate: /app/profile, /user/register-steps |
| DELETE | `/affiliate/v1/profile/synced-accounts/{id}` | DeleteSyncedAccountController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v1/questions` | RegisterQuestionListController | affiliate: /user/register-steps |
| POST | `/affiliate/v1/questions` | StoreRegisterQuestionController | affiliate: /user/register-steps |
| GET | `/web/v1/affiliate/login/amazon/callback` | AmazonLoginCallbackController | webhook / callback |
| GET | `/web/v1/affiliate/login/facebook/callback` | FacebookLoginCallbackController | webhook / callback |
| GET | `/web/v1/affiliate/login/google/callback` | GoogleLoginCallbackController | webhook / callback |

### AFA-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/categories` | IndexCategoryController | affiliate: /app/active-management, /app/associates/explore, /app/brand/:id, /app/brands-discovery |
| POST | `/affiliate/v1/social-medias` | CreateSocialMediaController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v1/social-medias` | IndexSocialMediaController | affiliate: /app/profile |
| POST | `/affiliate/v1/social-medias/check-website` | CheckWebsiteSocialMediaController | affiliate: /app/profile |
| POST | `/affiliate/v1/social-medias/delete` | DeleteSocialMediaController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v1/social-medias/types` | IndexSocialMediaTypeController | affiliate: /app/profile, /user/register-steps |
| PUT | `/affiliate/v1/social-medias/verify` | VerifySocialMediaController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v1/social-medias/{socialMediaId}` | ShowSocialMediaController | affiliate: /app/profile, /user/register-steps |
| PUT | `/affiliate/v1/social-medias/{socialMediaId}` | UpdateSocialMediaController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v2/social-medias` | IndexSocialMediaController | affiliate: /app/profile, /user/register-steps |
| PUT | `/affiliate/v2/social-medias/category` | UpdateSocialMediaCategoryController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v2/social-medias/oauth/facebook` | FacebookLoginController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v2/social-medias/oauth/instagram` | InstagramLoginController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v2/social-medias/oauth/pinterest` | PinterestLoginController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v2/social-medias/oauth/tiktok` | TikTokLoginController | affiliate: /app/profile, /user/register-steps |
| GET | `/affiliate/v2/social-medias/oauth/youtube` | YoutubeLoginController | affiliate: /app/profile, /user/register-steps |
| POST | `/affiliate/v2/social-medias/websites/social-verification-token` | GetWebsiteSocialMediaVerificationTokenController | affiliate: /app/profile, /user/register-steps |
| POST | `/affiliate/v2/social-medias/websites/verify` | VerifyWebsiteSocialController | affiliate: /app/profile, /user/register-steps |
| DELETE | `/affiliate/v2/social-medias/{socialMediaId}` | DeleteSocialMediaController | affiliate: /app/profile, /user/register-steps |
| GET | `/web/v2/affiliate/social-medias/login/facebook/callback` | FacebookLoginCallbackController | webhook / callback |
| GET | `/web/v2/affiliate/social-medias/login/instagram/callback` | InstagramLoginCallbackController | webhook / callback |
| GET | `/web/v2/affiliate/social-medias/login/pinterest/callback` | PinterestLoginCallbackController | webhook / callback |
| GET | `/web/v2/affiliate/social-medias/login/tiktok/callback` | TikTokLoginCallbackController | webhook / callback |
| GET | `/web/v2/affiliate/social-medias/login/youtube/callback` | YoutubeLoginCallbackController | webhook / callback |

### AFA-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/affiliate/v1/pins/toggle` | ToggleUserTypeListController | affiliate: /app/discovery-my-lists |
| GET | `/affiliate/v1/products` | IndexProductController | affiliate: /app/active-management, /app/brand/:id, /app/discovery-promote-all/:brandId?, /app/inactive-brands |
| GET | `/affiliate/v1/products/brands` | IndexBrandController | affiliate: /app/active-management, /app/brand/:id, /app/brands-discovery, /app/discovery-my-lists |
| GET | `/affiliate/v1/products/brands/drop-down` | BrandDropDownListController | affiliate: /app/partnerships/brands, /app/partnerships/products, /app/reports/brand-report, /app/reports/performance-report |
| GET | `/affiliate/v1/products/brands/{id}` | ShowBrandController | affiliate: /app/brand/:id, /app/discovery-promote-all/:brandId?, /app/partnerships/brands, /app/partnerships/products |
| GET | `/affiliate/v1/products/brands/{id}/assets` | ShowBrandAssetController | affiliate: /app/brand/:id |
| GET | `/affiliate/v1/products/brands/{id}/assets/{mediaId}` | ShowBrandAssetDetailController | affiliate: /app/brand/:id |
| GET | `/affiliate/v1/products/drop-down` | ProductDropDownListController | affiliate: /app/partnerships/products, /app/reports/product-report |
| GET | `/affiliate/v1/products/landing-page/index` | LandingPageListController | affiliate: /app/discovery-promote |
| POST | `/affiliate/v1/products/landing-page/promote/{id}` | LandingPagePromoteController | affiliate: /app/discovery-promote, /app/discovery-promote-all/:brandId? |
| GET | `/affiliate/v1/products/sellers/drop-down` | SellerDropDownListController | affiliate: /app/active-management, /app/brands-discovery, /app/discovery-my-lists, /app/discovery-promote-all/:brandId? |
| GET | `/affiliate/v1/products/{id}` | ShowProductController | affiliate: /app/active-management, /app/brand/:id, /app/brands-discovery, /app/discovery-my-lists |
| GET | `/affiliate/v1/products/{id}/assets` | ShowProductAssetController | affiliate: /app/product/:id |
| GET | `/affiliate/v1/products/{id}/assets/{mediaId}` | ShowProductAssetDetailController | affiliate: /app/product/:id |
| GET | `/affiliate/v1/products/{id}/variations` | IndexProductVariationController | affiliate: /app/product/:id |
| GET | `/affiliate/v1/profile/seller/{id}` | ShowSellerDetailController | affiliate: /app, /app/active-management, /app/brand/:id, /app/brands-discovery |
| GET | `/affiliate/v1/stores` | IndexStoreController | affiliate: /app/active-management, /app/brands-discovery, /app/discovery-promote, /app/lists/products/:id |
| POST | `/affiliate/v1/user-type-lists` | CreateUserTypeListController | affiliate: /app/discovery-my-lists |
| PATCH | `/affiliate/v1/user-type-lists/assign-items` | AssignUserTypeListItemController | affiliate: /app/brand/:id, /app/brands-discovery, /app/discovery-promote, /app/discovery-promote-all/:brandId? |
| GET | `/affiliate/v1/user-type-lists/brands` | IndexBrandListsController | affiliate: /app/discovery-my-lists |
| GET | `/affiliate/v1/user-type-lists/drop-down-brands` | UserTypeListBrandDropDownListController | affiliate: /app/brands-discovery, /app/discovery-promote |
| GET | `/affiliate/v1/user-type-lists/drop-down-products` | UserTypeListProductDropDownListController | affiliate: /app/brand/:id, /app/discovery-promote, /app/discovery-promote-all/:brandId?, /app/product/:id |
| GET | `/affiliate/v1/user-type-lists/products` | IndexProductListsController | affiliate: /app/discovery-my-lists |
| PATCH | `/affiliate/v1/user-type-lists/{id}` | UpdateUserTypeListController | affiliate: /app/brand/:id, /app/brands-discovery, /app/discovery-my-lists, /app/discovery-promote |
| DELETE | `/affiliate/v1/user-type-lists/{id}` | DeleteUserTypeListController | affiliate: /app/discovery-my-lists |
| GET | `/affiliate/v1/user-type-lists/{id}/brands` | ShowUserTypeListBrandController | affiliate: /app/lists/brands/:id |
| DELETE | `/affiliate/v1/user-type-lists/{id}/delete-items` | DeleteUserTypeListItemController | affiliate: /app/lists/brands/:id, /app/lists/products/:id |
| POST | `/affiliate/v1/user-type-lists/{id}/duplicate` | DuplicateUserTypeListController | affiliate: /app/discovery-my-lists |
| GET | `/affiliate/v1/user-type-lists/{id}/products` | ShowUserTypeListProductController | affiliate: /app/lists/products/:id |

### AFA-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| PATCH | `/affiliate/v1/partnerships/applied/brand/change-status/{id}` | ChangeStatusAppliedBrandRequestController | affiliate: /app/brand/:id, /app/partnerships/brands |
| PATCH | `/affiliate/v1/partnerships/applied/product/change-status/{id}` | ChangeStatusAppliedProductRequestController | affiliate: /app, /app/partnerships/products, /app/product/:id |
| POST | `/affiliate/v1/partnerships/brand/request` | BrandRequestController | affiliate: /app/associates/explore, /app/brand/:id, /app/brands-discovery, /app/product/:id; buyer: /seller/brand/[slug], /seller/store/[slug] |
| GET | `/affiliate/v1/partnerships/contracts` | IndexPartnershipContractController | affiliate: /app/partnerships/brands, /app/partnerships/products |
| GET | `/affiliate/v1/partnerships/contracts/{contractId}` | ShowPartnershipContractController | affiliate: /app |
| PUT | `/affiliate/v1/partnerships/contracts/{contractId}/revoke` | PartnershipContractRevokeController | affiliate: /app, /app/partnerships/brands, /app/partnerships/products |
| PATCH | `/affiliate/v1/partnerships/invited/brand/change-status/{id}` | ChangeStatusInvitedBrandRequestController | affiliate: /app, /app/partnerships/brands |
| PATCH | `/affiliate/v1/partnerships/invited/product/change-status/{id}` | ChangeStatusInvitedProductRequestController | affiliate: /app, /app/partnerships/products |
| POST | `/affiliate/v1/partnerships/product/check-pre-request` | ProductBulkRequestCheckController | affiliate: /app/brand/:id, /app/products-discovery |
| POST | `/affiliate/v1/partnerships/product/request` | ProductRequestController | affiliate: /app/associates/explore, /app/brand/:id, /app/product/:id, /app/products-discovery |
| GET | `/affiliate/v1/partnerships/requests` | IndexPartnershipRequestController | affiliate: /app, /app/partnerships/brands, /app/partnerships/products |
| GET | `/affiliate/v1/partnerships/sellers` | SellerListController | affiliate: /app/my-sellers |
| POST | `/affiliate/v1/temp_partnership/brand/confirm-request` | BrandConfirmRequestController | affiliate: /app |

### AFA-05

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/managements/brands` | BrandListController | affiliate: /app/active-management, /app/inactive-brands |
| GET | `/affiliate/v1/managements/products` | ProductListController | affiliate: /app/active-management, /app/inactive-products |
| POST | `/affiliate/v1/products/links` | CreateProductLinkController | affiliate: /app/active-management, /app/product/:id |
| GET | `/affiliate/v1/products/links/drop-down` | ProductLinkDropDownListController | affiliate: /app/landpage-list |
| GET | `/affiliate/v1/products/links/sources` | IndexProductLinkSourceController | affiliate: /app/active-management, /app/product/:id, /app/reports/brand-report, /app/reports/landingpage-report |
| POST | `/affiliate/v1/products/links/sources` | CreateProductLinkSourceController | affiliate: /app/active-management, /app/product/:id |
| GET | `/affiliate/v1/products/links/sources/drop-down` | ProductLinkSourceDropDownListController | affiliate: /app/active-management, /app/product/:id, /app/reports/source-report |
| GET | `/affiliate/v1/products/links/sources/{productLinkSourceId}` | ShowProductLinkSourceController | affiliate: /app/active-management, /app/product/:id, /app/reports/source-report |
| GET | `/affiliate/v1/products/links/{productLinkId}` | ShowProductLinkController | affiliate: /app/active-management, /app/landpage-list, /app/product/:id, /app/reports/brand-report |
| PUT | `/affiliate/v1/products/links/{productLinkId}` | UpdateProductLinkController | affiliate: /app/active-management, /app/product/:id |
| DELETE | `/affiliate/v1/products/links/{productLinkId}` | DeleteProductLinkController | affiliate: /app/product/:id |
| GET | `/affiliate/v1/products/promotions/drop-down` | PromotionDropDownListController | affiliate: /app/active-management, /app/product/:id |
| GET | `/affiliate/v1/products/{id}/promotions/drop-down` | ProductPromotionDropDownListController | affiliate: /app/active-management, /app/product/:id |
| GET | `/affiliate/v1/settings` | ShowUserTypeSettingController | affiliate: /app/active-management |
| POST | `/affiliate/v1/settings` | UpdateUserTypeSettingController | affiliate: /app/active-management |

### AFA-06

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/affiliate/v1/landing-pages` | CreateProductLandingPageController | affiliate: /app/new-landing-page, /app/update-landpage/:id |
| GET | `/affiliate/v1/landing-pages` | IndexLandingPageController | affiliate: /app/landpage-list |
| GET | `/affiliate/v1/landing-pages/drop-down` | LandingPageDropDownListController | affiliate: /app/active-management, /app/product/:id, /app/reports/landingpage-report |
| GET | `/affiliate/v1/landing-pages/template-configs` | IndexLandingPageTemplateConfigsController | affiliate: /app/new-landing-page, /app/update-landpage/:id |
| PUT | `/affiliate/v1/landing-pages/{landingPageId}` | UpdateLandingPageController | affiliate: /app/new-landing-page, /app/update-landpage/:id |
| GET | `/affiliate/v1/landing-pages/{landingPageId}` | ShowLandingPageController | affiliate: /app/active-management, /app/landpage-list, /app/new-landing-page, /app/product/:id |
| DELETE | `/affiliate/v1/landing-pages/{landingPageId}` | DeleteLandingPageController | affiliate: /app/landpage-list |
| GET | `/affiliate/v1/landing-pages/{landingPageId}/products` | IndexLandingPageProductController | affiliate: /app/landpage-list |
| PUT | `/affiliate/v1/landing-pages/{landingPageId}/set-default` | SetDefaultLandingPageController | affiliate: /app/landpage-list |
| PUT | `/affiliate/v1/landing-pages/{landingPageId}/user-type-collections` | UpdateUserTypeLandingPageCollectionController | affiliate: /app/landpage-list |

### AFA-07

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/associates/associate-brands/list` | IndexAssociateBrandController | affiliate: /app/associates/explore |
| GET | `/affiliate/v1/associates/associate-products/list` | IndexAssociateProductController | affiliate: /app/associates/explore |
| POST | `/affiliate/v1/associates/associate-products/scrape` | ScrapeAssociateProductController | affiliate: /app/associates/explore |
| GET | `/affiliate/v1/associates/associate-products/show/{id}` | ShowAssociateProductController | affiliate: /app/associates/explore |
| GET | `/affiliate/v1/associates/associate-promote-tags/list` | ListAssociatePromoteTagController | affiliate: /app/associates/analytics, /app/associates/explore, /app/associates/my-tags |
| PATCH | `/affiliate/v1/associates/associate-promote-tags/{promoteTag}/default` | SetDefaultAssociatePromoteTagController | affiliate: /app/associates/my-tags |
| GET | `/affiliate/v1/associates/associate-sale-reports/chart` | ChartAssociateSaleReportController | affiliate: /app/associates/analytics |
| GET | `/affiliate/v1/associates/associate-sale-reports/list` | IndexAssociateSaleReportController | affiliate: /app/associates/analytics |
| GET | `/affiliate/v1/associates/associate-social-medias/list` | IndexAssociateSocialMediaController | affiliate: /app/associates/my-tags |
| POST | `/affiliate/v1/associates/associate-social-medias/store` | StoreAssociateSocialMediaController | affiliate: /app/associates/explore, /app/associates/my-tags |

### AFA-08

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/integrations/ainfluencer/payouts` | IndexPayoutIntegrationController | core: AinfluencerAffiliateDriver::listPayoutsForAffiliate |
| GET | `/affiliate/v1/integrations/ainfluencer/reports/products` | IndexReportProductIntegrationController | core: AinfluencerAffiliateDriver::reportProductsForAffiliate |
| GET | `/affiliate/v1/integrations/ainfluencer/reports/summary` | ShowReportSummaryIntegrationController | core: AinfluencerAffiliateDriver::reportSummaryForAffiliate |
| GET | `/affiliate/v1/integrations/ainfluencer/reports/timeline` | IndexReportTimelineIntegrationController | core: AinfluencerAffiliateDriver::reportTimelineForAffiliate |
| GET | `/affiliate/v1/payments/charges` | IndexChargeController | affiliate: /app/payment-history/subscription |
| GET | `/affiliate/v1/payments/commission-transactions` | IndexCommissionTransactionController | affiliate: /app/payment-history/commission |
| GET | `/affiliate/v1/payments/commission-transactions/{paymentTransactionId}` | ShowCommissionTransactionController | affiliate: /app/payment-history/commission, /app/payment-history/commission-invoice/:id |
| GET | `/affiliate/v1/payments/commission-transactions/{paymentTransactionId}/export` | CommissionTransactionExportController | affiliate: /app/payment-history/commission-invoice/:id |
| GET | `/affiliate/v1/payments/commission-transactions/{paymentTransactionId}/items` | IndexCommissionTransactionItemsController | affiliate: /app/payment-history/commission-invoice/:id |
| GET | `/affiliate/v1/payments/invoices` | IndexInvoiceController | affiliate: /app/payment-history/subscription |
| GET | `/affiliate/v1/payments/onboarding-request` | OnboardingRequestController | affiliate: /app/settings |
| GET | `/affiliate/v1/payments/upcoming-commission-transactions` | IndexUpcomingCommissionTransactionController | affiliate: /app/payment-history/commission |

### AFA-09

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/plans` | IndexDefaultPlanController | affiliate: /app/manage-subscription |
| GET | `/affiliate/v1/plans/active` | ShowActivePlanController | affiliate: /app/manage-subscription |
| PUT | `/affiliate/v1/plans/cancel` | CancelPlanSubscriptionController | affiliate: /app/manage-subscription |
| PUT | `/affiliate/v1/plans/resume` | ResumePlanSubscriptionController | affiliate: /app/manage-subscription |
| GET | `/affiliate/v1/plans/subscriptions/status` | SubscriptionStatusController | affiliate: /app/manage-subscription |
| GET | `/affiliate/v1/plans/{id}/subscribe` | PlanSubscribeController | affiliate: /app/manage-subscription |
| PUT | `/affiliate/v1/plans/{id}/update` | UpdatePlanSubscriptionController | affiliate: /app/manage-subscription |

### AFA-10

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/dashboard/contracts` | IndexPartnershipContractController | affiliate: /app |
| GET | `/affiliate/v1/dashboard/report/brands-partnership` | ShowBrandPartnershipReportController | affiliate: /app |
| GET | `/affiliate/v1/dashboard/report/products-partnership` | ShowProductPartnershipReportController | affiliate: /app |
| GET | `/affiliate/v1/dashboard/report/recent-payments` | RecentPaymentsReportController | affiliate: /app |
| GET | `/affiliate/v1/dashboard/report/statistics` | ShowStatisticsReportController | affiliate: /app |
| GET | `/affiliate/v1/dashboard/report/top-products` | TopProductReportController | affiliate: /app |
| GET | `/affiliate/v1/dashboard/report/upcoming-payments` | ShowUpcomingPaymentReportController | affiliate: /app |
| POST | `/affiliate/v1/reports/brand` | BrandReportListController | affiliate: /app/reports/brand-report |
| POST | `/affiliate/v1/reports/brand/chart` | BrandReportChartController | affiliate: /app/reports/brand-report |
| POST | `/affiliate/v1/reports/brand/export` | BrandReportExportController | affiliate: /app/reports, /app/reports/brand-report |
| POST | `/affiliate/v1/reports/landing-page` | LandingPageReportListController | affiliate: /app/reports/landingpage-report |
| POST | `/affiliate/v1/reports/landing-page/chart` | LandingPageReportChartController | affiliate: /app/reports/landingpage-report |
| POST | `/affiliate/v1/reports/landing-page/export` | LandingPageReportExportController | affiliate: /app/reports, /app/reports/landingpage-report |
| GET | `/affiliate/v1/reports/performance` | PerformanceReportListController | affiliate: /app/reports/performance-report |
| GET | `/affiliate/v1/reports/performance/chart` | PerformanceReportChartController | affiliate: /app, /app/reports/performance-report |
| POST | `/affiliate/v1/reports/performance/export` | PerformanceReportExportController | affiliate: /app/reports, /app/reports/performance-report |
| POST | `/affiliate/v1/reports/product` | ProductReportListController | affiliate: /app/reports/product-report |
| POST | `/affiliate/v1/reports/product/chart` | ProductReportChartController | affiliate: /app/reports/product-report |
| POST | `/affiliate/v1/reports/product/export` | ProductReportExportController | affiliate: /app/reports, /app/reports/product-report |
| POST | `/affiliate/v1/reports/source` | SourceReportListController | affiliate: /app/reports/source-report |
| POST | `/affiliate/v1/reports/source/chart` | SourceReportChartController | affiliate: /app/reports/source-report |
| POST | `/affiliate/v1/reports/source/export` | SourceReportExportController | affiliate: /app/reports, /app/reports/source-report |

### AFA-11

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/teams/{teamId}/impersonate` | GetImpersonationTokenController | affiliate: (app layout: all /app/*) |

### AFB-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/client/buyer/v1/products/links/{productLinkId}` | ShowProductLinkController | buyer: /product/[id] |
| GET | `/client/buyer/v1/products/links/{productLinkId}/variations` | ShowProductLinkVariationsController | buyer: /product/[id] |

### AFB-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/client/buyer/v1/categories` | IndexCategoryController | buyer: /seller/brand/[slug], /seller/store/[slug], /user/[id] |
| GET | `/client/buyer/v1/profile/affiliate/{affiliateId}` | ShowAffiliateProfileLinkController | buyer: /user/[id] |
| GET | `/client/buyer/v1/profile/links/affiliate/{affiliateId}/products` | AffiliateProfileProductsController | buyer: /user/[id] |

### AFB-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/buyer/v1/auth/auth-status` | AuthStatusController | buyer: /auth?token=..&is_registered=.., ?modal=code (any page), ?modal=signin (any page) |
| POST | `/buyer/v1/profile` | UpdateProfileController | buyer: ?modal=completeprofile (any page) |
| POST | `/client/buyer/v1/auth/email-verify` | EmailVerifyController | buyer: ?modal=code (any page) |
| POST | `/client/buyer/v1/auth/forget-password` | ForgetPasswordController | buyer: ?modal=forgotpassword (any page) |
| POST | `/client/buyer/v1/auth/login` | LoginController | buyer: ?modal=signin (any page) |
| POST | `/client/buyer/v1/auth/login/amazon` | AmazonLoginController | buyer: ?modal=signin, ?modal=signup |
| POST | `/client/buyer/v1/auth/login/facebook` | FacebookLoginController | buyer: ?modal=signin, ?modal=signup |
| POST | `/client/buyer/v1/auth/login/google` | GoogleLoginController | buyer: ?modal=signin, ?modal=signup |
| POST | `/client/buyer/v1/auth/register` | RegisterController | buyer: ?modal=signup (any page) |
| POST | `/client/buyer/v1/auth/verification-code` | SendVerificationCodeController | buyer: ?modal=code (any page), ?modal=signup (any page) |
| GET | `/web/v1/buyer/login/amazon/callback` | AmazonLoginCallbackController | webhook / callback |
| GET | `/web/v1/buyer/login/facebook/callback` | FacebookLoginCallbackController | webhook / callback |
| GET | `/web/v1/buyer/login/google/callback` | GoogleLoginCallbackController | webhook / callback |
| GET | `/web/v1/client/login/amazon/callback` | AmazonLoginCallbackController | webhook / callback |
| GET | `/web/v1/client/login/facebook/callback` | FacebookLoginCallbackController | webhook / callback |
| GET | `/web/v1/client/login/google/callback` | GoogleLoginCallbackController | webhook / callback |

### AFB-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/client/affiliate/v1/products/brands/{sellerBrandSlug}` | ShowBrandController | buyer: /seller/brand/[slug] |
| GET | `/client/affiliate/v1/products/stores/{storeSlug}` | ShowStoreController | buyer: /seller/store/[slug] |
| GET | `/client/affiliate/v1/products/stores/{storeSlug}/products` | StoreProductListController | buyer: /seller/store/[slug] |
| GET | `/client/affiliate/v1/products/{sellerBrandSlug}` | ProductListController | buyer: /seller/brand/[slug] |
| POST | `/client/affiliate/v1/temp_partnership/brand/request` | BrandRequestController | buyer: /seller/brand/[slug], /seller/store/[slug] |

### AFC-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/chats/contacts` | IndexContactController | affiliate: /app/messages |
| POST | `/affiliate/v1/chats/contacts/tag` | CreateContactTagController | affiliate: /app/messages |
| DELETE | `/affiliate/v1/chats/contacts/tag` | DeleteContactTagController | affiliate: /app/messages |
| GET | `/affiliate/v1/chats/contacts/{userTypeId}/messages` | HistoryMessageController | affiliate: /app/messages |
| GET | `/affiliate/v1/chats/find-contact/{contactId}` | FindContactController | affiliate: /app, /app/active-management, /app/brand/:id, /app/brands-discovery |
| GET | `/affiliate/v1/chats/messages` | SearchMessageController | affiliate: /app/messages |
| POST | `/affiliate/v1/chats/messages` | CreateMessageController | affiliate: /app/messages |
| POST | `/affiliate/v1/chats/messages/find-page` | FindMessagePageController | affiliate: /app/messages |
| GET | `/affiliate/v1/chats/messages/saved` | IndexSavedMessageController | affiliate: /app/messages |
| POST | `/affiliate/v1/chats/messages/saved` | CreateSavedMessageController | affiliate: /app/messages |
| PUT | `/affiliate/v1/chats/read-contact-messages` | ReadContactMessagesController | affiliate: /app/messages |
| DELETE | `/affiliate/v1/message-templates/delete/{id}` | DeleteMessageTemplateController | affiliate: /app/settings |
| GET | `/affiliate/v1/message-templates/list` | ListMessageTemplateController | affiliate: /app/settings |
| POST | `/affiliate/v1/message-templates/store` | StoreMessageTemplateController | affiliate: /app/settings |
| PUT | `/affiliate/v1/message-templates/update/{id}` | UpdateMessageTemplateController | affiliate: /app/settings |
| GET | `/affiliate/v1/message-variables/list` | ListMessageVariableController | affiliate: /app/settings |
| GET/POST | `/broadcasting/auth` | BroadcastController@authenticate | seller: /app/messages; affiliate: (app layout: chat store instantiated where chat store is used), /app/messages |
| GET | `/seller/v1/chats/contacts` | IndexContactController | seller: /app/messages |
| POST | `/seller/v1/chats/contacts/tag` | CreateContactTagController | seller: /app/messages |
| DELETE | `/seller/v1/chats/contacts/tag` | DeleteContactTagController | seller: /app/messages |
| GET | `/seller/v1/chats/contacts/{userTypeId}/messages` | HistoryMessageController | seller: /app/messages |
| GET | `/seller/v1/chats/find-contact/{contactId}` | FindContactController | seller: /app (dashboard), /app/affiliate/:id, /app/discovery-affiliates, /app/lists/affiliates/:id |
| GET | `/seller/v1/chats/messages` | SearchMessageController | seller: /app/messages |
| POST | `/seller/v1/chats/messages` | CreateMessageController | seller: /app/messages |
| POST | `/seller/v1/chats/messages/find-page` | FindMessagePageController | seller: /app/messages |
| GET | `/seller/v1/chats/messages/saved` | IndexSavedMessageController | seller: /app/messages |
| POST | `/seller/v1/chats/messages/saved` | CreateSavedMessageController | seller: /app/messages |
| POST | `/seller/v1/chats/messages/send-as-bulk` | SendBulkMessageController | seller: /app/discovery-affiliates |
| POST | `/seller/v1/chats/messages/send-as-list` | SendListMessageController | seller: /app/discovery-lists |
| PUT | `/seller/v1/chats/read-contact-messages` | ReadContactMessagesController | seller: /app/messages |
| DELETE | `/seller/v1/message-templates/delete/{id}` | DeleteMessageTemplateController | seller: /app/settings |
| GET | `/seller/v1/message-templates/list` | ListMessageTemplateController | seller: /app/email-campaigns, /app/settings |
| POST | `/seller/v1/message-templates/store` | StoreMessageTemplateController | seller: /app/settings |
| PUT | `/seller/v1/message-templates/update/{id}` | UpdateMessageTemplateController | seller: /app/settings |
| GET | `/seller/v1/message-variables/list` | ListMessageVariableController | seller: /app/discovery-affiliates, /app/discovery-lists, /app/email-campaigns, /app/settings |

### AFC-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/admin/v1/integrations/ainfluencer/digests` | IndexSellerDigestsIntegrationController | core: AinfluencerAffiliateDriver::listSellerDigestsForAdmin |
| GET | `/admin/v1/integrations/ainfluencer/digests/sellers` | IndexActiveDigestSellersIntegrationController | core: AinfluencerAffiliateDriver::listDigestSellersForAdmin |
| GET | `/affiliate/v1/notifications` | NotificationListController | affiliate: (app layout: all /app/*) |
| PATCH | `/affiliate/v1/notifications/read-all` | ReadAllNotificationController | affiliate: (app layout: all /app/*) |
| GET | `/affiliate/v1/notifications/suggests` | SearchSuggestionNotificationController | affiliate: (app layout: all /app/*) |
| PATCH | `/affiliate/v1/notifications/{id}/read` | ReadNotificationController | affiliate: (app layout: all /app/*) |
| PATCH | `/affiliate/v1/notifications/{id}/toggle-bookmark` | ToggleBookmarkNotificationController | affiliate: (app layout: all /app/*) |
| GET | `/affiliate/v1/settings/notifications` | GetUserNotificationSettingController | affiliate: /app/settings |
| PUT | `/affiliate/v1/settings/notifications/update` | StoreUserNotificationSettingController | affiliate: /app/settings |
| POST | `/affiliate/v1/user-fcm-tokens` | CreateUserFcmTokenController | affiliate: (App.vue created: after login, push-permission prompt) |
| GET | `/client/v1/callback/email/unsubscribe/{token}` | UnsubscribeController | webhook / callback |
| GET | `/seller/v1/notifications` | NotificationListController | seller: (global: App/layout) |
| PATCH | `/seller/v1/notifications/read-all` | ReadAllNotificationController | seller: (global: App/layout) |
| GET | `/seller/v1/notifications/suggests` | SearchSuggestionNotificationController | seller: (global: App/layout) |
| PATCH | `/seller/v1/notifications/{id}/read` | ReadNotificationController | seller: (global: App/layout) |
| PATCH | `/seller/v1/notifications/{id}/toggle-bookmark` | ToggleBookmarkNotificationController | seller: (global: App/layout) |
| GET | `/seller/v1/settings/notifications` | GetUserNotificationSettingController | seller: /app/settings |
| PUT | `/seller/v1/settings/notifications/update` | StoreUserNotificationSettingController | seller: /app/settings |
| POST | `/seller/v1/user-fcm-tokens` | CreateUserFcmTokenController | seller: (global: App/layout) |

### AFC-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/affiliate/v1/tickets` | CreateTicketController | affiliate: /app/support-tickets |
| GET | `/affiliate/v1/tickets` | IndexTicketController | affiliate: /app/support-tickets |
| GET | `/affiliate/v1/tickets/{ticket}` | ShowTicketController | affiliate: /app/support-ticket-details/:id |
| PATCH | `/affiliate/v1/tickets/{ticket}` | CloseTicketController | affiliate: /app/support-ticket-details/:id, /app/support-tickets |
| POST | `/affiliate/v1/tickets/{ticket}/messages` | CreateTicketMessageController | affiliate: /app/support-ticket-details/:id |
| POST | `/seller/v1/tickets` | CreateTicketController | seller: /app/support-tickets |
| GET | `/seller/v1/tickets` | IndexTicketController | seller: /app/support-tickets |
| GET | `/seller/v1/tickets/{ticket}` | ShowTicketController | seller: /app/support-ticket-details/:id |
| PATCH | `/seller/v1/tickets/{ticket}` | CloseTicketController | seller: /app/support-ticket-details/:id, /app/support-tickets |
| POST | `/seller/v1/tickets/{ticket}/messages` | CreateTicketMessageController | seller: /app/support-ticket-details/:id |

### AFC-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/guidelines/{key}` | GuidelinePageController | affiliate: (app layout: all /app/*) |
| POST | `/affiliate/v1/profile/request-change-password-by-email` | RequestChangePasswordByEmailController | affiliate: /app/settings |
| GET | `/affiliate/v1/profile/sessions` | SessionListController | affiliate: /app/settings |
| DELETE | `/affiliate/v1/profile/sessions/{sessionId}` | DeleteSessionController | affiliate: /app/settings |
| GET | `/affiliate/v1/settings/client` | GetUserSettingController | affiliate: (app layout: all /app/*), /app, /app/active-management, /app/associates/analytics |
| POST | `/affiliate/v1/settings/client/store` | StoreUserSettingController | affiliate: (app layout: all /app/*), /app, /app/active-management, /app/associates/analytics |
| GET | `/affiliate/v1/system/basic-info` | BasicInfoController | affiliate: (router guard on /app/* and /user/*) |
| GET | `/seller/v1/guidelines/{key}` | GuidelinePageController | seller: (global: App/layout) |
| POST | `/seller/v1/profile/request-change-password-by-email` | RequestChangePasswordByEmailController | seller: /app/settings |
| GET | `/seller/v1/profile/sessions` | SessionListController | seller: /app/settings |
| DELETE | `/seller/v1/profile/sessions/all` | DeleteAllSessionsController | seller: /app/settings |
| DELETE | `/seller/v1/profile/sessions/{sessionId}` | DeleteSessionController | seller: /app/settings |
| GET | `/seller/v1/settings/client` | GetUserSettingController | seller: (global: App/layout), /app (dashboard), /app/affiliate/:id, /app/brand/:id |
| POST | `/seller/v1/settings/client/store` | StoreUserSettingController | seller: (global: App/layout), /app (dashboard), /app/affiliate/:id, /app/brand/:id |
| GET | `/seller/v1/system/basic-info` | BasicInfoController | seller: (global: App/layout), /app (dashboard), /app/affiliate/:id, /app/brand/:id |

### AFS-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/auth/auth-status` | AuthStatusController | seller: (global: App/layout), /app (dashboard), /app/affiliate/:id, /app/brand/:id |
| POST | `/seller/v1/auth/forget-password` | ForgetPasswordController | seller: /user/forgot-password |
| POST | `/seller/v1/auth/login` | LoginController | seller: /user/login |
| POST | `/seller/v1/auth/login/amazon` | AmazonLoginController | seller: /user/login, /user/register |
| POST | `/seller/v1/auth/login/facebook` | FacebookLoginController | seller: /user/login, /user/register |
| POST | `/seller/v1/auth/login/google` | GoogleLoginController | seller: /user/login, /user/register |
| POST | `/seller/v1/auth/mobile-verification-code` | SendMobileVerificationCodeController | seller: /user/register-steps |
| POST | `/seller/v1/auth/mobile-verify` | MobileVerifyController | seller: /user/register-steps |
| POST | `/seller/v1/auth/register` | RegisterController | seller: /user/register |
| POST | `/seller/v1/auth/reset-password` | ResetPasswordController | seller: /user/reset-password/:token |
| POST | `/seller/v1/auth/skip-step` | SkipStepController | seller: /user/register-steps |
| GET | `/seller/v1/profile` | ShowProfileController | seller: /app/profile |
| POST | `/seller/v1/profile` | UpdateProfileController | seller: /app/profile, /user/register-steps |
| GET | `/seller/v1/profile/affiliate/{id}` | ShowAffiliateDetailController | seller: /app (dashboard), /app/affiliate/:id, /app/discovery-affiliates, /app/lists/affiliates/:id |
| PUT | `/seller/v1/profile/change-mobile` | ChangeMobileNumberController | seller: /app/profile |
| GET | `/seller/v1/profile/overview` | ProfileOverviewController | seller: /app/profile |
| GET | `/seller/v1/profile/profile-completion-percentage` | ShowProfileCompletionPercentageController | seller: /app (dashboard) |
| GET | `/seller/v1/profile/questions` | ProfileQuestionListController | seller: /app/profile |
| POST | `/seller/v1/profile/questions` | StoreProfileQuestionController | seller: /app/profile |
| POST | `/seller/v1/profile/request-change-mobile` | RequestChangeMobileNumberController | seller: /app/profile |
| GET | `/seller/v1/profile/show-register-steps` | ShowRegisterStepsController | seller: /user/register-steps |
| GET | `/seller/v1/questions` | RegisterQuestionListController | seller: /user/register-steps |
| POST | `/seller/v1/questions` | StoreRegisterQuestionController | seller: /user/register-steps |
| GET | `/web/v1/seller/login/facebook/callback` | FacebookLoginCallbackController | webhook / callback |
| GET | `/web/v1/seller/login/google/callback` | GoogleLoginCallbackController | webhook / callback |

### AFS-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/profile/platform-accounts` | SellerPlatformAccountController | seller: /app/profile |
| GET | `/seller/v1/profile/platform-accounts/dropdown` | IndexPlatformAccountController | seller: (global: App/layout) |
| GET | `/seller/v1/profile/profiles` | AmazonAccountProfileDropDownListController | seller: /app/new-promotion, /app/update-promotion/:id |
| POST | `/seller/v1/profile/sync/account/amazon` | SyncAmazonAccountController | seller: /app/profile, /user/register-steps |
| POST | `/seller/v1/profile/sync/account/amazon/profile/{amazonAccountProfileId}` | SyncAmazonAccountProfileController | seller: /app/profile |
| POST | `/seller/v1/profile/sync/account/facebook` | SyncFacebookAccountController | seller: /app/profile, /user/register-steps |
| POST | `/seller/v1/profile/sync/account/google` | SyncGoogleAccountController | seller: /app/profile, /user/register-steps |
| POST | `/seller/v1/profile/sync/account/shopify` | SyncShopifyAccountController | seller: /app/profile, /user/register-steps |
| GET | `/seller/v1/profile/synced-accounts` | SyncedAccountListController | seller: /app/profile, /user/register-steps |
| DELETE | `/seller/v1/profile/synced-accounts/{id}` | DeleteSyncedAccountController | seller: /app/profile, /user/register-steps |
| DELETE | `/seller/v1/profile/synced-amazon/{id}` | DeletePlatformAccountController | seller: /app/profile, /user/register-steps |

### AFS-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/categories` | IndexCategoryController | seller: /app/discovery-affiliates, /app/discovery-brands, /app/lists/affiliates/:id, /app/partnerships/partnership-products |
| GET | `/seller/v1/products` | IndexProductController | seller: /app/affiliate/:id, /app/brand/:id, /app/discovery-affiliates, /app/discovery-lists |
| GET | `/seller/v1/products/brands` | IndexBrandController | seller: /app/brand/:id, /app/discovery-brands, /app/discovery-products, /app/reports/brands |
| PATCH | `/seller/v1/products/brands/change-auto-partnership-approval` | ChangeAutoPartnershipApprovalController | seller: /app/brand/:id |
| PUT | `/seller/v1/products/brands/change-status` | UpdateSellerBrandStatusController | seller: /app/brand/:id |
| GET | `/seller/v1/products/brands/drop-down` | BrandDropDownListController | seller: /app/affiliate/:id, /app/discovery-affiliates, /app/discovery-lists, /app/lists/affiliates/:id |
| PUT | `/seller/v1/products/brands/update/amazon-link` | UpdateSellerBrandAmazonLinkController | seller: /app/brand/:id |
| PUT | `/seller/v1/products/brands/update/socials/{id}` | UpdateSellerBrandSocialsController | seller: /app/brand/:id |
| POST | `/seller/v1/products/brands/update/terms-file` | UpdateBrandTermsFileController | seller: /app/brand/:id |
| GET | `/seller/v1/products/brands/{id}` | ShowBrandController | seller: /app/affiliate/:id, /app/brand/:id, /app/discovery-affiliates, /app/discovery-lists |
| GET | `/seller/v1/products/brands/{id}/assets` | ShowBrandAssetController | seller: /app/brand/:id |
| DELETE | `/seller/v1/products/brands/{id}/assets` | DeleteBrandAssetController | seller: /app/brand/:id |
| GET | `/seller/v1/products/brands/{id}/assets/{mediaId}` | ShowBrandAssetDetailController | seller: /app/brand/:id |
| POST | `/seller/v1/products/brands/{id}/profile` | UpdateSellerBrandProfileController | seller: /app/brand/:id |
| POST | `/seller/v1/products/brands/{id}/upload-assets` | UploadBrandAssetController | seller: /app/brand/:id |
| PUT | `/seller/v1/products/bulk-update-commissions` | BulkUpdateCommissionsProductController | seller: /app/brand/:id, /app/discovery-products |
| PUT | `/seller/v1/products/change-status` | UpdateSellerProductStatusController | seller: /app/product/:id |
| GET | `/seller/v1/products/drop-down` | ProductDropDownListController | seller: /app/partnerships/partnership-products, /app/promotion-list, /app/reports/product-report |
| GET | `/seller/v1/products/{id}` | ShowProductController | seller: /app/brand/:id, /app/discovery-brands, /app/discovery-products, /app/partnerships/partnership-products |
| POST | `/seller/v1/products/{id}/upload-assets` | UploadProductAssetController | seller: /app/product/:id |
| GET | `/seller/v1/products/{productId}/assets` | ShowProductAssetController | seller: /app/product/:id |
| DELETE | `/seller/v1/products/{productId}/assets` | DeleteProductAssetController | seller: /app/product/:id |
| GET | `/seller/v1/products/{productId}/assets/{mediaId}` | ShowProductAssetDetailController | seller: /app/product/:id |
| PUT | `/seller/v1/products/{productId}/commission` | UpdateCommissionProductController | seller: /app/product/:id |
| GET | `/seller/v1/products/{productId}/variations` | IndexProductVariationController | seller: /app/product/:id |
| GET | `/seller/v1/stores` | IndexStoreController | seller: /app/brand/:id, /app/discovery-brands, /app/discovery-products, /app/reports/brands |
| GET | `/seller/v1/stores/{id}` | ShowStoreController | seller: /app/brand/:id |

### AFS-04

| Method | Path | Controller | Caller |
|---|---|---|---|
| PUT | `/seller/v1/products/brands/update/landing-page/{id}` | UpdateSellerBrandLandingPageController | seller: /app/brand/:id |

### AFS-05

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/discover/affiliates` | IndexAffiliateController | seller: /app/discovery-affiliates, /app/messages, /app/my-affiliates, /app/partnerships/partnership-brands |
| GET | `/seller/v1/discover/affiliates/landing` | LandingDiscoverAffiliateController | seller: /app/discovery-affiliates |
| POST | `/seller/v1/pins/toggle` | ToggleUserTypeListController | seller: /app/discovery-lists |
| GET | `/seller/v1/reveal-information/list/affiliates` | ListAffiliateRevealInformationController | seller: /app/email-campaigns |
| GET | `/seller/v1/reveal-information/list/unlisted-affiliates` | ListUnlistedAffiliateRevealInformationController | seller: /app/email-campaigns |
| POST | `/seller/v1/reveal-information/process` | ProcessRevealInformationController | seller: /app/affiliate/:id, /app/discovery-lists, /app/email-campaigns, /app/expanded-network |
| GET | `/seller/v1/unlisted-affiliates` | IndexUnlistedAffiliateController | seller: /app/expanded-network |
| GET | `/seller/v1/unlisted-affiliates/filter-options` | ShowUnlistedAffiliateFilterOptionsController | seller: /app/expanded-network |
| GET | `/seller/v1/unlisted-affiliates/{id}` | ShowUnlistedAffiliateController | seller: /app/expanded-network, /app/lists/unlisted-affiliates/:id |
| POST | `/seller/v1/user-type-lists` | CreateUserTypeListController | seller: /app/discovery-lists, /app/expanded-network |
| GET | `/seller/v1/user-type-lists/affiliates` | IndexAffiliateListsController | seller: /app/discovery-lists, /app/email-campaigns |
| PATCH | `/seller/v1/user-type-lists/assign-items` | AssignUserTypeListItemController | seller: /app/affiliate/:id, /app/discovery-affiliates, /app/expanded-network, /app/lists/affiliates/:id |
| GET | `/seller/v1/user-type-lists/drop-down-affiliates` | UserTypeListAffiliateDropDownListController | seller: /app/affiliate/:id, /app/brand/:id, /app/discovery-affiliates, /app/discovery-products |
| POST | `/seller/v1/user-type-lists/revealed-stats` | RevealedStatsUserTypeListsController | seller: /app/email-campaigns, /app/lists/affiliates/:id |
| GET | `/seller/v1/user-type-lists/unlisted-affiliates` | IndexUnlistedAffiliateListsController | seller: /app/email-campaigns, /app/expanded-network |
| GET | `/seller/v1/user-type-lists/{affiliateGroupUuid}/unlisted-affiliate-membership` | UserTypeListUnlistedAffiliateMembershipController | seller: /app/expanded-network |
| PATCH | `/seller/v1/user-type-lists/{id}` | UpdateUserTypeListController | seller: /app/affiliate/:id, /app/discovery-affiliates, /app/discovery-lists, /app/expanded-network |
| DELETE | `/seller/v1/user-type-lists/{id}` | DeleteUserTypeListController | seller: /app/discovery-lists |
| GET | `/seller/v1/user-type-lists/{id}/affiliates` | ShowUserTypeListAffiliateController | seller: /app/lists/affiliates/:id |
| DELETE | `/seller/v1/user-type-lists/{id}/delete-items` | DeleteUserTypeListItemController | seller: /app/lists/affiliates/:id, /app/lists/unlisted-affiliates/:id |
| POST | `/seller/v1/user-type-lists/{id}/duplicate` | DuplicateUserTypeListController | seller: /app/discovery-lists |
| GET | `/seller/v1/user-type-lists/{id}/unlisted-affiliates` | ShowUserTypeListUnlistedAffiliateController | seller: /app/lists/unlisted-affiliates/:id |

### AFS-06

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/partnership/affiliate` | AffiliateContractListController | seller: /app/my-affiliates |
| GET | `/seller/v1/partnership/affiliate/drop-down` | AffiliateDropDownListController | seller: /app/partnerships/partnership-brands, /app/partnerships/partnership-products, /app/reports/affiliate-report |
| PATCH | `/seller/v1/partnership/applied/brand/change-status/{id}` | ChangeStatusAppliedBrandRequestController | seller: /app (dashboard), /app/partnerships/partnership-brands |
| PATCH | `/seller/v1/partnership/applied/product/change-status/{id}` | ChangeStatusAppliedProductRequestController | seller: /app (dashboard), /app/partnerships/partnership-products |
| POST | `/seller/v1/partnership/brand/check-pre-request` | BrandRequestCheckController | seller: /app/discovery-affiliates |
| POST | `/seller/v1/partnership/brand/request` | BrandRequestController | seller: /app/affiliate/:id, /app/discovery-affiliates, /app/lists/affiliates/:id |
| GET | `/seller/v1/partnership/contracts` | IndexPartnershipContractController | seller: /app/partnerships/partnership-brands, /app/partnerships/partnership-products |
| GET | `/seller/v1/partnership/contracts/{contractId}` | ShowPartnershipContractController | seller: /app (dashboard) |
| PUT | `/seller/v1/partnership/contracts/{contractId}/revoke` | PartnershipContractRevokeController | seller: /app (dashboard), /app/partnerships/partnership-brands, /app/partnerships/partnership-products |
| PATCH | `/seller/v1/partnership/invited/brand/change-status/{id}` | ChangeStatusInvitedBrandRequestController | seller: /app (dashboard), /app/partnerships/partnership-brands |
| PATCH | `/seller/v1/partnership/invited/product/change-status/{id}` | ChangeStatusInvitedProductRequestController | seller: /app (dashboard), /app/partnerships/partnership-products |
| POST | `/seller/v1/partnership/product/check-pre-request` | ProductRequestCheckController | seller: /app/brand/:id, /app/discovery-affiliates, /app/discovery-products |
| POST | `/seller/v1/partnership/product/request` | ProductRequestController | seller: /app/affiliate/:id, /app/brand/:id, /app/discovery-affiliates, /app/discovery-products |
| GET | `/seller/v1/partnership/requests` | IndexPartnershipRequestController | seller: /app (dashboard), /app/partnerships/partnership-brands, /app/partnerships/partnership-products |

### AFS-07

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/seller/v1/integrations/ainfluencer/coupons/create` | CreateCouponsIntegrationController | core: AinfluencerAffiliateDriver::createCoupon |
| GET | `/seller/v1/integrations/ainfluencer/coupons/list` | IndexCouponsIntegrationController | core: AinfluencerAffiliateDriver::listCoupon |
| GET | `/seller/v1/integrations/ainfluencer/coupons/show` | ShowCouponsIntegrationController | core: AinfluencerAffiliateDriver::showCoupon |
| GET | `/seller/v1/promotions` | IndexPromotionController | seller: /app/promotion-list |
| POST | `/seller/v1/promotions` | CreatePromotionController | seller: /app/new-promotion, /app/update-promotion/:id |
| GET | `/seller/v1/promotions/{promotionId}` | ShowPromotionController | seller: /app/new-promotion, /app/promotion-details/:id, /app/promotion-edit-assigner/:id, /app/promotion-edit-duration/:id |
| PATCH | `/seller/v1/promotions/{promotionId}` | UpdatePromotionController | seller: /app/promotion-edit-duration/:id, /app/promotion-edit/:id |
| DELETE | `/seller/v1/promotions/{promotionId}` | DeletePromotionController | seller: /app/promotion-details/:id, /app/promotion-list |
| GET | `/seller/v1/promotions/{promotionId}/affiliates` | IndexPromotionAffiliateController | seller: /app/promotion-details/:id, /app/promotion-edit-assigner/:id |
| PATCH | `/seller/v1/promotions/{promotionId}/affiliates` | AssignPromotionAffiliateController | seller: /app/promotion-details/:id, /app/promotion-edit-assigner/:id |

### AFS-08

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/email-campaigns/cancel/{id}` | CancelEmailCampaignController | seller: /app/email-campaigns |
| GET | `/seller/v1/email-campaigns/list` | ListEmailCampaignController | seller: /app/email-campaigns |
| GET | `/seller/v1/email-campaigns/resend/{id}` | ResendEmailCampaignController | seller: /app/email-campaigns |
| POST | `/seller/v1/email-campaigns/store` | StoreEmailCampaignController | seller: /app/email-campaigns |
| DELETE | `/seller/v1/smtp-connections/delete/{id}` | DeleteSmtpConnectionController | seller: /app/email-campaigns |
| GET | `/seller/v1/smtp-connections/list` | ListSmtpConnectionController | seller: /app/email-campaigns |
| POST | `/seller/v1/smtp-connections/store` | StoreSmtpConnectionController | seller: /app/email-campaigns |
| POST | `/seller/v1/smtp-connections/test` | TestSmtpConnectionController | seller: /app/email-campaigns |
| PUT | `/seller/v1/smtp-connections/update/{id}` | UpdateSmtpConnectionController | seller: /app/email-campaigns |

### AFS-09

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/dashboard/contracts` | IndexPartnershipContractController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/affiliate-partnerships` | ShowAffiliatePartnershipReportController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/brands` | ShowBrandReportController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/brands/chart` | BrandChartReportController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/contracts` | ShowContractReportController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/products` | ShowProductReportController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/promotions` | ShowPromotionReportController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/statistics` | ShowStatisticsReportController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/top-affiliates` | TopAffiliateReportController | seller: /app (dashboard) |
| GET | `/seller/v1/dashboard/report/top-products` | TopProductReportController | seller: /app (dashboard) |
| POST | `/seller/v1/reports/affiliate` | AffiliateReportListController | seller: /app/reports/affiliate-report |
| POST | `/seller/v1/reports/affiliate/chart` | AffiliateReportChartController | seller: /app/reports/affiliate-report |
| POST | `/seller/v1/reports/affiliate/export` | AffiliateReportExportController | seller: /app/reports, /app/reports/affiliate-report |
| POST | `/seller/v1/reports/brand` | BrandReportListController | seller: /app/reports/brands, /app/reports/brb-report |
| POST | `/seller/v1/reports/brand/chart` | BrandReportChartController | seller: /app/reports/brands |
| POST | `/seller/v1/reports/brand/export` | BrandReportExportController | seller: /app/reports, /app/reports/brands |
| GET | `/seller/v1/reports/brb` | BrbReportListController | seller: /app/reports/brb-report |
| GET | `/seller/v1/reports/brb/chart` | BrbReportChartController | seller: /app/reports/brb-report |
| POST | `/seller/v1/reports/brb/export` | BrbReportExportController | seller: /app/reports, /app/reports/brb-report |
| GET | `/seller/v1/reports/performance` | PerformanceReportListController | seller: /app/reports/performance-report |
| GET | `/seller/v1/reports/performance/chart` | PerformanceReportChartController | seller: /app (dashboard), /app/reports/performance-report |
| POST | `/seller/v1/reports/performance/export` | PerformanceReportExportController | seller: /app/reports, /app/reports/performance-report |
| POST | `/seller/v1/reports/product` | ProductReportListController | seller: /app/reports/brb-report, /app/reports/product-report |
| POST | `/seller/v1/reports/product/chart` | ProductReportChartController | seller: /app/reports/product-report |
| POST | `/seller/v1/reports/product/export` | ProductReportExportController | seller: /app/reports, /app/reports/product-report |

### AFS-10

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/client/v1/callback/stripe` | StripeWebhookController | webhook / callback |
| GET | `/seller/v1/integrations/ainfluencer/cards/list` | IndexCardsIntegrationController | core: AinfluencerAffiliateDriver::listCard |
| POST | `/seller/v1/integrations/ainfluencer/cards/sync` | SyncCardIntegrationController | core: AinfluencerAffiliateDriver::syncCard |
| GET | `/seller/v1/payments/charges` | IndexChargeController | seller: /app/payment-history/subscription |
| GET | `/seller/v1/payments/commission-transactions` | IndexCommissionTransactionController | seller: /app/payment-history/commission |
| GET | `/seller/v1/payments/commission-transactions/{paymentTransactionId}` | ShowCommissionTransactionController | seller: /app/payment-history/commission-invoice/:id |
| GET | `/seller/v1/payments/commission-transactions/{paymentTransactionId}/export` | CommissionTransactionExportController | seller: /app/payment-history/commission-invoice/:id |
| GET | `/seller/v1/payments/commission-transactions/{paymentTransactionId}/items` | IndexCommissionTransactionItemsController | seller: /app/payment-history/commission-invoice/:id |
| POST | `/seller/v1/payments/commission-transactions/{paymentTransactionId}/retry` | RetryCommissionTransactionController | seller: /app/payment-history/commission |
| GET | `/seller/v1/payments/create-billing-portal` | CreateBillingPortalController | seller: /app/settings |
| GET | `/seller/v1/payments/invoices` | IndexInvoiceController | seller: /app/payment-history/subscription |
| GET | `/stripe/payment/{id}` | PaymentController@show | webhook / callback |
| POST | `/stripe/webhook` | WebhookController@handleWebhook | webhook / callback |

### AFS-11

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/seller/v1/wallet/checkout/create` | CreateWalletCheckoutController | seller: /app/wallet |
| GET | `/seller/v1/wallet/transactions/list` | GetWalletTransactionsListController | seller: /app/wallet/transaction-history |

### AFS-12

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/plans` | IndexDefaultPlanController | seller: /app/manage-subscription |
| GET | `/seller/v1/plans/active` | ShowActivePlanController | seller: /app/manage-subscription |
| PUT | `/seller/v1/plans/cancel` | CancelPlanSubscriptionController | seller: /app/manage-subscription |
| PUT | `/seller/v1/plans/resume` | ResumePlanSubscriptionController | seller: /app/manage-subscription |
| GET | `/seller/v1/plans/subscriptions/status` | SubscriptionStatusController | seller: /app/manage-subscription |
| GET | `/seller/v1/plans/{id}/subscribe` | PlanSubscribeController | seller: /app/manage-subscription |
| PUT | `/seller/v1/plans/{id}/update` | UpdatePlanSubscriptionController | seller: /app/manage-subscription |

### AFS-13

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/seller/v1/teams/invitations/list` | ListTeamInvitationController | seller: /app/settings |
| PATCH | `/seller/v1/teams/invitations/{invitationId}/change-status` | ChangeTeamInvitationStatusController | seller: /app/settings |
| GET | `/seller/v1/teams/my` | MyTeamController | seller: /app/settings |
| GET | `/seller/v1/teams/permissions/list/{roleId?}` | ListTeamPermissionController | seller: /app/role-permission/:teamId/create|edit/:roleId |
| GET | `/seller/v1/teams/{teamId}` | ShowTeamController | seller: /app/settings |
| GET | `/seller/v1/teams/{teamId}/impersonate` | GetImpersonationTokenController | seller: (global: App/layout) |
| POST | `/seller/v1/teams/{teamId}/invitation` | SendTeamInvitationController | seller: /app/settings |
| POST | `/seller/v1/teams/{teamId}/members/cancel/{memberId}` | CancelTeamMemberController | seller: /app/settings |
| DELETE | `/seller/v1/teams/{teamId}/members/delete/{memberId}` | DeleteTeamMemberController | seller: /app/settings |
| GET | `/seller/v1/teams/{teamId}/members/list` | ListTeamMemberController | seller: /app/settings |
| PUT | `/seller/v1/teams/{teamId}/members/update/{memberId}` | UpdateTeamMemberController | seller: /app/settings |
| POST | `/seller/v1/teams/{teamId}/roles/create` | CreateTeamRoleController | seller: /app/role-permission/:teamId/create|edit/:roleId |
| DELETE | `/seller/v1/teams/{teamId}/roles/delete/{roleId}` | DeleteTeamRoleController | seller: /app/settings |
| GET | `/seller/v1/teams/{teamId}/roles/detail/{roleId}` | ShowTeamRoleController | seller: /app/role-permission/:teamId/create|edit/:roleId |
| GET | `/seller/v1/teams/{teamId}/roles/list` | ListTeamRoleController | seller: /app/settings |
| PUT | `/seller/v1/teams/{teamId}/roles/update/{roleId}` | UpdateTeamRoleController | seller: /app/role-permission/:teamId/create|edit/:roleId |

### AFX-01

| Method | Path | Controller | Caller |
|---|---|---|---|
| GET | `/affiliate/v1/auth/become-affiliate` | BecomeAffiliateController | seller: (global: App/layout) |
| GET | `/affiliate/v1/auth/local-login` | LocalLoginCallbackController | seller: (global: App/layout); affiliate: /auth?source=redirect&panel=X (pre-router script; opened by panel switcher in user menu) |
| POST | `/affiliate/v1/integrations/ainfluencer/register` | RegisterIntegrationController | core: AinfluencerAffiliateDriver::registerAffiliate |
| GET | `/buyer/v1/auth/local-login` | LocalLoginCallbackController | affiliate: /auth?source=redirect&panel=X (pre-router script; opened by panel switcher in user menu) |
| GET | `/seller/v1/auth/become-seller` | BecomeSellerController | affiliate: (app layout: all /app/*) |
| GET | `/seller/v1/auth/local-login` | LocalLoginCallbackController | seller: (global: App/layout); affiliate: /auth?source=redirect&panel=X (pre-router script; opened by panel switcher in user menu) |
| POST | `/seller/v1/integrations/ainfluencer/register` | RegisterIntegrationController | core: AinfluencerAffiliateDriver::registerSeller |
| GET | `/web/v1/sso/login` | SsoLoginController | core: AinfluencerAffiliateService::getRedirectUrl |

### AFX-02

| Method | Path | Controller | Caller |
|---|---|---|---|
| DELETE | `/admin/v1/integrations/ainfluencer/campaigns/delete/{id}` | DeleteCampaignIntegrationController | core: AinfluencerAffiliateDriver::deleteCampaignAmazonForAdmin |
| PUT | `/admin/v1/integrations/ainfluencer/campaigns/update/{id}` | UpdateCampaignIntegrationController | core: AinfluencerAffiliateDriver::updateCampaignAmazonForAdmin |
| GET | `/admin/v1/integrations/ainfluencer/campaigns/{id}/affiliates/list` | IndexCampaignAffiliatesIntegrationController | core: AinfluencerAffiliateDriver::listCampaignAmazonAffiliatesForAdmin |
| GET | `/admin/v1/integrations/ainfluencer/campaigns/{id}/products/list` | IndexCampaignProductsIntegrationController | core: AinfluencerAffiliateDriver::listCampaignAmazonProductItemForAdmin |
| POST | `/affiliate/v1/integrations/ainfluencer/campaigns/assign-affiliate` | AssignCampaignToAffiliateIntegrationController | core: AinfluencerAffiliateDriver::assignCampaignAmazonProductToUserForAffiliate |
| GET | `/affiliate/v1/integrations/ainfluencer/campaigns/list` | ListCampaignIntegrationController | core: AinfluencerAffiliateDriver::listCampaignsForAffiliate |
| GET | `/affiliate/v1/integrations/ainfluencer/campaigns/{id}/products` | IndexCampaignProductsItemsIntegrationController | core: AinfluencerAffiliateDriver::listCampaignAmazonProductItemForAffiliate |
| GET | `/seller/v1/integrations/ainfluencer/affiliate-programs/list` | IndexAffiliateProgramIntegrationController | core: AinfluencerAffiliateDriver::listAffiliateProgramsForAffiliate |
| GET | `/seller/v1/integrations/ainfluencer/brands/drop-down` | BrandDropDownIntegrationController | core: AinfluencerAffiliateDriver::listAmazonBrandDropdown / listShopifyBrandDropdown |
| POST | `/seller/v1/integrations/ainfluencer/campaigns/assign-items` | AssignCampaignToItemsIntegrationController | core: AinfluencerAffiliateDriver::assignCampaignItems |
| POST | `/seller/v1/integrations/ainfluencer/campaigns/create` | CreateCampaignIntegrationController | core: AinfluencerAffiliateDriver::createCampaign |
| DELETE | `/seller/v1/integrations/ainfluencer/campaigns/delete/{id}` | DeleteCampaignIntegrationController | core: AinfluencerAffiliateDriver::deleteCampaignAmazonForSeller |
| POST | `/seller/v1/integrations/ainfluencer/campaigns/products/assign` | AssignCampaignProductsToUsersIntegrationController | core: AinfluencerAffiliateDriver::assignCampaignAmazonProductToUser |
| GET | `/seller/v1/integrations/ainfluencer/campaigns/products/items` | IndexCampaignProductsItemsIntegrationController | core: AinfluencerAffiliateDriver::listCampaignAmazonProductItem |
| PUT | `/seller/v1/integrations/ainfluencer/campaigns/update/{id}` | UpdateCampaignIntegrationController | core: AinfluencerAffiliateDriver::updateCampaignAmazonForSeller |
| GET | `/seller/v1/integrations/ainfluencer/campaigns/{id}/affiliates/list` | IndexCampaignAffiliatesIntegrationController | core: AinfluencerAffiliateDriver::listCampaignAmazonAffiliatesForSeller |
| POST | `/seller/v1/integrations/ainfluencer/products/bulk-change-status` | BulkUpdateSellerProductStatusIntegrationController | core: AinfluencerAffiliateDriver::bulkChangeStatusProduct |
| POST | `/seller/v1/integrations/ainfluencer/products/bulk-generate-link` | BulkGenerateLinkProductIntegrationController | core: AinfluencerAffiliateDriver::bulkGenerateLinkAmazonProduct / bulkGenerateLinkShopifyProduct |
| PUT | `/seller/v1/integrations/ainfluencer/products/bulk-update-commission` | BulkUpdateCommissionProductIntegrationController | core: AinfluencerAffiliateDriver::bulkUpdateCommissionProduct |
| GET | `/seller/v1/integrations/ainfluencer/products/list` | IndexProductsIntegrationController | core: AinfluencerAffiliateDriver::listAmazonProduct / listShopifyProduct (X-Platform amazon-<mkt> | shopify) |

### AFX-03

| Method | Path | Controller | Caller |
|---|---|---|---|
| POST | `/admin/v1/integrations/ainfluencer/accounts/core-users` | LinkCoreUsersIntegrationController | core: AinfluencerAffiliateDriver::linkCoreUsersForAdmin |
| POST | `/client/v1/callback/shopify/track/event` | TrackPixelEventController | webhook / callback |
| POST | `/client/v1/callback/shopify/webhooks` | ShopifyWebhookController | webhook / callback |
| GET | `/seller/v1/integrations/ainfluencer/accounts/list` | IndexAccountIntegrationController | core: AinfluencerAffiliateDriver::listAccount |
| POST | `/seller/v1/integrations/ainfluencer/amazon-account/delete` | DeleteAmazonAccountIntegrationController | core: AinfluencerAffiliateDriver::deleteAmazonAccount |
| POST | `/seller/v1/integrations/ainfluencer/amazon-account/profiles/check` | CheckAmazonAccountProfileIntegrationController | core: AinfluencerAffiliateDriver::checkAmazonAccountProfile |
| GET | `/seller/v1/integrations/ainfluencer/amazon-account/profiles/list` | IndexAmazonAccountProfileIntegrationController | core: AinfluencerAffiliateDriver::listAmazonAccountProfile |
| POST | `/seller/v1/integrations/ainfluencer/amazon-account/sync` | SyncAmazonAccountIntegrationController | core: AinfluencerAffiliateDriver::syncAmazonAccount |
| POST | `/seller/v1/integrations/ainfluencer/shopify-account/delete` | DeleteShopifyAccountIntegrationController | core: AinfluencerAffiliateDriver::deleteShopifyAccount |
| POST | `/seller/v1/integrations/ainfluencer/shopify-account/stores/check` | CheckShopifyAccountStoreIntegrationController | core: AinfluencerAffiliateDriver::checkShopifyAccountStore |
| GET | `/seller/v1/integrations/ainfluencer/shopify-account/stores/list` | IndexShopifyAccountStoreIntegrationController | core: AinfluencerAffiliateDriver::listShopifyAccountStore |
| POST | `/seller/v1/integrations/ainfluencer/shopify-account/sync` | SyncShopifyAccountIntegrationController | core: AinfluencerAffiliateDriver::syncShopifyAccount |
| POST | `/seller/v1/integrations/ainfluencer/shopify-account/sync-with-token` | ReceiveShopifyTokenSyncController | core: AinfluencerAffiliateDriver::syncShopifyAccountWithToken |
| GET | `/web/v1/seller/login/amazon/callback` | AmazonLoginCallbackController | webhook / callback |
| GET | `/web/v1/seller/login/shopify/callback` | ShopifyLoginCallbackController | webhook / callback |

## 2. Client calls with no working backend route

| App | Method | Path | Problem |
|---|---|---|---|
| seller | DELETE | `seller/v1/profile/synced-shopify/{param}` | no backend route |
| seller | POST | `seller/v1/reveal-emails/user-types` | no backend route |
| seller | POST | `seller/v1/partnership/brand/request-as-list` | no backend route |
| seller | POST | `seller/v1/partnership/product/request-as-list` | no backend route |
| affiliate | DELETE | `<UNSET:VUE_APP_ALGO_PROFILE_DELETE_ALL_SESSIONS>` | unresolved path (env var not defined in spa-affiliate/.env) |
