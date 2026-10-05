---
id: ACC-01
title: Sign-up and email verification
domain: Account and sign-in
audience: [brand, influencer]
apps: [web, mobile, email]
related: [ACC-02, ACC-04, BRD-01, BRD-03, BRD-04, DSC-03, INF-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# ACC-01 · Sign-up and email verification

> How a brand creates an account on the web app, how an influencer creates one in the mobile app, and how both confirm their email address with a 5-digit code or a link. Nothing in the product works until the email is confirmed.

## Who and where

- **Brands** sign up on the web at `/register` (page title "Sign up | Ainfluencer"). The page has three tabs that only change the marketing text: "Affiliate", "Paid Collabs" and "Seeding / Gifting". A switch at the top chooses "I'm a brand" or "I'm a creator".
- **Influencers** sign up in the mobile app on the "Sign Up" screen (SIGN_UP_SCREEN), reached from "Sign Up" on the login screen. If they pick "I'm a creator" on the web page, they only see "Creators sign up in the app" with app-store buttons; the web form does not create influencer accounts.
- **Invited influencers** can also create an account from a campaign invitation page on the web: `/invitation`, `/affiliate/invitation` or `/gifting/invitation` (steps "Welcome", "Create an Account", "Download App"). See DSC-03 and PUBL-01.
- **Landing pages** (marketing sites) can create brand accounts directly; those users receive a verification **link** by email instead of a code.
- **Team members** invited by a brand register on a separate page (BRD-03).
- Email confirmation happens on `/verify` (web), on the "Verify Your Email" screen (mobile), or by clicking the link in the email (`/verifybylink/...` for brands, `/auto-verify?...` for influencers).

## How it works

**Brand, web, email and password**
1. On `/register`, the brand fills "First name", "Last name", "Work email", "Password" and "Phone", ticks "I agree to Ainfluencer's Terms of Service and Privacy Policy", completes the reCAPTCHA and clicks the button (for example "Create collab account").
2. Ainfluencer creates the account, a wallet, and (in production) a Stripe customer for the brand, then emails a **5-digit code** with the subject "Ainfluencer Verification Code". The same email also has a "Verify" button (a one-click link).
3. The brand is taken to `/verify` ("Verify your email", "Check your email and enter the 5-digit code below."), types the code and clicks "Verify". A toast says "Verify success".
4. The brand is sent to onboarding (BRD-01), then the dashboard.

**Brand, web, Google**
- "Sign up with Google" (or "Log in with Google" on `/login`) creates the account straight away as a **verified** brand. No code, no phone number and no reCAPTCHA are needed. The brand goes directly to onboarding.

**Influencer, mobile**
1. On "Sign Up", the influencer enters "First Name" (required), "Last Name", "Email" and "Password" and taps "Sign Up". Or taps "Continue with Google".
2. With email and password, a 5-digit code is emailed and the app opens "Verify Your Email" ("A 5-digit code has been sent to ..."). The influencer types the code and taps "Verify". "Open Email App" opens their mail app.
3. The app opens on Explore. About 3 seconds later it starts social-account onboarding if no account is connected yet (INF-01).
4. With Google, the account is created already verified, and the app goes straight to onboarding.

**Invited influencer, web invitation page**
- The page shows the campaign, then "Create My Account". The form asks for name, email, password ("Re-enter your password"), a category ("Select a category (Required)") and location ("Where are you located? (Required)"). Then "Check your Email to enter the 5 digit code below:" and "Verify". The last step, "Download App", sends them to the mobile app.

## Rules and limits

- Password: at least 8 characters (web and mobile).
- **Brand web sign-up requires a phone number** in international format (a "+" and country code, e.g. `+15551234567`). The phone is stored but **never verified**: there is no SMS code step anywhere today.
- First and last name can't contain links or email addresses ("unacceptable value" toast).
- One email = one account. An email can't be both a brand and an influencer.
- If the email already belongs to an **unverified** account, signing up again does not fail: Ainfluencer sends a new code and takes the user to the verify step.
- If the email belongs to a **verified** account: "The email you entered already exists in the Ainfluencer." (web) or "The email address has already been taken." (mobile).
- If the email belongs to a **deleted** account: "This account has been permanently deleted and can no longer be used to log in or sign up." (see ACC-05).
- If the email belongs to a **suspended brand**, the suspension and appeal pop-up appears instead (BRD-04).
- A brand email can't be used in the mobile app: "You are signed up as a brand. The mobile application is only for influencers."
- Codes: 5 digits. Codes **don't expire**, and every code sent stays valid until the email is confirmed. "Resend" is available after **60 seconds** on both web ("You may try to resend the activation code in 00:59", then "No code? Resend it now") and mobile ("Didn't get the code?", "Resend Code").
- The verification **link** works once. After it is used it shows "The provided email verification token is invalid."

## Statuses

| What the user sees | Meaning | Next |
|---|---|---|
| Sent to "Verify your email" after login or sign-up | Email not confirmed (`users.email_verified_at` is empty) | Enter the latest (or any earlier) code |
| Lands on onboarding or the dashboard | Email confirmed | Normal use |

## Timers and automatic actions

- None for sign-up itself. Codes never expire. Resend is limited only by the 60-second countdown on screen.
- In production, a missing wallet or Stripe customer is filled in by an hourly background job (`finance:create-missed-wallets`).

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Sign-up, "Resend", sign-up with an existing unverified email, login while unverified | User | Email "Ainfluencer Verification Code" (code + "Verify" link) |
| Landing-page sign-up | Brand | Email "Ainfluencer Verification Link" (link only) |

These emails are always sent, even if the user has unsubscribed from emails.

## Common questions

### "I never got the verification code."
**Answer for the user:** The code is sent to the email you signed up with, usually within a minute. Please check Spam, Junk, Promotions and other tabs. If it's not there, wait for the 60-second timer and tap "Resend". Any code we've sent you still works.
**What to check:** That the email on the account is spelled correctly (a typo is the most common cause). That the account exists and is still unverified (`users.email_verified_at` empty). Whether the user unsubscribed from emails (verification emails ignore that, so it shouldn't matter).
**Escalate when:** The email is correct, resending several times delivers nothing, and other users on the same email provider receive emails normally (possible mail delivery problem).

### "I made a typo in my email when signing up."
**Answer for the user:** We'll fix it for you. Please tell us the email you typed and the correct one.
**What to check:** The account with the wrong email exists and is unverified. Self-service email change is blocked in both apps (see ACC-04), so support must ask staff to update the email in the admin panel.
**Escalate when:** Always: an admin has to change the email (and can mark it verified).

### "It says my code is invalid."
**Answer for the user:** Please make sure you typed the 5 digits from the most recent email for this exact email address. If it still fails, tap "Resend" and use the new code.
**What to check:** The user is entering the code on the same account (logged in with the same email). An influencer who also tried the web may be mixing up accounts.
**Escalate when:** A fresh code is rejected several times.

### "Do I need to verify my phone number?"
**Answer for the user:** No. We ask for a phone number when brands sign up, but there is no SMS verification step.
**What to check:** Nothing.
**Escalate when:** Never.

### "I'm an influencer. Can I sign up on the website?"
**Answer for the user:** Influencer accounts are created in the Ainfluencer mobile app (iOS and Android). The website is for brands.
**What to check:** If they already created an account on the web, logging into the web app as an influencer shows "You are signed up as an influencer. The web platform is only for brands and businesses." Their account works in the mobile app.
**Escalate when:** Never.

### "It says my account was deleted, but I want to sign up again."
**Answer for the user:** A deleted account can't be used to sign up again with the same email. Our team may be able to restore it for you.
**What to check:** `users.archived_at` or `users.deleted_at` is set.
**Escalate when:** The user wants the account restored (staff can restore it in the admin panel; see ACC-05).

### "I signed up with Google. What's my password?"
**Answer for the user:** Google accounts don't have an Ainfluencer password. Keep using "Log in with Google" / "Continue with Google", or use "Forgot password?" to set one.
**What to check:** The account was created through Google (no password the user knows).
**Escalate when:** Password reset emails don't arrive.

### "I'm stuck on the verify screen in the app and the code never comes."
**Answer for the user:** If your email is a Google (Gmail) address, you can go back and use "Continue with Google": this confirms your email automatically.
**What to check:** Same as the first question.
**Escalate when:** Same as the first question.

## Known issues

- **Phone number is required but not used.** Brands sometimes ask why. There is no phone verification (the old phone OTP page is unreachable).
- **The web verify page needs to be opened from login or sign-up.** Opening `/verify` directly (for example from a bookmark) sends the user to `/login`. Workaround: log in, and the app opens the verify page.
- **Email typo can't be fixed by the user** (see ACC-04). Escalate to an admin.
- **The mobile "Verify Your Email" screen shows a pencil to change the email, but changing it after sign-up fails** with "The email cannot be changed here. Please submit a support ticket and an administrator will update it for you." See ACC-04.

## Related topics

- ACC-02: logging in, Google login, sessions.
- ACC-04: changing the email address.
- BRD-01: brand onboarding right after verification.
- BRD-03: how invited team members register.
- BRD-04: what a suspended brand sees at sign-up.
- DSC-03 / PUBL-01: campaign invitation pages that lead to sign-up.
- INF-01: what an influencer does after verifying (connecting a social account).

## Technical reference

- **Endpoints**
  - `POST /api/v2/auth/register` → `v2/Frontend/AuthController@register` → `Services/Implementation/UserService::register/doRegister` (web brand sign-up; reCAPTCHA required).
  - `POST /api/v2/auth/email/verify` (`{token}`, auth) and `POST /api/v2/auth/email/resend` → `UserService::verifyEmail/resendEmail`.
  - `POST /api/v2/email/confirm` (brand link) and `POST /api/v2/influencer/email/confirm` → `v2/Frontend/EmailVerificationController` (link token = md5 of `verification-{user_id}-{code}`).
  - `POST /api/v2/3rd-party/verify-auth` → `v2/ThirdParty/AuthController@verifyAuth` → `ThirdParty/OAuthService::verifyAuth` (web Google sign-up/login; new users created verified).
  - `POST /api/v2/guest/signup` → `v2/Guest/SignupController` → `Guest/InvitationSignupService` (invitation pages).
  - `POST /api/v2/webhook/signup` → `v2/Webhook/SignupController` → `Webhook/SignupService` (landing pages; sends the link email).
  - Mobile: `POST /api/en/v3/mobile/register`, `POST …/verify-email`, `POST …/resend-email-verification` → `Modules/Mobile/.../AuthController` → `Mobile/Services/Implementation/Auth/AuthService` and `InfluencerRegistrar`; Google: `POST …/auth/social/google` → `SocialAuthController` → `SocialAuthenticator` (only provider: `google`).
- **Data**
  - `users.email_verified_at` (null = unverified), `users.type` (`brand`/`influencer`), `users.registered_platform`, `users.phone`, `users.archived_at` / `deleted_at`.
  - `email_verifications` (`user_id`, `token` = the 5-digit code; one row per code sent; deleted when used).
  - `signup_details` + `utms` (UTM tracking from the sign-up URL).
- **Emails**: `app/Mail/EmailVerification.php` (code, views `emails.v2.brand|influencer.verification-code`), `app/Mail/EmailVerificationLink.php`. Code generator: `app/Traits/Tools/VerificationCodeGenerator.php`. Link builder: `app/Helpers/Functional/GetEmailVerificationLink.php`.
- **Events**: `UserRegisteredEvent` (wallet + CRM sync), `UserVerifiedEvent` (CRM tags).
- **Web**: `src/components/pages/register2`, `verifyEmail2`, `verifyByLink2`, `verifyInfluencerByLink`, `invitation`. Translations `lib/localization/locales/en/register2.json`, `verifyEmail2.json`, `invitation.json`.
- **Mobile**: `src/features/auth/screens/signup`, `verifyEmail`; strings `src/localization/translations/en/signup.js`, `submitVerifyCode.js`.
