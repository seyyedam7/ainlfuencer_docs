---
id: AFA-01
title: Affiliate sign-up, login and onboarding
domain: Affiliate app (creators)
audience: [affiliate]
apps: [affiliate-app, email]
related: [AFX-01, AFA-02, AFA-04, AFB-04, AFC-04, AFS-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-01 · Affiliate sign-up, login and onboarding

> How a creator creates an affiliate account on affiliate.ainfluencer.com, confirms their email, logs in, and goes through the onboarding steps (phone, questions, linked logins, promotion platforms, profile) before reaching the app.

## Who and where
- **Affiliates** (creators) on the affiliate app, affiliate.ainfluencer.com.
- Screens:
  - Login `/user/login` ("Login", "Forgot password?", "Register Now").
  - Register `/user/register` (heading "Register").
  - Forgot password `/user/forgot-password`.
  - Reset `/user/reset-password/<token>` (from the email).
  - Onboarding `/user/register-steps`.
- Influencers coming from the Ainfluencer mobile app arrive through single sign-on and skip this sign-up form (AFX-01).
- After login the affiliate lands on **Product Discovery** (`/app/products-discovery`).

## How it works
1. **Register.** The user enters "Full Name", "Email" and "Password", ticks the reCAPTCHA and clicks "Continue". The password must be **at least 8 characters with an upper-case letter, a lower-case letter and a number**.
2. **Check your email.** The screen shows "Check your Email … We sent you an email with instructions on how to set up your account." The email contains a **verification link valid for 1 hour**. Clicking it confirms the email and logs the user in.
3. **Onboarding steps** (`/user/register-steps`). The app shows the first step not yet done. Steps marked optional show "Skip". The default setup is below; an admin can change which steps are mandatory.

   | Step | What the user sees | Required by default |
   |---|---|---|
   | Phone | "Verify your phone number", then "Send Code" and a **6-digit SMS code** ("Check Your Phone"). The code is valid for **3 minutes**; "Resend" is available after 60 seconds | Yes |
   | Linked logins | "Connect to Your Accounts" (Google or Facebook, for quick login later) | No |
   | Questions | "Let Us Get to Know you Better" | No |
   | Promotion platforms | Connect at least one social account or website, or submit them for manual review (AFA-02) | Yes |
   | Profile | Avatar, name, gender, country and so on | No |

   At the end the user sees "Welcome to Ainfluencer, Affiliate!" and is taken to the dashboard after a 15-second countdown ("Go Now" skips the wait).
4. **Login.** "Email" + "Password" + reCAPTCHA → "Login". Or use the **Google** or **Facebook** button (a popup). The email must already be verified.
5. **Forgot password.** The user enters their email → "Send". The email contains a reset link **valid for 5 hours**. On "Set your new password" they enter "New password" twice → "Submit". The same password rules apply.
6. **From a brand's recruitment page.** If the user clicked "Apply for partnership" on a seller's brand page before having an account, that request is saved in the browser. It is sent automatically when they register or log in (AFB-04, AFA-04).

## Rules and limits
- One affiliate profile per verified email. If that email already has a verified affiliate account, registration fails with "Email already exists".
- The same login can also hold a **seller** profile. Switch from the user menu (AFX-01).
- **Phone countries that can't be chosen:** Pakistan, India, Russia, North Korea, Iraq, Iran, Cuba, Sudan, Syria.
- The phone number's country later decides the country of the affiliate's **Stripe payout account** (AFA-08), so it matters.
- **Amazon login is not available** in the affiliate app (the button is hidden). Only email, Google and Facebook work.
- "Remember me for 30 days" is shown but has no effect.
- While onboarding is incomplete, every `/app` page sends the user back to `/user/register-steps`. The "Logout" button is available on every step.

## Statuses
| What support may see | Meaning |
|---|---|
| Email not verified | The user hasn't clicked the link; login fails |
| Registration incomplete | A mandatory onboarding step isn't done; the user is held on onboarding |
| Account status (admin panel): pending / requested / accepted / rejected / suspended | Set by admins on the affiliate admin panel |

## Timers and automatic actions
- Email verification link: expires after **1 hour**.
- Phone code: expires after **3 minutes**; resend after **60 seconds**.
- Password reset link: expires after **5 hours**.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Registration | User | Email with the verification link |
| Forgot password | User | Email with the reset link |
| Phone step | User | SMS with a 6-digit code |

## Common questions

### "I never got the verification email."
**Answer for the user:** Please check your spam folder. The link is valid for 1 hour. If it has expired, register again with the same email to get a new link, or contact us.
**What to check:** That the email is typed correctly; whether the account exists and the email is still unverified.
**Escalate when:** Several users report missing emails at the same time (mail delivery problem).

### "It says 'Email already exists' but I can't log in."
**Answer for the user:** An affiliate account with this email already exists. Use "Forgot password?" to set a new password. If you signed up with Google or Facebook, use that button instead.
**What to check:** Whether the existing account was created through Ainfluencer single sign-on or social login (random password).
**Escalate when:** A password reset doesn't arrive or doesn't work.

### "I can't choose my country for the phone code."
**Answer for the user:** Phone verification isn't available for some countries (Pakistan, India, Russia, North Korea, Iraq, Iran, Cuba, Sudan, Syria). Please use a number from another country if you have one.
**What to check:** The country they're trying.
**Escalate when:** A country not on that list can't be selected, or the SMS never arrives for a supported country.

### "I'm stuck on the onboarding steps."
**Answer for the user:** Two steps are required: verifying your phone and adding at least one promotion platform. The others can be skipped with "Skip".
**What to check:** Which step is shown; for the platform step, see AFA-02 (a category must be chosen for each platform).
**Escalate when:** The user completed a step but the app keeps showing it.

### "Can I log in with Amazon?"
**Answer for the user:** Not in the affiliate app at the moment. Please use email, Google or Facebook.
**What to check:** —
**Escalate when:** —

### "I already use Ainfluencer as an influencer. Do I need a new account?"
**Answer for the user:** No. Open the affiliate area from the Ainfluencer app and you'll be signed in automatically with the same email.
**What to check:** That the email is the same on both platforms (AFX-01).
**Escalate when:** The user ends up with two accounts because the emails differ.

## Known issues
- "Remember me for 30 days" does nothing.
- The Amazon login button is hidden in this app, although the backend still supports it.
- The register-form text mentions a "6-digits verification code", but affiliates receive a **link**, not a code.

## Related topics
- AFX-01 · Single sign-on from Ainfluencer and switching between seller and affiliate.
- AFA-02 · The promotion-platform step in detail.
- AFA-04 / AFB-04 · Partnership requests started on a brand page before sign-up.
- AFC-04 · Sessions and password change from Settings → Security.
- AFS-01 · The same flow for sellers.

## Technical reference
- **Endpoints:**
  - `POST affiliate/v1/auth/register` (`RegisterController` → `Services/Actions/Affiliate/Auth/RegisterService` → `Services/General/Auth/UserRegister`: password regex, `email_exists` check).
  - `POST affiliate/v1/auth/login`, `POST affiliate/v1/auth/login/{google|facebook}` (callbacks `GET web/v1/affiliate/login/*/callback`).
  - `POST affiliate/v1/auth/forget-password`, `POST affiliate/v1/auth/reset-password`.
  - `POST affiliate/v1/auth/mobile-verification-code`, `POST affiliate/v1/auth/mobile-verify`.
  - `POST affiliate/v1/auth/skip-step`, `GET affiliate/v1/profile/show-register-steps`.
  - `GET/POST affiliate/v1/questions`, `POST affiliate/v1/profile`, `POST affiliate/v1/profile/sync/account/{google|facebook}`, `GET affiliate/v1/auth/auth-status`.
  - Email verify link `web/v1/affiliate/email/verify/{id}/{hash}` → front `/user/email-verified/{token}`.
- **Config:** `auth.verification_code_expiration_time` = 3600 s; `auth.mobile_verification_code_expiration_time` = 180 s; `sanctum.reset_password.expiration_time` = 5 h; front `VUE_APP_IGNORED_COUNTRIES`.
- **Data:** `users` (`country_calling_code` from the verified phone), `user_emails.email_verified_at`, `user_types` (type 2 = affiliate; `status`), `register_steps` + per-user passed steps (seeded in `database/seeders/RegisterStepTableSeeder.php`).
- **Frontend:** `shared-components/algo-auth/*`, `spa-affiliate/src/components/auth/register-steps.page.vue`, `auth-register-steps-platforms-v2.component.vue`.
