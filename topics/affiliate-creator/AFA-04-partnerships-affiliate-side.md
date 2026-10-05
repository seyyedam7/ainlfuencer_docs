---
id: AFA-04
title: Partnerships (affiliate side)
domain: Affiliate app (creators)
audience: [affiliate, seller]
apps: [affiliate-app, buyer-site, email]
related: [AFS-06, AFA-03, AFA-05, AFA-08, AFA-09, AFB-04, AFX-02, CHAT-01]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFA-04 · Partnerships (affiliate side)

> A partnership is the agreement between an affiliate and a seller to promote a product or a whole brand/store at a set commission. The affiliate applies (or the seller invites); once approved it becomes a **contract**, and only then do the affiliate's links earn commission.

## Who and where
- **Affiliates** in the affiliate app:
  - Sidebar "More" → **"Partnership"** (badge = open requests) → `/app/partnerships/products` and `/app/partnerships/brands`. Each has the sub-tabs **Invitations, Partnered, Applications, Revoked, Rejected**.
  - **"My Sellers List"** `/app/my-sellers`: sellers they have contracts with.
  - Dashboard request tables on `/app`.
  - Starting points in Discovery: "Generate Link", "Apply For Partnership", "Promote" (AFA-03).
- **Guests** on a seller's brand page on the buyer site ("Apply For Brand Partnership", AFB-04).
- **Sellers** handle the other side (AFS-06).

## How it works
**Applying for products ("Generate Link")**
1. In Discovery the affiliate selects one or more products and clicks "Generate Link".
2. The system checks for duplicates first. Products the affiliate already has a pending request or an active contract for (directly or through the brand) are skipped. If all are covered, the modal says "You have already partnered with all products." with a "Go to Managment" button.
3. If the brand/store has **automatic approval** on, the request is approved straight away. The contract and a default link are created, and the modal shows the links ready to copy.
4. For products that need seller approval, the affiliate may enter a commission % ("You can enter commission products that require seller approval here.", field "Default Commission"). Clicking "Next" sends the request to the seller as **pending**. The proposed % is shown to the seller, who decides.
5. Applying means agreeing to the "Sellers' Terms of Sales."

**Applying for a whole brand/store**
- "Apply For Partnership" on a brand/store or seller page sends a brand request with an optional message. With auto-approval it's approved at once and covers **all the brand's products at their default commission**. New products the seller adds later join the contract automatically.

**Answering seller invitations**
- Under "Invitations", the affiliate can "Accept" (shows "Partnership Successfully Initiated") or "Reject". **Rejecting can't be undone** ("If you reject a partnership you cannot revert this action.").

**Withdrawing and ending**
- "Cancel Application" withdraws a request that's still pending.
- "Revoke" ends an active contract ("Are You Sure You Want to Revoke This Partnership?" → "Yes, Revoke it"). The affiliate's links for those products stop working at once, and shoppers follow the affiliate's Redirect Settings (AFA-05).

**When a contract is created**, the system:
- saves the commission for each product (a "snapshot"),
- gives the affiliate the seller's contact details,
- creates the affiliate's default product links,
- notifies the seller.

**Requests started on a brand page before sign-up** (AFB-04) are stored in the browser. They are sent to the seller automatically after the affiliate registers or logs in, or when the dashboard opens with `?confirm-temp-partnership-requests=true`.

## Rules and limits
- A request is **pending → approved / rejected / canceled**. Only the receiver can approve or reject; only the sender can cancel.
- A product can't have two open requests or two active contracts for the same affiliate.
- **Commission comes from the seller.** A seller changing a product's commission changes it on existing contracts too, and re-prices sales not yet billed (AFS-03).
- A contract ends (**revoked**) when:
  - the affiliate or seller revokes it;
  - its Ainfluencer affiliate campaign finishes (the next night at 02:00 UTC, AFX-02);
  - the affiliate's plan is downgraded or cancelled and they have more active contracts than the new plan allows. The lowest-selling contracts are revoked first (AFA-09).
- After a revoke the affiliate can apply again. Commission earned before the revoke is still billed and paid as normal (AFA-08).
- **Plan limits** ("Daily Applications", active contracts) are shown, but **not enforced** when applying in the current version (AFA-09).
- The "Auto System-Generated Partnership Requests" switch is **hidden** in production.
- Joining a campaign from the Ainfluencer mobile app approves the partnership automatically (AFX-02, MKT-03).
- **Talking to the seller** about a partnership: use the **Ainfluencer chat** (CHAT-01). The "Start Chat" / message buttons here open the unsupported affiliate chat.

## Statuses
| Tab / status | Internal value | Meaning | Next |
|---|---|---|---|
| Invitations | request `pending`, sent by seller | Seller invited the affiliate | Accept or Reject |
| Applications | request `pending`, sent by affiliate | Waiting for the seller | Seller decides; affiliate may cancel |
| Partnered | contract `active` | Links work and earn | Can be revoked |
| Revoked | contract `revoked` | Ended by someone or by the system | Apply again if wanted |
| Rejected | request `rejected` | Declined | Final for that request |
| (Canceled) | request `canceled` | Sender withdrew it | — |

