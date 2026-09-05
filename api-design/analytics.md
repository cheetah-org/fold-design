# Analytics Service — API

Base URL: `https://api.wiingman.in/analytics/api/v1` — gateway convention `{baseURL}/{service}/api/v1/{resource}`; all paths below are relative to this base (e.g. `GET https://api.wiingman.in/analytics/api/v1/cities/{city}/ratio-state`) · Content-Type: `application/json` · Dates: ISO-8601 UTC · IDs: UUID

**Module:** `analytics` in the Spring Boot modulith. Owns `RATIO_STATE` (raw gender counts only — no gate state; ratio owns the open/closed decision), `PROFILE_VIEW_EVENT`, `ENGAGEMENT_FACT` (ER). **No end-user surface** — an internal, event-fed module whose outputs are (a) the count projection ratio reads, (b) two scheduled announcements notifications renders, (c) an admin/ops-only read surface. Powers the product promises "X women viewed your profile today" (daily push) and the weekly summary email.

**Auth:** `Authorization: Bearer <access_token>` — reads `oid`, `onb`, `roles` per `auth.md` §1.1. **Admin check order (same as `ratio.md`):** authenticate → require `DEV`/`ADMIN` (else `403 FORBIDDEN`) → onboarding check applies only to non-admin callers.

---

## Counting model — `RATIO_STATE` (internal contract)

| Mechanic | Rule |
|---|---|
| Event-fed counters | per city, `active_women` / `active_men` move ±1 on consumed status events (§Consumed events). Only `ACTIVE` users are counted — matches `user.md` counted-as-active. |
| Reconciliation | **hourly** job sync-calls `UserClient.getActiveGenderCounts(city)` (true counts from the users module) and overwrites `RATIO_STATE` — self-heals any drift from dropped/duplicated events. `POST /ratio-state/recompute` runs the same path on demand. |
| Single writer | one reconciliation/count-update writer per city (lock) so event updates and reconciliation never interleave; counter updates commit in the same transaction as the consumed event. |
| Shape | plain projection: `{ city, active_women, active_men, updated_at }`. No `admission_open` — gate decisions are ratio's (`computed_open`/`effective_open`). |

---

## View pipeline — `PROFILE_VIEW_EVENT`

- Consumes `ProfileViewed { viewer_id, profile_id, source }` from `matching` (`source=DECK`, per rendered card) and `users` (`source=PROFILE`, profile open). Raw events are retained.
- **A "view" = a distinct viewer × distinct profile per day** (dedup at aggregation; both sources count) — matches the product phrasing "X women viewed your profile" as distinct women.
- **Daily batch** (configurable morning run): trailing-24h distinct female viewers per `ACTIVE` male user → publishes `DailyViewCountReady { user_id, count, window: "daily" }` → `notifications` renders the `VIEW` push.
- **Weekly batch** (off-peak): trailing-7-days distinct views per user → `WeeklySummaryReady { user_id, views, window: "weekly" }` → `notifications` renders the `WEEKLY_SUMMARY` email.
- **Recipients:** `ACTIVE` men — both features are the men's-inbox retention surface (design doc); women's insights (like-delivery) are **deferred premium**, so no women-targeted batches in V1.
- Batches are **window-keyed idempotent** (`user_id` + window): a re-run publishes the same key and `notifications` dedupes on it — no double sends.

---

## Engagement facts — `ENGAGEMENT_FACT`

Consumes `LikeReceived` (likes), `MatchCreated` (matches), `UserRegistered` (new_users) into **hourly buckets** per city, rolled up to **day**; `ratio_snapshot` records the `RATIO_STATE` counts at bucket close. V1 consumer: admin dashboard endpoint only — nothing client-facing.

---

## Sync client contract

`commons.client.analytics.AnalyticsClient` (interface in commons; impl bean here) — consumed by `ratio`:

| Method | Returns | Doc |
|---|---|---|
| `getRatioState(city)` | `RatioStateDto` | feeds ratio's gate evaluation; `ratio` never writes `RATIO_STATE` |

---

## Published events (contract)

Spring Modulith transactional event publication — at-least-once, retried listeners (mechanism per `auth.md` §9.5). Payloads snake_case JSON.

