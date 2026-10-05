---
id: AFC-04
title: Settings and security
domain: Shared by seller and affiliate apps
audience: [seller, affiliate]
apps: [seller-app, affiliate-app, email]
related: [AFS-01, AFA-01, AFS-13, AFC-02, AFS-10, AFA-08]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFC-04 · Settings and security

> The Security tab of Account settings lets sellers and affiliates reset their password by email and see and end their active logins ("sessions"). This topic also covers the small background settings the apps load on every page (UI preferences, page help panels).

## Who and where
- **Account settings** (user menu "Account settings", `/app/settings`). Tabs: "Team" (sellers only, AFS-13), "Message Templates" (affiliate chat, not supported, AFC-01), "Billing" (AFS-10 / AFA-08), "Notifications" (AFC-02), "Security".
- **Security tab** ("Secure Your Profile"):
  - "RESET PASSWORD (Need to verify)" with a "Reset Password" button.
  - "Active Sessions" list: "Device(OS)", "IP", "Location", "Last Activity", the current one marked "Current Device", a "Terminate" button per row and "Terminate All Except Me".
- Team members see each tab only if their role allows it (AFS-13).

## How it works
**Reset password (while logged in)**
1. Click "Reset Password" → pop-up "Reset Your Password?" ("We’ll send a password reset link to your registered email … address.") → confirm.
2. "Check Your Inbox": an email with a reset link goes to the account's primary email right away.
3. The link opens `/user/reset-password/{token}` in the same app (seller or affiliate) where the new password is set.

**Sessions**
1. The list shows every device/browser currently logged in to this profile (seller or affiliate profile separately).
2. "Terminate" logs that device out (its next request fails and it returns to login).
3. "Terminate All Except Me" logs out all other devices (**seller app only**, see Known issues).

**Background settings (no user action)**
- On every page the app loads: basic system info (lists of statuses and options), the user's UI preferences (for example whether a welcome pop-up was already shown), and the help-panel text for the current page ("guidelines").

## Rules and limits
- Reset link **expires after 5 hours** (`RESET_PASSWORD_EXPIRATION_TIME`).
- A reset email can be requested **once per minute**; a quicker second click gets a "too many requests" error.
- The link always goes to the primary email of the login; a user who signs in only through Ainfluencer (single sign-on, AFX-01) can still set a password this way.
- Sessions are per profile: the same person's seller and affiliate profiles have separate session lists.
- Ending a session doesn't change the password; after a suspected compromise, reset the password **and** terminate other sessions.

## Common questions

### "I didn't receive the password reset email."
**Answer for the user:** Please check spam/junk and wait a minute before trying again (one request per minute). The link is valid for 5 hours.
**What to check:** the account's primary email (`user_emails`, primary); a recent row in `reset_password_tokens` with `expired_at` in the future.
**Escalate when:** a token was created but no email arrives after several tries.

### "The reset link says it's expired or invalid."
**Answer for the user:** Reset links work for 5 hours and only the latest one is valid. Please request a new one from Account settings → Security (or "Forgot password" on the login page).
**What to check:** `reset_password_tokens.expired_at`.
**Escalate when:** a fresh link fails immediately.

### "I see a device I don't recognise in Active Sessions."
**Answer for the user:** Click "Terminate" on that row, then reset your password from the same page to be safe.
**What to check:** `user_active_sessions` for the profile (IP, user agent, created time).
**Escalate when:** the user reports actions they didn't do (security team).

### "'Terminate All Except Me' does nothing in the affiliate app."
**Answer for the user:** That button doesn't work in the affiliate app right now. Please use "Terminate" on each other session instead.
**What to check:** —
**Escalate when:** individual "Terminate" also fails.

### "Why can't my team member see the Security tab?"
**Answer for the user:** Each settings tab follows the member's role permissions; the account owner can add the security permission in Account settings → Team → Your Team Roles.
**What to check:** role permissions (AFS-13).
**Escalate when:** the permission is set and the tab or its actions still fail.

## Known issues
- **"Terminate All Except Me" doesn't work in the affiliate app** (the button has no server address configured). It works in the seller app.
- **"Terminate All Except Me" may end only the first 15 other sessions** in one click *(to verify)*; click again if some remain.
- **Team members with the security permission may still be refused** on session/reset actions, because the server checks a differently named permission than the role editor offers *(to verify)*.

## Related topics
- AFS-01 / AFA-01 — sign-up, login and "forgot password" on the login page.
- AFS-13 — team roles that control which settings tabs a member sees.
- AFC-02 — notification settings.
- AFS-10 / AFA-08 — Billing tab.
- AFC-01 — message templates belong to the unsupported affiliate chat.

## Technical reference
- **Endpoints:** POST `{seller|affiliate}/v1/profile/request-change-password-by-email` (throttle 1/min) → `RequestChangePasswordByEmailService` → reset service (`reset_password_tokens`, link `{SELLER|AFFILIATE}_BASE_URL/user/reset-password/{token}`, `config('sanctum.reset_password.expiration_time')` hours); GET `{seller|affiliate}/v1/profile/sessions`, DELETE `/sessions/{sessionId}`, DELETE `/sessions/all` (route exists in both apps; the affiliate front end lacks `VUE_APP_ALGO_PROFILE_DELETE_ALL_SESSIONS`) → `app/Services/General/Profile/Session.php` (ending a session expires its login).
- **Background:** GET `{seller|affiliate}/v1/system/basic-info` (enums with labels, `app/Services/General/System/System.php`), GET/POST `{seller|affiliate}/v1/settings/client[/store]` (UI preferences), GET `{seller|affiliate}/v1/guidelines/{key}` (help panel HTML, managed by admins).
- **Seller route permissions:** `impersonation_permission:seller.edit_profile.security.view|update` on sessions and reset.
- **Data:** `user_active_sessions` (`user_type_id`, `token_id`, IP, user agent, location), `personal_access_tokens.expires_at`, `reset_password_tokens` (`token` hashed, `expired_at`), user settings table for client preferences, guideline pages.
- **UI:** `shared-components/algo-setting/algo-setting.page.vue`, `tabs/Security.vue`, `tabs/active-session-*`, `modals/reset-password-*.vue`.
