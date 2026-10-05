---
id: AFC-03
title: Help center (tickets)
domain: Shared by seller and affiliate apps
audience: [seller, affiliate, admin]
apps: [seller-app, affiliate-app]
related: [AFC-02, AFS-13]
last_verified: 2026-10-05
source: affiliate/ainflu-affiliate-backend@v1.5.27, affiliate/ainflu-affiliate-frontend@v1.4.02
---

# AFC-03 · Help center (tickets)

> Sellers and affiliates can open support tickets from inside the affiliate platform, add files, follow the conversation with the Ainfluencer team, and close the ticket when done. Admins answer from the affiliate admin panel.

## Who and where
- **Seller app:** sidebar **More → Help Center**, or the user menu "Help center" → `/app/support-tickets`. Team members need the help-center permissions (view / create / update) in their role (AFS-13).
- **Affiliate app:** user menu "Help center" → `/app/support-tickets`.
- Ticket page: `/app/support-ticket-details/:id`.
- **Admin:** affiliate admin panel (out of scope here).

## How it works
1. The list page shows "Tickets (n)", a "Frequently Asked Questions" section and "Create New Ticket".
2. **Create:** "Submit new ticket" form: "Department", "Impact of issue" (low, minor, major, critical), "Title", "Message", optional attachments → "Send Request". Confirmation: "Ticket Created Successfully" / "We will review your inquiry and get back to you as soon as possible."
3. **Conversation:** on the ticket page the user reads replies, writes a new message and can "Upload File". Each message changes the ticket's status (see Statuses).
4. **Admin reply:** the user gets "💬 New Ticket Response" in the bell immediately; the email copy goes out with the next batch (06:00 or 18:00 UTC, AFC-02).
5. **Close:** "Close Ticket" (from the list or the ticket page). Writing a new message on a closed ticket re-opens it as "Pending".
6. "Rate this conversation" shows stars.

## Rules and limits
- Title up to 255 characters; message required (a reply can be just a file).
- **Attachments:** up to 3 files per message. The app accepts PDF, JPG, PNG and DOC and says "maximum file size is 5MB". (The server itself allows more types and up to 20 MB each.)
- Departments come from a list managed by admins.
- Only the ticket owner (or their team members with permission) can see or answer it.

## Statuses
| Status (user) | Internal | Meaning |
|---|---|---|
| "Pending" | `new_ticket`, `user_response` | Waiting for the Ainfluencer team. |
| "Answered" | `admin_response` | The team replied; waiting for the user ("Waiting for your reply"). |
| "In Progress" | `in_progress` | The team is working on it. |
| "Closed" | `closed` | Closed by the user or an admin. A new message re-opens it. |

Admins see `user_response` as "Customer Response" and `admin_response` as "Responded".

## Timers and automatic actions
- No automatic closing. Reply emails are batched at 06:00 / 18:00 UTC.

## Notifications
| Trigger | Who | Channel |
|---|---|---|
| Admin replies | Ticket owner | In-app immediately + email (batched) |
| User creates/replies | — | None to the user |

## Common questions

### "I opened a ticket but nobody answered."
**Answer for the user:** Your ticket is in our queue; we reply as soon as possible. You'll see "Answered" and get a notification when we do.
**What to check:** `tickets.status`, last `ticket_messages` row and who wrote it.
**Escalate when:** the ticket has been "Pending" longer than the support target.

### "I can't upload my file."
**Answer for the user:** Please use a PDF, JPG, PNG or DOC file of 5 MB or less, up to 3 files per message.
**What to check:** file type and size.
**Escalate when:** a valid file still fails.

### "I got an email saying there's a reply, hours after it was posted."
**Answer for the user:** Reply emails are sent in batches twice a day; the in-app bell shows replies immediately.
**What to check:** `user_type_notification_queues` for `TicketRepliedNotification`.
**Escalate when:** no in-app notification either.

### "I closed my ticket by mistake."
**Answer for the user:** Just write a new message in it; that re-opens the ticket.
**What to check:** status becomes `user_response`.
**Escalate when:** —

### "My team member can't see our tickets."
**Answer for the user:** Ask the account owner to give their role the "Help Center" permissions.
**What to check:** role permissions (AFS-13).
**Escalate when:** permission is set and they still can't.

## Known issues
- **"Rate this conversation" saves nothing**: the stars are never sent to the server.
- The ticket detail page has no permission check in the seller app's menu guard; the server still checks access.

## Related topics
- AFC-02 — notifications and email batching.
- AFS-13 — team permissions for the help center.

## Technical reference
- **Endpoints:** `{seller|affiliate}/v1/tickets` (GET list, POST create, multipart), GET `/{ticket}`, PATCH `/{ticket}` (close), POST `/{ticket}/messages` → `app/Http/Controllers/{Seller|Affiliate}/V1/Ticket/*` → `app/Services/General/Ticket/Ticket.php` (`createMessage` sets status `admin_response` or `user_response`).
- **Admin reply notification:** `Services/General/Email/EmailServiceManager::sendTicketReplied` → `TicketRepliedNotification` via `NotificationQueue` (setting "Important Notifications").
- **Validation:** `app/Http/Requests/Seller/V1/Ticket/CreateTicketRequest.php`, `CreateTicketMessageRequest.php` (max 3 files, 20 MB each server-side).
- **Data:** `tickets` (`user_type_id`, `title`, `priority`, `ticket_category_id`, `status`), `ticket_messages` (`ticket_id`, `created_by_type`, `message`), attachments in media (`attachments` collection), `ticket_categories`.
- **UI:** `shared-components/algo-tickets/*`.
