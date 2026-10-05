---
id: ACC-02
title: Login, social login and sessions
domain: Account and sign-in
audience: [brand, influencer]
apps: [web, mobile, email]
related: [ACC-01, ACC-03, ACC-05, BRD-01, BRD-03, BRD-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# ACC-02 · Login, social login and sessions

> How brands log into the web app and influencers log into the mobile app, with email and password or Google; what happens to unverified, suspended or deleted accounts; one-click login links; staying logged in; and the forced app update on mobile.

## Who and where

- **Brands**: web app `/login` (title "Login | Ainfluencer"): "Email", "Password", reCAPTCHA, "Log in", "Log in with Google", "Forgot your password?", "Don't have an account yet? Join Ainfluencer".
- **Influencers**: mobile app "Sign In" screen: "Email", "Password", "Sign In", "Continue with Google", "Forgot password?", "Don't have an account Sign Up".
- **One-click login links**: `/tmp-link?token=...` on the web. These come from the security-alert email's "Go to My Panel" button and from staff.
- **App start (mobile)**: the splash screen checks for app updates before anything else.

## How it works

**Web (brands)**
1. The brand enters email and password, ticks the reCAPTCHA and clicks "Log in". Without the CAPTCHA: "You must complete the CAPTCHA verification before proceeding."
2. What happens next depends on the account:
   - **Email not verified** → "Please verify your email now." and the brand goes to `/verify`; a new code is emailed (ACC-01).
   - **Influencer account** → a "Warning" pop-up: "You are signed up as an influencer. The web platform is only for brands and businesses." with a link to download the app. (With Google, influencers are sent to the `/influencer-landing` page instead.)
   - **Suspended brand** → the "Account Suspended!" appeal pop-up (BRD-04).
   - **Deleted account** → "This account has been permanently deleted and can no longer be used to log in or sign up."
   - **Wrong password** → "It seems the password you entered for this account doesn't match." Unknown email → "The email or password you entered is incorrect."
   - Otherwise → "You are logged in now!" and the app checks onboarding: unfinished → `/onboarding` (BRD-01), finished → the dashboard `/`.
3. If the user is a **member of another brand's team**, they land in the team they used last (BRD-03).

**Google on the web**: "Log in with Google" logs in an existing account with that email, or creates a new verified brand account if the email is new (ACC-01).

**Mobile (influencers)**
1. Email and password → "Sign In". Verified accounts open the main app (Explore). Unverified accounts get a new code and the "Verify Your Email" screen.
2. "Continue with Google": existing account → main app (and an unverified email becomes verified automatically); new email → a new verified influencer account and the onboarding flow.
3. Brand, admin, suspended or deleted accounts are refused (for a brand: "You are signed up as a brand. The mobile application is only for influencers.").

**One-click link (`/tmp-link`)**: opening the link logs the user in without a password and runs the onboarding check. A link works **once** and expires **60 minutes** after it was created.

**Mobile forced update**: on every start the app compares its version with the store. If a newer store version exists, a full-screen update message appears ("A new and improved version is now available! ...") with **no skip option**: the user must update from the App Store or Google Play. Smaller over-the-air updates install in the background and may ask the user to reload.

## Rules and limits

- Web is for brands only; mobile is for influencers only. Admin accounts can't log into either app.
- Apple and Facebook login are **not available** in either app. Only Google.
- reCAPTCHA is required for web email login (not for Google login).
- **Security alert:** after **2 wrong passwords within 6 hours** on the **web** login, the user gets the email "Security Alert – Login Attempts on your Ainfluencer account". It contains a "Go to My Panel" one-click login button (valid 60 minutes, single use). Mobile logins don't trigger this email.
- There is **no lockout**: wrong passwords never block the account.
- Sessions: the web refreshes the login automatically when it expires; if the refresh fails, the user sees "Your session has expired. Please log in again." and returns to `/login`. Mobile does the same and returns to the "Sign In" screen. Session lengths are set in server configuration.
- **Mobile logout is only on the phone**: "Logout" in Account Settings clears the app and stops push notifications on that phone, but doesn't end sessions on other devices.
- Sessions end on all devices only when a brand is suspended (BRD-04), an account is deleted (ACC-05), or a team member is removed (BRD-03). **Changing or resetting the password does not log out other devices.**

## Statuses

| What the user sees | Meaning (data) | Next |
|---|---|---|
| Sent to verify screen | `users.email_verified_at` empty | Enter code (ACC-01) |
| Appeal pop-up / "You have submitted an appeal request..." | Brand suspended (`brand_details.status` temporary/permanent suspend) | BRD-04 |
| "This account has been permanently deleted..." | `users.archived_at` or `deleted_at` set | ACC-05 |
| Logged in, sent to onboarding | No business name or location yet | BRD-01 |

## Timers and automatic actions

- Failed-password counter: resets 6 hours after the first failure, or after a successful login.
- One-click login links: expire after 60 minutes; single use.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| 2 wrong web passwords within 6 hours | The account owner | Email (security alert, from the security team) + in-app notification |
| Login with an unverified email | User | Verification code email (ACC-01) |

## Common questions

### "I can't log in. It says my password is wrong."
**Answer for the user:** Please use "Forgot your password?" (web) or "Forgot password?" (app) to set a new one. If you signed up with Google, use the Google button instead.
**What to check:** The account exists with that exact email and isn't deleted. Whether it was created through Google (the user may never have had a password).
**Escalate when:** Reset emails don't arrive (ACC-03).

### "I'm an influencer and the website won't let me in."
**Answer for the user:** Influencer accounts use the Ainfluencer mobile app. Please download it and log in there with the same email and password.
**What to check:** `users.type` = influencer.
**Escalate when:** Never.

### "I'm a brand and the app says it's only for influencers."
**Answer for the user:** Brand accounts use the web app at app.ainfluencer.com. The mobile app is for influencers.
**What to check:** `users.type` = brand.
**Escalate when:** Never.

### "I got a 'Security Alert – Login Attempts' email. Was I hacked?"
**Answer for the user:** We send this after two wrong password attempts on the website within 6 hours. If it was you, you can ignore it. If it wasn't, please change your password right away using "Forgot your password?" and contact us.
**What to check:** Recent failed attempts can be seen in the login security log (`login_security_logs`) if needed.
**Escalate when:** The user reports actions in the account they didn't do.

### "The app keeps asking me to update and I can't skip."
**Answer for the user:** Every new app version is required. Please update Ainfluencer from the App Store or Google Play, then open it again.
**What to check:** The user's store actually shows an update (store rollouts can be gradual).
**Escalate when:** The store shows no update but the app still blocks the user.

### "My login link says it's invalid or expired."
**Answer for the user:** These links work once and only for 60 minutes. Please log in with your email and password, or use "Forgot your password?".
**What to check:** Nothing more is needed.
**Escalate when:** Never.

### "I logged out of the app but I'm still logged in on my other phone."
**Answer for the user:** Logging out in the app only signs out that phone. Please log out on the other phone too.
**What to check:** Nothing. Note that changing the password does not sign out other devices either.
**Escalate when:** The user believes someone else is using the account (staff/dev can end all sessions by removing the user's `tokens` rows).

### "I logged in and I'm in a different brand's account."
**Answer for the user:** You're a member of that brand's team. Use the team switcher in the profile menu to go back to your own brand.
**What to check:** The user is in `team_user` for another brand's team (BRD-03).
**Escalate when:** The user says they never accepted an invitation.

## Known issues

- **Influencer password login on the web shows a "Warning" pop-up**, not the influencer landing page. Both mean "use the app".
- **No error page for an expired `/tmp-link`.** The page stays blank apart from the error toast *(to verify)*. Workaround: log in normally.
- **Mobile logout doesn't end the session on the server.** Not a problem for users in practice, but support should not promise that logging out on one phone logs out others.

## Related topics

- ACC-01: sign-up and verifying the email.
- ACC-03: forgot and reset password.
- ACC-05: deleted accounts.
- BRD-01: onboarding right after the first login.
- BRD-03: team membership and the team switcher.
- BRD-04: suspended brands and the appeal form.

## Technical reference

- **Endpoints**
  - `POST /api/v2/auth/login` → `v2/Frontend/AuthController@login` → `Services/Implementation/UserService::login/doLogin` (reCAPTCHA, influencer refusal, failed-password counter, suspension check via `Modules/User/Login/Services/UserSuspensionService::inspectSuspension`).
  - `POST /api/v2/auth/refresh-token` (web, called by the axios interceptor in `src/lib/services/_config.ts`).
  - `POST /api/v2/3rd-party/verify-auth` → `ThirdParty/OAuthService::verifyAuth` (web Google).
  - `POST /api/tmp-link/verify` → `TmpLinkController@verify` → `app/Services/TmpLinkService.php` (60-minute, single-use).
  - Mobile: `POST /api/en/v3/mobile/login` → `Mobile/.../Auth/AuthService::login` (pipeline `PreventDeletedUser`, `PreventSuspendedUser`, `PreventBrands`, `PreventAdmins`, `ValidateEmailIsVerified`); `POST …/auth/social/google` → `SocialAuthenticator`; `POST …/refresh-token`; `GET …/versions` (splash update check).
- **Data**: `tokens` (one row per session: `access_token`, `refresh_token`, `firebase_token`, `expired_at`), `tmp_links` (`token`, `expires_at`, `used`), `login_security_logs`, Redis key `failed_login_attempts:{user_id}` (TTL 6 h, threshold 2).
- **Security alert**: `app/Services/Auth/LoginSecurityService.php`, `app/Notifications/LoginSecurityNotification.php` (view `emails.notifications.login_security`).
- **Web**: `src/components/pages/login2`, `tempLogin`, `InfluencerLanding`; strings `locales/en/login2.json`, `login.json`. Lang file `resources/lang/en/auth.php` (all auth messages).
- **Mobile**: `src/features/auth/screens/login`, `src/features/splash/controller.ts` (update logic), `src/utils/logout.ts`.
