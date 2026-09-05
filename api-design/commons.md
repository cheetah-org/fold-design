# Commons — Shared Kernel Contracts

**Module:** `commons` — the one shared library every module imports. **Depends on nothing**: no module imports, no business logic, no repositories, no controllers.

**Gateway convention (all services):** public URLs are `{baseURL}/{service}/api/v1/{resource}` — e.g. `https://api.wiingman.in/users/api/v1/users/{userId}`. Service prefixes: `/auth` · `/users` · `/matching` · `/messaging` · `/notifications` · `/ratio` · `/analytics`. Each service doc states its own base; paths inside it are relative to that base.

Base package: `commons` · Contracts are **additive-only** (no breaking renames — same rule as notification types) · Every DTO/exception/event in `api-design/*.md` must match its commons record 1:1 (docs = human contract, commons = code artifact).

**Dependency rule (locked):** everyone → commons, commons → ∅. Module-internal request/response models stay private to each module; commons DTOs are the only cross-module currency. The mapping internal↔commons happens inside each producer's adapter.

---

## Package structure

```
commons/
  client/                      # sync inter-module contracts — INTERFACE only
    users/UserClient.java
    ratio/RatioClient.java
    matching/MatchingClient.java        # added with the messaging doc
  events/                      # one record per published event, grouped by producer
    users/  auth/  matching/  ratio/  messaging/
  exceptions/                  # typed families + ErrorCode enum
  error/ErrorDto.java          # canonical envelope record (doc source: auth.md §2)
  web/PagedDto.java
    RequestContextFilter.java  # MDC filter — stamps traceId, requestId, userId, module (see observability.md)
  async/
    MdcTaskDecorator.java      # propagates MDC to async event listener threads
  security/                    # folded Auth Lib — JWT verification (see §Security)
```

---

## Client interfaces & impl wiring (locked)

**Interface in commons, impl in the producer.** Callers inject the commons interface; the producer owns the bean; Spring DI wires it. No module ever imports a producer.

| Rule | Value |
|---|---|
| Interface | `commons.client.<module>.<X>Client` — references **only** commons types |
| Impl | `<X>InProcessClient implements <X>Client` lives **in the producer module**, maps the producer's internal models ↔ commons DTOs, exposed as a `@Bean` |
| Why not impl-in-commons | An in-process impl must import the producer's API → `commons → producer`; since every module already imports commons (events, exceptions), that closes `commons → producer → commons` — a Modulith-verified cycle. DTO duplication does **not** avoid this: the cycle comes from the call direction, not the types. |
| After extraction | `Http<X>Client` **can** live in commons (real HTTP needs no producer import) — rebind via config, callers unchanged (§6 swap plan, inter-service-communication.md) |
| Modulith rules kept | public-API-only access · own-tables-only DB · acyclic deps · events for async |

```
commons (interfaces + DTOs + events + exceptions + security)   ← depends on NOTHING
      ▲                    ▲                ▲
      └────── every module imports commons only ──────┘
users owns InProcessUserClient @Bean; Spring wires it into auth/matching/messaging/notifications
```

### Client contract catalog

| Interface | Producer | Consumed by | Methods (doc source) |
|---|---|---|---|
| `UserClient` | `users` | `auth`, `matching`, `messaging`, `notifications`, `analytics` | `getPublicProfile(userId) → PublicUserResponse` (`user.md` §GET masked) · `lookupByCredential(credentialId) → Optional<UserId>` (`auth.md` §1.1 onb/oid resolution) · `getStatus(userId) → UserStatus` (`auth.md` §3 status gate) · `getActiveGenderCounts(region) → { women, men }` (`analytics.md` reconciliation) |
| `RatioClient` | `ratio` | `users` | `evaluateGenderRatio(region, gender) → GateDecisionDto` (`ratio.md` §Sync gate contract) |
| `AnalyticsClient` | `analytics` | `ratio` | `getRatioState(region) → RatioStateDto` (`analytics.md` §Sync client contract) |
| `MatchingClient` | `matching` | `messaging` | `getMatch(matchId) → MatchInfo { match_id, woman_id, man_id, status }` (`messaging.md` — thread creation + masking checks) |

