---
id: AFA-11
title: Teams (affiliate side)
domain: Affiliate app (creators)
audience: [affiliate]
apps: [affiliate-app]
related: [AFS-13, AFA-09, AFX-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-11 · Teams (affiliate side)

> In the affiliate app, a person who belongs to more than one team can switch between them. Affiliates can't create or manage teams in the app; team management exists only in the seller app.

## Who and where
- **Affiliates** who are members of a team, in the affiliate app sidebar: a **"Switch Teams"** drop-down listing their teams (a crown icon marks the team they own).
- Team management ("Team" tab in Settings) is **seller-only** (AFS-13). Affiliates don't see it.

## How it works
1. The "Switch Teams" block appears only if the user belongs to more than one team, or to one team that isn't their own.
2. Clicking a team switches the session into that team (the page reloads with a new login token for that team).
3. Inside another team, what the user can do depends on the permissions the team owner gave them. Pages or buttons they don't have rights to are hidden or blocked.
4. Clicking their own team (crown) switches back.

## Rules and limits
- Affiliates can't invite members or manage roles from the affiliate app.
- The affiliate plans include a "team members" value (Pro 10, Ultimate unlimited), but there's no screen for affiliates to use it today.
- Inside a team, for example, the Stripe "Billing" button needs the matching permission (AFA-08).

## Statuses
Team invitations (seller side): pending, accepted, rejected (AFS-13).

## Timers and automatic actions
None.

## Notifications
Team invitations are emailed by the seller app (AFS-13).

## Common questions

### "How do I add team members to my affiliate account?"
**Answer for the user:** Team management isn't available in the affiliate app at the moment. It's only offered for seller accounts.
**What to check:** —
**Escalate when:** The customer is on a paid affiliate plan and expected team seats. Forward to product/sales.

### "I don't see 'Switch Teams'."
**Answer for the user:** It only appears when you belong to another team besides your own. Ask the team owner to invite you, then accept the invitation.
**What to check:** The user's team memberships.
**Escalate when:** The user accepted an invitation but the switch doesn't appear after logging out and in.

### "I switched team and can't see some pages."
**Answer for the user:** In someone else's team you only have the permissions the owner gave you. Ask the team owner, or switch back to your own team.
**What to check:** The member's role and permissions.
**Escalate when:** —

## Known issues
- Affiliate team management isn't available even though affiliate plans list team members.
- Hidden team-management code in the affiliate app points at addresses that don't exist, so it would fail if it were ever shown.

## Related topics
- AFS-13 · Seller teams and roles (where teams are managed).
- AFA-09 · Affiliate plans.
- AFX-01 · Switching between seller and affiliate profiles (a different switch).

## Technical reference
- **Endpoint:** `GET affiliate/v1/teams/{teamId}/impersonate` (`GetImpersonationTokenController`, returns a new token).
- Team list from the auth status (`teams`, `member.teams`, `member.current_team_id`).
- **Frontend:** `shared-components/algo-sidebar/sidebar-nav.vue` ("Switch Teams"), `shared-components/store/settings.store.js::switchTeam` (`VUE_APP_SWITCH_TEAM_IMPERSONATE`). Settings Team tab hidden unless `AppInfo.isSeller` (`shared-components/algo-setting/algo-setting.page.vue`).
- **Data:** teams, team members (`status` 0 pending, 1 accepted, 2 rejected), team permissions.
