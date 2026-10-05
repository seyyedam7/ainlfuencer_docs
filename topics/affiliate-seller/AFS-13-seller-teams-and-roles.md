---
id: AFS-13
title: Seller teams and roles
domain: Seller app (brands)
audience: [seller]
apps: [seller-app, email]
related: [AFS-12, AFA-11, AFC-04, AFS-08, AFS-09, AFS-10, AFS-11]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-13 · Seller teams and roles

> Every seller account has its own team. The owner can invite other sellers by email, give each one a role (a set of permissions), and those members can then switch into the owner's account from the sidebar and work in it with only the access their role allows.

## Who and where
- **Team owner** (any seller): **Account settings → Team** (`/app/settings?tab=team`). Sections "Team Members" (heading "Your Team Members"), "Invited People", "Your Team Roles", and "Invited Teams" (invitations the seller received from other teams).
- **Role editor:** "Create New Role" → `/app/role-permission/:teamId/create`, edit at `/app/role-permission/:teamId/edit/:roleId`.
- **Team member:** accepts the invitation in **Account settings → Team → Invited Teams** ("Accept" / "Reject"), then uses **"Switch Teams"** in the sidebar (shown when the user belongs to more than one team).
- The Team tab exists only in the seller app. Affiliates can switch into teams but can't manage them (AFA-11).

## How it works
1. **Team created automatically** when the seller registers (named "{name}'s Team").
2. **Roles.** In "Your Team Roles" the owner creates a role, names it and ticks permissions per area (view / create / update / delete / export, depending on the area). Areas: brands (all brands or chosen brands only), products, affiliates (incl. Expanded Network), promotions, partnership, messages, reports, payments (subscription statements, commission statements), email campaigns, help center, wallet and wallet history, edit profile (information, "get better profile", accounts), settings (team, security, message templates, notifications, credit card/Stripe), lists.
3. **Invite.** "Send New Invitation" → "Email", "Role" ("Member Access Limits") → "Send Invitation". Message: "Invitation has been sent successfully". An email is sent right away to that address with a link to the seller login page:
   - if the email already has a **seller** account, the invitation appears in their "Invited Teams";
   - if not, the email asks them to register (see Known issues).
4. **Accept.** The invitee logs in, opens Account settings → Team → Invited Teams and clicks "Accept" ("Are you sure to accept the invitation?") or "Reject".
5. **Work in the team.** The member picks the owner's team in the sidebar "Switch Teams". The page reloads into the owner's account (other open tabs reload too). Menu items the role doesn't allow are hidden; opening a page without permission sends the member back to the dashboard without a message; a blocked action on the server answers "You don't have permission to do this action".
6. **Change or remove.** The owner can "Change Role", "Remove" a member ("Team member has been removed successfully"), or "Cancel Invitation" for pending ones ("Invitation has been canceled").

## Rules and limits
- Only **sellers** can be invited to a seller team ("You cannot invite a member with a different type"). The owner can't invite themselves ("You cannot invite owner of the team"). Someone already pending or accepted can't be invited again ("User is already a member of this team.").
- A role that still has members can't be deleted ("Team role has assigned users").
- Only pending invitations can be cancelled ("You cannot cancel a team member that is not pending"); the owner can't be removed.
- **Plan limit on team members** exists in the plans (seeded: Free 0, Starter 2, Pro 3, Ultimate unlimited, AFS-12) but is **not enforced** by the system today.
- A member's access inside another team **lasts 1 day** at a time; after that they switch again from the sidebar.
- The account owner always has full access.
- Actions done by a member are done in the owner's account (charges, partnerships, messages are the owner's).

## Statuses
| Invitation status | Meaning | Next |
|---|---|---|
| "Pending" (`0`) | Invitation sent, not answered. | Invitee accepts/rejects, or owner cancels. |
| "Accepted" (`1`) | Member of the team. | Can switch into the team. |
| "Rejected" (`2`) | Invitee declined. | Owner can invite again. |

