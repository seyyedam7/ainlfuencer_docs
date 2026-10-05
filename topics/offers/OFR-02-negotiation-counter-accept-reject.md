---
id: OFR-02
title: "Negotiation: counter, accept, reject"
domain: Offers and negotiation
audience: [brand, influencer, system]
apps: [web, mobile, email, background]
related: [OFR-01, OFR-03, OFR-04, OFR-05, OFR-06, CHAT-01, CHAT-03, INF-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# OFR-02 · Negotiation: counter, accept, reject

> After an influencer sends an offer, both sides can change its terms (price, delivery date, duration, bio link) until the brand pays. Each change goes to the other side for approval. Either side can also reject. The status label each person sees depends on **who changed the offer last**.

## Who and where
- **Brand**, web app: Inbox `/new-offersV2/:id`. The buttons sit in the chat header: **"Accept"**, **"Reject"**, **"Edit"**, **"Pay $&lt;price&gt;"** (shown depending on status), plus a **"More actions"** (⋮) menu. The older offers list `/offers` and the closed-deals grid `/new-offersV2/deals` still exist and link into the Inbox.
- **Influencer**, mobile app: the offer card at the top of the deal chat (CHAT_SCREEN) with **"Accept"**, **"Edit"**, **"Reject"**, **"Edit Offer"**, and the **EDIT_OFFER** screen ("Edit Offer").

## How it works
**The simple path:** the influencer's offer arrives already "waiting payment" (OFR-01). If the brand is happy, it just pays (OFR-03). Paying *is* the brand's acceptance.

**Brand counters ("Edit"):**
1. The brand clicks **"Edit"** → pop-up **"Edit Offer"** ("Influencer can accept or reject your changes. Rejection means no change to original terms."). Fields: "Price ($USD):", "Expected Delivery Date:", "Duration:" ("Permanent"/"Temporary"), "Bio URL in Influencer's profile:".
2. **"Send Edit for Approval"**. Rules: price at least the minimum (default $10), date must be after today (not today: "You can't expect influencers to publish your content today…"), a temporary post must stay up more than 4 hours.
3. The offer moves to **pending**. The brand now sees **"Waiting Influencer Approval"**, the influencer sees **"Waiting Your Approval"** and gets an "offer updated" email/push. A card in the chat lists what changed.

**Influencer answers a counter:**
- **"Accept"** → status goes back to **awaiting payment**; the brand sees "Waiting Payment" and the **"Pay"** button, and gets an "offer accepted" notification.
- **"Edit"** → EDIT_OFFER screen; their new terms put the offer back to awaiting payment (brand sees "Waiting Payment" with the changes).
- **"Reject"** (with a confirmation) → status **rejected**.

**Influencer changes their own offer before payment:** "Edit Offer" on the card (status "Waiting Brand Approval"). The offer stays awaiting payment with the new terms, and the brand is notified.

**Brand rejects:** **"Reject"** → "Are you sure you would like to reject this offer?" → **"Yes, Reject it"** → a short wizard ("Reject The Offer"):
1. Reasons ("Please select a reason", e.g. "The requested fees by influencer is out of my budget."), optional comment.
2. Optional "Let's get your campaign much more targeted influencers!": countries, languages, gender, influencer size. **These answers update the campaign's targeting** (country and gender limits), which affects who can make future offers (OFR-01).
3. Optional "Let's archive all the current UNMATCHED offers": archives other waiting-payment offers on this (and selected) campaigns that don't match the new targeting.

**After a rejection** either side can reopen: the influencer with "Edit Offer" (offer returns to awaiting payment), the brand with "Edit" (offer returns to pending, waiting for the influencer).

## Rules and limits
- Negotiation is only possible **before payment** (statuses pending, awaiting payment, rejected). After payment, the brand's "Edit" becomes a **revision** request (OFR-05), and date changes follow OFR-04.
- You can't accept or reject **your own** latest change ("you can't change your history updates status"). That's why the brand never sees "Accept" in normal use: an offer waiting for the brand is already "Waiting Payment", and the action is "Pay".
- The influencer can accept or reject only when the offer is **pending** (brand countered) and their social account is **active** (not pending review, not suspended): "Your … account's quality has been currently disapproved to make any offers."
- Countering is refused when the influencer's account is suspended. Blocked influencers can't edit: "Unfortunately, you have been blocked by the owner of this campaign…".
- If the brand counters while an **admin edit** is waiting for the influencer, the admin edit is discarded (OFR-06).

## Statuses
| Internal status | Brand sees | Influencer sees | Next step |
|---|---|---|---|
| `awaiting-payment` | "Waiting Payment" ("Influencer is waiting payment to be escrowed") | "Waiting Brand Approval" | Brand pays, edits or rejects; influencer can edit |
| `pending`, brand changed last | "Waiting Influencer Approval" | "Waiting Your Approval" | Influencer accepts, edits or rejects |
| `pending`, influencer changed last (rare, e.g. after an admin action) | "Waiting your Approval" | "Waiting Brand Approval" | Brand edits or rejects |
| `pending`, unclear who changed last | "In Negotiation" | – | Either side edits |
| `rejected` | "Rejected by Brand" / "Rejected by Influencer" | "Rejected by Brand" (with "See the brand`s reason(s)") / "Rejected by You" | Either side edits to reopen |
| `paid` | "Scheduled" and later statuses | "Paid & Scheduled" and later | PUB-01 |

In the admin panel the raw values are `pending`, `awaiting-payment`, `rejected`, `paid`, `refunded`.

## Timers and automatic actions
- **Frozen flag** (every 10 minutes): offers waiting payment for more than a day, or where one side has been inactive for more than a day, are marked `is_frozen` and pushed to the CRM. Nothing changes for users. Any new counter clears it.
- **Pruning** (monthly, 1st at 07:30 UTC): **pending** offers older than 6 months, with no payment, no publish job and no brand activity, are permanently deleted together with their chat.
- Payment reminders to the brand (1 and 3 days after the offer) are described in OFR-01.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Brand counters | Influencer | Email, push, in-app ("offer updated") |
| Brand rejects | Influencer | Email, push, in-app ("offer updated") |
| Influencer edits | Brand | Email/in-app per template ("offer updated") |
| Influencer accepts a counter | Brand | Email/push/in-app per template ("offer accepted") |
| Influencer rejects | Brand | Email/in-app per template ("offer updated") |
| Any of the above | Both | A card in the chat (CHAT-03); system cards don't send a chat push |

## Common questions

### "Where is the Accept button? I want to accept the influencer's offer."
**Answer for the user:** On Ainfluencer you accept an influencer's offer by paying for it. Click "Pay $…" in the chat header; your money is held safely until the content is delivered.
**What to check:** the offer status is `awaiting-payment` (brand sees "Waiting Payment").
**Escalate when:** the status is "Waiting Payment" but there is no Pay button.

### "I changed the price and now I can't pay."
**Answer for the user:** When you edit an offer, the influencer has to approve your new terms first. Once they accept, the status changes back to "Waiting Payment" and you can pay.
**What to check:** status `pending` with the brand as last modifier ("Waiting Influencer Approval").
**Escalate when:** the influencer says they accepted but the status didn't change.

### "The influencer can't accept my counter-offer."
**Answer for the user:** Their account may be under review or restricted at the moment. They'll be able to respond once it's active again.
**What to check:** the influencer integration `qc_status` (must be `active`), suspension (INF-04).
**Escalate when:** the account is active and the offer is pending, but accept still fails.

### "I rejected an offer by mistake. Can I undo it?"
**Answer for the user:** Yes. Open the chat and use "Edit" to send new terms; the offer goes back to the influencer for approval. (Influencers can do the same with "Edit Offer".)
**What to check:** status `rejected`.
**Escalate when:** Edit fails on a rejected offer.

### "Why did my campaign's targeting change?"
**Answer for the user:** When rejecting an offer, the optional "get more targeted influencers" step saves your answers (countries, gender) to the campaign. You can change them again in the campaign settings.
**What to check:** the rejection comments/settings saved on the offer and the campaign's `limit_country` / `limit_gender`.
**Escalate when:** the brand skipped that step and the targeting still changed.

### "Some offers disappeared from my inbox after I rejected one."
**Answer for the user:** In the reject wizard you chose to archive offers that don't match your campaign. They are archived, not deleted.
**What to check:** offers on the campaign with `state = archive` set at the same time.
**Escalate when:** offers vanished without the archive option being chosen.

### "My old pending offer is gone."
**Answer for the user:** Offers that stay in negotiation for 6 months with no activity are removed automatically.
**What to check:** offer age and status; it is force-deleted, so it won't be in the database.
**Escalate when:** a paid or recently active offer disappeared.

## Known issues
- **Brand "Accept" button.** In the rare case the brand sees "Waiting your Approval" (for example after an admin sends an offer back), the **"Accept" button does nothing useful**: the backend only supports "reject" through this action. Workaround: the brand uses "Edit" (sends terms to the influencer) or asks support/admin to reset the offer to "Waiting Payment" (OFR-06). Escalate to dev.
- **Brand "Reject Offer" in the ⋮ menu** only files the offer under the "rejected" inbox view; it does **not** reject the offer. Use the "Reject" button in the header to really reject it.

## Related topics
- OFR-01 creating the offer; OFR-03 paying; OFR-04 date changes; OFR-05 changes after payment; OFR-06 admin edits.
- CHAT-01 where negotiation happens; CHAT-03 the update cards.
- INF-04 suspensions that block actions.

## Technical reference
- **Brand endpoints** (`routes/api/v2/brand.php`, controller `app/Http/Controllers/v2/Frontend/OfferController.php`, service `app/Services/Implementation/OfferService.php`):
  - `PUT /api/v2/offers/{offer}/counter` → `counter()` (request `app/Http/Requests/v2/Offer/CounterRequest.php`; status logic in `app/Helpers/History/ChangeFinders/OfferChangeFinder.php` + `app/Helpers/History/HistoryLogger.php::apply()`: brand change → `pending`, influencer change → `awaiting-payment`; on `paid` the change is applied directly ("amend mode")).
  - `POST /api/v2/offers/{offer}/change-status` → `changeStatus()` (only `reject` is implemented; `accept` returns 400) → `app/Classes/Offer/Rejection/StartRejectionOffer.php` pipeline.
  - `GET /api/v2/offers`, `GET /api/v2/offers/{offer}`, `GET /api/v2/offers/{offer}/history`, `PATCH /api/v2/offers/{offer}/state` (inbox state: archive, spam, block, delete, reject-view; see CHAT-01).
- **Influencer endpoints** (`app/Modules/Mobile/Routes/api.php`):
  - `PUT /api/{locale}/v3/mobile/offers/{id}/history` `{action: accept|reject}` → `OfferHistoryController` → `app/Modules/Mobile/Services/Implementation/Offer/OfferHistoryService.php` (requires `offers.status = pending` and `qc_status = active`).
  - `PUT /api/{locale}/v3/mobile/offers/{id}` → `OfferUpdaterService::update()` (request `UpdateOfferRequest`, min price rule).
  - `GET …/offers/{id}`, `GET …/settings/offers/pricing`.
- **Data:** `offers.status`, `offers.state`, `offers.is_frozen`, `offers.brand_last_activity_at` / `influencer_last_activity_at`, `change_histories` (per-field changes, `batch`, `status` pending/accepted/rejected, `user_id` = who changed; latest row decides the label), `rejection_reasons`.
- **Overall status labels:** `app/Modules/OfferOverallStatus/OfferStatuses/*.php`; mobile mapping `app/Modules/Offer/Support/OfferOverallStatusBadge.php` and `src/types/offerTypes.ts` ("Waiting Payment" → "Waiting Brand Approval").
- **Jobs:** `offers:detect-frozen` (`app/Console/Commands/Offer/DetectFrozenOffer.php`, every 10 min), `offers:prune` (`PruneRedundantOffers.php`, monthly 1st 07:30 UTC).
- **Web:** `src/components/pages/inboxV37/InboxV37ChatArea.tsx`, `resolveOfferChatActions.ts`, `newSingleOfferV2/parts/modal/index.tsx` (Edit Offer / revision switch on "Scheduled").
