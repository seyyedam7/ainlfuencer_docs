---
id: AFS-01
title: Seller sign-up, login and onboarding
domain: Seller app (brands)
audience: [seller]
apps: [seller-app, email]
related: [AFX-01, AFX-03, AFS-02, AFA-01, AFB-04, AFC-04, AFS-12, AFS-13]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-01 · Seller sign-up, login and onboarding

> How a brand or store owner creates an account on the seller app (seller.ainfluencer.com), logs in, and completes the setup wizard (phone, store, questions, profile). Ainfluencer brands usually arrive through single sign-on instead (AFX-01).

## Who and where

- **Seller**, in the seller app:
  - "Register" page `/user/register` ("Create New Seller Account").
  - Login page `/user/login` ("Login to your Existing Seller Account").
  - "Forgot password?" → `/user/forgot-password`, then the email link `/user/reset-password/<token>`.
  - Setup wizard `/user/register-steps` (shown automatically until setup is finished).
  - Profile editing: user menu → "Edit profile" (`/app/profile`).
- An Ainfluencer brand who uses "Advanced Affiliate Dashboard" skips registration (AFX-01).

## How it works

**Sign up**
1. On `/user/register` the seller enters "Full Name" (up to 100 characters), "Email" and "Password" (8–20 characters with at least one capital letter and one number) and solves the reCAPTCHA. They can also use Google or Facebook.
2. The screen changes to "Check your email". An email with a verification link is sent. There is no code to type in.
3. Clicking the link verifies the email and logs the seller in. The setup wizard opens.
4. If the seller came from a brand's public recruitment page or has a referral cookie, that is attached to the new account automatically (AFB-04).

**Log in**
1. `/user/login`: "Email", "Password", reCAPTCHA, "Login". Or Google / Facebook (popup).
2. After login the seller lands on **Affiliate Discovery** (`/app/discovery-affiliates?grouped=true`), not the dashboard.
3. If setup isn't finished, any page sends them back to the wizard.

