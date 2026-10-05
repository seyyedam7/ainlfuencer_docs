---
id: DSC-01
title: Influencer search and profiles
domain: Finding and inviting influencers (brand)
audience: [brand]
apps: [web]
related: [DSC-02, DSC-03, SUB-04, PAY-01, OFR-03, PUBL-02, INF-06]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# DSC-01 · Influencer search and profiles

> Brands find influencers on the **Find Influencers** page with filters, open a quick profile or the full profile, and invite them to a campaign. Influencer handles are hidden ("locked") until the brand **reveals** them, using a monthly reveal credit from their plan or a small wallet payment. Hiring an influencer (a paid offer) reveals them automatically.

## Who and where

- **Brand** (and team members), web app.
- **`/search`** — sidebar "Find influencers", page title **"Find Influencers"**.
- Quick profile: "Profile" on a result card opens a side panel; "Full Profile" / "View full profile" opens **`/account/:id`**.
- `/accountnew/:id` (new "dossier" design) is reachable only from links inside itself; `/find/influencer-explorer` is an older search reached only by a "back" link.
- Public profile `/public/profile/:id` (no login) is PUBL-02.

## How it works

1. **Pick what you're recruiting for**: "Select campaign" (or "+ Create a new campaign"). The campaign type decides the action button: Collaboration → **"Invite to campaign"**, Affiliate → **"Invite to program"**, Seeding → **"Request to send product"**, Hybrid → "Invite to campaign".
2. **Filter**: "Username or keyword", #hashtag, Location, Categories, Language, Gender, Quality (Great / Good / OK), Influencer fees (rough estimate), Followers (Nano 100–10k, Micro 10k–100k, Macro 100k–1m, Mega over 1m, Custom). Seeding adds "Address ready to ship"; affiliate adds "Open to commission". Results come from Ainfluencer's search index (data refreshed per INF-06).
3. **Cards** show followers/subscribers, engagement, quality, fee estimate, "Save to list" (DSC-02) and the masked handle `@•••••••• (locked)`.
4. **Select** influencers ("{{count}} selected") and send invites (DSC-03), or save them to a favourites list.
5. **Profile** (`/account/:id` or side panel): about, social reach across platforms, gallery, ratings & reviews, audience, published content, add to favourites, invite to a campaign. Opening a profile counts a profile view.
6. **Reveal the handle**: click "Reveal". The pop-up **"Reveal {name}'s handle"** shows:
   - "Monthly reveal credits — X of Y left", "Resets <date>", **"Use 1 monthly credit"** → "Reveal handle · 1 credit";
   - or **"Pay $X from wallet"** ("Pay & reveal"), showing "WALLET BALANCE" and "THIS REVEAL";
   - when credits are used up: "You've used all N monthly reveals…" with "Upgrade" or pay one-off.
   After revealing, the handle shows as a link to the influencer's social profile. "Every reveal is logged to your team's audit trail."

## Rules and limits

- **Reveals are permanent and per influencer**: once revealed (by credit, by wallet, or because the brand paid an offer with that influencer), the brand sees the handle and real name on **all** of that influencer's accounts, for free, forever.
- A reveal by credit or wallet never charges twice for the same influencer (double clicks are safe).
- Monthly credits and wallet price come from the brand's plan (default values; the team can change them — see SUB-04):

| Plan | Reveal credits / month | Wallet price per reveal |
|---|---|---|
| Free | 5 | $1.00 |
| Starter | 50 | $1.00 |
| Pro | 100 | $0.75 |
| Ultimate | 6,000 | $0.50 |

  With no plan price defined, the wallet price falls back to $1.00.
- Credits reset with the plan's monthly period.
- Paying from the wallet needs enough balance (cash + credit wallets). Not enough → "You need to charge wallet" and a link to **Add funds** (`/funds`, PAY-01). Nothing is charged if the reveal fails.
- Running out of credits returns the plan-limit pop-up (HTTP 402, "Your plan does not allow more username reveals this month.").
- Only accounts that passed review and are not archived appear in search (INF-03, INF-04, INF-06) *(to verify the exact search filters)*.

