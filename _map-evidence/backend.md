# Backend behaviour that clients don't show (core/backend/api, master)

Source: `core/backend/api` at `old-origin/master` 99921fbce (2026-10-04). Paths are relative to that repo. This file backs the System / automations parts of [../flow-map.md](../flow-map.md).

Sections: 1 scheduled jobs · 2 statuses and transitions · 3 webhooks and integrations · 4 plan gating · 5 notifications and emails · 6 existing docs.

## 1. Scheduler (cron)

Source: `app/Console/Kernel.php::schedule()` — the only place anything is scheduled (no module provider registers schedules; `routes/console.php` only has `inspire`). Timezone **UTC** (`config/app.php`). All entries use `withoutOverlapping` unless noted. Env column: blank = all environments.
Most commands only *find* rows and enqueue per-row jobs; the business effect happens in the job (named in the description).

### Offer lifecycle
| Command (signature) | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `offers:detect-frozen` | app/Console/Commands/Offer/DetectFrozenOffer.php | every 10 min | | Offers not yet frozen, with no brand LOCKED/TRANSFERRED legacy transaction, and either AWAITING_PAYMENT >1 day or a party inactive >1 day (limit 500) → `OfferFreezed` → `OfferFreezeListener` sets `offers.is_frozen=1` and pushes a "negotiation frozen (from brand/influencer)" deal stage to **User.com CRM**. No user-facing effect besides the flag. |
| `offers:frozen-awaiting-payment` | app/Console/Commands/Offer/FrozenAwaitingPayment.php | daily 00:00 | | Intended: offers AWAITING_PAYMENT for ≥4 days → User.com payment deal stage "frozen awaiting payment". **Effectively a no-op**: the SQL is `DATEDIFF(updated_at, NOW()) >= 4`, which is negative for past dates, so it never matches. |
| `offers:flag --from=2024-03-03` | app/Console/Commands/Offer/FlagOffer.php | daily 03:30 | | Non-PAID offers with chat messages in the last 24h not yet checked → `FlagJob` (staggered 5s) scans messages for **contact-info sharing** and stores a flag in `offers.extra.flag`. |
| `offers:hide-suspended-offers` | app/Console/Commands/Offer/HideSuspendedOffers.php | daily 00:00 | | Offers whose influencer integration has `qc_status` temporary/permanent suspend for ≥48h → `HideSuspendedJob` sets `offers.extra.influencer_suspended=true` (hidden from brand side). |
| `offers:prune` | app/Console/Commands/Offer/PruneRedundantOffers.php | monthly, 1st 07:30 | | PENDING offers older than 6 months with no transactions, no publish job, no brand activity → `BulkPruneOffers` deletes the chatbox conversation, violations, insight requests and **force-deletes the offer**. |
| `revision:expire` | app/Modules/ReviseOffer/Console/Commands/ExpireRevision.php | every minute | | Revisions PENDING for ≥2 days → `RevisionExpirer`: refunds any revise transaction, sets revision EXPIRED/`is_payable=0`, removes the revision chat message. |

### Campaign (Ad) lifecycle
| Command | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `ads:expiry-check` | app/Console/Commands/AdExpiryCheck.php | every minute | | ACTIVE ads whose time tables have all ended, or whose affiliate `end_date` passed → `ExpireAdJob` (app/Jobs/Ad/ExpireAdJob.php): status → EXPIRED, notifies brand `AdExpiry`, re-indexes ad in Elastic. Max dispatches per run capped. |
| `product-seedings:expiry-check` | app/Console/Commands/ProductSeedingExpiryCheck.php | hourly | | Approved product seedings whose `end_date` < today → status `expired`, ad re-synced (`ProductSeedingService::expireEnded`). |
| `ads:calculate-scores` | app/Console/Commands/Ad/CampaignScoreCalculator.php | cron `0 */72 * * *` (effectively daily at 00:00 — hour field max is 23) | | Recomputes `ads.score` (marketplace ranking) and pushes each ad to User.com (`AdDealUpdated`). |
| `invitations:send` | app/Console/Commands/SendInvitations.php | twice daily 06:00, 18:00 | | Expires brand→influencer campaign invitations older than 1 week; then for each unsent `invitation_cycle` (≥1h old) sends one **aggregated invitation email** + in-app `AggregatedInvitationSentNotification` to the influencer, marks invitations sent. |
| `invitations:delete` | app/Console/Commands/DeleteCancelledInvitations.php | every 6 h | | Hard-deletes soft-deleted (cancelled) invitations older than 24h. |
| `invitations-logs:prune` | app/Console/Commands/Invitation/Log/PruneNonPendingLogs.php | daily 02:30 | | Deletes non-pending `invitation_logs`. |

### Publish jobs & settlement (offer money)
| Command | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `publish:jobs` | app/Console/Commands/PublishJobs.php | every minute | prod filter `id > 360` | (1) automatic jobs PUBLISHED whose duration elapsed → state REMOVAL; (2) automatic due jobs → `ProcessPublishJob`; (3) manual jobs with a submitted `published_url` and no approval → `ProcessManualPublishJob`; (4) jobs with `url_approval=pending` for 48h (no link) / 14 days (with link) → `ProcessPendingPublishJob`. |
| `publish:jobs:expiry-check` | app/Console/Commands/PublishJobExpiryCheck.php | every minute | | Manual SCHEDULED healthy jobs with no URL 2 days after `start` → `ExpirePublishJobJob` (status EXPIRED, clears deadline-extend requests, notification `publish-job:publishing-failed`); REMOVAL jobs 15 min past end → expired as well. |
| `modules:publish_jobs:finalize` | app/Modules/PublishJob/Console/Commands/FinalizePublishedJob.php | every minute | | **Settlement**. Healthy PUBLISHED/COMPLETED jobs whose offer's Finance `order_item` is TRANSFERRED and that are (a) approved by brand/admin, or (b) unapproved with link 14 days after start, or (c) unapproved without link 2 days after start, or (d) pending approval 14 days → `FinalizePublishJob` fires `OrderItemCompletedEvent` (Finance releases money to influencer wallet), `PublishJobFinalized`, `PaidAmountEvent`. |
| `modules:publish_jobs:refund` | app/Modules/PublishJob/Console/Commands/RefundPublishedJob.php | every minute | | **Refund**. EXPIRED/FAILED jobs on PAID offers with TRANSFERRED order item, disapproved for 14 days or no approval for 3 days → `RefundPublishJobJob`: `OrderService::refund(orderItem)` + `PublishJobRefunded`. |
| `review-email:send` | app/Console/Commands/SendReviewEmail.php | every minute | | For each newly finished publish job (cursor in cache) without a review → three `SendReviewEmailJob` reminders to the brand (now, +N h, +M h). |

