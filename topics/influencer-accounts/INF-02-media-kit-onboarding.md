---
id: INF-02
title: Media-kit onboarding
domain: Influencer social accounts
audience: [influencer]
apps: [mobile]
related: [INF-01, INF-03, INF-05, OFR-01, MKT-04, PUBL-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# INF-02 · Media-kit onboarding

> After a social account is connected (INF-01), the app asks six short sets of questions that make up the influencer's "media kit": bio, niche, audience, photos, shipping address and goals. Finishing the last step sends the account to Ainfluencer's quality review (INF-03) and gives the influencer a shareable media-kit link.

## Who and where

- **Influencer**, mobile app. The steps run per social account (each account has its own media kit).
- The app opens the first unfinished step automatically after connecting, and again whenever a **Draft** account tries to do something that needs a complete profile (Explore, campaign details, Make Offer show a "remaining steps"/"Complete profile" prompt).
- Profile → account card → **"Complete Profile"** (Draft) or **"Edit Profile"** (Pending / Active) reopens the steps, pre-filled.
- Each screen shows "STEP x OF 6".

## How it works

1. **Media kit** — "Let's build your media kit": bio (20–700 characters), language, gender, date of birth, optional profile photo. The bio is pre-filled from the social profile. Contact details in the bio (website, email, username, phone) are blocked: "Your bio contains your contact information, which goes against our platform's policies…". You must be at least 13: "You must be at least 13 years old to join our community."
2. **Niche** — "What's your world?": 1–10 categories and at least one hashtag.
3. **Audience** — "Who's following you?": up to 3 audience locations, post topics and audience description (at least 20 characters each), and **at least one insights/analytics screenshot (required)**.
4. **Photo gallery / past work** — "Show your proof": up to 10 photos and optional past collaborations (an Instagram post link or an upload). **"Finish later"** skips the step.
5. **Shipping address** — "Get free products": label, full name, street address, country, city, zip, phone, map pin. **"Add address later"** skips. This address is used for gifting (MKT-04) and is managed later in Settings → Shipping Address (INF-05).
6. **Goals** — "What do you want?": pick at least one goal ("Please select at least one goal!").
7. **"Your media kit is live"** (Congratulations) with **"Share my media kit"** / copy link. The link is the public profile page `app.ainfluencer.com/public/profile/<id>` (PUBL-02). Instagram and YouTube users go to Explore; TikTok users first see the "Verify & go live" screen (INF-01).

What the system does:
- Each step is saved to the server as soon as the user taps continue, so the app can resume at the first unfinished step.
- Saving the Goals step moves the account to **"Pending Admin Approval"** (quality review, INF-03) and marks onboarding as finished.

## Rules and limits

- A **Draft** account (media kit not finished) cannot send offers: "Your account for Instagram ID @username is in draft status on our platform. To make an offer, please complete your full profile and wait for admin approval."
- Rates and "Supercharge" steps exist in the code but are switched off.
- The older 15-step onboarding is not reachable anymore.
- Language: the app has English and Spanish texts but no way to switch language.

## Statuses

| Account card | Meaning (`integrations.qc_status`) | Next |
|---|---|---|
| "Draft" + "Complete Profile" | Steps not finished (`draft`) | Finish the steps |
| "Pending Admin Approval" + "Edit Profile" | Goals saved, waiting for review (`pending`) | INF-03 |
| "Active" + "Edit Profile" / "Share" | Approved (`active`) | — |

## Timers and automatic actions

- None. Each save is immediate. Finishing Goals also re-indexes the account for brand search and starts a data fetch for it.

## Notifications

None for this flow. The review result is notified under INF-03.

## Common questions

### "The app keeps sending me back to finish my profile."
**Answer for the user:** Your account still has unfinished media-kit steps. Please complete all six steps; the app opens the first one you haven't finished.
**What to check:** the account's `qc_status` (`draft`) and which steps are recorded as passed.
**Escalate when:** all steps are done, `qc_status` is still `draft`, and saving Goals fails.

### "It says my bio contains contact information."
**Answer for the user:** Bios can't include emails, phone numbers, websites or other handles. Please remove them and continue.

### "I can't continue on the Audience step."
**Answer for the user:** The audience step needs at least one screenshot of your Instagram insights / TikTok or YouTube analytics, plus a short description of your audience and topics (20+ characters each).

### "Where is my media-kit link?"
**Answer for the user:** On Profile, tap "Share" on your account card (active accounts), or copy it from the last onboarding screen. It's your public Ainfluencer profile page.

### "I finished everything — why can't I make offers yet?"
**Answer for the user:** Your account is waiting for our team's approval, which can take up to 24 hours. You can browse campaigns meanwhile.
**What to check:** `qc_status = pending` → INF-03.

## Known issues

- **Editing an active profile sends it back to review** *(to verify)*: re-saving the Goals step through "Edit Profile" sets the account to "Pending Admin Approval" again, even if it was already Active. Offers still work while pending (they are held as temporary offers, INF-03). If an active influencer suddenly shows "Pending Admin Approval" after editing their profile, this is why; escalate to an admin to re-activate.
- The final "Congratulations" texts in old translations say "Your Instagram was added successfully" for every platform.

## Related topics

- INF-01: connecting the account. INF-03: review after onboarding. INF-05: editing name, categories, picture and addresses later. MKT-04: gifting uses the shipping address. PUBL-02: the public media-kit page.

## Technical reference

- Endpoints (`api/{locale}/v3/mobile`):
  - `POST influencer/on-boarding` `{request: <step>, integration_id, …}` → `Modules/Mobile/Http/Controllers/API/OnboardingController@createInfluencerOnboarding` → `Modules/InfluencerOnBoarding/InfluencerOnBoardingFactory` (steps in `Modules/InfluencerOnBoarding/Steps/*`, list `Lists/InfluencerOnBoardingList`: address, biography, date-of-birth, gender, locations, language, audience, post-topics, profile-photo, categories, hashtags, insights, past-collaborations, goals, photo-gallery). Validation `Http/Requests/InfluencerOnBoardingRequest`.
  - `POST influencer/on-boarding/finished` → `OnboardingController@setOnboarded` (`integrations.is_onboarded`).
  - Lookups: `GET categories`, `GET social/hashtags/search`, `GET social/medias/feed`, `GET platforms/{platform}/accounts/data`, `GET goal/questions`, `GET api/{locale}/v3/locations/countries`; funnel logging `POST funnels`.
- `Steps/Goals::updateIntegration` sets `qc_status = pending` unconditionally (source of the known issue).
- Data: `integrations.qc_status`, `integrations.is_onboarded`, `passed_onboarding_steps` (per integration), `influencer_details` (objectives, etc.), `addresses`.
- Mobile: `features/onboarding/screens/{mediaKit,niche,Audience,showcase,ShippingAddress,Goals,Congratulation}`, constants `mediaKit/constants.ts` (BIO_MAX 700, TOTAL_STEPS 6), resume `mapNewFlowResumeRoute`.
