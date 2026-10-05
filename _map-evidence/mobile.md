# Ainfluencer influencer mobile app: live-code inventory

Source: snapshot of `develop` (store version 32.2.2, CodePush target `32.2.2`) at `Mobile/mobile-app (develop snapshot)`. Paths are relative to `src/` unless they say otherwise.

**Method.**
1. An import walker (`(generator script)`) starts at `index.js` and resolves `@app/*` aliases, relative imports, and `index.*` files. Result: 1486 code files are reachable and 275 are not (3 of the 275 are test files).
2. Screen reachability is decided separately, by searching for inbound `navigate`/`push`/`replace`/`reset` calls and deep links/linking config on each route constant or string.
3. A call is marked `live` only when all of these hold:
   - its file is reachable;
   - its function is actually *used* as a runtime value, not just imported for types;
   - at least one screen that uses it can be navigated to.

The machine-readable list is in `inv/mobile-calls.json` (178 rows; 146 live, which is 145 unique method+path pairs).

**Base URLs**
- `main`: `BASE_URL` = `https://influencers.ainfluencer.com/api/en` (prod) or `https://api-staging.ainfluencer.com/api/en` (staging/dev), used through `axiosInstance`.
  - **Note:** axios *appends* paths that start with `/` to the baseURL. So `/finance/...` and `/v3/...` both resolve under `/api/en/` and never from the host root.
- `api_no_locale`: `BASE_URL` with `/en` removed (`…/api/…`). Used by `ADS_ENDPOINTS.AD_METRICS`, `BRANDID_APIS.GET_LINK`, notify-settings, and suspension appeal.
- `chat`: `https://chat.ainfluencer.com`, or `https://chat-staging.ainfluencer.com` only when `ENV==='staging'`. The `development` env therefore talks to **prod** chat. Used by the socket.io connection and the REST message history.
- `other`: Mapbox, the TikTok proxy (`tiktok-api-staging.ainfluencer.com/tiktok`), and so on.

## 1. Live screens

**Navigators**
- `RouteStack` is the root native stack (`navigation/RouteStack.tsx`).
  - It always shows `SPLASH_SCREEN`, unless the user just logged out.
  - When **logged out** it registers only `AUTH_NAVIGATOR`.
  - When **logged in** it registers every entry in `navigation/routes.ts`.
- `MAIN_NAVIGATOR` holds the bottom tabs (`navigation/TabNavigator.tsx`, initial tab **Explore**).
  - It also starts the chat socket and shows the inbox unread badge, from GET `v3/mobile/chats`.
- `ExploreTopTabs` (`navigation/ExploreTabs.tsx`) sits inside the Explore tab.
- `ONBOARDING_FLOW` is its own stack (`navigation/OnboardingFlow.tsx`).
- `AUTH_NAVIGATOR` is the auth stack (`navigation/AuthNavigator.tsx`).

"Reached from" lists the callers I found in code (`navigate`/`replace`/`reset`). "DL" means deep link and "Push" means push notification.

### App start and auth
| Screen | Component | Navigator | Reached from | Purpose |
|---|---|---|---|---|
| SPLASH_SCREEN | features/splash/screens/SplashScreen.tsx | Root | App start | 1. Checks for an app update with GET `v3/mobile/versions`. A newer store version shows a mandatory update. 2. CodePush check. 3. With a token **and** a verified email it resets to MAIN; otherwise to AUTH. Splash does not check onboarding. |
| LOGIN_SCREEN | features/auth/screens/login/LoginScreen.tsx | Auth (initial) | Splash/logout/401 refresh failure, sign-up link, reset success | Email+password login and Google Sign-In. Verified users go to MAIN; unverified users go to CONFIRM_EMAIL. |
| SIGN_UP_SCREEN | features/auth/screens/signup/SignupScreen.tsx | Auth | Login "sign up" | First name, last name, email, password; also Google. Then CONFIRM_EMAIL. |
| CONFIRM_EMAIL_SCREEN | features/auth/screens/verifyEmail/VerifyEmailScreen.tsx | Auth | After sign-up, unverified login, forgot-password | Enter the emailed code. In sign-up mode a correct code leads to MAIN; in reset mode it leads to RESET_PASSWORD. Also offers Resend and "change email". |
| CHANGE_EMAIL_SCREEN | features/auth/screens/changeEmail/ChnageEmailScreen.tsx | Auth | Verify screen | Fix a mistyped email (sign-up) or re-send the reset code to a different email. |
| FORGET_PASSWORD_SCREEN | features/auth/screens/resetPassword/ResetPasswordScreen.tsx | Auth | Login "forgot password" | Enter email; sends the reset code. |
| RESET_PASSWORD_SCREEN | features/auth/screens/resetPasswordFormScreen/ResetPasswordFormScreen.tsx | Auth | Verify screen (reset mode) | Set a new password, then back to Login. The new password must differ from the old one. |

### Main tabs
| Screen | Component | Navigator | Reached from | Purpose |
|---|---|---|---|---|
| EXPLORE_SCREEN | features/explore/ExploreScreen.tsx | Tabs (initial) | Default tab; Home "discover more"/empty states; after onboarding; Push default | Campaign marketplace. It has: a social-account selector (with an auto-redirect to onboarding when the user has no account), search, top tabs, an appeal modal, the YouTube "is live" popup, and the support FAB. |
| HOME_SCREEN | features/home/HomeScreen.tsx | Tabs | Tab bar; cold-start invitation push | Dashboard widgets (see §2). |
| SCHEDULER_SCREEN | features/scheduler/SchedulerScreen.tsx | Tabs | Tab bar; Home schedules; Push publish-job published/failed; DL `scheduler/:tab` | Deals by status: Scheduled, Published, Failed tabs. Data from GET `v3/mobile/deals/{type}`. |
| INBOX_SCREEN | features/inbox/InboxScreen.tsx | Tabs | Tab bar; Home offers "see all"; cold-start chat push | Chat threads and rooms from `v3/mobile/chats`, with pin, archive, unarchive. Opens CHAT_SCREEN, or CHAT_THREADS_SCREEN for a brand room with several threads. |
| PROFILE_SCREEN | features/profile/ProfileScreen.tsx | Tabs | Tab bar; after verification submitted/errors; account pushes; DL `profile` | Influencer profile: social account cards (status, appeal, resubmit, sync, edit), earnings → Wallet, Settings. |

### Explore top tabs (inside EXPLORE_SCREEN)
| Tab | Component | Data |
|---|---|---|
| AllAds (initial) | features/explore/screens/allAds/AllAdsScreen.tsx | GET `v3/mobile/ads/all`, `ads/all-add-once`, plus affiliate and gifting sections (gifting section → GIFTING_CAMPAIGNS) |
| affiliate | features/explore/screens/affiliate/AffiliateScreen.tsx | GET `v3/mobile/ads/affiliate`, affiliate report summary |
| gifting | features/explore/screens/gifting/GiftingScreen.tsx | GET `v3/mobile/product-seedings` (+ `requests`) |
| most_paid / for_you / country / category | mostPaidAds, forYouAds, myCountryAds, myCategoryAds | GET `v3/mobile/ads/most-paid` / `for-you` / `my-country` / `my-categories` |
| Invited | features/explore/screens/invitedAds/InvitedAdsScreen.tsx | GET `v3/mobile/invitations` |