### Finance / orders / wallets
| Command | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `orders:expire-unpaid` | app/Modules/Finance/Console/ExpireUnpaidOrdersCommand.php | every minute | | Orders past `expired_at`: if PENDING → EXPIRED and coupon reservation released; otherwise just clears `expired_at`. |
| `finance:create-missed-wallets` | app/Modules/Finance/Console/CreateUserWalletsIfNotExists.php | hourly | | Backfills default/credit/lock wallets for any user or gateway missing one, and registers the user as a Stripe customer. |
| `update:brand-payment-score` | app/Console/Commands/UpdateBrandPaymentScore.php | daily 00:00 | | Recomputes `brand_details.offer_paid` / `payment_score` from legacy `transactions` (queue `brand:updated-payment-score`). |
| `score:recalculate` | app/Modules/Score/Console/Commands/BrandScoreRecalculator.php | daily 03:00 | **production only** | `UpdateBrandScore` for brands whose `brand_details` is older than a day. |

### Subscriptions / plan enforcement
| Command | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `plans:detect-downgrades` | app/Modules/PlanEnforcement/Console/Commands/DetectPlanDowngradesCommand.php | hourly | | Users whose `user_plan_snapshots` claims a paid plan but who no longer have an active Stripe subscription (cancelled + period ran out) → `SyncUserPlanJob` → `PlanChangeDetector` + `PlanQuotaEnforcer` pauses over-quota campaigns (`paused_by=system`) / removes over-quota seats. Limit 2000/run. |
| `subscriptions:resume-due-pauses` | app/Console/Commands/Subscription/ResumeDuePausesCommand.php | hourly | | Safety net for a missed Stripe webhook: subscriptions whose pause period elapsed are unpaused at Stripe first, then local `paused_at`/`pause_resumes_at` cleared and `SyncSubscriptionPauseJob` re-syncs (unfreezes) campaigns. |

### Team
| Command | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `teams-brand:expire-member-invitations` | app/Modules/Team/Brand/Console/Commands/ExpireTeamMemberInvitation.php | every minute | | Pending brand team invitations to non-registered emails older than `TEAM_INVITATION_EXPIRATION` (default 7 days) → status EXPIRED. |

### Notifications / digests / emails
| Command | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `brand-digest:send` | app/Modules/BrandDigest/Console/SendBrandDailyDigestCommand.php | daily at `brand_digest.send_at` (default 08:00) | | One daily email per brand with affiliate (links, partners, join requests — from affiliate service) and seeding (requests, accepted, delivered) activity since last run; queued in batches (`SendBrandDigestBatchJob`). Replaced the gifting-requests digest. |
| `digest-emails:influencer-campaigns` | app/Console/Commands/DigestEmails/InfluencerCampaigns.php | weekly Mon 08:00 | | Weekly "campaigns for you" email to every influencer account (de-duped per user, Mailgun-throttled, resumable cycle). |
| `digest-emails:brand-influencers` | app/Console/Commands/DigestEmails/BrandInfluencers.php | weekly Tue 08:00 | | Weekly "influencers for your campaign" email per active+approved ad to its brand. |
| `digest-emails:brand-without-campaign` | app/Console/Commands/DigestEmails/BrandWithoutCampaign.php | weekly Wed 08:00 | | Weekly influencer suggestions to brands that have no campaign. |
| `conversations:aggregated-message-notifier` | app/Modules/ChatBox/Console/Commands/AggregatedConversationMessageNotifier.php | hourly (no overlap TTL) | | Finds non-admin users with unread chat messages → `FetchUnreadMessages` per user (unread-messages reminder). |

### Data, scoring, search index, scraping
| Command | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `integration:update-trust-metric` | app/Console/Commands/Integration/UpdateInfluencerTrustMetric.php | twice monthly (1st & 16th) | | Recomputes `influencer_details.trust_metric` from accepted/rejected offers, delivered deals, reviews. |
| `categories:bind-categories` | app/Console/Commands/BindInfluencersToCategories.php | monthly | | Caches influencers per featured category from Elastic. |
| `elastic:sync-identity-emails` | app/Console/Commands/Elastic/SyncIdentityEmails.php | twice monthly | | Fills missing `identities.email` from Elastic. |
| `elastic:sync-identities` | app/Console/Commands/Elastic/SyncIdentities.php | weekly | | Imports identity data from the Elastic `integration` index into MySQL. |
| `elastic:manual-index-integrations` | app/Console/Commands/Elastic/Integration/ManualIndexIntegration.php | monthly (1st) | | Re-fetches account info (scraper API) for integrations, re-uploads avatar and updates integration/Elastic (throttled 15/min, resumable). |
| `integration:fill-temporary-from-scraper` | app/Console/Commands/Integration/FillTemporaryIntegrationsFromScraperCommand.php | every 30 min | | Pending Instagram temporary integrations missing external id/followers → `FillTemporaryIntegrationFromScraperJob`. |
| `crawlzo:refresh-instagram --scheduled` | app/Modules/Crawlzo/Console/Commands/StartMonthlyRefreshCommand.php | daily 02:00 (acts once per month) | **production only** | Starts the monthly Crawlzo bulk refresh of all active Instagram influencers (canary job first). No-op if this month already started or a previous run is unfinished. |
| `crawlzo:poll-jobs` | app/Modules/Crawlzo/Console/Commands/PollJobsCommand.php | every 15 min | | Polls Crawlzo bulk jobs, downloads results in chunks, re-dispatches stalled chunks, advances runs (username fallback, archiving not-found accounts). |

### Cleanup / housekeeping
| Command | File | Frequency | Env | What it does |
|---|---|---|---|---|
| `notifications:clear` | app/Console/Commands/Notifications/DropNotificationTableCommand.php | daily 14:00 (no overlap guard) | | **Rebuilds the `notifications` table keeping only the last 30 days** — in-app notifications older than 30 days disappear. |
| `mongo:prune-temporary-posts` | app/Console/Commands/Integration/PruneTemporaryPostsCommand.php | hourly at :25 | | Keeps newest N (default 10) scraped posts per integration in Mongo temporary post collections (time-boxed, resumable). |
| `public-api:prune-webhook-logs` | app/Modules/PublicApi/Console/Commands/PruneWebhookLogsCommand.php | daily | | Deletes outbound webhook delivery logs older than 30 days (PublicApi module is otherwise inert). |
| `storage:set-visibility` | app/Console/Commands/Storage/UpdateFileVisibility.php | twice daily 02:00, 14:00 | | Makes private files public in storage (`MakeFilesPublic`). |
| `twig:clear-cache` | app/Console/Commands/DeleteTwigCache.php | monthly | | Deletes email template (Twig) cache. |
| `table:clean --table=failed_jobs --days=3` | app/Console/Commands/CleanTableRecordsCommand.php | daily | production + staging | Purges failed_jobs older than 3 days. |
| `telescope:prune` (`--hours=N` hourly) | vendor | hourly | local + staging only | Telescope retention (the production branch inside `scheduleTelescopeCommands` is unreachable). |

