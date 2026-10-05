---
id: OFR-01
title: Making an offer
domain: Offers and negotiation
audience: [influencer, brand, system]
apps: [mobile, email, background]
related: [MKT-01, MKT-02, INF-02, INF-03, INF-04, OFR-02, OFR-03, CHAT-01, CHAT-03, DSC-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# OFR-01 · Making an offer

> On a **paid collaboration** campaign, the influencer proposes a deal from the mobile app: delivery date, how long the post stays up, price, a pitch and some details about their audience. Sending it creates the offer, opens a chat with the brand, and puts the offer straight into "waiting for the brand to pay".

## Who and where
- **Influencer**, mobile app only. Path: Explore (or search, brand profile, invitation) → campaign details screen (CAMPAIGN_DETAILS) → button **"Make offer"** (it reads **"Accept invitation"** if the brand invited them, or **"View offer"** if they already sent one) → offer form (MAKE_OFFER_SCREEN, title "Make An Offer").
- **Brand** does not create offers. It receives them in the web Inbox (`/new-offersV2`) and responds (OFR-02) or pays (OFR-03).
- Affiliate and gifting campaigns have no offer form; they use MKT-03 and MKT-04.

## How it works
1. **Influencer taps "Make offer".** The app first checks the account: maintenance → alert; suspended or archived account → message; no usable social account → onboarding (INF-01/INF-02); a *draft* (unfinished) account sees a "Complete profile" overlay that sends them back to the next onboarding step. If they arrived from an invitation link, an "Accept Invitation for Which Account?" picker appears first.
2. **The form**, top to bottom:
   - **Account** (only if they have more than one; it's display-only, switched from Explore's account selector).
   - **Link in bio: Yes** (only if the campaign requires a bio link).
   - **Deal terms**:
     - Delivery date and time ("Delivery Schedule"). Default is now + 2 weeks. Under 14 days a **"Short deadline"** warning appears ("Influencers usually set delivery 4–6 weeks out").
     - Duration: **"I will delete the content after:"** N Hour(s)/Day(s)/Week(s)/Month(s). Leaving it unticked means **"Permanent"**. Minimum is 4 hours.
     - Price ("Your Desired Price", "Enter your proposed price"). A suggested price is shown for the account. The help text says **Ainfluencer takes 20%** of the fee.
   - **Your pitch** ("Pitch what you're proposing to do for this collaboration"): 50–250 characters.
   - **About you**: "What topics do you post about?" and "Describe your audience details" (50–250 characters each), and **collab types** ("Select Collab Types", at least one required).
   - **Portfolio**: insight/analytics screenshots and past collaboration images or videos.
3. **"Save and Continue"** validates the form, then the **"Review your offer"** sheet shows Deal terms, Your pitch, About you and Portfolio. Tap **"Send Offer"** ("Making Offer...").
4. **What the system does:**
   - Checks the influencer's account, the brand and the campaign (see Rules).
   - Saves the topics, audience, insights and past collaborations onto the influencer's profile (they prefill the next offer).
   - Creates a **chat conversation** between the two users and links it to the offer (CHAT-01). If the chat can't be created, the offer is **not** created ("We could not start the chat for this offer. Please try again in a moment.").
   - Creates the offer with status **"awaiting payment"**: the influencer's proposal counts as their acceptance, so the brand can pay straight away (OFR-03).
   - Posts the first chat cards: "offer created", the influencer's **collaboration terms** (pitch, niches, audience, collab types) and an "awaiting payment" card (CHAT-03).
   - Marks any invitation to this campaign as accepted (DSC-03), notifies the brand, and schedules two payment reminder emails to the brand (see Timers).
5. **After sending**, the app returns to Explore with an **"Offer sent"** sheet explaining the next steps; **"Open offer chat"** opens the deal chat. The phone may also show the store's "rate this app" prompt.

**Accounts still under review.** If the influencer's account is still *pending* quality review (INF-03), the offer is saved as a **temporary offer**: no chat cards are posted and the brand does not see it until an admin activates the account. If the account is suspended instead, temporary offers are deleted.

## Rules and limits
- **Minimum price: $10** by default. The live minimum comes from a global "make offer pricing" setting (the app enforces $10; the server enforces the setting, default $10). It is **not** taken from the brand's plan in the current app.
- Duration minimum 4 hours (or permanent). Delivery date must be in the future.
- Pitch, topics and audience: 50–250 characters each. **Contact details** (emails, phone numbers, links, handles) in these texts are blocked by the app (CHAT-04).
- **One offer per account per campaign.** A second try shows "You have already made an offer for this campaign!". If an admin deleted the earlier offer: "You previously made an offer for this campaign but it was deleted by the admin. You cannot submit another offer for this campaign."
- **Influencer account** must be active or pending review. Draft: "Your account … is in draft status…". Suspended or other: "Your account … is temporarily suspended for account quality or qualification issues…". Profile must have gender and location: "You can only make an offer for a campaign once you have completed your profile information."
- **Campaign targeting:** if the campaign limits gender or countries, a non-matching influencer gets "Your gender or location does not meet the requirements…". **Invited influencers skip this check**, and accepting an invitation link skips the qualification checks entirely.
- **Campaign** must be active (and approved or pending approval): otherwise "The campaign "…" is not approved yet." or "This campaign is no longer active. The brand has paused it."
- **Brand** must be active (not suspended, BRD-04): "Unfortunately, due to limitations applied to the brand activities in the platform, at this moment, we cannot submit your offer."
- **Blocked by the brand:** "Unfortunately, you have been blocked by the owner of this campaign and you cannot submit your offer".

## Statuses
| What the user sees | Meaning | What happens next |
|---|---|---|
| Influencer: "Waiting Brand Approval" · Brand: "Waiting Payment" (internal `awaiting-payment`) | Offer sent, brand hasn't paid | Brand pays (OFR-03), counters or rejects (OFR-02) |
| Not visible to the brand (temporary offer) | Influencer account still under review | Appears to the brand once an admin activates the account (INF-03) |

## Timers and automatic actions
- **Payment reminders to the brand:** email after **1 day** and after **3 days**, only if the offer is still awaiting payment at that moment.
- **Automatic insight request:** when the influencer's saved insights are older than 60 days, the system may ask them for fresh insights about 1 hour after the offer (PUB-04).
- Offers that sit inactive for more than a day are flagged "frozen" internally (no visible effect). Pending offers with no activity for 6 months are deleted on the 1st of each month (OFR-02).

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| New offer received | Brand | Email, push/in-app (per template) |
| Offer still unpaid after 1 and 3 days | Brand | Email ("Your influencer Offer for … is awaiting your payment to get started.") |
| New offer chat cards | Both | In chat only (no push for system cards) |

## Common questions

### "The Make offer button sends me to onboarding / says complete your profile."
**Answer for the user:** Your account isn't finished yet. Complete the remaining profile steps in the app (media kit, niche, audience, address, goals) and then try again.
**What to check:** the account's quality-control status (draft vs pending vs active) and whether gender and country are filled.
**Escalate when:** the account is fully onboarded and active but the app still blocks the offer.

### "It says I already made an offer for this campaign, but I can't find it."
**Answer for the user:** Each account can make one offer per campaign. Your earlier offer is in your Inbox; open it from there. If it was removed by our team you can't send another one for this campaign.
**What to check:** offers for this influencer account and campaign, including soft-deleted ones (`deleted_at`) and `temporary_offers`.
**Escalate when:** no offer or temporary offer exists at all.

### "I sent an offer but the brand says they can't see it."
**Answer for the user:** If your account is still being reviewed, your offer is kept on hold and the brand will see it as soon as your account is approved (usually within 24 hours).
**What to check:** is it in `temporary_offers` (account `qc_status` pending)? Is the offer hidden because the account was suspended for 48 hours or more (`offers.extra.influencer_suspended`, INF-04)?
**Escalate when:** the offer is a normal offer, the account is active, and the brand still can't see it.

### "Why can't I set a price below $10?"
**Answer for the user:** The minimum offer on Ainfluencer is $10.
**What to check:** nothing; this is a rule.
**Escalate when:** the app refuses prices above $10 (the global minimum setting may have been raised).

### "My pitch won't save, it says contact sharing."
**Answer for the user:** For your protection, emails, phone numbers, links and social handles can't be shared before the brand pays. Remove them and describe your work in words.
**What to check:** nothing technical; see CHAT-04.
**Escalate when:** a message with no contact details is still rejected.

### "It says my gender or location doesn't meet the requirements."
**Answer for the user:** This campaign is limited to certain countries or a gender. You can only offer on it if the brand invites you.
**What to check:** the campaign's country/gender limits vs the account's countries and gender.
**Escalate when:** the account clearly matches the limits.

### "I got 'We could not start the chat for this offer'."
**Answer for the user:** Something went wrong on our side while opening the chat. Please wait a minute and send the offer again; nothing was created.
**What to check:** that no offer exists for this account and campaign.
**Escalate when:** it keeps happening (chat service problem).

## Known issues
- **Payment reminders for temporary offers** are scheduled at creation too; a brand might get an "awaiting your payment" email about an offer it can't see yet. *(to verify)* Escalate to dev if reported.
- The help text inside the app ("Ainfluencer takes 20%") describes the influencer-side cut; the brand separately pays a plan-based platform fee at checkout (OFR-03). Users sometimes mix these up.

## Related topics
- MKT-01 finding campaigns; MKT-02 invitations (invited users skip targeting checks).
- INF-02 onboarding; INF-03 account review and temporary offers; INF-04 suspensions.
- OFR-02 what happens after sending (counter, accept, reject); OFR-03 brand payment.
- CHAT-01 the offer chat; CHAT-03 the cards posted at creation; CHAT-04 contact-sharing rules.

## Technical reference
- **Endpoints:**
  - `POST /api/{locale}/v3/mobile/offers` → `app/Modules/Mobile/Http/Controllers/API/OfferController@store` → `app/Modules/Mobile/Services/Implementation/Offer/OfferCreatorService::store()`. Request: `app/Modules/Mobile/Http/Requests/Offer/CreateOfferRequest.php` (min price via `app/Http/Requests/Traits/HasMakeOfferPricingRule.php` → `SettingService::getMakeOfferPricing()`, setting row `integration_platform = make_offer_pricing`, default 10).
  - `POST /api/{locale}/v3/mobile/offers/temporary` → `storeTemporaryOffer` (same service; creates a `temporary_offers` row when `integrations.qc_status = pending`).
  - `GET /api/{locale}/v3/mobile/settings/collaboration-types`, `GET …/influencer/accounts/{id}/pricing`, `…/accounts/{id}/settings`, `POST …/files/upload`.
- **Data:** `offers` (`status` = `awaiting-payment` at creation, `identity_id` = influencer identity, `target_identity_id` = brand identity, `target_integration_id`, `ad_id`, `price`, `start`, `duration` (seconds, 0 = permanent), `chat_id` = `chatbox_conversation_id`, `context.collaboration_types`, `extra`), `temporary_offers`, `settings` (per-integration offer defaults), `change_histories`.
- **Jobs:** `app/Jobs/Offer/SendOfferInitialMessagesJob.php` (chat cards, not for temporary offers); `app/Jobs/Ad/AcceptOfferInvitations.php` (via `app/Observers/v2/OfferObserver::created`, which also sends `OfferNotification` `brand.offer:created`); `AwaitingPaymentFirstReminder`/`SecondReminder` delayed 1 and 3 days (`app/Helpers/Enums/Constants.php`), sent only if still `awaiting-payment`.
- **Mobile:** `src/features/make-offer/MakeOfferScreen.tsx`, `hook/useMakeOffer.ts` (MIN_OFFER_PRICE 10, 50–250 chars), translations `src/localization/translations/en/make-offer.js`.
