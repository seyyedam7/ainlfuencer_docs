---
id: CMP-02
title: Campaign review by admin
domain: Campaigns (brand side)
audience: [brand, admin, system]
apps: [web, email, background]
related: [CMP-01, CMP-03, CMP-04, CMP-05, SUB-04, DSC-03, MKT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# CMP-02 · Campaign review by admin

> A submitted paid-collaboration campaign isn't visible to influencers until someone on the Ainfluencer team approves it in the admin panel. Approval puts it on the marketplace and sends the invitations the brand queued. Rejection sends the brand an email with the reason. Affiliate campaigns skip this review.

## Who and where

- **Admin:** reviews campaigns in the admin panel (campaign list → campaign → approve or reject, with a reason picked from a preset list).
- **Brand:** sees the result on `/campaigns` (status chip on each card) and by email, push and in-app notification. The notification opens `/campaigns`.
- **Influencer:** sees the campaign in the mobile Explore tabs only after approval (MKT-01).

## How it works

1. The brand submits the campaign by saving the upload step of the wizard (CMP-01). The card on `/campaigns` shows **"Pending Approval"**.
2. An admin opens it and either approves or rejects it.
3. **Approve:**
   - the campaign becomes **"Approved"** and goes live on the marketplace (influencers can find it, and send offers if it's open to all);
   - the brand gets "Your Campaign Got Approved";
   - invitations the brand queued while the campaign was pending are converted into real invitations. They reach influencers in the next grouped invitation run (06:00 or 18:00 UTC, see DSC-03);
   - the plan is checked again. If the brand is now holding more campaigns than its plan allows (for example after a downgrade while this one was in review), **the campaign that was just approved is paused immediately** (see Rules).
4. **Reject:**
   - the card shows **"Disapproved by Admin"** with an info icon ("View rejection reason") that shows the reason;
   - the brand gets the email "Campaign Rejection and Reasons - Ainfluencer Marketplace" with the reason;
   - **all pending invitations for this campaign are deleted**, including the ones queued during creation.
5. **Fixing a rejected campaign:** the brand clicks "Edit" on the card, which reopens the wizard (CMP-01). Saving the "Who can send offers to you?" step or the upload step sends it back to **"Pending Approval"** automatically. A free campaign slot is needed for that.

## Rules and limits

- **Live = approved + active.** Influencers only see a campaign that is approved and not paused, expired or deleted.
- **Which campaigns are reviewed:**
  - paid collaborations: reviewed here;
  - **affiliate campaigns: not reviewed.** They go live (approved automatically) the moment the brand assigns products (CMP-04);
  - **gifting campaigns:** have their own review, with the same idea but different statuses and emails (CMP-05).
- **Editing an approved campaign sends it back to review.** If the brand goes through "Edit" and saves the upload step again, an approved campaign becomes "Pending Approval" and **disappears from the marketplace until it's approved again**. The same happens to a paused or expired campaign edited this way: it comes back as "Pending Approval".
- **Over-quota at approval:** a campaign under review already holds a slot but can't be paused. If the brand downgraded while it was in review, nothing happens until approval. At approval the system pauses *that* campaign (not an older one), marked as paused by the system. The brand can switch it on later only when a slot is free (CMP-03).
- **Brand suspension:** when an admin suspends a brand, every live campaign is switched to "rejected" with the reason "Owner suspended". When the brand is reactivated, those campaigns go back to "Pending Approval" and must be approved again.
- **Rejection reasons** come from a list managed by admins. A reason is required to reject.

## Statuses

| Chip on `/campaigns` | Meaning | Next |
|---|---|---|
| "Pending Approval" | Submitted, in the review queue. Holds a campaign slot. | Approve or reject. |
| "Approved" | Live on the marketplace. | Brand can pause (CMP-03). |
| "Paused" | Approved but stopped (by the brand, an admin, the plan check, or a subscription pause). | See CMP-03. |
| "Disapproved by Admin" | Rejected. Holds no slot. Reason in the info icon. | Brand edits and resubmits. |

Internal values: pending = `status=active, approval=pending`; approved = `status=active, approval=approved`; rejected = `status=active, approval=rejected` (`rejection_reason` set); owner suspended = `status=approval, approval=rejected, rejection_reason="Owner suspended"`.

## Timers and automatic actions

- There is **no automatic approval** and no review deadline in the code.
- At approval: queued invitations are converted (background job), and the plan check may pause the campaign (`paused_by=system`).
- Invitation emails go out at 06:00 and 18:00 UTC (DSC-03).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Campaign approved ("Your Campaign Got Approved") | Brand | Email, push, in-app |
| Campaign rejected ("Campaign Rejection and Reasons - Ainfluencer Marketplace", with reason) | Brand | Email, push, in-app |
| Campaign paused by an admin ("Turn back ON your Campaign to Stay Visible in Ainfluencer Marketplace- Reach Millions of People") | Brand | Email + in-app |
| Campaign paused by the plan check at approval | Nobody. The brand only gets the "approved" message. | – |
| Invitations released after approval | Influencers | Grouped invitation email + in-app (DSC-03) |

## Common questions

### "How long does approval take?"
**Answer for the user:** Our team reviews every new campaign by hand. You'll get an email as soon as it's approved or if something needs changing. (The system has no fixed review time. Use the review team's current turnaround.)
**What to check:** that the campaign is really "Pending Approval" (not "Draft"; a draft was never submitted, see CMP-01).
**Escalate when:** it has been pending longer than the review team's usual time. Ping the review team with the campaign ID.

### "My campaign was rejected. What do I do?"
**Answer for the user:** Hover the info icon next to "Disapproved by Admin" to see the reason, fix it with "Edit", and save. It goes back to our team automatically.
**What to check:** `ads.rejection_reason`. Confirm the brand has a free campaign slot, or resubmitting will show the upgrade pop-up.
**Escalate when:** the reason is unclear or the brand disputes it (review team).

### "It says Approved, but it's paused / I got 'approved' and then saw 'Paused'."
**Answer for the user:** Your plan allows a set number of live campaigns at the same time. When this one was approved you were already at that number, so it was paused to keep you within your plan. Pause another campaign or upgrade, then switch this one back on.
**What to check:** `ads.paused_by = system` and the brand's campaign limit vs. its live campaigns (SUB-04).
**Escalate when:** the brand is clearly under its limit.

### "I edited my live campaign and now it's 'Pending Approval' and influencers can't see it."
**Answer for the user:** Changes to a live campaign are reviewed again before they go public. It will be back as soon as our team approves it.
**What to check:** status `active` + `pending`. Ask the review team to prioritise if the brand has deals in progress (existing offers and deals are not affected).
**Escalate when:** it stays pending for long; the brand needs a fast re-review.

### "I invited influencers while creating my campaign and they never got it."
**Answer for the user:** Invitations are sent only after the campaign is approved. If it was rejected, the queued invitations were cancelled and need to be sent again after it's approved.
**What to check:** approval status. If it was rejected at any point, pending invitations were deleted.
**Escalate when:** the campaign was approved more than 24 hours ago and the invitations are still missing.

### "Why wasn't my affiliate campaign reviewed?"
**Answer for the user:** Affiliate campaigns go live as soon as you add products. They don't need a manual review.
**What to check:** CMP-04.
**Escalate when:** n/a.

## Known issues

- **No message when the plan check pauses a just-approved campaign.** The brand receives "Your Campaign Got Approved" but finds it "Paused". Explain the plan limit (above).
- **Invite allowance not given back after rejection** *(to verify)*: when an admin rejects a campaign, the pending invitations are deleted, but the "invites used" counter is rolled back on the admin's account instead of the brand's. A brand on a limited invite plan may lose those invites. Escalate to dev if a brand reports it.
- **Re-review after any edit** of an approved campaign (above). This is how the code works today, not a bug, but users don't expect it.

## Related topics

- CMP-01: submitting the campaign.
- CMP-03: pausing, resuming, plan pauses after approval.
- CMP-04: affiliate campaigns (no review). CMP-05: gifting review.
- SUB-04: campaign slots. DSC-03: invitation sending.
- MKT-01: what influencers see once approved.

## Technical reference

- **Endpoints (admin panel, out of scope for the brand app):** `PATCH /api/v2/admin/campaigns/{id}` → `v2/Admin/AdController@update` → `Services/Implementation/Admin/AdService::update`. The request (`Requests/v2/Admin/Ads/UpdateAdRequest`) takes `approval` in pending/approved/rejected/draft, `rejection_reason` (required when rejected, from settings `campaign_rejection_reason`) and `status` (required). Also `PATCH …/{id}/reactivate` (only approved + expired + not archived: sets active and deletes the time tables) and `PUT …/{id}/details`.
- **Events:**
  - `AdApprovedEvent` → `Listeners/v2/Ad/AdApprovedEventListener` → `Admin/AdService::handlePostAdApproval`: `AdApproval` notification, `TemporaryInvitationSender` job (queue `ads:temp-invite-sender`, temp invitations → `invitations`), Elastic reindex. Also `PlanEnforcement/Listeners/PauseApprovedCampaignOverQuota` → `ApprovedCampaignAdmission::admit` → `SystemCampaignPauser` (`status=paused, paused_by=system`).
  - Rejection: deletes `invitations` where `ad_id` and `status=pending`, destroys temp invitations, `rollbackFeatureUsage(auth()->user(), affiliates_invites, n)`, `AdRejectedEvent` → `AdRejectedEventListener` → `AdRejectedNotification`.
- **Resubmission:** `AdCreationProcess/Steps/InviteOnly.php` and `Steps/Medias.php` (approval rejected/approved → pending, `CampaignSlotGate` first). Legacy `PUT v2/campaigns/{ad}` (`Services/Implementation/AdService::update`) resets a non-approved campaign to pending for non-admins.
- **Suspension:** `Services/Implementation/Admin/BrandSuspensionService::suspendAds` / `restoreSuspendedAds`.
- **Data:** `ads.status`, `ads.approval`, `ads.rejection_reason`, `ads.approved_at`, `ads.paused_by`, `ads.is_owner_suspended`; `invitations`, `temporary_invitations`.
