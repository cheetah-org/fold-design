# Notification Service — API

Base URL: `https://api.wiingman.in/notifications/api/v1` — gateway convention `{baseURL}/{service}/api/v1/{resource}`; all paths below are relative to this base (e.g. `POST https://api.wiingman.in/notifications/api/v1/users/{userId}/tokens`) · Content-Type: `application/json` · Dates: ISO-8601 UTC · IDs: UUID

**Module:** `notifications` in the Spring Boot modulith. Owns the inbox (`NOTIFICATION`), the device/FCM-token registry, and per-user preferences. **Delivery channels:** push via **FCM** (`firebase-admin`) and email via **`JavaMailSender`** (spring-boot-starter-mail, plain SMTP credentials). Dispatch is internal (event-driven workers); this document specs the **client-facing REST surface** plus the type↔channel contract.

**Auth:** `Authorization: Bearer <access_token>` — this service reads exactly the claims defined in `auth.md` §1.1: `oid` (user id), `sid` (session id), `dfp` (session device fingerprint), `onb` (onboarded), `email` (recipient for email-bound types), `roles`. **Every endpoint requires a valid Bearer token.**

**Un-onboarded gate:** a token with `onb=false` has **no** `userId`; all endpoints below reject it with `403 ONBOARDING_INCOMPLETE` (canonical code, `auth.md` §2 — never a generic `FORBIDDEN`). `onb` flips only after `POST /users` + `POST /auth/refresh` (`auth.md` §1.1).

**Owner check:** for `onb=true` callers, `oid` vs path segment; a non-owner/non-privileged caller → `403 FORBIDDEN` (roles per `user.md` Authorization matrix). `DEV`/`ADMIN` bypass owner checks.

**Resources** addressed by `userId` — no `/users/me`. URIs are user-first (same convention as `user.md`/`auth.md`).

---

## Channels & types (contract)

A notification **event creates one `NOTIFICATION` row** (appears in the inbox) and is dispatched to the channels its type supports; the row's `channels` records what was attempted. Push is the fast path; email is deliberate and low-volume.

**Inbox rows are created for every notification regardless of the user's preference toggles.** Preferences gate **channel dispatch only** (`push`/`email`) — they never control inbox visibility. Disabling every channel for a type still leaves its row in the inbox; read/unread is the only inbox control.

`channel`: `push` (FCM) | `email` (SMTP). `type`:

| Type | Meaning | Push | Email | Inbox | Priority / cadence |
|---|---|---|---|---|---|
| `LIKE` | a woman liked him (see Assumption below) | ✅ | — | ✅ | high — immediate |
| `MATCH` | mutual match, chat opened | ✅ | — | ✅ | high — immediate |
| `EXPIRY` | conversation expiring ("made plans yet?") | ✅ | ✅ | ✅ | high — T−2 h before last message expiry |
| `QUEUE_UPDATE` | admission push — "You're in" (from `AdmittedFromQueue`) | ✅ | — | ✅ | medium — on event |
| `WEEKLY_SUMMARY` | profile view stats ("viewed Y times this week") | — | ✅ | ✅ | low — weekly batch |
| `VIEW` | anonymous daily count ("X women viewed you today") | ✅ | — | ✅ | low — daily |

> **Assumption (explicit, not changed):** `LIKE` means *a woman liking a man* under this product's browse model (women browse → men receive). This implies a female→male direction and interacts with the gender-ratio admission/queue in `user.md` "Admission & queue rules". If the product ever introduces male→female likes or same-sex matching, `LIKE` semantics and its channels change — flagged here so it's a deliberate decision, not embedded only in table prose.
> `EMAIL` carries only `EXPIRY` + `WEEKLY_SUMMARY` by design (cost/deliverability); push carries the transient alerts.
> **ER synced:** `VIEW` on `NOTIFICATION.type`, `channels`/`read`/`sent_at` on `NOTIFICATION`, `channel` on `NOTIFICATION_PREFERENCE` (composite PK), `session_id`/`device_fingerprint`/`created_at` on `DEVICE_TOKEN` — all reflected in `entities/er-diagram.md`.

