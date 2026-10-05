# Glossary

Terms used across the topic documents. Topic IDs in brackets point to where the term matters most.

## People and accounts

| Term | Meaning |
|---|---|
| **Brand** | A business account on Ainfluencer (web app). Creates campaigns, pays influencers. On the affiliate platform the same business is a **seller**. |
| **Influencer** | A creator account on Ainfluencer (mobile app). Connects social accounts, sends offers, delivers posts. On the affiliate platform the same person is an **affiliate**. |
| **Social account / integration** | One connected Instagram, TikTok or YouTube profile of an influencer. Offers, reviews and suspensions belong to a social account, not to the person. [INF-01, INF-05] |
| **Team member** | A person invited into a brand's account with limited access. Uses a **seat** from the brand's plan. [BRD-03, AFS-13] |
| **Admin** | Ainfluencer staff using the admin panel: approves campaigns and accounts, edits offers, pays payouts. |
| **Seller** | A brand/store on the affiliate platform (seller.ainfluencer.com). [AFS] |
| **Affiliate** | A creator on the affiliate platform (affiliate.ainfluencer.com). [AFA] |
| **Buyer / shopper** | Someone who clicks an affiliate link and lands on the public product page. Buys on Amazon or Shopify. [AFB] |
| **SSO** | Single sign-on: opening the affiliate platform from Ainfluencer without a separate login. Accounts are matched by email. [AFX-01] |

## Campaigns and offers

| Term | Meaning |
|---|---|
| **Campaign (ad)** | What a brand publishes to find influencers. Types: **paid collaboration**, **affiliate**, **product gifting (seeding)**. Live on the marketplace only when **active and approved**. [CMP] |
| **Campaign slot** | The number of campaigns a plan allows at the same time. [SUB-04] |
| **Invitation** | A brand asking a specific influencer to look at a campaign. Sent as a grouped email twice a day. [DSC-03, MKT-02] |
| **Offer** | An influencer's proposal for a paid campaign: price, delivery date, how long the post stays up. Statuses: pending, awaiting payment, paid, rejected, refunded. [OFR] |
| **Temporary offer** | An offer made while the influencer's account is under quality review; the brand sees it only after the account is activated. [INF-03] |
| **Counter-offer** | Changing the terms of an offer during negotiation (either side). [OFR-02] |
| **Revision** | A change the brand asks for after paying; if the price goes up, the brand pays the difference. [OFR-05] |
| **Deal** | A paid offer, from payment until settlement. |

## Delivery and money (Ainfluencer)

| Term | Meaning |
|---|---|
| **Publish job** | The delivery record created when an offer is paid: publish date ("start"), submitted link or files, approval state. [PUB-01] |
| **Approval** | The brand (or an admin) accepting the delivered post. **Money is released only after approval.** [PUB-02, PUB-03] |
| **Escrow / locked wallet** | Money the brand paid that Ainfluencer holds until approval or refund. Shown as "pending" or "on hold". [PUB-03] |
| **Release (finalize)** | Moving escrow money to the influencer's available wallet balance after approval. [PUB-03] |
| **Refund** | Money returned to the **brand's Ainfluencer wallet** (not the card). The platform fee isn't refunded. [PAY-02] |
| **Wallets** | Each user has a **default** wallet (available money), a **lock** wallet (escrow) and a **credit** wallet (promotional credit). A **legacy** wallet holds history from before August 2025. [PAY-01, PAY-03] |
| **Platform fee (brand)** | Added on top of the offer price at checkout, set by the brand's plan. [OFR-03] |
| **Influencer fee** | 20% taken when the influencer withdraws. [PAY-04] |
| **Payout / withdrawal** | The influencer asking to be paid out from the wallet (PayPal or crypto). **Paid by hand by an admin.** [PAY-04] |
| **Order / basket** | The checkout record for an offer payment. An unpaid order expires after 30 minutes. [OFR-03] |
| **Insurance** | Optional add-on at checkout. [OFR-03] |
| **Coupon / credit** | Discounts paid by Ainfluencer; their share goes back to Ainfluencer on refund. [PAY-05] |

## Plans

| Term | Meaning |
|---|---|
| **Plan** | Free, Starter, Pro, Ultimate (and custom) subscription that sets limits and fees. [SUB-01] |
| **Limit pop-up (402)** | The "upgrade" window shown when a plan limit is reached. [SUB-04, AFS-12] |
| **Grace period** | After cancelling, the plan stays active until the end of the paid period. [SUB-02] |
| **System pause** | Campaigns paused automatically because a plan ended or was downgraded. [SUB-02, CMP-03] |

## Affiliate platform

| Term | Meaning |
|---|---|
| **Partnership request** | A seller inviting an affiliate, or an affiliate applying to a seller's brand or products. Approved requests become **contracts**. [AFS-06, AFA-04] |
| **Contract** | An active partnership with a commission rate. When revoked, the affiliate's links stop earning. [AFS-06, AFA-04] |
| **Auto-approval** | A brand setting that approves affiliate applications instantly. [AFS-03] |
| **Commission** | The share of a sale paid to the affiliate (percentage by default; 15% unless the seller changes it). [AFS-03] |
| **Affiliate link** | A tracked link to a product; opens the product page on the buyer site, then Amazon or Shopify. [AFA-05, AFB-01] |
| **Storefront / landing page** | The affiliate's public page of promoted products. [AFA-06, AFB-02] |
| **Brand landing page** | A seller's public page that recruits affiliates. [AFS-04, AFB-04] |
| **Promotion** | A discount code a seller makes available to affiliates. [AFS-07] |
| **Commission billing** | The nightly charge of the seller's card for sales older than 35 days; the affiliate receives it minus the platform fee through Stripe Connect. [AFS-10, AFA-08] |
| **Stripe Connect** | The affiliate's payout account. Without it, affiliates aren't paid. [AFA-08] |
| **Associates** | Amazon Associates program run through Ainfluencer's tag, for affiliates given access. [AFA-07] |
| **Expanded Network** | Creators not registered on the platform whose contact details sellers can reveal. [AFS-05] |

## Chat

| Term | Meaning |
|---|---|
| **Ainfluencer chat** | The only supported messaging channel between brands/sellers and influencers/affiliates: offer chats plus affiliate and gifting threads. [CHAT-01] |
| **System message / offer card** | Automatic message in the chat about an offer event (paid, accepted, updated…). [CHAT-03] |
| **Affiliate platform chat** | The chat inside the seller and affiliate apps. **Not supported**; send users to the Ainfluencer chat. [AFC-01] |
