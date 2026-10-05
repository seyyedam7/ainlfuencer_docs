# Ainfluencer support knowledge base

How Ainfluencer and the Ainfluencer affiliate platform work, written for the support team and for the support assistant (Claude) that answers questions from these documents.

## What's here

| Path | What it is |
|---|---|
| [flow-map.md](flow-map.md) | The list of all topics (flows) with IDs, who they affect and where they happen. Start here. |
| `topics/` | One document per topic, grouped by domain (for example `topics/offers/OFR-03-paying-for-an-offer.md`). |
| [glossary.md](glossary.md) | Terms used across the documents (escrow, publish job, contract, …). |
| [_guide/writing-guide.md](_guide/writing-guide.md) | How topic documents are written and updated. |
| `_map-evidence/` | Raw inventories extracted from the code, used to build and verify the documents. Not meant for reading during support. |

## How support should use it

1. Find the topic: search for the feature, screen name or error the user mentions, or scan [flow-map.md](flow-map.md).
2. Open the topic document. "Common questions" usually has a ready answer; "What to check" says what to look at; "Escalate when" says when to hand over to the dev or finance team.
3. If nothing covers the question, say so and report the gap so a document can be added.

## Instructions for the support assistant

- Answer only from these documents. If the answer isn't here, say it isn't documented instead of guessing.
- Always cite the topic ID(s) you used, for example "(OFR-03)".
- Give support a short answer they can send to the user, then the checks and escalation path.
- The "Technical reference" sections are for you and for escalation; don't paste endpoint paths or table names into answers meant for users.
- Brands and influencers must message each other through the Ainfluencer chat (CHAT-01). Never suggest the affiliate platform's own chat.

## Source of truth

