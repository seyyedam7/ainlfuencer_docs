---
id: PAY-04
title: Payout methods and withdrawals
domain: Money (wallets, refunds, payouts)
audience: [influencer, admin]
apps: [mobile, email]
related: [PAY-03, PUB-03, OFR-03, PAY-05]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, Mobile/mobile-app@468f655d (v32.2.2)
---

# PAY-04 · Payout methods and withdrawals

> An influencer adds a payout method (PayPal or a crypto wallet), picks completed deals and sends a withdrawal request. **Ainfluencer staff pay every request by hand**; nothing is sent automatically. The request then shows as Completed (with a transaction hash for crypto) or Rejected.

## Who and where
- **Influencer** in the mobile app: WALLET_SCREEN → **"Wallet"** tab (payout method + "Submit Withdrawal Request") and **"Payouts"** tab (request history). See PAY-03 for the rest of the wallet.
- **Admin** (finance team) in the admin panel: payout list, per-influencer details, "pay" and "reject".

## How it works
**1. Add a payout method** (section "Setup Your Payout Method", later "Choose Your Payout Method")
- **PayPal**: tap "Add PayPal Account", enter the PayPal email, tap "Link PayPal Account". The option then shows "PayPal" with a "Connected" badge and an edit pencil.
- **Crypto**: tap "Add Crypto Wallet", choose a **currency** and a **network**, paste the wallet address ("{currency} address on {network}"), tap "Save Crypto Wallet". The option then shows "Crypto · USDT" (or the chosen coin).
  - Choices come from the server, so they can change without an app update. As configured at launch: **USDT** and **USDC**, each on **BNB Smart Chain (BEP20)**, **Ethereum (ERC20)**, **Solana (SOL)**, **Tron (TRC20)** and **BNB (BEP20)**.
  - The address is checked against the network's format ("The wallet address is not a valid {network} address.").
- One PayPal and one crypto method per influencer. Saving again **replaces** the old one. A request already sent keeps the address it was sent with.
- Stripe payouts are switched off in the app.

**2. Request a withdrawal**
1. Select the method with its radio button.
2. Tap **"Submit Withdrawal Request"** (grey if no method is selected or nothing is available to withdraw).
3. The sheet **"Select Completed Deals"** lists deals whose money has been released and that aren't already requested, plus a **"Legacy balance"** row if there is money from the old system. Text: "Funds get released on 1st and 15th of each month if they are already approved."
4. Select deals. The sheet shows **"Offer Amount"**, **"Ainfluencer Fee (X%)"** and **"You Will Receive"**.
5. Tap **"Request Withdrawal ($X)"**. The confirmation "Withdrawal requested" says "We'll send $X to your PayPal / USDT wallet", with the steps "Request sent", "Ainfluencer review" ("Our team checks the deal and approves your request.") and "Sent to your …" ("Released on the 1st and 15th. Takes up to 5 business days…"), the "Next payout" window, and "View in Payouts".
6. One request record is created **per deal** (plus one for the legacy balance). The payout admins get an email with the total.

**3. Ainfluencer pays (admin)**
- The admin sends the money outside the platform (PayPal or crypto) to the address saved **on the request**, then marks it paid, recording how and where it was paid and, for crypto, the **transaction hash**.
- On "paid": the request becomes **Completed**, the full deal amount leaves the influencer's wallet, the deal shows "Payout Made", and the influencer gets an in-app notification. The Payouts tab shows where it was sent and the transaction hash (tap to copy).
- Or the admin **rejects** with a reason: the request becomes **Rejected**, the reason is shown under "Details:", the money stays in the wallet and the deal goes back to the withdrawable list so the influencer can request again. An admin can still pay a rejected request later.
- When paying, the admin can choose to use the payout to settle a negative credit balance (a debt) on the influencer's account; the influencer then receives less.

