---
id: BRD-03
title: Team members
domain: Brand account and team
audience: [brand]
apps: [web, email]
related: [SUB-02, SUB-04, ACC-02, ACC-03, ACC-05, BRD-02, BRD-04, NTF-03, PUB-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# BRD-03 · Team members

> A brand can invite colleagues to work in its account. Each member keeps their own login and switches into the brand's team from the profile menu. The number of members is limited by the plan's **seats**.

## Who and where

- **Owner (brand)**: web `/team-members` ("Add & manage members"), from the header profile menu or the team switcher. Shows "Members" (marked "Owner" or "Admin") and pending invitations ("Invitation Pending").
- **Invitee**: receives the email "You're invited to {Brand}'s Team on Ainfluencer". The link opens a short check page and then either `/login` (existing account) or `/teams/members` (registration form for new users).
- **Team switcher**: header profile menu, lists the user's own brand and the teams they belong to.

## How it works

**Inviting**
1. The owner clicks "+Add Member", types one or more emails ("+ Add More") and clicks "Send Invite".
2. Each address gets the invitation email. The owner sees "Invitation sent" ("We've emailed an invite to ...") and can "Copy Invitation Link" to share it directly, or "+ Email link to me".
3. Per address, errors are shown instead, for example:
   - "You've already invited "{email}" to your team, we are waiting for them to join the platform."
   - ""{email}" is already your team member."
   - "The email address {email} is associated with an influencer on our platform and cannot be added as a team member."
4. "Resend Invite" sends the invitation again (and restarts its 7-day validity). Deleting a pending invitation frees its seat ("Invitation successfully deleted").

**Accepting**
- **Invitee already has a brand account**: clicking the link adds them to the team immediately and marks their email verified; they're sent to `/login`. After logging in they're in the team.
- **Invitee is new**: the link opens `/teams/members` with the email filled in. They enter first name, last name and password (twice), complete the reCAPTCHA and submit. The account is created already verified and opens in the team.

**Working in a team**
- Members switch between their own brand and the team from the profile menu. While in the team they act **as the brand**: campaigns, inbox and offers, payments, wallet, scheduler, analytics and notifications of the brand.
- Roles are not available yet ("(Roles coming soon)"). Every member is shown as "Admin" and has the same access.

**Removing**
- The owner clicks "Remove Member" ("This action will remove the member and revoke their access to the team."). The member is logged out of all their sessions and loses access ("Member successfully deleted"). The owner can't be removed.

## Rules and limits

- **Seats** come from the plan (SUB-04). The owner doesn't use a seat: a plan sold as "3 seats" allows the owner plus 2 members. Pending invitations also use a seat. Over the limit, the upgrade pop-up appears (extra seats: $19 on Pro, $29 on Ultimate, as shown in the pop-up).
- Invitations to people without an account **expire after 7 days**. An expired link shows "This invitation is no longer valid. Please contact the account owner for further assistance."
- The invitee must register with the **same email** the invitation was sent to.
- Influencer emails can't join a brand team. Suspended or deleted accounts can't accept.
- **What members can't do inside a team** (they are sent to the dashboard or get "This action is not allowed."):
  - open Profile (`/profile`), change the brand's profile or logo;
  - change password or email notification settings (they do this in their own brand);
  - open "Influencer content" (`/influencer-content`), even though the sidebar still shows it.
- If the brand owner is **suspended**, members can't use the team: "This account has been suspended and cannot be accessed at the moment." If the owner **deleted** the account, every member is logged out and pending team invitations expire.
- **Plan downgrade**: when a paid plan ends, extra seats are released automatically: first pending invitations (oldest first), then the oldest members (SUB-02).

## Statuses

| What the owner sees | Meaning (`team_member_invitations.status`) | Next |
|---|---|---|
| "Invitation Pending" | `pending`, not yet accepted | Accept within 7 days, resend, or delete |
| Member listed as "Admin" | `accepted`; user in `team_user` | Can be removed |
| (disappears from list) | `expired` after 7 days | Send a new invite |

## Timers and automatic actions

- Every minute: invitations to emails without an account that are older than 7 days become `expired`.
- Hourly (plan downgrade check): over-quota invitations and members are removed when a paid plan has ended (SUB-02).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Invite / Resend Invite | Invitee | Email "You're invited to {Brand}'s Team on Ainfluencer" |

No email is sent when a member joins or is removed.

## Common questions

### "My teammate didn't get the invitation."
**Answer for the user:** Ask them to check Spam, Promotions and Social tabs. You can also click "Copy Invitation Link" and send the link to them directly.
**What to check:** The invitation exists and is `pending`; the email is spelled correctly.
**Escalate when:** The invite email never arrives for a correct address and the copied link also fails.

### "The invitation link says it's no longer valid."
**Answer for the user:** Invitations expire after 7 days. Please ask the account owner to click "Resend Invite".
**What to check:** `team_member_invitations.status` = `expired` or older than 7 days.
**Escalate when:** A freshly resent link is also invalid.

### "I can't invite more people. It asks me to upgrade."
**Answer for the user:** Your plan includes a limited number of seats, and pending invitations count too. You can delete pending invitations you no longer need, buy extra seats, or upgrade your plan.
**What to check:** Plan seats (SUB-04), number of members and pending invitations. See the known issue below about removed members.
**Escalate when:** The brand is under the limit but still blocked.

### "How do I remove someone from my team?"
**Answer for the user:** Go to Team Members, find the person and click "Remove Member". They lose access immediately.
**What to check:** Nothing.
**Escalate when:** Removal fails.

### "My teammate can't see the Profile / Influencer content page."
**Answer for the user:** Team members can't open the brand's profile, password settings, email settings or Influencer content. Only the owner can.
**What to check:** Nothing.
**Escalate when:** Never.

### "Can I give my teammate limited access, for example no payments?"
**Answer for the user:** Not yet. Roles are coming soon; today every team member has the same access as the owner, except for the profile and settings pages.
**What to check:** Nothing.
**Escalate when:** Never.

### "The registration page gives an error when my teammate creates their account."
**Answer for the user:** Please ask your teammate to sign up as a brand at app.ainfluencer.com/register with the invited email, verify it, and then click the invitation link again. They'll be added to your team right away.
**What to check:** That the invitation email matches the email used.
**Escalate when:** Escalate to dev with the email and the error text (see known issues).

### "I'm an influencer and want to join my brand's team."
**Answer for the user:** An email registered as an influencer can't be a brand team member. Please use a different email for the team.
**What to check:** `users.type` of the invited email.
**Escalate when:** Never.

## Known issues

- **Removing a member may not free up a seat.** The seat counter goes up when an invitation is sent and is only reduced when a pending invitation is deleted or on a plan downgrade, not when a member is removed. A brand at its limit may still get the upgrade pop-up after removing someone. Escalate to dev to correct the count.
- **New invitees may get an error on the registration form** (`/teams/members`): the registration request is checked against plan seats as if a logged-in user sent it *(to verify)*. Workaround: the invitee signs up normally as a brand first, then clicks the invitation link (existing accounts are added directly).
- **Wrong email on the registration form** shows a raw message key instead of "Please use the same email address..." (text bug). Ask the invitee to use the invited email.
- The sidebar still shows "Influencer content" to team members, but the page sends them back to the dashboard.

## Related topics

- SUB-04: seat limits and extra seats.
- SUB-02: what happens to members when a plan ends.
- ACC-02: logging in lands members in their last team.
- ACC-05: deleting the owner account logs out all members.
- BRD-04: a suspended owner blocks the team.
- NTF-03: notification settings (members manage their own).

## Technical reference

- **Endpoints** (`app/Modules/Team/routes/api_brand.php`, prefix `/api/v2/teams/brand/members`): `GET /` (members + invitations), `POST /invite` (`feature:seats`, body `members[]`) → `MemberInvitationController@invite` → `Brand/Services/Implementation/MemberInvitationService::invite`; `DELETE /invitations/{id}` (rolls seat usage back by 1); `POST /register` (`feature:seats`) → `TeamAuthService::register`; `DELETE /{memberId}` → `TeamService::removeMemberFromTeam/detachMember`.
- **Invite link**: web route `GET /teams/brand/invitation?token=` → `MemberInvitationCallbackController@handle` → `followUpInvitation` (existing users added immediately) → redirect `${WEBSITE_URL}/teams/members?is_registered=0|1&email=&token=`.
- **Team context**: the web sends header `X-Team-Id` (team JWT); `Brand/Http/Middlewares/TeamIdentifier.php` checks membership and owner status and then acts as the owner. `TeamActivityBlocker` (`team.blocker`) blocks profile, password and brand-information updates.
- **Data**: `teams` (`owner_id`), `team_user` (members, including the owner), `team_member_invitations` (`email`, `status` pending/accepted/expired, `invited_user_id`, `expired_at`), `users.last_team_id`, `feature_usages` (seat counter).
- **Jobs**: `teams-brand:expire-member-invitations` (every minute, `TEAM_INVITATION_EXPIRATION` default 604800 s = 7 days); `plans:detect-downgrades` → `Modules/PlanEnforcement/Policies/SeatQuotaPolicy.php`.
- **Web**: `src/components/pages/teamMembersManagement`, `memberRegisteration`, team switcher `src/components/layouts/dashboard/header/parts/profile/parts/teams`. Strings `locales/en/teamMembers.json`. Lang `resources/lang/en/modules/team.php`.
