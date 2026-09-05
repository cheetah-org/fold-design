# Matching Service — API

Base URL: `https://api.wiingman.in/matching/api/v1` — gateway convention `{baseURL}/{service}/api/v1/{resource}`; all paths below are relative to this base (e.g. `POST https://api.wiingman.in/matching/api/v1/users/{targetId}/likes`) · Content-Type: `application/json` · Dates: ISO-8601 UTC · IDs: UUID

**Module:** `matching` in the Spring Boot modulith. Owns `LIKE`, `MATCH`, and the discovery deck (`FEED_CURSOR` + eligible-men pool — the `discovery` half per inter-service-communication.md §7 (2) is covered by **this** service's API surface; deck endpoints are specced in §Discovery feed below — the ranking/selection algorithm is a build-time decision, deliberately out of this contract). Premium surfaces (spotlight, advanced filters, like-delivery insights) are **deferred** — this doc is free-tier only; nothing here precludes bolting them on later.

**Auth:** `Authorization: Bearer <access_token>` — reads `oid`, `onb`, `roles` per `auth.md` §1.1. **Every endpoint requires a valid Bearer token.**

**Un-onboarded gate:** `onb=false` → `403 ONBOARDING_INCOMPLETE` (canonical, `auth.md` §2). **Owner check:** `oid` vs path segment; non-owner/non-privileged → `403 FORBIDDEN`; `DEV`/`ADMIN` bypass owner checks only — **business rules (gender, age, cooldown) apply to every role**.

**Resources** addressed by `userId` — no `/users/me`. Visibility/masking semantics (404 indistinguishable) follow `user.md`.

---

## Like & match rules (contract)

| Rule | Value |
|---|---|
| Direction | **Women like men** (hetero pairing per product + ER). Men's only actions: accept / pass / unmatch. A male caller on like-creation → `403 FORBIDDEN`, no role bypass. |
| Age gate | Target must be **strictly younger** than the liker — enforced server-side at like creation **and** re-checked at accept (`422 AGE_RULE`). A failed re-check leaves the like `PENDING` (review fix) — only `pass` clears it before the 48h TTL. |
| Like TTL | `expires_at = created_at + 48h`. Hourly sweep flips `PENDING → EXPIRED`. Silent: removed from his inbox, she is never told, nothing is notified. |
| Re-like cooldown | After a pair ends in `PASSED`, `EXPIRED`, or `UNMATCHED`, no new like either direction for **30 days** (configurable) → `409 RELIKE_COOLDOWN`. |
| Like volume | **Unlimited** for women (free tier) — no cap, no counters exposed. |
| Seen/unread | **None** on likes (per decision). The man learns of a like via the `LIKE` push (notifications module); the inbox is a plain paged list. |
| Match | One `ACTIVE` match per pair. Accept creates it; `MatchCreated` → `messaging` opens the conversation (its id backfills onto the match **asynchronously** — `conversation_id` may be `null` briefly). |
| Unmatch | Either party, any time. `MatchDeleted` → `messaging` purges the thread; the pair enters the 30-day cooldown. |

**Status enums:** `LIKE`: `PENDING | ACCEPTED | PASSED | EXPIRED` · `MATCH`: `ACTIVE | UNMATCHED`.

---

## Authorization matrix

| Capability | `USER` | `DEV`/`ADMIN` |
|---|---|---|
| `POST /users/{targetId}/likes` (female caller) | ✅ | ✅ |
| Male caller on like-creation (any role) | ❌ 403 | ❌ 403 |
| `GET /users/{userId}/feed` — deck (female caller, owner) | ✅ | ✅ |
| Male caller on deck (any role) | ❌ 403 | ❌ 403 |
| Own inbox / accept / pass / matches / unmatch | ✅ owner | ✅ |
| Same on another user's path | ❌ 403 | ✅ |

---

## Error

Canonical envelope (`auth.md` §2); Auth Lib `TOKEN_*` set global; `429 TOO_MANY_REQUESTS` may be returned by central infrastructure (mechanism TBD). None repeated per endpoint.

```json
{
  "error": {
    "code": "RELIKE_COOLDOWN",
    "message": "Pair on cooldown. Try again after 12 days.",
    "traceId": "8f2c1a3d-4b5e-4c6f-9a0d-1e2f3a4b5c6d"
  }
}
```

---

## DTOs

`LikeDto` — a like as seen by its creator (the woman)

```json
{
  "id": "d7c4e6a8-1b2c-4d3e-9f0a-5b6c7d8e9f01",
  "liked_id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d",
  "status": "PENDING",
  "created_at": "2026-09-01T10:00:00Z",
  "expires_at": "2026-09-03T10:00:00Z"
}
```

`LikeInboxItemDto` — the man's inbox row (embeds the woman's masked `PublicUserDto`, `user.md` §DTOs)

