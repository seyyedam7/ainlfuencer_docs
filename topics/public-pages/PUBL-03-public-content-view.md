---
id: PUBL-03
title: Public content view
domain: Public (no-login) pages
audience: [brand]
apps: [web]
related: [PUB-01, PUB-02, PUB-06, PUBL-02, BRD-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# PUBL-03 · Public content view

> A share link to the photos or videos an influencer delivered for a deal. Brands copy it to show the content to colleagues or agencies who don't have an Ainfluencer login. The link opens a simple viewer with the delivered files.

## Who and where

- **Brands** copy the link from:
  - "Influencer content" (`/influencer-content`): "Copy share link" on a content item;
  - the inbox chat (`/new-offersV2/:id`): the "Final Delivered Result" preview → "Copy".
- **Anyone** with the link opens `app.ainfluencer.com/view-contents/<code>?content_id=<id>`. No login is needed.

## How it works

1. The brand copies the share link. The link contains a coded reference to the offer and the id of the delivered item.
2. The visitor opens it and sees a carousel of the delivered images and videos (videos play with controls; arrows and dots appear when there are several files).
3. Nothing else is shown: no caption, no influencer details, no approve/disapprove actions. To download or approve, the brand uses the inbox or "Influencer content" (PUB-02).

## Rules and limits

- The link works without login and **doesn't expire**. Anyone who has it can view the files.
- It shows **uploaded files** delivered through Ainfluencer. A delivery that is only a link to a social media post has no files to show (open the post from the inbox instead).
- The link must include the `content_id` part. A shortened or edited link without it shows an empty page.
- Team members who can't open "Influencer content" (BRD-03) can still copy the link from the inbox.
- Translated or dubbed versions (PUB-06) are available to the brand in its account, not through this viewer *(to verify)*.

## Statuses

Not applicable.

## Timers and automatic actions

None.

## Notifications

None.

## Common questions

### "How can I share the influencer's content with my team or agency?"
**Answer for the user:** Open "Influencer content" (or the chat with the influencer) and use "Copy share link" / "Copy". Anyone with the link can view the files without logging in.
**What to check:** The delivery has uploaded files (not only a post URL).
**Escalate when:** The copy button is missing for a delivery with files.

### "The share link opens a blank page."
**Answer for the user:** Please make sure you copied the whole link, including the part after "?content_id=". If it still doesn't work, copy the link again from Influencer content.
**What to check:** The link has both the code and `content_id`; the delivered item still exists.
**Escalate when:** A freshly copied full link shows nothing.

### "Can I stop people from viewing a link I shared?"
**Answer for the user:** Share links don't expire and can't be switched off from your account. Please share them only with people you trust. Contact us if a link was shared by mistake.
**What to check:** Nothing.
**Escalate when:** The brand needs a link disabled (dev action).

### "The link says 'The offer link is invalid or has expired'."
**Answer for the user:** The link is damaged or incomplete. Please copy it again from your account.
**What to check:** The code part of the URL.
**Escalate when:** A newly copied link shows the same message.

## Known issues

- The viewer is minimal: it shows only the files of the delivered item, without caption or download button.
- Links without `content_id` show an empty page instead of an error.

## Related topics

- PUB-01: how the influencer delivers files or a post link.
- PUB-02: approving deliveries; "Influencer content" page.
- PUB-06: translated versions of content.
- PUBL-02: the influencer's public media kit.
- BRD-03: what team members can open.

## Technical reference

- **Endpoint** (no login): `GET /api/v2/published-contents/offers/{encodedOfferId}?content_id=` (registered in `app/Modules/OpenAI/Routes/api.php`) → `v2/Guest/PublishedContentController@showByOffer` (`ShowByOfferRequest` decodes the id with `OfferIdCodec`; invalid → "The offer link is invalid or has expired.") → `Services/Implementation/PublishedContentService::showByOffer`.
- **Data**: `publish_jobs` (by `offer_id`), `published_contents`, `contents` / `medias` / `files`.
- **Web**: page `src/components/pages/viewContentsNoAuth` (route `/view-contents/:code`); link built in `src/components/pages/contents2` and the inbox `mobileFrame` preview.
