---
id: CMP-03
title: Managing campaigns
domain: Campaigns (brand side)
audience: [brand, admin, system]
apps: [web, email, background]
related: [CMP-01, CMP-02, CMP-04, CMP-05, SUB-02, SUB-03, SUB-04, PUBL-01, DSC-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# CMP-03 · Managing campaigns

> The Campaigns page lists every campaign the brand has (paid collaborations, gifting and affiliate). From it the brand edits, pauses and resumes, duplicates, deletes and shares campaigns. Campaigns can also be paused by an admin, by the plan check or by a subscription pause, and some campaigns expire on their own.

## Who and where

- **Brand** (owner or team member), web app, sidebar "Campaigns" → `/campaigns`.
- Page title "Campaigns". Tabs "All", "Collaboration", "Seeding", "Affiliate", a search box ("Search campaigns"), status filters, grid or list view, and summary tiles ("Active campaigns", "Deals closed", "Total reach", "In draft"). "New Campaign" opens the type picker (CMP-01).
- Brands with unfinished drafts may see a "Welcome back! 👋" pop-up ("You have N campaigns waiting to launch...") with "Continue setup", "Maybe later" and "Don't show this modal again".
- Per-campaign influencer pages: `/campaigns/affiliate-influencers/:id` (affiliate, CMP-04) and `/campaigns/seeding-influencers/:id` (gifting pipeline, CMP-05).

## How it works

Each card has these actions (icons with tooltips):

| Action | Which campaigns | What it does |
|---|---|---|
| "Edit" | All | Collaboration: reopens the wizard at the first brief question (CMP-01). Affiliate: opens the affiliate campaign details (CMP-04). Gifting: opens the gifting wizard (CMP-05). |
| "Duplicate" | Collaboration only | Creates a copy as a **Draft** named "<title>_N". The copy has **no images/videos, no bio link, and is open to all** (not invite-only). The brand finishes it in the wizard. Toast: "The campaign got duplicated successfully". |
| "Copy campaign url" | Not available for drafts. Collaboration and gifting need Pending or Approved. | Copies the public preview link (`/view-campaign/:id`, or `/view-campaign-affiliate/:id` for affiliate). Toast: "The link is copied to clipboard". Tooltips when disabled: "Available after publishing" / "Available after approval". See PUBL-01. |
| "Preview" | Collaboration, affiliate | Shows how the campaign looks. |
| "Delete" | All | Confirmation "Delete campaign" / "Are you sure you want to delete this campaign?" / "Yes, delete it". Toast: "The campaign got deleted successfuly". |
| "Pause" switch | Collaboration, affiliate | Switch on = paused, switch off = live again. Enabled only when the campaign is Approved or Paused. |
| "Invite only" toggle | Collaboration | Switches between "only invited influencers can send offers" and "everyone can". Doesn't send the campaign back to review. |
| "Invite Influencer" | Approved, live collaboration | Goes to influencer search to invite (DSC-03). |
| "Manage Influencer(s)" | Approved gifting campaign; affiliate | Opens the gifting pipeline or the affiliate influencer list. |

**Pausing:** the campaign disappears from the marketplace at once. Existing offers and deals carry on as normal. The brand also gets the email "Turn back ON your Campaign to Stay Visible in Ainfluencer Marketplace- Reach Millions of People".

**Resuming** (switching "Pause" off) puts the campaign back on the marketplace. Two checks run first:
1. The brand's subscription must not be paused (SUB-03). If it is, the request is refused.
2. The brand must have a free campaign slot (SUB-04). If not, it's refused.

In both cases the card flips back and shows "Could not update pause status. Please try again.".

**Editing** a collaboration goes through the wizard again. Saving the upload step sends the campaign back to **"Pending Approval"**, even if it was approved, paused or expired. It disappears from the marketplace until it's approved again (CMP-02).

**Deleting** a collaboration archives it: it disappears for the brand and for influencers. Existing offers and deals aren't cancelled. The web app has no undo; an admin can see archived campaigns.

## Rules and limits

- **Who can pause:**

  | Who | How | Recorded as |
  |---|---|---|
  | Brand | "Pause" switch | `brand` |
  | Admin | admin panel | `admin` |
  | Plan check | after a downgrade or cancellation (oldest running campaigns first, SUB-02), or at approval when over quota (CMP-02) | `system` |
  | Subscription pause | freezes every live campaign at once (SUB-03) | `subscription_pause` |

- **Who can resume:**
  - a campaign paused by the brand, an admin or the plan check can be switched back on **by the brand, as long as a slot is free**. The code doesn't check who paused it;
  - a campaign frozen by a subscription pause **can't be switched on by the brand while the subscription is paused**. All such campaigns come back automatically when the subscription resumes. Campaigns the brand paused itself stay paused.
- **Slots:** running (approved and active) campaigns and campaigns waiting for review use a slot. Paused, expired, rejected, draft and deleted ones don't. Resuming or resubmitting needs a free slot. "Duplicate" also needs one, even though the copy is only a draft.
- **Expiry:**
  - paid collaborations made in today's wizard **have no end date and never expire**;
  - older campaigns with a date range (and campaigns given dates in the old editor) expire when their last date has passed;
  - affiliate campaigns expire on their end date (CMP-04);
  - gifting campaigns expire on their end date (hourly check, CMP-05);
  - a paused campaign doesn't expire while paused. It's expired within a minute of being switched on if its end date has passed.
- **Reactivating an expired campaign:** an admin can reactivate an approved, expired, non-deleted campaign. This removes its end date. The brand can revive an expired affiliate campaign by moving its end date into the future in "Edit" (needs a free slot).

## Statuses

Collaboration cards:

| Chip | Meaning |
|---|---|
| "Draft" | Not submitted yet (CMP-01). |
| "Pending Approval" | Waiting for admin review (CMP-02). |
| "Approved" | Live on the marketplace. |
| "Disapproved by Admin" | Rejected. Info icon shows the reason. |
| "Paused" | Not visible to influencers. See "Who can resume". |
| "Expired" | End date passed. Not visible to influencers. |

Gifting and affiliate cards use the labels "Draft", "Pending", "Approved"/"Active", "Paused", "Rejected" and "Expired" (see CMP-04 and CMP-05).

Internal values: `ads.status` (draft/active/paused/expired/approval), `ads.approval` (draft-pending/pending/approved/rejected), `ads.paused_by` (brand/admin/system/subscription_pause, empty unless paused).

## Timers and automatic actions

| Job | When (UTC) | What the brand notices |
|---|---|---|
| `ads:expiry-check` | every minute | Active campaigns whose dates have all ended, or affiliate campaigns past their end date, become "Expired". Brand gets "Campaign Expiry". Up to 2,000 per run. |
| `plans:detect-downgrades` (+ Stripe webhook) | hourly | After a paid plan ends, the oldest running campaigns over the new limit are paused (`system`). See SUB-02. |
| Subscription pause / resume | when Stripe reports it, safety net hourly | All live campaigns frozen, then restored (SUB-03). |
| `product-seedings:expiry-check` | hourly | Gifting campaigns past their end date expire (CMP-05). |

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Brand or admin pauses a campaign | Brand | Email + in-app ("Turn back ON your Campaign...") |
| Campaign expires ("Campaign Expiry") | Brand | Email, push, in-app |
| Plan check or subscription pause pauses campaigns | No campaign-specific message from this code | – |

## Common questions

### "I can't switch my campaign back on. It says 'Could not update pause status'."
**Answer for the user:** Your plan allows a set number of live campaigns at once, and you're at that number. Pause or delete another campaign, or upgrade your plan, and then switch this one on. (If your subscription is paused, campaigns stay off until you resume it.)
**What to check:** 1) Is the subscription paused (SUB-03)? 2) The plan's campaign limit vs. campaigns that are active+approved or active+pending (SUB-04).
**Escalate when:** the brand is under its limit, the subscription isn't paused, and the switch still fails.