### Campaigns, offers and invitations
| Screen | Component | Navigator | Reached from | Purpose |
|---|---|---|---|---|
| SEARCH_ADS_SCREEN | features/explore/screens/SearchAds/SearchAdsAcreen.tsx | Root | Explore top bar search | Searches campaigns through `ads/all` with a search query. |
| CAMPAIGN_DETAILS | features/ad/screens/campaignDetails/campaignDetalsScreen.tsx | Root | Explore lists, search, brand profile, chat header/options, invitations, scheduler item, InvitationRouter (DL) | Paid campaign detail. Records an ad view, and has a ⋯ menu with Report and Block brand. Its CTA reads "Make offer", "Accept invitation" or "View offer", and leads to MAKE_OFFER, or to onboarding when the user has no valid account. |
| MAKE_OFFER_SCREEN | features/make-offer/MakeOfferScreen.tsx | Root | Campaign details | Builds and submits an offer (§2). |
| InfluencersContent | features/ad/screens/influencers-content/index.tsx | Root | Campaign details "published influencers" section | Content other influencers already published for the ad. |
| AFFILIATE_DETAIL | features/explore/screens/affiliate/AffiliateDetailScreen.tsx | Root | Explore, brand profile, chat offer card, invitations, AffiliateInvitationRouter (DL) | Affiliate campaign: products, join (assign influencer), message the brand. |
| AFFILIATE_ANALYTICS | features/explore/screens/affiliate/AffiliateAnalyticsScreen.tsx | Root | Affiliate detail, product item, Home affiliate-performance widget | Affiliate reports: summary, timeline, products, payouts. |
| GIFTING_CAMPAIGNS | features/explore/screens/gifting/GiftingCampaignsScreen.tsx | Root | AllAds gifting section "see all" | Full gifting (product-seeding) list. |
| GIFTING_DETAIL | features/explore/screens/gifting/GiftingDetailScreen.tsx | Root | Explore/gifting lists, brand profile, chat offer card, invitations, GiftingInvitationRouter (DL) | Product seeding: apply or respond, address, request status, share posts, message the brand. |
| INVITED_CAMPAIGN_DETAILS | features/home/screens/InvitedCampaignScreen.tsx | Root | Home invitations widget, invitation pushes, DL `invitations`, InvitationRouter without ad_id | List of brand invitations. Marks them seen, can hide them, and opens the right detail screen (paid, affiliate or gifting). |
| BRAND_PROFILE | features/brand-profile/BrandProfileScreen.tsx | Root | Campaign/affiliate/gifting headers, chat offer card/options | Brand info and tabs (All, Affiliate and Gifting campaigns, Ongoing offers). The menu has About and **Block**; the app has no unblock UI. |
| InvitationRouter / AffiliateInvitationRouter / GiftingInvitationRouter | features/{InvitationRouter,AffiliateInvitationRouter,GiftingInvitationRouter}/index.tsx | Root | Deep links only (§3) | Loading screens that resolve `ad_id` and `replace()` to the right detail screen. |

### Chat and offer management
| Screen | Component | Navigator | Reached from | Purpose |
|---|---|---|---|---|
| CHAT_SCREEN | features/chat/screens/ChatScreen.tsx | Root | Inbox, offer pushes, DL `chat/:offerId`, Home schedules, scheduler, brand profile, wallet transaction rows, Explore (after offer) | Offer/deal chat. Messages travel over the socket (history over REST). The offer card shows the lifecycle actions (§2). |
| CHAT_THREADS_SCREEN | features/inbox/screens/ChatThreadsScreen.tsx | Root | Inbox room with several threads | Lists a brand room's threads. |
| EDIT_OFFER_SCREEN | features/chat/screens/EditOffer/index.tsx | Root | Offer card "edit" | Edit or counter an offer: price (suggested/minimum), content, publish time. Submits PUT `v3/mobile/offers/{id}`. |
| EXTEND_DEADLINE | features/chat/screens/ExtendDeadline/index.tsx | Root | Offer card, manual post step bubble | Ask for a later publish deadline. Submits PUT `deals/{jobId}/deadline/extend`. |
| PREVIEW_CONTENT | commonComponents/PreviewContent/index.tsx | Root | Chat options, transaction details | Preview the offer's content/post. |

### Profile, settings and wallet
| Screen | Component | Navigator | Reached from | Purpose |
|---|---|---|---|---|
| SETTINGS_SCREEEN (sic) | features/profile/screens/SettingsScreen.tsx | Root | Profile gear | Menu: Shipping address, Change password, Notifications & emails, Get help (support chat), About, Logout, Delete account. |
| CHANGE_PASSWORD | features/profile/screens/ChangePasswordSCreen.tsx | Root | Settings | Change password. |
| NOTIFICATIONS_EMAILS | features/profile/screens/NotificationsEmails.tsx | Root | Settings | Notification and email category toggles (v2 notify-settings). |
| SETTINGS_SHIPPING_ADDRESS | features/profile/screens/SettingsShippingAddressScreen.tsx | Root | Settings, Profile shipping card | Saved shipping addresses. |
| SETTINGS_SHIPPING_ADDRESS_FORM | features/profile/screens/SettingsShippingAddressFormScreen.tsx | Root | Address list "add/edit" | Add or edit an address, with Mapbox autocomplete. |
| ABOUTUS_SCREEN | features/profile/components/AboutUs/index.tsx | Root | Settings | Static about page. |
| EDIT_ACCOUNT_SCREEN | features/profile/screens/EditAccountScreen.tsx | Root | Profile user-info header | Edit the user (owner) profile. Submits PUT `v3/mobile/user/profile`. |
| EDIT_NAME | features/profile/screens/EditName.tsx | Root | Account card options | Edit a social account's display name and description. |
| EDIT_CATEGORY | features/profile/screens/EditCategory.tsx | Root | Account card options | Edit a social account's categories (saved through the onboarding endpoint). |
| NOTIFICATION_SCREEN | features/notification/NotificationScreen.tsx | Root | Header bell (tab screens, wallet) | In-app notification list. Marks items read and taps navigate (§3). |
| WALLET_SCREEN | features/new-wallet/screens/WalletScreen.tsx | Root | Profile earnings, Home earnings, chat offer card, `cost-transferred` push, DL `wallet` | **New wallet (finance service):** balance, payout method, withdraw, payments, payouts (§2). |
| WALLET_SCREEN_OLD | features/wallet/screens/WalletScreen.tsx | Root | New wallet "Transactions Before 9th August 2025" | Legacy v3 wallet: **history only** (Payments and Payouts tabs). Its Withdraw header and payout buttons are commented out. |
| TRANSACTION_DETAILS | features/wallet/screens/PaymentsDetails | Root | Legacy wallet Payments/Payouts rows | Legacy transaction detail. It offers Open offer (goes to chat) and Post review (goes to preview). |
| SUPPORT_CHAT_SCREEN | assets/WASupport/index.tsx (`SupportChatScreen`) | Root (slide from bottom) | Support FAB on tabs, Settings "Get help", wallet contact support | Chatgram support widget in a WebView. |

