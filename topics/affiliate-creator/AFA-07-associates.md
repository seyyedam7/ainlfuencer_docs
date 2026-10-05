---
id: AFA-07
title: Associates (Amazon Associates via Ainfluencer)
domain: Affiliate app (creators)
audience: [affiliate, admin]
apps: [affiliate-app]
related: [AFA-05, AFA-08, AFA-02]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-07 · Associates (Amazon Associates via Ainfluencer)

> A separate, invite-only programme. Selected affiliates promote any Amazon product using **Ainfluencer's own Amazon Associates tracking tags**, not a seller partnership. Sales data is imported by the Ainfluencer team from Amazon reports.

## Who and where
- **Only affiliates the Ainfluencer team has given access to.** Without it the menu doesn't appear at all.
- Sidebar **"Associate"** with three pages:
  - **Explore** `/app/associates/explore`: banner "Get Extra Commission via Ainfluencer", product and brand browsing, "Get Promo Link".
  - **My Tags** `/app/associates/my-tags`: tracking tags and "Social Media accounts".
  - **Analytics** `/app/associates/analytics`: sales chart and table.
- **Admins** grant access, assign tracking tags, approve associate social accounts and import sales reports in the affiliate admin panel.

## How it works
1. **Access.** An admin turns on the "associate access" permission for the affiliate's account.
2. **Tags.** The team assigns the affiliate one or more Ainfluencer tracking tags. On "My Tags" the affiliate sees them and can set one as "Default".
3. **Find products.** Explore lists Amazon products and brands (sort options such as "Best Sellers", "Highest Commissions", "Newest Products"). Under "Can't find what you want to promote?" the affiliate can paste product ASINs. The system then looks those products up on **Amazon US** in the background and adds them ("Request sent successfully").
4. **Get a link.** "Get Promo Link" → "Select a tracking tag to generate link" → copy the "Full link" or "Short link".
5. **Social accounts.** On "My Tags", "Add New Social Media" submits an account for the team to approve (pending → accepted / rejected).
6. **Analytics.** The team imports Amazon Associates sales reports. The affiliate then sees "Sales Units", "Shipped Items Revenue", "Total Earnings" and per-item details by date and tag.

## Rules and limits
- Visible only with the access permission. Removing the permission hides the menu.
- Product lookups by ASIN use the **US** Amazon marketplace only.
- **Payment:** no automatic payout exists for Associates earnings. They are **not** part of the Stripe commission payouts in AFA-08. Any payment is arranged by the Ainfluencer team outside the app *(to verify with finance)*.
- Analytics show only what has been imported, so numbers lag behind Amazon.

## Statuses
| Item | Values |
|---|---|
| Associate social account | pending, accepted, rejected |

## Timers and automatic actions
| When | What |
|---|---|
| Weekly | Associate product data refreshed from Keepa |
| Monthly | New associate products discovered from Keepa |
| On demand | ASIN lookup after "Find products to request" |

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| ASIN lookup finished | Requesting affiliate | Notification *(channel to verify)* |

## Common questions

### "I don't see the Associate menu."
**Answer for the user:** Associates is an invite-only programme that the Ainfluencer team turns on for selected creators. Contact us if you'd like to be considered.
**What to check:** The account's custom permission "associate access".
**Escalate when:** Access was granted but the menu still doesn't show (ask them to log out and back in first).

### "How do I get paid for Associates sales?"
**Answer for the user:** Associates earnings are handled separately from regular commission payouts. The Ainfluencer team will contact you about them; they don't go through your Stripe payouts.
**What to check:** —
**Escalate when:** Always forward payment questions on Associates to the finance team.

### "My Associates analytics are empty or out of date."
**Answer for the user:** Associates sales are imported from Amazon reports by our team, so they appear with a delay.
**What to check:** Date of the last imported report for this affiliate.
**Escalate when:** Nothing has been imported for more than a month.

### "I pasted ASINs but nothing appeared."
**Answer for the user:** The lookup runs in the background and only covers Amazon US products. Check back later.
**What to check:** That the ASINs are valid US ASINs.
**Escalate when:** Valid US ASINs never appear after a day.

## Known issues
- No automatic payout for Associates earnings (manual process).
- The ASIN lookup is US-only, even though the page has an "Amazon Region" filter.

## Related topics
- AFA-05 · Regular affiliate links (seller partnerships).
- AFA-08 · Regular commission payouts (Stripe).
- AFA-02 · Promotion platforms (separate from associate social accounts).

## Technical reference
- **Endpoints:**
  - `GET affiliate/v1/associates/associate-products/list`, `.../show/{id}`, `POST .../scrape` (`ScrapeAssociateProductService` → `KeepaAssociateProductScraperJob`, `AmazonLocale::US`).
  - `GET affiliate/v1/associates/associate-brands/list`.
  - `GET affiliate/v1/associates/associate-promote-tags/list`, `PATCH .../{promoteTag}/default`.
  - `GET affiliate/v1/associates/associate-sale-reports/list`, `.../chart`.
  - `GET/POST affiliate/v1/associates/associate-social-medias/*`.
- **Access:** `user_types.custom_permission` contains `associate_access` (admin `AssignCustomerPermissionController`).
- **Data:** associate products/brands (Keepa), promote tags (default tag config `AFFILIATE_AMAZON_TAG`), `associate_sale_reports` (`tracking_id`, `items_shipped`, `returns`, `revenue`, `ad_fees` shown as "Total Earnings"), associate social media (`status` 0/1/2).
- **Jobs:** `keepa:sync-associate-products` (weekly), `keepa:fetch-associate-products` (monthly).
- **Frontend:** `spa-affiliate/src/components/associate/**`.
