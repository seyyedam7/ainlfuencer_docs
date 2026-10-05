---
id: INF-05
title: Managing social accounts
domain: Influencer social accounts
audience: [influencer]
apps: [mobile]
related: [INF-01, INF-02, INF-04, INF-06, MKT-04, PUBL-02, ACC-05]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# INF-05 · Managing social accounts

> On the Profile tab the influencer sees a card for each connected social account. From there they can refresh the account's numbers ("sync"), edit the display name, description, categories and picture, share the public profile link, and manage their shipping addresses.

## Who and where

- **Influencer**, mobile app, **Profile** tab → "Social Accounts" ("Brands match deals to your live numbers"), with **Active** and **Archived** tabs and "+ Add".
- The **⋯** on a card opens **"Account options"**:
  - **"Sync with my Instagram/TikTok/YouTube"**
  - **"Edit Name (@username)"** — display name and description
  - **"Edit Category"**
  - **"Edit Profile Picture"** (upload or remove)
- Card buttons: "Edit Profile" + "Share" (Active), "Edit Profile" (Pending), "Complete Profile" (Draft), "Appeal Suspension" (suspended), "Resubmit Verification Request" (rejected manual request).
- Shipping addresses: Profile → "Shipping Address" card, or Settings → **"Shipping Address"** (list, add, edit, delete, map search).
- The owner's own first/last name is edited from the Profile header (ACC-05), not here.

## How it works

1. **Sync** asks the platform for the latest followers, posts and profile data and saves them.
   - Instagram and TikTok: allowed once every **24 hours** (server setting). Too soon → "You have recently synced your profile with Instagram. Due to the recent sync, you will have to wait for 23 hours before you can send this request again."
   - YouTube: uses the influencer's Google permission. The app blocks re-syncing for **5 minutes** ("Please wait N min before syncing again"). If the permission was withdrawn or expired, the message asks the influencer to reconnect YouTube.
   - If the username no longer exists, the account is **archived** (INF-04).
   - If the data provider is down: "Server is busy. Please try again in a few minutes."
2. **Edit Name / Edit Category / Edit Profile Picture** save immediately ("Name updated successfully", "Category updated successfully") and update what brands see in search.
3. **Share** opens the public media-kit link `app.ainfluencer.com/public/profile/<id>` (PUBL-02).
4. **Shipping addresses**: the first address added becomes the default; marking another as default replaces it. Addresses are used for gifting (MKT-04) and can be shared in chat.

## Rules and limits

- Editing (name, category, picture, Edit Profile) is **disabled for archived and suspended accounts**. **Sync stays available** for them.
- **Removing a social account is not available** in the app (and nothing in the web app does it either). If an influencer wants an account disconnected, escalate to an admin. Deleting the whole Ainfluencer login is ACC-05.
- Each social account can belong to only one Ainfluencer user (INF-01).

## Statuses

Card labels are explained in INF-03 (Draft, Pending Admin Approval, Active, Pending Verification, Verification Rejected), INF-04 (Disapproved, Archived).

## Timers and automatic actions

- Instagram/TikTok sync cooldown: 24 h after the last data update (also counts updates made by the monthly refresh, INF-06).
- YouTube: 5-minute cooldown on the phone only.

## Notifications

None.

## Common questions

### "My follower count is wrong / out of date."
**Answer for the user:** Tap ⋯ → "Sync with my Instagram" on the account card. You can sync once every 24 hours. We also refresh Instagram accounts automatically once a month.
**What to check:** `integrations.last_ig_update_date`, `followers`.
**Escalate when:** sync succeeds but the number stays clearly wrong.

### "Sync says I have to wait."
**Answer for the user:** Syncs are limited to once every 24 hours per account. Please try again after the time shown.

### "I can't edit my name or category."
**Answer for the user:** Editing is turned off while an account is suspended or archived. See your account's status on the Profile tab.

### "How do I remove one of my social accounts?"
**Answer for the user:** This isn't possible in the app yet. Tell us which account and we'll ask our team to remove it.
**Escalate when:** always (admin action).

### "YouTube sync asks me to reconnect."
**Answer for the user:** Your YouTube permission expired or was removed. Add the YouTube account again from "+ Add" and sign in with the same Google account.

### "How do I change my shipping address?"
**Answer for the user:** Settings → Shipping Address. For a gift that is already on its way, the address can't be changed; message the brand in the Ainfluencer chat (MKT-04).

## Known issues

- The evidence map said YouTube has the sync cooldown; in fact Instagram/TikTok have the 24-hour server limit and YouTube only has a 5-minute limit on the phone.

## Related topics

- INF-01 connecting, INF-02 Edit Profile steps, INF-04 suspended/archived accounts, INF-06 automatic refresh, MKT-04 gift addresses, PUBL-02 public profile, ACC-05 owner profile and account deletion.

## Technical reference

- Endpoints (`api/{locale}/v3/mobile`):
  - `GET influencer/accounts` (main + temporary), `GET influencer/accounts/{id}`, `PUT influencer/accounts/{id}` (name, description, categories, picture_id), `DELETE influencer/accounts/{id}/picture`, `POST influencer/accounts/{id}/sync` → `Modules/Mobile/Http/Controllers/API/AccountController`, `Modules/Mobile/Services/Implementation/AccountService`, sync → `Services/Implementation/Influencer/InfluencerService::syncInformation` (cooldown `config('ainfluencer.integration_sync_cooldown_minutes')`, env `INTEGRATION_SYNC_COOLDOWN_MINUTES`, default 1440; YouTube path `fetchYouTubeChannel` has no cooldown).
  - `GET/POST/PUT/DELETE addresses` → `AddressController` → `Services/Implementation/Common/AddressService`.
  - `GET influencer/accounts/{id}/pricing`, `…/settings`, `…/insights` are used by offer screens (OFR-01).
- `DELETE api/v2/integrations/{id}` exists, but the web notification list only declares the call and never triggers it.
- Data: `integrations` (full_name, description, picture_id, followers, last_ig_update_date, archived_at, qc_status), `identities` (name, description, logo_id, categories), `addresses` (user_id, default).
- Mobile: `features/profile/components/Accounts/{index,AccountCard,AccountCardHeader,ActionButton,OptionsOverlay}`, `features/onboarding/utils/youtubeSyncCooldown.ts` (5 min), `features/profile/screens/{EditName,EditCategory,SettingsShippingAddressScreen}`.