---

## Error

Canonical envelope (shared, `auth.md` §2); `field_errors` only on `VALIDATION_ERROR`. Auth failures return the shared Auth Lib set (`TOKEN_*`), and the un-onboarded case returns `403 ONBOARDING_INCOMPLETE` — none repeated per endpoint.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "fcm_token is required",
    "traceId": "8f2c1a3d-4b5e-4c6f-9a0d-1e2f3a4b5c6d",
    "field_errors": [{ "field": "fcm_token", "message": "required" }]
  }
}
```

---

## DTOs

`DeviceDto` — **full** device record (returned on create only; the raw `fcm_token` is echoed exactly once)

```json
{
  "id": "a41f0b2c-3d4e-4f5a-8b6c-7d8e9f0a1b2d",
  "platform": "android",
  "fcm_token": "fcm-eJz...",
  "created_at": "2026-08-29T10:00:00Z",
  "last_seen_at": "2026-08-30T09:58:00Z"
}
```

`DeviceListItemDto` — **list view** (raw `fcm_token` is a credential-like value, so it is masked — never serialized in a collection)

```json
{
  "id": "a41f0b2c-3d4e-4f5a-8b6c-7d8e9f0a1b2d",
  "platform": "android",
  "fcm_token_masked": "fcm-****9Jz3",
  "created_at": "2026-08-29T10:00:00Z",
  "last_seen_at": "2026-08-30T09:58:00Z"
}
```

`NotificationDto` — one inbox row

```json
{
  "id": "b52c1d3e-4f5a-4b6c-9c7d-8e9f0a1b2c3e",
  "type": "EXPIRY",
  "channels": ["push", "email"],
  "status": "SENT",
  "payload": {
    "conversation_id": "c63d2e4f-5a6b-4c7d-9d8e-9f0a1b2c3d4f",
    "deep_link": "fold://chat/c63d2e4f-5a6b-4c7d-9d8e-9f0a1b2c3d4f",
    "message": "Your conversation with Rohan is expiring. Made plans yet?"
  },
  "read": false,
  "created_at": "2026-08-30T07:00:00Z",
  "sent_at": "2026-08-30T07:00:01Z"
}
```

| Field | Type | Notes |
|---|---|---|
| `id` | string | UUID |
| `type` | enum | see Channels & types |
| `channels` | string[] | channels this row was dispatched to (dispatch result — unaffected by preference state at read time) |
| `status` | enum | `PENDING` \| `SENT` \| `FAILED` |
| `payload` | object | open; always carries `message` + `deep_link` when a target exists |
| `read` | boolean | inbox state; false until marked read |
| `created_at` / `sent_at` | ISO-8601 | `sent_at` null while `PENDING` |

`UnreadCountDto` · `NotificationPreferenceDto`

```json
{ "unread": 3 }
```

```json
{ "type": "EXPIRY", "channel": "email", "enabled": true }
```

`PagedDto<T>`

```json
{ "items": [], "page": 0, "size": 20, "total": 0 }
```

---

## Devices (FCM token registry)

A `Device` row is **tied to the auth session** that registered it — it stores `session_id` (from the token's `sid` claim) and `device_fingerprint` (from the token's `dfp` claim). Lifecycle:

- **Logout stops push:** auth emits `SessionRevoked { session_id, credential_id }` (§9.5) on logout/logout-all; this module deletes `Device` rows whose `session_id` matches. Delivery is at-least-once via Spring Modulith transactional events — a failed listener is retried, never dropped.
- **Re-login on the same device:** `POST /tokens` with a token whose `dfp` matches an existing row for the same `(userId, fingerprint)` replaces that row's `fcm_token` and also removes any *other* sessions' rows on the same fingerprint (one active token per install — **advisory** (review fix): the fingerprint is client-asserted, so this is best-effort dedup, not a security guarantee).

`Device` row: `id, user_id, session_id, platform, fcm_token, device_fingerprint, created_at, last_seen_at`.

### `GET /users/{userId}/tokens`
List of the caller's registered devices → discoverable `id`s for `DELETE`. Raw FCM tokens are **masked** (`DeviceListItemDto`).

`?page=&size=` → 200 `PagedDto<DeviceListItemDto>`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` (not owner) |
| 404 | `NOT_FOUND` — user missing |

