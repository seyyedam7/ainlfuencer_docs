---
id: AFA-10
title: Reports and dashboard (affiliate)
domain: Affiliate app (creators)
audience: [affiliate]
apps: [affiliate-app]
related: [AFA-08, AFA-05, AFA-06, AFS-09, MKT-03]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-10 · Reports and dashboard (affiliate)

> The affiliate's home dashboard and the "Reports" section: clicks, add-to-carts, sales, conversions and commission, broken down by brand, product, link source and landing template, with CSV export.

## Who and where
- **Affiliates** in the affiliate app:
  - **Dashboard** `/app`: a "Complete Profile" banner (until 100%), cards for brand and product partnerships ("Active", "Pending", "Rejected"), the **"Upcoming Payments"** card ("Total Sales", "Revenue", "Remaining"), a last-week performance chart, statistic boxes, top products, recent payments and recent partnerships.
  - **Reports**: sidebar "More" → "Reports" → tabs **"Performance"**, **Brand** (or "Store" for Shopify), **"Product"**, **"Sources"**, **"Landings"** (`/app/reports/...`). The button "Export Data As CSV" downloads the current report.
- The Ainfluencer mobile app shows a summary of the same data (MKT-03).

## How it works
1. Data depends on the platform picked in the header (Amazon marketplace or Shopify).
2. **Amazon numbers** come from the seller's Amazon Attribution reports. They're pulled daily at 06:00 UTC for stores whose last pull is older than **72 hours**, covering the **last 60 days**. So recent days can change and appear 1–3 days late.
3. **Shopify numbers** come from the Ainfluencer tracking pixel on the seller's store (page views, add-to-cart, checkout, purchase). They show sales and units, but **commission stays 0** (AFA-08).
4. Performance shows "Total Clicks", "Conversions", "Avg Conversion Rate", "Sales", "Commission" over a date range ("From Date" / "To Date").
5. Brand, Product, Sources and Landings break the same numbers down. "Sources" uses the source chosen when creating links; "Landings" uses the landing template (AFA-05, AFA-06).
6. **Dashboard "Upcoming Payments"**:
   - "Total Sales" and "Revenue" are all-time totals.
   - "Remaining" is commission on sales at least 35 days old that **hasn't been billed yet**, before the platform fee (AFA-08).

## Rules and limits
- **"Demo Data:" switch** on every report page replaces the numbers with generated sample data. It's off by default. Screenshots from users may show fake numbers if it's on.
- Commission in reports is the **gross** commission (before the platform fee) and includes sales not yet billed.
- A seller changing a commission rate re-prices sales from the last 60 days that haven't been billed yet, so report commission can change.

## Statuses
None (read-only reports).

## Timers and automatic actions
| When (UTC) | What |
|---|---|
| Daily 06:00 | Amazon report pull (stores older than 72 h, last 60 days) |
| Real time | Shopify pixel events |
| Weekly / monthly | Statistics emails (AFC-02) |

## Notifications
Weekly (Monday 01:00 UTC) and monthly (last day of the month 01:00 UTC) statistics emails to affiliates, see AFC-02.

## Common questions

### "My report numbers don't match Amazon."
**Answer for the user:** We receive Amazon's attribution reports every few days, and Amazon keeps updating the last weeks of data. Small differences and a delay of 1–3 days are normal.
**What to check:** The store's last report sync; whether "Demo Data" is on.
**Escalate when:** Nothing has updated for more than 5 days for a connected store.

### "The report shows huge numbers I don't recognise."
**Answer for the user:** Please check that the "Demo Data" switch at the top of the report is off. When on, it shows sample data.
**What to check:** —
**Escalate when:** Numbers are still wrong with the switch off.

### "Why is my Shopify commission 0?"
**Answer for the user:** Shopify commission isn't being calculated at the moment. We've reported it to the team.
**What to check:** AFA-08.
**Escalate when:** Always log it with dev.

### "What does 'Remaining' on the dashboard mean?"
**Answer for the user:** It's commission on sales that are at least 35 days old and are waiting to be billed to the seller. Once billed, it moves to your payment history (minus the platform fee).
**What to check:** —
**Escalate when:** "Remaining" stays high for weeks and nothing is billed (AFA-08).

### "My commission in the report went down."
**Answer for the user:** The seller may have changed the commission rate. Changes apply to sales from the last 60 days that haven't been paid yet.
**What to check:** Commission change notifications; the product's commission history.
**Escalate when:** —

## Known issues
- "Demo Data" can make reports look wrong if left on.
- Shopify commission is always 0 (AFA-08).
- *(To verify)* The shopper site loads each product page from the server twice, so product-page views may be counted twice.

## Related topics
- AFA-08 · Payouts and the 35-day rule.
- AFA-05 / AFA-06 · Sources and landing templates used in breakdowns.
- AFS-09 · The seller's version of these reports.
- MKT-03 · The summary in the Ainfluencer mobile app.

## Technical reference
- **Endpoints:**
  - Dashboard: `GET affiliate/v1/dashboard/report/{upcoming-payments|brands-partnership|products-partnership|top-products|recent-payments|statistics}`, `GET affiliate/v1/dashboard/contracts`, `GET affiliate/v1/profile/profile-completion-percentage`.
  - Reports: `GET affiliate/v1/reports/performance[/chart]`, `POST .../performance/export`; `POST affiliate/v1/reports/{brand|product|source|landing-page}[/chart|/export]`.
- **Sources:** `amazon_reports` (from `seller-account:update-report`, daily 06:00) and `shopify_reports` (pixel `POST client/v1/callback/shopify/track/event`). Upcoming: `Services/Actions/Affiliate/Dashboard/ShowUpcomingPaymentService`.
- **Frontend:** `spa-affiliate/src/components/report/**` (demo data in `*/fake-data.js`), `dashboard/affiliate-dashboard.page.vue`.
