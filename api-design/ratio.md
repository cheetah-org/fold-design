# Ratio Service — API

Base URL: `https://api.wiingman.in/ratio/api/v1` — gateway convention `{baseURL}/{service}/api/v1/{resource}`; all paths below are relative to this base (e.g. `POST https://api.wiingman.in/ratio/api/v1/users/{userId}/admit`) · Content-Type: `application/json` · Dates: ISO-8601 UTC · IDs: UUID

**Module:** `ratio` in the Spring Boot modulith. Owns `ADMISSION_QUEUE` + `RATIO_EVENT_LOG` (ER). Reads `RATIO_STATE` via `AnalyticsClient.getRatioState(region)` (analytics-owned: `active_women`, `active_men`, per region). **Analytics exposes `active_women` / `active_men` only; ratio is the sole source of the open/closed decision (`computed_open`, `effective_open`).** This module is **mostly internal** — a gate, not a feature surface:

- **No client-facing endpoints.** The mobile queue view is `GET /users/{userId}/admission` in `user.md` (composed from ratio data via client interface) — that doc stays the single client contract.
- **Sync gate contract** (§Sync gate contract) called by the `users` module during `POST /users` — in-process client interface today, HTTP after extraction (inter-service-communication.md §6).
- **Events out** (§Published events) that flip `USER.status` — applied by the `users` module, pushed by `notifications`.
- **Admin/debug surface** (§Admin surface) — `DEV`/`ADMIN` only, user-first URIs per convention (no admin namespace).

**Auth:** `Authorization: Bearer <access_token>` — reads `oid`, `onb`, `roles` per `auth.md` §1.1. **Check order:** authenticate → require `DEV`/`ADMIN` in `roles` (else `403 FORBIDDEN`). Every route in this service is admin-only, so the onboarding check never fires here (review fix — dead `ONBOARDING_INCOMPLETE` rows removed).

---

## Gate & queue mechanics (internal contract)

Client-visible rules live in `user.md` §Admission & queue rules; the mechanics below are what ratio implements:

| Mechanic | Rule |
|---|---|
| Effective gate | `effective_open = mode == AUTO ? (open_slots > 0) : mode`, where `open_slots = ceil(max_men_per_woman × active_women) − active_men` (counts from `RATIO_STATE`; `max_men_per_woman` = `fold.ratio.max-men-per-woman`, default 1.0; decimal arithmetic so ratios like 0.1 stay exact). Lowering the ratio never evicts anyone — `open_slots` just stays 0 until women catch up. `mode` is ratio-local (`AUTO` \| `OPEN` \| `CLOSED` override, §Admin surface). Analytics stays the source of the computed counts; the override never writes `RATIO_STATE`. |
| Evaluation triggers | a periodic sweep only (`fold.ratio.promotion-interval`, default 5m). No per-status-event triggers — women joining, men leaving and ratio changes are all picked up by the next sweep. Single running instance assumed (V1). |
| Enqueue | happens synchronously inside `POST /users` (§Sync gate contract): a man registering while the gate is closed gets an `ADMISSION_QUEUE` row (FIFO by `created_at`) and `status=QUEUED`. |
| Promotion | each pass fills **every** open slot: fetch the `open_slots` oldest queued men (FIFO by `created_at`, via `UserClient.getQueueHead`) and publish one `AdmittedFromQueue` per man; `users` flips `QUEUED→ACTIVE` on the event. A delayed flip means the next pass re-announces the same men (the flip is idempotent; known V1 side effect: a duplicate "You're in" push). |
| Counted as active | `ACTIVE` users only. Any other status is excluded from the count. Ratio only needs a dedicated consumed event for transitions that affect the queue or counts (deactivate/suspend/reactivate); other status changes, including shadow-banning, are out of ratio's scope in v1. |

---

## Sync gate contract

Owned by the caller (`users` module) per inter-service-communication.md §6; in-process today (`RatioClient.isGateOpen()` — **V1 code is single-region, so no `region` argument**; the `region` fields below are the multi-region target shape), the same DTOs become the HTTP shape after extraction.

**Request — `GateCheckRequest`**
```json
{ "region": "Bangalore", "gender": "MALE" }
```

**Response — `GateDecisionDto` (man, gate open or woman)**
```json
{ "decision": "ADMIT", "queue": null }
```

**Response — `GateDecisionDto` (man, gate closed)**
```json
{
  "decision": "ENQUEUE",
  "queue": { "rank": 3, "total_waiting": 12, "estimated_wait_ms": 3600000 }
}
```

