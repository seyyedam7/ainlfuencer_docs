---
id: CMP-06
title: Shipping and tracking for gifts and Shopify orders
domain: Campaigns (brand side)
audience: [brand, influencer, system]
apps: [web, mobile, email, background]
related: [CMP-01, CMP-05, INT-01, OFR-03, PUB-01, CHAT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# CMP-06 · Shipping and tracking for gifts and Shopify orders

> When a paid-collaboration campaign is linked to a Shopify product, the brand can send that product to the influencer straight from the inbox. Ainfluencer creates a free (100% discounted) order in the brand's Shopify store, marks it fulfilled, and then follows the parcel through the tracking provider **Track123** until it's delivered. This works for **paid offers on Shopify campaigns only**. Gifting (seeding) campaigns have no shipping integration (CMP-05).

## Who and where

- **Brand**, web app, inbox: `/new-offersV2/:id` → right sidebar box "Product Tracking". The box appears only when:
  - the campaign is a Shopify campaign (created with "I am a Shopify user", CMP-01 / INT-01), **and**
  - the offer has been paid (OFR-03).
- **Influencer:** gets notifications in the mobile app and by email, and sees a message in the chat. The influencer can't enter an address for this flow in the app (see Known issues).
- **System:** Track123 sends delivery updates to Ainfluencer.

## How it works

1. In the "Product Tracking" box the brand clicks "Send via Shopify" ("Send Product to Influencer"). If the Shopify store isn't connected (or the connection expired), the Shopify connection window opens first (INT-01).
2. "Select Product(s) to Send to the Influencer": the brand picks one or more products from its Shopify store. One unit of each product is sent.
3. "Confirm Shipping Address", with three options:
   - "Use influencer's default address": the address the influencer saved in the app;
   - "Enter a different address": first name, last name, address, country, province, city, zip code and mobile phone, all required;
   - "Request influencer to provice shipping address" (sic).
4. **With an address**, Ainfluencer immediately:
   - creates a draft order in the brand's Shopify store with a 100% discount ("Free Order For Influencer");
   - completes it into a real order;
   - creates the fulfilment.

   Nothing is charged to anyone, and the brand ships the item as usual.
5. **Without an address** (request option), the influencer gets a chat message: "The brand wants to send you product(s) and needs your address. Submit your address to the brand as soon as possible.", plus a push/email. The box shows "wait for influencer to enter address".
6. **Add the tracking number.** After shipping, the brand picks the carrier ("Tracking Company", list provided by Track123) and enters the "Tracking number...", then clicks "Submit & Track". Ainfluencer:
   - adds the tracking to the Shopify fulfilment;
   - registers the number with Track123;
   - notifies the influencer.

   Toast: "Tracking number added successfully". Until the first update arrives, the box says "Waiting for tracking system. Use refresh button to check for updates." with a "Refresh" button.
7. **Tracking updates:** Track123 sends status changes (in transit, out for delivery, delivered and so on). The box shows "Product(s)", "Carrier", "Tracking", "Status", "Last update", "Shipped on" and "Current Location". The brand can edit the tracking number until the parcel is delivered (pencil icon).
8. **Delivered:** brand and influencer both get a "delivered" notification.

## Rules and limits

- Shopify campaigns only, after payment. A normal (non-Shopify) campaign has no "Product Tracking" box. The brand shares tracking in the chat (CHAT-01).
- The brand's Shopify connection must be valid. If Shopify refuses (app uninstalled, store closed), the action fails with an error (INT-01).
- Quantity is always 1 per selected product.
- The tracking number can only be added after the Shopify order/fulfilment exists. If the brand used the "request address" option, that never happens (see Known issues).
- Editing the tracking number updates Ainfluencer and Track123, but not the tracking already saved on the Shopify fulfilment *(to verify)*.
- **AfterShip is no longer used.** Its webhook still exists but nothing registers parcels with it. Track123 is the live provider.

## Statuses

Shipment status shown in the box (from Track123): `init`, `pending`, `info_received`, `in_transit`, `waiting_delivery`, `out_for_delivery`, `failed_attempt`, `delivery_failed`, `exception`, `abnormal`, `expired`, `no_record`, `no_information_available`, `received`, `delivered`. Anything else shows as `unknown`.

Internal states of the fulfilment record: `pending:influencer` = waiting for the influencer's address; empty status = order created, waiting for tracking updates.

## Timers and automatic actions

- No scheduled jobs. Updates arrive whenever Track123 calls Ainfluencer.
- "Refresh" in the box re-reads the latest stored status. It doesn't ask Track123 for news.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Brand asks for the address | Influencer | Chat message + push + email |
| Brand adds the tracking number | Influencer | Push + email |
| Track123 reports "delivered" | Influencer | Push + email |
| Track123 reports "delivered" | Brand | Push + email |

## Common questions

### "I don't see the 'Send via Shopify' option in the inbox."
**Answer for the user:** Shipping through Shopify is available for campaigns created from a Shopify product, once the offer is paid.
**What to check:** the campaign's store type (`ads.third_party = shopify`) and that the offer is paid.
**Escalate when:** both are true and the box is missing.

### "I asked the influencer for their address and nothing happens."
**Answer for the user:** Please ask the influencer for their address in the chat, then send the product again and choose "Enter a different address".
**What to check:** fulfilment status `pending:influencer`.
**Escalate when:** always report it as a known gap (influencers can't submit the address in the app).

### "Sending the product failed with an error."
**Answer for the user:** We couldn't reach your Shopify store. Please reconnect it and try again.
**What to check:** the Shopify connection (INT-01): was the app uninstalled, did the token expire, is the store closed, are the products still in stock or active?
**Escalate when:** the store is connected and the error repeats.

### "The tracking status hasn't changed for days."
**Answer for the user:** Status comes from the carrier through our tracking partner. Some carriers update only at major checkpoints. Please check the carrier's website with the tracking number.
**What to check:** the carrier and number entered. A wrong carrier often gives `no_record`, and the brand can edit it.
**Escalate when:** the carrier site shows progress but Ainfluencer stays on the same status for more than a day.

### "The influencer says they didn't get a tracking email."
**Answer for the user:** The tracking number was sent by push notification and email. It's also visible to you in the "Product Tracking" box, so you can share it in the chat.
**What to check:** the influencer's email notification settings.
**Escalate when:** n/a.

### "Can I track gifts from my gifting campaign?"
**Answer for the user:** Not yet. For gifting programs, please share the tracking number with the influencer in the chat.
**What to check:** n/a (CMP-05).
**Escalate when:** n/a.

## Known issues

- **"Request influencer to provide shipping address" leads nowhere.** The influencer gets the chat message, but the mobile app has no working way to submit the address for this flow (the address bubble is switched off and the endpoint isn't called). The order is never created and the box keeps showing "wait for influencer to enter address". Workaround: get the address in chat and use "Enter a different address". Escalate to dev for stuck records.
- **Brands with several Shopify stores** *(to verify)*: sending and tracking use the brand's Shopify connection found for its account, which may not be the store of the campaign's product.
- Typo in the UI: "provice" instead of "provide".

## Related topics

- INT-01: connecting the Shopify store. CMP-01: creating a Shopify campaign.
- OFR-03: paying the offer (makes the box appear). PUB-01: content delivery after the product arrives.
- CMP-05: gifting programs (manual shipping). CHAT-01: messaging the influencer.

## Technical reference

- **Endpoints (web):**
  - `POST user-third-party/shopify/v1/fulfill-to-influencer` (`ShopifyFulfillmentController@sendToInfluencer` → `NotificationShopifyFulfillmentServiceDecorator::createFulfillment` → `ShopifyFulfillmentService::createFulfillment`: draft order 100% off → complete → fulfillment order → fulfillment; without an address, status `pending:influencer` plus chat message `offer:fulfillment_complete_address_by_influencer` and `NeedInfluencerAddressNotification`).
  - `GET …/has-fulfillment/{offer_id}`.
  - `POST …/add-tracking-info` (`importTrackingInfo` → `completeFulfillment`: needs `fulfillment_id`; updates Shopify tracking, `TrackingService::createTracking` (Track123), `ImportTrackingInfoNotification`).
  - `POST …/edit-tracking-info` (`TrackingServiceController@edit` → `TrackingService::editTracking`), `GET …/carriers?provider=track123`.
- **Not used by any app:** `POST …/complete-fulfill` (influencer address, `AddAddressByInfluencerNotification`); `POST user-third-party/aftership/webhook`.
- **Webhook:** `POST user-third-party/track123/webhook` (`Track123WebhookController`: maps `transitStatus` → `shopify_fulfillments.status`, stores `checkpoints`, `shipment_pickup_date`; on `delivered`, `FulfillmentDeliveredNotification` to the influencer and `FulfillmentDeliveredBrandNotification` to the brand).
- **Data:** `shopify_fulfillments` (`offer_id`, `shop_domain`, `products`, `address`, `draft_order_id`, `order_id`, `fulfillment_order_id`, `fulfillment_id`, `carrier`, `tracking_number`, `status`, `checkpoints`, `shipment_pickup_date`); `ads.third_party`.
- **Key files:** `core/backend/api/app/Modules/UserThirdParty/Services/ShopifyFulfillmentService.php`, `…/NotificationShopifyFulfillmentServiceDecorator.php`, `…/TrackingService.php`, `…/Driver/Track123Driver.php`; `core/frontend/web/src/components/pages/newSingleOfferV2/parts/newSidebar/parts/shopifyFulfillmentBox/*`, `core/frontend/web/src/components/common/modals/shopifyFullfillment/*`; mobile `src/features/chat/components/MessageBubble.tsx` (`showFullfillmentRequest = false`).