### "My campaign was paused but I didn't do it."
**Answer for the user:** Campaigns can be paused automatically when a plan ends or is downgraded, when a subscription is paused, or by our team. We can tell you which one applies.
**What to check:** `ads.paused_by`. `system`: plan downgrade or approval over quota (SUB-02, CMP-02). `subscription_pause`: SUB-03. `admin`: check internal notes.
**Escalate when:** `paused_by` is empty or doesn't match what happened.

### "My campaign disappeared from the marketplace after I edited it."
**Answer for the user:** Changes to a live campaign are reviewed again. It's back as soon as our team approves it.
**What to check:** status Pending Approval (`active` + `pending`).
**Escalate when:** the brand needs a fast re-review (review team).

### "My campaign expired. Can I extend it?"
**Answer for the user:** Paid collaborations created today don't expire. Older campaigns with an end date can be reactivated by our team. For affiliate campaigns, open "Edit" and choose a new end date.
**What to check:** the campaign type and whether it has dates (`time_tables` or `ads_affiliate.end_date`).
**Escalate when:** a paid collaboration without dates shows "Expired" (bug), or the brand wants a reactivation (admin).

### "I duplicated a campaign and the copy has no images."
**Answer for the user:** A duplicate copies the brief but not the uploaded content, and it starts as a draft. Open "Edit", add your images or videos and finish the steps to submit it.
**What to check:** nothing.
**Escalate when:** the copy is missing text fields too.

