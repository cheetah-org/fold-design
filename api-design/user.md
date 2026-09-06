# User Service — API

Base URL: `https://api.wiingman.in/users/api/v1` — gateway convention `{baseURL}/{service}/api/v1/{resource}`; all paths below are relative to this base (e.g. `GET https://api.wiingman.in/users/api/v1/users/{userId}`) · Content-Type: `application/json` · Dates: ISO-8601 UTC · IDs: UUID

**Auth:** `Authorization: Bearer <access_token>` — JWT issued by the Auth Service. The users module reads only the token claims it needs (`sub` = `AUTH_CREDENTIAL.id`, `oid` = `USER.id`, `onb`, `roles`) — see `auth.md` §1.1 for the full token contract. **Every endpoint below requires a valid Bearer token** unless stated otherwise.

**Roles:** `roles` (array claim) — `USER` default, `DEV`/`ADMIN` granted by the Auth Service allowlist. Role is sourced and minted by Auth; this service only enforces it. A role change is picked up on the next token issue (`POST /auth/refresh` or re-login); in-flight access tokens keep the old role until `exp` (≤ 15 min).

**Resources** addressed by `userId` (`USER.id`, the `oid` claim) — no `/users/me`. Client learns its own `userId` from `POST /users` (and from `POST /auth/refresh` once `oid` appears); a fresh (`onb=false`) client has **no** `userId` yet.

