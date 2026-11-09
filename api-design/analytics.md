# Analytics Service — API

Base URL: `https://api.wiingman.in/analytics/api/v1` — gateway convention `{baseURL}/{service}/api/v1/{resource}`; all paths below are relative to this base (e.g. `GET https://api.wiingman.in/analytics/api/v1/regions/{region}/ratio-state`) · Content-Type: `application/json` · Dates: ISO-8601 UTC · IDs: UUID

**Module:** `analytics` in the Spring Boot modulith. Owns `RATIO_STATE` (raw gender counts only — no gate state; ratio owns the open/closed decision), `PROFILE_VIEW_EVENT`, `ENGAGEMENT_FACT` (ER). **No end-user surface** — an internal, event-fed module whose outputs are (a) the count projection ratio reads, (b) two scheduled announcements notifications renders, (c) an admin/ops-only read surface. Powers the product promises "X women viewed your profile today" (daily push) and the weekly summary email.

**Auth:** `Authorization: Bearer <access_token>` — reads `oid`, `onb`, `roles` per `auth.md` §1.1. **Check order (same as `ratio.md`):** authenticate → require `DEV`/`ADMIN` (else `403 FORBIDDEN`). Every route in this service is admin-only, so the onboarding check never fires here (review fix — dead rows removed).

---

## Counting model — `RATIO_STATE` (internal contract)

| Mechanic | Rule |
|---|---|
| Event-fed counters | per region (derived from user geolocation), `active_women` / `active_men` move ±1 on consumed status events (§Consumed events). Only `ACTIVE` users are counted — matches `user.md` counted-as-active. |
| Reconciliation | **hourly** job sync-calls `UserClient.getActiveGenderCounts(region)` (true counts from the users module) and overwrites `RATIO_STATE` — self-heals any drift from dropped/duplicated events. `POST /ratio-state/recompute` runs the same path on demand. |
| Single writer | one reconciliation/count-update writer per region (lock) so event updates and reconciliation never interleave; counter updates commit in the same transaction as the consumed event. |
| Shape | plain projection: `{ region, active_women, active_men, updated_at }`. No `admission_open` — gate decisions are ratio's (`computed_open`/`effective_open`). |

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

Consumes `LikeReceived` (likes), `MatchCreated` (matches), `UserRegistered` (new_users) into **hourly buckets** per region, rolled up to **day**; `ratio_snapshot` records the `RATIO_STATE` counts at bucket close. V1 consumer: admin dashboard endpoint only — nothing client-facing.

---

## Sync client contract

`commons.client.analytics.AnalyticsClient` (interface in commons; impl bean here) — consumed by `ratio`:

| Method | Returns | Doc |
|---|---|---|
| `getRatioState(region)` | `RatioStateDto` | feeds ratio's gate evaluation; `ratio` never writes `RATIO_STATE` |

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
  "region": "Bangalore",
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

### `GET /regions/{region}/ratio-state`
→ 200 `RatioStateDto`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first; every route here is admin-only, so `ONBOARDING_INCOMPLETE` never fires — review fix) |
| 422 | `VALIDATION_ERROR` — unknown region |

### `POST /regions/{region}/ratio-state/recompute`
Force reconciliation: sync-call `UserClient.getActiveGenderCounts(region)`, overwrite `RATIO_STATE`. Idempotent. → 200 `RatioStateDto` (fresh counts).

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first; every route here is admin-only, so `ONBOARDING_INCOMPLETE` never fires — review fix) |
| 422 | `VALIDATION_ERROR` — unknown region |
| 503 | `TEMPORARY_ERROR` — users module unreachable; retry |

### `GET /regions/{region}/engagement`
Paged facts. `?bucket=hour|day&from=&to=&page=&size=` (default `bucket=day`, newest first) → 200 `PagedDto<EngagementFactDto>`.

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first; every route here is admin-only, so `ONBOARDING_INCOMPLETE` never fires — review fix) |
| 422 | `VALIDATION_ERROR` — bad `bucket`/range |

> `/regions/{region}/...` is region-scoped like `ratio.md`'s gate — analytics state is per-region, not per-user. Region is derived server-side from user geolocation. V1 has exactly one region (`Bangalore`).

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

- **`ratio`** reads counts via `AnalyticsClient.getRatioState(region)` — the only sync consumer; ratio never writes `RATIO_STATE` (sole decision-maker for the gate, `ratio.md`).
- **`notifications`** renders the two announced events — it owns nothing analytical.
- **`users`** is the source of truth for reconciliation counts (`UserClient.getActiveGenderCounts(region)` — added to the `commons.md` catalog).
- **Emitters:** `ProfileViewed` arrives from `matching` (deck renders, `matching.md` §Discovery feed) and `users` (profile opens, `user.md`).
- **ER synced:** `RATIO_STATE.region` (was `city`), `ENGAGEMENT_FACT.region` + `period_start` — all reflected in `entities/er-diagram.md`.

---

## Implementation notes — view pipeline & announcements (V1 backend)

This covers the view pipeline and the two announcements, then the engagement facts and the admin endpoints (next section). The `RATIO_STATE` projection is deliberately skipped (see "Deviation" below).