### Onboarding (inside ONBOARDING_FLOW)
| Screen | Component | Reached from | Purpose |
|---|---|---|---|
| WELCOME_SCREEN | onboarding/screens/wellcome/WelcomeScreen.tsx | Explore auto-redirect (no accounts), new Google sign-up, Home setup tasks / add-account card, campaign details (no or draft account), Explore select-account | "Get started" → GET_USER_NAME. |
| GET_USER_NAME_SCREEN | onboarding/screens/getUserName/GetUserNameOnboarding.tsx | Welcome; Profile "add account"; Explore "add social account"; YouTube popup; TikTok resubmit | Pick a platform: Instagram, TikTok or YouTube. Instagram asks for a username (lookup); TikTok and YouTube use OAuth. Renders ConfirmAccountScreen inline for TikTok resubmit. |
| VERIFY_INSTAGRAM_SCREEN | onboarding/screens/getUserName/VerifyInstagramScreen.tsx | GetUserName (IG), Profile resubmit (IG) | Instagram DM-code verification and manual review. |
| MEDIA_KIT → NICHE → YOUR_AUDIENCE → PHOTO_GALLERY → SHIPPING_ADDRESS_SCREEN → GOALS → CONGRATULATIONS | onboarding/screens/{mediaKit/MediaKitScreen, niche/NicheScreen, Audience/AudienceScreen, showcase/ShowcaseScreen, ShippingAddress/ShippingScreen, Goals, Congratulation} | After account connect (resume step), Profile "Edit profile" (→ MEDIA_KIT), remaining-steps overlay | The 6-step media-kit onboarding (§2). |
| VERIFY_TIKTOK_SCREEN → TIKTOK_VERIFIED | onboarding/screens/getUserName/VerifyTikTokScreen.tsx, onboarding/screens/TikTokVerified | Congratulations (TikTok accounts only) | "DM this code" instructions, then a "you're verified" screen → Explore. No API call in between. |

**Count:** 90 unique registered screen names, not counting the 3 container navigators.
- **64 are live.**
- **26 have no user-reachable inbound navigation** (§5):
  - 6 orphan routes;
  - 15 legacy onboarding screens/modals;
  - the stub CREATE_OFFER;
  - 4 legacy wallet sub-screens.

Breakdown by navigator:
- Root stack: 47 entries
- Onboarding: 27
- Explore top tabs: 8
- Auth: 7
- Bottom tabs: 5

## 2. Step sequences as the UI implements them

### 2.1 Sign-up, login and password reset
- **Email sign-up:**
  1. SIGN_UP: first name (required), last name, email, password. Calls `POST v3/mobile/register`.
  2. CONFIRM_EMAIL: enter the code (`POST v3/mobile/verify-email`; "Resend" calls `resend-email-verification`; "change email" calls `POST v3/mobile/change-email`).
  3. The app resets to MAIN, which opens on Explore. Explore's account selector auto-opens onboarding (Welcome) about 3 s later if the user has no social account.
- **Google:** native Google Sign-In → `POST v3/mobile/auth/social/google`.
  - New user (201 / `is_new`): stack becomes MAIN with ONBOARDING_FLOW/Welcome on top.
  - Existing user: MAIN.
- **Login:** `POST v3/mobile/login`. A verified email goes to MAIN; an unverified one goes to CONFIRM_EMAIL.
- **Forgot password:**
  1. FORGET_PASSWORD: enter email → `forgot-password/forgot`.
  2. CONFIRM_EMAIL in reset mode → `forgot-password/verify`. This returns a reset token.
  3. RESET_PASSWORD → `forgot-password/reset` with that token as Bearer.
  4. Back to LOGIN. A 400 means "must differ from previous password".
- **Session handling:**
  - A 401 triggers a token refresh (`v3/mobile/refresh-token`). If the refresh fails, the user is reset to Login.
  - Logout is **local only**: it clears state and deletes the FCM token. There is no backend logout call.
  - Delete account calls `DELETE v3/mobile/user/profile/remove`.
- **Splash:** GET `v3/mobile/versions`. Any newer store version is treated as a **mandatory** update with no Skip. Then a CodePush check, then MAIN or AUTH.

### 2.2 Connecting a social account (GET_USER_NAME_SCREEN, "1 of 3")
The platform choice is Instagram (platform id 1), TikTok (2) or YouTube (4). Facebook and X can't be connected; they appear only as share targets.

**Instagram (DM verification)**
1. "2 of 3": enter the username → `POST v3/mobile/influencer/on-boarding/profile/lookup`. Possible errors:
   - user not found;
   - profile fetch error;
   - 422;
   - "already added" (checked on the device).
2. "3 of 3", VERIFY_INSTAGRAM_SCREEN: the app shows code `ain-<token>`.
   - "Open Instagram & Follow + DM" copies the code and opens a DM to **@ainfluencer.official**.
   - "Confirm you did it" waits 6 s, then calls `GET v3/mobile/influencer/integration-verifications/status?username=`.
3. Outcomes:
   - Verified, with an integration → onboarding at the resume step.
   - Under manual review or temporary → "Submitted" overlay: "reviewed by admins, up to 24 hours, notified by email". OK goes to PROFILE.
   - Failed → alert. On the **2nd failure** the app auto-submits `POST …/integration-verifications/manual` and shows "Submitted".
   - Not qualified (`IgNotQualifiedError`) → account-requirements message.
   - A 422 from the verification endpoint → error overlay → Profile.

**TikTok (native OAuth)**
1. TikTok SDK login with scopes user.info.basic, profile and stats. The redirect is `https://app.ainfluencer.com/influencer`. The code plus PKCE verifier go to `POST v3/mobile/influencer/on-boarding/profile/verification` with `platform_id: 2`. The call is never retried.
2. Outcomes:
   - New account → onboarding resume step.
   - Already onboarded → Profile.
   - Already linked → stays on platform select.
   - Timeout → alert.
3. Android cold-start recovery runs in `useTikTokColdStartResume`, mounted in the navigation Container.
4. After Congratulations, TikTok accounts see VERIFY_TIKTOK_SCREEN: "follow and DM `ain-<integration_id>` to **@ainfluencer2024**". "Verify & go live" opens TIKTOK_VERIFIED ("You're verified, brands can book you") **with no API check**, then Explore.

**YouTube (Google consent in a browser)**
1. `GET v3/mobile/influencer/youtube/consent-screen` returns a Google OAuth URL as text.
2. The app opens it in Custom Tabs or SafariViewController with `prompt=select_account consent`.
3. The app catches the redirect (backend `…/influencer/youtube/callback` or `ainfluencer://youtube/oauth`) and sends the code to the same `…/profile/verification` endpoint (`auth_source: consent_screen`).
4. Outcomes:
   - Several channels → channel picker.
   - Temporary or no channel → Profile.
   - Already linked or onboarded → Profile.
   - Success → onboarding resume.
   - 400 → "Try again".
   - 500 → "Something went wrong".
   - User cancels → nothing happens.

**Resubmitting a rejected account**
- The Profile account card shows the rejection reasons and a "Resubmit" button, followed by a confirmation overlay.
- **Instagram:** VERIFY_INSTAGRAM_SCREEN (`isResubmit`) → `POST v3/mobile/temporary-integrations/{id}/resubmit` → "Submitted".
- **TikTok:** the old username + bio-code overlay (ConfirmAccountScreen, rendered inline).