**Setup wizard** (steps in this order; a step can be skipped only if it isn't mandatory)

| Step | Screen | Mandatory |
|---|---|---|
| Phone | "Verify your phone number" → "Send Code" → 6-digit SMS code ("Check Your Phone"); "Resend" after 60 s; code valid 3 minutes | Yes |
| Store | "Connect Your Stores to Setup Your Affiliate Program": "Connect Your Amazon" (popup) or "Connect to Shopify" (enter shop domain, then redirect). "Next" is enabled once a store is connected | Yes |
| Questions | "Let Us Get to Know you Better" (answers save automatically) | No ("Skip") |
| Profile | Avatar (JPG/PNG), bio, gender, country | No ("Skip") |
| Done | "Your profile is complete!" → "Go to Dashboard" | – |

"Go to Dashboard" actually goes to Affiliate Discovery, or to the page an SSO link asked for.

**Forgot password**
1. "Forgot password?" → enter email → an email with a reset link is sent.
2. The link opens "Set your new password". It is valid **5 hours** and works once.

**Profile editing** (`/app/profile`): tabs Overview (plan usage), "Edit profile information" (name, nickname, gender, birthday, country, company name/website/address, bio; changing the phone needs a new SMS code), "Get better profile" (questions), "Manage accounts" (stores, AFS-02).

## Rules and limits

- **One email = one login.** The same login can have a seller and an affiliate profile; switch with "Change Panel" (AFX-01).
- Phone numbers from these countries can't be selected: Pakistan, India, Russia, North Korea, Iraq, Iran, Cuba, Sudan, Syria, Afghanistan.
- The verification link expires after **60 minutes**. Login is refused until the email is verified.
- An account created by Ainfluencer has a random password; use "Forgot password?" to log in directly (AFX-01).
- "Remember me for 30 days" is shown on the login page but does nothing.
- Team members get their own login and then switch into the owner's team (AFS-13).

## Statuses

| Status | Meaning |
|---|---|
| Email not verified (`user_emails.email_verified_at` empty) | Can't log in; link not clicked or expired |
| Registration not completed (`user_types.register_completed` = false) | Wizard is shown on every page |
| Registration completed | Normal access |
| Account status (`user_types.status`): 1 pending, 2 requested, 3 accepted, 4 rejected, 5 suspended | Set by admins. A verified seller becomes "accepted". Suspended sellers are skipped by commission billing (AFS-10) |

## Timers and automatic actions

- Verification link: 60 minutes. SMS code: 3 minutes, resend after 60 s. Reset link: 5 hours.
- No scheduled job is involved.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Sign up | Seller | Email with verification link |
| Forgot password | Seller | Email with reset link |
| Password changed | Seller | Notification (`changePassword`) |
| Phone verification / change | Seller | SMS code |

## Common questions

### "I never got the verification email / the link says it expired."
**Answer for the user:** Please check your spam folder. The link is valid for 60 minutes. If it expired, register again with the same email; a new link will be sent to you.
**What to check:** the email in `user_emails` (personal) and whether `email_verified_at` is empty; that the user isn't registering on the affiliate app by mistake.
**Escalate when:** a fresh link still doesn't arrive after checking spam.

### "Login says 'These credentials do not match our records.'"
**Answer for the user:** Either the password is wrong, the email isn't verified yet, or this email has no seller account (only an affiliate one). Try "Forgot password?" first. If you use Ainfluencer, you can also open the seller app with the "Advanced Affiliate Dashboard" button.
**What to check:** a verified personal email exists; the user has a seller profile (`user_types.type_id` seller); account created by Ainfluencer (`users.register_source` = ainfluencer).
**Escalate when:** all three are fine and a reset password still fails.

### "I can't pick my country for the phone number."
**Answer for the user:** Phone verification isn't available for numbers from a few countries. Please use a phone number from another country.
**What to check:** the country is in the blocked list above.
**Escalate when:** the user's country isn't on the list and still can't be chosen.

### "The 'Next' button is disabled on the Connect stores step."
**Answer for the user:** You need to connect at least one Amazon or Shopify store to continue. For Shopify, enter your permanent `.myshopify.com` address.
**What to check:** connection errors (AFX-03): Amazon eligible marketplace, Shopify store already linked to another account.
**Escalate when:** a store connected successfully but the wizard still blocks.

### "I didn't receive the SMS code."
**Answer for the user:** Wait 60 seconds and press "Resend". Make sure the number includes the right country code.
**What to check:** number format; repeated attempts.
**Escalate when:** codes never arrive for a supported country.

### "Why do I land on Affiliate Discovery and not the dashboard after logging in?"
**Answer for the user:** That's by design: after login the seller app opens Affiliate Discovery. The dashboard is the first item in the left menu.
**What to check:** nothing.
**Escalate when:** n/a.

## Known issues

- **"Remember me for 30 days" has no effect.**
- **Onboarding store step:** connected Shopify stores aren't listed on that step, and the "Add Seller Central account" item under the Shopify list starts the Amazon flow. Shopify-only sellers are not blocked in practice, because a successful Shopify connection marks the step as done on the server.
- **No "resend verification email" button.** Workaround: register again with the same email (allowed while unverified).
- **SSO-created accounts may loop on the "Your profile is complete!" screen** *(to verify)*, see AFX-01.

## Related topics

- AFX-01 — Single sign-on from Ainfluencer, panel switching, random passwords.
- AFX-03 / AFS-02 — Connecting stores.
- AFA-01 — The same flows on the affiliate app.
- AFB-04 — Applying from a brand page before signing up.
- AFC-04 — Sessions and password reset from Settings → Security.
- AFS-13 — Team members.

## Technical reference

- **Endpoints:** `POST seller/v1/auth/register` (`RegisterController`), `POST seller/v1/auth/login`, `POST seller/v1/auth/login/{google|facebook}` + callbacks `GET web/v1/seller/login/{google|facebook}/callback`, `GET web/v1/seller/email/verify/{id}/{hash}` (signed, throttle 6/min), `POST seller/v1/auth/forget-password`, `POST seller/v1/auth/reset-password`, `GET seller/v1/auth/auth-status` (`register_completed`), `GET seller/v1/profile/show-register-steps`, `POST seller/v1/auth/skip-step`, `POST seller/v1/auth/mobile-verification-code`, `POST seller/v1/auth/mobile-verify`, `GET/POST seller/v1/questions`, `GET/POST seller/v1/profile`, `GET seller/v1/profile/overview`, `POST seller/v1/profile/request-change-mobile`, `PUT seller/v1/profile/change-mobile`.
- **Services:** `Services/General/Auth/UserRegister.php` (password rule, `addRegisterSteps`), `Services/General/Auth/VerifyCode/SendVerificationLink.php` (expiry `auth.verification_code_expiration_time` = 3600 s), `Services/General/Auth/Sanctum.php` (`loginByEmail` requires verified personal email), `Services/General/Auth/ResetPassword.php` (`sanctum.reset_password.expiration_time` = 5 h), `Services/General/Profile/Profile.php` (`passUserRegisterStep` → `checkRegisterCompleted`). SMS code lifetime `auth.mobile_verification_code_expiration_time` = 180 s.
- **Steps (seeded):** seller `verify_mobile` (mandatory, order 0), `sync_social_account` (mandatory, 1), `questions` (optional, 2), `profile` (optional, 4), `sso` (auto, SSO accounts only) — `database/seeders/RegisterStepTableSeeder.php`.
- **Data:** `users` (`register_source`, `reference_id`), `user_emails` (`type`, `email_verified_at`), `user_types` (`type_id`, `status`, `register_completed`, `profile_completed`), `user_register_steps` (`is_passed`, `is_skipped`), `reset_password_tokens` (`expired_at`, `status`).
- **Frontend:** `shared-components/algo-auth/auth-login.vue`, `auth-register/auth-register.vue`, `auth-register-steps/*` (wizard, socialite step), `shared-components/router/auth.router.js` (guards), i18n `shared-components/algo-auth/i18n/*.js`; blocked countries `spa-seller/.env` `VUE_APP_IGNORED_COUNTRIES`.
