---
id: NTF-04
title: Digest and reminder emails
domain: Notifications and emails
audience: [brand, influencer, system]
apps: [email, background]
related: [NTF-03, CMP-01, CMP-04, CMP-05, OFR-01, OFR-03, DSC-03, CHAT-02, PUB-05, BRD-02, AFX-02]
last_verified: 2026-10-05
source: core/backend/api@99921fbce
---

# NTF-04 · Digest and reminder emails

> Emails Ainfluencer sends on a schedule or after a delay, not as a direct reaction to a click: the brand daily digest, three weekly digests, reminders about unfinished campaigns, and reminders about offers waiting for payment. All times are UTC.

## Who and where

Email only. Brands and influencers receive them at their account email. Links in the emails open the web app (brands) or the app/website (influencers).

## How it works

| Email (subject) | Who | When |
|---|---|---|
| **Brand daily digest**: "{name} was busy on your campaigns today", "{n} creators took action on your campaigns today", or "Your daily campaign digest" | Brands with new creator activity | Daily 08:00 |
| **Campaigns for you**: "Daily new campaigns for you to consider to send offers to \| Ainfluencer" | Every influencer account (one email per person) | Weekly, Monday 08:00 |
| **Influencers for your campaign**: "A list of most relevant influencers for your product to go viral- Ainfluencer Marketplace" | Brands, one per live (active and approved) campaign | Weekly, Tuesday 08:00 |
| **Suggestions for brands without a campaign** (same subject as above) | Brands that have no campaign | Weekly, Wednesday 08:00 |
| **Unfinished campaign**: "Complete your Campaign to Get Published in Ainfluencer Marketplace- Reach Millions of People" | Brand | 24 h and 72 h after a paid-collaboration campaign draft is created, only if it's still a draft |
| **Offer awaiting payment**: "Your influencer Offer for {campaign} is awaiting your payment to get started." | Brand | 1 day and 3 days after an influencer sends an offer, only if the offer is still waiting for payment |
| **Abandoned checkout**: first reminder, then "Seems like you left your Payment behind!" | Brand | 3 h and 24 h after the brand opens the checkout for an offer and doesn't pay, only if still unpaid |

**Brand daily digest** contents: affiliate activity (new tracked links, new partners, join requests, top performers by clicks or sales over the last 7 report days) from the affiliate platform, and gifting activity (new requests, accepted invitations, products received). Each creator's action is reported **once**; a creator who already appeared for the same kind of action isn't repeated (top performers can repeat with fresh numbers). Creators the brand hasn't unlocked show only a first name and a blurred handle. Each creator has an "Open chat" link. Brands with no new activity get no email that day.

Related emails documented elsewhere: grouped campaign invitations twice a day (DSC-03), the hourly "unread messages" email (CHAT-02), review reminders after a deal (PUB-05).

## Rules and limits

- Digests and marketing emails go through a separate mail provider and a separate queue from transactional emails, so a big digest run never delays login codes or payment emails.
- Deleted or archived accounts receive none of these.
- Users who clicked "Unsubscribe" in an email don't receive them (NTF-03).
- Weekly digests can be turned off with "Marketing Updates" → "Email" *(category link to verify in production)*. The brand daily digest has its own category "Daily Digest", which brands **can't see on the web settings page** today (NTF-03).
- Reminders check the current state when they are due: a campaign that's no longer a draft, or an offer that's paid or rejected, gets no reminder.
- Gifting (product seeding) drafts don't get the "Complete your Campaign" reminder.
- Abandoned-checkout **SMS** reminders are switched off; only the two emails are sent.

## Statuses

Not applicable.

## Timers and automatic actions

| Job | When (UTC) | Effect |
|---|---|---|
| `brand-digest:send` | Daily 08:00 (configurable) | Queues the brand daily digest for brands with activity since the last run |
| `digest-emails:influencer-campaigns` | Monday 08:00 | Weekly "campaigns for you" to influencers |
| `digest-emails:brand-influencers` | Tuesday 08:00 | Weekly influencer suggestions per live campaign |
| `digest-emails:brand-without-campaign` | Wednesday 08:00 | Weekly suggestions to brands without a campaign |
| Delayed draft reminder | +24 h, +72 h after draft creation | "Complete your Campaign..." |
| Delayed payment reminders | +1 day, +3 days after the offer | "...awaiting your payment..." |
| Delayed abandoned-checkout emails | +3 h, +24 h after opening checkout | Two reminder emails |

Large runs are sent gradually, so a digest may arrive later than 08:00.

## Notifications

All rows in the tables above are email only.

## Common questions