```json
{
  "like_id": "d7c4e6a8-1b2c-4d3e-9f0a-5b6c7d8e9f01",
  "from": {
    "id": "3f8c2a11-5b6d-4c7e-9f0a-b1c2d3e4f5a6",
    "name": "Priya",
    "age": 31,
    "gender": "FEMALE",
    "location_name": "BTM Layout, Bangalore",
    "bio": "Coffee, music, long walks",
    "description": "Looking to meet someone younger.",
    "preferences": ["coffee", "live music"],
    "photos": []
  },
  "created_at": "2026-09-01T10:00:00Z",
  "expires_at": "2026-09-03T10:00:00Z"
}
```

`MatchDto` — one match as seen by a member

```json
{
  "id": "e8d5f7b9-2c3d-4e4f-8a1b-6c7d8e9f0a12",
  "counterpart": {
    "id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d",
    "name": "Rohan",
    "age": 28,
    "gender": "MALE",
    "location_name": "BTM Layout, Bangalore",
    "bio": "I make coffee",
    "description": "Looking to meet someone older.",
    "preferences": ["coffee"],
    "photos": []
  },
  "status": "ACTIVE",
  "conversation_id": null,
  "created_at": "2026-09-01T10:05:00Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `counterpart` | object | masked `PublicUserDto` resolved via `UserClient` — never the raw other-side account |
| `status` | enum | `ACTIVE` \| `UNMATCHED` |
| `conversation_id` | string\|null | set once `messaging` creates the thread (`MatchCreated` handler); `null` until then |

`PagedDto<T>`

```json
{ "items": [], "page": 0, "size": 20, "total": 0 }
```

`FeedPageDto` — one deck page (cursor-based, not offset-paged)

```json
{
  "items": [
    {
      "id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d",
      "name": "Rohan",
      "age": 28,
      "gender": "MALE",
      "location_name": "BTM Layout, Bangalore",
      "bio": "I make coffee",
      "description": "Looking to meet someone older.",
      "preferences": ["coffee"],
      "photos": []
    }
  ],
  "next_cursor": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d"
}
```

---

## Likes

### `POST /users/{targetUserId}/likes`
The woman likes the man. Creates `LIKE(PENDING, expires_at=+48h)`; publishes `LikeReceived` → push. No body.

**Guards (in order):** caller female (else `403 FORBIDDEN` — business rule, no role bypass) → target exists & visible (else `404 USER_NOT_VISIBLE`, same code/shape as `user.md` — review fix) → not self (`422 CANNOT_LIKE_SELF`) → target strictly younger (`422 AGE_RULE`) → no duplicate `PENDING` (`409 LIKE_EXISTS`) → pair not in cooldown (`409 RELIKE_COOLDOWN`).

**Response 201 — `LikeDto`**

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` — male caller |
| 404 | `USER_NOT_VISIBLE` — target missing/hidden/`DEACTIVATED` (canonical masking code per `user.md`) |
| 409 | `LIKE_EXISTS` — duplicate `PENDING` · `RELIKE_COOLDOWN` |
| 422 | `CANNOT_LIKE_SELF` · `AGE_RULE` · `VALIDATION_ERROR` — unexpected body |

> A woman's *sent*-likes list is intentionally absent (product: no delivery feedback on free tier; insights are deferred premium).

### `GET /users/{userId}/likes`
Likes **received by** this user — the man's inbox. Always empty for women (by product rule). Newest first. `?page=&size=` → 200 `PagedDto<LikeInboxItemDto>`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not owner |
| 404 | `NOT_FOUND` — user missing |

### `POST /users/{userId}/likes/{likeId}/accept`
The man accepts a `PENDING` like → like `ACCEPTED`, `MATCH(ACTIVE)` created, `MatchCreated` published. No body. → 200 `MatchDto`.

**Concurrency:** conditional update `SET status='ACCEPTED' WHERE id=:id AND status='PENDING'` — a racing duplicate accept loses and gets `409 LIKE_NOT_PENDING`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not the receiver |
| 404 | `NOT_FOUND` — like not on this user |
| 409 | `LIKE_NOT_PENDING` — already accepted/passed/expired · `PAIR_INELIGIBLE` — blocked either direction, or either party not `ACTIVE` at accept time |
| 422 | `AGE_RULE` — re-check failed; the like stays `PENDING` (only `pass` clears it before TTL) |

### `POST /users/{userId}/likes/{likeId}/pass`
The man passes — like `PASSED`, **she is never notified**, pair enters cooldown. No body. → 200 `LikeDto` (status `PASSED`).

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not the receiver |
| 404 | `NOT_FOUND` |
| 409 | `LIKE_NOT_PENDING` · `PAIR_INELIGIBLE` |

---

## Matches

### `GET /users/{userId}/matches`
This user's matches. `?status=ACTIVE|UNMATCHED&page=&size=` (default `ACTIVE`). Newest first. → 200 `PagedDto<MatchDto>`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not owner |
| 404 | `NOT_FOUND` — user missing |

