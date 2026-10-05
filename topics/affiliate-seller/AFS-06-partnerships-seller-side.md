---
id: AFS-06
title: Partnerships (seller side)
domain: Seller app (brands)
audience: [seller, affiliate, system]
apps: [seller-app, email, background]
related: [AFA-04, AFS-03, AFS-04, AFS-05, AFS-07, AFS-10, AFS-12, AFX-02, AFB-04, AFC-02, CHAT-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-06 · Partnerships (seller side)

> A **partnership** lets an affiliate promote a seller's brand or products with tracked links. It starts as a **request**: the seller invites an affiliate, or the affiliate applies. When the receiving side approves, the request becomes a **contract** with a fixed commission per product, the affiliate gets links, and sales start earning commission. The affiliate side is AFA-04.

## Who and where

- **Seller** (team permission "Partnership": view, request, invite, revoke; AFS-13).
- Sidebar "More → Partnership" (badge = new applications): `/app/partnerships/partnership-products` and `/app/partnerships/partnership-brands` (a "Products / Brands" switch; "Stores" for Shopify). Tabs: **Applications**, **Partnered**, **Invitations**, **Revoked**, **Rejected**.
- "My affiliate list" → `/app/my-affiliates`: partnered affiliates with platforms, active products, total sales, revenue and partner date.
- Dashboard: "Recent Partnerships" (Details, Revoke) and incoming/outgoing request tables.
- Invite buttons: Discovery cards, the affiliate page ("Request New Partnership"), My Products ("Promote", "Promote Products To My Lists"), brand page.

## How it works

**Inviting affiliates**
1. "Invite affiliate" → choose **Brand** or **Product**.
2. Pick a brand/store (only active brands without an existing partnership with that affiliate are offered); for products, pick active, not-yet-partnered products.
3. Optional (products only): "Propose custom commission" (more than 0, up to 100%; defaults to the product's commission).
4. Write the "Message for invitation" and send. For several affiliates, the summary shows "Total selected / Currently Partnered / Unpartnered" and "Daily limit left"; combinations that already exist are skipped.
5. Invitations are created in the background. They appear under **Invitations** as pending. The affiliate accepts or rejects in the affiliate app.

**Handling applications** (Applications tab)
- **Accept** → "Partnership Successfully Initiated – You can now check your contract details in {Brand} active contracts section." with a "Start Chat" button. For conversations, ask both sides to use the **Ainfluencer chat** (CHAT-01).
- **Reject** → "Are You Sure to Reject Partnership? If you reject a partnership you cannot revert this action."
- With **"Automatic Partnership Approval"** on for the brand/store (AFS-03), applications without a custom commission are approved instantly; applications proposing a custom commission still wait for the seller.

**Cancelling an invitation**: Invitations tab → cancel → "Are You Sure You Want to Cancel This Invitation?"

**What approval does**
1. A **contract** is created with one item per product and a **snapshot of each product's commission** (or the custom one).
2. The affiliate's product links are created automatically.
3. The seller's contact information is shared with the affiliate, and the affiliate's email is revealed to the seller, free of charge.
4. The other side is notified.
5. For **brand** contracts, products added to the brand later are added to the contract automatically.

**Revoking a contract**: Partnered tab or dashboard → Revoke → "Are You Sure You Want to Revoke This Partnership? Revoking this partnership will prevent your affiliate from promoting this {brand/product}, and all their efforts will be lost." The contract is closed, the affiliate's links for those products are revoked (shoppers see the affiliate's "product removed" behaviour), and private promotions are withdrawn.

## Rules and limits

- **A card is required to invite or to accept**: without a saved payment method the server answers HTTP 499 and the app jumps to Settings → Billing (AFS-10).
- **Daily invitation limit** (counts seller invitations created since 00:00 UTC) → "Daily Invitation Limit Reached – You've reached today's invitation limit. Upgrade your plan or wait until tomorrow for a reset."
- **Active contract limit** (when inviting or accepting) → "Active Contract Limit Reached".
- **Custom commission on product invitations** needs a plan that includes private commission; otherwise the upgrade pop-up appears (AFS-12). Brand invitations always use product commissions.
- Duplicate pairs (affiliate + brand/product already pending or partnered) are skipped: "All affiliate-brand combinations already have active or pending partnerships."
- Only the receiver can approve or reject; the sender can only cancel.
- Commission changes on a product later also update non-custom contract items (AFS-03).
- Contracts also end automatically when an Ainfluencer affiliate campaign finishes (AFX-02) or a plan downgrade removes them (AFS-12).

## Statuses

| Shown | Internal | Meaning / next |
|---|---|---|
| Pending (Applications / Invitations) | `partnership_requests.status` = `pending` | Waiting for the receiver |
| Partnered | request `approved` → `partnership_contracts.status` = `active` | Links work, sales earn commission |
| Rejected | request `rejected` | Ends the request |
| Cancelled | request `canceled` | Sender withdrew it |
| Revoked | contract `revoked` (`revoked_at`, `revoked_by`) | Links stop earning; final |

Request type: `seller_to_affiliate` (invitation) or `affiliate_to_seller` (application). The partnership filter in Discovery uses derived values: pending, partnered, not partnered, revoked, rejected, expired.

## Timers and automatic actions

| What | When (UTC) | Effect |
|---|---|---|
| New-requests email (`email:new-partnership-requests`) | daily 08:00 | Receivers with pending requests get an email |
| Ended-campaign revocation | daily 02:00 | Contracts of finished Ainfluencer campaigns revoked (AFX-02) |
| Invitation counter | 00:00 | Daily invite count restarts |

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| New invitation or application | Receiver | Web notification; daily 08:00 email with pending requests |
| Request approved / rejected / cancelled | The other party | Web + email (email in the 06:00/18:00 batch) |
| Contract revoked by seller or affiliate | – | No notification for manual revocation |
| Contract revoked by a plan downgrade | Both | Email |

## Common questions

### "I can't send invitations; I'm sent to Billing."
**Answer for the user:** A payment card is needed before you can invite affiliates or accept applications, because commissions are charged to it. Add a card in Settings → Billing → "Manage Credit Cards".
**What to check:** saved payment methods for the seller (`stripe_customer_payment_methods`).
**Escalate when:** a card is saved and the redirect still happens.

### "It says 'Daily Invitation Limit Reached'."
**Answer for the user:** Your plan allows a certain number of invitations per day; the count resets at midnight UTC. Upgrade or wait until tomorrow.
**What to check:** invitations created today vs. the plan's `affiliate_invite` value (user menu "Daily Invites").
**Escalate when:** the counter shows remaining invites and sending still fails.

### "I invited an affiliate but nothing shows in Invitations."
**Answer for the user:** Invitations are created in the background and pairs that already exist are skipped. Refresh in a minute; if the affiliate already has a pending or active partnership for that brand/product, no new invitation is created.
**What to check:** existing `partnership_requests`/contracts for the pair; queue `bulk-partnerships` health.
**Escalate when:** a new pair never appears after a few minutes.

### "Applications are approved without me doing anything."
**Answer for the user:** "Automatic Partnership Approval" is on for that brand/store. Turn it off on the brand page if you want to review each application. It may also be switched on or off when your subscription changes.
**What to check:** `auto_partnership_approval` on the brand/store; recent subscription change.
**Escalate when:** applications are auto-approved with the switch off.

### "An affiliate's links stopped working."
**Answer for the user:** The partnership was revoked (by you, the affiliate, a finished Ainfluencer campaign, or a plan change), or the product was deactivated. A revoked partnership can't be reopened; send a new invitation.
**What to check:** contract `status`, `revoked_at`, `revoked_by`; product status (AFS-03).
**Escalate when:** contract active and product active but links fail (AFB-01).

### "I accepted and clicked 'Start Chat'. Where do I talk to the affiliate?"
**Answer for the user:** Please use the Ainfluencer chat for conversations with affiliates and influencers. After approval you also have each other's contact details.
**What to check:** n/a (CHAT-01).
**Escalate when:** n/a.

### "Can I undo a rejection?"
**Answer for the user:** No. Ask the affiliate to apply again, or send them a new invitation.
**What to check:** n/a.
**Escalate when:** n/a.

## Known issues

- **"Send Partnership Request" from a list fails** (no server route). Use "Promote Products To My Lists" (AFS-05).
- **No notification when a contract is revoked manually**; the affiliate only notices that links stop.
- **"My Affiliates" page has no permission check** in the app; team members without Partnership access can open it by URL *(minor)*.
- **Allowance wording** may say "per month" on plan pages while invitations are counted per day *(to verify with product)*.

## Related topics

- AFA-04 — Affiliate side of the same requests and contracts.
- AFS-03 — Commission per product, auto-approval switch.
- AFS-05 — Finding affiliates and lists.
- AFS-07 — Private promotions for partnered affiliates.
- AFS-10 — Card requirement and commission charges.
- AFS-12 — Invite, contract and custom-commission limits.
- AFX-02 — Partnerships created by Ainfluencer campaigns.
- AFB-04 — Applications from the public brand page.
- CHAT-01 — Supported messaging channel.

## Technical reference

- **Endpoints:** `POST seller/v1/partnership/brand/request` (gates `affiliate_invite`, `active_contract`, `seller_stripe_connected` 499; job `ProcessBrandPartnershipRequest` on queue `bulk-partnerships`), `POST seller/v1/partnership/product/request` (+ gate `private_commission`; `type` affiliate|affiliate_list), `POST …/{brand|product}/check-pre-request`; received requests (approve/reject) `PATCH seller/v1/partnership/invited/{brand|product}/change-status/{id}` (gates `active_contract` when approving, card 499); sent requests (cancel) `PATCH seller/v1/partnership/applied/{brand|product}/change-status/{id}`; `GET seller/v1/partnership/requests`, `GET seller/v1/partnership/contracts`, `GET …/contracts/{id}`, `PUT …/contracts/{id}/revoke`; `GET seller/v1/partnership/affiliate` (My Affiliates), `GET …/affiliate/drop-down`.
- **Services/listeners:** `Services/Actions/Seller/Partnership/Request/Brand/BrandRequestService.php` (duplicate filtering), `Listeners/AutoPartnershipApproval.php` (affiliate-created requests only, skips custom commission), on approval `CreatePartnershipContract` (+ free email reveal), `CreateDefaultProductLink`, `CreateChangePartnershipNotification`; revoke `Services/General/Partnership/Contracts::revoke` → `PartnershipContractRevokedEvent` → `RevokeProductLinkListener`, `RemovePrivatePromotionAccessListener`. Limits: `Plan/PlanAccess/Seller/Checkers/AffiliateInviteFeatureChecker.php` (since start of day), `ActiveContractFeatureChecker.php`, `PrivateCommissionFeatureChecker.php`. Daily email `SendNewPartnershipRequestsEmailCommand` 08:00.
- **Data:** `partnership_requests` (`status`, `type`, `seller_type_id`, `affiliate_type_id`, `campaign_id`, `created_at`), `partnership_request_items` (`is_custom_commission`), `partnership_contracts` (`status`, `revoked_at`, `revoked_by`, `accountable_type/id`), `partnership_contract_items`, `partnership_contract_item_products` (`commission`, `is_custom_commission`), `affiliate_product_links.status`, `information_reveals`.
- **Frontend:** `spa-seller/src/components/partnership/*` (tabs, received/sent tables, modals), `discovery/affiliates/partnership-request-modal.vue`, `spa-seller/src/store/partnership.store.js`.
