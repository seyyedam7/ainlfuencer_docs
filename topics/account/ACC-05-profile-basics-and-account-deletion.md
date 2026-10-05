---
id: ACC-05
title: Profile basics and account deletion
domain: Account and sign-in
audience: [brand, influencer, admin]
apps: [web, mobile]
related: [ACC-02, ACC-04, BRD-02, BRD-03, SUB-02, PAY-01, PAY-04, INF-05]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d
---

# ACC-05 · Profile basics and account deletion

> The person's own details (first and last name) and deleting the account. Deleting is permanent for the user: the same email can't log in or sign up again. Only Ainfluencer staff can restore a deleted account.

## Who and where

- **Brands, web** `/profile` (header avatar → Profile). The top section has "First Name", "Last Name", "Company Name", "Email Address" (greyed out), "Location" and "Language"; the rest of the page is the company profile (BRD-02). At the bottom: "Delete Account" ("Permanently removes your brand account and data. This cannot be undone."). Changes are saved with the save bar ("Unsaved changes", "Discard").
- **Influencers, mobile**: Profile tab → tap the name header → "Edit Account Information" ("FIRST NAME", "LAST NAME", "EMAIL", "Save"). Deletion: Profile → gear → "Account Settings" → "Delete Account".
- **Team members** working inside another brand's team can't open `/profile` (they're sent to the dashboard). In their own brand they can.

## How it works

**Editing the name**
- Web: change "First Name" / "Last Name" and save. Mobile: change the fields and tap "Save" ("Your profile updated successfully.").
- The email can't be changed here (ACC-04). Brands can't edit their phone number after sign-up.

**Deleting a brand account (web)**
1. On `/profile`, click "Delete Account".
2. A pop-up: "Are you sure?" / "Your account is about to be deleted permanently. Once deleted, all your data will be removed, and you won't be able to sign up with this account again. This action cannot be undone." → "Delete My Account" or "Cancel".
3. After confirming, the brand is logged out. In the background, Ainfluencer:
   - marks the account deleted and unsubscribes it from emails;
   - **cancels all active subscriptions immediately** (SUB-02);
   - takes the brand's live campaigns off the marketplace (marked rejected);
   - expires the brand's pending campaign invitations to influencers;
   - logs out every team member of the brand's team and expires pending team invitations (BRD-03);
   - ends all of the brand's own sessions.

**Deleting an influencer account (mobile)**
1. "Delete Account" → "Are you sure you want to delete your account?" → "Yes, Delete".
2. The account is marked deleted, unsubscribed from emails, all sessions end, and the app logs out.

## Rules and limits

- After deletion, login and sign-up with that email both show: "This account has been permanently deleted and can no longer be used to log in or sign up."
- **Money is not paid out or refunded automatically on deletion.** A brand's wallet balance stays in the deleted account, and influencer withdrawals don't happen by themselves. Users should withdraw or spend their balance first (PAY-01, PAY-04).
- Deleting a brand doesn't refund the current subscription period *(to verify with finance: Stripe "cancel now" without proration)*.
- Influencer deletion doesn't remove their social accounts, offers or deals; those records stay *(to verify whether the accounts still appear to brands)*.
- Names can't be empty on the web form. Links or email addresses aren't accepted in the company name.
- **Restoring:** staff can restore a deleted account from the admin panel. Restoring re-enables the account, re-subscribes it to emails, marks the email verified, restores its social accounts, and sends campaigns that were taken down at deletion **back to admin review** (not straight to live). Cancelled subscriptions are **not** restored.

## Statuses

| What support sees | Meaning | Next |
|---|---|---|
| `users.archived_at` set | Deleted by the user (or archived by staff) | Can't log in or sign up; staff can restore |
| `users.deleted_at` set | Deleted | Same |
| `users.unsubscribe_at` set | Unsubscribed from emails (set on deletion) | Cleared when staff restore the account |

## Timers and automatic actions

None. Deletion takes effect immediately.

## Notifications

None. No confirmation email is sent when an account is deleted.

## Common questions

### "How do I delete my account?"
**Answer for the user:** Brands: go to Profile on the website and click "Delete Account" at the bottom. Influencers: in the app, open Profile → settings (gear) → "Delete Account". Please withdraw or use any balance first, because deletion doesn't pay it out, and you won't be able to sign up again with the same email.
**What to check:** Open payments: brand wallet balance, influencer pending earnings or payout requests, paid deals still running.
**Escalate when:** The user has money in the account or an active paid deal and still wants to delete.

### "I deleted my account by mistake. Can I get it back?"
**Answer for the user:** We can usually restore it. Please confirm the email address of the account.
**What to check:** `users.archived_at`/`deleted_at`. For brands, note that subscriptions were cancelled and campaigns will need admin re-approval after restoring.
**Escalate when:** Always: staff restore it in the admin panel.

### "I want to sign up again with the email of my deleted account."
**Answer for the user:** That email can't be used for a new account, but we can restore your old one.
**What to check:** As above.
**Escalate when:** The user wants it restored.

### "Will my subscription be refunded if I delete my account?"
**Answer for the user:** Deleting cancels your plan immediately. Refunds for the current billing period aren't automatic; we'll check your case.
**What to check:** Subscription status and last invoice (SUB-01/SUB-02).
**Escalate when:** The user asks for a refund (finance team).

### "How do I change my name?"
**Answer for the user:** Brands: Profile on the website, edit "First Name" / "Last Name", then save. Influencers: Profile tab → tap your name → "Edit Account Information" → "Save".
**What to check:** If a team member can't open Profile, they are inside another brand's team (BRD-03).
**Escalate when:** Saving fails repeatedly.

### "How do I change my phone number?"
**Answer for the user:** The phone number can't be edited in the app today. Tell us the new number and we'll update it.
**What to check:** `users.phone`.
**Escalate when:** Always (admin action).

## Known issues

- The mobile "Edit Account Information" screen shows an editable "EMAIL" field, but saving a different email is refused (ACC-04).
- Deleting doesn't settle balances. Escalate to finance if a deleted user asks for their money.

## Related topics

- ACC-02: what deleted users see at login.
- ACC-04: changing the email.
- BRD-02: company profile fields on the same page.
- BRD-03: team clean-up when a brand is deleted.
- SUB-02: subscription cancellation.
- PAY-01 / PAY-04: balances and withdrawals.
- INF-05: managing or removing social accounts.

## Technical reference

- **Endpoints**
  - `DELETE /api/v2/user/current-user` → `Modules/User/Http/Controller/UserController@deleteCurrentUser` → `Modules/User/Services/UserService::deleteById` (sets `archived_at`, clears `email_verified_at`, sets `unsubscribe_at`) and fires `BrandDeletedEvent` → listeners `DeleteCxaUserListener`, `Modules/Campaign/Listeners/DeleteBrandCampaignsListener` (active ads → `approval = rejected`), `DeleteBrandInvitationListener`, `Modules/Team/Brand/Listeners/CleanUpTeamListener`, `DeleteUserTokensListener`, `CancelBrandSubscriptionsListener` (`SubscriptionService::cancelAllNow`).
  - `PUT /api/v3/brand/profile` → `ProfileController@update` (names + company profile; `team.blocker`).
  - Mobile: `PUT /api/en/v3/mobile/user/profile` → `Mobile/.../UserController@updateProfile`; `DELETE /api/en/v3/mobile/user/profile/remove` → `UserService::remove_account` (archive, unsubscribe, delete tokens).
  - Admin restore: `PUT /api/v2/admin/users/{id}/restore` → `Services/Implementation/Admin/UserService::restore`.
- **Data**: `users.first_name`, `last_name`, `phone`, `archived_at`, `deleted_at`, `unsubscribe_at`, `email_verified_at`; `tokens` (removed on deletion).
- **Web**: `src/components/pages/userInfo2` (`ProfileForm.tsx`), strings `locales/en/userInfo.json`. **Mobile**: `src/features/profile/screens/EditAccountScreen.tsx`, `SettingsScreen.tsx`.