| Field | Type | Notes |
|---|---|---|
| `decision` | enum | `ADMIT` \| `ENQUEUE` — women always `ADMIT`; men `ENQUEUE` when the gate is closed |
| `queue` | object\|null | `QueueSnapshotDto` — `rank` (FIFO position), `total_waiting`, `estimated_wait_ms` (`pending_ahead × rolling_avg_admission_interval`, floor 5 min; exact heuristic build-time) |

**Side effects on `ENQUEUE`:** insert `ADMISSION_QUEUE` row + `RATIO_EVENT_LOG(type=QUEUED)`. No event published (the caller returns the snapshot inline; later promotions are event-driven).

---

## Published events (contract)

Spring Modulith transactional event publication — at-least-once, retried listeners (mechanism per `auth.md` §9.5). Payloads snake_case JSON.

| Event | Payload | Consumers |
|---|---|---|
| `AdmittedFromQueue` | `{ user_id, rank, total_waiting, estimated_wait_ms }` | `users` (flip `QUEUED → ACTIVE`), `notifications` (admission push) |

**Consumed events:** `UserDeactivated`, `UserSuspended` (remove from queue if queued / log `DEACTIVATED` — counts are `analytics`' job) · `UserReactivated` (re-admit path → `ACTIVE` or re-enqueue per gate) — all trigger a re-evaluation.

---

## Admin surface

`DEV`/`ADMIN` only (per `user.md` roles); business semantics apply to every role. State flips are **eventual** — ratio emits, `users` applies — so mutations return **202** with the event name, not a promised status.

### `POST /users/{userId}/admit`
Force-admit from the queue, **bypassing the gate** (audit-logged as manual). Guards: target must exist (`404 NOT_FOUND`) and be `QUEUED` (else `409 INVALID_STATE`). Emits `AdmittedFromQueue`. No body.

**Response 202**
```json
{ "user_id": "7a2b9c1d-3e4f-4a5b-8c6d-9e0f1a2b3c4d", "event": "AdmittedFromQueue" }
```

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first; every route here is admin-only, so `ONBOARDING_INCOMPLETE` never fires — review fix) |
| 404 | `NOT_FOUND` — user missing or not in queue |
| 409 | `INVALID_STATE` — target not `QUEUED` |

### `GET /users/{userId}/ratio-events`
Per-user `RATIO_EVENT_LOG` (ops debugging without DB access). `?type=&page=&size=` → 200 `PagedDto<RatioEventDto>`, newest first.

**Response 200**
```json
{
  "items": [
    { "type": "ADMITTED", "at": "2026-09-02T09:00:00Z" },
    { "type": "QUEUED", "at": "2026-08-29T10:00:00Z" }
  ],
  "page": 0,
  "size": 20,
  "total": 2
}
```

| Status | Code |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first; every route here is admin-only, so `ONBOARDING_INCOMPLETE` never fires — review fix) |
| 404 | `NOT_FOUND` — user missing |

### `GET /regions/{region}/admission-gate`
Effective gate view (override + computed). → 200 `GateDto`.

**Response 200**
```json
{
  "region": "Bangalore",
  "mode": "AUTO",
  "computed_open": true,
  "effective_open": true,
  "active_women": 520,
  "active_men": 480,
  "updated_at": "2026-09-02T09:00:00Z"
}
```

### `PUT /regions/{region}/admission-gate`
Override the gate for a region. Body `{ "mode": "AUTO" | "OPEN" | "CLOSED" }` — `AUTO` clears any override. Idempotent (same mode → 200, no event). Changing to `OPEN`/`CLOSED` triggers an immediate evaluation pass (promotions flow as events).

**Request**
```json
{ "mode": "CLOSED" }
```

**Response 200 — `GateDto`** (as `GET`).

| Status | Code (both gate routes) |
|---|---|
| 403 | `FORBIDDEN` — caller lacks `DEV`/`ADMIN` role (checked first; every route here is admin-only, so `ONBOARDING_INCOMPLETE` never fires — review fix) |
| 422 | `VALIDATION_ERROR` — unknown region / bad `mode` |

> Region as a resource (`/regions/{region}/...`) is the one non-user-first path in the API set — gate state is region-scoped, not user-scoped. Region is derived server-side from user geolocation. V1 has exactly one region (`Bangalore`).

---

## Rate limiting

Rate limiting is a cross-cutting concern handled by shared infrastructure, not implemented per-endpoint here — mechanism TBD.

---

## Non-functional notes

