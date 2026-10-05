---
id: AFS-05
title: Finding affiliates, lists and revealing emails
domain: Seller app (brands)
audience: [seller]
apps: [seller-app]
related: [AFS-06, AFS-08, AFS-11, AFS-12, AFA-02, AFC-01, CHAT-01, DSC-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-05 · Finding affiliates, lists and revealing emails

> Sellers look for creators to partner with in **Affiliate Discovery** (registered affiliates) and the **Expanded Network** (creators who aren't registered yet), save them in **lists**, and can **reveal contact emails**: a free daily number on the plan, then $0.10 each from the seller wallet.

## Who and where

- **Seller** (team permissions "Affiliates" and "Lists", AFS-13).
- Sidebar "Affiliates & Influencers" → "Affiliate Discovery" (`/app/discovery-affiliates?grouped=true`, also the page after login). "Influencer Discovery" in the same menu opens Ainfluencer's own search (DSC-01).
- Affiliate profile: `/app/affiliate/<id>`.
- Expanded Network: from Discovery, button "Invite via Email Campaigns / +72.5M Affiliates" → `/app/expanded-network`.
- Lists: sidebar "More → My Lists" (`/app/discovery-lists`), list detail `/app/lists/affiliates/<id>`.
- The user menu shows "Daily Invites", "Daily Email Revealed" and "Daily New Conversations" (used / limit).

## How it works

**Affiliate Discovery**
1. The landing view "Find the Perfect Affiliates Effortlessly" shows groups of affiliates with "View All" and "See All Affiliates", plus the seller's daily quotas.
2. The full list has search, grid/table view and filters: Platforms, Categories, Partnership status, number of partners, Location, Gender, Promotional methods, "Only Pro Affiliates", and follower ranges per network (YouTube, Instagram, Facebook, TikTok, Pinterest, X).
3. Each card: "Invite affiliate" (AFS-06), "Add to list", "Send message".
4. The affiliate page shows their platforms, contracts with the seller (brand and product), "Show Affiliate Email", "Add To List", "Request New Partnership", related affiliates and a link to the affiliate's public storefront.

**Expanded Network** ("Your Next Top Affiliates Are Just a Click Away")
- Creators imported by Ainfluencer who haven't joined the platform. Filters, a detail drawer with networks and stats; emails, phones and websites show "Reveal to access".
- Actions: "Reveal Emails (n)", "Add To List". The intended use is to reach them by email (Email campaigns, AFS-08).

**Lists**
1. "Create New List": **Manual List** ("you manually add affiliates") or **Smart List** ("automatically updates based on your filters"). "NOTE: After creation, Smart List filters will be permanent and cannot be changed."
2. List card actions: pin, "Edit List Name", "Duplicate as Manual", "Delete", "Add Affiliate", "Send Email" (AFS-08), "Message", "Emails Reveal", "Send Partnership Request".
3. From My Products, "Promote Products To My Lists" sends product invitations to everyone in the chosen lists (AFS-06).
4. List detail shows members with revealed/unrevealed counts; members can be removed from manual lists only ("You can't remove affiliate from a smart list.").

**Revealing emails**
1. From the affiliate page ("Show Affiliate Email"), a list ("Emails Reveal"), the Expanded Network, or Email campaigns step 3.
2. If the selection fits within the free reveals left today, the plan option is offered. Otherwise the wallet option appears: "Pay for remaining emails using Wallet", at **$0.10 per email**.
3. Only emails not revealed before are charged. On success: "Emails revealed successfully". Revealed contacts stay revealed.
4. When a partnership is approved, the seller gets the affiliate's contact info for free (AFS-06).

## Rules and limits

- **Who appears in Discovery:** affiliates who finished onboarding and have at least one approved social account with an approved channel (AFA-02). Others are invisible.
- **Free reveals** come from the plan (AFS-12) and are counted over the **last 24 hours** (rolling), not per calendar day. Wallet reveals also count toward that 24-hour usage.
- **Wallet price:** $0.10 per reveal (system setting). The wallet must hold enough money; top up in Wallet (AFS-11).
- Invites from these screens follow the invite rules in AFS-06 (daily limit, card on file).
- List names up to 100 characters. Smart lists can't be edited after creation (only renamed), and Expanded Network lists are manual only.
- Messaging: tell users to contact creators through the **Ainfluencer chat** (CHAT-01). The seller app's own "Message" buttons are not a supported channel (AFC-01).

## Statuses

| Item | Internal | Meaning |
|---|---|---|
| Revealed / unrevealed | `information_reveals` row from seller to affiliate/unlisted affiliate | A row = revealed (free or paid) |
| Manual / smart list | `user_type_lists.is_filterable` false/true | Smart lists recalculate from `filter_data` |
| Partnership status filter | computed: pending, partnered, not partnered, revoked, rejected, expired | Shown on cards and filters |

## Timers and automatic actions

- Free reveal allowance: rolling 24 hours.
- Daily invites reset at 00:00 UTC (AFS-06).

## Notifications

None specific. Wallet usage appears in Wallet → Transactions History → "Usage" (AFS-11).

## Common questions

### "Why can't I find a certain affiliate in Discovery?"
**Answer for the user:** Discovery only shows affiliates who completed their setup and have an approved social account. If they're new or their platforms are still under review, they won't appear yet.
**What to check:** affiliate `user_types.register_completed`; `user_social_accounts` / channels approved (AFA-02).
**Escalate when:** both are fine and the affiliate is still missing.

### "Reveal Emails says I've reached my daily limit, but the counter shows one left."
**Answer for the user:** The free reveals are counted over the last 24 hours, and the last free reveal can't currently be used on its own. You can pay $0.10 per email from your wallet, or wait.
**What to check:** `information_reveals` with `payment_triggered` = true in the last 24 h vs. the plan's `information_reveal` value.
**Escalate when:** the user is blocked with several reveals left.

### "I have money in my wallet but it says insufficient balance."
**Answer for the user:** The balance check currently only counts whole dollars of your balance. Top up a little more and try again.
**What to check:** wallet `balance` vs. number of emails × $0.10.
**Escalate when:** the whole-dollar balance clearly covers the cost.

### "The 'Reveal Emails' button on Discovery doesn't work."
**Answer for the user:** That bulk button is currently broken. Please reveal from the affiliate's profile ("Show Affiliate Email"), or add the affiliates to a list and use "Emails Reveal" on the list.
**What to check:** n/a (known issue).
**Escalate when:** the profile/list reveal also fails.

### "'Send Partnership Request' on my list fails."
**Answer for the user:** That list action is currently broken. Use "Promote Products To My Lists" on My Products, or invite affiliates individually.
**What to check:** n/a (known issue).
**Escalate when:** product promotion to lists also fails.

### "Can I change the filters of my smart list?"
**Answer for the user:** No, smart-list filters are fixed after creation. Create a new smart list, or "Duplicate as Manual" to edit members by hand.
**What to check:** n/a.
**Escalate when:** n/a.

### "How do I contact an affiliate?"
**Answer for the user:** Please use the Ainfluencer chat to talk to creators. Once a partnership is approved you also get their contact details.
**What to check:** n/a (CHAT-01).
**Escalate when:** n/a.

## Known issues

- **Bulk "Reveal Emails" in Discovery fails** (calls an address the server doesn't have; 404). Workaround: reveal from the profile or a list.
- **"Send Partnership Request" from a list fails** (no server route). Workaround: "Promote Products To My Lists" or individual invites.
- **Last free reveal can't be used**: the plan check refuses when the request uses up exactly the remaining allowance.
- **Wallet check rounds the balance down to whole dollars** (e.g. $0.95 can't pay for one $0.10 reveal).
- **The wallet modal shows a total for all selected emails**, but only not-yet-revealed ones are charged.
- **Allowance wording**: plan pages may describe reveals and invites "per month", while the system counts them per day / per 24 hours *(to verify with product)*.

## Related topics

- AFS-06 — Inviting affiliates and handling partnerships.
- AFS-08 — Email campaigns to revealed emails and lists.
- AFS-11 — Wallet top-up and transaction history.
- AFS-12 — Plan quotas for reveals and invites.
- AFA-02 — Why an affiliate's platforms may not be approved.
- CHAT-01 — Supported messaging channel.

## Technical reference

- **Endpoints:** `GET seller/v1/discover/affiliates` (`IndexAffiliateService`: `register_completed` + available social account/channel), `GET seller/v1/discover/affiliates/landing`, `GET seller/v1/profile/affiliate/{id}`; unlisted `GET seller/v1/unlisted-affiliates`, `…/filter-options`, `…/{id}`; reveal `POST seller/v1/reveal-information/process` (`item_ids`, `model_type` user_type|unlisted_affiliate, `payment_method` plan|wallet), `GET seller/v1/reveal-information/list/{affiliates|unlisted-affiliates}`; lists `POST/PATCH/DELETE seller/v1/user-type-lists[/{id}]`, `GET …/affiliates`, `GET …/unlisted-affiliates`, `PATCH …/assign-items`, `DELETE …/{id}/delete-items`, `POST …/{id}/duplicate`, `POST …/revealed-stats`, `GET …/drop-down-affiliates`, `POST seller/v1/pins/toggle`. Missing routes: `POST seller/v1/reveal-emails/user-types`, `POST seller/v1/partnership/{brand|product}/request-as-list`.
- **Services:** `Services/General/RevealInformation/RevealService.php`, `PaymentHandlers/PlanPayment.php` (`(limit − spend) − count <= 0` → `DailyInformationRevealLimitReachedException`), `PaymentHandlers/WalletPayment.php` (price from purchasable feature `information_reveal` = 0.10 USD; `intval(balance)`), `RevealInformationLimitService.php` (spend = rows with `payment_triggered` in last 24 h). Docs: `docs/RevealInformation.md`, `docs/UserTypeList.md`.
- **Data:** `information_reveals` (`from_user_type_id`, `to_modelable_type/id`, `payment_triggered`, `created_at`), `unlisted_affiliates`, `user_type_lists` (`is_filterable`, `filter_data`, `type`), list items table, `wallets.balance`, `wallet_transactions` (action reveal_information), `purchasable_features`.
- **Frontend:** `spa-seller/src/components/discovery/affiliates/*`, `discovery/unlisted-affiliates/*`, `discovery/lists/*`, `spa-seller/src/mixins/reveal-info.mixin.js`, broken modal `affiliate-reveal-email-modal.vue`.