**Suspended accounts:** an appeal modal on Profile and Explore. It loads `GET {api}/v2/settings/influencer/suspension/appeal` and `…/reasons`, then submits `POST {api}/v2/influencer/suspension-appeal`.

**Other account actions** (Profile account-card options):
- Sync: `POST …/accounts/{id}/sync`.
- Edit name/description: `PUT …/accounts/{id}`.
- Edit categories: onboarding endpoint.
- Change photo: `files/upload` + `PUT …/accounts/{id}` `{picture_id}`.
- Remove photo: `DELETE …/accounts/{id}/picture`.
- Public profile link: `https://app.ainfluencer.com/public/profile/{id}`.

### 2.3 Onboarding (the media-kit flow; the live order)
- After connecting, the app loads `GET v3/mobile/influencer/accounts/{integration_id}`.
  - If the account is already onboarded, the user goes to Profile.
  - Otherwise the app jumps to the first incomplete step (`mapNewFlowResumeRoute`), with the earlier steps stacked underneath so Back works.
- Each step saves with `POST v3/mobile/influencer/on-boarding {request:<slug>, …, integration_id}` and logs a funnel event with `POST v3/mobile/funnels` (process `influencer-onboarding`).

| # | Screen | User enters | Saved as |
|---|---|---|---|
| 1 | MEDIA_KIT | Bio (20–700 chars; contact details blocked by regex), language, gender, date of birth (13+), optional profile photo. Bio data is prefilled from `GET v3/mobile/platforms/{platform}/accounts/data`. | biography, language, gender, date-of-birth (+ photo through files/upload) |
| 2 | NICHE | 1–10 categories (`GET v3/mobile/categories`) and at least 1 hashtag (`GET v3/mobile/social/hashtags/search`) | categories, hashtags |
| 3 | YOUR_AUDIENCE | Up to 3 audience locations, post topics (20+ chars), audience description (20+ chars), and at least one **insights screenshot (required)** | locations, post-topics, audience, insights |
| 4 | PHOTO_GALLERY (Showcase) | Up to 10 gallery photos, plus optional past collaborations (Instagram post URL lookup via `GET v3/mobile/social/medias/feed`, or a manual upload). "Finish later" skips. | photo-gallery, past-collaborations |
| 5 | SHIPPING_ADDRESS_SCREEN | Label, zip, phone, full name, address, country (`GET v3/locations/countries`), city, map pin (Mapbox). "Add later" skips. | address |
| 6 | GOALS | At least one goal (`GET v3/mobile/goal/questions`) | goals, then `POST v3/mobile/influencer/on-boarding/finished` (fire-and-forget) |
| → | CONGRATULATIONS | "Your media kit is live": copy link / share. Instagram and YouTube → Explore; TikTok → VERIFY_TIKTOK_SCREEN → TIKTOK_VERIFIED → Explore. | – |

- Rates and Supercharge steps are disabled ("until backend is ready").
- The legacy 15-step chain (MORE_ABOUT_YOU → … → GOALS) is not reachable.
- **Re-entry:**
  - Profile "Edit profile" (Pending or Active account) → MEDIA_KIT, prefilled.
  - The "remaining steps" overlay for draft accounts (Explore, campaign details, Make Offer) → the resume step.

### 2.4 Making an offer on a paid campaign
1. **CAMPAIGN_DETAILS CTA** ("Make offer" / "Accept invitation" / "View offer"). The checks run in this order:
   - maintenance → alert;
   - suspended account (temporary or permanent) → "account suspended" message;
   - archived account → "account archived" message;
   - no allowed social account → ONBOARDING_FLOW;
   - otherwise → MAKE_OFFER_SCREEN.

   When the user arrived from an invitation deep link, an "Accept Invitation for Which Account?" picker comes first. The offer is then sent with `bypass_influencer_qualification: true`. CREATE_OFFER_SCREEN is a stub and is never reached.
2. **MAKE_OFFER_SCREEN sections**, top to bottom:

   | # | Section | Contents |
   |---|---|---|
   | 1 | Account | Shown only when the user has more than one account. Display-only: the account is switched from Explore's account selector. |
   | 2 | Link in bio: Yes | Shown only when the campaign requires a bio link. |
   | 3 | Deal terms | Delivery date/time (default now + 2 weeks; a "Short deadline" warning appears under 14 days).<br>Duration: "I will delete the content after: N hours/days/weeks/months". Unchecked means Permanent; the minimum is 4 hours.<br>Proposed price: minimum $10; suggested price from `GET v3/mobile/influencer/accounts/{id}/pricing?duration=`. The help text says Ainfluencer keeps 20%. |
   | 4 | Your pitch | 50–250 chars; contact details are blocked. |
   | 5 | About you | Topics (50–250), Audience (50–250), collaboration types (required, from `GET v3/mobile/settings/collaboration-types`). |
   | 6 | Portfolio | Insight/analytics screenshots and past collaborations (uploaded with `POST v3/mobile/files/upload`). |

   Prefill comes from `GET v3/mobile/influencer/accounts/{id}/settings` and `GET v3/mobile/influencer/accounts/{id}`.
3. **Submitting:**
   1. "Save and Continue" validates the form. A draft account sees the "Complete profile" overlay instead (it goes to the onboarding resume step).
   2. "Review your offer" sheet → **Send Offer**.
   3. The offer goes to `POST v3/mobile/offers`, or to `POST v3/mobile/offers/temporary` when the account's quality control is still pending. The payload has start, duration (0 = permanent), price, insights, past collaborations, collaboration types and `exclusivity: 0`.
4. **After sending:** Explore opens with an "Offer sent" sheet. "Open offer chat" goes to CHAT_SCREEN. The app may also show the native in-app review prompt.

### 2.5 Offer lifecycle in chat (CHAT_SCREEN offer card)
| State key | Label shown | Influencer actions |
|---|---|---|
| `waiting_your_approval` | Waiting Your Approval (the brand countered) | Accept / Edit / Reject. Accept and reject call `PUT v3/mobile/offers/{id}/history {action}`; Edit opens EDIT_OFFER. |
| `waiting_payment` | Waiting Brand Approval | Edit Offer, See details |
| `scheduled` | Paid & Scheduled | Countdown, **Submit Post URL**, **Extend Deadline**, Campaign delivery terms. When the brand asks for a revision: Accept / Reject (`POST v3/mobile/offer/{id}/revise/accept\|reject`). |
| `published_waiting_approval` | Published & Waiting Approval | Info only |
| `published_approved` | Published & Approved | Shows the paid amount (price × 0.8). "See payout" opens the new WALLET_SCREEN on the Payments tab. |
| `rejected_by_brand` / `rejected_by_you` / `rejected` | Rejected… | See reason (brand), Edit Offer |
| `rejected_by_admin` | Rejected by Admin | None |
| `failed` | Failed | If paid (in escrow): Extend Deadline + Submit Post URL. Otherwise "failed to deliver" and Edit. |

- **Edit offer:** content, link in bio, delivery date, duration, price → `PUT v3/mobile/offers/{id}`. The minimum price comes from `GET v3/mobile/settings/offers/pricing`.
- **Extend deadline:** `PUT v3/mobile/deals/{jobId}/deadline/extend`.
- **Submitting the post:**
  - Post URL → `POST v3/mobile/deals/{jobId}/url`.
  - Uploaded content → `POST/DELETE v3/mobile/deals/{jobId}/published-contents/files`.
  - The user can edit or delete afterwards, but at least one item must stay.
