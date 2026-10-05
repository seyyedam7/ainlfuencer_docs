---
id: AFS-09
title: Reports and dashboard (seller)
domain: Seller app (brands)
audience: [seller, system]
apps: [seller-app, background]
related: [AFS-02, AFS-03, AFS-06, AFS-10, AFS-13, AFX-03, AFA-10]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-09 · Reports and dashboard (seller)

> The seller's dashboard and the Reports section show how affiliates are performing: clicks, add-to-carts, sales, conversions and commission, by day, brand, product and affiliate, plus Amazon's Brand Referral Bonus. Amazon numbers come from Amazon's attribution reports, pulled every few days; they are not live.

## Who and where
- **Seller** (and team members with the reports permission) in the seller app:
  - **Dashboard** (`/app`): "Complete Profile" and "Connect Your Credit Card" cards, counters for brands, products, affiliates, promotions and contracts, "Performance Report (Last Week)" chart, brand pie chart, Top Products, Top Affiliates, "Recent Partnerships" (with Details and Revoke), and incoming/outgoing partnership request tables.
  - **Reports** (sidebar **More → Reports**, `/app/reports/...`), tabs: "Performance", "Brand" (or "Store" for Shopify wording), "Product", "Affiliate", and "BRB" (only when an Amazon store is selected).
  - Every page shows the data of the store chosen in the header store switcher (AFS-02).

## How it works
1. **Where Amazon data comes from.** Affiliate links carry Amazon Attribution tags. Every day at 06:00 UTC the system asks Amazon for the attribution report of each active Amazon profile whose last pull is about 3 days old (72 h, compared by date), covering the **last 60 days**. Rows are matched to the affiliate's link, product and contract, and the commission is worked out (AFS-10).
2. **Where Shopify data comes from.** The Ainfluencer tracking pixel installed in the Shopify store sends page views, product views, add-to-cart, checkout and purchase events as they happen (AFX-03). Shopify numbers appear on the dashboard (top products, top affiliates, totals).
3. **Reports pages.** Each tab has a date range, filters (brands, products, affiliates), a chart and a table:
   - Performance: "Date", "Total Clicks", "Conversions", "Avg Conversion Rate", "Sales", "Commission".
   - Brand / Product / Affiliate: "Clicks", "Sales", "Add To Cart", "Conversions", "Conversion Rate", with "Select to compare".
   - BRB: "Date", "Amount" (Amazon Brand Referral Bonus) per product.
