---
id: CHAT-04
title: Contact-sharing detection
domain: Messaging
audience: [system, admin, brand, influencer]
apps: [web, mobile, email, background]
related: [CHAT-01, OFR-01, OFR-03, INF-04, BRD-04, SUB-01, SUB-04]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e, Mobile/mobile-app@468f655d (v32.2.2)
---

# CHAT-04 · Contact-sharing detection

> Brands and influencers must keep deals and payments on Ainfluencer. Before an offer is paid, contact details (emails, phone numbers, links, social handles, payment apps) are **hidden or blocked in the chat**, and every night an **AI check** flags offer chats where someone tried to share contacts, so the team can review and warn or suspend the account.

## Who and where
- **Brand**, web Inbox: when sending a message with contact details, a pop-up **"Unlock full sharing with &lt;name&gt;"** appears instead of sending.
- **Influencer**, mobile chat: when sending contact details, a sheet **"We kept your contact safe 💛"** explains the rule; such parts of messages show as **"Contact hidden · tap to learn why"**. The offer form (OFR-01) also refuses contact details in the pitch.
- **Admin**, admin panel: list of flagged offers to approve/disapprove, and a tool to send a violation notice.

## How it works
**1. Hiding and blocking in the apps (before payment)**
- Applies to **offer (collab) chats** before payment and to affiliate threads. It does **not** apply when the offer is paid/scheduled/published (or failed after payment), or when the brand is on the **Ultimate** plan ("unlimited chat": "Open messaging — contacts and links are visible"). Gifting chats are not masked.
- **Brand sends** a message containing an email, phone number, link, handle-like word, or apps such as WhatsApp/Telegram/PayPal → the message is **not sent**. The pop-up offers **"Get Ultimate"** ("Upgrade your plan to Ultimate") or **"Keep chatting"** ("Continue"), then **"Continue, a quick heads-up"**: "While you're on other plans, our AI moderator will block messages with any of the below. Repeat violations can pause your account." The brand ticks "I understand contact sharing stays locked…" and **"Confirm and continue chatting"** (or "Watch the 2-min explainer instead"). The brand then removes the contact details and sends again.
- **Influencer sends** contact details → the warning sheet appears ("Sharing contacts or moving off-app before funding isn't allowed and could risk your account."; button "I acknowledge this") and the contact part is shown hidden ("Contacts & links stay hidden until the offer is funded or the deal is closed.").
- Links to ainfluencer.com, brandid.app and YouTube are allowed.

**2. Nightly AI check** (03:30 UTC)
1. The system picks **unpaid offers** whose chat had new messages in the last 24 hours and that haven't been checked since.
2. It sends the recent message texts (Ainfluencer links removed) to an AI model that answers whether contact sharing happened and why.
3. If yes, the offer is **flagged "pending review"** with the AI's reason. Nothing is shown to users.
4. Admins review flagged offers and mark them approved or disapproved.

**3. Consequences (manual)**
- An admin can send a **violation notice** to the influencer or the brand on that offer: email "Violation Notice: Ainfluencer Guidelines Breach - Last Warning." and push/in-app "Possible Suspension | Ainfluencer Guidelines Infraction Notice." ("Your account is under review for violating our terms. Check your email for more details").
- Suspensions are decided by admins (INF-04 for influencers, BRD-04 for brands). Flags never suspend anyone automatically.

## Rules and limits
- Contact sharing is allowed after the offer is paid and with Ultimate-plan brands.
- The AI check covers **offer chats only** (not affiliate or gifting threads) and **only unpaid offers** created since March 2024.
- Each offer is re-checked only when new messages arrive after its last check.

## Statuses
| Flag status (admin) | Meaning |
|---|---|
| (none) | Not flagged |
| `pending` | AI found possible contact sharing; waiting for admin review |
| `approved` / `disapproved` | Reviewed by an admin |

## Timers and automatic actions
| Job | When (UTC) | What |
|---|---|---|
| `offers:flag` | Daily 03:30 | AI contact-sharing check of recent unpaid offer chats (one offer every 5 seconds) |

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Admin sends violation notice | Influencer or brand | Email, push, in-app |
| AI flag | Nobody (admin list only) | – |

## Common questions