## Statuses

Not applicable (a handle is either locked or revealed).

## Timers and automatic actions

- Reveal credits reset at the start of each monthly plan period.
- Paying for an offer automatically reveals that influencer (and uses a credit only if one is left; the deal never fails for lack of credits).

## Notifications

None.

## Common questions

### "Why can't I see the influencer's username?"
**Answer for the user:** Handles are locked until you reveal them. Click "Reveal" and use one of your monthly credits or pay a small amount from your wallet. Once revealed it stays visible for good, and influencers you hire are revealed automatically.

### "I paid to reveal but I still see the lock."
**Answer for the user:** Please refresh the page. If the charge appears in your transactions but the handle is still locked, contact us with the influencer's profile link.
**What to check:** `user_unlocked_integrations` row for the brand user and integration; `service_charges` (type `influencer_reveal`) for the charge.
**Escalate when:** a charge exists without an unlock row (money taken, nothing revealed).

### "My reveal credits ran out."
**Answer for the user:** Credits reset at the start of your next monthly period. Meanwhile you can pay per reveal from your wallet, or upgrade your plan for more credits.

### "Was I charged twice for the same influencer?"
**Answer for the user:** Each influencer is charged at most once; revealing again is free.
**What to check:** `service_charges` rows for that integration.
**Escalate when:** two charges for the same influencer and brand.

### "A teammate revealed an influencer but I still see it locked."
**Answer for the user:** Reveals are recorded for the account that made them. *(to verify how reveals behave inside teams)*
**Escalate when:** the team expects shared reveals.

### "The follower numbers look old."
**Answer for the user:** Instagram data is refreshed monthly (INF-06); the influencer can also sync it themselves.

## Known issues

- The search page sends invitations as "pending invitations" (see DSC-03 Known issues) *(to verify)*.
- `/accountnew/:id` and `/find/influencer-explorer` are old pages still reachable from a few links.

## Related topics

- DSC-02 favourite lists, DSC-03 invitations, SUB-04 plan limits, PAY-01 wallet top-up, OFR-03 paying an offer (auto-reveal), PUBL-02 public profile, INF-06 data refresh.

## Technical reference

- Search: `POST api/v2/elastic/extended/search` → `Http/Controllers/v2/Frontend/ExtendedElasticController@search` (`Services/Implementation/ElasticExtendedService`); old `POST api/v2/elastic/search` → `ElasticController`.
- Profiles: `api/v3/influencer/profile/{all-data,elastic-data,preview-data}/{id}`, `…/{id}/all-accounts`, `…/user/{userId}/all-data`, `PATCH …/hit-profile-views/{id}`, `POST …/feedbacks/like` → `Http/Controllers/v3/Influencer/ProfileController` (`Services/Implementation/Influencer/InfluencerProfileService`).
- Reveal: `POST api/integrations/{id}/reveal-username` (quota) and `…/reveal-username-by-wallet` → `Http/Controllers/v3/Brand/UsernameRevealController` → `Services/Susbscription/InfluencerRevealService` (`unlock`, `unlockByWallet`, `unlockForDeal`); masking helpers `app/Helpers/Functional/RevealInfluencerIdentity.php` (`can_view_influencer_identity`, `unlocked_influencer_access`), deal unlock `Observers/v2/OfferPaidUnlockObserver`. Price fallback `config/influencer_reveal.php` (`default_price_cent` 100). Plan values: features `influencer_username_reveal` / `influencer_username_reveal_price` in `plan_price_feature` (seed `database/seeders/Subscription/plan_price_feature_data_production.php`).
- Data: `user_unlocked_integrations` (user_id, integration_id), `service_charges` (type `influencer_reveal`, amount_cent, user_id), `feature_usages`.
- Web: `components/pages/searchV3/*`, `components/pages/influencerProfile`, `components/common/revealUsername/*`; strings `lib/localization/locales/en/search.json`, `common.json` (`common.revealUsername`).