- **Chat menu:**
  - Offer progress, Shipping details, Submit post URL
  - View brand profile, View campaign
  - Download, preview or copy content
  - Archive/Unarchive (`v3/mobile/chats/{id}/archive|unarchive`)
  - Report campaign
- **Message input:**
  - Text, sent over the socket.
  - Message templates (`v3/mobile/message-templates` CRUD in an overlay).
  - Attachments: media through files/upload, or insights via `POST v3/mobile/offers/{id}/insights`. These answer the brand's "insight request" bubble.
  - Shipping-address pin, shown when paid, for gifting requests and for affiliate chats. It manages `v3/mobile/addresses` and sends the address to `offers/{id}/send-address`, `product-seedings/requests/{id}/send-address`, `product-seedings/{id}/send-address` or `chats/affiliate/{adId}/send-address`.
  - Reactions are sent over the socket (`react-message`).
  - The input is hidden when the brand or the influencer is suspended.
- **Inbox list actions:** pin/unpin a thread or room (rooms have a pinned maximum) and archive. The unread badge comes from `v3/mobile/chats`.
- **Not in the app:** there are **no cancel or refund request actions** in the mobile UI.

### 2.6 Gifting (product seeding)
- **Entry points:**
  - Explore "gifting" tab: pipeline boxes Needs You / Requested / Accepted / In Transit / Received / Published & Completed, plus filters.
  - AllAds gifting section → GIFTING_CAMPAIGNS.
  - Invitations, brand profile, chat.
  - Deep link `gifting/invitation`.
- **GIFTING_DETAIL single CTA by state:**

  | CTA | When | Steps and calls |
  |---|---|---|
  | **Send Gift Request** | — | Pick or create an address, optional note (255 chars) → `POST v3/mobile/product-seedings/{id}/apply`. The button is disabled with "This brand does not ship to X" when the address country isn't shipped to. |
  | **Accept Invite** | Brand invitation, `pending_accept` | Address + note → `PUT …/requests/{id}/respond {action:'accept'}`. A 400 "no brand invitation" falls back to `apply`. **Reject** → `respond {action:'reject'}`. |
  | **Confirm Received** | Status `shipped` | `PUT …/requests/{id}/status {status:'received', note}`. This is the only status the influencer can set. |
  | **Tell the brand I posted** | — | Tick every post condition, enter URLs → `POST …/requests/{id}/posts`. Edit a post with `PUT …/posts/{postId}` (**no backend route, see §6**); delete with `DELETE …/posts/{postId}`, keeping at least one. |

- **Address change:** allowed only while the request is requested or accepted → `PUT …/requests/{id} {address_id}`.
- **Message brand:** opens the existing thread (`GET v3/mobile/chats/{id}`) or starts one (`POST v3/mobile/chats/messages`, `subject_type: product_seeding`).

### 2.7 Affiliate
- **AFFILIATE_DETAIL** shows the header, products with search (`GET v3/mobile/ads/affiliate/{id}/products`), an earnings slider, requirements and match info.
- **Generate links** → `POST v3/mobile/ads/affiliate/{id}/assign-influencer {custom_items:[{item_id, commission}]}`. Then a BrandID promo modal appears ("Build My Storefront" opens brandid.app).
- **Message brand:** same start-chat flow as gifting.
- **Link analytics** (AFFILIATE_ANALYTICS): 7d / 30d / 90d / all summary, timeline and products (`v3/mobile/ainfluencer-affiliate/reports/*`), plus payouts (`…/ainfluencer-affiliate/payouts`). The same summary feeds the Home "affiliate performance" widget.

### 2.8 Wallet and payouts (new wallet, finance service)
WALLET_SCREEN has three tabs: **Wallet / Payments / Payouts**.
- **Wallet tab:** balance ("Available to withdraw", "Pending requests") from `GET finance/current-user/wallets`, and payout methods from `GET finance/mobile/payout-methods`.
- **Payout methods you can add:**
  - **PayPal:** email → `POST finance/mobile/payout-methods {gateway: paypal, value}`.
  - **Crypto** ("USDT, USDC and more"): currency and network come from `GET finance/mobile/payout-methods/crypto-options`, then the user pastes a wallet address → `POST finance/mobile/payout-methods {gateway: crypto, network, value}`.
  - **Stripe:** code present but **disabled** (commented out).
- **Withdrawing:**
  1. Select a payout method.
  2. "Submit Withdrawal Request". It is disabled with no method or a zero balance.
  3. "Select Completed Deals": payable orders from `GET finance/mobile/orders/payable`, plus an optional "Legacy balance" row sent as `LegacyPayout`. The sheet shows the fee % and the net amount.
  4. "Request Withdrawal ($X)" → `POST finance/mobile/payouts {orderItemIds, walletSettingId}`.
  5. A "submitted" modal appears, then possibly the in-app review prompt.
- **Payments tab:** `GET finance/mobile/influencer-orders`. Tapping a row opens the offer chat.
- **Payouts tab:** `GET finance/mobile/payouts`, with statuses Pending / Processing / Completed / Rejected / Canceled.
- **Legacy wallet:** the "Transactions Before 9th August 2025" link opens WALLET_SCREEN_OLD. It is **history only** (`v3/mobile/wallet`, `…/transactions/payments|payouts`, TRANSACTION_DETAILS); the old withdraw and PayPal-connect screens can't be reached.
- **Support:** "Contact support" in the wallet opens the Chatgram support chat.

### 2.9 Other tabs
- **Home** widgets, in order:
  1. Greeting + earnings → WALLET
  2. Shipping-address card
  3. Setup tasks ("Connect your TikTok/Instagram/YouTube" → onboarding Welcome)
  4. Schedules → Scheduler / chat
  5. Offers (`v3/mobile/offers`) → Inbox
  6. Affiliate performance → AFFILIATE_ANALYTICS
  7. Invitations (`v3/mobile/ads/invited/summary`) → INVITED_CAMPAIGN_DETAILS
  8. Your Pulse (per-account offer stats) → Profile
  9. Add social → onboarding
  10. Discover more → Explore "for_you" tab
  11. Grow with family and Tips & guides (external brandid.app / YouTube links)
  12. Support FAB

  The BrandID banner is not rendered.
- **Scheduler:** Scheduled / Published / Failed tabs (`GET v3/mobile/deals/{scheduled|published|failed}`). The card menu has View on Instagram/TikTok, View offer (chat; blocked with an alert when suspended) and View campaign.
- **Brand profile:** tabs appear only when they have data. Invite-only or paused campaigns show a restriction popup. The menu has About and Block.
- **Profile:**
  - User header → EDIT_ACCOUNT (first/last name).
  - Settings, Wallet, shipping address.
  - Account cards (Active/Archived tabs) with these buttons: Edit Profile / Share (active), Complete profile (pending), Submit Appeal or appeal status (suspended), Resubmit (rejected).
  - The ⋯ menu has Sync (YouTube has a cooldown), Edit name, Edit category and Profile picture. It is disabled for archived or suspended accounts.