## Timers and automatic actions
- Team access for a member expires after 1 day (`TEAM_IMPERSONATION_EXPIRATION_DAYS`, default 1); they switch again.
- No scheduled jobs.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Owner sends an invitation | Invitee's email address | Email (sent immediately) |
| Invitation accepted / rejected / member removed | — | None |

## Common questions

### "I invited a colleague but they can't see the invitation."
**Answer for the user:** Your colleague needs a seller account with the same email address. Ask them to sign up at seller.ainfluencer.com (or open the affiliate area from Ainfluencer), then send the invitation again; it will appear under Account settings → Team → Invited Teams.
**What to check:** `team_members` row: `email`, `user_type_id` (empty = no seller account at invite time), `status`.
**Escalate when:** the invitee had a seller account with that email when invited and still sees nothing.

### "My team member keeps getting sent back to the dashboard."
**Answer for the user:** Their role doesn't include that page. Edit the role in Account settings → Team → Your Team Roles and give the needed permission.
**What to check:** the member's `team_role_id` and the role's permissions.
**Escalate when:** the permission is ticked and they're still redirected.

### "How many team members can I add?"
**Answer for the user:** Your plan lists a number of seats; the system doesn't block extra invitations at the moment.
**What to check:** plan (AFS-12).
**Escalate when:** —

### "My team member was logged out of my account."
**Answer for the user:** Access to another team lasts one day at a time. They can switch back in from "Switch Teams" in the sidebar.
**What to check:** membership `status` = accepted.
**Escalate when:** switching fails with "User is not a member of this team."

### "I don't see the Switch Teams box."
**Answer for the user:** It appears once you've accepted at least one invitation from another team. Check Account settings → Team → Invited Teams.
**What to check:** `team_members` rows for the user with `status` = 1.
**Escalate when:** an accepted membership exists and the box is missing.

## Known issues
- **Invitations to people without a seller account are never linked** after they sign up (the code that would do it is never called), and such pending invitations don't show in the owner's lists. Workaround: invitee creates a seller account first, owner invites again.
- **Team-member plan limit is not enforced.**
- **Silent redirect** to the dashboard when a member opens a page without permission.

## Related topics
- AFS-12 — plan seats (not enforced).
- AFA-11 — affiliates switching into teams.
- AFC-04 — settings and security (sessions).
- AFS-08, AFS-09, AFS-10, AFS-11 — areas controlled by role permissions.

## Technical reference
- **Endpoints:** `seller/v1/teams/my`, `/{teamId}`, POST `/{teamId}/invitation`, GET `/invitations/list`, PATCH `/invitations/{invitationId}/change-status`, GET `/{teamId}/impersonate` (switch team), `/{teamId}/members/list|update/{id}|delete/{id}|cancel/{id}`, `/{teamId}/roles/list|detail|create|update|delete`, `/permissions/list/{roleId?}` → `app/Http/Controllers/Seller/V1/Team/*` → `app/Services/General/Team/*` (`TeamInvitationActions/SendTeamInvitationService`, `TeamInvitation::changeStatus`, `GenerateImpersonationTokenService`).
- **Permission enforcement:** route middleware `impersonation_permission:seller.<area>.<resource>.<action>` (`app/Http/Middleware/ImpersonationPermissionMiddleware.php`); front-end `$can()` and router guard redirect to `/app`.
- **Emails:** `app/Notifications/Team/SendInvitationTeamNotification.php` (existing seller), `SendInvitationTeamWithRegistrationNotification.php` (no account).
- **Data:** `teams` (`user_id`, `user_type_id` owner), `team_members` (`team_id`, `email`, `user_type_id`, `status` 0/1/2, `team_role_id`), `team_roles`, role permissions tables. `Team::AcceptPendingTeamMember` exists but has no caller. Team-member limit checker `TeamMemberInvitationFeatureChecker` is not wired to any route.
- **Config:** `config/sanctum.php` `impersonation_expiration` (days).
- **UI:** `shared-components/algo-setting/tabs/my-team/*`, `spa-seller/src/components/role-permission/*`, sidebar `shared-components/algo-sidebar/sidebar-nav.vue`.
