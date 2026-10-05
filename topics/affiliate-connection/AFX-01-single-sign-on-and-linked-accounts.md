---
id: AFX-01
title: Single sign-on and linked accounts
domain: Ainfluencer ↔ affiliate platform
audience: [brand, seller, influencer, affiliate, admin]
apps: [web, seller-app, affiliate-app, background]
related: [CMP-04, MKT-03, AFX-02, AFX-03, AFS-01, AFA-01, CHAT-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d, affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFX-01 · Single sign-on and linked accounts

> The affiliate platform (seller.ainfluencer.com and affiliate.ainfluencer.com) is a separate product with its own accounts. Ainfluencer creates and links those accounts automatically, by email, and lets a brand jump into the seller app without a second password ("single sign-on", SSO).

## Who and where

- **Brands (Ainfluencer web app).** On `/affiliate/dashboard` the banner "Take affiliate marketing further" has the button **"Advanced Affiliate Dashboard"**. It opens the seller app in a new tab, already logged in.
- **Influencers (mobile app).** There is **no** SSO button in the current app (v32.2.2). An influencer's affiliate account is created silently the first time they use an affiliate feature (joining an affiliate campaign, affiliate analytics, payouts; see MKT-03). They see their affiliate data inside the Ainfluencer app.
- **Sellers/affiliates on the affiliate platform.** The user menu has a **"Change Panel"** switch (Affiliate | Seller). It opens the other panel in a new tab. If the user doesn't have that kind of account yet, a prompt **"Create Seller Account?"** / **"Create Affiliate Account?"** appears first.
- **Ainfluencer staff** can open a seller's or affiliate's app as that user from the affiliate admin panel (impersonation). In that mode "Logout" sends them back to the admin site.

## How it works

**Brand → seller app ("Advanced Affiliate Dashboard")**
1. When `/affiliate/dashboard` loads, the web app asks Ainfluencer for a sign-in link. The link contains a signed token with the brand's Ainfluencer user id, email and name.
2. The brand clicks "Advanced Affiliate Dashboard". The link opens the affiliate platform.
3. The affiliate platform checks the token. A brand becomes a **seller**; any other type becomes an **affiliate**.
4. It looks for an existing seller account **with the same email**. If none exists, it creates one (random password, email marked verified, Ainfluencer user id stored as the link).
5. It tells Ainfluencer "this user is now linked" (Ainfluencer stores the link; if this call fails nothing visible happens).
6. The browser lands on the seller app, logged in, at the page requested (or the dashboard). If the seller hasn't finished onboarding, the onboarding wizard is shown first (AFS-01).

**Accounts created without clicking anything**
- Brands are **not** registered on the affiliate platform at Ainfluencer sign-up. The seller account is created the first time the brand uses an affiliate feature that needs it: connecting an Amazon/Shopify store, adding the card, loading the affiliate dashboard, or creating an affiliate campaign (CMP-04, AFX-03).
- Influencers get an affiliate account the first time an affiliate feature in the mobile app needs one (MKT-03).
- In both cases the affiliate platform reuses an existing account if the **personal email** already exists there (it just adds the seller or affiliate profile to it).
- The brand's Stripe customer id is copied to the seller account at registration, so a card added in Ainfluencer is the card the affiliate platform charges (AFS-10).

**Switching panels on the affiliate platform**
1. User menu → "Change Panel" → pick the other side.
2. If the account already has that profile, the other app opens in a new tab, logged in.
3. If not, the user confirms "Create Affiliate Account?" (or Seller). The profile is added to the same login, then the other app opens. The new profile starts with its **own onboarding** (phone, store or platforms, questions, profile).

## Rules and limits

- **Matching is by email only.** If the brand's Ainfluencer email differs from the email they used on seller.ainfluencer.com, they get **two separate seller accounts**. Accounts are never merged automatically.
- The same email can hold a seller profile and an affiliate profile (one login, two panels).
- **Passwords:** accounts created by Ainfluencer get a random password. To log in directly at seller.ainfluencer.com or affiliate.ainfluencer.com the user must use "Forgot password?" with the same email (reset link valid 5 hours, single use). Their Ainfluencer password does not work there.
- **The sign-in link expires** 60 minutes after it was generated (server setting, default 60). The link is generated when the dashboard page loads, not when the button is clicked.
- An influencer logging into the Ainfluencer web app is sent to "download the app", so in practice only brands use the SSO button.
- Team members of a brand (BRD) open the seller app as the brand account they are logged into on Ainfluencer.

## Statuses

| What support sees | Meaning |
|---|---|
| Ainfluencer `users.connect_ainfluencer_affiliate_account` = true | The brand has been registered/linked on the affiliate platform |
| Ainfluencer `user_third_parties` row, provider `ainfluencer_affiliate`, type `seller` / `affiliate` (or `brand` / `influencer` when created by SSO) | The link: stores the affiliate user id and profile id |
| Affiliate `users.reference_id` | The Ainfluencer user id (written once) |
| Affiliate `users.register_source` = `ainfluencer` | Account created by Ainfluencer (SSO or automatic registration) |

## Timers and automatic actions

- Sign-in link lifetime: 60 minutes by default.
- No scheduled job is involved. Linking happens at the moment of registration or SSO.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Account created by SSO or by Ainfluencer | – | None (no verification email; the email is marked verified) |
| "Forgot password?" on the affiliate platform | User | Email with reset link |

## Common questions

### "I clicked 'Advanced Affiliate Dashboard' and got 'SSO Login Failed'."
**Answer for the user:** The sign-in link is only valid for a limited time. Please refresh the Affiliate dashboard page in Ainfluencer and click the button again.
**What to check:** whether the page had been open for more than an hour; the error text ("Invalid or expired token" means expired). Ask the user to refresh and retry.
**Escalate when:** a freshly loaded page still fails, or the error is "An unexpected error occurred during SSO login."

### "I already had a seller account, but the advanced dashboard opened an empty account."
**Answer for the user:** The two platforms link accounts by email. Your Ainfluencer email is different from the email on your existing seller account, so a second account was created. We can't merge them automatically.
**What to check:** the email on the Ainfluencer account vs. the email the user logs in with on seller.ainfluencer.com.
**Escalate when:** the user needs the accounts merged or the stores moved (dev task).

### "How do I log in to seller.ainfluencer.com directly? My Ainfluencer password doesn't work."
**Answer for the user:** Your seller account was created for you by Ainfluencer, so it has its own password. Use "Forgot password?" on seller.ainfluencer.com with the same email, or keep using the "Advanced Affiliate Dashboard" button.
**What to check:** that the user enters the same email as on Ainfluencer.
**Escalate when:** the reset email never arrives after checking spam.

### "After opening the advanced dashboard I'm asked to verify my phone / connect a store."
**Answer for the user:** The seller app has a short setup (phone, store, questions, profile). Please complete it once; after that you'll land on the dashboard.
**What to check:** affiliate `user_types.register_completed` and the `user_register_steps` rows for the seller profile.
**Escalate when:** the wizard keeps coming back after "Go to Dashboard" (see Known issues).

### "I switched to the Affiliate panel and now I have to onboard again."
**Answer for the user:** Seller and affiliate are separate profiles. The first time you open the other panel it has its own short setup.
**What to check:** nothing unusual; this is expected.
**Escalate when:** the switch shows an error instead of opening the other panel.

### "Can I message my affiliates from the seller app?"
**Answer for the user:** Please use the Ainfluencer chat to talk to influencers and affiliates; the affiliate platform's own messaging isn't a supported channel.
**What to check:** nothing; direct them to CHAT-01.
**Escalate when:** n/a.

## Known issues

- **Different email = two accounts.** No merge tool. Escalate to dev.
- **Expired link after a long session.** The link is generated on page load and expires after 60 minutes. Workaround: refresh the page.
- **Onboarding loop for some SSO-created accounts** *(to verify)*. Accounts created by Ainfluencer get all onboarding steps marked done but the "registration completed" flag is only set when country and gender are known, which Ainfluencer doesn't send. Such a seller may be sent to the "Your profile is complete!" screen and back after "Go to Dashboard". Escalate to dev with the email.
- **Social accounts are not imported for affiliates** *(to verify)*. When an affiliate account is created by SSO, the affiliate platform tries to fetch the influencer's social accounts with a GET request; Ainfluencer only accepts POST there, so nothing is imported. (Rarely hit, since influencers don't use SSO today.)