## Timers and automatic actions
| When (UTC) | What |
|---|---|
| Instantly | Auto-approval when the brand/store has it on |
| Daily 08:00 | Email to anyone with pending requests waiting for them ("new partnership requests") |
| Daily 02:00 | Contracts of finished Ainfluencer campaigns are revoked |
| On plan downgrade or cancel | Extra contracts revoked (lowest sales first) |

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Request approved / rejected / canceled | The other party | In-app now, email in the 06:00/18:00 batch |
| Pending requests waiting | Receiver (e.g. affiliate with invitations) | Email daily 08:00 |
| Contract revoked by plan downgrade | Affiliate and seller | Email (batched) |
| Contract revoked by hand | — | **No notification** to the other side |
| Seller changes commission or deactivates a product | Partnered affiliates | In-app (some also email) |

## Common questions

### "I applied but the seller hasn't answered."
**Answer for the user:** The seller reviews requests themselves; there's no deadline. You can keep applying to other brands. If you'd like to follow up, contact the brand through the Ainfluencer chat.
**What to check:** The request is `pending` and when it was sent.
**Escalate when:** —

### "My links stopped working."
**Answer for the user:** The partnership has ended. Either the seller or you revoked it, the campaign it belonged to finished, or your plan changed. Your links now send shoppers to the place chosen in your Redirect Settings. You can apply again if the brand is still active.
**What to check:** The contract's status, `revoked_at` and `revoked_by`; whether it belonged to a finished campaign; recent plan changes.
**Escalate when:** The contract is still active but the link shows "removed".

### "I accidentally rejected an invitation."
**Answer for the user:** A rejection can't be undone, but you can send your own application to the same brand or product.
**What to check:** —
**Escalate when:** —

### "Will I still be paid for sales from before the partnership ended?"
**Answer for the user:** Yes. Commission earned while the partnership was active is billed and paid on the normal schedule (35+ days after the sale).
**What to check:** AFA-08.
**Escalate when:** —

### "I applied from the brand's page before I had an account. Where's my request?"
**Answer for the user:** It's sent automatically when you register or log in on the same browser. If you used a different browser or device, or closed the browser, please apply again from the app.
**What to check:** Whether a request exists for that brand. The saved request lives in a browser cookie that lasts until the browser is closed.
**Escalate when:** —

### "Can I ask for a higher commission?"
**Answer for the user:** For products that need seller approval you can propose a commission % when applying. The seller decides. Brands with automatic approval always use their default commission.
**What to check:** —
**Escalate when:** —

## Known issues
- Manual revokes send **no notification** to the other side.
- The "Go to Managment" button label has a typo.
- Plan limits are displayed but not enforced when applying (AFA-09). Downgrades still revoke extra contracts.
- A shopper-account session on the buyer site that applies gets "Access denied" unless that person also has an affiliate account (AFB-04).

## Related topics
- AFS-06 · Seller side of partnerships.
- AFA-03 · Where requests start.
- AFA-05 · Links created after approval and redirect behaviour.
- AFA-08 · Getting paid for contract sales.
- AFA-09 · Plan limits and downgrade effects.
- AFB-04 · Applying from a brand's public page.
- AFX-02 · Ainfluencer affiliate campaigns (auto-approval, end-of-campaign revoke).
- CHAT-01 · Ainfluencer chat.

## Technical reference
- **Endpoints:**
  - `POST affiliate/v1/partnerships/product/check-pre-request` (`ProductBulkRequestCheckService`), `POST affiliate/v1/partnerships/product/request` (`ProductPartnershipService` → `PartnershipRequestService::register`), `POST affiliate/v1/partnerships/brand/request` (`BrandPartnershipService`).
  - `PATCH affiliate/v1/partnerships/invited/{brand|product}/change-status/{id}`, `PATCH .../applied/{brand|product}/change-status/{id}`.
  - `PUT affiliate/v1/partnerships/contracts/{id}/revoke` (`Services/General/Partnership/Contracts::revoke` → `PartnershipContractRevokedEvent` → `RevokeProductLinkListener`, `RemovePrivatePromotionAccessListener`).
  - `GET affiliate/v1/partnerships/requests`, `.../contracts[/{id}]`, `.../sellers`.
  - `POST affiliate/v1/temp_partnership/brand/confirm-request` (`ConfirmPartnershipService`). Also triggered by register/login with `partnership_request_token` (`ConfirmPartnershipRequestJob`).
- **Route note:** the plan-limit middleware `affiliate_access_feature:*` is commented out on all these routes in `routes/affiliate/v1/partnership.php`.
- **Auto-approval:** listener `AutoPartnershipApproval` (brand/store `auto_partnership_approval = true` and no custom commission). On approval: `CreatePartnershipContract`, `CreateDefaultProductLink`, `CreateChangePartnershipNotification`.
- **Data:** `partnership_requests` (`status`, `type`, `affiliate_id`, `seller_id`, `message`), `partnership_request_items` (`commission`, `is_custom_commission`), `partnership_contracts` (`status` active/revoked, `revoked_at`, `revoked_by`), `partnership_contract_items`, `temp_partnership_requests` (`token`).
- **Jobs:** `email:new-partnership-requests` (08:00), `RevokePartnershipContractsForEndedCampaignsCommand` (02:00), downgrade `Plan/Downgrade/Affiliate/ActiveContractFeatureDowngrader`.
- **Frontend:** `spa-affiliate/src/components/partnership/**`, `discovery/products/merge/modals/generate-link/*`.