### Concurrency & idempotency
- **Single evaluation lock per region** — the 5-min sweep and event-triggered passes serialize on the region, so a promotion pass and an admin force-admit can't double-admit the same queue row (row-level conditional claim: admit only when `admitted_at IS NULL`).
- `POST .../admit` — retried after a lost response is **not** safe to blindly repeat (admit twice is harmless — second gets `409 INVALID_STATE` after the flip lands — but callers should treat 409-after-202 as success-confirm).
- `PUT .../admission-gate` — idempotent per mode.

### Fairness & audit
- FIFO by `ADMISSION_QUEUE` order; `rank` is computed once at enqueue via `rank_of(onboarded_time)` and persisted; FIFO order follows rank order, which is monotonic with onboarding time under the current identity function.
- Every gate transition (enqueue/admit/deactivate) writes `RATIO_EVENT_LOG` before the event publishes — the log row and the event live in the same transaction (§Published events).

### Cross-module notes
- **`RATIO_STATE` is analytics-owned** — ratio only reads it. The gate override (`mode`) is ratio-local state and deliberately **not** written into `RATIO_STATE`.
- **Status flips:** ratio never writes `USER.status` — the `users` module applies flips from the events above (mirrors the `SessionRevoked` consume pattern in `notifications.md`).
- **Client contract:** `user.md` §Admission & queue rules + `GET /users/{userId}/admission` remain the only client-visible surface; this doc's §Sync gate contract defines what feeds it.

---

## Implementation notes — admin surface & audit log (V1 backend)

The four admin routes exist at `ratio/api/v1`: `POST /users/{userId}/admit`, `GET /users/{userId}/ratio-events`, `GET /admission-gate`, `PUT /admission-gate`. Where the build differs from the sections above:

- **Gate (spec difference).** Every route requires the **moderator identity** (`sub = moderator`, minted through the god token), not `DEV` / `ADMIN`: nothing issues those roles yet, so they would be unreachable. Anyone else gets `403 FORBIDDEN`. The check lives in one place (`OperatorGuard`, shared with `analytics`), so swapping in the roles later is a one-line change.
- **No `{region}`** in the gate paths (V1 has one region and the region concept was removed): `/admission-gate`, and `GateDto` has no `region`.
- **Gate override.** One row (`admission_gate`); no row means `AUTO`. `effective_open` is `computed_open` in `AUTO`, `true` in `OPEN`, `false` in `CLOSED`, and the same decision now answers `RatioClient.isGateOpen()` at registration.
  - **`OPEN`** drains the whole queue: the evaluation pass admits every queued man, oldest first, regardless of free slots. **`CLOSED`** admits nobody, even with free slots, and sends every new man to the queue. `AUTO` fills the open slots, as before.
  - `PUT` with the mode already set changes nothing (200, no pass). Switching to `OPEN` / `CLOSED` runs the promotion pass immediately, in the same transaction; switching back to `AUTO` waits for the next sweep, as the spec says.
  - `updated_at` is the time of the read (the counts are live), as in `analytics`' `ratio-state`; the time the override was last changed is not exposed.
  - A bad or missing `mode` is `422 VALIDATION_ERROR` (see `commons.md`: this needed unconvertible enum values in a body to be a 422 globally).
- **No `ADMISSION_QUEUE` table.** The queue of record is still the users with `status = QUEUED`, ordered by `created_at` (unchanged). The spec's "row-level conditional claim" therefore does not exist: if the sweep and a force-admit pick the same man at the same moment, both publish `AdmittedFromQueue` (the flip is idempotent, since only `QUEUED` men flip) and the audit trail shows two `ADMITTED` entries.
- **`RATIO_EVENT_LOG` is now written**, in the same transaction as the step it records: `QUEUED` when a man is enqueued (at registration under a closed gate, or on reinstatement), `ADMITTED` on every promotion, `DEACTIVATED` when a queued man deactivates or is shadow-banned. **Addition:** each entry carries `manual` (true for a force-admit, which the spec asks to be "audit-logged as manual" without saying how); `GET /ratio-events` returns it next to `type` and `at`.
- **Force-admit** answers `404 NOT_FOUND` for a missing user and `409 INVALID_STATE` for anyone not `QUEUED` (so a repeated call after the flip lands is a 409, as the spec warns).
- **Paging** for the trail: `fold.ratio.default-page-size` (20) / `max-page-size` (100), else `422`. `type` filters by `ADMITTED` / `QUEUED` / `DEACTIVATED`.
- **Fix found while testing.** The `users` listener for `AdmittedFromQueue` was declared on the generic payload type, which Spring cannot match at runtime, so **a promotion event never flipped anyone to `ACTIVE`** (only the direct `admitFromQueue` call had been tested). It now takes the event envelope and routes on the payload, like every other listener.