---
id: PUB-06
title: Content translation
domain: Content delivery and settlement
audience: [brand, system]
apps: [web, email, background]
related: [PUB-02, OFR-03, PAY-01, PAY-02, BRD-03]
last_verified: 2026-10-05
source: core/backend/api@99921fbce, core/frontend/web@588fe862e
---

# PUB-06 · Content translation

> A brand can order a dubbed version of a video (an influencer's delivered video or one the brand uploads) in other languages, "in the influencer's own voice". It's a paid service run by BrandID's translation service; the finished videos appear in the brand's "Translated Videos" page and an email is sent when the order is complete.

## Who and where

- **Brand, web app:**
  - `/profile/translated-contents` ("Translated Videos", from the header menu and profile tabs): "Translate a video" button, sections "In progress", "Translated" and "Failed", with play, "Download" and "Copy link".
  - `/influencer-content` ("Contents"): "Translate" action on a creator's video.
  - Inbox chat, delivered content preview: "Translate Video" ("Need this content in another language?") and "See Translation".
- **Team members** can see the account owner's translations; the Contents page itself is owner-only (PUB-02).
- Influencers are not involved and are not notified.

## How it works

1. The brand opens "Translate a video". Step "Choose the video": "Upload a new video" (MP4, MOV or WebM, up to 500 MB) "or pick from your creator content".
2. The modal shows the video length and how it's billed ("{duration} long · billed as {minutes}").
3. Step "Choose languages": search the list or use presets "Top 5 markets", "Europe", "Asia", "LatAm". The total is shown as "{duration} ({billed}) × {languages} × ${rate}" and an estimate "Ready in roughly {N} minutes, we email you each one".
4. "Pay and translate" creates the request and opens the normal checkout (card, wallet or PayPal; see OFR-03/PAY-01). Closing the payment window without paying deletes the unpaid request.
5. After payment the system sends the video and languages to BrandID. The brand receives a "translation request paid" email and in-app notification. The request shows under "In progress" ("Working on it…", "{ready} of {total} ready").
6. BrandID calls back when each language is done. Ainfluencer downloads the dubbed video and stores it. When **all** languages are finished, the request moves to "Translated" and the brand gets a "translation completed" email.
7. If BrandID reports a failure for a language, that language is marked failed and the request appears under "Failed" ("These translation jobs did not finish successfully", "charged"). "Try again" opens a new, separately paid request.

## Rules and limits

- **Price:** billed minutes × number of languages × the per-minute rate shown in the modal (default **$0.99 per minute per language**; the rate is a server setting).
- **Billed minutes:** any video up to 59 seconds counts as 1 minute. Longer videos: whole minutes, plus 0.5 minute if the extra part is under 30 seconds, plus 1 minute if it's 30 seconds or more. Examples: 1:20 → 1.5 minutes; 1:45 → 2 minutes.
- The price sent from the browser must match the server's calculation within $0.10, otherwise the request is refused.
- One video per request from the new modal; any number of languages.
- An unpaid request can be deleted; a request that has started can't.
- Payment goes through the normal Ainfluencer checkout and appears in billing history.

## Statuses

| Section on the page | Internal `translation_requests.status` / `translations.status` | Meaning |
|---|---|---|
| (not shown) | request `todo` | Created but not paid yet (or the provider didn't accept the job, see Known issues) |
| "In progress" | request `in_progress`; translation `todo`/`in_progress`, `external_status` `pending`/`running`/`success` | Paid and being dubbed |
| "Translated" | request `completed`; translation `completed` | All languages ready to play/download |
| "Failed" | translation `failed`, `external_status=failed` | Provider couldn't dub one or more languages |

Messages shown per language: "The video rendering waiting in queue.", "The video is rendering.", "The video render success.", "An error occurred when rendering video."

## Timers and automatic actions

- After the job is sent, a status check runs about 1 minute later; afterwards the result arrives through BrandID's callback.
- No automatic refund or retry exists for failed or stuck translations.

## Notifications

| Trigger | Who | Channel |
|---|---|---|
| Translation paid | Brand ("translation request paid") | email, in-app |
| All languages finished | Brand ("translation completed") | email |
| Failure | nobody | none |

## Common questions

### "How much will a translation cost?"
**Answer for the user:** $0.99 per minute of video, per language (as shown in the modal). Videos under a minute count as one minute. For example, a 1 minute 20 second video into 3 languages is 1.5 × 3 × $0.99 = $4.46.

### "I paid but don't see my translation."
**Answer for the user:** Translations usually take around 20 minutes and appear under "In progress" on the Translated Videos page. We'll check the order for you.
**What to check:** `translation_requests` for the brand user (`status`, `price`, `created_at`); the related order/order item is paid; `translations.external_status` and `external_message`.
**Escalate when:** the request is still `todo` after payment, or nothing changed for more than a few hours.

### "My translation failed. Do I get my money back?"
**Answer for the user:** We're sorry the translation didn't work. We'll review it and arrange a refund or a new translation.
**What to check:** `translations.status=failed`, `external_message`.
**Escalate when:** always; refunds for translations are manual (finance/admin).

### "One language is ready but I didn't get an email."
**Answer for the user:** The email is sent when all languages in your order are ready. You can already play and download the finished ones on the Translated Videos page.

### "Can I translate a video that wasn't delivered through Ainfluencer?"
**Answer for the user:** Yes, use "Upload a new video" in the translate window (MP4, MOV or WebM, up to 500 MB).

### "Can my team member order translations?"
**Answer for the user:** Team members can see the account's translations. The Contents page is only for the account owner. *(ordering from the Translated Videos page by a team member: to verify)*

## Known issues

- **"If not available within 72h, you'll be refunded"** is shown in the older translation window, but there is no automatic refund in the system. Refunds must be done by the team. Escalate to finance.
- "We email you as each language lands" is not accurate: one email is sent when the whole order is finished, and none on failure.
- If the provider refuses the job when it's sent, the request goes back to `todo`, which hides it from the brand's page even though it was paid. *(to verify)* Escalate to dev.
- A request with one failed language never reaches "Translated", even if the other languages are done; the finished files can still be downloaded.

## Related topics

- PUB-02 Contents page and delivered content preview (entry points).
- OFR-03 / PAY-01 checkout and billing history; PAY-02 refunds.
- BRD-03 team members.

## Technical reference

- **Endpoints** (routes in `app/Modules/Translation/Routes/api/brand.php`):
  - `GET /api/v2/translations/languages`, `POST /api/v2/translations` (`translateable{type,id}`, `file_ids[]`, `languages[]`, `price`), `DELETE /api/v2/translations/{id}` (only `todo`) → `TranslationRequestController` → `app/Modules/Translation/Services/Implementation/TranslationRequestService`.
  - `GET /api/v2/translations` (by translateable) → `TranslationController@showTranslateable`.
  - `GET /api/v2/translation-requests` (excludes `todo`), `…/{id}/download/all`, `…/{id}/translated/{fileId}/download` → `TranslationRequestController`.
  - Callback `POST /api/v2/webhooks/translations/callback` → `TranslationCallbackController` → `Services/Implementation/TranslationCallbackService` (`video_translate.success` / `video_translate.fail`).
- **Pricing:** `app/Modules/Translation/Util/TranslationPriceCalculator.php`, rate `translation.default_price_per_minute` (`Config/translation.php`, default 0.99); check in `Http/Requests/StoreTranslationRequest.php`.
- **Flow:** `OrderPaidEvent` → `Listeners/TranslationRequestPaidListener` → `Jobs/TranslateMultipleVideos` → `Drivers/BrandId/BrandIdDriver::handleTranslations` (request → `in_progress`) → callback → `Jobs/UploadTranslatedVideo` → `Listeners/TranslationUploadedListener` (request `completed`, `OrderItemCompletedEvent`, `Email/TranslationCompleted`). Failure: `Listeners/TranslationFailedListener`. Driver default `brand_id` (HeyGen inactive).
- **Data:** `translation_requests` (`user_id`, `translateable_type/id`, `languages`, `price`, `status`), `translations` (`translation_request_id`, `original_file_id`, `language`, `status`, `external_status`, `external_message`, `external_translation_id`, `external_url`, `translated_file_id`); order items with `item_type=TranslationRequest`.
- **Web:** `components/pages/translation2` (`TranslateModal.tsx`), `components/common/modalsTranslationVideo`, `lib/services/translateContent.ts`; labels in `locales/en/translation2.json` and `contentTranslation.json`.