- **Settings:**
  - Shipping Address
  - Change Password (`POST v3/mobile/change-password`)
  - Notifications & Emails (`GET/PUT {api}/v2/notify-settings/categories`)
  - Get Help (Chatgram)
  - About
  - Logout (confirm; local only)
  - Delete Account (confirm → `DELETE v3/mobile/user/profile/remove` → logout)

## 3. Push notifications and deep links

### Push notification → screen
All push taps go through the same handler: `utils/notificationNavigation.ts → handleNotificationPress`. It is wired up in three places:
- `hooks/useNotification.ts`, mounted in RouteStack and **only while logged in**. It covers the FCM initial notification, `onNotificationOpenedApp`, and the Notifee foreground press.
- `lib/registerNotificationHandlers.ts` (imported by `index.js`). It covers the Notifee background press. The FCM background message is shown as a local Notifee notification.

The handler routes on `data.event` through `features/notification/utils/redirectTo.ts` and uses `data.offer_id` or `data.offerId`. On a cold start (Splash still on the stack) it rebuilds the stack with `resetFromColdStartNotification`. When a push arrives in the foreground, the app shows a local notification and runs `queryClient.invalidateQueries()` (refetches everything).

| `data.event` values | Target | Cold-start stack |
|---|---|---|
| `events:mobile:offer:message-received`, `chat:new-message`, `offer:message`, `events:mobile:publish-job:publish-reminder`, `…offer:revision:requested-by-brand`, `…offer:updated`, `…offer:rejected`, `…offer:paid`, `…offer:start-date-changed`, `…offer:updated-by-admin`, `…publish-job:publish-disapproved-by-admin`, `publish-job:approved-by-admin` (legacy name, see note), `…publish-job:publish-approved-by-brand` | `CHAT_SCREEN {offerId}`. If there is no `offer_id`, it falls through to a plain `navigate(CHAT_SCREEN)` without params. | Inbox tab + Chat |
| `events:mobile:publish-job:cost-transferred` | `WALLET_SCREEN` (new wallet) | Profile tab + Wallet |
| `events:mobile:publish-job:published` | `SCHEDULER_SCREEN {tab:'published'}` | Scheduler tab |
| `events:mobile:publish-job:failed` | `SCHEDULER_SCREEN {tab:'failed'}` | Scheduler tab |
| `events:mobile:invitation:internal:sent`, `…invitation:internal:created`, `…invitation:external:created` | `INVITED_CAMPAIGN_DETAILS` | Home tab + Invitations |
| `events:mobile:account-activated`, `…account:permanently-suspended`, `…account:temporarily-suspended`, `…account:temporary:rejected`, `…account:temporary:verified`, `…account:health-check:failed`, `…account:unarchived` | `PROFILE_SCREEN`, and the social-accounts list is force-refetched | Profile tab |
| `events:mobile:offer:review:submitted-by-brand`, `events:mobile:account:violation-detected`, and **every other or unknown event** | `EXPLORE_SCREEN` | Explore tab |

**Notes**
- `events:mobile:publish-job:publish-disapproved-by-admin` is in the "failed" list (`openFailedTabIfContains`), but `redirectTo` sends it to **Chat**, so the scheduler failed-tab rule never applies to it.
- These events have no case of their own, so they land on Explore:
  - `events:mobile:auth:email-verification-required`
  - `events:mobile:auth:verification:email-changed`
  - `events:mobile:publish-job:publish-approved-by-admin` (only the legacy `publish-job:approved-by-admin` string is mapped)
  - `…content-delivered`
  - `…content-failed-to-publish`
  - `…completion-failed`
  - all legacy `influencer.*`, `offer:*` and `brand.*` names
- **In-app notification list** (`NOTIFICATION_SCREEN`, `features/notification/components/RenderItem.tsx`) uses the same `redirectTo`, with two differences:
  - chat events with an `offer_id` open Chat;
  - scheduler events open the Scheduler **without** the tab param, because of a buggy `Paths.includes` check, so they always land on the default "Scheduled" tab.
- `lib/PushNotification.ts` (react-native-push-notification `onNotification`) does no navigation. It handles only the download/gallery actions.
- No push handling happens before login: `useNotification` is gated on the access token.

### Deep links
- **Prefixes:** `ainfluencer://`, `https://ainfluencer.app.link`, `https://app.ainfluencer.com` (and http). The linking config is in `navigation/Container.tsx`.

| Link | Goes to |
|---|---|
| `wallet` | `WALLET_SCREEN` |
| `invitations` | `INVITED_CAMPAIGN_DETAILS` |
| `chat/:offerId` | `CHAT_SCREEN` |
| `profile` | Profile tab |
| `scheduler/:tab` | Scheduler tab |
| `invitation?ad_id=` or `view-campaign/<id>` | `InvitationRouter` → GET `v3/mobile/ads/{id}` → `CAMPAIGN_DETAILS` (if `ad_id` is missing: `INVITED_CAMPAIGN_DETAILS`) |
| `affiliate/invitation?ad_id=` or `view-campaign-affiliate/<id>` | `AffiliateInvitationRouter` → GET affiliate ad → `AFFILIATE_DETAIL` (on error: Main) |
| `gifting/invitation?ad_id=&email=` | `GiftingInvitationRouter` → looks the seeding up in product-seedings list, requests and invitations → `GIFTING_DETAIL` (if not found: Main) |

- **Cold start:** the app builds the initial state `[MAIN_NAVIGATOR, router]`.
- **Logged out:** deep links can't resolve. The router screens are registered only when an access token exists, so a logged-out user just lands on Login.
- **Not deep links:** TikTok OAuth redirects (`app.ainfluencer.com/influencer…`) are filtered out of linking. The YouTube OAuth return (`ainfluencer://youtube/oauth`) is handled inside the consent browser session.

## 4. Socket (chat) events
- **Connection:** `lib/socket/ChatSocket.ts` uses socket.io, `transports:['websocket']`.
  - `auth`: `{ 'x-chat-auth': 'Bearer <chatbox_token | chat_token | chat_token_plus>', 'x-public-key': <hardcoded key> }`.
  - It starts in `TabNavigator` when a chat token exists (`CHAT_PROVIDER === 'socket'`, hardcoded in `config/chatProvider.ts`), and is re-initialised from ChatScreen and before sending if disconnected.

| Direction | Event | Payload / effect | File |
|---|---|---|---|
| listen | `connect`, `disconnect`, `connect_error` | debug logging; on `disconnect` the received-message de-dupe set is cleared | ChatSocket.ts, redux/chat/socketThunks.ts |
| listen | `new-message` | `msg.data` with `conversationId`. The message goes into redux `receiveMessage` (de-duped by msid); the thunk itself does no inbox/unread invalidation | socketThunks.ts |
| listen | `delivered-message` | marks the message delivered | socketThunks.ts |
| listen | `seen-message` | marks the message read | socketThunks.ts |
| listen | `unread_count_updated` | `{offer_id, unread_messages_count}`. A count of 0 clears offer unread; otherwise the inbox badge query is invalidated | socketThunks.ts |
| listen (debug only) | `onAny` | logs everything; `edit-message`, `delete-message`, `error-event` and `channel_updated` are declared in the type but **not handled** | ChatSocket.ts |
| emit | `new-message` | `{conversationId, body?, attachment?, attributes?}`, with ack. An ack timeout or empty ack falls back to waiting for the broadcast echo, then fails | socketThunks.ts `sendSocketConversationMessage` (called from `features/chat/hooks/useChat.tsx`) |
| emit | `seen-message` | `{messageId}`, with ack. Sent when messages are viewed | features/chat/screens/ChatScreen.tsx |
| emit | `react-message` | `{messageId, reaction, participant:'influencer'}` | ChatScreen.tsx (`onReact`) |
| emit (dead) | `send-message` | `{channel_id, body, attributes}`. `sendSocketMessage` is imported in useChat but never called | socketThunks.ts |
| emit (dead) | `edit-message` | commented out | ChatScreen.tsx |

