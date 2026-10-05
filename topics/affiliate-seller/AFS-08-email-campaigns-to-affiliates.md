---
id: AFS-08
title: Email campaigns to affiliates
domain: Seller app (brands)
audience: [seller, admin]
apps: [seller-app]
related: [AFS-05, AFS-12, AFS-13, AFC-01, AFC-04]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFS-08 · Email campaigns to affiliates

> Sellers can write an email to a selection of affiliates (from their lists or the "Expanded Network") and send it through **their own email server (SMTP)**. Each campaign must be approved by the Ainfluencer team. **Important:** in the current production version, approved campaigns are **not actually sent** — the automatic sending step is switched off.

## Who and where
- **Seller** in the seller app: sidebar **More → Email Campaigns** (`/app/email-campaigns`), also "Send Email" on a list card (opens with that list pre-selected).
  - "Send" tab: a 3-step wizard. "History" tab: past campaigns.
  - Team members need the email-campaign permissions (view / create / cancel) in their role (AFS-13). Without "create", only History is shown.
- **Admin**: approves or rejects campaigns in the admin panel (Requests).

## How it works
1. **"Select an SMTP for This Campaign".** Pick one of "Your SMTP Connections" or click "Add New SMTP" and fill in: Name ("visible only to you"), "SMTP Host", "SMTP Port", "Username", "From Email", "Password / App Password", "Encryption Type" (TLS, SSL or None). A connection can be **tested**: the system sends "SMTP Test Email" to the seller's own login email and shows "A test email was sent successfully to your address …".
2. **"Select Recipients".** Choose "Ainfluencer Affiliates" or "Expanded Network", then "My Affiliate Lists" or revealed affiliates. Only affiliates whose email is **revealed** can receive the email. If some aren't, the pop-up "Some Affiliates Will Miss This Email!" offers to reveal them ("Use your daily reveal quota", then the wallet, AFS-05).
3. **"Write & Finalize Your Message".** "Subject" and "Message Body" are required; "Custom Values" (personalised fields) and "Message Templates" can be inserted. There's a "Schedule Email" switch with a date/time.
4. **Submit.** The screen "Campaigns Email Submitted" says it "will be sent after approval by the Ainfluencer team at the selected time… You will receive a notification once it has been reviewed." The campaign is saved as waiting for approval.
5. **Approval.** An admin accepts or rejects it in the admin panel.
6. **Sending.** Meant to happen automatically after approval, through the seller's SMTP server. **This step is not running in production** (see Known issues), so nothing is sent.
7. **History.** Shows "Email Subject", "Status", "Submitted on", "Reviewed on", "Sent on", "SMTP Name", "# of Recipients". While waiting for approval the row has **"Cancel"** ("Cancel Email?"). After it's accepted, rejected or cancelled, the row has **"Resend"**, which creates a copy waiting for approval again.

## Rules and limits
- **Plan:** creating a campaign needs the plan feature "email campaign"; otherwise the "Upgrade Needed" pop-up appears (AFS-12). SMTP connections can be added without it.
- **Recipients:** 1–100 recipient entries per campaign (a list counts as one entry). Only revealed emails are used.
- **Subject:** required, up to 255 characters. Body required.
- **Cancel:** only while waiting for approval.
- **Resend:** only after accepted, rejected or cancelled; it needs a new approval.
- **Schedule:** the date/time chosen in the form is **not saved** (the app never sends it).
- Emails go out from the seller's own server and "From Email"; delivery problems (spam, bounces, wrong password) are on the seller's email provider.
- Admin bulk emails to affiliates are a different feature and are also not sent automatically.

## Statuses
Two values exist; the History "Status" badge shows only the **sending** status.

| Sending status (badge) | Meaning |
|---|---|
| "Pending" (`pending`) | Not sent yet. Today every campaign stays here. |
| "Running" / "Completed" (`running`, `completed`) | Sending / sent. Not reached in production. |
| "Cancelled" (`cancelled`) | Not used by the seller's Cancel button (see below). |

| Approval status (`approval_status`, not shown as a badge) | What the seller sees |
|---|---|
| `waiting_for_approval` | "Cancel" button visible, "Reviewed on" empty. |
| `accepted` | "Resend" button, "Reviewed on" filled. |
| `rejected` | "Resend" button, "Reviewed on" filled (reason stored for admins). |
| `cancelled` (seller cancelled) | "Resend" button; badge still says "Pending". |

## Timers and automatic actions
- The scheduled job that sends approved campaigns (`email-campaign:process-notification`) is **commented out** in the production scheduler. No other automatic action.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| SMTP test | Seller's own login email | Email via the seller's SMTP |
| Admin accepts/rejects a campaign | — | **None** (the success screen promises a notification, but none is sent) |