- **Raw rows.** Every `ProfileViewed` (deck and profile open) becomes a `profile_view_event` row (`viewer_id`, `profile_id`, `source`, `viewed_at`, `viewed_on`), duplicates included. `viewed_on` is the calendar day in `fold.analytics.zone` (Asia/Kolkata), stamped at consumption.
- **Only women's views count, on active men.** The event carries ids only, so viewer gender is looked up at counting time through a new `UserClient.getWomenAmong(ids)` (batched, `gender-lookup-batch-size`). Recipients are `ACTIVE` men (`getActiveMaleUserIds`); a man who is deactivated, suspended or shadow-banned by the time the batch runs is skipped.
- **Daily.** Distinct women who viewed the man in the trailing `daily-window` (24h) → `DailyViewCountReady`. **Weekly.** Views in the trailing `weekly-window` (7d), where a view is one woman × one day, as the spec defines a view (so a woman who looks on three different days counts 3) → `WeeklySummaryReady`.
- **Zero is silent (addition).** A man with no views from women gets nothing, so nobody is told "0 women viewed you".
- **Batch key includes the period (spec fix).** The spec keys batches on `user_id + window`, but notifications dedupes on its event key forever, so the second day would be dropped. The announcement's event id is `daily:<user_id>:<yyyy-mm-dd>` (zone date) and `weekly:<user_id>:<iso-year>-W<week>`. A re-run in the same period repeats the id and is deduplicated; the next period gets a new one.
- **Schedules in config:** `daily-cron` (09:00), `weekly-cron` (Sunday 03:00), both read in `zone`. The jobs assume a **single running instance**; with several, every instance would run the batch (harmless because of the dedupe keys, but wasteful).
- **Retention (addition).** The spec keeps raw rows indefinitely, but the deck writes one per card shown. A sweep (`retention-sweep-interval`, 1h) deletes rows older than `view-retention` (30d), which covers the 7-day window with margin.
- **Profile opens.** `GET /users/{userId}` now publishes `ProfileViewed { source: PROFILE }` when a caller other than the owner (and not through the self / `DEV` / `ADMIN` view) opens a profile; see `user.md`.
- **Known limit.** Counting loads the window's distinct (profile, viewer, day) rows into memory and resolves them in Java. Fine at V1 volume; a larger deployment would push the gender join into SQL or pre-aggregate.

### Deviation — no `RATIO_STATE` projection in V1

The `RATIO_STATE` table, its event-fed ±1 counters and the hourly reconciliation are **deliberately not built**. `AnalyticsClient.getRatioState()` reads the live active-women / active-men counts from `users` (`UserClient.getActiveGenderCounts()`), as `ratio` already does for the gate and the queue-promotion sweep.

- **Why.** The counts decide who is admitted, and a live read can never drift. An asynchronous, at-least-once projection can: a duplicated or dropped event skews it until the next reconcile, so the gate could over-admit or stall the queue for up to an hour. The spec's reason for the projection (analytics owning the counts once services are split) does not apply while everything runs in one process, where it would only be a second source of truth.
- **Consequences.** `POST /ratio-state/recompute` has nothing to recompute and is dropped. `GET /ratio-state` stays as a plain admin read of the live counts. The `ratio_snapshot` on engagement facts takes the live counts at bucket close.
- **When to revisit.** On extracting `analytics` (or `users`) into its own service, or when a history of the counts is wanted. The `AnalyticsClient` seam is unchanged, so `ratio` would not need to change.
---

## Implementation notes — engagement facts & admin surface (V1 backend)

- **Counting.** `LikeReceived`, `MatchCreated` and `UserRegistered` each add 1 to the `likes` / `matches` / `new_users` of **both** the event's hour row and its day row (so the day is always current; it is not rolled up later). Counts are atomic increments, and the row is created on first use (two writers racing to create it settle on the unique key `(bucket, period_start)`), so concurrent events never lose a count. The bucket comes from the event's own publish time, so an event handled just after the hour turns still lands in the hour it happened in.
- **Buckets follow the analytics zone (spec difference).** Hours and days start on the hour / midnight of `fold.analytics.zone` (Asia/Kolkata), not UTC, so a day's `period_start` is `18:30Z` of the previous UTC day. The spec's example shows a UTC midnight.
- **Ratio snapshot at close.** A job (`engagement-close-cron`, five past each hour) records the active women and men on every ended bucket that still has none, and makes sure the hour that just ended, and its day, have a row even if nothing happened (so the series has no holes after an uneventful hour). The counts are the live ones read at that moment, a few minutes after the bucket ended (see the `RATIO_STATE` deviation above). `ratio_snapshot` is `null` while a bucket is open. A missed run is made up by the next one for existing rows, but an hour with no events *and* no run stays absent.
- **Best-effort counts.** Events are counted once per delivery, with no per-event dedupe, so a redelivered event would be counted twice. Today's listeners do not redeliver; revisit if a durable, retrying event bus is added.
- **Admin surface.** `GET /analytics/api/v1/ratio-state` → `{ active_women, active_men, updated_at }`, the live counts as the gate sees them (`updated_at` is the read time). `GET /analytics/api/v1/engagement?bucket=hour|day&from=&to=&page=&size=` → `PagedDto<EngagementFact>` newest first; `bucket` defaults to `day`, the range is `[from, to)` on `period_start` (default: everything up to now), `size` defaults to 24 (max 100). A bad `bucket`, `from` after `to`, or bad paging is `422 VALIDATION_ERROR`.
- **No `{region}` in the paths** (V1 has one region, and the region concept was removed), and **`POST /ratio-state/recompute` is dropped** (see the `RATIO_STATE` deviation).
- **Gate (spec difference).** The spec requires the `DEV` / `ADMIN` role, but nothing in the system mints those roles: tokens only ever carry `USER`. The routes instead require the **moderator identity** (`sub = moderator`, minted through the god token), the same privileged identity the moderation APIs use; anyone else gets `403 FORBIDDEN`. When `DEV` / `ADMIN` role granting exists, swap the check in one place (`AnalyticsAccess`).