### `POST /users/{userId}/tokens`
Upsert — create or replace the caller's token for this install. Replace-on-re-login is by `(userId, device_fingerprint)`: an existing row for the fingerprint gets its `fcm_token` overwritten and `last_seen_at` touched, and tokens of other sessions on the same fingerprint are removed.

**Request**
```json
{
  "fcm_token": "fcm-eJz...",
  "platform": "android",
  "device": { "fingerprint": "iid-8f2c9a..." }
}
```

| Field | Type | Req | Constraints |
|---|---|---|---|
| `fcm_token` | string | ✅ | FCM registration token; ≤ 4 KB |
| `platform` | `ios` \| `android` | ✅ | |
| `device.fingerprint` | string | ✅ | must equal the token's **`dfp`** claim — the fingerprint recorded at login (required there since the `dfp` fix, `auth.md` §3/§4), so `dfp` is always present |

**Validation:** `device.fingerprint` ≠ the verified token's `dfp` claim → **422 `DEVICE_MISMATCH`** (the presented token wasn't minted by a session for this install). `fcm_token` is echoed in the response exactly once.

**Response 200** (upsert — token already registered) · **201** (new) → `DeviceDto` (full, `fcm_token` included).

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` (not owner) |
| 404 | `NOT_FOUND` — user missing |
| 422 | `VALIDATION_ERROR` — missing/bad fields · `DEVICE_MISMATCH` — fingerprint ≠ `dfp` |

### `DELETE /users/{userId}/tokens/{tokenId}`
Explicit removal (uninstall / dev remediation) of a row discovered via `GET /tokens`. Logout-driven removal is automatic via `SessionRevoked`. → `204`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` (not owner) |
| 404 | `NOT_FOUND` — token id |

---

## Inbox

Every `NOTIFICATION` row is listable regardless of preferences (§ Channels & types); badge = rows with `read=false`.

### `GET /users/{userId}/notifications`
Paged inbox. `?type=&channel=&status=&read=&page=&size=&sort=` (default newest first). `read` filters `true|false`. → 200 `PagedDto<NotificationDto>`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` (not owner) |
| 404 | `NOT_FOUND` — user missing |

### `GET /users/{userId}/notifications/unread_count`
Badge polling. → 200 `UnreadCountDto`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` (not owner) |
| 404 | `NOT_FOUND` |

> Route ordering: literal `unread_count` and `read-all` are matched **before** `{notificationId}` (same rule as `user.md` `reports/filed`).

### `PATCH /users/{userId}/notifications/{notificationId}`
Set read state (the only mutable inbox field), applied idempotently. → 200 `NotificationDto`.

**Request**
```json
{ "read": true }
```

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` (not owner) |
| 404 | `NOT_FOUND` — notification or user |
| 422 | `VALIDATION_ERROR` |

### `POST /users/{userId}/notifications/read-all`
Mark every row of this user `read=true` (clear badge). → `204`.

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` (not owner) |
| 404 | `NOT_FOUND` |

---

## Preferences

Same matrix as before; defaults all-enabled. **Scope confirmed:** toggles affect dispatch only (see § Channels & types). `GET` returns the full matrix; `PATCH` accepts a partial set.

