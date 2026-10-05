---
id: INF-01
title: Connecting a social account
domain: Influencer social accounts
audience: [influencer, admin, system]
apps: [mobile, background]
related: [INF-02, INF-03, INF-04, INF-05, INF-06, ACC-01, OFR-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# INF-01 · Connecting a social account

> Before an influencer can do anything on the marketplace, they connect an Instagram, TikTok or YouTube account in the mobile app and prove they own it. Instagram is proved by sending us a code in a direct message; TikTok and YouTube are proved by signing in with the platform. After a successful connection the app moves on to the media-kit questions (INF-02).

## Who and where

- **Influencer**, mobile app only. The web app sends influencer accounts to a "download the app" page.
- Ways in:
  - Explore opens the onboarding "Welcome" screen by itself about 3 seconds after login when the user has no social account. "Get started" leads to platform selection.
  - Profile tab → Social Accounts "+ Add"; Home → "Add Social Account" / setup tasks; Explore account selector "Add Your Social Account".
  - Campaign details when the user has no usable account.
- Screens: **"Connect your platforms"** (platform picker: Instagram, TikTok, YouTube; header "1 of 3"), Instagram username screen ("add your instagram", "1 · Enter your username", "Next"), Instagram verification screen ("verify @username").
- Facebook and X **cannot** be connected. They only appear as share targets.

## How it works

### Instagram (code by direct message)

1. The influencer picks Instagram and types the username. The app checks the username:
   - If another Ainfluencer user already connected it: "This Instagram account has already been added by another Ainfluencer user. If you own this username, please contact the Ainfluencer support team."
   - Otherwise Ainfluencer creates a **6-digit code**. Asking again creates a new code and the old one stops working.
2. The verification screen shows the code as **`ain-123456`** with three steps:
   - "1 · Copy your code"
   - "2 · Follow @ainfluencer.official & DM the code". The button **"Open Instagram & Follow + DM"** copies the code and opens a DM to @ainfluencer.official.
   - "3 · Click the button below to confirm" → **"Confirm you did it"**.
3. When the DM arrives, an outside DM-reading service passes it to Ainfluencer. Ainfluencer answers in the DM:
   - code accepted: "Thanks! We received the code for @username and we are setting your account up right now…"
   - code replaced by a newer one, code sent from a different account, code missing or invalid: a matching explanation (see Common questions).
   The account itself is then built in the background (profile data, picture, search index).
4. "Confirm you did it" waits 6 seconds, then asks Ainfluencer whether the DM arrived:
   - **Verified** (or still being set up after the DM): the app continues to the media kit (INF-02), or to Profile if the account was already onboarded.
   - **Under manual review**: a "Submitted" pop-up: "Your account will be reviewed by the admins shortly… This process may take up to 24 hours and you'll be notified by email." OK → Profile.
   - **No DM found**: alert "We couldn't find your verification code. Please send the verification code as a separate DM message without any other text, then try again."
   - On the **2nd failed check** the app sends the account to **manual review automatically** and shows "Submitted".
5. Manual review is done by an admin (INF-03). The influencer is told by email and push.

The old "put the code in your bio" method is switched off in the app. Some old texts still mention the bio (see Known issues).

### TikTok (sign in with TikTok)

1. The influencer picks TikTok and approves access in the TikTok app (basic info, profile and stats).
2. Ainfluencer reads the account from TikTok and checks it (see Rules). If accepted, the account is created and the app goes to the media kit.
3. Possible answers: new account → media kit; already onboarded → Profile; already linked to another user → error and stays on platform selection; timeout → alert. The request is not retried automatically.
4. After the media kit, TikTok users see an extra **"Prove it's you, then you're live"** screen asking them to follow and message **@ainfluencer2024** with a code, then **"Verify & go live"** → "You're verified" screen. **This screen does not check anything on the server.** The account is already connected at this point; its review status is decided by INF-03, not by this screen.

### YouTube (Google sign-in in a browser)

1. The influencer picks YouTube. The app opens Google's sign-in page in a browser window (account chooser + consent).
2. After consent the app sends the result to Ainfluencer:
   - The Google account has several channels → **"Which YouTube channel?"** picker → "Continue".
   - Success → media kit. Already linked or already onboarded → Profile.
   - Error 400 → "Try again" (for example the Google code expired: sign in again). Error 500 → "Something went wrong". User cancels → nothing happens.
3. Each channel becomes its own Ainfluencer account. A channel that breaks the rules or belongs to another user is skipped; if all are skipped the influencer sees the first reason.

## Rules and limits

- One social account can belong to only one Ainfluencer user. If someone else already connected it, the connection is refused with the "already been added by another Ainfluencer user" message. Support must not move accounts between users without the dev team.
- **TikTok and YouTube** (production only): the account must be public and have at least the minimum followers set by the team (default seed value **100**; admins can change it). Messages: "Your … account must be public…" and "…does not meet the minimum number of followers required…".
- **Instagram**: the private-account rule is skipped (owning the DM proves ownership). See Known issues for the follower rule.
- Manual review attempts (all platforms): after a rejection the influencer can resubmit; after **5 rejected requests** within the retry window (**7 days**, default settings) they are blocked: "…five verification requests have previously been submitted for this account, which were rejected by the admins… you won't be able to submit a new verification request for the next 7 days." While a request is pending, a new one is refused: "You currently have a pending request for this account verification…"
- A newly connected account starts as **Draft** and cannot make offers until the media kit is finished (INF-02) and it passes review (INF-03).

## Statuses

| What the influencer sees (Profile account card) | Meaning | Next |
|---|---|---|
| "Pending Verification" | Ownership not proven automatically; waiting for an admin (manual review, `temporary_integrations.status = pending`) | Admin verifies or rejects (INF-03) |
| "Verification Rejected" + reasons pop-up | Admin rejected the manual request (`rejected`) | "Resubmit Verification Request" (INF-03) |
| "Draft" + "Complete Profile" | Connected, media kit not finished (`integrations.qc_status = draft`) | Finish INF-02 |
| "Pending Admin Approval" | Media kit finished, waiting for quality review (`pending`) | INF-03 |
| "Active" | Approved | Can make offers |

## Timers and automatic actions

- The app waits 6 s before each status check.
- Every 30 minutes a background job fills in profile data (followers, picture) for Instagram manual-review requests, so admins can judge them (INF-06).
- Manual review has no automatic deadline; the "up to 24 hours" text is a promise, not a timer.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| DM received | Influencer | Reply in the Instagram DM (from the DM service) |
| Manual request verified by admin | Influencer | Email ("You're In!…"), push and in-app: "Congratulations! Your Instagram @username was approved for Ainfluencer Marketplace." |
| Manual request rejected | Influencer | Email and push |

## Common questions

### "I sent the code but the app says it couldn't find it."
**Answer for the user:** Please send the code exactly as shown (for example `ain-123456`) as its own message, from the account you are connecting, to @ainfluencer.official, then tap "Confirm you did it" again. If it still fails, the app sends your account for manual review automatically.
**What to check:** that they DMed from the same username they typed; that they used the newest code (a new code is created every time they re-enter the username); the latest `integration_verifications` row for the user and username (`verified_at` filled = DM received).
**Escalate when:** `verified_at` is filled but no account appears after 30 minutes and there is no pending manual request.

### "The DM reply says my code was replaced / belongs to another account."
**Answer for the user:** "Replaced" means you generated a newer code; open the app, copy the code shown now and send that. "Different account" means the DM came from another Instagram profile than the one you're connecting; send it from the account you want to connect.
**What to check:** username typed in the app vs. the account that sent the DM.
**Escalate when:** the usernames match and the reply is still wrong.

### "My account has been 'Pending Verification' for days."
**Answer for the user:** Manual reviews are done by our team; we'll email you as soon as it's reviewed. Please keep your account public.
**What to check:** `temporary_integrations` row (status `pending`, `last_tried_at`).
**Escalate when:** pending for more than 48 hours (manual review backlog).

### "It says my account was already added by another user."
**Answer for the user:** This account is already connected to another Ainfluencer login. If it's yours (for example an old login), tell us both email addresses and we'll look into it.
**What to check:** which user owns the integration with that username/external id.
**Escalate when:** always, if the user claims ownership. Never transfer accounts without the dev team.

### "TikTok/YouTube says I don't have enough followers."
**Answer for the user:** TikTok and YouTube accounts need to be public and have a minimum number of followers to join. Please try again once you meet it.
**Escalate when:** the account clearly meets the minimum shown in the message.

### "The TikTok screen said 'You're verified' but my account is still pending / brands can't book me."
**Answer for the user:** That screen is a final reminder to follow us; your account still goes through our normal review, which can take up to 24 hours.
**What to check:** the account's status on Profile (`qc_status`).
**Escalate when:** pending review for more than 48 hours.

### "YouTube sign-in says try again."
**Answer for the user:** The Google sign-in expired or was already used. Start the YouTube connection again and finish it in one go.
**Escalate when:** it fails repeatedly with "Something went wrong".

### "I can't submit another verification request."
**Answer for the user:** After 5 rejected requests you need to wait 7 days before trying again. Please fix the rejection reasons first.
**What to check:** `temporary_integrations.tries`, `last_tried_at`, `status`.

## Known issues

- **Instagram DM verifications may end up in manual review even when everything is right** *(to verify)*: in production the follower check for Instagram uses a value that is never loaded for Instagram, which makes the automatic step fail and fall back to manual review whenever the profile data provider returns follower numbers. Influencers then see "Pending Verification" instead of going straight on. Workaround: admins verify manually. Escalate to dev if many accounts pile up.
- Old texts still mention the bio: the manual-review pop-up says "Please do not remove the 6-digit verification code from your Instagram bio" even though the flow is DM-only. Tell users they don't need the code in their bio.
- TikTok "Verify & go live" makes no server check (see How it works).
- Facebook and X cannot be connected.

## Related topics

- INF-02: media-kit questions after connecting.
- INF-03: manual review and quality review.
- INF-04: suspension and appeal.
- INF-05: managing connected accounts.
- INF-06: background refresh of account data.

## Technical reference

- Endpoints (mobile, prefix `api/{locale}/v3/mobile`):
  - `POST influencer/on-boarding/profile/lookup` → `Modules/Mobile/Http/Controllers/API/OnboardingController@store` → `InfluencerOnBoarding/Steps/Bypassable/IntegrationProfileLookup` (creates `integration_verifications` row, token `mt_rand(100000, 999999)`, soft-deletes previous).
  - `POST influencer/on-boarding/profile/verification` → `OnboardingController@integrationAccountVerification` → `Steps/Bypassable/IntegrationAccountVerification` (TikTok/YouTube OAuth paths `tiktokOAuthVerification`, `youTubeOAuthVerification`; `guardAgainstClaimedAccount`; `checkRulesViolation` only when `isProduction()`; min followers from `settings` `suspension_account_followers_count` per platform). Creates via `Steps/Bypassable/CreateIntegration` (`qc_status` default `draft`).
  - `GET influencer/integration-verifications/status`, `POST …/manual` → `IntegrationVerificationStatusController` → `Modules/Mobile/Services/Implementation/IntegrationVerificationStatusService` (`dm_received`, `is_verified`, `is_processing`, `is_under_manual_review`; platform defaults to Instagram).
  - `GET influencer/youtube/consent-screen`, `GET influencer/youtube/callback` → `YouTubeOAuthController`.
  - Webhooks: `POST api/v3/webhooks/instagram/dm-verification`, `…/tiktok/dm-verification` → `InfluencerOnBoarding/Http/Controllers/DmVerificationWebhookController` → `Services/DmVerificationService::verify` → job `Jobs/CompleteDmVerificationJob` (queue `influencer-onboarding:dm-verification`) → `Steps/Bypassable/InstagramDmAccountVerification` / `TiktokDmAccountVerification`. Code regex `ain[\s\-_]*(\d{3,10})`.
- Known-issue source: `IntegrationAccountVerification::$minFollowersCount` is only set for TikTok/YouTube; `InstagramDmAccountVerification::checkRulesViolation` reads it; the resulting error is caught and turned into `updateOrCreateTemporaryIntegration()` for Instagram.
- Manual review: `Services/Implementation/Mobile/TemporaryIntegrationService` (`store`, `checkIsPending`, `checkHasBeenRejected`); model `Models/v2/TemporaryIntegration` (settings `maximum_tries` = 5, `tries_time_interval` = 7 in `database/seeders/data/production_settings.json`).
- Data: `integration_verifications` (user_id, username, platform_id, token, verified_at, verification_metadata), `temporary_integrations` (status pending/rejected/verified, tries, last_tried_at, rejection_reasons), `integrations` (qc_status, user_id, username, external_id, platform_id), `identity_integration`.
- Mobile: `features/onboarding/screens/getUserName/{GetUserNameOnboarding,VerifyInstagramScreen,VerifyTikTokScreen}.tsx`, `features/onboarding/hooks/useAccountVerification.ts` (6 s delay, auto manual after 2 fails), strings `localization/translations/en/onboarding.js`.
- Language files: `resources/lang/en/instagram_dm_verification.php`, `manual_verification.php`, `integration_data_provider.php`, `integration.php` (`username-taken`).
