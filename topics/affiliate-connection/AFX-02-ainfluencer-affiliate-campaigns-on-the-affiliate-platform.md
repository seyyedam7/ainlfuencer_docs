---
id: AFX-02
title: Ainfluencer affiliate campaigns on the affiliate platform
domain: Ainfluencer ↔ affiliate platform
audience: [brand, influencer, seller, affiliate, system]
apps: [web, mobile, background]
related: [CMP-04, MKT-03, AFX-01, AFX-03, AFS-03, AFS-06, AFS-10, AFA-04, AFA-08, CHAT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, affiliate/ainflu-affiliate-backend@v1.5.27
---

# AFX-02 · Ainfluencer affiliate campaigns on the affiliate platform

> An "affiliate campaign" created in the Ainfluencer web app is really stored and run on the affiliate platform. This topic explains what happens there: how the campaign is created, how an influencer joining it becomes an approved partnership with tracked links, how stats come back to Ainfluencer, and what the platform does on its own (ending, revoking, pausing). The brand screens are in CMP-04 and the influencer screens in MKT-03.

## Who and where

- **Brand**, in the Ainfluencer web app (`/affiliate/*`, "Campaigns"). The brand never has to open the seller app, but the same campaign data is behind the seller app (AFX-01).
- **Influencer**, in the mobile app (affiliate campaign detail, "generate link", affiliate analytics). Behind the scenes they are an affiliate on the platform.
- **System**: the affiliate platform's scheduled jobs and its "campaign updated" messages to Ainfluencer.

## How it works

1. **Prerequisites.** The brand has a seller account on the platform (created automatically, AFX-01), a connected Amazon or Shopify store (AFX-03) and a **card** saved on the platform (AFS-10).
2. **Creating.** When the brand finishes the campaign form, Ainfluencer first asks the platform whether a card exists. Without one, nothing is created and the brand sees "Please connect a payment method before creating an affiliate campaign." With a card, Ainfluencer saves its own copy and asks the platform to create the campaign. It is created **active** on the platform, linked to the Ainfluencer campaign id.
3. **Products and commission.** The brand picks products and a commission per product. Ainfluencer sends them to the platform ("assign items"). Commission changes made from Ainfluencer update the product on the platform exactly like a change in the seller app, so they also change existing contracts for that product (AFS-03).
4. **Influencer joins.** When an influencer joins from the mobile app, the platform:
   - creates a partnership request from the affiliate for every campaign product, with the campaign's commission (or a custom one if the brand set it for that influencer);
   - **approves it immediately** (no seller action, no auto-approval setting needed);
   - creates the partnership contract and the affiliate's **tracked product links**, and returns the links to the app.
   From now on sales through those links earn commission like any other contract (AFS-06, AFA-04).
5. **Stats back to Ainfluencer.** After every change (create, products, joins, commission or product status changes) and again **every hour**, the platform sends Ainfluencer the campaign's product count, average commission, number of affiliates, sales amount, currency and country. Ainfluencer shows them on the campaign cards and "Performance".
6. **Money.** Commission is charged to the brand's card about 35 days after the sale and paid to the influencer through Stripe Connect, minus the platform fee (AFS-10, AFA-08). The Ainfluencer wallet isn't used.

## Rules and limits

- **A card is required to create** a campaign (checked by Ainfluencer before saving and again by the platform, which answers HTTP 499 without one).
- **Joining always auto-approves**, regardless of the brand's "Automatic Partnership Approval" setting.
- The platform does **not** check the campaign's status when someone joins: whether a paused or finished campaign can be joined depends on what the Ainfluencer apps allow.
- Plan limits for commission percentage and status changes come from the seller plan (AFS-12); a blocked change returns "limit reached" (HTTP 402).
- Product deactivation, a store going inactive (AFX-03) or deleting the store affects the campaign's products the same way as everywhere else.

## Statuses

| Platform status (`campaigns.status`) | Meaning | What happens next |
|---|---|---|
| `active` | Running (status at creation) | Becomes `finished` after the end date |
| `finished` | End date passed | All its active contracts are revoked at the next 02:00 UTC run; links stop earning |
| `archived` | Set when the brand pauses from the Ainfluencer affiliate screens (CMP-04) | – |
| `cancelled`, `draft`, `pending` | Possible values set by Ainfluencer/admin updates | – |
| `is_paused = true`, `paused_by = system` | Seller had no card at the 02:30 check | Never un-paused automatically; has no other effect inside the platform |

A finished campaign becomes `active` again if its end date is moved into the future, but contracts already revoked stay revoked.

Per-affiliate status inside a campaign (shown in campaign affiliate lists) is computed: active, inactive, revoked, pending, approved, rejected, canceled, assigned.

## Timers and automatic actions

| Job | When (UTC) | Effect | User notices |
|---|---|---|---|
| `ExpireCampaignsCommand` | hourly | Active campaigns past `end_date` → `finished`; stats sent to Ainfluencer | Campaign ends |
| `RevokePartnershipContractsForEndedCampaignsCommand` | daily 02:00 | Every active contract of finished campaigns → revoked; links revoked; private promotion access removed | Influencer's links stop earning; shopper sees the affiliate's "product removed" behaviour |
| `campaign:pause-without-payment-method` | daily 02:30 | Active campaigns of sellers without a card → paused flag | Nothing in Ainfluencer (it isn't told) |
| `SyncCampaignsWebhookDataCommand` | hourly | Re-sends stats for every active or pending campaign that came from Ainfluencer | Stats refresh |

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Contract revoked by the ended-campaign job | – | No notification is sent for this path |
| Product commission or status changed | Partnered affiliates | Affiliate web notification (+ email digest), see AFS-03 |
| Commission paid | Seller and affiliate | Web notification only (AFS-10) |

## Common questions

### "My campaign ended and influencers say their links stopped working."
**Answer for the user:** When an affiliate campaign reaches its end date it finishes, and that night the influencers' partnerships for it are closed, so their links stop earning. To keep working with them, create a new campaign or invite them to your products from the seller app.
**What to check:** platform `campaigns.status` = `finished` and `end_date`; contracts with `revoked_at` around 02:00 UTC the next day.
**Escalate when:** contracts were revoked while the campaign is still `active` with a future end date.

### "I extended the end date but the influencers' links are still dead."
**Answer for the user:** Extending the date restarts the campaign, but partnerships that were already closed don't reopen. Influencers need to join again or be re-invited.
**What to check:** campaign is `active` again; the affiliate's contracts are `revoked`.
**Escalate when:** the user needs bulk restoration (dev).

### "Influencer says they joined but the brand didn't approve anything."
**Answer for the user:** Joining an Ainfluencer affiliate campaign approves the partnership automatically and creates the links right away. No approval step is needed.
**What to check:** a partnership request with message "ainfluencer campaign integration" and status `approved`, and a contract for that affiliate.
**Escalate when:** the join returned an error or no links were created.

### "The affiliate campaign stats in Ainfluencer look old."
**Answer for the user:** Stats are refreshed whenever something changes and at least once an hour. Sales numbers depend on Amazon/Shopify data, which can lag by a few days (AFS-09).
**What to check:** the campaign is `active`/`pending` on the platform (finished campaigns are not re-sent hourly); Ainfluencer `ads_affiliate` counters.
**Escalate when:** an active campaign hasn't updated for more than 2 hours.

### "My campaign still shows as live, but the platform says it's paused."
**Answer for the user:** The affiliate platform pauses campaigns when no card is on file. Please add a card in the Payouts tab (or Settings → Billing in the seller app).
**What to check:** `campaigns.is_paused`, `paused_by = system`; the seller's saved payment methods.
**Escalate when:** the card is present but the flag stays set and the user needs it cleared (no automatic un-pause; dev).

### "Can the brand chat with influencers in the campaign?"
**Answer for the user:** Yes, through the Ainfluencer chat. The affiliate platform's own messages aren't a supported channel.
**What to check:** n/a (CHAT-01).
**Escalate when:** n/a.

## Known issues

- **Pause is never lifted and Ainfluencer never hears about it.** The platform sends a "campaign paused" message that Ainfluencer does not accept, and nothing un-pauses the campaign after a card is added. The flag has no other effect on the platform today. Escalate to dev if a user is affected.
- **Ainfluencer is not told when a campaign finishes.** The stats message carries only counts and sales, not the status. *(to verify how core shows the end.)*
- **Revoked contracts don't come back** when the end date is extended. No workaround except re-joining/re-inviting.
- **No status check on join** on the platform side *(to verify against the mobile app)*: an influencer could join a finished or paused campaign; contracts of a finished one are revoked the next night.

## Related topics

- CMP-04 — Brand screens: creating, editing, pausing affiliate campaigns.
- MKT-03 — Influencer screens: joining, links, analytics, payouts.
- AFX-01 — How the brand and influencer accounts are linked.
- AFX-03 — Store connections that the campaign products come from.
- AFS-03 — Product commission and status (changes affect contracts).
- AFS-06 / AFA-04 — Partnerships and contracts in general.
- AFS-10 / AFA-08 — Card charges and payouts.
- CHAT-01 — Messaging between brands and influencers.

## Technical reference

- **Core:** campaign creation `app/Modules/AdCreationProcess/Services/AdCreationProcessAffiliateService.php` (card check `hasCard` → `AffiliatePaymentMethodRequiredException`, then `AinfluencerAffiliateService::createCampaign`); webhook receiver `POST api/user-third-party/ainfluencer-affiliate/v1/webhook` → `Listeners/AinfluencerAffiliateWebhookListener.php` (queue `ainfluencer-affiliate:webhooks`) → `webhookUpdateCampaign` (updates `ads_affiliate.products_count, average_commission, affiliates_count, sale_amount, sale_currency, country_code`). Accepted event keys: `Enums/AinfluencerAffiliateWebhookEventKey.php` (no `campaign.paused`).
- **Affiliate endpoints (called by core, API key):** `POST seller/v1/integrations/ainfluencer/campaigns/create` (`seller_stripe_connected` → 499), `…/campaigns/assign-items`, `…/campaigns/products/assign`, `GET …/campaigns/products/items`, `PUT …/campaigns/update/{id}`, `DELETE …/campaigns/delete/{id}`, `GET …/campaigns/{id}/affiliates/list`; `POST affiliate/v1/integrations/ainfluencer/campaigns/assign-affiliate` (`Services/Actions/Affiliate/Integrations/Ainfluencer/Campaign/AssignCampaignToAffiliateIntegrationService.php`: registers requests with `campaign_id`, approves, creates contracts and links); `GET affiliate/v1/integrations/ainfluencer/campaigns/list`, `…/campaigns/{id}/products`; products `PUT …/products/bulk-update-commission` (plan gate commission), `POST …/products/bulk-change-status` (gates brand, asin), `POST …/products/bulk-generate-link` (auto-approves); admin equivalents under `admin/v1/integrations/ainfluencer/campaigns/*`.
- **Data (affiliate):** `campaigns` (`status`, `source_campaign_id`, `start_date`, `end_date`, `is_paused`, `paused_by`, `item_type` product_amazon/product_shopify), `campaign_items`, campaign item ↔ user type assignments, `partnership_requests.campaign_id`, `partnership_contracts` (`status`, `revoked_at`, `revoked_by`), `affiliate_product_links.status`, `webhook_logs` (delivery attempts).
- **Jobs:** `ExpireCampaignsCommand`/`ExpireCampaignJob`, `RevokePartnershipContractsForEndedCampaignsCommand`/`RevokePartnershipContractsForEndedCampaignJob`, `PauseCampaignsWithoutPaymentMethodCommand`/`PauseCampaignWithoutPaymentMethodJob`, `SyncCampaignsWebhookDataCommand`, all in `app/Console/Kernel.php` (production only, UTC). Webhook sender `Services/General/Webhook/SendWebhookService.php` (targets configured in the `webhooks` table; 3 retries).