## Common questions

### "My campaign was approved but no affiliate received it."
**Answer for the user:** Thanks for flagging this. Sending of approved email campaigns is currently paused on our side, so the email hasn't gone out yet. We've passed it to the team. Meanwhile you can contact affiliates through the Ainfluencer chat.
**What to check:** `email_campaigns.approval_status` = accepted (1), `status` = pending (0), `started_at` empty.
**Escalate when:** always (dev must decide whether to run the sending job).

### "My campaign still says Pending."
**Answer for the user:** "Pending" means it hasn't been sent. If "Reviewed on" is empty it's still waiting for review; if it's filled, it has been reviewed, but sending is currently paused.
**What to check:** `approval_status`, `reviewed_at`.
**Escalate when:** waiting for approval for more than a few business days (admin team) or accepted (dev).

### "I scheduled my email for a specific time. Will it go out then?"
**Answer for the user:** The scheduled time isn't saved at the moment; campaigns are handled after approval. We've noted your preferred time.
**What to check:** `email_campaigns.schedule` is empty.
**Escalate when:** the seller needs a timed send (dev).

### "The SMTP test fails."
**Answer for the user:** Please check the host, port, username, app password and encryption with your email provider. Many providers (for example Gmail) need an "app password" and port 587 with TLS or 465 with SSL.
**What to check:** the error text shown; whether the test email arrived in the seller's login inbox.
**Escalate when:** the same settings work in another email client.

### "I can't create a campaign; it asks me to upgrade."
**Answer for the user:** Email campaigns are a plan feature. Please check the plans on the Manage Subscription page.
**What to check:** the seller's plan and the `email_campaign` value on it (AFS-12).
**Escalate when:** the plan should include it.

### "Resend gives an error."
**Answer for the user:** We're looking into it; please create the campaign again from the Send tab for now.
**What to check:** server error on `email-campaigns/resend/{id}`.
**Escalate when:** always (dev).

## Known issues
- **Approved campaigns are never sent**: the sending job is switched off in the scheduler. Escalate to dev.
- **Schedule time is ignored**: the app doesn't send it to the server.
- **No review notification**, although the success screen promises one.
- **Seller-cancelled campaigns still show "Pending"** as status.
- **Resend probably fails** *(to verify)*: it copies recipients with the wrong IDs/types, which makes the request error. Workaround: create the campaign again.
- An unreachable "Reveal Affiliate Emails" modal and bulk reveal in discovery are broken (AFS-05); reveal from the wizard's pop-up works.

## Related topics
- AFS-05 — revealing affiliate emails (needed before an affiliate can receive a campaign).
- AFS-12 — plan feature for email campaigns.
- AFS-13 — team permissions.
- AFC-01 / CHAT-01 — messaging affiliates: use the **Ainfluencer chat**.

## Technical reference
- **Endpoints:** `seller/v1/email-campaigns/list`, POST `store` (plan gate `seller_access_feature:email_campaign`), GET `cancel/{id}`, GET `resend/{id}`; `seller/v1/smtp-connections/list|store|update/{id}|delete/{id}|test` → controllers in `app/Http/Controllers/Seller/V1/EmailCampaign/` and `.../SmtpConnection/`; services `app/Services/Actions/Seller/EmailCampaign/*`, `app/Services/General/EmailCampaign/EmailCampaign.php`, `app/Services/General/SmtpConnection/*`.
- **Admin approval:** `admin/v1/requests/...` → `Services/Actions/Admin/Request/EmailCampaign/UpdateApprovalStatusEmailCampaignService` (sets `approval_status`, `rejection_reason_id`, `reviewed_at`, `reviewed_by`; no notification, no send).
- **Sending (not scheduled):** `app/Console/Commands/EmailCampaign/ProcessEmailCampaignNotificationCommand.php` → `Jobs/EmailCampaign/ProcessEmailCampaignEmailsJob` (queue `email-campaign`) → `SendEmailCampaignMailService` → recipient handlers in `Services/General/EmailCampaign/EmailCampaignRecipientTypes/`. Commented out in `app/Console/Kernel.php`. See `docs/EmailCampaignProcess.md`.
- **Data:** `email_campaigns` (`user_type_id`, `smtp_connection_id`, `subject`, `message`, `schedule`, `status` 0 pending/1 running/2 completed/3 cancelled, `approval_status` 0 waiting/1 accepted/2 rejected/3 cancelled, `reviewed_at`, `started_at`), `email_campaign_recipients` (`model_type`, `model_id`, `type`), `smtp_connections` (`host`, `port`, `encryption_type`, `username`, `from_email`).
- **UI:** `spa-seller/src/components/email-campaigns/*`.