## Rules and limits
- Only deals with status "Payment Released" (`completed`) can be withdrawn; each deal can be in only one open or completed request.
- The total can't exceed the released balance minus requests already in progress: "Withdrawal request exceeds available balance. Pending payouts: $X, Total balance: $Y."
- The amount per deal is the deal's **current price** (it includes revisions the influencer accepted).
- **Fee:** the percentage set for that payout method by Ainfluencer (normally 20%), frozen on the request when it is sent. The legacy balance has no fee.
- **No minimum** amount is checked.
- The influencer **can't cancel** a request from the app (there is no cancel button).
- If the payout gateway is switched off, the request fails with "Payment gateway unavailable…". If a crypto currency/network was switched off after saving: "This currency and network are not available for payouts. Please update your crypto payout method."
- "1st and 15th" and "up to 5 business days" are the finance team's schedule shown in the app; the system doesn't enforce any date.

## Statuses
| Payouts tab | Internal (`payment_receipts.status`) | Meaning / next |
|---|---|---|
| "Processing" | `WITHDRAW_IN_PROGRESS` | Waiting for the finance team |
| "Completed" | `WITHDRAW_COMPLETED` | Paid; `transaction_hash` set for crypto; deal → `settled` |
| "Rejected" | `WITHDRAW_REJECTED` | Reason in "Details:"; deal can be requested again |
| "Canceled" | `WITHDRAW_CANCELED` | Cancelled (not possible from the current app) |
| "Pending" | any other value | Fallback label |

Legacy balance: `legacy_payouts.status` `pending` → `in_progress` (requested) → `settled` (paid); back to `pending` if rejected.

## Timers and automatic actions
None. There is no scheduled payout run; every payout is a manual admin action.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Withdrawal requested | Payout admins (configured list) | Email (net amount, influencer email, date) |
| Payout marked paid | Influencer | In-app only (no email or push) |
| Offer paid and no PayPal payout method saved ("Connect Your PayPal Account for Payouts…") | Influencer | Email |

## Common questions

### "I requested a withdrawal. When will I get paid?"
**Answer for the user:** Our finance team reviews and pays withdrawal requests by hand, usually around the 1st and 15th of each month. After it's sent, it can take up to 5 business days to arrive. You'll see "Completed" in the Payouts tab when it's paid.
**What to check:** `payment_receipts` (user, `is_income = 0`): `status`, `created_at`, `payout_destination`.
**Escalate when:** a request is still "Processing" after the next 1st/15th plus 5 business days → finance team.

### "My payout says Completed but I didn't receive it."
**Answer for the user:** Please check the PayPal email / wallet address shown on the payout in the Payouts tab. For crypto, you can copy the transaction hash and look it up on the network's block explorer. If everything matches and it still hasn't arrived, we'll check with our finance team.
**What to check:** `payout_destination` (gateway, value, currency, network), `paid_via`, `paid_to`, `paid_at`, `transaction_hash`.
**Escalate when:** destination is correct and nothing arrived, or the destination differs from what the influencer saved → finance team.

### "I entered the wrong PayPal email / wallet address."
**Answer for the user:** Edit your payout method in the Wallet tab (pencil icon or the crypto option). Requests you already sent keep the old address, so contact us right away so we can reject the old request and you can send a new one.
**What to check:** open requests in `WITHDRAW_IN_PROGRESS` and their `payout_destination`.
**Escalate when:** an open request points to a wrong address → finance team to reject it before paying.

### "Why was my withdrawal rejected?"
**Answer for the user:** The reason is shown under "Details" on the payout. The money is still in your wallet and the deal is back in your list, so you can send a new request once the issue is fixed.
**What to check:** the request's description (reason) and `payment_receipt_logs` (`payout:reject`).
**Escalate when:** no reason is shown, or the influencer disputes it.

### "The 'Submit Withdrawal Request' button is grey."
**Answer for the user:** Select a payout method first (tap PayPal or Crypto so the circle is filled). The button also stays grey when you have nothing available to withdraw, for example if all released deals are already in a request.
**What to check:** payout methods (`wallet_settings` key `payout`), available to withdraw, open requests.
**Escalate when:** a method is selected, deals show "Payment Released" and nothing is in progress.