### "Why can't I send my email/WhatsApp to the influencer?"
**Answer for the user:** To protect both sides, contact details can only be shared after the offer is paid, or if you're on the Ultimate plan. Please keep the conversation and payment on Ainfluencer; your payment is held in escrow until delivery.
**What to check:** brand plan (SUB-01), offer status.
**Escalate when:** the offer is paid (or brand is Ultimate) and messages are still blocked.

### "My normal message was blocked but it has no contact details."
**Answer for the user:** Our filter looks for things like numbers, links and words such as "email" or "WhatsApp". Try rephrasing without numbers or those words.
**What to check:** the text (phone-like number patterns, app names, the word "username").
**Escalate when:** plain text without numbers or keywords is blocked; collect the exact text for dev.

### "The influencer's message says 'Contact hidden'."
**Answer for the user:** The influencer tried to share contact details, which stay hidden until the offer is paid. Pay the offer to continue securely, or ask them to describe what they need in words.
**What to check:** offer status.
**Escalate when:** the offer is paid and contacts are still hidden.

### "I got a 'Possible Suspension' notice. What did I do?"
**Answer for the user:** Our team found that contact details or off-platform payment were discussed in one of your chats, which breaks our guidelines. Please keep all communication and payments on Ainfluencer; repeated violations can lead to suspension.
**What to check:** the violation record (offer, templates used) and the offer's flag reason.
**Escalate when:** the user disputes it (admin team reviews the chat).

### "Can I share contacts in a gifting chat?"
**Answer for the user:** Gifting chats aren't hidden by the app, but moving deals or payments off Ainfluencer is still against our guidelines.
**Escalate when:** never.

## Known issues
- The in-app filters are simple text patterns: they can block harmless messages that contain numbers or words like "contact" or "email", and can miss creative spellings. The nightly AI check is the backstop.
- The AI check doesn't look at affiliate or gifting threads.
- The brand pop-up mentions "Pay $1 and unlock" style options in some copy; the one-off unlock is "Coming soon" and not available *(to verify whether it is shown)*.

## Related topics
- CHAT-01 chat rules; OFR-01 contact details blocked in the pitch; OFR-03 paying unlocks sharing.
- INF-04 influencer suspension and appeal; BRD-04 brand suspension; SUB-01/SUB-04 Ultimate plan.

## Technical reference
- **Nightly check:** `offers:flag --from=2024-03-03` (`app/Console/Commands/Offer/FlagOffer.php`, cron `30 3 * * *` in `app/Console/Kernel.php`) → `app/Jobs/Offer/FlagJob.php` (reads chat history from the chat service, last 24 h) → `app/Modules/OpenAI/Services/Integrations/Offer/Flag.php` (up to 5 calls; parses "ContactSharing= Yes/No" and "Summary:"). Writes `offers.extra.flag` (`status`, `ai_reason`, `flagged_at`, `last_flag_checked_at`, `reviewed_at`); `offers.extra.flag_checking` while busy.
- **Admin review:** `app/Http/Controllers/v2/Admin/OfferController` (`updateFlagStatus`, bulk) → `app/Services/Implementation/Admin/OfferService::flaggedOffers/updateFlagStatus`.
- **Violation notice:** `app/Services/Implementation/Admin/ViolationService::store()` → `ViolationStoreEvent` → `app/Listeners/v2/NewViolationListener.php` → `app/Modules/Mobile/Notifications/Account/ViolationNotification.php`; table `violations` (`offer_id`, `model_type`, `model_id`).
- **Web filter:** `src/lib/utils/tools.ts::checkPreventMessage` (skipped for after-payment statuses in `newSingleOfferV2/parts/newChat/model.ts::afterPaymentList`, for "Ultimate", and for seeding threads), pop-up `newSingleOfferV2/parts/newChat/parts/sender/parts/contactSharingPopover`, copy in `src/lib/localization/locales/en/singleOffer.json` (`unlockGate*`, `freeAck*`).
- **Mobile filter:** `src/utils/regularExpression.ts` (`sanitizeMessage`, `secureText`), `src/features/chat/utils/chatSecureRestrictions.ts` (statuses `waiting_your_approval`, `waiting_payment`, `rejected_by_brand`, `rejected_by_you`; Ultimate plan exempt; gifting masking off), `components/WarningPaymentModal`, copy in `src/localization/translations/en/chat.js`.
