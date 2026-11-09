# Messaging Service (Chat) — API

Base URL: `https://api.wiingman.in/messaging/api/v1` — gateway convention `{baseURL}/{service}/api/v1/{resource}`; all paths below are relative to this base (e.g. `GET https://api.wiingman.in/messaging/api/v1/users/{userId}/conversations`) · Content-Type: `application/json` · Dates: ISO-8601 UTC · IDs: UUID

**Module:** `messaging` (the flow docs' "Chat Service"). Owns `CONVERSATION`, `CONVERSATION_MEMBER`, `MESSAGE` (ER). A conversation is created when a match happens (`MatchCreated` → thread opens, 1:1 with match) and purged on unmatch (`MatchDeleted`). **Text only** — no media, no unsend, no read receipts. Every message self-destructs **48h after send** (per-message TTL).

**Transport split (locked):** REST for everything mutative (send, read, clear); **WebSocket is delivery-only** — the server pushes new-message frames to the recipient. Reconnect = refetch list + messages.

**Auth:** `Authorization: Bearer <access_token>` — reads `oid`, `onb`, `roles` per `auth.md` §1.1. **Owner checks:** chat resources are strictly per-member — even `DEV`/`ADMIN` get masked reads (chat content is never exposed cross-user; privileged access = the same view the owner sees, via owner path only).

**Un-onboarded gate:** `onb=false` → `403 ONBOARDING_INCOMPLETE` (canonical, `auth.md` §2).

---

## Chat rules (contract)

| Rule | Value |
|---|---|
| Message TTL | `expires_at = sent_at + 48h`; a 1-min sweep deletes expired messages. Deleted messages are gone for **both** sides (no history extension, no recovery). |
| Expiry warning | when a conversation's **last** message crosses **T−2h**, emit `MessageExpiring` (once per message) → `notifications` renders the `EXPIRY` push + email ("made plans yet?"). |
| Content | text only, `body` ≤ 2000 chars, no media/links-preview/edits/unsend. |
| Read state | **per-member unread counts only** — `last_read_message_id` drives badges; the sender can **never** see read status (no read receipts). |
| Masking ("User no longer exists") | a member is masked when the match is `UNMATCHED`, the counterpart is `DEACTIVATED`, or either side **blocked** the other. Masked thread: the window/list entry **remains** with `state=USER_NO_LONGER_EXISTS`, history stays until TTL, **sends fail with `404 USER_NOT_VISIBLE`** (indistinguishable which of the three happened — same semantics as `user.md`). The masked side can delete the thread from their end any time (WhatsApp-style clear). |
| Unmatch purge | `MatchDeleted` → all messages in the thread are deleted; the initiator's view disappears immediately (member hidden); the unmatched side keeps the masked window (§ above). |
| Deactivation / block | `UserDeactivated` / `BlockCreated` → the counterpart's side gets full masking (§ above); `BlockRemoved` restores visibility. |
| Conversation identity | 1:1 with match — no group chat, no new conversations initiated in this module. |

---

## DTOs

`MessageDto`

```json
{
  "id": "f1a2b3c4-d5e6-4f7a-8b9c-0d1e2f3a4b5c",
  "conversation_id": "c63d2e4f-5a6b-4c7d-9d8e-9f0a1b2c3d4f",
  "sender_id": "9f3a1c4e-2b7d-4a81-9c5e-8d2f0a6b3c11",
  "body": "Coffee tomorrow?",
  "sent_at": "2026-09-05T18:00:00Z",
  "expires_at": "2026-09-07T18:00:00Z"
}
```

`ConversationDto` — list item

```json
{
  "conversation_id": "c63d2e4f-5a6b-4c7d-9d8e-9f0a1b2c3d4f",
  "match_id": "e8d5f7b9-2c3d-4e4f-8a1b-6c7d8e9f0a12",
  "state": "ACTIVE",
  "counterpart": {
    "id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d",
    "name": "Rohan",
    "username": "rohan_m",
    "age": 28,
    "gender": "MALE",
    "location_name": "BTM Layout, Bangalore",
    "bio": "I make coffee",
    "description": "Looking to meet someone older.",
    "height": 175,
    "drinking": "SOMETIMES",
    "smoking": "NEVER",
    "photos": []
  },
  "unread_count": 2,
  "last_message": {
    "body": "Coffee tomorrow?",
    "sent_at": "2026-09-05T18:00:00Z",
    "expires_at": "2026-09-07T18:00:00Z"
  }
}
```

| Field | Type | Notes |
|---|---|---|
| `state` | enum | `ACTIVE` \| `USER_NO_LONGER_EXISTS` — masked threads keep the entry (§Chat rules); `counterpart` is **not serialized** in the masked state (never even as null-padded fields) |
| `counterpart` | object | masked `PublicUserDto` via `UserClient` |
| `unread_count` | integer | caller's own badge (`last_read_message_id` → count of later messages) |
| `last_message` | object\|null | caller-visible latest message; `null` when thread empty/purged |

`PagedDto<T>`

```json
{ "items": [], "page": 0, "size": 20, "total": 0 }
```

---

## Conversations

### `GET /users/{userId}/conversations`
The caller's threads, newest activity first. Masked threads included (`state=USER_NO_LONGER_EXISTS`) until the caller clears them. `?page=&size=` → 200 `PagedDto<ConversationDto>`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not owner |
| 404 | `NOT_FOUND` — user missing |

### `GET /users/{userId}/conversations/{conversationId}/messages`
History for a thread the caller is a member of. Cursor-paged, newest first: `?before=<messageId>&limit=` (default 50). Expired messages are already gone (sweep). Masked thread → history still readable until TTL.

**Response 200 — `PagedDto<MessageDto>`**

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not a member |
| 404 | `NOT_FOUND` — thread gone (purged/cleared/never existed) |

### `POST /users/{userId}/conversations/{conversationId}/messages`
Send. Guards in order: member (`403 FORBIDDEN`) → thread exists (`404 NOT_FOUND`) → counterpart visible (`404 USER_NOT_VISIBLE` — masked/unmatched/deactivated/blocked; indistinguishable) → validation (`422`). Sets `expires_at = now + 48h`; delivers to the counterpart via WebSocket (§Delivery).

**Request**
```json
{ "body": "Coffee tomorrow?" }
```

| Field | Type | Req | Constraints |
|---|---|---|---|
| `body` | string | ✅ | 1–2000 chars, text only |

**Response 201 — `MessageDto`**

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not a member |
| 404 | `NOT_FOUND` — thread gone · `USER_NOT_VISIBLE` — counterpart masked |
| 422 | `VALIDATION_ERROR` — empty/oversized `body` |

### `POST /users/{userId}/conversations/{conversationId}/read`
Mark-read: sets the caller's `last_read_message_id = message_id` (unread → 0 from there). No receipts — the counterpart learns nothing.

**Request**
```json
{ "message_id": "f1a2b3c4-d5e6-4f7a-8b9c-0d1e2f3a4b5c" }
```

**Response 204**

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not a member |
| 404 | `NOT_FOUND` — thread or message not in it |

### `DELETE /users/{userId}/conversations/{conversationId}`
Clear **own view** (WhatsApp-style) — idempotent `204`; never affects the other side. The thread stays gone from the caller's list even if new activity would have been masked anyway.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not a member |
| 404 | `NOT_FOUND` |

---

## WebSocket delivery (server → client only)

**Endpoint:** `GET /ws/chat` (public path: `wss://api.wiingman.in/messaging/api/v1/ws/chat`) — upgrade with `Authorization: Bearer <access_token>` (same token, same claims). Each authenticated connection is bound to the principal's user id; the server pushes frames for **their** threads only. No client→server frames (send is REST).

**Frame — new message**
```json
{
  "type": "MESSAGE",
  "message": {
    "id": "f1a2b3c4-d5e6-4f7a-8b9c-0d1e2f3a4b5c",
    "conversation_id": "c63d2e4f-5a6b-4c7d-9d8e-9f0a1b2c3d4f",
    "sender_id": "9f3a1c4e-2b7d-4a81-9c5e-8d2f0a6b3c11",
    "body": "Coffee tomorrow?",
    "sent_at": "2026-09-05T18:00:00Z",
    "expires_at": "2026-09-07T18:00:00Z"
  }
}
```

- Delivery is best-effort: offline recipients catch up via `GET .../conversations` + `.../messages` on reconnect (no offline queue, no delivery guarantees beyond the store).
- Match-creation and expiry awareness arrive via notifications push (FCM) — not WebSocket frames.

---

## Non-functional notes

### TTL sweep & expiry warning
- 1-min sweep deletes `MESSAGE.expires_at < now` rows.
- When a thread's **last** message crosses `T−2h`, emit `MessageExpiring { conversation_id, expires_at }` once (dedup per message) → `notifications` renders the `EXPIRY` push + email. If a newer message lands, the warning moves to the new last message.

### Consumed events (lifecycle)
| Event | Effect |
|---|---|
| `MatchCreated` | create `CONVERSATION` (1:1) + two visible members (idempotent on `match_id`) |
| `MatchDeleted` | delete all messages in the thread; hide the initiator's member view; the other side keeps the masked window |
| `UserDeactivated` | mask threads with that counterpart (`state=USER_NO_LONGER_EXISTS`) |
| `BlockCreated` | full masking for the blocked pair's thread (§Chat rules) |
| `BlockRemoved` | restore visibility (unless self-cleared) |

All transactional, at-least-once (mechanism per `auth.md` §9.5); lifecycle handlers are idempotent.

### Flood control

Removed for now — chat flood/abuse controls will be defined together with the central cross-cutting rate-limiting mechanism (placeholder policy as in `ratio.md`/`analytics.md`).

### ER synced
`CONVERSATION_MEMBER` now includes `last_read_message_id` (unread badges) and `cleared_at` (self-clear — distinct from `visible`, which masking flips, so a `BlockRemoved` restore doesn't resurrect a deliberately cleared thread) in `entities/er-diagram.md`.

---

## Cross-module notes

- **`MatchingClient` finalized:** `getMatch(matchId) → MatchInfo { match_id, woman_id, man_id, status }` — used by `MatchCreated` (thread creation) and for masking checks (match `UNMATCHED` ⇒ masked). See `commons.md`.
- **`UserClient`** provides counterpart cards (`PublicUserDto`) and status (`DEACTIVATED` masking); `user.md` masking semantics apply verbatim.
- **No new-message push:** product specifies push for likes/matches/expiry — offline message catch-up is the client's refetch on reconnect.
- **Unsend/read receipts:** deliberately absent — nothing in this API mutates or reveals delivery/read state after send.

---

## Implementation notes — core chat (V1 backend)

Choices made where the contract is silent or differs; the code follows these.

- **Thread creation.** `MatchCreated` (handled asynchronously) creates the `CONVERSATION` and its two members, idempotent on `match_id`. It then publishes **`ConversationCreated { conversation_id, match_id }`** (contract addition — the spec said the id "backfills" without saying how); `matching` records it as `match.conversation_id`, first recorded id wins. So `conversation_id` is `null` on the match for a moment after it is created.
- **Masking is derived live, not stored.** A thread is masked (`USER_NO_LONGER_EXISTS`, no `counterpart`) when the match is `UNMATCHED` (or gone), or the counterpart is not visible to the viewer through `UserClient` — blocked either way, deactivated, suspended, shadow-banned. So `CONVERSATION_MEMBER.visible` does not exist, `UserDeactivated` / `BlockCreated` / `BlockRemoved` are not consumed, and lifting a block restores the thread by itself. A shadow-banned sender's thread therefore looks masked to the counterpart, which is how "their messages don't deliver" is honoured.
- **Unmatch (`MatchDeleted`).** All messages are purged for both sides; the initiator's `cleared_at` is set (their view disappears at once); the other side keeps the now-empty masked thread until they clear it. Idempotent.
- **Clearing.** Sets the caller's `cleared_at` only; a cleared thread answers `404` to that member's history, send and read. When both members have cleared (or one cleared and the other was the unmatch initiator), the conversation becomes `DELETED` and any remaining messages are removed.
- **Strict privacy.** Every endpoint requires `oid == userId` — no role bypass, moderators included (unlike the matching endpoints).
- **Unread badge.** Counts the *other* side's messages after the member's `last_read_message_id`; their own messages never count. The marker only moves forward (reading an older message leaves it). If the marker message has already expired, every remaining message from the other side counts.
- **History.** Newest first, keyset on `before=<messageId>` (ordered by send time, id as tie-break). `limit` defaults to 50 and is capped by `fold.messaging.max-history-limit` (100); an unknown `before` or an out-of-range `limit` is `422 VALIDATION_ERROR`. `total` is the number of messages in the thread. The conversation list's `page`/`size` are validated the same way as in matching.
- **Sending.** Guards in the documented order; a masked thread answers `404 USER_NOT_VISIBLE`. `body` must be non-blank and at most `fold.messaging.max-body-length` (2000) characters.
- **Config:** `fold.messaging.message-ttl` (48h), `expiry-warning-lead` (2h), `sweep-interval` (1m), `max-body-length`, `default-history-limit`, `max-history-limit`, `default-page-size`, `max-page-size`, `ws-send-time-limit` (5s), `ws-buffer-size-limit` (64 KiB).
- **WebSocket delivery (implemented).** `GET /messaging/api/v1/ws/chat`, upgraded with the same `Authorization: Bearer` token. The ordinary token filter authenticates the handshake request; the socket is then bound to the token's user id. A missing/invalid token is refused by the filter (`401`), and a caller who has not finished onboarding gets `403` (the handshake can only carry a status, not the JSON error body). Frames are server-to-client only, and anything a client sends is ignored. Spring's WebSocket support (`spring-boot-starter-websocket`) is the only new dependency.
- **Frames.** One type today: `{ "type": "MESSAGE", "message": MessageDto }`. A frame is pushed to **every open socket of the recipient only** (not the sender's own devices) after the sending transaction commits, off the request thread, so a slow client never delays the sender. Delivery is best effort exactly as specced: no offline queue, and a failed or slow send (`ws-send-time-limit`, `ws-buffer-size-limit`) drops that one socket.
- **Sockets end with their token (addition).** A socket is closed once the access token it connected with expires, checked when a frame is about to be sent and by the 1-minute sweep. The client reconnects with a fresh token. This stops a revoked or expired session from receiving messages for longer than the access-token lifetime.
- **Single instance.** The set of open sockets lives in memory, so this works on one running instance. Running several, or extracting the service, would need a broker to fan frames out to the right instance.
- **Expiry warning (implemented).** A sweep (same `sweep-interval` as the expiry sweep) finds messages that are **the newest in their thread**, not yet warned, and expire within `fold.messaging.expiry-warning-lead` (2h). Each gets `expiry_warned_at` stamped (so it fires once) and a `MessageExpiring` event. A message that is no longer the thread's last is skipped, so when a newer message lands the warning belongs to that one when its own window opens.
- **Payload gains `recipient_ids` (addition to the spec).** The spec's `{ conversation_id, expires_at }` names no recipients, and `notifications` must not reach into messaging's membership or privacy rules. Messaging therefore decides who to nudge and ships them in the event: members who have **not cleared** the thread and can still see the other side. A thread whose match has ended, that is deleted, or is masked (block, shadow-ban, deactivation) warns **nobody**, so the warning can never hint at one of those. The message is still marked warned in that case. A thread where both members cleared sends nothing.
- **Last-message caveat.** The warning concerns the newest message only, as specced. Older messages in the thread expire silently without one.