- **REST message history:** `GET {chat}/message/conversation/{chatbox_conversation_id}?page=&pageSize=20&order=desc`, using the same `x-chat-auth` / `x-public-key` headers (ChatScreen).
- **Twilio:** `connectTwilio`, `chatThunks.ts` and `unreadMessageSlice` Twilio code are imported but only run when `CHAT_PROVIDER !== 'socket'`. They are dead in this build.
- **Support chat (separate system):** the floating chat FAB (`assets/WASupport`) on Home, Explore, Scheduler, Inbox and Profile, plus "Get help" in Settings and wallet "Contact support", opens `SUPPORT_CHAT_SCREEN`.
  - That screen is a WebView that loads the **Chatgram widget** (`cdn.jsdelivr.net/npm/chatgram-widget@1.1.0`) with base URL `https://app.ainfluencer.com` and `CHATGRAM_PUBLIC_KEY` from env.
  - If the key is missing, the app shows the alert "Support unavailable".

## 5. Dead code left out of the flow map

### Screens that are registered but have no inbound navigation
These routes are reachable by import but not by the user:

| Route | Component | Why it's dead |
|---|---|---|
| `NOT_VERIFY_SCREEN` (Auth) | features/auth/screens/notVerifyEmail/NotVerifyEmailScreeen.tsx | Nothing navigates to it. Unverified logins go to `CONFIRM_EMAIL_SCREEN`. |
| `ADD_SOCIAL_SCREEN` | features/addSocial/AddSocialScreen.tsx | No `navigate`. Social accounts are added through `ONBOARDING_FLOW` instead. |
| `CREATE_MSG_TEMPLATE` | features/chat/screens/MessageTemplates/CreateEditTemplate/index.tsx | No `navigate`. Templates are managed in `features/chat/modals/MsgTemplatesOverlay` via `useMsgTemplates`. Its PATCH also uses bare `axios` with no baseURL. |
| `EDI_CONTENT_SCREEN` | features/chat/screens/EditContent | No `navigate`. |
| `USER_TIKTOK_CONTENT_SHARE` | features/onboarding/screens/UserTiktokContentShare.tsx | No `navigate`. It is a dev test screen: TikTok proxy, hardcoded username, ngrok header. |
| `BRAND_ID_LANDING` | features/home/screens/BrandIdLandingScreen.tsx | Its only caller, `features/home/components/BrandIdBanner.tsx`, is never imported. |
| Legacy onboarding: `MORE_ABOUT_YOU`, `DATE_OF_BIRTH`, `GENDER`, `LOCATIONS`, `LANGUAGE`, `AUDIENCE`, `POST_TOPICS`, `CATEGORIES`, `HASHTAGS`, `INSIGHTS`, `AUDIENCE_INSIGHTS`, `PAST_COLLABORATIONS`, plus the modals `ONBOARDING_CUSTOM_MODAL`, `HASH_TAG_MODAL`, `IMAGE_CONTAINER_MODAL` | features/onboarding/screens/{moreAboutYou,dateOfBirth,Gender,Location,Language,Audience/index,Topics,Categories,Hashtags,Insights,AudienceInsights,PostCollab}, modals/OnboardingCustomModal, Hashtags/HashTagCustomModal, PostCollab/components/CustomModal | Nothing enters the old `screenProperty` chain. `MORE_ABOUT_YOU` has no inbound links, and `CATEGORIES` is reached only from the unregistered ProfilePhotosOnboarding. Resume (`mapNewFlowResumeRoute` / `mapLegacyOnboardingResumeRoute`) only returns consolidated-flow steps. Their API calls are counted only through live screens that share the same hooks. |
| `CREATE_OFFER_SCREEN` | features/ad/screens/createOffer/CreateOfferScreen.tsx | A 10-line stub that renders the text "Screen". Campaign details navigates there only when the route param `isExternalLink` is set, and nothing ever sets it. |
| `WITHDRAW_SCREEN`, `PAYOUT_SETUP`, `PAYOUT_METHOD_LIST`, `CHECKOUT_LIST` | features/wallet/screens/{Withdrawal,PayoutSetup,PayoutMethodList,CheckoutsList} | Their only entry points are commented out: the legacy `WalletHeader` with the Withdraw button (wallet/screens/WalletScreen.tsx) and the Add-payout / Payout-methods buttons in the legacy Payouts tab. Their calls (`v3/mobile/wallet/checkout-requests`, `…/payout-methods/*`, `…/paypal/connect`) are therefore **not live**. |
| `RATES`, `SUPERCHARGE` | onboarding/screens/rates, superCharge | Commented out of `OnBoardingScreens` ("until backend is ready"). The files can't be reached at all. |
| `CONFIRM_ACCOUNT_SCREEN`, `INITIAL_STACK` | — | Route names exist but no screen is registered. `ConfirmAccountScreen.tsx` is rendered *inline* by GetUserNameOnboarding: TikTok resubmit or the legacy bio-code overlay. |

### Whole files and folders that aren't reachable
About 275 files (list in `(generator script) → unreach`). The ones that matter:
- `features/userProfile/screens/InfProfileScreen.tsx`: the whole `userProfile` feature is dead.
- Old wallet: only its history (the Payments and Payouts tabs, plus `TRANSACTION_DETAILS`) is still reachable, through `WALLET_SCREEN_OLD`. The withdraw and payout-setup screens can't be opened (see the table above).
- The new-wallet `StripeCountryModal.tsx` and Stripe UI are commented out. `useCreateStripeAccount` is instantiated but never fired.
- `commonComponents/RateReviewPopup` (POST `/v2/mobile-app/reviews`) is unused. In-app review is native (`utils/inAppReview`) and fires after an offer, on Explore and in the wallet.
- `utils/photoUploadHooks.ts`, `utils/checkUserRemainingSteps.ts`.
- `features/chat/hooks/getOfferHistory.tsx` and `useGetOfferHistory.tsx` (the latter posts to the literal `:offerId`).
- `features/make-offer/actions/useGetAdDetail.ts` (`AD_APIS.SHOW_AD`), `features/onboarding/hooks/useGetFunnelProccess.tsx`.
- Old message-template UIs: `features/chat/components/InputMessage/MessageTemplates/**` and `features/chat/screens/MessageTemplates/MsgTemplatesOverlay`, which use undefined `API_ADDRESS` with v2 paths.
- `commonComponents/View/**`: Conversation, PublishJob, OfferObjectiveForm (v2 goal-questions), Timing, Interval and others, about 60 files.
- `features/onboarding/Components/{ConfirmAccount,BottomSheet,GetUserNameForm,HeaderInfo,LogoImg,ManualVerification/Rejected,ManualVerification/Resubmit,PlatformTipBanner}`, `onboarding/screens/rates/**`, `superCharge/**`.
- `features/brand-profile/screens/{Campaigns,OngoingOffers}.tsx`, `features/inbox/components/{FilterModal,HeaderRight,RenderItem}.tsx` (the old offers-based inbox), and `features/home/components/{BrandIdBanner,add_social/AddOneMore,schedules/ScheduleItem,...}`.
- Mock and test data: `scheduler/actions/mock.ts`, `home/components/schedules/mockData.ts`, `inbox/data/mock.ts`, `new-wallet/PayoutsList/mockData.ts`, `__tests__`, `sentry.test.ts`.