### Not scheduled / disabled (for reference)
- Imported in Kernel but **not scheduled**: `AddMultipleIntegration` (manual agency tool), `AggregatedTwilioMessageNotifier` (class file missing).
- `commentedCommands()` (never called): `SendEmailWaitingPaymentCommand`, `UpdateIdentityEmail`, `prune:files`, `PruneFileStorage`, `IntegrationSuspensionChecker`, `IntegrationRestorationChecker`; also commented `table:clean --table=logs`.
- `GiftingRequestsDigestCommand` — superseded by `brand-digest:send`, no longer scheduled.
- `plans:enforce-quotas` / `plans:repair-system-pauses` — manual-only (PlanEnforcement README).
- No affiliate-sync command is scheduled (`app/Console/Commands/AinfluencerAffiliate/RegisterBrandsInAffiliateCommand.php` is manual). Affiliate data arrives via webhooks/API calls.
- Scheduled but effectively no-op: `offers:frozen-awaiting-payment` (SQL bug above). `crawlzo:refresh-instagram` is a no-op on 29-30 days of each month by design.
- Legacy-data caveat: `offers:detect-frozen` and `update:brand-payment-score` read the legacy `transactions` table; offers paid purely through the Finance module (orders) may not be seen by them.

## 2. Statuses and transitions

Roles: offers.identity_id = influencer (maker), target_identity_id = brand. Two payment stacks: Finance/Order (current: Order, OrderItem, PaymentReceipt, wallets default/lock/credit) and legacy transactions/CheckoutRequest (still routed).
### Offer (Models/v2/Offer.php)
status: pending, rejected, awaiting-payment (default), paid, refunded (expired, suspended never written). Fields: state (inbox: read/unread/archive/spam/block/deals_closed...), flag_status (admin/FlagOffer cron), is_frozen (DetectFrozenOffer), extra.influencer_suspended (HideSuspendedOffers).
Transitions: influencer creates (mobile OfferCreatorService) → awaiting-payment (TemporaryOffer if integration qc pending → made permanent when admin activates integration); brand counter (v2 OfferController@counter) → pending; influencer accept (mobile offers/{id}/history action=accept) → awaiting-payment; reject by either → rejected; pay (POST finance/payment → OrderPaidEvent → TransferOrderTransactions + OfferPaidListener) → paid + PublishJob scheduled; legacy pay v2/offers/{offer}/pay; refund (admin disapprove / refund cron) → refunded; admin repay / reset-to-awaiting-payment / waiting-for-influencer.
Overall status shown (Modules/OfferOverallStatus): Waiting Payment; In Negotiation / Waiting Influencer Approval / Waiting your Approval; Rejected by Brand/Influencer/You; paid → Failed / Published (& Approved / & Waiting Approval) / Disapproved by Admin / Scheduled; refunded → Disapproved by Admin / Published / Failed; Expired.
### Ad / Campaign (Models/v2/Ad.php)
status draft/active/paused/expired/approval(owner suspended); approval draft-pending/pending/approved/rejected; paused_by brand/admin/system/subscription_pause. Live on marketplace = active+approved. Wizard (AdCreationProcess) creates draft → submit (Steps/Medias) → pending → admin approve (AdApprovedEvent: invitations sent, may auto-pause over quota) / reject (AdRejectedNotification). Brand pause/resume. System pause on plan downgrade (PlanEnforcement) or subscription pause (CampaignFreezer). ads:expiry-check → expired (AdExpiry notif). Admin reactivate. Brand suspension → ads approval/rejected "Owner suspended"; restore → pending re-review.
### PublishJob
state scheduled/published/removal/completed; status healthy/failed/expired; url_approval null/pending/admin:|brand:approved|disapproved. Created on offer paid. Influencer submits links (mobile PublishJobController) → url_approval pending. Expiry: start+2d no link → expired (publishing-failed). Brand/admin approve (offer/v2/{id}/approve) → completed + approved. Brand disapprove → failed brand:disapproved (refund later by cron); admin disapprove → immediate refund. FINALIZE cron (every min): approved, or link & 14d after start, or no link 2d after start → influencer lock → default wallet (publish-job:transferred notif). REFUND cron: disapproved 14d / no approval 3d → refund brand. Deadline extension (mobile extendPublishJob) → scheduled again.
### Money (current Finance)
Order: pre_factor (basket) → pending (30 min lifetime) → paid → completed; pending → expired (ExpireUnpaidOrders, coupon released); refunded.
OrderItem: pending → transferred (escrow: brand → influencer lock) → completed (lock → influencer default) → settled (admin payout paid); refunded.
PaymentReceipt deposits: DEPOSIT_PENDING → REQUESTED (redirect gateway) → PAID / FAILED / CANCELED. Withdrawals: influencer POST finance/mobile/payouts → WITHDRAW_IN_PROGRESS (email admins) → influencer cancel → CANCELED; admin reject → REJECTED; admin pay (finance/admin/payouts/pay) → COMPLETED (WithdrawPaidByAdminNotification). Payouts paid manually by admin.
Legacy: transactions pending/locked/transferred/refunded/cancelled/revising/failed/expired ("On hold in Escrow" etc.); CheckoutRequest pending → done (v2/user/wallet/checkouts, admin v2/admin/users/{user}/payouts).
Admin wallet adjustment: finance/admin/modify-wallet.
### Subscription (Cashier)
stripe_status incomplete/active/past_due/canceled/unpaid/paused...; ends_at (grace), paused_at/pause_resumes_at, scheduled downgrade. Plans free/starter/pro/ultimate/custom. subscribe (Stripe Checkout) → webhook → SyncUserPlanJob; upgrade = swap+invoice now; downgrade = scheduled at period end then campaigns over quota paused; cancel → grace → resume possible; end → campaigns paused, seats removed; pause N months (pause_collection, campaigns frozen, features kept) → unpause / auto-resume. No own trial logic.
### Product seeding (gifting)
Influencer participation: requested (influencer applied) / pending_accept (brand invited) → accepted → shipped → received → creating_content → published; rejected, archived. Brand drives most transitions; influencer only shipped→received and publishing a post; responds to invitation. Address frozen once dispatched. Campaign: draft → pending → approved/rejected (admin) → paused/expired (hourly expiry). Shipping via Shopify fulfillment + Track123.
### Revisions
Current: RevisionOffer pending → paid (brand pays delta via finance/payment/entity) → approved/rejected by influencer (v3/mobile/offer/{id}/revise/accept|reject); overwritten, expired. Limit 2 per offer (setting). Legacy ReviseOffer pending/accepted/rejected/expired/overridden (ExpireRevision cron 2 days).
Admin offer edits (EditOfferMessage): pending → accepted/rejected by influencer, discarded on brand counter.
### Suspension
Influencer integration: AutomaticSuspension rules (Instagram) on integration create; qc_status active/pending/draft/temporary suspend/permanent suspend; offers to integrations suspended ≥48h hidden. Brand: BrandDetail active/temporary suspend/permanent suspend by admin; appeal (pending/approved/disapproved) restores.
### Cancellation/refund summary
Before pay: reject; order expires 30 min. After pay before completion: admin disapprove / refund cron / brand self-refund (finance/current-user/orders/refund) / admin refund → lock → brand default. After completion: money in influencer default wallet until payout.

## 3. Webhooks and integrations