### `DELETE /users/{userId}/matches/{matchId}`
Unmatch — either party (must be a member). Sets `UNMATCHED` + `unmatched_by`/`unmatched_at`; publishes `MatchDeleted` → `messaging` purges the thread (async, at-least-once); pair enters cooldown. → `204`, idempotent (re-unmatch → `204`).

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` · `FORBIDDEN` — not owner |
| 404 | `NOT_FOUND` — match not on this user |

---

## Discovery feed (deck)

**Women only. Men cannot browse** — product rule, enforced server-side (business rule, no role bypass). Cards are served from the **event-fed eligible-men pool** (inter-service-communication.md L1): men who are `ACTIVE`, **strictly younger** than the caller, not blocked either direction, and not already liked/passed by her. The deck is algorithm-agnostic — cards carry no scores, no reasons, no filters in V1.

### `GET /users/{userId}/feed`
The woman's deck. Owner check + female-only gate. Cursor-based pagination via `FEED_CURSOR` (server-tracked): a call with no `cursor` continues from the server-stored `last_seen_profile_id` and **advances** it; passing `cursor` re-reads from that point (re-scroll) without advancing.

**Query params:** `?limit=` (default 20, max 50) · `?cursor=` (optional; a card id from a previous `next_cursor`)

**Response 200 — `FeedPageDto`**
```json
{
  "items": [
    {
      "id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d",
      "name": "Rohan",
      "age": 28,
      "gender": "MALE",
      "location_name": "BTM Layout, Bangalore",
      "bio": "I make coffee",
      "description": "Looking to meet someone older.",
      "preferences": ["coffee"],
      "photos": []
    }
  ],
  "next_cursor": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d"
}
```

| Field | Type | Notes |
|---|---|---|
| `items` | `PublicUserDto[]` | masked cards, no duplicates within a page; like/pass is **not** done here — cards link out to the matching actions (`POST /users/{targetId}/likes`) |
| `next_cursor` | string\|null | last card id of the page; `null` = deck exhausted (refill arrives via pool events) |

**Side effect:** each rendered card emits `ProfileViewed { viewer_id, profile_id, source: "DECK" }` → `analytics` (powers "X women viewed your profile today"; profile-open views are emitted by the `users` module with `source: "PROFILE"`).

**Like / pass from a card:** the client calls the matching endpoints directly (`POST /users/{targetId}/likes`, §Likes) — the deck has no embedded action endpoints, and a liked/passed card never reappears (matching state excludes it from the pool view).

**Algorithm:** ranking/selection inside the pool is a build-time decision (design-doc "TBD") — this contract is deliberately independent of it.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` — male caller (any role) or not owner |
| 404 | `NOT_FOUND` — user missing |
| 422 | `VALIDATION_ERROR` — bad `limit`/`cursor` |

---

## Rate limiting

Rate limiting is a cross-cutting concern handled by shared infrastructure, not implemented per-endpoint here — mechanism TBD (same placeholder policy as `ratio.md`/`analytics.md`).

---

## Non-functional notes

### Idempotency & concurrency
- `POST .../likes` — **not** idempotent by design: duplicate intent surfaces as `409 LIKE_EXISTS` (truthful, and the client can treat it as success).
- accept/pass — single-winner via conditional status update; losers get `409 LIKE_NOT_PENDING`.
- unmatch — idempotent `204` (anti-enumeration, same semantics as `auth.md` logout / `user.md` unblock).

### Expiry sweep
Hourly job flips `PENDING → EXPIRED` where `expires_at < now`; `expires_at` on the row is authoritative, so the client can hide expired inbox items before the sweep runs. Expiry is silent on all sides.

### Published events (contract)

Spring Modulith transactional event publication — at-least-once, retried listeners (mechanism per `auth.md` §9.5). Payloads snake_case JSON.

| Event | Payload | Consumers |
|---|---|---|
| `LikeReceived` | `{ like_id, liker_id, liked_id }` | `notifications` (push), `analytics` |
| `MatchCreated` | `{ match_id, woman_id, man_id }` | `messaging` (open conversation), `notifications` |
| `MatchDeleted` | `{ match_id, unmatched_by }` | `messaging` (purge thread), `notifications` |
| `ProfileViewed` | `{ viewer_id, profile_id, source: DECK\|PROFILE }` | matching (deck, §Discovery feed) / users (profile open) → `analytics` |

### Cross-module notes
- **Profile reads:** inbox/match cards embed `PublicUserDto` via the `users` module's `UserClient` (sync, in-process — inter-service-communication.md L3). Masking/mirrored-404 semantics owned there.
- **Eligibility pool:** the deck is served from the event-fed pool (inter-service-communication.md L1) — `UserRegistered`/`UserDeactivated`/`ProfileUpdated`/`PhotoChanged` listeners maintain it; never per-card sync calls.
- **View events:** deck cards emit `ProfileViewed(source=DECK)` per rendered card; the `users` module emits `source=PROFILE` on profile open — `analytics` aggregates both for "X women viewed you today" and the weekly summary.
- **ER synced:** `LIKE.status` includes `EXPIRED`; `FEED_CURSOR` (server-tracked cursor, §Discovery feed) is owned by this module's discovery half; `PROFILE_VIEW_EVENT.source` includes `DECK` — all reflected in `entities/er-diagram.md`.
- **Deferred (premium):** spotlight boost, advanced feed filters, like-delivery insights — future contract additions; the like/match/deck model above doesn't preclude them.