All documents describe the production code as of the commits listed in [flow-map.md](flow-map.md#sources). When the code changes, the affected topics need updating; each document lists the commit it was verified against in its header.

## All topics

**Account and sign-in**

- [ACC-01 · Sign-up and email verification](topics/account/ACC-01-sign-up-and-email-verification.md)
- [ACC-02 · Login, social login and sessions](topics/account/ACC-02-login-social-login-and-sessions.md)
- [ACC-03 · Passwords: forgot, reset, change, set](topics/account/ACC-03-passwords-forgot-reset-change-set.md)
- [ACC-04 · Change email](topics/account/ACC-04-change-email.md)
- [ACC-05 · Profile basics and account deletion](topics/account/ACC-05-profile-basics-and-account-deletion.md)

**Brand account and team**

- [BRD-01 · Brand onboarding](topics/brand-account/BRD-01-brand-onboarding.md)
- [BRD-02 · Brand profile, dashboard and brand score](topics/brand-account/BRD-02-brand-profile-dashboard-and-brand-score.md)
- [BRD-03 · Team members](topics/brand-account/BRD-03-team-members.md)
- [BRD-04 · Brand suspension and appeal](topics/brand-account/BRD-04-brand-suspension-and-appeal.md)

**Influencer social accounts**

- [INF-01 · Connecting a social account](topics/influencer-accounts/INF-01-connecting-a-social-account.md)
- [INF-02 · Media-kit onboarding](topics/influencer-accounts/INF-02-media-kit-onboarding.md)
- [INF-03 · Account review (quality control)](topics/influencer-accounts/INF-03-account-review-quality-control.md)
- [INF-04 · Account suspension, violations and appeal](topics/influencer-accounts/INF-04-suspension-violations-appeal.md)
- [INF-05 · Managing social accounts](topics/influencer-accounts/INF-05-managing-social-accounts.md)
- [INF-06 · Account data refresh](topics/influencer-accounts/INF-06-account-data-refresh.md)

**Finding and inviting influencers**

- [DSC-01 · Influencer search and profiles](topics/discovery/DSC-01-influencer-search-and-profiles.md)
- [DSC-02 · Favourite lists](topics/discovery/DSC-02-favourite-lists.md)
- [DSC-03 · Campaign invitations](topics/discovery/DSC-03-campaign-invitations.md)

**Campaigns (brand side)**

- [CMP-01 · Choosing a campaign type and creating a paid collaboration](topics/campaigns/CMP-01-choosing-a-type-and-creating-a-paid-collaboration.md)
- [CMP-02 · Campaign review by admin](topics/campaigns/CMP-02-campaign-review-by-admin.md)
- [CMP-03 · Managing campaigns](topics/campaigns/CMP-03-managing-campaigns.md)
- [CMP-04 · Affiliate campaigns (brand)](topics/campaigns/CMP-04-affiliate-campaigns-brand.md)
- [CMP-05 · Product gifting (seeding) campaigns (brand)](topics/campaigns/CMP-05-product-gifting-campaigns-brand.md)
- [CMP-06 · Shipping and tracking for gifts and Shopify orders](topics/campaigns/CMP-06-shipping-and-tracking-for-shopify-orders.md)

**Campaign marketplace (influencer side)**

- [MKT-01 · Exploring campaigns and brands](topics/marketplace/MKT-01-exploring-campaigns-and-brands.md)
- [MKT-02 · Invitations (influencer side)](topics/marketplace/MKT-02-invitations-influencer-side.md)
- [MKT-03 · Joining affiliate campaigns (influencer)](topics/marketplace/MKT-03-joining-affiliate-campaigns.md)
- [MKT-04 · Joining gifting campaigns (influencer)](topics/marketplace/MKT-04-joining-gifting-campaigns.md)

**Offers and negotiation**

- [OFR-01 · Making an offer](topics/offers/OFR-01-making-an-offer.md)
- [OFR-02 · Negotiation: counter, accept, reject](topics/offers/OFR-02-negotiation-counter-accept-reject.md)
- [OFR-03 · Paying for an offer](topics/offers/OFR-03-paying-for-an-offer.md)
- [OFR-04 · Changing dates: publish date and deadline extension](topics/offers/OFR-04-changing-dates.md)
- [OFR-05 · Revisions after payment](topics/offers/OFR-05-revisions-after-payment.md)
- [OFR-06 · Admin changes to offers](topics/offers/OFR-06-admin-changes-to-offers.md)

**Content delivery and settlement**

- [PUB-01 · Scheduling and submitting the post](topics/content-delivery/PUB-01-scheduling-and-submitting-the-post.md)
- [PUB-02 · Approving or disapproving content](topics/content-delivery/PUB-02-approving-or-disapproving-content.md)
- [PUB-03 · Releasing money and automatic refunds](topics/content-delivery/PUB-03-releasing-money-and-automatic-refunds.md)
- [PUB-04 · Insights and post analytics](topics/content-delivery/PUB-04-insights-and-post-analytics.md)
- [PUB-05 · Reviews and ratings](topics/content-delivery/PUB-05-reviews-and-ratings.md)
- [PUB-06 · Content translation](topics/content-delivery/PUB-06-content-translation.md)

**Money: wallets, refunds, payouts**

- [PAY-01 · Brand wallet, cards and billing history](topics/money/PAY-01-brand-wallet-cards-and-billing-history.md)
- [PAY-02 · Refunds to the brand](topics/money/PAY-02-refunds-to-the-brand.md)
- [PAY-03 · Influencer wallet and earnings](topics/money/PAY-03-influencer-wallet-and-earnings.md)
- [PAY-04 · Payout methods and withdrawals](topics/money/PAY-04-payout-methods-and-withdrawals.md)
- [PAY-05 · Coupons and credits](topics/money/PAY-05-coupons-and-credits.md)

**Subscriptions and plan limits**

- [SUB-01 · Plans and buying or upgrading](topics/subscriptions/SUB-01-plans-and-buying-or-upgrading.md)
- [SUB-02 · Downgrade, cancel and resume](topics/subscriptions/SUB-02-downgrade-cancel-and-resume.md)
- [SUB-03 · Pausing a subscription](topics/subscriptions/SUB-03-pausing-a-subscription.md)
- [SUB-04 · Plan limits ("upgrade" pop-ups)](topics/subscriptions/SUB-04-plan-limits.md)

**Messaging**

- [CHAT-01 · Brand–influencer chat](topics/chat/CHAT-01-brand-influencer-chat.md)
- [CHAT-02 · Unread counts and new-message alerts](topics/chat/CHAT-02-unread-counts-and-alerts.md)
- [CHAT-03 · System messages and offer cards in chat](topics/chat/CHAT-03-system-messages-and-offer-cards.md)
- [CHAT-04 · Contact-sharing detection](topics/chat/CHAT-04-contact-sharing-detection.md)

**Notifications and emails**

- [NTF-01 · In-app notifications](topics/notifications/NTF-01-in-app-notifications.md)
- [NTF-02 · Push notifications and where they open](topics/notifications/NTF-02-push-notifications-and-where-they-open.md)
- [NTF-03 · Email and notification preferences](topics/notifications/NTF-03-email-and-notification-preferences.md)
- [NTF-04 · Digest and reminder emails](topics/notifications/NTF-04-digest-and-reminder-emails.md)

**Store and partner connections**

- [INT-01 · Shopify store connection](topics/integrations/INT-01-shopify-store-connection.md)
- [INT-02 · BrandID (sister product)](topics/integrations/INT-02-brandid-sister-product.md)

**Public pages**

- [PUBL-01 · Public campaign preview and join](topics/public-pages/PUBL-01-public-campaign-preview-and-join.md)
- [PUBL-02 · Public influencer profile (media kit)](topics/public-pages/PUBL-02-public-influencer-profile-media-kit.md)
- [PUBL-03 · Public content view](topics/public-pages/PUBL-03-public-content-view.md)

**Affiliate: Ainfluencer ↔ affiliate platform**

- [AFX-01 · Single sign-on and linked accounts](topics/affiliate-connection/AFX-01-single-sign-on-and-linked-accounts.md)
- [AFX-02 · Ainfluencer affiliate campaigns on the affiliate platform](topics/affiliate-connection/AFX-02-ainfluencer-affiliate-campaigns-on-the-affiliate-platform.md)
- [AFX-03 · Store connections (Amazon, Shopify)](topics/affiliate-connection/AFX-03-store-connections-amazon-shopify.md)

**Affiliate: seller app**

- [AFS-01 · Seller sign-up, login and onboarding](topics/affiliate-seller/AFS-01-seller-signup-login-onboarding.md)
- [AFS-02 · Connecting and switching stores](topics/affiliate-seller/AFS-02-connecting-and-switching-stores.md)
- [AFS-03 · Products, brands and commissions](topics/affiliate-seller/AFS-03-products-brands-and-commissions.md)
- [AFS-04 · Brand landing page](topics/affiliate-seller/AFS-04-brand-landing-page.md)
- [AFS-05 · Finding affiliates, lists and revealing emails](topics/affiliate-seller/AFS-05-finding-affiliates-lists-and-revealing-emails.md)
- [AFS-06 · Partnerships (seller side)](topics/affiliate-seller/AFS-06-partnerships-seller-side.md)
- [AFS-07 · Promotions (coupon codes)](topics/affiliate-seller/AFS-07-promotions-coupon-codes.md)
- [AFS-08 · Email campaigns to affiliates](topics/affiliate-seller/AFS-08-email-campaigns-to-affiliates.md)
- [AFS-09 · Reports and dashboard (seller)](topics/affiliate-seller/AFS-09-reports-and-dashboard-seller.md)
- [AFS-10 · Commission billing and card](topics/affiliate-seller/AFS-10-commission-billing-and-card.md)
- [AFS-11 · Seller wallet](topics/affiliate-seller/AFS-11-seller-wallet.md)
- [AFS-12 · Seller subscription and plan limits](topics/affiliate-seller/AFS-12-seller-subscription-and-plan-limits.md)
- [AFS-13 · Seller teams and roles](topics/affiliate-seller/AFS-13-seller-teams-and-roles.md)

**Affiliate: affiliate app**

- [AFA-01 · Affiliate sign-up, login and onboarding](topics/affiliate-creator/AFA-01-affiliate-signup-login-onboarding.md)
- [AFA-02 · Connecting promotion platforms](topics/affiliate-creator/AFA-02-connecting-promotion-platforms.md)
- [AFA-03 · Discovering products, brands and sellers](topics/affiliate-creator/AFA-03-discovering-products-brands-sellers.md)
- [AFA-04 · Partnerships (affiliate side)](topics/affiliate-creator/AFA-04-partnerships-affiliate-side.md)
- [AFA-05 · Affiliate links, coupons and redirect settings](topics/affiliate-creator/AFA-05-affiliate-links-coupons-redirects.md)
- [AFA-06 · Landing pages and storefront](topics/affiliate-creator/AFA-06-landing-pages-and-storefront.md)
- [AFA-07 · Associates (Amazon Associates via Ainfluencer)](topics/affiliate-creator/AFA-07-associates.md)
- [AFA-08 · Earnings and payouts (affiliate)](topics/affiliate-creator/AFA-08-earnings-and-payouts.md)
- [AFA-09 · Affiliate subscription and limits](topics/affiliate-creator/AFA-09-affiliate-subscription-and-limits.md)
- [AFA-10 · Reports and dashboard (affiliate)](topics/affiliate-creator/AFA-10-reports-and-dashboard-affiliate.md)
- [AFA-11 · Teams (affiliate side)](topics/affiliate-creator/AFA-11-teams-affiliate-side.md)

**Affiliate: shared**

- [AFC-01 · Seller–affiliate chat (not supported, use the Ainfluencer chat)](topics/affiliate-shared/AFC-01-seller-affiliate-chat-not-supported.md)
- [AFC-02 · Notifications and emails](topics/affiliate-shared/AFC-02-notifications-and-emails.md)
- [AFC-03 · Help center (tickets)](topics/affiliate-shared/AFC-03-help-center-tickets.md)
- [AFC-04 · Settings and security](topics/affiliate-shared/AFC-04-settings-and-security.md)

**Affiliate: buyer site**

- [AFB-01 · Product page behind an affiliate link](topics/affiliate-buyer/AFB-01-product-page-behind-affiliate-link.md)
- [AFB-02 · Affiliate storefront](topics/affiliate-buyer/AFB-02-affiliate-storefront.md)
- [AFB-03 · Shopper account](topics/affiliate-buyer/AFB-03-shopper-account.md)
- [AFB-04 · Brand recruitment page and applying](topics/affiliate-buyer/AFB-04-brand-recruitment-page-and-applying.md)