### Inbound webhooks / callbacks (master) — condensed
| Route | Controller | Effect | Live |
|---|---|---|---|
| POST /api/webhooks/stripe | StripeController | legacy charges/transfers -> transactions status | legacy wired |
| POST /api/subscriptions/webhook | Subscription/CashierWebhookController | subscription created/updated/deleted, pause, downgrade reconcile, SyncUserPlanJob; invoice.payment_succeeded -> active; StripeEventListener stores invoices/charges/PIs, subscription PaymentReceipt | live (main subscription webhook) |
| POST /stripe/webhook | vendor Cashier default | plain Cashier, skips custom logic | accidental registration |
| POST /api/en/finance/stripe/webhook | Finance StripeWebhookController | payment_intent.succeeded -> PaymentReceiptPaidEvent (wallet credit, ChargeWalletNotification, gateway->brand transfer, fee); failed -> DEPOSIT_FAILED | live (current checkout) |
| POST /v3/webhooks/stripe | Payment module | broken route | dead |
| POST /api/webhook, /webhook/publish-job/{id}, /webhook/insta-dm/{id}, /webhook/instagram/publish-job | WebhookController | legacy Instagram publisher health/publish/DM | legacy, reachable |
| POST /api/v2/webhook/signup | v2/Webhook/SignupController | landing-page signup: user+wallet+stripe customer, verification email | live |
| POST /api/v2/webhooks/scraper/notify, /scraper/tiktok/reports/video | ScraperController | post index update; TikTok video report -> publish job published_date, metrics, media | live |
| POST /api/v3/webhooks/{instagram,tiktok}/dm-verification | InfluencerOnBoarding DmVerificationWebhookController | DM code -> IntegrationVerification -> CompleteDmVerificationJob (creates integration) | live |
| POST /api/v2/chatbox/webhook | ChatBox WebhookController | message-sent -> offer last_message + push/db Message notifications | live |
| POST /api/v2/webhooks/translations/callback | Translation callback | BrandID video translate success/fail | live |
| POST user-third-party/brandid/v1/{user-created,username} | BrandIDIntegrationController | connect_brandid_account, store username | live |
| POST user-third-party/track123/webhook | Track123WebhookController | shipment tracking -> Shopify fulfillment status (product seeding) | live |
| POST user-third-party/aftership/webhook | Aftership | old tracking driver | effectively dead |
| POST user-third-party/shopify/webhook/{app-uninstall,get-customer,erase-customer} | ShopifyWebhookController | uninstall deletes connection; GDPR stubs | uninstall live |
| POST ainfluencer-affiliate/v1/webhook | AinfluencerAffiliateWebhookController | amazon/shopify account deleted, campaign.updated counters, shopify synced | live |
| POST ainfluencer-affiliate/v1/sso/callbacks/{user-registered,social-media-sync} | AinfluencerAffiliateSsoController | link affiliate account; return social accounts | live |
Browser returns: PayPal charge & pay-offer success/cancel (/v3/callbacks/paypal/*, legacy wallet/escrow), /paypal/callback (PayPal payout connect), finance/payment/callback/{success,failed}/{tracking} (Finance checkout: PaymentReceiptPaid + OrderPaid/EntityPaid), finance/stripe/{callback,refresh} (Stripe Connect payout), legacy api/payment/stripe/connect/callback, Shopify OAuth auth/callback & app home.
Notes: no PayPal/NowPayments server webhook — state changes only via browser return. Finance Stripe webhook does not fire OrderPaidEvent; if user doesn't return to success URL, order may stay unpaid (support-relevant symptom).

### Outbound integrations (master) — condensed
- Stripe Cashier subscriptions: SubscriptionController (subscriptions/*), CashierWebhookController, DetectPlanDowngrades/ResumeDuePauses. Brand signup creates Stripe customer (prod/staging).
- Stripe Finance gateways (card / stripe redirect), gated by DB gateways.active: CardController (cards), WalletController (top-up), PaymentController (offer/basket pay), PaymentCallbackController. Stripe Connect new flow: Finance StripeController (stripe/connect, callback, refresh); legacy wallets/{wallet}/stripe/connect still registered.
- Influencer PAYOUTS ARE MANUAL: PayoutController only creates payout receipts; Admin AdminPayoutController records payment by hand; StripeGateway::payout / PaypalGateway::payout never called. Crypto payout: address stored, paid manually (Finance/crypto-payout.md).
- PayPal: Finance gateway (createPayment/verify) live + legacy v2 stack (/paypal/connect, charge, pay-offer) + /v3/callbacks/paypal/*.
- NowPayments crypto pay-in via Finance gateway if active.
- Shopify: user-third-party/shopify/v1/* (connect, products, campaign start/save/finish; fulfill-to-influencer, complete-fulfill, add-tracking-info). Track123 tracking live, AfterShip dead.
- Ainfluencer Affiliate platform: SSO authorize-url, programs/accounts/amazon/shopify/products/campaigns/coupons/cards controllers, mobile AdAffiliate/AffiliateReport, brand digest affiliate source. Brands are NOT auto-registered on affiliate at signup (artisan command only).
- BrandID: link/billboard/tips + webhooks; video translation driver (TranslationRequestPaidListener → TranslateVideo jobs). HeyGen inactive.
- ActiveCampaign: DEAD (all triggers commented). User.com (CXA): live (signup, onboarding updates, offer/payment deal stages).
- OpenAI: daily offer flagging (FlagOffer 03:30), campaign title/description & hashtags, influencer summary.
- Internal AI service (pricing, gender, language, quality score). Crawlzo = default Instagram scraper (monthly refresh, poll 15 min). IG/TikTok/YouTube scrapers for published-content capture. Aigrow only hashtag search & InstaDM.
- Advanced analytics: InsightIQ via api/v1/advance-analytics (feature-gated or wallet-charged).
- FCM push (mobile notifications, chat). Twilio: SMS/WhatsApp OTP + abandoned cart SMS live; Twilio chat legacy.
- Mail: SES SMTP for transactional; Mailgun SMTP for external invitations + digests.
- reCAPTCHA on login/register/team signup. Google & Apple OAuth live (3rd-party/*); Instagram OAuth unconfirmed.
- PublicApi outbound webhooks: infrastructure only, nothing emits.

## 4. Plan / feature gating

Plan features live in `plan_price_feature` (per Stripe plan price); keys in `app/Helpers/Enums/Subscription/FeatureNameEnum.php`. Request-time check: middleware alias `feature:<name>` → `app/Http/Middleware/CheckFeature.php` → `app/Services/Susbscription/FeatureGuardService.php::can()`; denial = **HTTP 402 `feature_limit_exceeded`** (`FeatureLimitExceededException`, carries max/current quota). Count-based features increment `feature_usages` on success and roll back if the request fails (≥400). Downgrade reconciliation: `app/Modules/PlanEnforcement` (README is good) — `campaign.activation.quota` middleware + `CampaignSlotGate` at every wizard submit + hourly `plans:detect-downgrades` / Stripe webhook → `SyncUserPlanJob`.

### Route-level gates (from routes.json; prod paths are `/api/en/...` for v2/v3)
| Route | Controller | Gate | Notes |
|---|---|---|---|
| POST api/v2/campaign/create/step1 | AdController@storeStep1 | `feature:campaigns` | Only when `request` is empty or `create-campaign`; other steps pass. |
| POST api/v2/campaigns/{ad}/duplicate | AdController@duplicate | `feature:campaigns` | |
| POST api/v3/campaigns | AdCreationProcessController@store | `feature:campaigns` | v3 wizard; slot also asserted at submit (`AbstractAdCreationProcess:133`). |
| POST api/v3/campaigns/affiliate | AdCreationProcessAffiliateController@store | `feature:campaigns` | |
| PUT api/v2/campaigns/{ad} | AdController@update | `campaign.activation.quota` | Only when the update would (re)activate a campaign: blocks if subscription **paused** (400) or no free campaign slot (402). |
| PATCH api/v2/campaigns-affiliate/{id} | AdAffiliateController@update | `campaign.activation.quota` | Same; extending an expired affiliate campaign also calls `CampaignSlotGate` in `AdAffiliateService:129`. |
| POST api/product-seedings/handle | ProductSeedingController@handle | `feature:product_seeding_access` | Only checked at `current_step=finish` (launch). Slot asserted in `ProductSeedingService:615`. |
| POST api/v2/user/invitations | InvitationController@store | `feature:affiliates_invites` | Usage = count of `identities` + `favorite_list_id`. Rolled back when admin rejects ad (`Admin/AdService:210`). |
| POST api/v2/temp-invitations | TemporaryInvitationController@store | `feature:affiliates_invites` | |
| POST api/v2/teams/brand/members/invite | MemberInvitationController@invite | `feature:seats` | Usage = count of `members`; seats value excludes owner (Pro stores 2 = "3 seats"). Lifetime counter — rolled back on cancel (`MemberInvitationController:77`). |
| POST api/v2/teams/brand/members/register | MemberController@register | `feature:seats` | |
| GET api/v1/advance-analytics | AdvanceAnalyticsController@get | `feature:advanced_analytics` | If not allowed, falls through with `pay_base=wallet` (paid per report at `deep_analytics_price` from wallet) instead of 402. |

### In-code gates / plan-valued settings
| Feature key | Where enforced | Effect |
|---|---|---|
| `campaigns` (slot count) | `PlanEnforcement/Services/CampaignSlotGate.php` called from `AbstractAdCreationProcess`, `AdService:649` (re-activate on update), `AdAffiliateService:129`, `AinfluencerAffiliateService:1450` (assigning items = submitting affiliate campaign), `ProductSeedingService:615` | Counts campaigns *held* (active+approved/pending, gifting in review), via `CampaignQuotaPolicy::usage()`. On downgrade oldest running campaigns paused `paused_by=system`; newly approved over-quota campaign paused at approval (`ApprovedCampaignAdmission` on `AdApprovedEvent` / `CampaignModerated`). |
| `seats` | `SeatQuotaPolicy` on downgrade | Cancels open invitations first, then removes the oldest members (membership rows only). |
| `collab_transaction_fee_percent` | `app/Modules/Payment/Helpers/AmountHelper.php::get_fee_percentage()` (+ duplicate `app/Helpers/Financial/AmountHelper.php`), `Mail/AwaitingPaymentOffer` | Brand's platform fee on offer payments (fallback env default 0.2). |
| `min_required_payment_per_collabs` | `app/Http/Requests/v2/Mobile/Offer/MakeOfferRequest.php` | Minimum offer price an influencer may propose to that brand (brand's plan). Used by `v2/Mobile/OfferController`. |
| `deep_analytics_price` | `AdvanceAnalyticsProxy:120` | Wallet price of an advanced-analytics report. |
| `influencer_username_reveal` / `_price` | `app/Services/Susbscription/InfluencerRevealService.php`, routes `POST api/integrations/{id}/reveal-username` (quota) and `/reveal-username-by-wallet` (paid) — `UsernameRevealController`; auto-unlock on paid offer (`Observers/v2/OfferPaidUnlockObserver`) | Unmasking influencer usernames. |
| `affiliate_commission_percent`, `brands_connected_affiliates` (and `asin`) | Enforced by the **affiliate service**; core maps its 402 to `FeatureLimitExceededException` (`AinfluencerAffiliateService:1151, 1194`) | |
| Subscription paused | `EnforceCampaignActivationQuota::assertSubscriptionIsNotPaused`, `Services/Susbscription/Pause/CampaignFreezer.php` | Campaigns frozen & read-only while paused (see `documentations/cancellation-flow-api.md §3.6-3.7`). |

### Features with no enforcement in core (display only in plan comparison — `PlanService::getComparePlans`)
`affiliates_influancers_chats`, `collab_with_influancers`, `affiliates_emails_reveal`, `ai_dubbing_videos`, `branding_landing_page`, `affiliate_auto_payouts_escrow`, `premium_affiliate_tools`, `dedicated_success_manager`, `product_seeding_creator_outreach`, `product_seeding_shipment_content_tracking`. (`FeatureGuardService` has `can*` helpers for several, but no caller outside the `feature:` middleware, and no route uses those keys. They may be enforced in the affiliate service or frontend.)

## 5. Notification & email catalogue

**How channels work**
- Notifications extend `app/Notifications/Core/QueueableNotification.php` (queue `notification`).
- The default `via()` in `BaseNotification` sends on `mail` + `fcm` + `database`, or only database + fcm if the user `is_unsubscribed`. Nothing is sent to archived or deleted users.
- `shouldSend()` checks the per-user setting in the NotificationSetting module (`isChannelEnabledFor`; items are seeded in `Modules/NotificationSetting/Database/Seeders/NotificationItemSeeder.php`).
- Channel keys come from `app/Broadcasting/Enums/Notification/Channel.php`: `mail`, `fcm` (push), `database` (in-app), `twilio-sms`, `twilio-whats-app`. There is no `broadcast` channel.
- Mailables extend `app/Mail/Core/BaseMailable.php` and are sent via the `Mail::sendToUser` macro or `MailHelper::sendEmail`.

**NotificationTemplate**
- Model: `app/NotificationTemplate.php` (fields name, title, body, html, variables), managed in the admin UI (`Admin/NotificationTemplateController`).
- Loaded by name in `OfferNotification` (`:116`), `PublishJobNotification` (`:81`), `Message` (`:76`) and `AdApproval` (`:47`), and also by `InvitationCreated`, `AdExpiry`, `AdDrafted`, `AdRejectedNotification` and `UserQualityControlNotification`.
- Defaults come from `app/Notifications/NotificationTemplates.php::generateTemplateRecords()`; `php artisan notifications:reset-templates` (`app/Console/Commands/ResetNotificationTemplates.php`) truncates and reseeds them and refuses to run on production.
- Template keys:
  - Chat / wallet / invitations: `chat:new-message`, `wallet:payment-success`, `wallet:withdraw-success`, `invitation:created`, `invitation:created-external`
  - Offers: `brand.offer:created`, `offer:updated`, `brand.offer:accepted`, `influencer.offer:accepted`, `influencer.offer:paid`, `brand.offer:paid`
  - Publish jobs: `[brand.|influencer.]publish-job:{published,completed,publishing-failed,completion-failed,transferred,health-failed}`
  - Integrations: `{influencer,brand,mixed}.integration:health-failed`
  - Ads: `ad:approved`, `ad:rejected`, `ad:expiry`, `ad:<draft states>`
  - Cancel/refund flow: `*.offer:cancel*` / `*.offer:refund*` / `*force-cancelled` / `*force-refunded` (no current trigger found)
- Event enums: `app/Notifications/Enums/{OfferEvents,PublishJobEvents}.php`.

**Notifications**

| Name | Trigger (file) | Recipient | Channel |
|---|---|---|---|
| `Notifications/OfferNotification` (template `brand.offer:created` / `offer:updated` / `brand.offer:accepted` / `*.offer:paid`) | `Observers/v2/OfferObserver.php:30`; `Http/Controllers/OfferController.php:354,368`; `Services/Implementation/OfferService.php:752,956`; `Repositories/Offer.php:878`; Mobile `OfferHistoryService.php:188,213`, `OfferUpdaterService.php:413`; `Modules/Offer/Listeners/OfferPaidListener.php:98,118` | brand (created / accepted / updated by influencer / paid); influencer (paid) | mail, fcm, db, sms (`toTwilioSms`) |
| Mobile `Offer/OfferPaidNotification` | `OfferPaidListener.php:101` | influencer | mail, fcm, db |
| Mobile `Offer/OfferUpdatedNotification` | `OfferService.php:759,967`, `Repositories/Offer.php:890`, `Jobs/OfferUpdatedMessage.php:71` | influencer | mail, fcm, db |
| Mobile `Offer/OfferRejectedNotification` | `Http/Controllers/OfferController.php:377`, Mobile `OfferHistoryService.php:223`, `Jobs/OfferUpdatedMessage.php:69` | influencer | mail, fcm, db |
| Mobile `Offer/StartChangeNotification` | `OfferService.php:1352` | influencer | db, fcm |
| Mobile `Offer/AdminUpdatedOfferNotification` | Modules `OfferController.php:410`, `Admin/OfferService.php:303` | influencer | mail, fcm, db |
| Mobile `Offer/RevisionRequestedByBrandNotification` | `Offer/Listeners/SendRevisionOfferNotificationListener.php:31`, legacy `ReviseOffer/Traits/ReviseNotifier.php:31` | influencer | fcm, db |
| Mobile `Offer/ReviewSubmittedByBrandNotification` | `Admin/ReviewService.php:92,140` (review approved) | influencer | mail, fcm, db |
| `Modules/Offer/Notifications/AcceptedRevisionNotification` / `RejectedRevisionNotification` | `SendRevisionOfferNotificationListener.php:34,37` | brand | db, mail (+sms on accepted in the ReviseOffer copy) |
| `ReviseOffer/Notifications/Accepted*` / `Rejected*` (legacy duplicates) | `ReviseOffer/Traits/ReviseNotifier.php:33,35` | brand | db, mail, sms |
| `Offer/` and `ReviseOffer/` `PendingRevisionNotification` | imported only | brand | fcm (no usage found) |
| `Reminders/AwaitingPaymentFirstReminder` / `AwaitingPaymentSecondReminder` | `Repositories/Offer.php:567,573`, Mobile `OfferCreatorService.php:237,243` (delayed) | brand | mail |
| `AbandonedCarts/FirstAbandonedNotification` / `SecondAbandonedNotification` / `AbandonedSmsNotification` | `Services/Implementation/v3/PaymentTrackingService.php:51,64,71,77`; test command | brand | mail+db / sms |
| `Message` (template `chat:new-message`) | `Modules/ChatBox/Http/Controllers/WebhookController.php:168-176`, `Jobs/Offer/SendOfferInitialMessagesJob.php`, `ChatboxMessageSenderService.php:159` | brand / influencer | db, fcm |
| ChatBox `ConversationMessageNotification` | `ChatBox/Services/ConversationMessageNotifier.php:67` (chat webhook) | chat participant | db, fcm |
| ChatBox `ConversationAggregatedMessage` | `ChatBox/Jobs/FetchUnreadMessages.php:50` (`AggregatedConversationMessageNotifier`, hourly) | any user with unread messages | mail |
| `PublishJobNotification` (static `dispatch()` sends to the brand plus a Mobile counterpart to the influencer) | `PublishJobFinalizedListener:41`, `PublishJobRefundedListener:38`, `Jobs/ProcessPublishJob.php:119,137,148`, `Jobs/ProcessContentMetrics.php`, `Http/Controllers/WebhookController.php`, `PublishJobController.php:216,241`, `Jobs/DisApprovedAdmin.php:71`, `Jobs/PublishJob/ExpirePublishJobJob.php:58`, `Services/Implementation/PublishJobService.php:397`, legacy `Payment/Jobs/Offer/Process{Finalization,Refundation}` | brand | mail, fcm, db |
| Mobile `PublishJob/{PublishJobPublished,PublishJobCostTransferred,PublishJobFailed,PublishJobCompletionFailed}Notification` | `Notifications/PublishJobNotification.php:428-443` | influencer | mail, fcm, db |
| Mobile `ContentPublishing/PublishApprovedByBrandNotification` | `Jobs/ApprovedBrand.php:63` | influencer | fcm |
| Mobile `ContentPublishing/PublishReminderNotification` | `Jobs/PostReminderJob.php:63` | influencer | fcm |
| Mobile `ContentPublishing/PublishDeliveredNotification` / `PublishFailedNotification` | Only instantiated inside other notifications (`PublishJobCostTransferred` / `PublishJobFailed`) | influencer | fcm |
| Mobile `PublishApprovedByAdminNotification`, `PublishDisapprovedByAdminNotification` | no usage found | | fcm |
| `ApprovedByAdminNotification` | `Jobs/ApprovedAdmin.php:61` | brand | mail, db |
| `DisapprovedByAdminNotification` | `Jobs/DisApprovedAdmin.php:74` | brand | mail, fcm, db |
| `AdApproval` (`ad:approved`) | `Admin/AdService.php:416`, `AdService.php:530`, legacy `AdController.php:692` | brand | mail, fcm, db |
| `AdRejectedNotification` | `Listeners/v2/Ad/AdRejectedEventListener.php:34` | brand | mail, fcm, db |
| `AdDrafted` | `Jobs/Ad/AdDraftedJob.php:48` (+24h/+72h from AdObserver) | brand | mail, db |
| `AdExpiry` | `Jobs/Ad/ExpireAdJob.php:50` (AdExpiryCheck, every minute) | brand | mail, fcm, db |
| `AdPaused` | `Admin/AdService.php:179`, `AdService.php:574` | brand | mail, db |
| `Insight` | `Listeners/InsightCreatedListener.php:29` | influencer | mail, fcm, db |
| Mobile `Invitation/AggregatedInvitationSentNotification` | `Console/Commands/SendInvitations.php:255` (twice daily) | influencer | db, fcm |
| `InvitationCreated`, Mobile `InvitationCreatedNotification` | Seeder / queue names only | influencer | no usage found |
| `BrandStatusChangeNotification` / `BrandAdditionalVerificationNotification` | `Listeners/v2/Identity/BrandStatusChangeListener.php:85,55` | brand | mail |
| Mobile QC `Account{TemporarilySuspended,PermanentlySuspended,Activated}Notification` | `Listeners/v2/Integration/UserQualityControlListener.php:72-82` | influencer | mail, fcm, db |
| Mobile `Account/AccountUnarchivedNotification` | `Admin/IntegrationService.php:323` | influencer | mail, fcm, db |
| Mobile `Temporary/AccountVerifiedNotification` / `AccountRejectedNotification` | `Admin/TemporaryIntegrationService.php:192,239,259` | influencer | mail, fcm |
| Mobile `Account/ViolationNotification` | `Listeners/v2/NewViolationListener.php:33` | influencer | mail, fcm, db |
| `ChangeEmailNotification` / Mobile `Auth/ChangeEmailNotification` | `Services/Implementation/UserService.php:579,581` | brand / influencer | mail, db |
| Mobile `Auth/VerificationNotification` | Mobile `UserService.php:59`, `Repositories/User.php:42` | influencer | mail, fcm, db |
| `User/Notifications/EmailOtpNotification` / `PhoneOtpNotification` | `Modules/User/Login/Services/OtpService.php:67,82` | any user | mail / sms |
| `LoginSecurityNotification` | `Services/Auth/LoginSecurityService.php:73` | any user | db, mail (SES) |
| `CustomizedAdminNotification` | `Admin/UserService.php:269` | any user (admin-composed) | mail, db |
| `Twilio/SmsNotification` | `Admin/BrandSuspensionService.php:220`, `Tools/Strategies/Verify/Strategies/PhoneVerificationStrategy.php:259`, test controller | brand / user | sms |
| `Twilio/WhatsAppNotification` | Test controller only | | whatsapp |
| Finance `ChargeWalletNotification` | `Finance/Listeners/PaymentReceiptPaidListener.php:68` | brand | mail, db |
| Finance `CreditReceivedNotification` | `Finance/Http/Controllers/Admin/AdminChargeController.php:81` | brand | mail, db |
| Finance `CouponAssignedNotification` | `Finance/Services/CouponService.php:274` | brand | mail |
| Finance `InfluencerRefundNotification` | `TransferRefundOrderTransactionsListener.php:122` | influencer | mail, db |
| Finance `WithdrawPaidByAdminNotification` | `Admin/AdminPayoutController.php:258` | influencer | db (no `toMail`) |
| Translation `TranslationRequestPaidNotification` | `Translation/Listeners/TranslationRequestPaidListener.php:39` | brand | mail, db |
| ProductSeeding `GiftingCampaignApprovedNotification` / `GiftingCampaignRejectedNotification` | `ProductSeeding/Listeners/NotifyBrandOfCampaignModeration.php:32,35` | brand | db, fcm, mail |
| ProductSeeding `CampaignInvited` / `CampaignAccepted` / `CampaignShipped` | `NotifyParticipantsOfStatusChange` via `ParticipationNotificationRules` | influencer | db, fcm, mail |
| ProductSeeding `GiftingRequestReceived` / `ProductReceived` / `CampaignPublished` | Same | brand | db, fcm, mail |
| ProductSeeding `GiftingRequestsDigestNotification` | `ProductSeeding/Console/GiftingRequestsDigestCommand.php:60`; **not scheduled** (replaced by BrandDigest) | brand | mail |
| Shopify `NeedInfluencerAddress`, `ImportTrackingInfo`, `FulfillmentDelivered` | `UserThirdParty/Services/NotificationShopifyFulfillmentServiceDecorator.php:49,115,152` | influencer | fcm, mail |
| Shopify `AddAddressByInfluencerNotification` | Same file `:89` | brand | fcm |
| Shopify `FulfillmentDeliveredBrandNotification` | Same file `:149` | brand | fcm, mail |
| Digest `v2/Digest/Influencer/CampaignNotification` | `Services/DigestEmails/Influencer/CampaignsEmail.php:103` ← `Console/Commands/DigestEmails/InfluencerCampaigns` (Mon 08:00) | influencer | mail (Mailgun SMTP) |
| Digest `v2/Digest/Brand/InfluencerNotification` | `Services/DigestEmails/Brand/InfluencersEmail.php:112` ← `BrandInfluencers` (Tue 08:00) | brand | mail (Mailgun SMTP) |
| Digest `v2/Digest/Brand/BrandWithoutCampaignNotification` | `Services/DigestEmails/Brand/BrandWithoutCampaign/Email.php:101` ← `BrandWithoutCampaign` (Wed 08:00) | brand | mail (Mailgun SMTP) |
| `BrandDigest/Notifications/BrandDailyDigestNotification` | `BrandDigest/Jobs/SendBrandDigestBatchJob.php:97` ← `SendBrandDailyDigestCommand` (daily at `BRAND_DIGEST_SEND_AT`, default 08:00; config `app/Modules/BrandDigest/Config/brand_digest.php`) | brand | mail |
| `TestNotification` | `Console/Commands/FCMTest.php` | test | fcm |
| No usage found | `AdminEditOfferToInfluencerNotification`, `InfluencerPostApprovedByBrand`, `InfluencerPostFailedToPublish`, `InfluencerPostSuccessfullyPublished`, `InfluencerUnarchivedNotification`, `UserQualityControlNotification`, `TemporaryIntegration/Account{Verified,Rejected}Notification` (all only in seeders); `OfferStartChangeNotification`, `PostReminderNotification`, `VerificationNotification`, `v2/ReviewSubmittedNotification` (imported, not instantiated), `v2/ViolationSentNotification`, `TestSendVideoNotification` | | |

**Mailables**

| Name | Trigger (file) | Recipient | Channel |
|---|---|---|---|
| `Mail/EmailVerification` | `User::sendEmailVerificationNotification` (`Models/v2/User.php:783`, `app/User.php:679`), `Repositories/v2/EmailVerification/EmailVerificationRepository.php` | user | mail |
| `Mail/EmailVerificationLink` | `Models/v2/User.php:796` | user | mail |
| `Mail/PasswordReset`; Mobile / MobileBrand `PasswordResetWithCode` | `Models/v2/User.php:806,815,1160`, `Repositories/v2/PasswordReset/PasswordResetRepository.php` | user | mail |
| `Mail/v2/ConnectPaypalEmail` | `OfferPaidListener.php:107`, `Traits/Paypal/ConnectionReminder.php:43`, `ConnectPaypalReminder` command (not scheduled) | influencer | mail |
| `Mail/InfluencerPostApprovedByAdmin` | `Jobs/ApprovedAdmin.php:65` | influencer | mail |
| `Mail/DisapprovedByAdmin` | `Jobs/DisApprovedAdmin.php:69` | influencer | mail |
| `Mail/InfluencerPostDisapprovedByBrand` | `Jobs/DisApprovedBrand.php:52` | influencer | mail |
| `Mail/PostFailed` | `Jobs/SendEmailPostFailed.php:42` ← `ProcessPublishJob` | influencer | mail |
| `Mail/PostDeleted` | `Http/Controllers/WebhookController.php:67` | brand | mail |
| `Mail/AdminEditOfferToBrand` | Modules `OfferController.php:400`, `Admin/OfferService.php:293` | brand | mail |
| `Mail/AdRejected`, `Mail/AdApproval` | `AdService.php:534`, legacy `AdController.php:692,696` | brand | mail |
| `Mail/v2/ReviewCreatedEmail` | `Jobs/PublishJob/SendReviewEmail.php:94` ← `SendReviewEmail` command (every minute, EmailTracker) | review recipient (`reviewee->user`) | mail |
| `Mail/Invitations/AggregatedInvitation` / `AggregatedExternalInvitation` | `Helpers/InvitationHelper.php:80,84` ← `SendInvitations` (twice daily) | influencer (registered: SES / external: Mailgun) | mail |
| `Mail/v2/Admin/InChatEmail` | `Admin/TemplateService.php:261` | offer participant (admin-sent) | mail |
| `Mail/v2/Admin/InfluencerUnarchived` | `Admin/IntegrationService.php:318` | influencer | mail |
| `Mail/v2/Admin/InvitationReport` | `Jobs/Ad/Admin/AfterInvitationByFilter.php:40` | admin | mail |
| `Mail/NewTask` | `Helpers/TaskHelper.php:18`, `Traits/Tasks/CreatesTask.php:20` | super admin | mail |
| `Mail/v2/WithdrawRequestCreated` | Legacy `Financial/WalletService.php:157` (production only) | super admin | mail |
| `Finance/Mail/AdminPayoutEmail` | `Finance/Listeners/SendPayoutEmailToAdminsListener.php:24` | admins (`PAYOUT_ADMIN_EMAILS`) | mail |
| `Mail/AdminAddCredit` | Legacy `Financial/FinanceService.php:223` | brand | mail |
| `Mail/BrandDepositConfirmed` | Legacy `Financial/Wallet/ChargeService.php:105` | brand | mail |
| `Mail/ContactUs` | `StaticWebController.php:146` | submitter (if registered) | mail |
| `Team/Brand/Email/MemberInvitation` | `Team/.../MemberInvitationService.php:138` | invitee email | mail |
| `Translation/Email/TranslationCompleted` | `Translation/Listeners/TranslationUploadedListener.php:38` | brand | mail |
| `Mail/AwaitingPaymentOffer` | `Jobs/SendAwaitingPaymentEmailToBrands.php:40` ← `SendEmailWaitingPaymentCommand` (schedule commented out) | brand | mail (manual only) |
| `Mail/IgNotConnect` | `Jobs/SendEmailNotConnect.php:43` ← `SendEmailNotConnect` command (not scheduled) | influencer | mail (manual only) |
| `Bundles/Marketing/ColdEmail/Mail/ColdEmail` | `Console/Commands/Marketing/ColdEmail/BulkSendFromFile.php:190` | marketing list | mail |
| `Mail/MockDriver` | `Drivers/InstagramMockDriver.php` | dev | mail |
| No usage found | `AfterPaymentMade` (its job `SendEmailAfterPayment` is never dispatched), `ExternalInfluencerInvitation`, `InfluencerPostApprovedByBrand` (mail), `LegacyAggregatedInvitation` (queue name only), `v2/Admin/UserSuspended`, `v2/Offer/BrandWalletChargedEmail` (queue name only), `WelcomeEmail`, `Team/Brand/Email/MemberJoinedTeam`, `AdminAddMessageToBrand`, `AdminAddMessageToInfluencer` (seeder only), `Annotations/Mail/Subscribeable` / `Unsubscribeable` | | |

**Uncertainties (section 5)**
- Channel columns are `via()` intersected with the `toX` methods each class implements. At runtime the per-user NotificationSetting toggles can also suppress a channel.
- I did not check which classes in this catalogue have `notification_items` rows; classes without a row may default to enabled or disabled.
- Recipients of the `Message` and `ConversationMessageNotification` chat notifications are whichever participant did not send the message.
- `ReviewCreatedEmail` goes to `reviewee->user`; I did not confirm whether that is the brand or the influencer.
- The usage search matched `new X(`, `X::class` and `X::dispatch` (with alias detection) and excluded Tests. A class built dynamically (e.g. `new $notification(...)` in ProductSeeding) shows up only through its rules map; other dynamic constructions may have been missed.

## 6. Existing docs worth citing

| Doc | What it describes | Support value |
|---|---|---|
| documentations/cancellation-flow-api.md | Subscription cancel modal: cancellation summary, survey forms, **pause instead** (pause/unpause, auto-resume, what a pause stops, campaigns read-only while paused), cancel/resume. | High — authoritative pause/cancel behaviour. |
| documentations/subscription.md | Postman-style list of subscription endpoints (subscribe, plans, receipts…), generated 2026-04-15. | Medium — endpoint reference, no business rules. |
| documentations/chatroom.md | (Persian) design discussion for a brand↔influencer chat-room model with per-deal context. A proposal, not current behaviour. | Low. |
| documentations/chat-system-decoupling-plan.md | Phased plan to decouple chat from offers (`chat_conversations`). Plan, not shipped behaviour. | Low. |
| documentations/identity-user-integration-chat-findings.md | (Persian) findings on User / Identity / Integration layering, reveal being user-level, team/seat being user-level. | Medium — explains data model quirks. |
| documentations/feature_test.md | Test plan for plan-feature gating (invites, campaigns, seats, advanced analytics, deep analytics price, min payment per collab). | Medium — matches §6. |
| documentations/checklist.md | AP-65: security notification after 2 wrong passwords in 6h. | Low. |
| docs/AffiliateIntegrationReports.md | Affiliate reports & payouts read API (summary, timeline, products, payouts; Amazon + Shopify combined). | Medium — what the affiliate analytics screen shows. |
| docs/PublicApiRunbook.md / docs/ApiKeyAndWebhookPortingPlan.md | API-key + outbound webhook engine (PublicApi module). Phase 1: **built but inert** (middleware attached to no route; only internal keys). | Low for support. |
| docs/InternalInfluencerSearchApi.md | Machine-to-machine influencer catalogue API (internal key). | Low. |
| docs/endpoints.md, docs/Crawlzo.postman_collection.json | Crawlzo Instagram scraper vendor API. | Low. |
| docs/metabase-queries.md | Metabase SQL for brand/influencer KPIs (signups, paid, retention…). | Medium for ops reporting. |
| app/Modules/PlanEnforcement/README.md | Downgrade reconciliation, campaign/seat quotas, gifting pause semantics, manual repair commands. | High. |
| app/Modules/BrandDigest/README.md | Brand daily digest: how a run works, de-dupe, sections. | Medium. |
| app/Modules/Finance/docs.md | Wallet model: default / lock / credit wallets per holder, transfer rules. | High — explains balances. |
| app/Modules/Finance/crypto-payout.md | Crypto payouts: influencer picks currency/network, finance pays manually and records the hash. | High for payout tickets. |
| app/Modules/NotificationSetting/README.md | How notifications become user-switchable (`notification_items`); unregistered ones fail open. | Medium. |
| app/Modules/Crawlzo/README.md | Onboarding bio-code verification + monthly bulk refresh via Crawlzo. | Medium. |
| app/Modules/UserThirdParty/Docs/ainfluencer-affiliate-*.md (campaign, card, coupon, program, sso) | Affiliate integration APIs incl. SSO and callbacks (user registered, social sync, Amazon account sync/delete). | Medium. |
| app/Modules/FormBuilder/README.md | Code-defined forms (seeders must be run on deploy). | Low. |
