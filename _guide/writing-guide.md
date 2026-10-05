# Writing guide for topic documents

Every topic in [../flow-map.md](../flow-map.md) gets one document. This guide is the contract for those documents, so they read the same and an AI assistant can rely on them.

## Who reads these documents

1. **Support agents** who don't know the code or the flows. They need to understand what the user is experiencing, explain it in plain words, and know what to check or escalate.
2. **The support assistant** (Claude connected to this repository). It answers support agents' questions from these documents only, so anything not written here effectively doesn't exist for it.

Write for a smart support agent on their first week: plain English, no code, no jargon without a definition. Technical details go only in the last section ("Technical reference").

## File name and location

`topics/<domain-folder>/<ID>-<short-slug>.md`, for example `topics/offers/OFR-03-paying-for-an-offer.md`.

| Prefix | Folder |
|---|---|
| ACC | `topics/account` |
| BRD | `topics/brand-account` |
| INF | `topics/influencer-accounts` |
| DSC | `topics/discovery` |
| CMP | `topics/campaigns` |
| MKT | `topics/marketplace` |
| OFR | `topics/offers` |
| PUB | `topics/content-delivery` |
| PAY | `topics/money` |
| SUB | `topics/subscriptions` |
| CHAT | `topics/chat` |
| NTF | `topics/notifications` |
| INT | `topics/integrations` |
| PUBL | `topics/public-pages` |
| AFX | `topics/affiliate-connection` |
| AFS | `topics/affiliate-seller` |
| AFA | `topics/affiliate-creator` |
| AFC | `topics/affiliate-shared` |
| AFB | `topics/affiliate-buyer` |

## Template

Use these sections in this order. Leave out a section only if it truly doesn't apply (for example, a topic with no statuses). Don't add other top-level sections.

```markdown
---
id: OFR-03
title: Paying for an offer
domain: Offers and negotiation
audience: [brand]            # brand, influencer, seller, affiliate, buyer, admin, system
apps: [web]                  # web, mobile, seller-app, affiliate-app, buyer-site, email, background
related: [OFR-02, OFR-04, PUB-01, PAY-01]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# OFR-03 · Paying for an offer

> One or two sentences: what this is and when it happens.

## Who and where
Who does it, in which app, on which screen (exact UI names and URLs/screen names), and how they get there.

## How it works
Numbered steps from the user's point of view. Quote button and screen labels exactly as the UI shows them, in "double quotes". Say what the system does at each step in plain words (for example "the money is held by Ainfluencer and not yet given to the influencer").

## Rules and limits
Bullets: minimums, maximums, plan limits, deadlines, who is allowed to do what, what blocks the action.

## Statuses
A table: status as the user sees it → what it means → what happens next. Include the internal value in `code` only if support may see it in the admin panel.

## Timers and automatic actions
What the system does by itself, when (UTC), and what the user notices.

## Notifications
Table: what triggers it → who gets it → channel (email, push, in-app, SMS).

## Common questions
Real questions a user would ask support, each as:

### "Why can't I pay? The button is disabled."
**Answer for the user:** short, friendly explanation support can paste or adapt.
**What to check:** the concrete things support (or the assistant with data access) should look at, in order.
**Escalate when:** the condition that means it's a bug or needs the dev/finance team.

Write 4–10 of these. Prefer questions that come from the rules, timers and known issues above.

## Known issues
Bugs and gaps that exist in production today and that a user may hit. For each: what the user sees, the workaround if any, and "Escalate to dev" if there's none. Mark unconfirmed items with *(to verify)*. Never describe security weaknesses here (see below).

## Related topics
Links to other topic documents by ID with one line each.

## Technical reference
For the assistant and for escalation, not for users:
- Main endpoints (method + path) and the controller/service that handles them.
- Data: the main tables/models and the fields that tell you the state (for example `offers.status`, `publish_jobs.url_approval`). These will be used for read-only lookups later.
- Scheduled jobs and webhooks involved.
- Key source files (paths relative to the repo).
```

## Rules

1. **Only production behaviour.** Describe what the code in the snapshots does today, not what a comment, a doc, or a variable name suggests. If code and an older document disagree, the code wins.
2. **Verify every claim in the code.** The map and the evidence files are a starting point; open the source and confirm before you write. If you can't confirm something, mark it *(to verify)*.
3. **Only live code.** Don't document dead pages, unused endpoints or features that are switched off, except to say plainly "this is not available" when users are likely to ask about it.
4. **Plain words first, exact labels second.** Explain the concept, then give the UI label in quotes so support can find it on screen.
5. **Be concrete.** Numbers, durations, time zones (UTC), limits, exact statuses. "After a while" is not acceptable.
6. **No security weaknesses.** This repository is shared with the support team and stored on GitHub. Never describe vulnerabilities, missing authentication, secrets, keys, tokens, passwords, or how to bypass checks. If you find one, leave it out of the document and report it separately to the person running the work.
7. **No personal data.** No real user names, emails, IDs or production data in examples. Use placeholders like `brand@example.com`.
8. **Money and status claims need extra care.** If a user could lose or be charged money because of what the document says, double-check it in the code and say exactly where the rule comes from in the Technical reference.
9. **Messaging policy.** The affiliate platform's own chat (AFC-01) is not a supported channel. Wherever messaging between a brand/seller and an influencer/affiliate comes up, tell support to direct users to the **Ainfluencer chat** (CHAT-01).
10. **Length.** Usually 600–1,500 words. Long is fine if every line helps support answer a question; cut anything that doesn't.
11. **Cross-link, don't repeat.** If another topic owns a rule (for example the plan limits in SUB-04), summarise it in one line and link to it.
12. **Keep the IDs stable.** Never rename or renumber a topic. If a topic needs splitting, ask first.

## Where the facts come from

| Area | Code (production snapshot) | Evidence file |
|---|---|---|
| Brand web app | `core/frontend/web` at `master` 588fe862e | `_map-evidence/web.md` |
| Core API | `core/backend/api` at `master` 99921fbce | `_map-evidence/api-endpoints.md`, `_map-evidence/backend.md` |
| Chat services | `core/backend/chatbox/*` at `master` | `_map-evidence/chat.md` |
| Influencer app | `Mobile/mobile-app` at `develop` 468f655d (v32.2.2) | `_map-evidence/mobile.md` |
| Affiliate backend | `affiliate/ainflu-affiliate-backend` at `v1.5.27` | `_map-evidence/aff-backend.md`, `_map-evidence/aff-api-endpoints.md` |
| Seller and affiliate apps | `affiliate/ainflu-affiliate-frontend` at `v1.4.02` | `_map-evidence/aff-seller.md`, `_map-evidence/aff-affiliate.md` |
| Buyer site | `affiliate/ainflu-affiliate-buyer` at `v1.3.04` | `_map-evidence/aff-buyer.md` |