**Owner check:** `userId` from `oid` claim is compared to the path segment. A caller whose token has `onb=false` (un-onboarded) is rejected with `403 ONBOARDING_INCOMPLETE` on every endpoint except `POST /users` (and auth's own refresh/logout) — never a generic `FORBIDDEN`; see `auth.md` §2. An absent/unknown `oid` with `onb=true` is a server-side minting defect → `401 TOKEN_INVALID_CLAIMS` (review fix — `TOKEN_MALFORMED` is reserved for structural failures).

---

## Authorization matrix

| Capability | `USER` | `DEV` | `ADMIN` |
|---|---|---|---|
| `POST /users` (onboarding, `onb=false` token) | ✅ | ✅ | ✅ |
| Own `/users/{userId}/...` (incl. full read) | ✅ | ✅ | ✅ |
| Other users' full read, PATCH, DELETE, profile, photos | ❌ 403 | ✅ | ✅ |
| `GET /users/{userId}` other → masked | ✅ | full | full |
| File report · block / unblock | ✅ | ✅ | ✅ |
| `GET /users/{userId}/reports/filed` | own | ✅ | ✅ |
| Reports against (`GET`, detail, review) | ❌ | ✅ | ✅ |
| `POST /users/{userId}/suspend` · `/reactivate` | ❌ | ✅ | ✅ |
| `GET /users` (search + review queue) | ❌ 403 | ✅ | ✅ |

**Visibility:** non-owner, non-privileged `GET /users/{userId}` of a blocked (either direction), `SUSPENDED`, `SHADOW_BANNED`, `DEACTIVATED`, or `QUEUED` user → `404 USER_NOT_VISIBLE` (self never masked — see §GET /users/{userId}). The same single response is returned for "no row" (un-onboarded credential), unknown id, and hidden users — callers can't distinguish.

**DTO rules:** this service **never exposes `email`** — it is owned by the Auth Service (`AUTH_CREDENTIAL`, surfaced via `GET /auth/me`). Full `UserDto` exposes `dob` only to self/DEV/ADMIN; public views expose only computed `age`. `priority`, `notes`, `reviewed_by` are moderation-internal.

**Location model:** user provides a location via a places API (OpenStreetMap/Google Places). Backend stores `location_name` (display string, e.g. "BTM Layout, Bangalore"), `latitude`, and `longitude`. Region (e.g. "Bangalore metro") is derived server-side from the coordinates for ratio and analytics aggregation. `PublicUserDto` exposes only `location_name` — coordinates are never shared with other users.

**Limits:** max 6 photos (min 1 at onboarding) · `bio` ≤ 500 · `description` ≤ 2000 · `username` ≤ 30 · age gate 21+ · reports ≤ 5/reporter/day · pagination default `page=0, size=20`. **Rate limiting:** cross-cutting, handled centrally later — no per-endpoint limits in this doc.

**Auth failures (global):** any endpoint requiring a Bearer token returns the shared Auth Lib set (`auth.md` §2) — `401 TOKEN_MISSING` / `TOKEN_MALFORMED` / `TOKEN_EXPIRED` / `TOKEN_INVALID_SIGNATURE` / `TOKEN_INVALID_AUDIENCE` / `TOKEN_UNKNOWN_KID`, or `429 TOO_MANY_REQUESTS`. These are not repeated in per-endpoint tables below.

---

## Error

Canonical envelope (shared across all modules — see `auth.md` §2). `field_errors` only on validation/body errors.

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "location is required",
    "traceId": "8f2c1a3d-4b5e-4c6f-9a0d-1e2f3a4b5c6d",
    "field_errors": [{ "field": "location.name", "message": "required" }]
  }
}
```

---

## Admission & queue rules

Product invariants that drive registration, the admission endpoint, and reactivation:

- **Who queues:** men only. Women are always admitted immediately (the product keeps `active_women ≥ active_men`).
- **Ratio measured at:** region level (`RATIO_STATE.active_women` vs `active_men`, Bangalore metro for V1). Region is derived server-side from the user's geolocation coordinates. `DEACTIVATED`, `SUSPENDED`, `SHADOW_BANNED`, and `QUEUED` users are **not** counted as active for either side.
- **Gate:** `computed_open = active_women >= active_men` — computed by `ratio` (which holds any manual override); never a stored field (`admission_open` dropped from `RATIO_STATE`).
  - Woman registering → `ACTIVE`.
  - Man registering + gate open → `ACTIVE`.
  - Man registering + gate closed → `QUEUED`, FIFO by onboarding time; `rank` = number of earlier queued men + 1.
- **Rank / wait estimate:** `estimated_wait_ms` = `distinct_active_women_churn_rate`-based heuristic: `pending_ahead × rolling_avg_admission_interval`, floor 5 min, computed by the `ratio` module and stored with the queue snapshot. Exact algorithm is a build-time decision.
- **Promotion:** the `ratio` module evaluates on every deactivation event and on a 5-min cadence. While the gate is open it admits from the front (FIFO), emitting `AdmittedFromQueue` → users module flips `status=ACTIVE`, `notifications` pushes the admission ("You're in").
- **Profile editing is allowed while `QUEUED`** (men complete profiles while waiting); it is blocked only for `DEACTIVATED` (`404`).

---

## DTOs

`UserDto` (full) · `PublicUserDto` (masked)

```json
{
  "id": "9f3a1c4e-2b7d-4a81-9c5e-8d2f0a6b3c11",
  "name": "Rohan",
  "username": "rohan_m",
  "gender": "MALE",
  "dob": "1998-09-02",
  "age": 28,
  "location_name": "BTM Layout, Bangalore",
  "status": "QUEUED",
  "created_at": "2026-08-29T10:00:00Z"
}
```

```json
{
  "id": "9f3a1c4e-2b7d-4a81-9c5e-8d2f0a6b3c11",
  "name": "Rohan",
  "username": "rohan_m",
  "age": 28,
  "gender": "MALE",
  "location_name": "BTM Layout, Bangalore",
  "bio": "I make coffee",
  "description": "Looking to meet someone older.",
  "height": 175,
  "job_title": "Software Engineer",
  "company": "Flipkart",
  "school": "Christ University",
  "education_level": "UNDERGRAD",
  "hometown": "Mumbai",
  "religion": "HINDU",
  "languages": ["English", "Hindi", "Kannada"],
  "drinking": "SOMETIMES",
  "smoking": "NEVER",
  "photos": [
    { "id": "b7e2f9aa-1c3d-4e5f-8a6b-9c0d1e2f3a4b", "position": 0, "url": "...", "prompt": "Golden hour" }
  ]
}
```

`status`: `ACTIVE` | `QUEUED` | `SUSPENDED` | `SHADOW_BANNED` | `DEACTIVATED`

All profile fields except `bio`, `description`, and `photos` are nullable — omitted when the user hasn't set them.

`ProfileDto` · `PhotoDto` · `AdmissionDto`

```json
{
  "bio": "I make coffee",
  "description": "Looking to meet someone older.",
  "height": 175,
  "job_title": "Software Engineer",
  "company": "Flipkart",
  "school": "Christ University",
  "education_level": "UNDERGRAD",
  "hometown": "Mumbai",
  "religion": "HINDU",
  "languages": ["English", "Hindi", "Kannada"],
  "drinking": "SOMETIMES",
  "smoking": "NEVER",
  "photos": [{ "id": "b7e2f9aa-1c3d-4e5f-8a6b-9c0d1e2f3a4b", "position": 0, "url": "...", "prompt": "Golden hour" }]
}
```

```json
{ "id": "b7e2f9aa-1c3d-4e5f-8a6b-9c0d1e2f3a4b", "position": 0, "url": "...", "prompt": "Golden hour" }
```

```json
{
  "state": "ACTIVE",
  "queue": null
}
```

`state`: `ACTIVE` | `QUEUED` | `DEACTIVATED`; `queue` present only when `QUEUED`:

```json
{
  "state": "QUEUED",
  "queue": { "rank": 3, "total_waiting": 12, "estimated_wait_ms": 3600000 }
}
```

`BlockDto` · `BlockTx`

```json
{ "id": "d6b2e4f8-5a7b-4c8d-9e0f-1a2b3c4d5e6f", "blocked_id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d", "created_at": "2026-08-29T09:15:00Z" }
```

```json
{ "blocked_id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d", "created_at": "2026-08-29T09:15:00Z" }
```

`FiledReportDto` (reporter's view) — `ReportDto` (moderation view)

```json
{
  "id": "c5a1d3e7-4f6a-4b2c-9d8e-1f0a2b3c4d5e",
  "reported_id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d",
  "category": "UNDERAGE",
  "status": "PENDING",
  "notes": "Photos look younger",
  "created_at": "2026-08-29T09:00:00Z",
  "reviewed_at": null
}
```

`category`: `HARASSMENT` | `FAKE_PROFILE` | `UNDERAGE` | `INAPPROPRIATE` | `BLOCK` · `status`: `PENDING` | `NOT_UPHELD` | `UPHELD`

```json
{
  "id": "c5a1d3e7-4f6a-4b2c-9d8e-1f0a2b3c4d5e",
  "category": "UNDERAGE",
  "priority": "HIGH",
  "status": "PENDING",
  "notes": "Photos look younger",
  "created_at": "2026-08-29T09:00:00Z",
  "reviewed_at": null,
  "reviewed_by": null,
  "reporter": { "id": "3f8c2a11-5b6d-4c7e-9f0a-b1c2d3e4f5a6", "name": "Priya" }
}
```

`PagedDto<T>` · `UserStatusResponse`

```json
{ "items": [], "page": 0, "size": 20, "total": 0 }
```

```json
{ "user_id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d", "status": "SUSPENDED" }
```

---

## Register

### `POST /users`
**Onboarding — auth's identity becomes a `USER`.** Completes the two-step bootstrap: Auth upserts `AUTH_CREDENTIAL` (login) → this endpoint creates the `USER` + empty `PROFILE` linked to the credential via `credential_id` (plain-ID reference, modulith rule 7). Gender/location drive ratio admission (`Admission & queue rules`) → 201 `status` `ACTIVE` or `QUEUED` + `queue`. After this call the client runs `POST /auth/refresh` once so subsequent access tokens carry the new `oid` (`auth.md` §3/Appendix A).

**Re-registration after self-deactivation (review fix):** a `DEACTIVATED` credential signing in again gets `onboarded=false` from auth (`auth.md` §3) and may call `POST /users` to create a **fresh** `USER` row — the deactivated row is retained for audit, one live row per credential. This closes the previous dead-end where `DELETE /users/{id}` was irreversible.

**Auth:** Bearer token with `onb=false` (fresh credential). An `onb=true` principal is rejected (409).

**Request**
```json
{
  "name": "Rohan",
  "username": "rohan_m",
  "gender": "MALE",
  "location": {
    "name": "BTM Layout, Bangalore",
    "latitude": 12.9166,
    "longitude": 77.6101
  },
  "dob": "1998-09-02",
  "photos": [
    { "storage_path": "users/9f3a1c4e-2b7d-4a81-9c5e-8d2f0a6b3c11/photos/abc.jpg", "prompt": null }
  ]
}
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `name` | string | ✅ | from the Google profile (the ID token carries `name`; our access token doesn't, so the client passes it here); ≤ 64 chars |
| `username` | string | ✅ | display handle, not unique; ≤ 30 chars |
| `gender` | `FEMALE` \| `MALE` | ✅ | drives ratio admission; immutable later |
| `location.name` | string | ✅ | display name from places API (e.g. "BTM Layout, Bangalore"); ≤ 128 chars |
| `location.latitude` | double | ✅ | WGS 84 latitude; server derives region for ratio |
| `location.longitude` | double | ✅ | WGS 84 longitude |
| `dob` | date | ✅ | **always required** — Google/Firebase ID tokens never carry DOB (review fix), so the short form collects it; 21+ gate enforced server-side |
| `photos` | array | ✅ | 1–6 photo references; client uploads to Firebase Storage first, then passes `storage_path` + optional `prompt` |

> **Transport note (review fix):** `name`/`dob` were previously assumed to arrive "from Google" with no transport. Reality: the Google ID token carries `name` + `email` but **never** `dob` (`auth.md` §3). `email` stays auth-owned; `name` + `dob` are collected by the short form and transported here.

**Response 201**
```json
{
  "user": {
    "id": "9f3a1c4e-2b7d-4a81-9c5e-8d2f0a6b3c11",
    "name": "Rohan",
    "username": "rohan_m",
    "gender": "MALE",
    "dob": "1998-09-02",
    "age": 28,
    "location_name": "BTM Layout, Bangalore",
    "status": "QUEUED",
    "created_at": "2026-08-29T10:00:00Z"
  },
  "queue": { "rank": 3, "total_waiting": 12, "estimated_wait_ms": 3600000 },
  "profile": { "bio": null, "description": null, "photos": [{ "id": "b7e2f9aa-...", "position": 0, "url": "...", "prompt": null }] }
}
```
`queue` is `null` when `status` is `ACTIVE`. Profile fields not set at onboarding (height, job_title, etc.) are omitted from the response.

| Status | Code |
|---|---|
| 409 | `USER_EXISTS` — a non-`DEACTIVATED` `USER` row already exists for this credential |
| 422 | `VALIDATION_ERROR` · `AGE_VERIFICATION_REQUIRED` · `AGE_GATE_BLOCKED` |

---

## Account

### `GET /users/{userId}`
Self / DEV / ADMIN → 200 full `AccountResponse`; any other `USER` → 200 `PublicUserDto`; anything else → `404 USER_NOT_VISIBLE` (hidden **or** no row — un-onboarded credentials have no `userId` to target, see header notes). **Masking never applies to self** (review fix): a `QUEUED`/`SHADOW_BANNED` caller always gets their own full `AccountResponse`.

**Response 200 — full**
```json
{
  "user": {
    "id": "9f3a1c4e-2b7d-4a81-9c5e-8d2f0a6b3c11",
    "name": "Rohan",
    "username": "rohan_m",
    "gender": "MALE",
    "dob": "1998-09-02",
    "age": 28,
    "location_name": "BTM Layout, Bangalore",
    "latitude": 12.9166,
    "longitude": 77.6101,
    "status": "QUEUED",
    "created_at": "2026-08-29T10:00:00Z"
  },
  "admission": {
    "state": "QUEUED",
    "queue": { "rank": 3, "total_waiting": 12, "estimated_wait_ms": 3600000 }
  },
  "profile": {
    "bio": "I make coffee",
    "description": "Looking to meet someone older.",
    "height": 175,
    "job_title": "Software Engineer",
    "company": "Flipkart",
    "school": "Christ University",
    "education_level": "UNDERGRAD",
    "hometown": "Mumbai",
    "religion": "HINDU",
    "languages": ["English", "Hindi", "Kannada"],
    "drinking": "SOMETIMES",
    "smoking": "NEVER",
    "photos": [{ "id": "b7e2f9aa-1c3d-4e5f-8a6b-9c0d1e2f3a4b", "position": 0, "url": "...", "prompt": "Golden hour" }]
  }
}
```

| Status | Code |
|---|---|
| 404 | `USER_NOT_VISIBLE` — blocked (either direction), `SUSPENDED`/`SHADOW_BANNED`/`DEACTIVATED`/`QUEUED`, or no row |

### `PATCH /users/{userId}`
Self / DEV / ADMIN. Mutable: `location`, `username` (`gender`/`dob`/`name` immutable — Google-sourced / ratio-governing). Location change triggers region re-derivation and ratio re-evaluation. → 200 `UserDto`.

**Request**
```json
{
  "username": "rohan_dev",
  "location": {
    "name": "Whitefield, Bangalore",
    "latitude": 12.9698,
    "longitude": 77.7500
  }
}
```

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — not owner / not privileged |
| 404 | `NOT_FOUND` — user doesn't exist |
| 422 | `IMMUTABLE_FIELD` — `gender`/`dob`/`name` · `VALIDATION_ERROR` |

### `DELETE /users/{userId}`
Self / DEV / ADMIN. Soft-deactivate → `status=DEACTIVATED` (row kept for audit; publishes `UserDeactivated` → ratio recompute, messaging's "User no longer exists", notifications stop). → `204`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |

---

## Admission & queue

### `GET /users/{userId}/admission`
Self / DEV / ADMIN. Live rank/wait for `QUEUED` men → 200 `AdmissionDto`. Not queued → `state=ACTIVE`, `queue=null`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `USER_NOT_VISIBLE` — `DEACTIVATED` or no row |

---

## Profile & photos

Owner-only (DEV/ADMIN bypass). All → `403 FORBIDDEN` for non-owners. User must exist → `404 NOT_FOUND`. Editing is allowed for `QUEUED` users; `DEACTIVATED` → `404`.

### `PATCH /users/{userId}/profile`
Partial — only supplied fields are updated; omitted fields are untouched. Passing `null` clears a nullable field. → 200 `ProfileDto`.

**Request**
```json
{
  "bio": "I make coffee",
  "description": "Looking to meet someone older.",
  "height": 175,
  "job_title": "Software Engineer",
  "company": "Flipkart",
  "school": "Christ University",
  "education_level": "UNDERGRAD",
  "hometown": "Mumbai",
  "religion": "HINDU",
  "languages": ["English", "Hindi", "Kannada"],
  "drinking": "SOMETIMES",
  "smoking": "NEVER"
}
```

| Field | Type | Notes |
|---|---|---|
| `bio` | string | ≤ 500 chars |
| `description` | string | ≤ 2000 chars |
| `height` | integer | cm |
| `job_title` | string | freetext, ≤ 64 chars |
| `company` | string | freetext, ≤ 64 chars |
| `school` | string | freetext, ≤ 128 chars |
| `education_level` | enum | `HIGH_SCHOOL` \| `UNDERGRAD` \| `POSTGRAD` \| `TRADE_SCHOOL` |
| `hometown` | string | freetext, ≤ 128 chars |
| `religion` | enum | `HINDU` \| `MUSLIM` \| `CHRISTIAN` \| `CATHOLIC` \| `SIKH` \| `BUDDHIST` \| `JAIN` \| `SPIRITUAL` \| `AGNOSTIC` \| `ATHEIST` \| `OTHER` |
| `languages` | string[] | multi-select from predefined list |
| `drinking` | enum | `YES` \| `SOMETIMES` \| `NEVER` |
| `smoking` | enum | `YES` \| `SOMETIMES` \| `NEVER` |

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 422 | `VALIDATION_ERROR` — field length exceeded · invalid enum value · wrong types |

### `POST /users/{userId}/photos`
After direct-to-Firebase-Storage upload, registers metadata. `storage_path` must be under `users/<uid>/` where `<uid> == userId`. → 201 `PhotoDto`.

**Request**
```json
{ "storage_path": "users/9f3a1c4e-2b7d-4a81-9c5e-8d2f0a6b3c11/photos/abc.jpg", "prompt": null }
```

| Status | Code |
|---|---|
| 400 | `INVALID_STORAGE_PATH` — not under the owner's namespace |
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 409 | `PHOTO_EXISTS` — duplicate `storage_path` |
| 422 | `PHOTO_LIMIT` — already 6 · `VALIDATION_ERROR` |

### `PATCH /users/{userId}/photos/{photoId}`
Only `prompt` mutable (reorder has the bulk endpoint). → 200 `PhotoDto`.

**Request**
```json
{ "prompt": "Golden hour" }
```

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` — user or photo |
| 422 | `VALIDATION_ERROR` |

### `PUT /users/{userId}/photos/order`
Exact permutation of existing photo ids (no add/remove). → 200 `PhotoDto[]`.

**Request**
```json
{ "ordered_ids": ["b7e2f9aa-1c3d-4e5f-8a6b-9c0d1e2f3a4b", "8d3f0b2c-4e5a-4b6d-9f7c-0a1b2c3d4e5f"] }
```

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 422 | `INVALID_ORDER` — not an exact permutation |

### `DELETE /users/{userId}/photos/{photoId}`
Removes metadata; Firebase Storage object deleted async. → `204`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |

---

## Blocking

### `POST /users/{blockedUserId}/block`
Creates `BLOCK` + linked audit `REPORT(category=BLOCK, status=UPHELD)`. **Block reports are exempt from all moderation effects** (explicitly): they never trigger suspension, never enter the review queue (`reports_status=PENDING` ignores them), can't be reviewed (`409 NOT_REVIEWABLE`), and don't count toward the reporter's false-report threshold. Masking applies immediately. → 201 `BlockResponse`; already blocked → 200 (idempotent).

**Response 201**
```json
{
  "block": {
    "id": "d6b2e4f8-5a7b-4c8d-9e0f-1a2b3c4d5e6f",
    "blocked_id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d",
    "created_at": "2026-08-29T09:15:00Z"
  },
  "report_id": "e7c3f5a9-6b8c-4d9e-0f1a-2b3c4d5e6f7a"
}
```

| Status | Code |
|---|---|
| 404 | `NOT_FOUND` — target user missing |
| 422 | `CANNOT_BLOCK_SELF` · `VALIDATION_ERROR` |

### `DELETE /users/{blockedUserId}/block`
Unblock → `204` — **always idempotent**: 204 whether or not a block exists (anti-enumeration, same semantics as `auth.md` logout; review fix — previously contradicted itself by 404ing on never-blocked). The linked audit report is kept as history; masking stops.

### `GET /users/{userId}/blocks`
Self / DEV / ADMIN. Blocks **made by** this user. `?page=&size=` → 200 `PagedDto<BlockTx>`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |

---

## Reporting

**Shadow-ban policy (anti-abuse, explicit rules):** a filed report records a `PENDING` report and may shadow-ban the target:

- `UNDERAGE` → **immediate** shadow-ban + `priority=HIGH`.
- All other categories → shadow-ban only once **≥ 2 distinct reporters** have `PENDING` reports against the same target within a 48 h lookback (counted from `REPORT.created_at` rows, not the request limiter; review nit). Below that, the report stays `PENDING` with no status change.
- A target already `SHADOW_BANNED` (or `SUSPENDED`/`DEACTIVATED`) is **not** re-banned by further reports.
- **Per-reporter rate cap:** max 5 report submissions per reporter per 24 h lookback (counted from `REPORT.created_at` rows, not the request limiter; review nit) → `429 REPORT_LIMIT_REACHED`.

### `POST /users/{reportedUserId}/reports`
File a report. Applies the shadow-ban policy above. `UNDERAGE` → `priority=HIGH`. → 201 `FiledReportDto`. **Visibility carve-out (review fix):** reporting is allowed regardless of block/visibility state — block-then-report must work, so `USER_NOT_VISIBLE` never gates this endpoint (404 only for a missing or `DEACTIVATED` target).

**Request**
```json
{ "category": "UNDERAGE", "notes": "Photos look younger" }
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `category` | `HARASSMENT` \| `FAKE_PROFILE` \| `UNDERAGE` \| `INAPPROPRIATE` | ✅ | `BLOCK` is set only by the block flow |
| `notes` | string | — | ≤ 2000 chars |

| Status | Code |
|---|---|
| 404 | `NOT_FOUND` — target missing or `DEACTIVATED` (visibility never gates reporting) |
| 409 | `REPORT_EXISTS` — duplicate `PENDING` (same reporter→reported) |
| 422 | `REPORT_SELF` · `VALIDATION_ERROR` |
| 429 | `REPORT_LIMIT_REACHED` — > 5 reports in 24 h |

### `GET /users/{userId}/reports/filed`
Self / DEV / ADMIN. Reports **filed by** this user. `?status=&page=&size=` → 200 `PagedDto<FiledReportDto>`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |

### `GET /users/{userId}/reports`
DEV / ADMIN. Reports **against** this user — `BLOCK`-category history included, `priority=HIGH` first. `?status=&page=&size=` (default `status=PENDING`) → 200 `PagedDto<ReportDto>`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |

### `GET /users/{userId}/reports/{reportId}`
DEV / ADMIN. → 200 `ReportDto`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |

### `PATCH /users/{userId}/reports/{reportId}`
DEV / ADMIN — review (only `PENDING`, non-`BLOCK` reports). → 200 `ReviewResponse`.

**Request**
```json
{ "decision": "UPHELD", "notes": "moderation memo" }
```

**Review outcomes (explicit):**
- `UPHELD` → reported user permanently `SUSPENDED`.
- `NOT_UPHELD` → **restoration recompute, not a stored snapshot**: the change to `ACTIVE`/`QUEUED` is applied **only if** the user's status is still `SHADOW_BANNED` at review time; the restored status is ratio-aware (women → `ACTIVE`; men → per `Admission & queue rules`). If the user was re-suspended or otherwise moved in between, the intervening status wins and restoration is a no-op. (A stored `status_before_shadow_ban` field was considered and rejected — it goes stale across intervening moderation actions.)
- **False-reporter rule:** reporter with ≥ 3 `NOT_UPHELD` outcomes across their reviews → also moved to `SUSPENDED` (`BLOCK` reports never count).

**Response 200**
```json
{
  "report": {
    "id": "c5a1d3e7-4f6a-4b2c-9d8e-1f0a2b3c4d5e",
    "category": "UNDERAGE",
    "priority": "HIGH",
    "status": "UPHELD",
    "notes": "moderation memo",
    "created_at": "2026-08-29T09:00:00Z",
    "reviewed_at": "2026-08-29T11:00:00Z",
    "reviewed_by": "8a4f2c3d-5e6f-4a7b-8c9d-0e1f2a3b4c5d"
  },
  "reported_user_status": "SUSPENDED",
  "reporter_status": "ACTIVE"
}
```

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 409 | `ALREADY_REVIEWED` — not `PENDING` · `NOT_REVIEWABLE` — `BLOCK`-category / auto-generated |
| 422 | `VALIDATION_ERROR` — bad `decision` |

---

## Moderation

### `POST /users/{userId}/suspend`
DEV / ADMIN. Immediate `status=SUSPENDED` (publishes `UserSuspended`). → 200 `UserStatusResponse`.

**Request**
```json
{ "reason": "TOS violation" }
```

| Field | Type | Req | Notes |
|---|---|---|---|
| `reason` | string | ✅ | ≤ 500 chars, recorded for audit |

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 409 | `INVALID_STATE` — already `SUSPENDED` |
| 422 | `VALIDATION_ERROR` |

### `POST /users/{userId}/reactivate`
DEV / ADMIN. Reverse a suspend → restoration recompute (§ Reporting): ratio-aware — a man returns to `ACTIVE`/`QUEUED` per the gate. Publishes `UserReactivated`. → 200 `UserStatusResponse`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |
| 404 | `NOT_FOUND` |
| 409 | `INVALID_STATE` — not `SUSPENDED` |

### `GET /users`
DEV / ADMIN. Search + review queue. `?q=&status=&gender=&reports_status=PENDING&page=&size=` — `reports_status=PENDING` returns users with ≥ 1 reviewable (`PENDING`, non-`BLOCK`) report against them, highest `priority` first. → 200 `PagedDto<UserDto>`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` |

---

## Rate limiting

Rate limiting is a cross-cutting concern handled by shared infrastructure, not implemented per-endpoint here — mechanism TBD (same placeholder policy as `ratio.md`/`analytics.md`).

---

## Cross-module notes

- **Identity:** users module consumes the token's `sub` (credential id), `oid` (user id), `onb`, `roles`. Full token contract, refresh rotation, and logout semantics live in `auth.md` — not restated here.
- **Un-onboarded state:** a valid JWT with `onb=false` has **no** `USER` row and therefore **no** `userId`; such a client can only `POST /users` then refresh (`auth.md` Appendix A).
- **Deactivation:** `DELETE /users/{id}` publishes `UserDeactivated`; the `ratio`, `messaging`, and `notifications` modules react — this API documents only the trigger.
- **Role changes** take effect on next token issue (`POST /auth/refresh` / re-login); neither doc stores or mints roles here.