4. **Export.** "Export Data As CSV" downloads the current tab as a spreadsheet file (for example `performance_report.xlsx`). Needs the "export reports" permission ("You are not authorized to export reports.").
5. **"Demo Data:" switch.** Every report tab has a switch that replaces the real numbers with made-up sample numbers generated in the browser (filters hide while it's on). It's **off** by default and turns off again when the page is reloaded.

## Rules and limits
- **Amazon data is delayed:** a sale may take up to ~4 days to appear (72 h cycle, date-based, run at 06:00 UTC), and Amazon attribution itself can lag.
- **60-day window:** rows from the last 60 days are refreshed on each pull (so recent numbers can change, including commission after a commission change, AFS-03). Older rows are never refreshed.
- **Attribution window:** Amazon "14-day" metrics: a sale counts if it happens within 14 days of the click.
- **Sales after a product was deactivated** are dropped and don't appear (AFS-03).
- **Only active, connected Amazon profiles** are pulled. If Amazon refuses access during a pull (expired authorisation), the Amazon account is marked **disconnected** and pulls stop until the seller reconnects (AFX-03).
- **Reports tabs read Amazon data only.** Shopify activity shows on the dashboard but not in the Reports tabs.
- **Conversion rate** = purchases ÷ clicks.
- **Permissions (team members):** view reports, export reports (AFS-13).

## Timers and automatic actions
| When (UTC) | What | User notices |
|---|---|---|
| Daily 06:00 | Amazon report pull for profiles last pulled ~3+ days ago, last 60 days. | New/updated numbers. |
| Real time | Shopify pixel events counted. | Dashboard numbers move. |
| Daily (no fixed time) | Amazon profile/product refresh every 48 h (AFX-03). | Product list/status updates. |
| Weekly / monthly | Statistics emails (AFC-02). | Email summary. |

## Notifications
- None from the reports themselves. Weekly and monthly statistics emails are described in AFC-02.

## Common questions

### "My report shows sales that don't match Amazon Seller Central."
**Answer for the user:** Our reports only count sales Amazon attributes to your affiliates' links (Amazon Attribution, 14-day window), not all store sales. Data is refreshed every few days and covers the last 60 days.
**What to check:** `amazon_account_profiles.report_synced_at` for the seller's profiles; `amazon_reports` rows for the date range (`attributed_sales_14d`).
**Escalate when:** attributed sales in Amazon Attribution differ from `amazon_reports` for dates older than 4 days.

### "My reports are empty / stopped updating."
**Answer for the user:** Let's check your Amazon connection. If Amazon's authorisation expired, please reconnect the account in Profile → Manage accounts.
**What to check:** `amazon_accounts.status` (connected/disconnected), `amazon_account_profiles.status` (active), `report_synced_at`; whether the selected header store is Shopify (Reports tabs are Amazon-only).
**Escalate when:** account connected, profile active, and `report_synced_at` older than 4 days.

### "The screenshot from my team shows huge numbers we never had."
**Answer for the user:** That screenshot was probably taken with the "Demo Data" switch on, which shows sample numbers. Switch it off to see real data.
**What to check:** ask whether "Demo Data:" was on.
**Escalate when:** real data (switch off) still shows unexpected numbers.

### "I use Shopify. Why are my Reports tabs empty?"
**Answer for the user:** The detailed Reports tabs currently show Amazon data only. Your Shopify affiliate activity appears on the dashboard (top products, top affiliates and totals).
**What to check:** `shopify_reports` rows for the store; pixel installed (AFX-03).
**Escalate when:** the dashboard also shows nothing despite Shopify traffic from affiliate links.

### "Numbers for last month changed."
**Answer for the user:** Amazon updates attributed sales for recent weeks (returns, late attribution), and we refresh the last 60 days on each pull, so recent numbers can change slightly.
**What to check:** row `updated_at` and date in `amazon_reports`.
**Escalate when:** changes are for dates older than 60 days.

### "I can't export."
**Answer for the user:** Exporting needs the "export reports" permission; ask your account owner to add it to your role.
**What to check:** the member's role permissions (AFS-13).
**Escalate when:** the account owner gets the error.

## Known issues
- **Demo Data switch** can make screenshots show fake numbers.
- **Reports tabs ignore Shopify data** (Amazon only).
- **Export button says CSV** but the file is an Excel `.xlsx`.
- **Total commission on a contract** (shown in some partnership views) is overwritten by each billing run with only that run's amounts, so it is not a lifetime total (AFS-10).

## Related topics
- AFS-02 — store switcher; which store's data is shown.
- AFS-03 — product status and commission (affect report rows).
- AFS-06 — partnerships (contracts behind the numbers).
- AFS-10 — how report rows become commission charges.
- AFS-13 — report permissions for team members.
- AFX-03 — Amazon/Shopify connection health and the Shopify pixel.
- AFA-10 — the affiliate's own reports.

## Technical reference
- **Endpoints:** `seller/v1/reports/performance` (GET), `/performance/chart`, POST `/performance/export`; POST `seller/v1/reports/{brand|product|affiliate}` + `/chart` + `/export`; GET `seller/v1/reports/brb` + `/chart`, POST `/brb/export` → `app/Http/Controllers/Seller/V1/AmazonReport/*` → `app/Services/Actions/Seller/AmazonReport/*` (all query `amazon_reports` for the seller's profiles; metrics in the `reportFields` scope).
- **Dashboard:** `seller/v1/dashboard/report/{statistics|brands|brands/chart|products|promotions|contracts|affiliate-partnerships|top-products|top-affiliates}`, `seller/v1/dashboard/contracts` → `app/Services/Actions/Seller/Dashboard/*` (top products/affiliates switch on the `X-Platform` header between Amazon and Shopify).
- **Sync job:** `seller-account:update-report` (`app/Console/Commands/SellerAccount/UpdateSellerAmazonReport.php`), daily 06:00 UTC → `SyncSellerProfileAmazonReportJob` → `Services/General/AmazonReport/AmazonReport::saveSellerProfileAmazonReport` → `GetAmazonReport::getAdGroupReports` (4xx → `amazon_accounts.status = disconnected`) → `SaveAmazonReport::saveAdGroupReport` (upsert on date + profile + link). Config: `ainfluencer.seller.amazon_update_report_period` 60 days, `amazon_performance_report.update_seller_time` 72 h.
- **Shopify pixel:** POST `client/v1/callback/shopify/track/event` → `ShopifyPixelEventHandlerJob` → `Services/General/Shopify/Report/Handlers/*` (increments `shopify_reports`).
- **Data:** `amazon_reports` (`date`, `amazon_account_profile_id`, `affiliate_product_link_id`, `seller_id`, `affiliate_id`, `seller_product_id`, `seller_brand_id`, `clicks`, `attributed_add_to_cart_clicks_14d`, `attributed_purchases_14d`, `attributed_sales_14d`, `units_sold_14d`, `commission`, `brb_bonus_amount`), `amazon_account_profiles.report_synced_at`, `shopify_reports` (`date`, store, link, `clicks`, `add_to_cart`, `purchases_completed`, `total_revenue`, `units_sold`), `user_statistics` (dashboard totals).
- **UI:** `spa-seller/src/components/report/*` (fake data in `*/fake-data.js`, switch `useFakeData`), dashboard in `spa-seller/src/components/dashboard/*`.