### "Can I cancel my withdrawal request?"
**Answer for the user:** There's no cancel button in the app. Tell us which request and we'll ask the finance team to reject it, so the money becomes available again.
**What to check:** request is still `WITHDRAW_IN_PROGRESS`.
**Escalate when:** always (finance team action).

### "I added a crypto wallet but I keep getting emails to connect PayPal."
**Answer for the user:** You can ignore that email. Your crypto wallet is saved and can be used for withdrawals.
**What to check:** crypto method exists (`wallet_settings.provider = nowpayments`, key `payout`).
**Escalate when:** n/a.

### "Can I be paid by bank transfer or Stripe?"
**Answer for the user:** At the moment payouts are available to PayPal or a crypto wallet (USDT/USDC).
**What to check:** n/a.
**Escalate when:** the influencer can use neither → finance team for a manual arrangement.

## Known issues
- The "Connect Your PayPal Account" email is sent on every paid offer to influencers without PayPal, even if they have a crypto method.
- Influencers can't cancel a request in the app; the finance team must reject it.
- Admins are told the net amount by email, but the influencer gets only an in-app notification when paid.

## Related topics
- PAY-03: influencer wallet and fee explanation.
- PUB-03: when deal money becomes withdrawable.
- PAY-05: staff credits and negative balances.

## Technical reference
- **Influencer endpoints** (`/api/{locale}/finance/mobile/...`, middleware `auth:api2` + `influencer`):
  - `GET payout-methods`, `POST payout-methods` (PayoutMethodController; `gateway` paypal | nowpayments; crypto needs `currency`, `network`; one row per provider, `updateOrCreate`), `GET payout-methods/crypto-options`.
  - `GET orders/payable` (PayoutController@payableOrderList; extra `availableToWithdraw`, `pendingPayout`, `legacyPayout`).
  - `POST payouts` (PayoutController@store: `orderItemIds[]` incl. `"LegacyPayout"`, `walletSettingId`), `GET payouts` (history). `DELETE payouts/{id}` exists but the app doesn't call it.
- **Admin endpoints:** `GET finance/admin/payouts`, `GET …/details/{userId}`, `POST …/pay` (`payment_receipt_ids`, `paid_via`, `paid_to`, `paid_at`, `transaction_hash`, `useCredit`), `POST …/reject` (`payment_receipt_id`, `reason`) — `app/Modules/Finance/Http/Controllers/Admin/AdminPayoutController.php`.
- **Pay logic:** fee = amount × `payment_receipts.fee`%; full amount moved influencer default → system default; `OrderService::settleOrderItem` (deal and its revisions → `settled`); `WithdrawPaidByAdminNotification` (database only, event `influencer.withdraw:paidByAdmin`).
- **Data:** `payment_receipts` (`status`, `amount`, `fee`, `creator_type` OrderItem | LegacyPayout, `creator_id`, `payout_destination` JSON, `transaction_hash`, `paid_at`, `paid_via`, `paid_to`, `description`), `payment_receipt_logs`, `wallet_settings` (`key = payout`, `provider`, `value`, `extra.currency|network`), `crypto_currencies`, `crypto_networks` (`address_regex`), `crypto_currency_network` (`is_active`), `legacy_payouts`, `gateways.fee_per_transaction`.
- **Events/listeners:** `PayoutRequestedEvent` → `SendPayoutEmailToAdminsListener` (`Mail/AdminPayoutEmail`, recipients `PAYOUT_ADMIN_EMAILS`); `OfferPaidListener::notifyInfluencer` → `Mail/v2/ConnectPaypalEmail` when no PayPal payout method.
- **Docs/files:** `app/Modules/Finance/crypto-payout.md`, `Services/CryptoPayoutService.php`, migrations `2026_10_01_00000*`; mobile `src/features/new-wallet/components/{PayoutMethod,CryptoPayoutOption,AddCryptoWalletModal,AddPaypalAccountModal,WithdrawalModal,WithdrawalSubmittedModal}.tsx`, `PayoutsList/PayoutItem`.
