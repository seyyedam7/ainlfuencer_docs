# Ainfluencer chat system: live inventory (support flow map)

Sources are the production snapshots under `` (`chatbox-gateway`, `chatbox-api`, `chatbox-processor` on master, `api` (Laravel) on master, `web` snapshot, `mobile-app` on develop). In this document, `GW` = `chatbox-gateway/src`, `CA` = `chatbox-api/src`, `CP` = `chatbox-processor/src`, `L` = `api/`, `W` = `web/src`, `M` = `mobile-app/src`.

## 0. Architecture in one paragraph

```
 web (brand)  ──socket.io + REST (x-chat-auth / x-public-key)──┐
 mobile (influencer) ─────────────────────────────────────────┤
                                                              ▼
                                                  chatbox-gateway (NestJS, socket.io + Redis adapter)
            Laravel ──HTTP (http://chatbox-gateway)──────────▶│  │ reads   ──HTTP──▶ chatbox-api (Mongo reads, internal)
            ▲                                                 │  │ writes  ──Kafka request/reply──▶ chatbox-processor (Mongo writes + Redis unread cache)
            └──POST /api/v2/chatbox/webhook {event:'message-sent'}┘
 Laravel hourly digest reads processor's Redis keys directly (identities_with_unread, latest_unread:*)
```

Clients talk to Laravel for inbox lists, thread metadata, templates, address cards, archive/pin, and bulk messages. They talk to the gateway directly for live messages (socket), history (`GET /message/conversation/:id`), and unread counts (web only).

---

## 1. chatbox-gateway

### 1.2 Connection / room logic (`GW/modules/gateway/ws.service.ts`)
- On connect:
  1. Authenticate.
  2. Store the socket id in Redis `socketId:{identityId}` and Mongo `SocketConnection.socketIds`.
  3. Load rooms from Redis `conversations:{identityId}` (fallback Mongo `SocketConnection.rooms`).
  4. `client.join(rooms)`. **Room = conversationId.**
- **No client-driven join/subscribe events exist.** Rooms are joined only server-side:
  - `joinConversation()` on conversation create (`POST /conversation`).
  - On participant create.
  - On participant/conversation status → ACTIVE.
  
  `removeParticipant()` runs on leave, on status != ACTIVE, and on conversation delete.
- On disconnect: mark disconnected in Mongo and remove the id from the Redis set.
- `emitToRoom(room, event, data, status, message, exclude)` sends `{data, message, status}` to the room, optionally excluding socket ids.
- Transport: socket.io with the Redis sharded adapter (`GW/redis.io.adaptor.ts`), websocket plus polling allowed.

### 1.3 Socket events handled (client → gateway)
| Event | DTO | Handler | Effect |
|---|---|---|---|
| `new-message` | `{conversationId, body?, attachment?{path,content_type,file_name}, attributes?, emitToSender?}` | `GW/modules/gateway/message.gateway.ts` → `message.service.ts newMessage` | Kafka `new-message-topic` (request/reply, key=conversationId). On reply: emit `new-message` to the room, **excluding all of the sender's sockets unless `emitToSender`**. Then POST the Laravel webhook. The return value is the socket **ack** (the created message). |
| `edit-message` | `{messageId, conversationId, body?, attributes?}` | same | Kafka `edit-message-topic`, then emit `edit-message` to the room (excluding the requester). |
| `delete-message` | `{messageId}` | same | Kafka `delete-message-topic`, then emit `delete-message` to the room (excluding the requester). |
| `react-message` | `{messageId, reaction, participant}` | same | Kafka `react-message-topic`, then emit `react-message` to the room (excluding the requester). |
| `seen-message` | `{messageId}` | `GW/modules/gateway/delivery.gateway.ts` → `delivery.service.ts seenMessage` | Kafka `seen-message-topic`. **Returns an ack only. Nothing is broadcast**, so there are no read receipts to the other party. |

Errors: `WsExceptionFilter` (`GW/filter/ws.exception.filter.ts`) emits **under the originating event name** (for example `seen-message` with `{status:400,message:'MESSAGE_ALREADY_SEEN'}`), or `error-event` when no event is attached. A 401 disconnects the socket.