### Code that's reachable but never runs
- **Twilio chat path:** `redux/chat/chatSlice.tsx` `connectTwilio`/`refreshTokenChat`, `redux/chat/chatThunks.ts`, and the Twilio parts of `unreadMessageSlice.tsx`. Not executed while `CHAT_PROVIDER='socket'`.
- **Unused API helpers that look live:**
  - `useGetInboxOffers` (`INBOX_APIS.OFFERS`): imported only for types. The Inbox tab uses `v3/mobile/chats`.
  - `useReactToMessageMutation`: imported, never called.
  - `sendSocketMessage` ("send-message"): imported, never called.
  - `useCitiesByCountries` (`LOCATIONS_APIS.GET_CITIES`): never used.
  - `PostCollab/controller.fetchPostByUrl` (v2/aigrow): never used.
  - `AD_APIS.INVITED_TO`: only a fallback that never fires.
- **endpoints.js constants that are never referenced:**
  - `COMMON_APP.LOG_OUT`: logout is client-only.
  - `ADS_ENDPOINTS.UN_BLOCK_BRAND`
  - `ONBOARDING_APIS.GET_USER_SOCIAL_INFO`
  - `NOTIFICATIONs_API.READ_ALL`
  - `FORGET_PASSWORD_ENDPOINTS.RESET_PASSWORD`: only in the public whitelist.
  - `CHAT_APIS.UPDATE_INSIGHT`: dead file.
- **Splash `handleLoginStatus`:** reachable only from `skipUpdate`, but the Skip button never shows because every store update is treated as mandatory.
- **`MainNavigator.LanguageScreen`:** a language switcher that is defined but commented out of the render. The app has en/es translations, but no live UI to switch language.

## 6. Calls with no obvious backend counterpart, or that look suspicious

I checked every live call against `the core route dump` (the core Laravel route dump) using `the route matcher`.

| Call | Issue |
|---|---|
| **All new-wallet `finance/*` calls**: `GET finance/current-user/wallets`, `GET/POST finance/mobile/payout-methods`, `GET …/crypto-options`, `GET finance/mobile/orders/payable`, `GET/POST finance/mobile/payouts`, `GET finance/mobile/influencer-orders` | The app calls these through `axiosInstance`, so the real URL is `https://<host>/api/en/finance/...`. The backend registers them as `api/finance/...`, with no locale; the core v3 routes are literally `api/en/v3/...`. This works only if something strips `/en` (a gateway rewrite or locale middleware). **Needs confirming.** If wallet complaints show 404s, look here first. Notify-settings and appeal calls strip `/en` explicitly; the finance calls don't. |
| `PUT v3/mobile/product-seedings/requests/{requestId}/posts/{postId}` (GIFTING_APIS.UPDATE_POST, "edit shared post URL") | The backend has only `DELETE` on this path, so editing a gifting post URL probably fails with 405. |
| `POST v3/mobile/product-seedings/{id}/send-address` (GIFTING_APIS.SEND_ADDRESS_CAMPAIGN) | No matching route. Only `…/requests/{requestId}/send-address`, `offers/{id}/send-address` and `chats/affiliate/{adId}/send-address` exist. The app uses it when it knows only the campaign id, not the request id. |
| `HEAD v3/mobile/health` (utils/networkStatus.ts) | No route. The probe treats any HTTP response as "reachable", so a 404 is harmless. |
| Chat REST `GET {chat}/message/conversation/{id}` and all socket events | These live in the chatbox services (`core/backend/chatbox`), not core. They weren't checked against core routes. |
| `POST v3/mobile/offer/{offerId}/revise/accept\|reject` | Singular `offer`, unlike every other offer path. The route exists in core, so this is just a naming oddity. |
| `PUT v3/mobile/offers/{id}/history` (accept/decline-style actions from OfferCard) | Uses `history` as an action endpoint. The route exists. |
| `v3/mobile/refresh-token` | Hardcoded `version: '2.9.1'` in the 401-refresh body (axiosInstance). syncFirebaseToken sends the real version. The same endpoint is also used purely to sync the FCM token. |
| `GET v3/mobile/campaigns/statistics`, `POST …/forgot-password/reset` | Use bare `axios` with `${BASE_URL}`, so they skip the interceptors: no 401 refresh and no retry. The reset call carries its own reset token. |
| `INBOX_APIS.OFFERS`, `GIFTING`/`BRAND_PROFILE` lists, `ORDER_PAYABLE` | Built with `new URL(relativePath)`. React Native's URL shim appends a trailing `/` before the query (`…/offers/with-brand/?…`). This depends on the backend tolerating the trailing slash. |
| `CHAT_APIS` duplicate keys | `GET_TEMPLATES`, `ADD_TEMPLATES` and `DELETE_MESSAGE_TEMP` are defined twice (v2, then v3). The v3 values win. |
| `GET v3/mobile/ads/my-categories` from TikTokVerified | Used only to count matching campaigns (limit 1) for the "brands can book you" screen. |
| TikTok "verify & go live" (`VERIFY_TIKTOK_SCREEN` → `TIKTOK_VERIFIED`) | **No API call.** The screen tells the user to DM a code to `@ainfluencer2024`, and the "verified" screen appears regardless. If an influencer says the app showed "verified" but brands can't book them, this is why. |
| Instagram verification | DM-only: the user DMs `ain-<token>` to `@ainfluencer.official`. On the 2nd failed status check, the app automatically submits manual review (`VERIFY_DM_MANUAL`). |
| Mapbox (`api.mapbox.com` geocode v6 forward/reverse, static map images) | Third-party, called directly from the app with a token. Used for address autocomplete in chat, gifting, settings and onboarding shipping. |
| Dev and staging leftovers | `config/tiktok.ts` points at `tiktok-api-staging` (only the dead UserTiktokContentShare uses it). The `development` env uses **prod** chat (`ENV!=='staging'`). |
| `POST v3/mobile/privacy/brand/block` on campaign details | The call isn't awaited, so `resBlock.data` is always undefined. The request goes out, but the screen doesn't close and the "brand blocked" toast never shows. Brand profile uses `/v3/mobile/privacy/brand/block` (BLOCK_OR_UNBLOCK_BRAND). Nothing in the app calls unblock. |
| `POST v3/mobile/privacy/reports` | The "reported" toast is in a `finally` block, so it shows even when the request fails. |
| Scheduler suspension alert | The "Contact support" button has `onPress: () => {}`, so it does nothing. |