| Event | Payload | Consumers |
|---|---|---|
| `DailyViewCountReady` | `{ user_id, count, window: "daily" }` | `notifications` (VIEW push) |
| `WeeklySummaryReady` | `{ user_id, views, window: "weekly" }` | `notifications` (WEEKLY_SUMMARY email) |

## Consumed events

`UserRegistered` (+count) · `UserDeactivated` / `UserSuspended` / `UserShadowBanned` (−count) · `UserReactivated` (+count) — shadow-banned users are not counted as active per `user.md`; counting them here is `analytics`' job (ratio deliberately doesn't react) · `ProfileViewed` (matching + users) · `LikeReceived` / `MatchCreated` (facts). All handled transactionally; counts and facts commit with the event consumption.

---

## DTOs

`RatioStateDto`

```json
{
  "city": "Bangalore",
  "active_women": 520,
  "active_men": 480,
  "updated_at": "2026-09-02T09:00:00Z"
}
```

`EngagementFactDto`

```json
{
  "bucket": "day",
  "period_start": "2026-09-01T00:00:00Z",
  "likes": 34,
  "matches": 12,
  "new_users": 7,
  "ratio_snapshot": { "active_women": 512, "active_men": 479 }
}
```

---

## Admin surface (ops-only, `DEV`/`ADMIN`)

### `GET /cities/{city}/ratio-state`
→ 200 `RatioStateDto`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first) |
| 403 | `ONBOARDING_INCOMPLETE` — non-admin caller has not completed onboarding |
| 422 | `VALIDATION_ERROR` — unknown city |

### `POST /cities/{city}/ratio-state/recompute`
Force reconciliation: sync-call `UserClient.getActiveGenderCounts(city)`, overwrite `RATIO_STATE`. Idempotent. → 200 `RatioStateDto` (fresh counts).

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first) |
| 403 | `ONBOARDING_INCOMPLETE` — non-admin caller has not completed onboarding |
| 422 | `VALIDATION_ERROR` — unknown city |
| 503 | `TEMPORARY_ERROR` — users module unreachable; retry |

### `GET /cities/{city}/engagement`
Paged facts. `?bucket=hour|day&from=&to=&page=&size=` (default `bucket=day`, newest first) → 200 `PagedDto<EngagementFactDto>`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first) |
| 403 | `ONBOARDING_INCOMPLETE` — non-admin caller has not completed onboarding |
| 422 | `VALIDATION_ERROR` — bad `bucket`/range |

> `/cities/{city}/...` is city-scoped like `ratio.md`'s gate — analytics state is per-city, not per-user. V1 has exactly one city (`Bangalore`).

---

## Rate limiting

Rate limiting is a cross-cutting concern handled by shared infrastructure, not implemented per-endpoint here — mechanism TBD (same as `ratio.md`).

---

## Non-functional notes

- **Counter correctness:** counters commit with the consumed event (same transaction); reconciliation is the safety net — the gate is never more than one reconciliation interval stale.
- **Dedup semantics:** raw `PROFILE_VIEW_EVENT` rows kept; distinct-viewer dedup applied at aggregation time (`viewer_id × profile_id × day`).
- **Batch idempotency:** window-keyed (`user_id` + window); `notifications` dedupes on the key.
- **Schedules configurable:** daily batch time, weekly day/time, reconciliation interval — all config, not code.

---

## Cross-module notes

- **`ratio`** reads counts via `AnalyticsClient.getRatioState(city)` — the only sync consumer; ratio never writes `RATIO_STATE` (sole decision-maker for the gate, `ratio.md`).
- **`notifications`** renders the two announced events — it owns nothing analytical.
- **`users`** is the source of truth for reconciliation counts (`UserClient.getActiveGenderCounts(city)` — added to the `commons.md` catalog).
- **Emitters:** `ProfileViewed` arrives from `matching` (deck renders, `matching.md` §Discovery feed) and `users` (profile opens, `user.md`).
- **ER:** no changes — `PROFILE_VIEW_EVENT.source` already gains `DECK` (see `matching.md`), `RATIO_STATE` is already counts-only.