### "The duplicate button shows the upgrade pop-up."
**Answer for the user:** Making a copy needs a free campaign slot on your plan, even though it starts as a draft.
**What to check:** campaign limit vs. held campaigns.
**Escalate when:** the brand has free slots.

### "I deleted a campaign by mistake."
**Answer for the user:** Deleted campaigns can't be restored from the app. Our team can look into it.
**What to check:** the campaign is archived (`ads.archived_at`/`deleted_at`).
**Escalate when:** always. Restoring needs an admin or dev.

## Known issues

- **Resume failures show a generic error.** When resuming is blocked by the plan limit or a paused subscription, the user sees "Could not update pause status. Please try again." instead of the upgrade pop-up or the real reason.
- **Affiliate campaigns: pause and delete on `/campaigns` change only the Ainfluencer side.** The "Pause" switch and "Delete" on this page don't update the campaign on the affiliate platform, so affiliates may keep promoting it there. The same actions on `/affiliate/affiliate-campaign-list` do update the affiliate platform (pause there = archived). Workaround: pause or delete affiliate campaigns from the affiliate campaign list. Escalate to dev if an affiliate campaign keeps earning after deletion.
- **Gifting campaigns have no pause switch** on this page (CMP-05).
- **Copied link for gifting campaigns** opens the generic preview `/view-campaign/:id`, not the gifting preview (PUBL-01).

## Related topics

- CMP-01: creating and editing in the wizard. CMP-02: review after edits.
- CMP-04: affiliate campaigns. CMP-05: gifting campaigns.
- SUB-02: downgrade/cancel pauses. SUB-03: subscription pause. SUB-04: campaign slots.
- PUBL-01: the public campaign link. DSC-03: inviting influencers.

## Technical reference

- **Endpoints:**
  - `GET /api/v2/campaigns` (`v2/Frontend/AdController@index` → `Services/Implementation/AdService::list`), `GET /api/v2/campaigns/{ad}`.
  - `PUT /api/v2/campaigns/{ad}` (`@update`, middleware `campaign.activation.quota` = `PlanEnforcement/Http/Middleware/EnforceCampaignActivationQuota`: 400 if the subscription is paused, 402 if no slot, only when `status=active` is requested for a non-active ad by a non-admin).
  - `DELETE /api/v2/campaigns/{ad}` (`@destroy`, archive for brands), `POST /api/v2/campaigns/{ad}/duplicate` (`feature:campaigns`).
  - Gifting list/delete: `GET product-seedings/list`, `DELETE product-seedings/{id}`. Affiliate list: `GET v2/campaigns-affiliate`.
- **Pause attribution:** `Support/Campaign/CampaignPauseAttribution` (brand/admin), `Helpers/Enums/Ad/AdPausedByEnum`, `PlanEnforcement/Services/SystemCampaignPauser` (`system`), `Services/Susbscription/Pause/CampaignFreezer` (`subscription_pause`; freezes every `status=active`, non-archived ad; thaws only its own).
- **Expiry:** `Console/Commands/AdExpiryCheck.php` (every minute, max 2,000 dispatches; time tables all ended, or `ads_affiliate.end_date < now`) → `Jobs/Ad/ExpireAdJob.php` (status `expired`, `Notifications/AdExpiry`). Admin reactivate: `Admin/AdService::reActivate`.
- **Data:** `ads` (`status`, `approval`, `paused_by`, `invite_only`, `archived_at`, `deleted_at`), `time_tables` (`end`), `ads_affiliate` (`end_date`), `activity_logs` (delete actions).
- **Key files:** `core/frontend/web/src/components/pages/campaignsV2/index.tsx`, `CampaignCard.tsx`, `collaborationStatus.ts`; `core/backend/api/app/Services/Implementation/AdService.php` (`update`, `duplicate`, `destroy`).
