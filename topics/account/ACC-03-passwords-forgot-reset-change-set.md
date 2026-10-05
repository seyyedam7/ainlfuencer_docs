---
id: ACC-03
title: "Passwords: forgot, reset, change, set"
domain: Account and sign-in
audience: [brand, influencer]
apps: [web, mobile, email]
related: [ACC-01, ACC-02, BRD-03, BRD-04, INT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# ACC-03 · Passwords: forgot, reset, change, set

> Resetting a forgotten password (a link by email on the web, a 5-digit code on mobile), changing the password while logged in, and the first password that brands who installed the Shopify app must create.

## Who and where

- **Brands, forgot password**: web `/forgot-password` ("Forgot password", "Enter your email and we'll send you a reset link.", button "Send me reset link"), reached from "Forgot your password?" on `/login`. The email link opens `/reset/password/<token>` ("Reset password").
- **Influencers, forgot password**: mobile "Forgot Password?" screen (from "Forgot password?" on "Sign In"), then "Verify Your Email" (code), then "Reset Password".
- **Brands, change password**: web `/profile`, section "Password & security" ("Current password", "New password", "Confirm new password", "Change password").
- **Influencers, change password**: mobile Profile → gear → "Account Settings" → "Change Password" ("Current Password", "New Password", "Password Confirmation", "Save").
- **Shopify-installed brands**: forced to `/oauth/set-password` ("Create Password for Your Ainfluencer Account", "Create a Password", "Confirm Password", "Save & Continue").

## How it works

**Web: forgot and reset (brands)**
1. The brand enters their email and clicks "Send me reset link". A pop-up confirms: "An email was just sent to you with a link to reset your password."
2. The email contains a link to `/reset/password/<token>`.
3. The brand enters "New password" and "Re-enter your password" and clicks "Update password". Toast: "Your password changed successfully!" and the brand is sent to `/login`.

**Mobile: forgot and reset (influencers)**
1. The influencer enters their email and taps "Send". A **5-digit code** is emailed.
2. On "Verify Your Email" they type the code. If the email was wrong they can tap the pencil to send the code to another email.
3. On "Reset Password" they enter the new password twice and tap "Reset Password", then log in on "Sign In".

**Change password while logged in**: the user types the current password and the new one twice. A wrong current password shows "These credentials do not match our records." (web) or "It seems the password you entered for this account doesn't match." (mobile).

**Shopify first password**: brands created by installing the Ainfluencer Shopify app have no password. Until they set one, the web app redirects them to `/oauth/set-password`. After "Save & Continue" ("Your password set successfully!") they go to onboarding (BRD-01).

## Rules and limits

- New passwords: at least 8 characters, entered twice. The web strength meter (uppercase, number, special character) is only advice.
- **Mobile reset: the new password must be different from the old one** ("This password should be different from the previous password."). The web reset has no such rule.
- Web reset links and mobile reset codes **don't expire** and each works **once**. Requesting again sends a new one; older unused ones keep working.
- Forgot password on the web, unknown email: "The selected email is invalid." On mobile: "We could not find an account with that email."
- A **brand email on the mobile** forgot screen is refused (the message wrongly says "The email you entered is not registered as a brand on the platform."). Brands must use the web.
- An **influencer email on the web** forgot page works: the influencer gets a reset link and can then log into the app with the new password.
- A **suspended brand** can't reset: the appeal pop-up (BRD-04) or "Your account has been suspended..." appears instead.
- **Team members acting inside another brand's team can't change the password** (the page isn't available to them; see BRD-03). They change it while in their own brand.
- Changing or resetting the password **does not log out other devices** (ACC-02).
- Accounts created with Google have a random password the user doesn't know. They can use "Forgot password" to set one.
- The Shopify "set password" step only works when the account has no password yet.

## Statuses

Not applicable.

## Timers and automatic actions

None. Reset links and codes have no time limit.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Web forgot password | User | Email with reset link to `/reset/password/<token>` |
| Mobile forgot password (or pencil "send to another email") | Influencer | Email with 5-digit reset code |

## Common questions

### "I didn't receive the password reset email."
**Answer for the user:** Please check Spam, Junk and Promotions folders, and make sure you used the email your account was created with. You can request again; the newest email works.
**What to check:** The email exists in `users`. Brand vs influencer: brands must use the web, influencers can use either. Whether the brand is suspended (the reset is blocked).
**Escalate when:** The email is correct and nothing arrives after several tries.

### "The app says my email is 'not registered as a brand'."
**Answer for the user:** That message appears when a brand account tries to reset its password in the influencer app. Please reset your password on the website (app.ainfluencer.com, "Forgot your password?").
**What to check:** `users.type` = brand.
**Escalate when:** The account is actually an influencer.

### "It says my new password must be different."
**Answer for the user:** In the app, the new password can't be the same as your current one. Please choose a different password.
**What to check:** Nothing.
**Escalate when:** Never.

### "I signed up with Google and the app asks for my current password."
**Answer for the user:** Accounts created with Google don't have a password you know. Log out, tap "Forgot password?", and set a new one with the emailed code.
**What to check:** The account was created via Google.
**Escalate when:** Never.

### "Why does the site keep sending me to 'Create Password for Your Ainfluencer Account'?"
**Answer for the user:** Your account was created by installing our Shopify app, so it doesn't have a password yet. Please create one; you'll only need to do this once.
**What to check:** The account came from Shopify (INT-01) and has no password.
**Escalate when:** The page shows an error after saving, or keeps appearing after a password was set (clearing the browser's site data may help).

### "My teammate can't find the change password section."
**Answer for the user:** Passwords can only be changed from your own brand, not while working inside someone else's team. Switch to your own brand with the team switcher, then go to Profile.
**What to check:** The user is currently in another brand's team (BRD-03).
**Escalate when:** Never.

## Known issues

- **Wrong wording on mobile**: a brand email on the mobile forgot screen gets "The email you entered is not registered as a brand on the platform." It should say "not registered as an influencer". Escalate to dev for the text fix.
- **Password change doesn't sign out other devices.** If a user suspects someone else is using their account, escalate so sessions can be ended.
- The old `/profile/change-password` page still exists but nothing links to it; the live form is on `/profile`.

## Related topics

- ACC-01: Google sign-up (no password).
- ACC-02: login and sessions.
- BRD-03: what team members can't do.
- BRD-04: suspended brands.
- INT-01: Shopify app installation.

## Technical reference

- **Endpoints**
  - `POST /api/v2/auth/forgot-password` (`email` must exist) → `UserService::forgot_password` → `app/Mail/PasswordReset.php` (60-char token, link `${WEBSITE_URL}/reset/password/<token>`).
  - `POST /api/v2/auth/reset-password` (`token`, `password` min 8) → `UserService::reset_password` (deletes the used token, returns a session; web then sends the user to `/login`).
  - `POST /api/v2/user/change-password` (`old_password`, `password`, `password_confirmation`; middleware `team.blocker`) → `InformationController@changePassword`.
  - `POST /api/v2/user/set-password` → `Modules/User/.../UserController@setPassword` (fails if a password is already set).
  - Mobile: `POST /api/en/v3/mobile/auth/forgot-password/forgot` → `AuthService::sendForgotPasswordCode` (mail `PasswordResetWithCode`, 5-digit code); `…/verify` (`email`, `code`) returns a temporary session; `…/reset` (`password`, `password_confirmation`) → `resetPasswordWithCode` (must differ; deletes all reset codes and the temporary session); `POST …/change-password`.
- **Data**: `password_resets` (`user_id`, `token` = web link token or 5-digit code; no expiry column used). `users.password`.
- **Web**: `src/components/pages/forgotPassword2`, `resetPassword2`, `userInfo2` (password section), `shopifyCreatePassowrd`; forced redirect in `src/lib/utils/auth.ts` (`isUserNeedToSetShopifyPassword`).
- **Mobile**: `src/features/auth/screens/resetPassword`, `resetPasswordFormScreen`, `verifyEmail` (reset mode), `src/features/profile/screens/ChangePasswordSCreen.tsx`.