### 1.4 Socket events emitted (gateway → clients)
`new-message`, `edit-message`, `delete-message`, `react-message`, `new-conversation`, `edit-conversation`, `new-participant`, `edit-participant`, `error-event`. Enum: `GW/types/events.enums.ts`. The gateway never emits `delivered-message`, `unread_count_updated`, `channel_updated`, `message_received`, or a broadcast `seen-message`.

### 1.5 HTTP routes (the gateway proxies reads to chatbox-api at `CHATBOX_API_BASE_URL`)
| Method + path | Goes to | Notes |
|---|---|---|
| GET `/conversation/byQuery` | CA | system only (`identityId<=0`); filter `attributes.offer_id` etc. |
| POST `/conversation` | CA, then `joinConversation` for every participant, then emit `new-conversation` | |
| GET `/conversation` | CA | paginated list for caller (empty for system ids) |
| DELETE `/conversation/delete/:id` | CA | system only (`identityId<0`) |
| GET `/conversation/:id` | CA | conversation + participants; ACTIVE only |
| PATCH `/conversation/:id` | CA, then emit `edit-conversation`, then join/remove rooms on status change | |
| GET `/message/conversation/:id?page&pageSize&order&startIndex` | CA | history (non-deleted), sorted by `index` |
| GET `/message/unread` | CA | unread messages grouped by conversation |
| GET `/message/unread-count?excludeRoles=a,b` | CA | `{total, <conversationId>: n}` |
| GET `/message/search/:id?body=` | CA | regex on body |
| GET `/message/:id` | CA | admin/system only |
| POST `/message/send-message` | Kafka (same as socket `new-message`, plus webhook) | used by Laravel |
| PATCH `/message/edit-message` | Kafka | used by Laravel |
| DELETE `/message/delete-message/:messageId` | Kafka | used by Laravel |
| POST `/participant` | CA, then join room, then emit `new-participant`, then Redis `participantId:` | admin join |
| GET `/participant/conversation/:id` | CA, plus `onlineParticipants` from Redis socket sets | |
| PATCH `/participant/:id` | CA, then join/remove room, then emit `edit-participant` | |
| DELETE `/participant/leave/:id` | CA, then remove room | |
| GET `/delivery/seen/:id` | CA | deliveries for a message |
| POST `/delivery/seen/:id` | Kafka `seen-message-topic` | HTTP twin of socket seen (used by Laravel mobile offer-state) |
| GET `/delivery/remove-cached-unread/:cachedIdentityId` | Kafka `remove-cached-unread-topic` | system only; used after the email digest |

### 1.6 Kafka topics produced (request/reply via `ClientKafka.send`, `GW/services/kafka/kafka.producer.ts`; topic enum `GW/types/topic.enums.ts`)
`new-message-topic`, `edit-message-topic`, `delete-message-topic`, `react-message-topic`, `seen-message-topic`, `remove-cached-unread-topic`. The gateway has no Kafka consumers of its own; it only subscribes to replies.

### 1.7 Outbound calls to Laravel
- `POST {AINFLUENCER_BASE_URL}/api/v2/chatbox/webhook` with `{event:'message-sent', message:<created message doc>}`.
- Fired **only after `newMessage`**. That covers both socket and HTTP send-message, and therefore also every Laravel-originated system message.
- It is not fired for edit, delete, react, or seen.
- Failure is swallowed (`console.log`).
- Source: `GW/modules/message/message.service.ts coreWebhook`.

---

## 2. chatbox-api and chatbox-processor

### 2.1 chatbox-api routes (`CA/modules/*/*.controller.ts`)
- `/conversation`: GET `byQuery`, POST `/`, GET `/`, DELETE `delete/:id`, GET `:id`, PATCH `:id`.
- `/message`: GET `unread`, GET `unread-count`, GET `conversation/:id`, GET `search/:id`, GET `:id`.
- `/participant`: POST `/`, GET `conversation/:id`, PATCH `:id`, DELETE `leave/:id`.
- `/delivery`: GET `seen/:id`.

