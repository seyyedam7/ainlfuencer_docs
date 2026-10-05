---
id: BRD-01
title: Brand onboarding
domain: Brand account and team
audience: [brand]
apps: [web]
related: [ACC-01, ACC-02, BRD-02, BRD-04, CMP-01, DSC-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# BRD-01 · Brand onboarding

> The short questionnaire a brand sees right after confirming its email. Only the **business name** and **location** are required; after those two answers the brand can use the dashboard. The other screens can be skipped.

## Who and where

- Brands, web app `/onboarding` (page title "Welcome"). Every dashboard page sends a brand here until onboarding is complete.
- Team members invited into an existing brand don't go through it for that brand (BRD-03).
- Brands that installed the Shopify app first set a password (ACC-03), then come here.

## How it works

1. **"What is your business name?"** The brand types the name and clicks "Next". Links or contact details are refused: "Do not include links or contact info in the business name field." Empty: "Please enter a name". This step creates the brand's company profile.
2. **"Where is your business located?"** The brand picks a country and clicks "Next" ("Please select a location" if empty). **From this point onboarding counts as complete**: if the brand leaves now, the dashboard works on the next visit.
3. Six optional screens follow, each with "Next" (saves) and "Skip":
   - "How large is your company?"
   - "Select a category:"
   - "What is your preferred social media platform for influencer marketing campaigns?"
   - "What type of influencers do you need for your campaigns?"
   - "Upload your brand logo" (JPG/JPEG/PNG, max 8 MB)
   - "What would you like to achieve with influencers?" (up to 2: "Please select maximum 2 objectives")
   A "Start Exploring Influencers" button (on phones: "Skip Onboarding") at the top jumps straight to influencer search.
4. **Final screen**: "Hi {business name}!" with a welcome message and two buttons: **"Create Your First Campaign"** (opens the campaign-type picker, CMP-01) and **"Start Exploring Influencers"** (opens `/search`, DSC-01).
5. "Back" returns to the previous screen and "Logout" is available on every screen.

## Rules and limits

- Required: business name and location (country). Everything else is optional and can be edited later on `/profile` (BRD-02).
- Onboarding is checked when the brand logs in and on every dashboard page. If the business name or location is missing, the brand is sent back to `/onboarding` and resumes at the missing question.
- The logo must be an image (JPG/JPEG/PNG), up to 8 MB.
- The business name becomes the brand name influencers see.
- An admin can only suspend a brand that has started onboarding (BRD-04).

## Statuses

| What the brand sees | Meaning | Next |
|---|---|---|
| Sent to `/onboarding`, first question | No business name yet | Answer "What is your business name?" |
| Sent to `/onboarding`, location question | Name saved, no location | Choose a country |
| Dashboard opens | Name and location saved | Optional screens can still be filled on `/profile` |

## Timers and automatic actions

None.

## Notifications

None.

## Common questions

### "I keep being sent back to the onboarding page."
**Answer for the user:** We need your business name and your location before you can use the dashboard. Please answer those two questions; the rest is optional and you can click "Skip".
**What to check:** The brand profile has a name and a country (`identities` of type brand: `name`, `country_id`).
**Escalate when:** Both are filled and the brand is still redirected (try a fresh login first).

### "Can I skip onboarding?"
**Answer for the user:** Only the business name and location are required. After those, click "Skip" or "Start Exploring Influencers" / "Skip Onboarding".
**What to check:** Nothing.
**Escalate when:** Never.

### "I made a mistake in my business name or location."
**Answer for the user:** You can change them anytime on your Profile page ("Company Name", "Location").
**What to check:** If the user is a team member inside another brand's team, they can't edit the profile (BRD-03).
**Escalate when:** Saving fails.

### "My logo won't upload."
**Answer for the user:** Please use a JPG or PNG image under 8 MB. You can also skip this and add the logo later on your Profile page.
**What to check:** File type and size.
**Escalate when:** A valid small image fails repeatedly.

### "It says no links or contact info in the business name."
**Answer for the user:** The business name can't contain a website address or email. Please enter just the name of your business.
**What to check:** Nothing.
**Escalate when:** A plain name is refused.

## Known issues

- The progress bar counts 8 screens, which can make brands think all of them are required. Only the first two are.
- Older internal documents say the optional screens are switched off. They still appear (with "Skip") once the brand finishes the first two in one session.

## Related topics

- ACC-01 / ACC-02: sign-up and login lead here.
- BRD-02: editing the company profile later.
- BRD-04: suspension needs a started onboarding.
- CMP-01: "Create Your First Campaign".
- DSC-01: "Start Exploring Influencers".

## Technical reference

- **Endpoints**: `GET /api/v3/brand/on-boarding` (current answers) and `POST /api/v3/brand/on-boarding` (`request` = `business-name`, `location`, `company-size`, `category`, `platforms`, `target-influencers`, `logo`, `objectives`) → `BrandOnBoardingController` → `app/Modules/BrandOnBoarding/BrandOnBoardingFactory.php` (one handler class per step, e.g. `BusinessName.php`, `Location.php`). Funnel tracking: `POST /api/v2/funnels`, `GET /api/v2/funnels/processes` (process `brand-onboarding`). `GET /api/v2/user/brand-information` is also read by the layout guard.
- **Data**: the brand `identities` row (`type = brand`, `name`, `country_id`, description, logo, categories) and `brand_details`.
- **Web**: `src/components/pages/onboarding` (`model.ts` `getOnboardingStep`: only `business_name` and `location` are checked; `parts/forms/index.tsx` renders the 8 screens); guard `src/components/hoc/auth`. Strings `locales/en/onboarding.json`.
