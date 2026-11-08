# Matching Service — API

Base URL: `https://api.wiingman.in/matching/api/v1` — gateway convention `{baseURL}/{service}/api/v1/{resource}`; all paths below are relative to this base (e.g. `POST https://api.wiingman.in/matching/api/v1/users/{targetId}/likes`) · Content-Type: `application/json` · Dates: ISO-8601 UTC · IDs: UUID

**Module:** `matching` in the Spring Boot modulith. Owns `LIKE`, `MATCH`, and the discovery deck (`FEED_IMPRESSION` log + `FEED_CANDIDATE` eligible-men pool — the `discovery` half per inter-service-communication.md §7 (2) is covered by **this** service's API surface; deck endpoints are specced in §Discovery feed below — the ranking/selection algorithm is a build-time decision, deliberately out of this contract). Premium surfaces (spotlight, advanced filters, like-delivery insights) are **deferred** — this doc is free-tier only; nothing here precludes bolting them on later.

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
| Passed re-feed | After a `PASS` or `EXPIRED`, the man re-enters her deck after **7 days** (view only — she sees him again as a card). |
| Re-like cooldown | After a pair ends in `PASSED`, `EXPIRED`, or `UNMATCHED`, no new like either direction for **30 days** (configurable) → `409 RELIKE_COOLDOWN`. The 7-day re-feed and 30-day re-like are independent: she sees him at day 7, can like again at day 30. |
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
    "username": "priya_k",
    "age": 31,
    "gender": "FEMALE",
    "location_name": "BTM Layout, Bangalore",
    "bio": "Coffee, music, long walks",
    "description": "Looking to meet someone younger.",
    "height": 165,
    "drinking": "SOMETIMES",
    "smoking": "NEVER",
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

**Women only. Men cannot browse** — product rule, enforced server-side (business rule, no role bypass). Cards are served from the **event-fed eligible-men pool** (inter-service-communication.md L1): men who are `ACTIVE`, **strictly younger** than the caller (narrowable via `max_age`/`min_age` filters), within her distance radius, not blocked either direction, and not already liked/passed by her (passed profiles re-enter the pool after a **7-day cooldown**; she can't re-like until the 30-day cooldown expires).

### `GET /users/{userId}/feed`
The woman's deck. Owner check + female-only gate. Cursor-based pagination over a server-tracked queue (`FEED_IMPRESSION`, which replaced the earlier `FEED_CURSOR`). **Serving a card is not seeing it:** her server-tracked position is the last card the client reported as reached (`POST .../feed/seen`, below). A call with no `cursor` returns the served-but-unseen cards after that position first (so a retry or restart resumes where she actually was), and only picks a fresh batch when none are left. A call with `cursor=<card id>` returns the cards after that card — already-served ones first, else a fresh batch — which is how the client prefetches and re-scrolls.

**Query params:** `?limit=` (default 20, max 50) · `?cursor=` (optional; a card id from a previous `next_cursor`)

**Filter params (all optional, all free tier):**

| Param | Type | Notes |
|---|---|---|
| `distance_km` | integer | max distance from caller's location; default: no limit (region-wide) |
| `min_age` | integer | floor within structural "strictly younger" rule |
| `max_age` | integer | ceiling; must be < caller's age |
| `min_height` | integer | cm |
| `max_height` | integer | cm |
| `religion` | string | comma-separated enum values; match any |
| `education_level` | string | comma-separated enum values; match any |
| `drinking` | string | exact enum match |
| `smoking` | string | exact enum match |

Filters narrow the eligible pool before random shuffle. Men missing a filtered field (e.g. no `religion` set) are **excluded** when that filter is active — incentivizes profile completion.

**Response 200 — `FeedPageDto`**
```json
{
  "items": [
    {
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
    }
  ],
  "next_cursor": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d"
}
```

| Field | Type | Notes |
|---|---|---|
| `items` | `PublicUserDto[]` | masked cards, no duplicates within a page; like/pass is **not** done here — cards link out to the matching actions (`POST /users/{targetId}/likes`) |
| `next_cursor` | string\|null | last card id of the page; `null` = deck exhausted (refill arrives via pool events) |

**Side effect:** each card the client reports as reached (`POST /users/{userId}/feed/seen`) emits `ProfileViewed { viewer_id, profile_id, source: "DECK" }` → `analytics` (powers "X women viewed your profile today"; profile-open views are emitted by the `users` module with `source: "PROFILE"`).

**Like / pass from a card:** the client calls the matching endpoints directly (`POST /users/{targetId}/likes`, §Likes) — the deck has no embedded action endpoints, and a liked/passed card never reappears (matching state excludes it from the pool view).

**Algorithm:** V1 uses **random shuffle** within the filtered eligible pool. Vector-similarity ranking (embedding bio/description for compatibility scoring) is a future enhancement.

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
- **ER synced:** `LIKE.status` includes `EXPIRED`; `FEED_IMPRESSION` (server-tracked feed position, §Discovery feed) and `FEED_CANDIDATE` (the eligible-men pool) are owned by this module's discovery half; `PROFILE_VIEW_EVENT.source` includes `DECK` — all reflected in `entities/er-diagram.md`.
- **Deferred (premium):** spotlight boost, advanced feed filters, like-delivery insights — future contract additions; the like/match/deck model above doesn't preclude them.

---

## Implementation notes — likes & matches slice (V1 backend)

Choices made where the contract is silent or ambiguous; the code follows these, so a change here is a change there.

- **"Strictly younger" = exact date of birth.** The target's DOB must be strictly after the liker's (same DOB fails). It is *not* a whole-year age comparison; the feed's `min_age`/`max_age` filters still use whole years.
- **Female target on like-creation → `404 USER_NOT_VISIBLE`** (the pool is men-only; same masking as a hidden user). The guard order is otherwise as specced, with the self check before the gender check so a self-like reports `CANNOT_LIKE_SELF`.
- **A woman with an `ACTIVE` match to the target who likes again → `409 LIKE_EXISTS`** (one active match per pair).
- **Expiry is enforced lazily as well as hourly.** Accept/pass treat a `PENDING` like past `expires_at` as `409 LIKE_NOT_PENDING`; like-creation first flips that pair's overdue likes to `EXPIRED` (so the 30-day cooldown anchors on `expires_at`). `LIKE.resolved_at` (see ER) records pass time / expiry instant for the cooldown; unmatch time anchors on `MATCH.unmatched_at`.
- **Pass keeps the spec'd `PAIR_INELIGIBLE` check** (blocked, or either party not `ACTIVE`). A blocked pending like therefore cannot be passed and simply expires.
- **Inbox hides likes from senders the man can no longer see** (blocked, deactivated); the page `total` still counts them. **Match `counterpart` is `null`** when the other side is no longer visible (the client renders "User no longer exists").
- **Onboarding gate (`onb=false` → `403 ONBOARDING_INCOMPLETE`)** is enforced in the matching controllers (it was not enforced anywhere before). Owner check: `oid` vs path `userId`, with the moderator identity (`Roles.isModerator`) bypassing the owner check only.
- **Paging:** `page` ≥ 0, `size` 1..`fold.matching.max-page-size` (default 20 / max 50), else `422 VALIDATION_ERROR` with field errors.
- **Unconvertible query/path values** (e.g. `?status=FOO`) → `422 VALIDATION_ERROR` with a `field_errors` entry naming the parameter. This is a global behavior of the shared exception handler (previously `400 INVALID_REQUEST_BODY`), so it applies to every module's endpoints.
- **Config:** `fold.matching.like-ttl` (48h), `relike-cooldown` (30d), `expiry-sweep-interval` (1h), `default-page-size`, `max-page-size`.
- **Not in this slice:** the `messaging` consumer that backfills `conversation_id`. (The discovery feed is covered in the next section; the `LikeReceived`/`MatchCreated` notifications are documented in `notifications.md`.)

---

## Implementation notes — discovery feed (V1 backend)

- **Pool = event-fed projection (`FEED_CANDIDATE`)** in `matching`. Change signals (`UserRegistered`, `UserDeactivated`, `UserSuspended`, `UserShadowBanned`, `UserReactivated`, `AdmittedFromQueue`, and the new `ProfileUpdated`) carry only the user id; the handler re-reads the user's current facts via `UserClient.findFeedProfile` and upserts or removes the row (eligible = `ACTIVE` man). It never depends on event payload completeness or ordering. `PhotoChanged` is not consumed — cards are read live. A reconcile job runs at startup (backfilling existing users) and every `fold.matching.pool-reconcile-interval` (1h) to self-heal drift.
- **`ProfileUpdated { user_id }`** is new, published on profile edits and on account edits (location).
- **Blocks are not mirrored** into the pool: each fresh page makes one bulk `UserClient.getBlockedUserIds` call, so a block takes effect immediately.
- **Served is not seen.** A fresh batch is queued in `FEED_IMPRESSION` with `seen_at` null and fires nothing. Her position is the last card with `seen_at` set.
- **No `cursor`** → resume: the queued cards after her last seen card (skipping any that became unavailable — blocked, liked, matched, deactivated), up to `limit`; if there are none, a fresh random batch of men she hasn't been shown is picked and queued. `next_cursor` is the last returned card's id; `null` means nothing is queued and nothing fresh is left (deck exhausted).
- **With `cursor`** (a card id) → the same, but "after that card" instead of "after her last seen card": already-queued cards first, else a fresh batch. This is the client's prefetch/re-scroll call and never changes what she has seen. Unknown cursor → `422 VALIDATION_ERROR` on `cursor`.
- **New endpoint (contract addition): `POST /users/{userId}/feed/seen`** with body `{ "last_seen_profile_id": "<card id>" }` → `204`. Marks every queued card up to and including that one as seen and emits `ProfileViewed` once for each newly seen card. Idempotent and tolerant of reordering (a repeat or an older card changes nothing). Women only (`403 FORBIDDEN` otherwise); owner check and onboarding gate as elsewhere; blank id → `422`; a card never served to her → `422 VALIDATION_ERROR` on `last_seen_profile_id`. The client should call it as she advances (debounced) and when she leaves the screen; a lost call only delays the event, it never loses her place.
- **Hidden from her deck:** blocked either way; men with a `PENDING` like; `ACTIVE` matches; likes `PASSED`/`EXPIRED` within `fold.matching.refeed-after-pass` (7d); matches `UNMATCHED` within `relike-cooldown` (30d); cards already shown within `fold.matching.feed-reshow-after` (30d), after which they may appear again.
- **Rules and filters:** the structural rule uses exact DOB (strictly after her DOB); `min_age`/`max_age` are whole years. `422 VALIDATION_ERROR` (field errors in the query-param names) for: `limit` outside 1..`max-page-size`, non-positive numbers, `min_*` above `max_*`, or `max_age` ≥ her age. `religion`/`education_level` are comma lists, case-insensitive; an unknown value simply matches nobody. Men missing a filtered field are excluded.
- **Distance:** a bounding box narrows the SQL query, then an exact Haversine check runs in Java (`distance_km`). The candidate set is loaded in memory per fresh page — fine at V1 scale, to be revisited if the pool grows large.
- **Stale pool rows never shrink a page.** The pool is updated asynchronously, so a just-deactivated man can briefly still be a candidate. A fresh batch keeps drawing from the shuffled pool until it has `limit` visible cards or runs out, and refreshes (so removes) each stale candidate it meets. An empty page therefore always means a genuinely exhausted deck.
- **`ProfileViewed { viewer_id, profile_id, source: DECK }`** is emitted once per card when it becomes seen (via `.../feed/seen`), never when it is merely served. Nothing consumes it until `analytics` exists.
- **Retries are safe:** a `GET` that fails in flight loses nothing — the unseen cards stay queued and the next call returns them again. Only a fresh batch is queued by a call, and only when none are left.
- **Config:** `fold.matching.feed-reshow-after` (30d), `refeed-after-pass` (7d), `pool-reconcile-interval` (1h).