Notable behavior:
- Create conversation runs in a transaction: Conversation plus Participants, with Redis `participantId:{conv}:{identityId}=participantId` (`CA/modules/conversation/conversation.service.ts`).
- Conversation status update sets every participant to ACTIVE or **RESTRICTED** and sets or clears the `participantId:` Redis keys.
- The list excludes CLOSED conversations and BLOCKED/LEAVED participations.
- Get conversation, search, and unread consider only **ACTIVE** conversations.

### 2.2 Processor Kafka consumers (`CP/services/kafka/consumers/*.ts`)
| Topic | Service | Persists |
|---|---|---|
| `new-message-topic` | `CP/modules/message/message.service.ts newMessage` | Validates the body (users need a valid body or an attachment). Conversation must be ACTIVE for `identityId>0`. Access check (system, admin auto-joins, or participant). Admin/system messages get `attributes.role` = `admin`/`system_user`; user messages keep the client-supplied `attributes.role`. Transaction: `Message{index=messageCount+1, author=participantId, status=CREATED}` plus `Conversation.messageCount++`. Redis unread cache `setUnreadData`. |
| `edit-message-topic` | `editMessage` | Author-only for normal users. Sets `status=UPDATED` and merges body/attributes (preserves `attributes.twilio_sid`). Updates the unread cache. |
| `delete-message-topic` | `deleteMessage` | Author-only for normal users. Soft delete `status=DELETED`. Removes it from the unread cache. |
| `react-message-topic` | `reactMessage` | Appends `{reaction, participant}` to `attributes.react` (an **array**). No dedupe or toggle. |
| `seen-message-topic` | `CP/modules/delivery/delivery.service.ts seenMessage` | Rejects the message's own author (`AUTHOR_OF_MESSAGE_CANT_HAVE_SEEN_STATUS`). Rejects `index <= lastReadIndex` (`MESSAGE_ALREADY_SEEN`). Transaction: `Delivery{messageId, participantId, index}` plus `Participant.lastReadIndex=index, lastReadAt=now`. Then Redis `deleteUnread`. |
| `remove-cached-unread-topic` | `removeCachedUnread` | System only. Clears the Redis unread cache for one identity. |

Also in `CP/script/migration/*`: a Twilio → chatbox migration script (`npm run migrate`). It is ops-only, not runtime.

### 2.3 Unread logic (two independent mechanisms)
1. **Authoritative count (Mongo)**: `CA/modules/message/message.service.ts getUnreadScopes/getUnreadMessageCount` and `CA/repositories/message.repository.ts buildUnreadMatch`.
   - For each ACTIVE participant row of the identity, in an ACTIVE conversation, where `messageCount > lastReadIndex`:
   - count messages with `index in (lastReadIndex, messageCount]`, `author != own participant`, `status != DELETED`.
   - Optional `excludeRoles`: `attributes.role $nin`. Messages without a role still count.
   - Marking seen any message **advances `lastReadIndex` to that message's index**, which clears everything before it.
2. **Digest cache (Redis, written by the processor)**: `CP/services/cache/cache.service.ts`.
   - On new, edit, or react, for every other participant found via `participantId:{conv}:*`:
     - `SADD identities_with_unread {id}`
     - `LPUSH latest_unread:{id}:{conv}` (keeps the last 3)
     - `ZADD identity:conversations:{id}` (keeps the last 20)
   - Seen trims these keys.
   - Laravel reads them for the hourly email digest (section 3.4).

