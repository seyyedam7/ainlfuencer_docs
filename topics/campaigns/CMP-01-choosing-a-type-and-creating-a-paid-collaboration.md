---
id: CMP-01
title: Choosing a campaign type and creating a paid collaboration
domain: Campaigns (brand side)
audience: [brand]
apps: [web, email]
related: [CMP-02, CMP-03, CMP-04, CMP-05, INT-01, SUB-04, DSC-03, OFR-01, PAY-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# CMP-01 · Choosing a campaign type and creating a paid collaboration

> Every new campaign starts on the campaign-type picker. A "Paid Collaboration" is a brief that influencers see in the mobile app and answer with a priced offer. The brand builds it in a step-by-step wizard; when the brand uploads content in the wizard, the campaign is sent to the Ainfluencer team for review (CMP-02).

## Who and where

- **Who:** a brand (or a team member working inside the brand's team) in the web app (app.ainfluencer.com).
- **Picker:** `/affiliate/select_campaign_type`, title "Select the type of campaign for your brand". About 20 buttons lead here, for example "New Campaign" on `/campaigns`. Brands also land here after onboarding (BRD-01).
- **Paid-collaboration wizard:** `/create-campaign`. The page shows one step at a time; the URL keeps `?adId=<campaign id>&step=<number>` so a refresh returns to the same step.
- **Old wizard:** `/campaign/objectives/:id` and the steps after it. It opens only from the "Edit ad" link in the "select campaign" pop-up on an influencer's profile. Treat it as legacy: new campaigns are not created there.

## How it works

### The picker

The picker shows four cards:

| Card | What it says | What happens on click |
|---|---|---|
| "Affiliate Campaign" | "Pay influencers a commission (% or $ amount) for each sale they make" | Opens "How will you track affiliate sales?" (see below). |
| "Paid Collaboration" | "Pay influencers a fixed fee to create and post content." / "Requires a $10 minimum payment per influencer." | Opens `/create-campaign`. |
| "Paid collab + Affiliate" | badge "Coming Soon" | Nothing. It can't be clicked. |
| "Product Gifting & Seeding" (highlighted) | "Send free products to influencers in exchange for a post" / "Requires an active subscription." | Opens the gifting wizard (CMP-05). |

"How will you track affiliate sales?" has two choices:
- "Track sales through Ainfluencer" (marked "Recommended"): if the brand already has a **card on file and a connected Amazon or Shopify store** on the affiliate platform, the affiliate campaign setup starts (CMP-04). Otherwise the brand is sent to `/affiliate/dashboard` to connect a store and a card first.
- "I already have an affiliate platform": opens the Paid Collaboration wizard. The idea is that the brand pays a fixed fee and shares its own affiliate links ("Works with any platform · $10 minimum per influencer").

"Back" returns to `/affiliate/dashboard`.

### The paid-collaboration wizard

1. **Start screen** ("Create Your Campaign in 3 Steps!"). The brand clicks "Let's Go". The system creates an empty **draft** campaign straight away (internal title "No title"). This is where the **campaign-slot check** first happens: if the brand's plan has no free campaign slot, the upgrade pop-up opens and nothing is created (SUB-04).
2. **Store choice.** "I am a Shopify user" opens the Shopify store picker ("Select Your Shopify Store" / "Connect To Shopify", see INT-01). "Others: My products are not on Shopify" continues with the normal form. Then "Next". If the brand picked Shopify but the store isn't connected, it sees "Connect Your Shopify Store".
3. **The brief, one question per screen.** Each "Next" saves that answer to the draft, so a brand can stop at any point and continue later. The normal (non-Shopify) order is:
   - "What do you expect influencers to do for your campaign?" (post my existing content / create and publish content / only create content for me)
   - "What type of influencers would you need for your campaigns?" (Nano 5k–10k, Micro 10k–100k, Macro 100k–1m, Mega over 1m, "All and Any Range")
   - "Are you looking to work with influencers of a given gender?" (Male / Female / All)
   - "Are you aiming to ship physical products to influencers?" If "Yes", the brand picks the countries influencers must live in, and the next screen is skipped.
   - "Select countries where you want influencers to be located:"
   - "The language your target influencers speak:"
   - "How long would you like the post to stay on the Influencer’s social accounts?" ("I will negotiate it for each offer", "I'd like it to be permanent", or "Let's set a specific duration for it")
   - "Would you like to have influencers add your link in their bio when publishing your campaign?" If "Yes", "Enter your product URL:"
   - "Your campaign categories" (at least one)
   - "Related hashtags to your campaign" (can be skipped)
   - "Who can send offers to you?" ("Only invited influencers" or "Allow all influencers to make offers")
   - "Campaign title" and "Campaign description". "Use Templates" fills in an example. "Create Title & Description with AI" writes a draft from the brief (optionally from the brand's website URL).
   - Platforms and post types: Instagram ("Feed", "Story", "Reel"), "TikTok Video", "YouTube Video", with an optional caption.
   - "Upload some content": images or videos for influencers to use. Uploads, the brand's content library and stock photo libraries are allowed.
   - "About your brand": a short brand description.
4. **Submission happens at the upload step.** When the brand saves the upload step with at least one file, the campaign stops being a draft. It becomes **"Pending Approval"** and enters the admin review queue (CMP-02). The campaign-slot check runs again here.
5. **After the last step** ("About your brand"), a pop-up opens: "Fund your wallet & start closing deals with top creators". The brand can pick an amount and go to `/funds`, or click "Skip for now — launch without funding". Funding the wallet is **optional**: it isn't needed for approval or for receiving offers (PAY-01).
6. If the brand had chosen influencers to invite *before* creating the campaign (from search or favourites), the wizard sends those invitations at the end. They are **held until an admin approves the campaign** ("After admin approval, your invitations will be sent.").

A Shopify campaign follows the same idea, but the brand first picks a Shopify product, and AI can pre-fill parts of the brief ("Proceed with AI" / "Skip AI & Continue"). The product can't be changed later ("Note: You cannot change the Shopify store product while editing the campaign...").

## Rules and limits

- **Campaign slots (plan limit).** The plan sets how many campaigns a brand can *hold* at the same time. Running campaigns and campaigns waiting for review both count. Drafts, rejected campaigns and expired ones don't. When the brand is at its limit, clicking "Let's Go" already fails with the upgrade pop-up, even though a draft would be free. Plan numbers are in SUB-04.
- **Only the brand owner's plan counts.** If a team member creates the campaign, the owner's slots are used.
- **Minimum price per influencer:** the picker says "$10 minimum payment per influencer". The real minimum offer price comes from the brand's plan and is checked when influencers send offers (OFR-01, SUB-04).
- **Content:** at least one image or video, up to 10 in total. Images: JPG, JPEG or PNG, 8 MB max. Videos: MP4, WEBM, WMV, AVI, MOV, OGG, OGV or MPEG, 100 MB max.
- **Post duration:** if the brand sets a specific duration, it must be at least 4 hours ("The duration of published posts on social channels must be longer than four hours.").
- **Caption:** up to 2,200 characters and 30 hashtags. Contact details aren't allowed ("You cannot add your contact information yet...").
- **About your brand:** up to 1,024 characters.
- **AI title and description:** **2 uses per campaign** ("1 more use left for AI title & description generator!", then "Feature used twice. Limit reached."). Only the brand account owner can use it.
- **No end date.** The current wizard doesn't ask for campaign dates. A paid collaboration created here stays on the marketplace until the brand pauses or deletes it. It never expires on its own (CMP-03).
- **Visibility:** influencers see the campaign only after admin approval (CMP-02). With "Only invited influencers", only invited influencers can send offers.

## Statuses

| What the brand sees | Meaning | What happens next |
|---|---|---|
| "Draft" | Created with "Let's Go" but the upload step isn't saved yet. Invisible to influencers. Holds no slot. | Reminder emails after 24 and 72 hours. The brand finishes the wizard from "Edit" on `/campaigns`. |
| "Pending Approval" | Submitted, waiting for the Ainfluencer team. Holds a slot. Invisible to influencers. | Admin approves or rejects (CMP-02). |

Internal values (admin panel): draft = `status=draft`, `approval=draft-pending`; submitted = `status=active`, `approval=pending`.

## Timers and automatic actions

- **Draft reminders:** 24 hours and 72 hours after the draft is created, the system checks whether the campaign is still a draft. If it is, the brand gets the email "Complete your Campaign to Get Published in Ainfluencer Marketplace- Reach Millions of People", with a link back to the wizard. Gifting campaigns don't get these reminders.
- Nothing else runs automatically before submission.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Draft still unfinished at 24 h and at 72 h | Brand | Email + in-app (in-app only if the brand unsubscribed from emails) |
| Submission | Nobody is notified. The brand sees "Pending Approval" on `/campaigns`. | – |
| Approval or rejection | Brand | See CMP-02 |

## Common questions

### "I click 'Let's Go' and get an upgrade pop-up. I don't even have a campaign yet."
**Answer for the user:** Your plan allows a set number of campaigns at the same time, and campaigns waiting for review count too. You're at that number, so a new one can't be started. You can pause or delete a campaign you no longer need, or upgrade your plan.
**What to check:** the brand's plan and its "campaigns" limit (SUB-04). Count its campaigns that are active+approved or active+pending (not archived or deleted).
**Escalate when:** the brand holds fewer campaigns than its limit and still gets the pop-up.

### "Where do I set the campaign start and end dates?"
**Answer for the user:** Paid collaborations created today have no end date. They stay visible until you pause or delete them from the Campaigns page. Influencers propose the posting date in their offer.
**What to check:** nothing. This is by design in the current wizard.
**Escalate when:** never. If the brand needs a fixed end date, suggest pausing it on the day.

### "I finished the wizard but my campaign isn't visible to influencers."
**Answer for the user:** Every new campaign is checked by our team before it goes live. You'll get an email when it's approved or if something needs changing.
**What to check:** the status on `/campaigns`. "Pending Approval" means it's in the review queue (CMP-02). "Draft" means the upload step wasn't saved: the brand must add at least one image or video.
**Escalate when:** the campaign has been "Pending Approval" for an unusually long time (ask the review team), or it's "Approved" but influencers say they can't find it.

### "The 'Affiliate Campaign' option sends me to the affiliate dashboard instead of creating a campaign."
**Answer for the user:** To run an affiliate campaign through Ainfluencer you first need to connect your Amazon or Shopify store and add a payment card. Once both are done, the Affiliate option starts the campaign setup.
**What to check:** on `/affiliate/dashboard`, whether a store is connected and a card is added (CMP-04).
**Escalate when:** both are connected and the brand is still redirected.

### "The AI title button says 'Limit reached'."
**Answer for the user:** The AI writer can be used twice per campaign. You can still edit the title and description by hand or use "Use Templates".
**What to check:** nothing.
**Escalate when:** the brand reports it never used it and sees the limit. A team member (not the owner) also can't use it; see Known issues.

### "Why does it say $10 minimum?"
**Answer for the user:** Influencers can't offer less than the minimum price set for your plan. You agree on the final price with each influencer in the inbox.
**What to check:** the brand's plan setting for the minimum offer price (SUB-04).
**Escalate when:** influencers report being blocked from offering above the minimum.

### "I invited influencers while creating the campaign. They haven't received anything."
**Answer for the user:** Invitations you send while a campaign is being reviewed are held and go out as soon as it's approved.
**What to check:** campaign approval status. After approval, invitations go out in the grouped invitation emails at 06:00 and 18:00 UTC (DSC-03).
**Escalate when:** the campaign has been approved for more than 24 hours and the influencers still have no invitation.

## Known issues

- **The Affiliate card never looks disabled.** The card looks clickable even without a store or card, and clicking "Track sales through Ainfluencer" just redirects to the affiliate dashboard with no explanation. Workaround: explain the store + card requirement (CMP-04).
- **Upgrade pop-up at "Let's Go".** Even creating a draft is blocked when the brand is at its campaign limit, although drafts don't use a slot.
- **AI title/description fails for team members** *(to verify)*: the check compares the user to the brand owner, so a team member working in the brand's team gets an error. Workaround: the owner uses the AI button, or the text is written by hand.
- **The gifting "founder call" pop-up in the picker is never shown.** The "Product Gifting & Seeding" card always opens the wizard directly.
- **The legacy editor (`/campaign/*`)** is still reachable from influencer profiles ("Edit ad"). Edits made there send a non-approved campaign back to review.

## Related topics

- CMP-02: admin review after submission.
- CMP-03: editing, pausing, duplicating, deleting campaigns.
- CMP-04: affiliate campaigns. CMP-05: gifting campaigns.
- INT-01: connecting a Shopify store for a Shopify campaign.
- SUB-04: campaign slots and other plan limits.
- DSC-03: inviting influencers. OFR-01: influencers sending offers.

## Technical reference

- **Endpoints**
  - `POST /api/v3/campaigns` → `AdCreationProcess/Http/Controllers/AdCreationProcessController@store`, middleware `feature:campaigns` (only enforced when the `request` field is empty or `create-campaign`). The `request` field picks the step class in `AdCreationProcessFactory`: `create-campaign`, `influencer-task-type`, `target-influencers-followers`, `target-influencers-gender`, `physical-product`, `target-influencers-countries`, `target-influencers-language`, `campaign-duration`, `campaign-bio-link`, `campaign-categories`, `campaign-hashtags`, `campaign-invite-only`, `campaign-title-description`, `campaign-content`, `campaign-medias`.
  - `POST /api/v2/campaigns/progress-step` (stores `ads.extra.steps_completed`); `GET /api/v2/campaigns/templates-list`; `GET /api/v2/ai/campaign-creation/{adId}/title-and-description` (limit `TitleAndDescriptionRequest::REQUEST_LIMIT = 2`); `POST /api/files`, `GET /api/v2/all-medias`.
  - "About your brand" saves through the brand onboarding endpoint (BRD-01). Pending invites: `POST /api/v2/temp-invitations`.
  - Shopify: `POST user-third-party/shopify/v1/campaign/start-process`, `…/save-product`, `…/finish-process` (INT-01).
  - Picker status check: `GET user-third-party/ainfluencer-affiliate/v1/status` (`card_status`, `amazon_account_status`, `shopify_account_status`).
  - Legacy wizard: `POST v2/campaign/create/step1` (`feature:campaigns`), `POST v2/campaign/create/content`, `POST/PUT v2/campaigns/{ad}/publish-terms`, `POST v2/campaigns/{ad}/influencer-filters`, `PUT v2/campaigns/{ad}`.
- **Data**
  - `ads`: `status` (draft/active/paused/expired/approval), `approval` (draft-pending/pending/approved/rejected), `paused_by`, `invite_only`, `campaign_type` (`sponsored` for paid collaborations; `performance` = affiliate; `product_seeding`), `third_party` (`shopify` or null), `duration`, `bio_link`, `limit_country`, `limit_gender`, `settings`, `extra` (`steps_completed`, `website_url`).
  - Drafts live under the brand's default `campaigns` row "Main Ad Group". Media: `contents` + `medias`.
  - Submission rule: `Steps/Medias.php`. At least 1 media sets `status=active, approval=pending`, after `CampaignSlotGate::assertRoomFor`.
- **Jobs:** `AdDraftedJob` (queue `ad:drafted`), queued 24 h and 72 h after creation by `Observers/v2/V2AdObserver::created`. It sends `Notifications/AdDrafted` only if `status` is still `draft` and the campaign isn't a gifting one.
- **Key files:** `core/frontend/web/src/components/pages/affiliate/parts/selectCampaignType/index.tsx`; `core/frontend/web/src/components/pages/createAd/parts/froms/index.tsx` and `…/normalAd/index.tsx` (step order and `request` mapping); `core/backend/api/app/Modules/AdCreationProcess/*`; `core/backend/api/app/Modules/PlanEnforcement/Services/CampaignSlotGate.php`; `core/backend/api/app/Http/Middleware/CheckFeature.php`; `core/backend/api/app/Services/Susbscription/FeatureGuardService.php` (`heldCampaigns`).