### `GET /users/{userId}/notification-preferences`
→ 200 `{ "preferences": [ { "type", "channel", "enabled" } ] }` (applicable pairs only — e.g. no `LIKE`/`email`).

**Response 200**
```json
{
  "preferences": [
    { "type": "LIKE", "channel": "push", "enabled": true },
    { "type": "EXPIRY", "channel": "email", "enabled": false },
    { "type": "WEEKLY_SUMMARY", "channel": "email", "enabled": true }
  ]
}
```

### `PATCH /users/{userId}/notification-preferences`
Partial update. `{ "preferences": [ ...to enable/disable... ] }` — entries must be pairs the type supports (`email` for `LIKE` → 422). → 200 full matrix (as `GET`).

| Status | Code |
|---|---|
| 403 | `ONBOARDING_INCOMPLETE` (onb=false) · `FORBIDDEN` (not owner) |
| 404 | `NOT_FOUND` |
| 422 | `VALIDATION_ERROR` — unknown `type`/`channel`, unsupported pair, blank `enabled` |

---

## Rate limiting

Rate limiting is a cross-cutting concern handled by shared infrastructure, not implemented per-endpoint here — mechanism TBD (same placeholder policy as `ratio.md`/`analytics.md`). Delivery side is **not** client-rate-limited (PENDING queue); email keeps its configurable daily cap (default 200/day, `WEEKLY_SUMMARY` batch runs off-peak) — that's a delivery budget, not rate limiting.

---

## Non-functional notes

### Idempotency
- `POST .../tokens` — upsert keyed by `(user_id, device_fingerprint)`; safe to retry.
- `PATCH .../notifications/{id}` (`read`) and `read-all` — idempotent.
- `PATCH .../notification-preferences` — last-write-wins per pair.

### Delivery status & retry
- Workers dispatch PENDING rows per channel; success → `SENT`; failure → retry with backoff (push ≤ 3 over 5 min; email ≤ 3 over 30 min), then `FAILED` + dead-letter log. `channels` records attempts.
- A push that fails on a permanently-invalid FCM token triggers async cleanup of the `Device` row.

### Consumed event — `SessionRevoked` (at-least-once)
Publish side: `auth.md` §9.5 — Spring Modulith **transactional event publication** (`@ApplicationModuleListener` + event-publication registry). Contract below is identical to auth's:

| Property | Value |
|---|---|
| Event | `SessionRevoked` |
| Payload | `{ "session_id", "credential_id" }` (snake_case JSON) |
| Delivery | at-least-once; the registry retries until the listener succeeds |
| Handling | delete `Device` rows where `session_id == session_id` |

A dropped event would leave a logged-out device receiving pushes — the transactional registry exists precisely to guarantee this handler runs.

### Cross-module notes / integration points
- **Claims used:** `oid`, `sid`, `dfp`, `onb`, `email`, `roles` — all spelled as in `auth.md` §1.1's authoritative claims table.
- **Recipient email:** token `email` claim for client-presented paths; for server-initiated email jobs (`EXPIRY`/`WEEKLY_SUMMARY`) resolved from a local `recipient_email` projection fed by auth's `CredentialUpserted { credential_id, email }` events (transactional, at-least-once — `auth.md` §9.5). No sync calls back to auth, per the modulith rule that modules never call auth. (Review fix: earlier text said "Auth Lib credential lookup", contradicting that rule.)
- **Un-onboarded gate:** `onb=false` → `403 ONBOARDING_INCOMPLETE` on every endpoint here (no `userId` exists yet).
- **Types are additive:** adding a type is a contract change (this table + matching/user DTOs), not a schema migration.
- **ER synced:** `VIEW`, `channels`, `read`, `sent_at` on `NOTIFICATION`; composite PK `(user_id, type, channel)` on `NOTIFICATION_PREFERENCE`; `session_id`/`device_fingerprint`/`created_at` on `DEVICE_TOKEN` — all reflected in `entities/er-diagram.md`.