---
id: AFA-02
title: Connecting promotion platforms
domain: Affiliate app (creators)
audience: [affiliate, admin]
apps: [affiliate-app, email]
related: [AFA-01, AFA-04, AFX-01, AFC-02, INF-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-02 · Connecting promotion platforms

> Affiliates tell Ainfluencer where they promote products: social accounts connected by login (verified instantly), a website verified with a DNS record, or other accounts submitted for manual review by the Ainfluencer team.

## Who and where
- **Affiliates** in the affiliate app:
  - During onboarding, at the promotion-platform step (required, AFA-01).
  - Later at "Edit Profile" → **"Manage Platforms"** tab (`/app/profile?tab=manage_platforms`).
- **Admins** approve or reject manually submitted accounts in the affiliate admin panel.
- This is separate from the Instagram/TikTok/YouTube connection in the Ainfluencer mobile app (INF-01). Accounts are not copied between the two *(importing at first single sign-on probably fails, see AFX-01)*.

## How it works
**A. Automatic Verification ("Verified instantly")**
1. Under "Choose Your Platform", the affiliate picks **YouTube, TikTok, Instagram, Pinterest or Facebook**. ("X" is shown as "Coming Soon" and can't be clicked.)
2. A popup opens the platform's login and consent page. After they approve, the account and its channels/pages are saved as **Verified**.
3. Under "Choose Your Category", they must pick a category for **each** connected account. Without one the app shows "Choose platform category" and won't continue.

**B. Website (also under Automatic Verification)**
1. Pick "Website", choose a category and enter the domain. The app shows "Verification Instructions for <domain>" with a **TXT record** such as `ainfluencer-site-verification=…`.
2. The affiliate adds that TXT record at their domain provider (for example GoDaddy or Namecheap), then clicks "Verify".
3. If the record is found, the website becomes **Verified**. If not, the app shows "Verification Failed". The app reminds them that DNS changes can take a few minutes to 48 hours ("Retry to Verify").

**C. Manual Verification ("May take up to 12 hours for verification")**
1. Click "Add New Platform". For each platform, choose the type, a category, and the account handle (`@Platform-ID`).
2. The app then shows "Manual Verification Instructions" with a **code** for each account and instructions for where to put it (for example, in the bio). "Copy" copies the code.
3. Tick "I confirm that I have completed all necessary steps to verify the above accounts and they are now ready for Ainfluencer's review." and click **"Submit For Verification"**. The accounts move to **Waiting for approval**, and the onboarding step counts as done.
4. An admin checks the account and marks it **Verified** or **Rejected**. The affiliate is notified.

**Removing.** Each connected account can be deleted from "Manage Platforms".

## Rules and limits
- Onboarding needs **at least one** platform ("Add Minimum One Item") and a category for each one.
- Connecting by login or verifying a website completes the onboarding step immediately. Manual accounts complete it as soon as they are **submitted**, before an admin approves them.
- **Today an approved platform is not required** to request partnerships, accept invitations or create links. A future release (in testing, not in production) will require one.
- A rejected or still-pending manual account can be submitted again.
- Reconnecting a platform by login refreshes its channel list. Channels no longer returned by the platform are removed.

## Statuses
| Status shown | Internal value | Meaning |
|---|---|---|
| Verified | `verified` | Connected by login, website TXT found, or approved by an admin |
| Pending | `pending` | Manual account added (or website added) but not yet submitted/verified |
| Waiting for approval | `waiting_for_approval` | Manual account submitted; an admin will review it (up to 12 hours) |
| Rejected | `rejected` | An admin rejected it; it can be fixed and resubmitted |

## Timers and automatic actions
- No automatic re-checks. Manual review is done by people ("up to 12 hours" in the app; the welcome screen says "may take up to one day").

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Admin verifies or rejects a manual account ("✅ Social Media Account Verified" / "❌ … Rejected") | Affiliate | In-app immediately, email in the next 06:00 or 18:00 UTC batch |

## Common questions

### "I submitted my account for verification. How long does it take?"
**Answer for the user:** Manual checks are done by our team and usually take up to 12 hours (at most about a day). You'll get a notification and an email when it's reviewed. You can keep using the app meanwhile.
**What to check:** The account's status is "Waiting for approval" (`user_social_account_channels.system_status`).
**Escalate when:** It's still waiting after 24 hours (ask the admin team to review).

### "My account was rejected. Why?"
**Answer for the user:** Usually the verification code wasn't found where the instructions said, or the account doesn't match. Please follow the instructions again, check the code is visible, and resubmit.
**What to check:** The handle entered, and whether the code is still in place.
**Escalate when:** The user insists the code was in place. Ask the admin team to re-check.

### "Website verification keeps failing."
**Answer for the user:** Add the TXT record exactly as shown, on the main domain (without "www"), then wait. DNS changes can take from a few minutes up to 48 hours. Then click "Retry to Verify".
**What to check:** The domain entered (we strip `http://` and `www.`); whether a public DNS lookup shows the TXT record.
**Escalate when:** The TXT record is publicly visible but verification still fails.

### "Instagram connection shows an error page instead of coming back to the app."
**Answer for the user:** Sorry about that. The Instagram connection hit an error. Please try again, or add Instagram under Manual Verification instead.
**What to check:** Time of the attempt.
**Escalate when:** Always report it to dev (known issue below).

### "The app won't let me continue: 'Choose platform category'."
**Answer for the user:** Pick a category from the drop-down next to each connected platform, then continue.
**What to check:** —
**Escalate when:** The category list is empty.

### "I connected Instagram in the Ainfluencer app. Why isn't it here?"
**Answer for the user:** The affiliate platform keeps its own list of promotion platforms. Please connect it here too.
**What to check:** —
**Escalate when:** —

## Known issues
- **Instagram login errors show a technical error page** instead of returning to the app. Workaround: retry, or use Manual Verification. Escalate to dev.
- **"X" (Twitter)** is shown but disabled ("Coming Soon").
- The welcome text says review "may take up to one day", while the manual section says "up to 12 hours".

## Related topics
- AFA-01 · Onboarding (this is the required platform step).
- AFA-04 · Partnerships. A verified platform will be required there in a future release.
- AFX-01 · Single sign-on from Ainfluencer.
- AFC-02 · Notification settings and email batching.
- INF-01 · Social accounts in the Ainfluencer mobile app (a separate system).

## Technical reference
- **Endpoints:**
  - OAuth: `GET affiliate/v2/social-medias/oauth/{youtube|tiktok|instagram|pinterest|facebook}` (returns `redirect_link`) → callbacks `GET web/v2/affiliate/social-medias/login/{platform}/callback` → `Services/Actions/Affiliate/SocialMedia/V2/OAuth/*LoginCallbackService` → `SocialMediaService::syncAccount` / `syncAccountChannels`.
  - `GET affiliate/v2/social-medias`, `DELETE affiliate/v2/social-medias/{id}`, `PUT affiliate/v2/social-medias/category`.
  - Website: `POST affiliate/v2/social-medias/websites/social-verification-token`, `POST .../websites/verify` (`V2/Website/VerifyWebsiteSocialService`, DNS TXT lookup).
  - Manual: `POST affiliate/v1/social-medias` (creates `pending` + 10-character token), `PUT affiliate/v1/social-medias/verify` (→ `waiting_for_approval`, throttle 50/hour), `PUT/GET affiliate/v1/social-medias/{id}`, `POST affiliate/v1/social-medias/delete`, `GET affiliate/v1/social-medias/types`, `GET affiliate/v1/categories`.
- **Data:** `user_social_accounts` (`user_type_id`, `platform`, `account_id`, `system_status`, `verification_type`), `user_social_account_channels` (`identifier`, `category_id`, `system_status`, `token`, `verified_at`, `followers_count`).
- **Notification:** `Notifications/SocialMedia/UpdateStatusSocialMediaNotification` (mail + web, via NotificationQueue).
- **Frontend:** `shared-components/algo-components/platforms-v2/platform.component.vue`, `website-platform.modal.vue`, `spa-affiliate/src/components/auth/auth-register-steps-platforms-v2.component.vue`, profile tab "Manage Platforms".
