---
id: BRD-04
title: Brand suspension and appeal
domain: Brand account and team
audience: [brand, admin, influencer]
apps: [web, email]
related: [ACC-02, ACC-03, BRD-03, CMP-02, CMP-03, DSC-03, CHAT-01, CHAT-04, INF-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# BRD-04 · Brand suspension and appeal

> Ainfluencer staff can suspend a brand (temporarily or permanently), usually for breaking the community rules (for example asking influencers for contact details or off-platform payment). A suspended brand is logged out, its live campaigns leave the marketplace, and it can send **one appeal** from the login page. Staff approve or reject the appeal.

## Who and where

- **Admin**: suspends, reactivates and reviews appeals in the admin panel. There are **no automatic suspension rules for brands** (automatic rules exist only for influencer accounts, INF-04).
- **Brand**: sees the appeal pop-up on the web when trying to log in (`/login`), sign up (`/register`), use Google login, or reset the password (`/forgot-password`).
- **Influencers**: see the brand's campaigns disappear from the marketplace and can't message the brand in the app chat.

## How it works

**Suspension (admin)**
1. An admin sets the brand to **temporarily suspended** or **permanently suspended** and selects one or more reasons (with an optional note).
2. Immediately:
   - the brand and anyone logged into it are **logged out**;
   - every campaign of the brand is flagged as owner-suspended, and **live campaigns** (active and approved) are taken off the marketplace: they show as rejected with the reason **"Owner suspended"**;
   - the brand's **pending campaign invitations** to influencers are withdrawn;
   - team members can no longer enter the team ("This account has been suspended and cannot be accessed at the moment...").
3. The brand receives the email **"Action Required: Your account on Ainfluencer has been temporarily suspended"** (or "permanently suspended") listing the reasons. If the reason "Additional Verification Required" was chosen, a second email **"Important: Additional Verification Required for Your Account"** is sent.

**Appeal (brand, web)**
1. At login, the pop-up **"Account Suspended!"** shows "Reason for Suspension:" and **"Appeal Suspension"**.
2. Step 2 ("Account Suspended"): the email is filled in and locked; the brand writes the **"Reason for Appeal:"** and clicks "Next".
3. Step 3 ("Appeal: Don'ts to Follow"): the brand ticks "I acknowledge that I have read and understood all the rules mentioned above..." and clicks "Submit Appeal".
4. Step 4 ("Appeal: Dos & Don'ts to Watch"): a video; clicking **"Confirm Watched"** actually sends the appeal. Then "Appeal Submitted!" / "Your request is pending admin review."
5. Closing the pop-up before the last step shows "Warning: Unfinished Appeal Process" and **nothing is sent**.

**Decision (admin)**
- **Approved** → the brand is reactivated (same as an admin setting it back to active).
- **Rejected** → nothing else happens; at the next login the brand sees "Your account has been suspended due to violating our community guidelines. If you have any questions, please contact our support team." No email is sent.

**Reactivation** (appeal approved, or admin sets the brand back to active):
- the suspension record (and its appeal) is cleared;
- campaigns taken down with "Owner suspended" go **back to admin review** (pending approval), not straight to live (CMP-02);
- the withdrawn pending invitations are restored;
- the brand receives **"Your account has been reactivated - Ainfluencer Marketplace"**.

## Rules and limits

- Temporary and permanent suspensions behave the same in the product. A "temporary" suspension **does not end by itself**; an admin must reactivate the brand.
- **One appeal per suspension.** A second attempt: "You have previously submitted an appeal."
- While an appeal is pending, login shows: "You have submitted an appeal request and are now waiting for admin approval. Until your request is approved, you won't be able to log in."
- A suspended brand can't log in, sign up again with the same email, use Google login, or reset the password.
- Only campaigns that were live at suspension are restored to review afterwards; drafts and paused campaigns are just un-flagged.
- An admin can suspend only brands that have started onboarding (business name saved, BRD-01).
- Money in the brand's wallet and paid deals are not touched by the suspension itself.

## Statuses

| What support sees | Meaning | Next |
|---|---|---|
| `brand_details.status` = `active` | Normal | |
| `temporary suspend` / `permanent suspend` | Suspended; brand can't log in | Appeal or admin reactivation |
| `suspension_appeals.status` = `pending` | Appeal sent, waiting for staff | Approve or disapprove |
| `approved` | Brand reactivated | Campaigns back in review |
| `disapproved` | Rejected; no new appeal possible | Only staff can reactivate |
| Campaign `rejection_reason` = "Owner suspended" | Taken down by the suspension | Back to pending review on reactivation |

## Timers and automatic actions

None. Suspensions never expire automatically, and there are no automatic brand suspension rules.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Brand suspended | Brand | Email "Action Required: Your account on Ainfluencer has been {temporarily/permanently} suspended" |
| Reason "Additional Verification Required" selected | Brand | Extra email "Important: Additional Verification Required for Your Account" |
| Brand reactivated (incl. appeal approved) | Brand | Email "Your account has been reactivated - Ainfluencer Marketplace" |
| Appeal submitted / rejected | — | No notification |

## Common questions

### "Why was my account suspended?"
**Answer for the user:** The reasons are listed in the suspension email and in the pop-up when you try to log in ("Reason for Suspension"). Most suspensions relate to our rules, for example asking influencers for contact details or payment outside Ainfluencer before a deal is paid. You can appeal from the login page.
**What to check:** Suspension reasons and note in the admin panel (brand suspension history).
**Escalate when:** The brand disputes the facts; the decision belongs to the trust/admin team.

### "How do I appeal?"
**Answer for the user:** Try to log in on app.ainfluencer.com. In the pop-up, click "Appeal Suspension", explain your case, accept the rules, watch the short video and click "Confirm Watched". If you close the pop-up before the end, the appeal isn't sent.
**What to check:** Whether an appeal already exists (`suspension_appeals`).
**Escalate when:** The pop-up doesn't appear for a suspended brand.

### "I sent an appeal. How long does it take?"
**Answer for the user:** Appeals are reviewed by our team. There is no fixed time; you'll get an email if your account is reactivated.
**What to check:** Appeal status is `pending`.
**Escalate when:** The appeal has been pending for a long time (ask the admin team to review).

### "My appeal was rejected. Can I appeal again?"
**Answer for the user:** Only one appeal is possible per suspension. If you have new information, send it to us and we'll pass it to the team.
**What to check:** Appeal status `disapproved`.
**Escalate when:** The user provides new information (admin decision).

### "My account is active again, but my campaigns are not live."
**Answer for the user:** After reactivation, your campaigns go back to our review queue before they're published again. This usually happens without any action from you.
**What to check:** Campaigns with `approval = pending` after reactivation (CMP-02).
**Escalate when:** Campaigns stay in review unusually long.

### "The brand I'm working with disappeared / I can't message them." (influencer)
**Answer for the user:** The brand's account is currently restricted, so its campaigns and chat are unavailable. If you have a paid deal in progress, please contact us so we can check it.
**What to check:** Brand `brand_details.status`; any paid offers or publish jobs with that brand (PUB-01, PUB-03).
**Escalate when:** There is money in escrow or a deal in progress with the suspended brand.

### "My team member can't access our team." (team member of a suspended brand)
**Answer for the user:** The brand account is suspended, so its team is unavailable until it is reactivated.
**What to check:** Owner's suspension status.
**Escalate when:** The owner isn't suspended.

## Known issues

- Rejected appeals aren't announced by email; the brand only finds out at the next login.
- The rejection and register-page messages contain a misspelled support address ("suuport@ainfluencer.com"). Tell users to write to the correct support address.
- Staff may suspend from a separate admin system that writes to the database directly. If side effects (campaigns taken down, invitations withdrawn, logout, emails) are missing after a suspension or reactivation, escalate to dev *(to verify which admin tool is used)*.

## Related topics

- ACC-02 / ACC-03: where the appeal pop-up appears.
- BRD-03: team members of a suspended brand.
- CMP-02: campaigns going back to review after reactivation.
- DSC-03: campaign invitations withdrawn and restored.
- CHAT-01 / CHAT-04: chat and contact-sharing detection, the usual cause of suspensions.
- INF-04: influencer suspensions (different rules, including automatic ones).

## Technical reference

- **Endpoints**: `POST /api/v2/brand/suspension-appeal` (`email`, `description`) → `v2/Frontend/SuspensionAppealController@storeBrandAppeal` → `Services/Implementation/SuspensionAppealService` (one appeal per active suspension). Admin: `PUT /api/v2/admin/identities/{id}/status` → `Services/Implementation/Admin/BrandSuspensionService::update`; `PUT /api/v2/admin/identities/{id}/suspension-appeal` → `Admin/SuspensionAppealService::updateBrandAppeal` (approved → `BrandSuspensionService::update(status = active)`); `GET …/{id}/suspensions` (history).
- **Login check**: `Modules/User/Login/Services/UserSuspensionService::inspectSuspension` (throws `SuspendedException` → 403 with `is_appealable`, `reasons`, `other_reason_note`; `HasPendingAppealException`; `DisapprovedAppealException`). Used by web login, register, Google login and forgot password.
- **Side effects** (`BrandSuspensionService`): `suspendAds` (`ads.is_owner_suspended = 1`; active+approved → `status = approval`, `approval = rejected`, `rejection_reason = 'Owner suspended'`; Elastic update job `AfterBrandSuspendedOrActivated`), `deletePendingInvitations` (soft delete), `revokeBrandAccess` (delete `tokens`); reactivation reverses these (`approval = pending`).
- **Emails**: `Listeners/v2/Identity/BrandStatusChangeListener` → `Notifications/BrandStatusChangeNotification` (views `emails.v2.brand.suspension.active_suspended` / `suspended_activated`) and `BrandAdditionalVerificationNotification`.
- **Data**: `brand_details.status`, `brand_suspension_details` (`reason_ids`, `other_reason_note`, `suspender_id`), `suspension_appeals` (`status` pending/approved/disapproved, `description`, `reviewer_id`), `ads.is_owner_suspended`, `ads.rejection_reason`.
- **Web**: `src/components/common/modals/appealSuspension` (4 steps; submit on "Confirm Watched"); strings `locales/en/common.json` → `modals.appealSuspensionModal`.