### 2.4 Data model (Mongo; `GW/entities/*.ts` = `CP/entities/*.ts` = `CA/entities/*.ts`)
- **Conversation**: `status` (ACTIVE | SYSTEM_ONLY | CLOSED), `messageCount`, `attributes` (Laravel sets `offer_id`, `name`, `influencer{id,username,platform}`, or context attributes for affiliate/product_seeding).
- **Participant**: `identityId` (Laravel user id, or negative for system), `identityName`, `conversationId`, `status` (ACTIVE | LEAVED | BLOCKED | RESTRICTED), `lastReadIndex`, `lastReadAt`, `attributes.role` (`brand` / `influencer` / `admin` / `system_user`). Unique on (identityId, conversationId).
- **Message**: `index` (per-conversation sequence), `author` (participantId), `conversationId`, `status` (CREATED | UPDATED | DELETED), `body`, `attachment{path,content_type,file_name}`, `attributes` (Map), `createdAt`. Attributes in use:
  - `role` (sender role; Laravel's webhook and notifier rely on it).
  - `type`: marks system or card messages. Values:
    - offer lifecycle: `offer:created`, `offer:collaboration-terms`, `offer:awaiting-payment`, `offer:accepted`, `offer:updated`, `offer:paid`, `offer:rejected`, `offer:failed`, `offer:scheduled`, `offer:published`, `offer:completed`, `offer:published_content`, `offer:insight_requested`, `offer:insight_sent`, `offer:automatic_insight_requested`, `offer:fulfillment_complete_address_by_influencer`
    - admin edits: `admin.offer:pending|ignored|accepted|rejected|discarded|updated`
    - publish jobs: `publish-job:*` (published, completed, refunded, transferred, publishing-failed, completion-failed, health-failed, approved/disapproved-by-admin/brand)
    - revisions: `revise:requested|accepted|rejected`
    - cards and bulk: `meta:influencer-address` (address card), `meta:affiliate_link_message` (web affiliate link), `brand:bulk-message`
  - `event`, `data` (`data.batch` = offer-update batch; `data.ignore`), `is_systematic`, `link`.
  - `react` (array, from socket react).
  - `reactions` (single string toggle, from Laravel REST react).
  - legacy `twilio_sid`.
- **Delivery**: `messageId`, `participantId`, `index` (unique on message+participant).
- **Clients**: `clientName`, `publicKey`, `privateKey`, `participantId`.
- **SocketConnection**: `identityId`, `socketIds[]`, `rooms[]`.

---

## 3. Laravel side

### 3.3 How conversations are created
- **Offers (influencer → brand)**: `POST /api/{loc}/v3/mobile/offers` → `L/app/Modules/Mobile/Services/Implementation/Offer/OfferCreatorService.php store()`. It calls `setupChatBoxConversation()` (gateway `POST /conversation` with 2 participants: influencer role `influencer`, brand role `brand`, as the influencer user).
  - Saved as `offers.chat_id = offers.chatbox_conversation_id`.
  - Then `updateChatBoxConversation()` PATCHes attributes `{offer_id, name, influencer{...}}`.
  - Then `SendOfferInitialMessagesJob` (`L/app/Jobs/Offer/SendOfferInitialMessagesJob.php`) posts `offer:created` (system), `offer:collaboration-terms` (as influencer), and `offer:awaiting-payment` (as influencer). It is idempotent: it checks existing `attributes.type` first.
  - Temporary offers are created the same way.
  - Brands do not create offers in this codebase (no brand POST /offers).
- **Affiliate / product-seeding threads (brand ↔ influencer, campaign-scoped)**: created **lazily on the first message** through `L/app/Modules/ChatBox/Services/ConversableMessageSender.php`:
  1. `ConversationContextService::getOrCreate(..., isFirstMessage:true)` (`L/app/Modules/ChatBox/Services/ConversationContextService.php`).
  2. Room (pair of users) via `RoomService`.
  3. `ConversationContext` scoped by (ad_id, social_account_id, room).
  4. `createContext()`: gateway `POST /conversation`, then PATCH attributes, then a `conversation_contexts` row with `provider_conversation_id`.
  
  Entry points:
  - Brand web `POST /api/v2/chatbox/chats/messages` (`subject_type`, `subject_id`, `influencer_user_id`, `message`).
  - Mobile `POST /api/{loc}/v3/mobile/chats/messages`.
  - Mobile affiliate send-address.
  - `SendProductSeedingAddressJob`.
  
  The subject types registered in `ChatBoxServiceProvider` are `affiliate` and `product_seeding` (`L/app/Modules/ChatBox/Adapters/*`).
- **Legacy sync**: Offer and TemporaryOffer observers (`L/app/Modules/Offer/Observers/*ConversationSyncObserver.php`) mirror offer conversations into `rooms` / `conversation_contexts` (`LegacyConversationContextSynchronizer.php`), so the unified inbox lists offers too.
- **Conversation status**: `UpdateConversationStatus` job (`L/app/Jobs/v2/Offer/UpdateConversationStatus.php`). Triggered by:
  - `ConversationStateChangeListener`
  - `UserQualityControlChangeListener`
  - `ConversationStatusChanger` command (sets `SYSTEM_ONLY`)
  
  A non-ACTIVE status makes participants RESTRICTED. Users cannot send, the conversation drops out of unread counts, and sockets leave the room.
- **Admin join/leave**: `PUT/DELETE /api/v2/admin/offers/{id}/chat/join|leave` → `L/app/Services/Implementation/Admin/OfferService.php` (gateway `POST /participant`, `DELETE /participant/leave/:id`).
- **Maintenance**:
  - `chatbox:append-conversation-id` / `AppendConversationIdToOffer` (gateway `GET /conversation/byQuery`)
  - `chatbox:audit-participants` (repair or recreate)
  - `PruneConversations`, `BulkPruneOffers`, `BulkRemoveConversation` (gateway `DELETE /conversation/delete/:id`)

### 3.4 Webhook on message-sent (`L/app/Modules/ChatBox/Http/Controllers/WebhookController.php`)
- Route: `POST /api/v2/chatbox/webhook` (`L/app/Modules/ChatBox/Routes/api.php`; prefix in `Providers/RouteServiceProvider.php`).
- Validation: `conversationId` must exist in `offers.chatbox_conversation_id` or `conversation_contexts.provider_conversation_id`.
- **Offer-backed conversation**:
  - Sets `offers.last_message`. The preview body is replaced by "Offer updated." for any typed message except `offer:created`, `offer:collaboration-terms`, `offer:awaiting-payment`.
  - Sets `last_message_at`, and `updated_at` when newer.
  - Then `notifyMessageReceiver`:
    - Skipped when `SystemMessageDetector` says it is a system message (`attributes.type` present, `data.batch` set, or `is_systematic===true`; `L/app/Modules/ChatBox/Support/SystemMessageDetector.php`).
    - Otherwise sends `App\Notifications\Message` (channels **DATABASE + FIREBASE push**). Role `brand` → influencer. Role `influencer` → brand. Other or missing → both.
  - No email and no offer-state change here.
- **Context-backed thread** (affiliate / product_seeding): `ConversationContextService::touchLastMessage`, then `ConversationMessageNotifier::notify` (`L/app/Modules/ChatBox/Services/ConversationMessageNotifier.php`). It sends `ConversationMessageNotification` (DATABASE + FIREBASE) to the room participants other than the sender, and skips system messages.
- **Side effect**: `brand:bulk-message` carries a `type`, so bulk messages produce **no push notification**.

**Email digest** (`AggregatedConversationMessageNotifier`, scheduled **hourly** in `L/app/Console/Kernel.php`):
1. `DispatchUnreadMessageFetcher` (queue `chatbox:unread-message-collector`) reads Redis `identities_with_unread`.
2. `FetchUnreadMessages` per non-admin user reads `latest_unread:{userId}:*` and sends `ConversationAggregatedMessage` (**EMAIL only**; separate brand and influencer templates).
3. `L/app/Listeners/NotificationSentListener.php` then calls gateway `GET /delivery/remove-cached-unread/{userId}`, which clears the cache.

Files: `L/app/Modules/ChatBox/Console/Commands/AggregatedConversationMessageNotifier.php`, `Jobs/DispatchUnreadMessageFetcher.php`, `Jobs/FetchUnreadMessages.php`, `Services/AggregatedMessageService.php`, `Notifications/ConversationAggregatedMessage.php`.

### 3.5 System messages Laravel posts into chat (`L/app/Modules/ChatBox/Services/ChatboxMessageSenderService.php` and others; all go through gateway `POST /message/send-message`, which in turn triggers the webhook)
| Trigger | Caller | Method / type |
|---|---|---|
| Offer created | `L/app/Jobs/Offer/SendOfferInitialMessagesJob.php` | `offer:created`, `offer:collaboration-terms`, `offer:awaiting-payment` |
| Offer accepted / updated (counter, history accept) by brand | `L/app/Http/Controllers/OfferController.php:350,366`; `L/app/Services/Implementation/OfferService.php:750,943-951`; `L/app/Repositories/Offer.php:862-872` | `sendOfferUpdate` (`offer:accepted` / `offer:updated`) |
| Offer accepted / updated by influencer (mobile) | `L/app/Modules/Mobile/Services/Implementation/Offer/OfferHistoryService.php:184,211`; `OfferUpdaterService.php:397-405` | `sendOfferUpdate` |
| Offer update (queued) | `L/app/Modules/Offer/Jobs/ConversationSendOfferUpdateJob.php` | `sendOfferUpdate` |
| Offer paid | `L/app/Listeners/v2/Offer/OfferPaidEventListener.php:95` | `sendOfferUpdate(OFFER_PAID)` |
| Start date changed at payment / by brand | `L/app/Modules/Payment/Services/Implementations/Offer/PaymentService.php:142`; `PayPal/PaymentCallbackService.php:161`; `OfferService.php:1289,1353` | `sendOfferStartChangeMessage` |
| Start date changed by influencer | `L/app/Modules/Mobile/Services/Implementation/PublishJobService.php:73`; `L/app/Services/Implementation/Mobile/PublishJobService.php:87` | `sendOfferStartChangeMessageByInfluencer` |
| Status change (rejected, failed, scheduled, published, completed, ...) | `L/app/Listeners/v2/Offer/OfferStatusChangedEventListener.php:27` | `sendChangeStatusMessage` |
| Awaiting payment | `L/app/Jobs/Offer/UpdateStatusToAwaitingPayment.php:69` | `sendChangeStatusMessage(offer:awaiting-payment)` |
| Admin edits offer (pending, ignored, then accepted/rejected/discarded) | `L/app/Services/Implementation/Admin/OfferService.php:287`; `L/app/Modules/Offer/Http/Controllers/OfferController.php:393`; resolution via `updateOfferUpdatedByAdminMessage` in `OfferController.php:412,438`, `OfferService.php:1009-1062`, `OfferHistoryService.php:129,155` | `admin.offer:*`. Messages are **edited in place** through `PATCH /message/edit-message`, which emits `edit-message`. |
| Revision requested / accepted / rejected | `L/app/Modules/ReviseOffer/Jobs/SendRevisionChatMessage.php`; `L/app/Modules/Offer/Listeners/SendRevisionOfferMessageListener.php` | `sendRevise` (`revise:*`). Removal: `RevisionConversationMessageRemover`, `RemoveRevisionOfferMessageListener` (delete-message). |
| Publish-job updates (published, failed, completed, refunded, ...) | `L/app/Notifications/PublishJobNotification.php:422` | `sendPublishJobUpdate` |
| Content approved / disapproved | `L/app/Jobs/ApprovedBrand.php`, `ApprovedAdmin.php`, `DisApprovedAdmin.php` | `sendPublishJobEvent` |
| Published content card | `L/app/Modules/PublishJob/Jobs/SendPublishedContentMessage.php` | `sendPublishedContent` (`offer:published_content`; replaces the previous card) |
| Insight requested / sent | `L/app/Services/Implementation/InsightService.php:138` | `sendInsight` |
| Influencer address on an offer | `L/app/Modules/Mobile/Jobs/SendInfluencerAddressJob.php` (from mobile `offers/{id}/send-address`) | `sendInfluencerAddressMessage` (`meta:influencer-address`) |
| Address on affiliate / product-seeding | `ChatConversationController::sendAffiliateAddress`, `L/app/Modules/ProductSeeding/Jobs/SendProductSeedingAddressJob.php` | `ConversableMessageSender` + `InfluencerAddressCard` |
| Shopify fulfillment address flow | `L/app/Modules/UserThirdParty/Services/NotificationShopifyFulfillmentServiceDecorator.php` | `sendCustomMessage(offer:fulfillment_complete_address_by_influencer)` |
| Offer questionnaire answers | `L/app/Jobs/v2/Offer/ProcessSendOfferQuestionnaire.php` | `sendCustomMessage` (as influencer, no type, so it notifies the brand) |
| Brand bulk message | `L/app/Jobs/v2/Offer/ProcessBulkMessage.php` (offers), `L/app/Modules/ChatBox/Jobs/ProcessBulkThreadMessage.php` (threads) | `brand:bulk-message` |
| Reaction (legacy REST) | `L/app/Services/Implementation/Common/ReactionService.php` | edits the message with `attributes.reactions` toggle (as system), which emits `edit-message` |

### 3.6 Laravel chat endpoints and whether clients call them
Brand (prefix `/api/v2/`, file `L/routes/api/v2/brand.php`) and mobile (prefix `/api/{locale}/v3/mobile/`, file `L/app/Modules/Mobile/Routes/api.php`). Both audiences share the controller `L/app/Modules/ChatBox/Http/Controllers/ChatConversationController.php`.

| Endpoint | Web | Mobile |
|---|---|---|
| GET `chatbox/chats` (rooms + threads + unread + bulk stats) / GET `chats` | yes (inbox, bulk, dashboardV3) | yes (inbox, badge) |
| GET `chatbox/chats/{ctx}` / `chats/{ctx}` | yes (ChatDrawer) | yes |
| POST `chatbox/chats/messages` / `chats/messages` | yes (affiliate, seeding, ChatDrawer) | yes (start chat) |
| POST `chats/affiliate/{adId}/send-address` | n/a | yes |
| archive / unarchive / pin / unpin thread, pin / unpin room | **no** (no web caller) | yes (`M/features/inbox/actions/useChatToggle.ts`) |
| GET `chatbox/unread-messages-count` (brand) | no | n/a |
| GET `/api/v2/chats` (legacy `OfferController::chats`) | only from the unmounted `newSingleOfferV2`/`singleOffer` rooms | n/a |
| POST `offers/send-bulk-message` | yes | n/a |
| `message-templates` CRUD + `default-chat-templates` | yes (v2) | yes (v3; v2 `/api/v2/mobile/message-templates` only from unreferenced legacy components) |
| POST `/api/v2/chat/react` | only from unmounted legacy pages | n/a |
| POST `v3/mobile/offers/{chatId}/messages/{id}/react` | n/a | hook imported, never invoked |
| POST `v3/mobile/offers/{id}/send-address` | n/a | yes |
| PUT `v3/mobile/offers/{id}/state` (Read → gateway `POST /delivery/seen/{last_message.id}`) | n/a | yes |
| POST `refresh-conversation-token` | no | no |
| POST `refresh-twilio-token` | yes (Twilio leftover, sidebar) | no |
| POST `/api/v2/offers/list-by-twilio-ids` | no | no |
| POST `/api/v2/offers/list-by-conversation-ids` | called by the unmounted header widget; **no such route** | n/a |

---

## 4. Client ↔ server cross-match

Full machine-readable list: `chat-calls.json`. Summary below.

**Web (brand)**. The live inbox is `/new-offersV2[/:id]` → `W/components/pages/inboxV37`. Legacy `/offers/:id` (`singleOffer`) redirects there. `newSingleOfferV2/index.tsx` is not routed; only its `model.ts`, `context`, and parts are reused.

LIVE:
- socket emits `new-message` (`attributes.role='brand'`, no `emitToSender`, no ack callback) and `seen-message`
- socket listeners `new-message`, `edit-message`, `delete-message`, `error-event`
- REST to the gateway: `GET /message/unread-count` (sidebar badge, polled with throttling) and `GET /message/conversation/:id` (history)
- Laravel calls listed in 3.6

DEAD:
- emits `subscribe_channel`, `unsubscribe_channel`, `subscribe_all_conversations`, `join_channel`, `leave_channel`, `update_channel` (no gateway handlers; rooms are server-joined)
- listeners `delivered-message`, `unread_count_updated`, `channel_updated`, `message_received`
- listener `seen-message` (it only ever receives error payloads, never receipts)
- `GET /message/unread/id` (wrong path, and it uses the `Authorization` header)
- `POST v2/offers/list-by-conversation-ids`

TWILIO:
- `W/components/layouts/dashboard/sidebar/index.tsx` still instantiates `useTwilioClient` (from `singleOffer/model.ts`) using `chat_token_plus`. It refreshes via `/v2/refresh-twilio-token` and listens to Twilio `messageAdded` to fill redux `twilio*` counters.
- The visible badge comes from the chatbox `unread-count`, so this is a still-executing leftover.

Web never listens to `react-message`, `new-conversation`, `edit-conversation`, `new-participant`, or `edit-participant`. A new thread created while the inbox is open appears only after a list refetch.

**Mobile (influencer)**. `CHAT_PROVIDER='socket'` (`M/config/chatProvider.ts`).

LIVE:
- emits `new-message` with ack (`M/redux/chat/socketThunks.ts sendSocketConversationMessage`, `attributes.role='influencer'`), `seen-message` (`M/features/chat/screens/ChatScreen.tsx`), `react-message`
- listener `new-message`
- REST to the gateway: `GET /message/conversation/:id`
- Laravel v3 endpoints in 3.6

DEAD:
- emit `send-message` (`sendSocketMessage`; imported but never called, and there is no handler)
- listeners `delivered-message`, `seen-message` (receipt), `unread_count_updated`
- `product-seedings/{id}/send-address` (no route)
- REST react hook

MISSING listeners: mobile does not listen to `edit-message`, `delete-message`, or `react-message`. Admin offer edits, Laravel reactions, and brand edits are therefore not live on mobile until history is refetched.

TWILIO: `M/redux/chat/unreadMessageSlice.tsx initializeListeners` and `chatSlice` Twilio client are never initialised. They are dead, but `@twilio/conversations` is still a dependency.

**Laravel → gateway**: all used endpoints exist. One mismatch: `SeenEndpoint::get` targets `/messages/%s/seen` (the gateway route is `GET /delivery/seen/:id`); it is unused, so it is latent only.

---

## 5. User-visible chat features

| Feature | Web brand (inboxV37) | Mobile influencer | Backend path |
|---|---|---|---|
| Send text | yes (socket) | yes (socket, ack) | `new-message` → Kafka → Mongo, then webhook |
| Start a new affiliate / gifting thread | yes (REST from affiliate / seeding pages, ChatDrawer) | yes (REST `chats/messages`) | lazy conversation create |
| Attachments (image / file) | yes (`InboxV37ChatSender` uploads, then `attachment`) | yes | message `attachment` |
| Edit own message | no UI | no (commented out) | supported by the gateway |
| Delete own message | no UI | no | supported by the gateway |
| React | no (inbox has no reaction UI; legacy REST react only in unmounted pages) | yes (`react-message`; renders `attributes.react.reaction` on brand bubbles, but the processor stores an array, so it may not render after reload) | `react-message` / legacy `attributes.reactions` |
| Seen / read | sends seen (clears unread); no read-receipt ticks | sends seen and also `PUT offers/{id}/state` | `lastReadIndex`; no receipt broadcast |
| Unread badge | sidebar: gateway `GET /message/unread-count` (all roles); room list: Laravel `chatbox/chats` | Laravel `v3/mobile/chats` `total_unread_messages_count` (60 s poll); offers list excludes influencer-role messages | chatbox-api Mongo count |
| Push notification for a new message | n/a (web) | Firebase via Laravel webhook (`Message` / `ConversationMessageNotification`) | non-system messages only |
| Email digest of unread | yes (hourly) | yes (hourly) | Redis cache, then email |
| Message search | no client | no client | gateway `GET /message/search/:id` exists |
| Message templates | yes (v2 CRUD + default templates) | yes (v3 CRUD + default) | Laravel `message-templates` |
| Offer cards / system messages in chat | yes (renderers for `offer:*`, `admin.offer:*`, `publish-job:*`, `meta:*`, `brand:bulk-message`) | yes (`offer:*`, `admin.offer:*`, `publish-job:*`, `revise:*`, `meta:influencer-address`, `brand:bulk-message`) | section 3.5 |
| Address sharing | renders `meta:influencer-address` card | sends (offer, affiliate, product-seeding endpoints) | address card |
| Affiliate link message | yes (`meta:affiliate_link_message` via socket from `InboxV37AffiliateTab`) | renders as message | socket |
| Shopify fulfillment address request | yes (`confirmShippingAddress` sends a socket text; system card) | renders | `offer:fulfillment_*` |
| Bulk message | yes (offers + threads) | n/a | `brand:bulk-message` (no push) |
| Archive / pin threads and rooms | **no** | yes | Laravel per-viewer state |
| Online presence | no | no | gateway returns `onlineParticipants` on `GET /participant/conversation/:id` (unused) |
| Admin joins chat | n/a (admin panel) | sees `admin` role messages | `POST /participant` |

