---
id: MKT-04
title: Joining gifting campaigns (influencer)
domain: Campaign marketplace (influencer side)
audience: [influencer]
apps: [mobile, email]
related: [CMP-05, CMP-06, INF-05, MKT-01, MKT-02, CHAT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# MKT-04 · Joining gifting campaigns (influencer)

> In a Product Gifting (seeding) campaign the brand sends a free product and the influencer posts about it; no money changes hands. The influencer requests the gift (or accepts the brand's invitation), picks a shipping address, confirms when the parcel arrives and then shares the links to their posts.

## Who and where

- **Influencer**, mobile app:
  - Explore → **Gifting** tab with **"My Gifting Pipeline"** (boxes such as "Needs You", "Requested", "In Transit") and campaign lists ("New This Week", "Almost Full", "Free Gift");
  - All tab → Gifting section → "Show all" (Gifting campaigns list); brand profile → "Gifting Campaigns"; invitations (MKT-02); email links `gifting/invitation?ad_id=…&email=…`; chat offer cards.
- **GIFTING_DETAIL** screen: "ABOUT THIS CAMPAIGN", "THE DEAL" ("You get"), "Eligibility", "Campaign Brief", "Message", and one main button that changes with the stage.

## How it works

1. **Request a gift** (open campaigns): **"Send Gift Request"** → pick or add a shipping address, optional note (up to 255 characters) → "Save and send request" → "Gift request sent." Status: requested ("The brand is reviewing your request").
2. **Brand invitation**: the screen shows "INVITATION" with **"Accept Invite"** (pick address, optional note → "Save and accept" → "Invite accepted.") or **"Reject invite"** → "Reject".
3. **Brand accepts and ships** (CMP-05/CMP-06): status moves to accepted, then shipped ("In Transit"). Tracking updates may arrive by push/email.
4. **Confirm Received**: when the parcel arrives, tap **"Confirm Received"**. This is the only status the influencer sets.
5. **Tell the brand I posted**: tick every post condition, paste the post URL(s), send. The collaboration becomes **published** and the brand is notified. More links can be added later; a link can be withdrawn ("Withdraw post") but at least one must stay ("You need to keep at least one posted link.").
6. **Change address**: allowed until the gift ships ("Save address" → "Shipping address updated."). The address can also be shared from the chat's address pin.
7. **Message brand** opens the Ainfluencer chat thread for this gift (CHAT-01).

## Rules and limits

- Requests only for campaigns that are open (approved): "This campaign is not open for requests." One participation per influencer per campaign: "The influencer is already part of this campaign."
- **An address must be chosen** to request or accept: "Choose the address this gift should be shipped to." Address book: INF-05.
- The app disables "Send Gift Request" with **"This brand does not ship to <country>"** when the address country isn't in the brand's ship-to list (checked in the app).
- Accepting an invitation is blocked if the campaign is no longer open: "This campaign is no longer accepting influencers." Declining is always allowed.
- **Address changes only while requested, invited (waiting for you) or accepted.** After shipping: "The shipping address can no longer be changed for this gift." — fix it with the brand in chat.
- "Confirm Received" only from shipped; confirming is allowed even if the campaign has since expired.
- Post links can be shared only after the gift is received.
- No payment is involved; gifting does not use the wallet.

## Statuses

| Influencer sees | Internal (`product_seeding_influencers.status`) | Next |
|---|---|---|
| Requested ("The brand is reviewing your request") | `requested` | Brand accepts or rejects |
| Invited / "Needs You" | `pending_accept` | Accept or reject |
| Accepted | `accepted` | Brand ships |
| In Transit | `shipped` | Confirm Received |
| Received | `received` | Share posts |
| Creating content | `creating_content` (set by brand) | Share posts |
| Published & Completed | `published` | Done |
| Rejected / archived | `rejected`, `archived` | Final |

## Timers and automatic actions

- Gifting campaigns past their end date expire (hourly check); open requests can no longer be accepted.
- Shipment tracking updates come from the tracking provider (CMP-06).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Brand invites to a gifting campaign | Influencer | Email, push, in-app |
| Brand accepts request | Influencer | Email, push, in-app |
| Gift shipped / tracking added | Influencer | Email, push, in-app |
| Brand asks for an address (Shopify) | Influencer | Push, email |
| Delivered (tracking) | Influencer and brand | Push, email |
| Request received / product received / post shared | Brand | Email, push, in-app (CMP-05) |

## Common questions

### "I can't send a gift request."
**Answer for the user:** Choose a shipping address first. If the button says the brand doesn't ship to your country, this brand can't send to that address. The campaign may also be full or closed.

### "My address is wrong and the gift is already shipped."
**Answer for the user:** Once a gift ships the address is locked. Please message the brand in the Ainfluencer chat right away so they can correct it with the carrier.

### "I got the product. What now?"
**Answer for the user:** Tap "Confirm Received", post the content as agreed, then tap "Tell the brand I posted" and paste your post link(s).

### "I can't edit the post link I shared."
**Answer for the user:** Editing a shared link doesn't work at the moment. Add the correct link as a new one, then withdraw the wrong one (one link must always remain).
**Escalate when:** only one wrong link exists and the user can't replace it.

### "The invite says it's no longer available."
**Answer for the user:** The brand's campaign has closed or is being edited. You can still decline it; look for other gifting campaigns in Explore.

### "When will the brand ship?"
**Answer for the user:** Shipping is done by the brand. Ask them in the Ainfluencer chat; tracking will appear once they add it.

## Known issues

- **Editing a shared post URL fails** (the app uses an action the server doesn't support). Workaround: add a new link and withdraw the old one.
- **Sending an address when the app only knows the campaign** (not the request) fails. Workaround: open the gift from the Gifting pipeline/detail screen and change the address there.
- Public seeding preview's "Join" button goes to a missing page (CMP-05).

## Related topics

- CMP-05 brand side of gifting, CMP-06 shipping and tracking, INF-05 address book, MKT-02 invitations, CHAT-01 chat.

## Technical reference

- Endpoints (`api/{locale}/v3/mobile`, `Modules/Mobile/Http/Controllers/API/ProductSeedingController`): `GET product-seedings` (feed + pipeline summary), `GET product-seedings/{id}`, `GET product-seedings/requests`, `POST product-seedings/{id}/apply` `{address_id, note}`, `PUT product-seedings/requests/{id}/respond` `{action, address_id, note}` (campaign-scoped `respond` is deprecated), `PUT product-seedings/requests/{id}` `{address_id}`, `POST …/requests/{id}/send-address`, `PUT …/requests/{id}/status` `{status: received}`, `POST …/requests/{id}/posts` `{urls}`, `DELETE …/requests/{id}/posts/{postId}`.
- Services: `Modules/ProductSeeding/Services/ProductSeedingInfluencerService` (`sendRequestByInfluencer`, `respondToRequest`, `changeStatusByInfluencer`, `resolveAddressChange`, `resolveShippingAddress`), `ProductSeedingPostService` (posts → `published`), `Modules/Publishing/Services/PublishedPostService`; statuses and rules `Modules/ProductSeeding/Helpers/Enums/ProductSeedingInfluencerStatus.php` (`addressChangeableStatuses` = requested, pending_accept, accepted; `postableStatuses` = received, creating_content, published); transition policy (`assertInfluencerCanChange`).
- Broken client calls: `PUT …/requests/{id}/posts/{postId}` (route only has DELETE), `POST product-seedings/{id}/send-address` (no route).
- Job: `product-seedings:expiry-check` (hourly).
- Notifications: `Modules/ProductSeeding` participation rules (`CampaignInvited`, `CampaignAccepted`, `CampaignShipped`; brand `GiftingRequestReceived`, `ProductReceived`, `CampaignPublished`), Shopify `NeedInfluencerAddress`, `ImportTrackingInfo`, `FulfillmentDelivered`.
- Data: `product_seedings` (status, end_date), `product_seeding_influencers` (status, address_id, address_snapshot, identity_id), status logs, published posts.
- Mobile: `features/explore/screens/gifting/{GiftingScreen,GiftingCampaignsScreen,GiftingDetailScreen}.tsx`, `network/endpoints.js` `GIFTING_APIS`.