### "Why do I get an email every day about my campaigns?"
**Answer for the user:** That's your daily digest. It tells you which creators joined, made links, requested gifts or received products since yesterday. You only get it on days with new activity.
**What to check:** The brand has affiliate or gifting activity.
**Escalate when:** The brand wants it turned off (the "Daily Digest" switch isn't on the web settings page; dev or staff must change it), or the digest lists wrong data.

### "How do I stop the weekly influencer/campaign suggestion emails?"
**Answer for the user:** Turn off "Email" for "Marketing Updates" in "Notifications & Emails" and save, or click "Unsubscribe" at the bottom of the email (this stops most emails).
**What to check:** Saved settings; whether the user really wants to unsubscribe from everything.
**Escalate when:** Emails continue after switching off.

### "I keep getting 'awaiting your payment' emails but I don't want that offer."
**Answer for the user:** These reminders stop as soon as the offer is paid or rejected. If you don't want to work with this influencer, reject the offer in your inbox.
**What to check:** Offer status (`offers.status` = `awaiting-payment`).
**Escalate when:** Reminders arrive for an offer that's already paid or rejected.

### "I got 'Complete your Campaign' but I already published it."
**Answer for the user:** The reminder is only sent if the campaign is still a draft at that moment. If you published it in the meantime, you can ignore the email.
**What to check:** Campaign status at the time of the email (CMP-01).
**Escalate when:** The campaign was clearly submitted before the email time.

### "The weekly email says 'Daily new campaigns' but I get it once a week." (influencer)
**Answer for the user:** It's a weekly summary of new campaigns (sent on Mondays), despite the word "daily" in the subject.
**What to check:** Nothing.
**Escalate when:** Never.

### "The digest shows a creator's name blurred."
**Answer for the user:** Creator details are hidden until you unlock them (by revealing the username or paying an offer with them).
**What to check:** DSC-01 (username reveal).
**Escalate when:** Never.

## Known issues

- **Brands can't switch off the daily digest** from the web settings page (category not displayed). Escalate to dev/staff.
- The influencer weekly digest subject says "Daily" although it's weekly.
- The Tuesday and Wednesday brand digests use the same subject line, so brands may think they received a duplicate.

## Related topics

- NTF-03: switching emails off; unsubscribe.
- CMP-01: draft campaigns. CMP-04, CMP-05, AFX-02: affiliate and gifting activity in the daily digest.
- OFR-01, OFR-03: offers waiting for payment and checkout.
- DSC-03: grouped invitation emails. CHAT-02: unread-message email. PUB-05: review reminders.
- BRD-02: the brand score orders campaigns in the influencer weekly digest.

## Technical reference

- **Brand daily digest**: `app/Modules/BrandDigest` (README; command `SendBrandDailyDigestCommand` = `brand-digest:send`; `Jobs/SendBrandDigestBatchJob` (200 brands per batch); `Notifications/BrandDailyDigestNotification`; config `Config/brand_digest.php` (`BRAND_DIGEST_SEND_AT`, default 08:00); copy `resources/lang/en/brand_digest.php`; views `resources/views/emails/v3/brand_digest`). Replaces the unscheduled `GiftingRequestsDigestCommand`.
- **Weekly digests**: `app/Console/Commands/DigestEmails/{InfluencerCampaigns,BrandInfluencers,BrandWithoutCampaign}.php` → `Services/DigestEmails/*` → `Notifications/v2/Digest/Influencer/CampaignNotification`, `Brand/InfluencerNotification`, `Brand/BrandWithoutCampaignNotification` (Mailgun). Influencer campaigns ordered by `brand_details.score` (`Services/DigestEmails/Influencer/Queries.php`).
- **Draft reminders**: `Observers/v2/V2AdObserver::created` → `Jobs/Ad/AdDraftedJob` (+24 h, +72 h; skipped unless `ads.status = draft`; skipped for product seeding) → `Notifications/AdDrafted`.
- **Awaiting payment**: `Mobile/.../OfferCreatorService::schedulePaymentReminders` → `Notifications/Reminders/AwaitingPaymentFirstReminder` / `SecondReminder` (delays `Constants::AWAITING_PAYMENT_*_REMINDER_DELAY` = 1 and 3 days; sent only while `offers.status = awaiting-payment`).
- **Abandoned checkout**: `Services/Implementation/v3/PaymentTrackingService::abandonedTracking` → `Notifications/AbandonedCarts/FirstAbandonedNotification` (+3 h), `SecondAbandonedNotification` (+24 h); SMS calls commented out; de-duplicated with `EmailTracker`.
- **Schedule**: `app/Console/Kernel.php`.