---

## Event records (centralized)

`commons.events.<module>.<Event>` — the single source for every payload shape. Producers publish these records transactionally (Spring Modulith, at-least-once — mechanism per `auth.md` §9.5); consumers import the same records. In-process Java objects today; snake_case JSON over the broker after extraction.

| Record | Payload | Producer → consumers (doc) |
|---|---|---|
| `users.UserRegistered` | `{ user_id, credential_id, gender, region, status, created_at }` | users → matching, notifications, analytics |
| `users.ProfileUpdated` | `{ user_id }` (projection refetches via `UserClient`) | users → matching |
| `users.PhotoChanged` | `{ user_id, photo_id, change: ADDED\|REMOVED\|REORDERED }` | users → matching |
| `users.UserDeactivated` | `{ user_id, credential_id }` | users → messaging, ratio, notifications, matching |
| `users.UserShadowBanned` | `{ user_id }` | users → notifications, analytics |
| `users.UserSuspended` | `{ user_id }` | users → matching, notifications, ratio |
| `users.UserReactivated` | `{ user_id }` | users → matching, notifications, ratio |
| `users.ReportFiled` | `{ report_id, reporter_id, reported_id, category }` | users → analytics |
| `users.ReportResolved` | `{ report_id, decision, reported_id, reporter_id }` | users → analytics |
| `users.BlockCreated` | `{ blocker_id, blocked_id }` | users → matching, messaging |
| `users.BlockRemoved` | `{ blocker_id, blocked_id }` | users → matching, messaging |
| `auth.SessionRevoked` | `{ session_id, credential_id }` (pinned, `auth.md` §9.5) | auth → notifications (device cleanup) |
| `auth.CredentialUpserted` | `{ credential_id, email }` | auth → notifications (email projection) |
| `matching.LikeReceived` | `{ like_id, liker_id, liked_id }` | matching → notifications, analytics |
| `matching.MatchCreated` | `{ match_id, woman_id, man_id }` | matching → messaging, notifications |
| `matching.MatchDeleted` | `{ match_id, unmatched_by }` | matching → messaging, notifications |
| `matching.ProfileViewed` | `{ viewer_id, profile_id, source: DECK\|PROFILE }` | matching (deck) + users (profile open) → analytics |
| `ratio.AdmittedFromQueue` | `{ user_id, rank, total_waiting, estimated_wait_ms }` | ratio → users, notifications |
| `analytics.DailyViewCountReady` | `{ user_id, count, window: "daily" }` | analytics → notifications (VIEW push) |
| `analytics.WeeklySummaryReady` | `{ user_id, views, window: "weekly" }` | analytics → notifications (WEEKLY_SUMMARY email) |
| `messaging.MessageExpiring` | `{ conversation_id, expires_at }` | messaging → notifications (EXPIRY push + email) |

---

## Exceptions & error codes

Typed families fix the HTTP status; the `ErrorCode` enum carries the machine-readable code. `ValidationException` additionally carries `field_errors`. Raising module-specific cases = family exception + code — **no per-code exception classes**.

| Family | HTTP | Default code | Notes |
|---|---|---|---|
| `ApiException` (base) | — | — | `{ code, httpStatus, message, field_errors }` |
| `UnauthorizedException` | 401 | `TOKEN_*` per case | thrown by `commons.security` |
| `ForbiddenException` | 403 | `FORBIDDEN` | role / owner checks |
| `OnboardingIncompleteException` | 403 | `ONBOARDING_INCOMPLETE` | `onb=false` principal |
| `NotFoundException` | 404 | `NOT_FOUND` | code may be `USER_NOT_VISIBLE` (users masking) |
| `ConflictException` | 409 | per call site | `LIKE_EXISTS`, `ALREADY_REVIEWED`, … |
| `ValidationException` | 422 | `VALIDATION_ERROR` | carries `field_errors` |
| `TooManyRequestsException` | 429 | `TOO_MANY_REQUESTS` | |
| `TemporaryException` | 503 | `TEMPORARY_ERROR` | |

