---
id: ACC-04
title: Change email
domain: Account and sign-in
audience: [brand, influencer, admin]
apps: [web, mobile]
related: [ACC-01, ACC-03, ACC-05, BRD-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# ACC-04 · Change email

> **Users can't change their own email address today**, in either app, even before verification. Every self-service attempt is refused with a message asking them to contact support. Support asks Ainfluencer staff to change it in the admin panel.

## Who and where

- **Influencers, mobile**: on "Verify Your Email" there is a pencil next to the address, which opens "Change Email" ("Enter your new email address and we'll send you a verification code.", button "Send Verification Code").
- **Influencers, mobile, logged in**: Profile → name header → "Edit Account Information" shows an editable "EMAIL" field. Saving a different email is refused with the same message.
- **Brands, web**: the "Email Address" field on `/profile` is greyed out. There is also an old `/change-email` page, but nothing links to it.
- **Staff**: change the address from the admin panel (user edit), and can mark an account as verified.

## How it works

1. **Mobile, during sign-up**: the influencer taps the pencil, types a new email and taps "Send Verification Code". The app shows the error: **"The email cannot be changed here. Please submit a support ticket and an administrator will update it for you."** Nothing changes.
2. **Mobile, during "Forgot password"**: the same "Change Email" screen is used, but here it simply sends the password-reset code to the email typed. This works (ACC-03). It doesn't change the account's email.
3. **Web**: the profile page doesn't allow editing the email. The unlinked `/change-email` page gets the same refusal.
4. **Support path**: the user contacts support with the current (or mistyped) email and the correct one. Staff update the email in the admin panel. If the user was stuck before verification, staff can also mark the account as verified so the user can log in straight away.

## Rules and limits

- Self-service email change is blocked for brands and influencers on every screen and API (only admins can change it).
- The new email must not already belong to another account.
- Team invitations are tied to the exact email they were sent to: an invited member must register with that email (BRD-03).
- Saving the profile with the same email is fine; only a different value is refused.

## Statuses

Not applicable.

## Timers and automatic actions

None.

## Notifications

None for a self-service attempt. *(To verify: whether staff changing the email in the admin panel sends any email to the user. The core admin endpoint just saves the new address.)*

## Common questions

### "I typed my email wrong when I signed up and the code goes to the wrong address."
**Answer for the user:** No problem, we'll correct it for you. Please send us the email you typed and the correct email address.
**What to check:** Find the account by the mistyped email; confirm it's unverified (`users.email_verified_at` empty) and that the correct email isn't already used by another account.
**Escalate when:** Always: staff change the email (and can verify the account) in the admin panel.

### "How do I change my email address?"
**Answer for the user:** For security, email changes are done by our team. Please tell us your current email and the new one, and we'll update it.
**What to check:** The new email isn't already registered. For brands with team members or a Shopify connection, mention that logins will use the new email.
**Escalate when:** Always (admin action).

### "The app says 'The email cannot be changed here. Please submit a support ticket'."
**Answer for the user:** That's expected: email changes are handled by our team. Send us the correct email and we'll update it.
**What to check:** Same as above.
**Escalate when:** Always (admin action).

### "I want to use a different email for my affiliate account."
**Answer for the user:** The affiliate platform matches accounts by email. If you change your Ainfluencer email, please tell us so we can check the link between the two accounts.
**What to check:** See AFX-01 (accounts are matched by email; different emails create separate accounts).
**Escalate when:** The user ends up with two affiliate accounts.

## Known issues

- **The mobile "Change Email" button looks like it should work but always fails after sign-up.** Users stuck on the verify screen with a wrong email must contact support. Escalate to dev if the screen should be hidden instead.
- The web `/change-email` page still exists and is unusable; nothing links to it.

## Related topics

- ACC-01: verification codes.
- ACC-03: the same screen used during password reset.
- ACC-05: profile details that users can change.
- BRD-03: team invitations are tied to an email.
- AFX-01: affiliate accounts matched by email.

## Technical reference

- **Rule**: `app/Rules/ImmutableEmail.php`: passes only if the value equals the current email (or the caller is an admin); message "The :attribute cannot be changed here. Please submit a support ticket and an administrator will update it for you." Used in `Modules/Mobile/Http/Requests/Auth/Email/ChangeEmailRequest.php`, `Modules/Mobile/Http/Requests/User/UpdateRequest.php`, `Http/Requests/v2/User/UpdateProfileRequest.php`, `Http/Requests/v3/Brand/Profile/UpdateRequest.php`, `Http/Requests/v2/UpdateBrandInformation.php`, `Http/Requests/UpdateUser.php`.
- **Endpoints**: `POST /api/en/v3/mobile/change-email` → `Mobile/.../Auth/AuthService::changeEmail` (never reached with a different email); `PUT /api/v2/user/profile` → `InformationController@update`. Admin: `PUT /api/v2/admin/users/{id}` (update) and `PUT /api/v2/admin/users/{id}/verify`.
- **Data**: `users.email` (unique), `users.email_verified_at`.
- **Mobile**: `src/features/auth/screens/changeEmail/ChnageEmailScreen.tsx` (`source === 'reset-pass'` → resend reset code; otherwise calls change-email). **Web**: `src/components/pages/changeEmail` (unlinked).
