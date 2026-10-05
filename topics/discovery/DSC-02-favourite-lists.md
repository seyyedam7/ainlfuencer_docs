---
id: DSC-02
title: Favourite lists
domain: Finding and inviting influencers (brand)
audience: [brand]
apps: [web]
related: [DSC-01, DSC-03, SUB-04, BRD-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# DSC-02 · Favourite lists

> Brands can save influencers into named lists ("favourites") and later invite a whole selection or list to a campaign.

## Who and where

- **Brand**, web app.
- **`/favorite-influencers`** — sidebar and header menu "Favorite Influencers". Heading "Influencers": "Your saved influencers, organized into lists you can share, export, or invite to campaigns."
- Saving from other pages: "Save to list" on Find Influencers cards and profile panel (DSC-01), the heart / "Add to Favorites" on the influencer profile, and the favourites pop-ups in inbox and analytics.
- Old favourites grids still open by URL: `/new-offersV2/favorite`, `/offers/favorite`, `/profile/favorites`.

## How it works

1. **Create a list**: "Create list" → "List name" (e.g. "Q4 priority influencers") → "Create list". From "Save to list" you can also "Create new list".
2. **Add** an influencer to one or more lists ("Added to {name}" / "Added to your favorite list successfully"). Adding someone already in the list is ignored.
3. **Browse**: the "All" tab shows everyone; each list shows its members. Grid or List view, filters (Followers tiers, Gender, Quality), "Select all".
4. **Remove** from one list ("Removed from {name}") or, from the profile heart, from all lists at once ("Removed from favorites").
5. **Invite**: select influencers → **"Invite selected to campaign"** / "Invite to campaign" → **"Invite N influencer(s) to a campaign"**, "Pick campaign" (or "+ New campaign") → send. Toast "{count} influencer(s) invited to campaign". This uses the normal invitation flow (DSC-03).
6. **Delete list** removes the list (not the influencers' accounts).

## Rules and limits

- List names: required, up to **50 characters**, unique per brand user.
- Lists belong to the user who created them *(to verify whether team members share the owner's lists)*.
- Inviting from favourites uses the plan's **influencer invites** allowance like any other invite (SUB-04, DSC-03).
- Invites only go to influencers that have an email address on file (DSC-03).

## Statuses

None.

## Timers and automatic actions

- Adding/removing a favourite updates the brand's search index in the background, so the heart state in search may take a moment to update.

## Notifications

None to the influencer when added to a list. Invitations follow DSC-03.

## Common questions

### "I can't create a list."
**Answer for the user:** List names must be unique and at most 50 characters. Try a different name.

### "I removed an influencer but they're still in another list."
**Answer for the user:** Removing from a list only affects that list. To remove from all lists, open the influencer's profile and click the heart.

### "I invited my whole list but some influencers didn't get it."
**Answer for the user:** Invitations go out in grouped emails twice a day (06:00 and 18:00 UTC), only to influencers with an email on file, and not to ones who already sent you an offer for that campaign.
**What to check:** see DSC-03.

### "Inviting says I've reached my limit."
**Answer for the user:** Your plan's monthly influencer-invite allowance is used up. You can invite fewer people, buy an invite pack, or upgrade (SUB-04).

## Known issues

- The mapping evidence flagged two missing server actions ("toggle favourite" and "delete all favourites"). In the current code these calls are declared on several pages but **no visible button triggers them**; the working buttons use add/remove. If a user reports a favourites button that always fails, note the page and escalate *(to verify)*.

## Related topics

- DSC-01 search and profiles, DSC-03 invitations, SUB-04 invite allowance.

## Technical reference

- Endpoints: `GET/POST api/v2/favorite-lists`, `PUT/DELETE api/v2/favorite-lists/{id}` → `Http/Controllers/v2/Frontend/FavoriteListController` (title `max:50`, unique per `user_id`); `GET api/v2/favorite-influencers`, `POST …/add` `{integration_id|user_id, list_ids[]}`, `POST …/remove` `{integration_id|user_id, list_ids?}` (empty `list_ids` = remove from all) → `BrandFavoriteInfluencerController` → `Services/Implementation/BrandFavoriteInfluencerService` (dispatches favourite sync to Elastic).
- Invite from favourites: `POST api/v2/user/invitations` with `created_via=favorites`, `favorite_list_id` (or `all`) → `InvitationService::store` (DSC-03).
- Missing routes (no backend): `POST v2/favorite-influencers/toggle`, `DELETE v2/favorite-influencers/delete-all/{id}` (`lib/services/influencer.ts`).
- Data: `favorite_lists` (user_id, title), `brand_favorite_influencers` (favorite_list_id, integration_id).
- Web: `components/pages/favoriteInfluencersV2/*`, `components/pages/searchV3/SaveToListPopover.tsx`; strings `lib/localization/locales/en/favorites.json`.
