---
id: CMP-05
title: Product gifting (seeding) campaigns (brand)
domain: Campaigns (brand side)
audience: [brand, admin, system]
apps: [web, email, background]
related: [CMP-01, CMP-02, CMP-03, CMP-06, MKT-04, DSC-03, PUBL-01, SUB-04, CHAT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# CMP-05 · Product gifting (seeding) campaigns (brand)

> A gifting campaign ("Product Seeding program") sends free products to influencers. No fee is paid. The brand builds the program in a wizard, the Ainfluencer team approves it, and then influencers request a gift (or the brand invites them). The brand moves each influencer through a pipeline: accepted → shipped → received → creating content → completed.

## Who and where

- **Brand**, web app.
  - Wizard: `/affiliate/product_seeding_wizard` (new) or `/affiliate/product_seeding_wizard?edit=<id>` ("Edit" on a gifting card in `/campaigns`). Reached from the type picker card "Product Gifting & Seeding" (CMP-01).
  - Pipeline: `/campaigns/seeding-influencers/:id` ("Manage Influencer(s)" on an approved gifting card).
  - Inviting: from influencer search (`/search`).
- **Admin:** approves or rejects gifting campaigns in the admin panel (gifting campaigns list).
- **Influencer:** requests, accepts, confirms receipt and reports posts in the mobile app (MKT-04).

## How it works

### Building the program (wizard)

Steps: "Service" → "Products" → "Cadence" → "Recipients" → "Shipping" → "Review". Every step is saved as a draft when the brand continues ("Save & exit" leaves and keeps the draft).

1. **Service:** "Give your seeding program a title*" and "How would you like to run this program?": "Self-serve" (you approve creators and ship from your warehouse, "~15 minutes") or "Managed" ("Our team runs it for you", "1 brief call"). The choice is stored and shown on the campaign. The system doesn't run anything differently for "Managed"; the team handles it by hand.
2. **Products:** at least one product, from the brand's Shopify store or Amazon account, or entered by hand (name, retail price, image).
3. **Cadence:** "One-time" (with "Total units to ship *" and "Total program value") or "Monthly" ("Units per month *", optional "Monthly value cap", "Roll over unused units", "Pause when stock runs low", "Email me a monthly summary"). "Start date *" is required. "End date" is optional ("leave blank for ongoing").
4. **Recipients:** niches/categories (at least one), audience size, platforms (Instagram, TikTok, YouTube; at least one), languages (at least one), "limit one product per creator per quarter", and optional requirements (minimum engagement rate, minimum age 18+, disclosure, tagging/hashtags, competitor exclusivity, custom terms).
5. **Shipping:** "Ship from *" (one country; "must match your address on Shopify or Amazon"), "Ship to *" (countries; "Only creators in these countries will be matched"), "Brand covers all shipping costs", "Send tracking notifications".
6. **Review → "Launch program":** the campaign is submitted. This is where the **"product seeding access"** plan feature and the **campaign slot** are checked. If either fails, the upgrade pop-up opens and the program stays a draft (SUB-04). On success: "Program launched", and the status is **Pending** (waiting for review).

### Review

- An admin approves or rejects the submitted program. The brand gets a notification either way. A rejection includes the reason.
- A draft can't be approved or rejected; the brand must launch it first.
- When approved, the program shows to influencers in the mobile Gifting tab and they can request the gift (MKT-04). If the brand is over its campaign limit at approval, the program is paused right away (same rule as CMP-02).

### Running the program (pipeline)

The pipeline is a board with one column per stage:

| Column | Internal status | How influencers get there |
|---|---|---|
| "Requested by Influencer" | `requested` | Influencer asked for the gift in the app |
| "Pending Acceptance" | `pending_accept` | Brand invited the influencer, waiting for their answer |
| "Accepted - Awaiting Ship" | `accepted` | Brand accepted a request, or the influencer accepted an invitation |
| "Shipped" | `shipped` | Brand moved the card here |
| "Received" | `received` | Influencer confirmed receipt in the app (or brand moved it) |
| "Creating content" | `creating_content` | Brand moved the card |
| "Completed" | `published` | Influencer told the brand about a post, or brand moved it |
| "Rejected" | `rejected` | Brand rejected a request, or the influencer declined an invitation |
| "Archived" | `archived` | Brand archived the card |

- A **pending request** (either kind) can only be accepted or rejected, not dragged to another stage.
- After acceptance, the brand can move a card **forward or backward** between Accepted, Shipped, Received, Creating content and Completed, or archive it.
- **Rejected is final:** a rejected influencer can't be moved again.
- The brand sees the influencer's shipping address once the request is accepted.
- The pipeline also offers an **outreach message** (templates like "Warm & friendly", "Short & sweet"). It's sent through the **Ainfluencer chat** (CHAT-01), linked to the gifting campaign.
- **Export** downloads a CSV of everyone in the campaign except open requests.
- **Inviting influencers:** from `/search`, the brand picks a gifting campaign. Invitations can only be sent for **approved** campaigns. Influencers already in the campaign are skipped. Invitees also get the grouped invitation email (DSC-03).

### Shipping

There's no shipping-label or tracking integration for gifting. The brand ships the product itself and moves the card to "Shipped". The "Send tracking notifications" switch is stored but not connected to any tracking provider today. Shopify fulfilment and Track123 tracking apply to paid-collaboration offers only (CMP-06).

## Rules and limits

- **Plan:** launching needs the "product seeding access" feature (the review screen says "Included with Pro & Ultimate") and a free campaign slot. A gifting campaign in review already uses a slot.
- **Address freeze:** an influencer can change the shipping address only while their status is Requested, Pending Acceptance or Accepted. Once the card reaches Shipped, the address on the deal is frozen as a record of where the parcel went. A wrong address after that is fixed in chat with the brand.
- **Requests** are open only while the campaign is approved. Influencers must live in a "Ship to" country (MKT-04).
- **Unit caps are not hard limits:** "Total units to ship" and "Units per month" are used for display and ordering ("spots left"). The system doesn't stop requests when the number is reached *(based on code search)*. The same goes for "Pause when stock runs low", "Roll over unused units" and "Email me a monthly summary": stored, but no automation uses them.
- **Editing a live program:** changing the **products** of an approved program sends it back to **Pending** (re-review). Moving the end date of an **expired** program into the future (or clearing it) also sends it back to Pending. Other edits keep the status.
- **Pausing:** the brand has **no pause switch** for gifting campaigns on `/campaigns`. Gifting campaigns can be paused by an admin or by the plan check (`paused` status); parcels already agreed keep moving.
- **Deleting** ("Delete" on the card) removes the campaign for everyone, with no check for gifts in progress.

## Statuses

Campaign (`product_seedings.status`):

| Chip | Meaning | Next |
|---|---|---|
| "Draft" | Wizard not launched | Brand launches |
| "Pending" | Launched, waiting for admin | Approve or reject |
| "Approved" | Live; influencers can request | Expires on end date |
| "Rejected" | Admin rejected (reason shown) | Brand edits and relaunches |
| "Paused" | Stopped by admin or plan check | Admin/plan resumes |
| "Expired" | End date passed | Extend end date → Pending |

Influencer statuses: see the pipeline table above.

## Timers and automatic actions

| Job | When (UTC) | Effect |
|---|---|---|
| `product-seedings:expiry-check` | hourly | Approved programs whose end date is before today become "Expired". Programs without an end date never expire. |
| Plan check at approval | on approval | Over-quota program paused at once |
| `plans:detect-downgrades` | hourly | After a plan ends, running campaigns over the limit are paused (SUB-02) |

Gifting campaigns don't get the 24 h / 72 h draft reminder emails.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Admin approves the program | Brand | In-app, push, email |
| Admin rejects the program (with reason) | Brand | In-app, push, email |
| Influencer requests a gift | Brand | In-app, push, email |
| Brand invites influencer (`pending_accept`) | Influencer | In-app, push, email (+ grouped invitation email) |
| Brand accepts the influencer | Influencer | In-app, push, email |
| Brand marks shipped | Influencer | In-app, push, email |
| Influencer confirms received | Brand | In-app, push, email |
| Influencer reports the post (completed) | Brand | In-app, push, email |

## Common questions

### "I click 'Launch program' and get an upgrade pop-up."
**Answer for the user:** Gifting programs are part of the Pro and Ultimate plans, and each program uses one of your campaign slots. Upgrade, or free a slot by pausing or deleting another campaign.
**What to check:** the plan's product seeding access and the campaign limit vs. held campaigns (SUB-04).
**Escalate when:** the brand is on Pro/Ultimate with free slots and still blocked.

### "I can't invite influencers to my gifting campaign."
**Answer for the user:** Invitations can be sent once the program is approved by our team.
**What to check:** campaign status is Approved (not Pending, Paused or Expired). An error saying the influencers aren't active means archived accounts were selected.
**Escalate when:** the campaign is approved and invites still fail.

### "The influencer gave the wrong address but I already marked it shipped."
**Answer for the user:** Once a gift is marked shipped, the address is locked as the delivery record. Please sort out the delivery with the influencer in the Ainfluencer chat.
**What to check:** card status (Shipped or later = locked). If the brand hasn't really shipped yet, it can move the card back to "Accepted - Awaiting Ship", and the influencer can then change the address.
**Escalate when:** n/a.

### "How do I add a tracking number for a gift?"
**Answer for the user:** Gifting doesn't have built-in tracking yet. Please share the tracking number with the influencer in the chat.
**What to check:** nothing.
**Escalate when:** n/a (feature gap).

### "I changed the products and my program went back to Pending."
**Answer for the user:** Changing the products of a live program needs a new review, so influencers always see what you'll really send. It goes live again after approval.
**What to check:** status Pending.
**Escalate when:** the brand needs a quick re-review.

### "How do I pause my gifting program?"
**Answer for the user:** There's no pause button for gifting programs right now. You can set an end date in "Edit" (Cadence), or ask us to pause it for you.
**What to check:** n/a.
**Escalate when:** the brand asks for a pause (admin can pause it).

### "Influencers can't join from the link I shared."
**Answer for the user:** We're aware of a problem with the join button on the gifting preview page. Influencers can find and request your gift in the Gifting tab of the Ainfluencer app.
**What to check:** see Known issues and PUBL-01.
**Escalate when:** n/a (known bug).

## Known issues

- **Broken "Join" link on the public gifting preview.** On `/view-campaign-seeding/:id` the join button goes to `/seeding/invitation`, which doesn't exist (logged-out users land on the login page, logged-in users on a 404). The working page is `/gifting/invitation`. Also, "Copy campaign url" on `/campaigns` copies the generic `/view-campaign/:id` link for gifting campaigns. Escalate to dev.
- **Swapped tooltips in the pipeline.** "Pending Acceptance" says "The creator has seen your request and is deciding" (correct). "Requested by Influencer" says "You've asked this creator if they'd like a free product", which is wrong: that column holds influencers who asked the brand.
- **"Requesting fees" column is always empty.** The board shows it, but there's no such status in the system.
- **Unit caps and the "pause when stock runs low", roll-over and monthly-summary options don't do anything automatically** (see Rules).
- **No brand pause switch** although the review screen says "You can pause or end the program anytime".
- **Deleting a program with gifts in transit** is allowed and removes it for the influencers too *(to verify impact on the influencer side)*.

## Related topics

- CMP-01: the type picker. CMP-02: the same over-quota-at-approval rule. CMP-03: the Campaigns page.
- MKT-04: the influencer side (requesting, accepting, confirming receipt).
- CMP-06: Shopify shipping and tracking (paid offers only).
- PUBL-01: public preview. DSC-03: invitation emails. SUB-04: plan limits. CHAT-01: messaging.

## Technical reference

- **Endpoints (brand):**
  - `POST product-seedings/handle` (`ProductSeeding/Http/Controllers/ProductSeedingController@handle` → `Services/ProductSeedingService::handle`; middleware `feature:product_seeding_access` only at `current_step=finish`; `CampaignSlotGate` when status pending/approved). Steps: `services`, `products`, `cadence`, `recipients`, `shipping`, `finish`.
  - `GET product-seedings/list`, `GET product-seedings/{id}`, `DELETE product-seedings/{id}` (soft delete of seeding and its ad, `deleted_by`).
  - Pipeline: `GET product-seedings/{id}/influencers`, `GET …/influencers/export` (CSV), `POST …/influencers` (`sendRequestByBrand`, approved only), `PUT …/influencers/{influencerId}/status` (`changeStatusByBrand`, rules in `Services/ProductSeedingInfluencerStatusPolicy`).
  - Outreach: `sendChatboxMessage` with `subject_type=product_seeding`.
- **Endpoints (admin):** `GET/PUT /api/v2/admin/campaigns-seeding/{id}` (`AdminProductSeedingController` → `updateForAdmin`; statuses draft/pending/approved/rejected/expired; draft can't be moderated) → `CampaignModerated` event → `Listeners/NotifyBrandOfCampaignModeration` (`GiftingCampaignApprovedNotification` / `GiftingCampaignRejectedNotification`) and `PlanEnforcement/Listeners/PauseApprovedGiftingCampaignOverQuota`.
- **Notifications:** `ParticipationStatusChanged` → `Listeners/NotifyParticipantsOfStatusChange` with `Notifications/ParticipationNotificationRules` (invited/accepted/shipped → influencer; requested/received/published → brand).
- **Data:** `product_seedings` (`status` draft/pending/approved/rejected/paused/expired, `seed_type` self_serve/managed, `current_step`, `settings` JSON (cadence, recipients, shipping), projected columns `end_date`, `total_units_to_ship`, `units_per_month`, `ship_from`, `ad_id`, `rejection_reason`, `deleted_by`); `product_seeding_influencers` (`status`, `requested_by` brand/influencer, `address_id`, `address_snapshot` frozen at dispatch); `product_seeding_influencer_status_logs`. The linked `ads` row (`campaign_type=product_seeding`) mirrors the status (`resolveAdStatus`: approved→active/approved, paused→paused/approved, pending→draft/pending, rejected→draft/rejected, expired→expired/approved).
- **Jobs:** `Console/Commands/ProductSeedingExpiryCheck.php` (hourly) → `ProductSeedingService::expireEnded`. `GiftingRequestsDigestCommand` exists but isn't scheduled.
- **Key files:** `core/backend/api/app/Modules/ProductSeeding/*`, `core/backend/api/app/Modules/ProductSeeding/Services/SeedingPauseMirror.php`, `core/frontend/web/src/components/pages/affiliate/parts/productSeedingWizard/*`, `core/frontend/web/src/components/pages/seedingInfluencers/*`.