## Related topics

- CMP-04 — Brand affiliate dashboard where the button lives.
- MKT-03 — Influencer side; affiliate account created silently.
- AFX-02 — Affiliate campaigns stored on the affiliate platform.
- AFX-03 — Store connections (also trigger seller registration).
- AFS-01 / AFA-01 — Normal sign-up, login and onboarding on each app.
- CHAT-01 — The supported channel for brand ↔ influencer messages.

## Technical reference

- **Core endpoints:** `GET api/user-third-party/ainfluencer-affiliate/v1/sso/authorize-url` (`AinfluencerAffiliateSsoController@authorizeUrl`, type `brand` or `influencer` from `is_influencer`); `GET api/{locale}/v3/mobile/ainfluencer-affiliate/sso/authorize-url` exists (type `affiliate`) but the mobile app doesn't call it. Callbacks from the affiliate platform: `POST …/v1/sso/callbacks/user-registered` (`userRegistered`: sets `users.connect_ainfluencer_affiliate_account`, upserts `user_third_parties`), `POST …/v1/sso/callbacks/social-media-sync`.
- **Core services:** `app/Modules/UserThirdParty/Services/AinfluencerAffiliateService.php` (`getRedirectUrl`, `register`, `registerAffiliate`, `getUserThirdParty` = find or register lazily), token payload `Dto/AinfluencerAffiliate/AinfluencerAffiliateTokenInfoDTO.php`, expiry `Config/user_third_party.php` → `ainfluencer_affiliate.token_expire_time` (env `AINFLUENCER_AFFILIATE_TOKEN_EXPIRED_TIME`, default 60 min). Bulk back-fill command `ainfluencer-affiliate:register-brands` (manual, Shopify-connected brands only) → `Jobs/RegisterUserInAinfluencerAffiliateJob` (queue `ainfluencer-affiliate:register`).
- **Web:** `src/hooks/affiliate/useGetAffiliateLink.ts` (fetches link on mount), `useOpenAdvancedAffiliateDashboard.ts`, button `components/pages/affiliate/parts/dashboardAffiliate/DashboardAffCta.tsx` (label `affiliate.json` → `dashboard.banner.ctaButton`).
- **Affiliate endpoints:** `GET web/v1/sso/login?token=&redirect=` (`Web\Sso\SsoLoginController` → `Services/General/Sso/Sso.php` → `Sources/AinfluencerSsoSource.php`; error view `resources/views/sso/login_error.blade.php`); `POST seller/v1/integrations/ainfluencer/register` and `POST affiliate/v1/integrations/ainfluencer/register` (`Services/Actions/{Seller,Affiliate}/Integrations/Ainfluencer/RegisterIntegrationService.php`, matches personal email, 30 s lock); `GET seller/v1/auth/become-seller`, `GET affiliate/v1/auth/become-affiliate` (adds profile, `Sanctum::appendUserTypeToExistsUser`); `GET {seller|affiliate|buyer}/v1/auth/local-login` (panel switch link).
- **Data (affiliate DB):** `users.reference_id`, `users.register_source`, `user_types` (`type_id`, `register_completed`, `status`), `user_emails` (`type` personal/amazon/shopify, `email_verified_at`), `user_register_steps.is_passed`. **Core DB:** `users.connect_ainfluencer_affiliate_account`, `user_third_parties` (`provider`, `type`, `third_party_user_id`, `third_party_unique_identity`), `user_third_party_accounts`.
- **Frontend (affiliate):** `shared-components/algo-widgets/widget-user-menu/user-menu-dropdown.component.vue` (panel switch, "Create {0} Account?"), `shared-components/algo-auth/script/auth-init.script.js` (`/auth`, `/impersonate`), `shared-components/algo-auth/auth-email-verified.vue` (stores token and `?redirect=`).