### `ErrorCode` registry (single enum, globally unique)

| Module | Codes |
|---|---|
| cross-cutting | `VALIDATION_ERROR` · `INVALID_REQUEST_BODY` · `FORBIDDEN` · `NOT_FOUND` · `TOO_MANY_REQUESTS` · `TEMPORARY_ERROR` · `ONBOARDING_INCOMPLETE` |
| `auth` | `TOKEN_MISSING` · `TOKEN_MALFORMED` · `TOKEN_EXPIRED` · `TOKEN_INVALID_SIGNATURE` · `TOKEN_INVALID_AUDIENCE` · `TOKEN_UNKNOWN_KID` · `TOKEN_REVOKED` (reserved) · `TOKEN_INVALID_CLAIMS` · `GOOGLE_TOKEN_MALFORMED` · `GOOGLE_TOKEN_EXPIRED` · `GOOGLE_TOKEN_INVALID_SIGNATURE` · `GOOGLE_TOKEN_AUDIENCE_MISMATCH` · `GOOGLE_VERIFICATION_UNAVAILABLE` · `ACCOUNT_DISABLED` · `REFRESH_TOKEN_INVALID` · `REFRESH_TOKEN_EXPIRED` · `REFRESH_TOKEN_REUSE` · `KEY_UNREACHABLE` |
| `users` | `USER_NOT_VISIBLE` · `USER_EXISTS` · `AGE_VERIFICATION_REQUIRED` · `AGE_GATE_BLOCKED` · `IMMUTABLE_FIELD` · `INVALID_STORAGE_PATH` · `PHOTO_EXISTS` · `PHOTO_LIMIT` · `INVALID_ORDER` · `CANNOT_BLOCK_SELF` · `REPORT_EXISTS` · `REPORT_SELF` · `REPORT_LIMIT_REACHED` · `ALREADY_REVIEWED` · `NOT_REVIEWABLE` · `INVALID_STATE` |
| `matching` | `LIKE_EXISTS` · `RELIKE_COOLDOWN` · `CANNOT_LIKE_SELF` · `AGE_RULE` · `LIKE_NOT_PENDING` · `PAIR_INELIGIBLE` |
| `notifications` | `DEVICE_MISMATCH` |

Rule: a new code = enum entry **first**, doc row second. Codes are never renamed (additive-only).

---

## `commons.error` / `commons.web`

`ErrorDto` — canonical envelope (`auth.md` §2 is the doc source; this record is the code artifact):

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

`PagedDto<T>`:

```json
{ "items": [], "page": 0, "size": 20, "total": 0 }
```

---

## `commons.security` (folded Auth Lib)

Per decision, the Auth Lib folds into commons. Contents: RS256 JWT verification against `GET /.well-known/jwks.json` (kid-based key pick, 30 s clock-skew, JWKS cache + one refresh on `TOKEN_UNKNOWN_KID`), and the authenticated principal record read off the token:

| Claim | Type | Always | Notes |
|---|---|---|---|
| `sub` | string | ✅ | `AUTH_CREDENTIAL.id` — permanent identity |
| `oid` | string | ⚠️ | `USER.id` — post-onboarding only |
| `sid` | string | ✅ | refresh session id |
| `dfp` | string | ✅ | session device fingerprint |
| `onb` | boolean | ✅ | onboarded flag |
| `email` | string | ✅ | Google-verified |
| `roles` | string[] | ✅ | `USER` / `DEV` / `ADMIN` |

Token contract (issuance, refresh rotation, logout, claims lifecycle) stays documented in `auth.md` — commons only verifies. Failures raise `UnauthorizedException` with the `TOKEN_*` codes above.

---

## Non-rules

- No controllers, repositories, business logic, or DB access in commons — records, enums, exceptions, interfaces, and the security verifier only. Jackson annotations allowed.
- Adding/changing a commons contract is a **doc-first change**: update the `api-design/*.md` row, then the record.
- The Auth Service module (`auth`) remains the issuer; `commons.security` is only the verifier — no overlap.