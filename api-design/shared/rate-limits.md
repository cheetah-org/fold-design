# Shared — Rate Limits (cross-service policy)

Applies to **every** module API (`auth`, `users`, `notifications`, `matching`, …). One mechanism, configured per endpoint; each service doc keeps only the per-endpoint numbers and points here for semantics.

## Mechanism

- Per-scope counters (per `userId` and/or per IP — the coarser-sounding one wins, `429` is returned).
- Fixed windows (V1, in-memory, deterministic); counters reset on window roll. Review fix: doc previously said "sliding" while endpoint tables assumed fixed windows — standardized on fixed for V1; swap to rolling when a shared store (Redis) lands, no client-visible contract change.
- Every response carries:
  - `X-RateLimit-Limit` (window cap)
  - `X-RateLimit-Remaining`
  - `X-RateLimit-Reset` (epoch seconds)
- Breach → `429 TOO_MANY_REQUESTS` with the canonical envelope (`auth.md` §2) + `Retry-After` header. Failed `429`s are not recorded (they don't consume quota).

## Defaults when a service doesn't specify otherwise

| Scope | Default |
|---|---|
| per `userId` | 600 / min |
| per IP (public endpoints) | 600 / min, 1 200 / 5 min |

Public endpoints (login paths, `/.well-known/*`) are scoped per IP only. Authenticated endpoints are scoped per `userId` (from the verified `oid`/`sub`).

## Enabling / disabling

- Rate limiting is a cross-cutting concern applied at the edge/edge-filter layer, not inside each module's business logic.
- Dev/staging may raise or disable limits by config; production limits are the reviewed values in each service doc's table.

## Per-service tables

Each service doc's rate-limit table lists only endpoint-specific numbers; the header/envelope/scope semantics live here.

- `auth.md` §9.1
- `users` (`user.md`) — §Rate limits table (review fix: pointer previously cited a nonexistent §"Conventions")
- `matching` (`matching.md`) — §Rate limits table
- `notifications.md` §"Rate limits"