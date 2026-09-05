# Inter-Service Communication — Design Decision

**Scope:** how services talk to each other in the four core flows — registration, like & match, chat, unmatch (`core-flow/*.md`).

**Status:** aligns with the modulith rules in `backend/CLAUDE.md`. Service names below follow the core-flow docs; §2 maps them to the backend's actual modules.

---

## 1. Communication model — two channels

Every interaction between two services is **either synchronous or asynchronous**, by intent, and the choice maps directly onto how the app is (and can later be) deployed.

| Channel | What it means | Mechanism in the modulith | Mechanism after extraction to microservices |
|---|---|---|---|
| **SYNC** — "I need the answer now to proceed" | Caller blocks until the producer returns a value | **Client interface owned by the caller**, backed by a direct in-process call into the producer module's public API (`InProcessUserClient` today) | Same interface, new `HttpUserClient` impl — swapped via Spring profile/config, callers unchanged |
| **ASYNC** — "Something happened, others may react independently" | Fire-and-forget event; producer does not wait | Standard **Spring application event** (`@ApplicationModuleListener`, backed by the Modulith event-publication registry so events survive restarts) | Same publish call, externalized via the Modulith **outbox** to a broker (Kafka/RabbitMQ) |
| **CLIENT** (context only) | Mobile client → service over HTTP/WebSocket | Not an inter-service channel | n/a |
| **DB** (context only) | A service reading/writing the shared Postgres | Single datasource, module-owned tables (rule 7) | Separate DB per service |

**Rule of thumb for choosing:** if the caller's own logic cannot continue without the answer → **SYNC**. If the producer merely announces something and other services react on their own schedule → **ASYNC**.

---

## 2. Service ↔ module mapping

The core-flow docs name *services*; the backend implements them as *modules* in one deployable. `inter-service` = `inter-module` here.

| Core-flow service | Backend module | Notes |
|---|---|---|
| Auth Service | `auth` | Infrastructure, not a peer domain: sits in front of every request, resolves principal (userId + roles) and verifies the Firebase ID token from Google sign-in. **Not an event participant; modules never call it back.** |
| User Service | `users` | ONE source of truth for `User`. Hosts account state + settings **plus `PROFILE`/`PHOTO` (bio, photos, captions) and `REPORT`/`BLOCK` (moderation)** — the ER diagram overrules the earlier "separate `profile` module" idea, so profile is NOT its own module here. Owns onboarding: client completes a short form and calls `POST /users`. |
| Chat Service | `messaging` | Conversations + messages. Name differs from the flow docs. |
| Matching Service | `matching` | Swipes, likes, match records, match algorithm. |
| Notification Service | `notifications` | Push delivery via external FCM; email/SMS later. |
| Ratio Service | `ratio` **(new module)** | Gender-ratio gate + waiting queue from `reg-flow`. **Decision: dedicated `ratio` module** — sync admission calls from `users` (during `POST /users`), async queue events outward. |
| *(not in flows)* | `discovery` | Feeds the woman's deck; owns the event-fed candidate pool, see §7.2. |
| *(not in flows)* | `commons` | Shared kernel — sync client **interfaces**, DTO contracts for both sync returns and event records, typed exceptions + `ErrorCode` registry, error/paged envelopes, and `commons.security` (the folded Auth Lib). Depends on nothing; see `api-design/commons.md`. |
| Database | Single Postgres | One datasource; each module owns its own tables; cross-module references are plain IDs. |
| Firebase | External | All-Firebase for external services: **Firebase Auth** (Google sign-in), **FCM** (push), **Firebase Storage** (photos). Out of module boundaries. |

---

## 3. Channel classification guide

| Situation | Channel |
|---|---|
| Caller can't proceed without the answer — e.g. admission gate, a profile read the user's action depends on | **SYNC** (client interface) |
| Something happened, others react independently — like received, match created, message expiry | **ASYNC** (domain event) |
| One-off read that must be fresh, low volume | **SYNC** via `commons` DTO |
| High-frequency read, staleness-tolerant — e.g. the eligible-profiles pool served to the feed | **ASYNC event-fed projection** (never per-card sync calls) |

---

## 4. Per-flow breakdowns

Solid arrows (`->>`) = synchronous; dashed arrows (`-->>`) = asynchronous. Diagrams are faithful to `core-flow/*.md`; arrow *style* only encodes the channel.

### 4.1 Registration — `reg-flow`

Onboarding is **two steps**: Firebase Google sign-in (auth module) → short form + `POST /users` (users module). The users module owns account creation; auth only verifies the ID token and answers "is this a returning user?".

```mermaid
sequenceDiagram
    participant User as User
    participant Client as Mobile Client
    participant Auth as Auth Service
    participant UserSvc as User Service (users)
    participant Ratio as Ratio Service
    participant Notif as Notification Service
    participant DB as Database
    participant Firebase as Firebase (Google OAuth)

    User ->> Client: Sign in with Google
    Client ->> Firebase: OAuth sign-in
    Firebase -->> Client: Firebase ID token (name, DOB, email)
    Client ->> Auth: Verify Firebase ID token (POST /auth/google)
    Auth ->> DB: Upsert AUTH_CREDENTIAL (google_sub, email)
    Auth -->> Client: access token + onboarded flag
    alt New user (no USER row yet)
        Client ->> UserSvc: POST /users (gender, city, dob if Google lacks it)
        UserSvc ->> DB: Age gate check (21+, server-side)
        UserSvc ->> Ratio: Check gender ratio (SYNC via RatioClient)
        alt Woman OR man with ratio OK
            Ratio -->> UserSvc: Admit
            UserSvc ->> DB: Create USER + empty PROFILE (status ACTIVE)
        else Man with ratio not OK
            Ratio -->> UserSvc: Queue
            UserSvc ->> DB: Create USER (status QUEUED), enqueue
            loop Every ratio check cycle
                Ratio -->> UserSvc: AdmittedFromQueue (ASYNC event)
                UserSvc ->> DB: Update USER status -> ACTIVE
                Ratio -->> Notif: AdmittedFromQueue (same event, fan-out)
                Notif -->> Client: Queue update push (FCM)
            end
        end
        UserSvc -->> Client: 201 created (user + queue position)
    else Returning user
        Auth -->> Client: existing session
    end
```

| # | From → To | Interaction | Channel | Modulith implementation | After extraction |
|---|---|---|---|---|---|
| R1 | UserSvc → Ratio | Check gender ratio during `POST /users` | **SYNC** | In-process client-interface call (`RatioClient.evaluateGenderRatio(city, gender)`) into the `ratio` module | HTTP client-interface |
| R2 | Ratio → UserSvc | Admit from queue | **ASYNC** | Domain event `AdmittedFromQueue` → users listener flips account to `ACTIVE` | Broker + outbox → listener |
| R3 | Ratio → Client | Admission push ("You're in") | **ASYNC** | Domain event `AdmittedFromQueue` → `notifications` → FCM | Broker → `notifications` → FCM |
| — | Client → Firebase | OAuth sign-in | EXTERNAL | Firebase Auth (Google) | n/a (external) |
| — | UserSvc → DB | User create / status update | DB | `users` module writes its own row (own tables, rule 7); auth never writes `USER` | — |

### 4.2 Like & Match — `like-match-flow`

```mermaid
sequenceDiagram
    participant Woman as Woman
    participant Client as Mobile Client
    participant Matching as Matching Service
    participant UserSvc as User Service
    participant Notif as Notification Service
    participant Man as Man
    participant DB as Database

    Woman ->> Client: Browse feed
    Client ->> Matching: Get profiles
    Matching ->> DB: Get eligible men profiles
    DB -->> Matching: Profiles
    Matching -->> Client: Feed profiles
    Client -->> Woman: Display feed
    Woman ->> Client: Like profile
    Client ->> Matching: Create like
    Matching ->> DB: Save like
    Matching -->> Notif: Send push notification
    Notif -->> Man: New like notification
    Man ->> Client: View like
    Client ->> Matching: Get like details
    Matching ->> DB: Get like and woman's profile
    DB -->> Matching: Like and profile
    Matching -->> Client: Like details
    Client -->> Man: Display woman's profile
    Man ->> Client: Accept like
    Client ->> Matching: Accept like
    Matching ->> DB: Create match, update like status
    Matching -->> Notif: Send match notifications
    Notif -->> Woman: Match notification
    Notif -->> Man: Match notification
```

| # | From → To | Interaction | Channel | Modulith implementation | After extraction |
|---|---|---|---|---|---|
| L1 | users → matching (implied) | Keep the eligible-profiles pool fresh | **ASYNC** | Event-fed local projection: `UserRegistered`, `UserDeactivated`, `ProfileUpdated`, `PhotoChanged` listeners update the pool | Broker → projection |
| L2 | Matching → Notif | New-like push to the man | **ASYNC** | Domain event `LikeReceived` → `notifications` → FCM | Broker → `notifications` → FCM |
| L3 | Matching → users/profile (implied) | View like + woman's profile (`Get like and woman's profile`) | **SYNC** | Client-interface `UserClient`/`ProfileClient` reads (or maintained projection); never a DB join | HTTP client-interface |
| L4 | Matching → Notif | Match notifications to both | **ASYNC** | Domain event `MatchCreated` → `notifications` → FCM | Broker → `notifications` → FCM |
| — | Matching → DB | `Get profiles` / save like | DB | In the real codebase the deck is served by `discovery`, not `matching` — see §7.2 | — |

### 4.3 Chat — `chat-flow`

```mermaid
sequenceDiagram
    participant User1 as User1
    participant Client1 as Mobile Client (User1)
    participant Chat as Chat Service
    participant DB as Database
    participant Notif as Notification Service
    participant Client2 as Mobile Client (User2)
    participant User2 as User2

    User1 ->> Client1: Write message
    Client1 ->> Chat: Send message
    Chat ->> DB: Save message with TTL
    Chat -->> Client2: New message
    Client2 ->> User2: Display message
    loop Every minute
        Chat ->> DB: Check for expiring messages
        alt Message expires
            DB -->> Chat: Expiring messages
            Chat ->> DB: Delete message
            Chat -->> Notif: Send expiry notification
            Notif -->> Client1: Message expiry notification
            Client1 ->> User1: Display notification
            Notif -->> Client2: Message expiry notification
            Client2 ->> User2: Display notification
        else No expiring messages
            DB -->> Chat: No expiring messages
        end
    end
```

| # | From → To | Interaction | Channel | Modulith implementation | After extraction |
|---|---|---|---|---|---|
| C1 | Chat → Notif | Message-expiry notification to both | **ASYNC** | Scheduled job publishes `MessageExpiring` → `notifications` → FCM | Broker → `notifications` → FCM |
| — | Chat → Client2 | Real-time `New message` | CLIENT | Spring WebSocket push, server→client (not a domain event) | n/a |

### 4.4 Unmatch — `unmatch-flow`

```mermaid
sequenceDiagram
    participant User1 as User1
    participant Client1 as Mobile Client (User1)
    participant Matching as Matching Service
    participant Chat as Chat Service
    participant DB as Database
    participant Client2 as Mobile Client (User2)
    participant User2 as User2

    User1 ->> Client1: Unmatch
    Client1 ->> Matching: Unmatch request
    Matching ->> DB: Update match status
    Matching -->> Chat: Delete chat
    Chat ->> DB: Delete messages
    Matching -->> Client1: Unmatch successful
    Client1 ->> User1: Chat removed
    Matching -->> Client2: Match update
    Client2 ->> User2: "User no longer exists"
    User2 ->> Client2: Delete chat (optional)
    Client2 ->> Chat: Delete chat request
    Chat ->> DB: Delete messages (if not already deleted)
```

| # | From → To | Interaction | Channel | Modulith implementation | After extraction |
|---|---|---|---|---|---|
| U1 | Matching → Chat | Delete conversation on unmatch | **ASYNC** (recommended — flow draws it as a sync call, see §7.4) | Domain event `MatchDeleted` → `messaging` listener purges the conversation | Broker → `messaging` listener |
| U2 | Matching → Client2 | Real-time `Match update` ("User no longer exists") | CLIENT | WebSocket / FCM push | n/a |
| — | Client2 → Chat | Delete chat (optional, initiator-side) | CLIENT | HTTP from client | n/a |

---

## 5. Cross-flow decision table

Every inter-service interaction from the four flows, consolidated.

| Interaction | From → To | Flow | Channel | Why | Modulith impl | After extraction |
|---|---|---|---|---|---|---|
| Check gender ratio | users → Ratio | reg | **SYNC** | users cannot proceed with onboarding without the admission verdict | Client-interface in-process call into the `ratio` module | HTTP client-interface |
| Admit from queue | Ratio → Auth | reg | **ASYNC** | Background admission, no blocking answer | `AdmittedFromQueue` event → listener flips user active | Broker + outbox |
| Admission push ("You're in") | Ratio → Client | reg | **ASYNC** | Announcement, others react | `AdmittedFromQueue` event → `notifications` → FCM | Broker → FCM |
| Feed the eligible pool | users/profile → matching | like & match | **ASYNC** | High-frequency, staleness-tolerant | Event-fed projection | Broker → projection |
| New-like push | Matching → Notif | like & match | **ASYNC** | Reaction to `LikeReceived` | Event → `notifications` → FCM | Broker → FCM |
| Like details + woman's profile | Matching → users/profile | like & match | **SYNC** | Man's action needs the profile now | Client-interface `UserClient`/`ProfileClient` | HTTP client-interface |
| Match notifications | Matching → Notif | like & match | **ASYNC** | Reaction to `MatchCreated` | Event → `notifications` → FCM | Broker → FCM |
| Message-expiry notification | Chat → Notif | chat | **ASYNC** | Scheduled announcement | `MessageExpiring` event → `notifications` → FCM | Broker → FCM |
| Delete conversation on unmatch | Matching → Chat | unmatch | **ASYNC** | No blocking answer needed for the unmatch response | `MatchDeleted` event → `messaging` purges | Broker → `messaging` |

**Pattern:** swipe/match/queue/alarm outcomes are **events** (ASYNC); anything a user's action depends on *right now* (admission verdict, profile read) is a **client-interface call** (SYNC).

---

## 6. Client-interface swap plan

- Interfaces live in **`commons`** (`commons.client.<module>.<X>Client` — pure interfaces referencing only commons types); the **in-process impl bean lives in the producer module** (`<X>InProcessClient implements <X>Client`, calls the producer's own public API, exposed via `@Bean`). Consumers inject the commons interface — Spring DI does the binding; no consumer ever imports a producer.
  ```
  commons/client/users/UserClient.java            (interface — commons owns it)
  users/InProcessUserClient.java                  (impl bean — producer owns it, calls its own API)
  ```
- On extraction, add the HTTP implementation **in commons** (safe there — real HTTP needs no producer import) and rebind via config:
  ```
  commons/client/users/HttpUserClient.java        (later: REST call to users service)
  ```
  Swap via Spring profile/config. **Callers never change.** Why impl-in-commons is banned today: it would import the producer's API → `commons → producer`; every module already imports commons, so that closes a Modulith-verified cycle. (Supersedes the earlier "caller-owned interfaces in `internal/client/`" arrangement — `api-design/commons.md` is authoritative.)
- Apply the interface (not a direct service-class import) to modules likely to be extracted or called at high frequency (`discovery`, `matching`, and friends of `users`). All dialogue DTOs live in `commons` — the contracts' single home.
- Async events need no caller change either: the publish call stays identical and the Modulith outbox externalizes it to a broker when the modules are split.

---

## 7. Deviations from the backend modulith rules (call-outs)

The flows above are faithful to the core-flow diagrams. Where the real backend differs, it's called out here — nothing was silently rewritten.

1. **Auth creates the user (reg-flow) vs `users` owns `User`.** Onboarding is two steps: (a) the `auth` module verifies the Firebase ID token and upserts `AUTH_CREDENTIAL` against `google_sub`, returning an access token + an `onboarded` flag; (b) the client fills the short form with whatever Google did not provide (`gender`, `city`, and `dob` when the Google account lacks it) and calls `POST /users` **inside the `users` module** — the one place `USER` is written. `users` enforces the 21+ gate server-side, then runs the ratio admission via `RatioClient` (R1). The flow's old `Auth ->> DB: Create user` maps to that write. After a successful registration, `users` publishes `UserRegistered`, and discovery/matching/messaging/notifications seed their local projections from it — a fan-out the flow diagram doesn't show.

2. **`Matching` serves the feed (like-match-flow) vs `discovery` owns the deck.** The woman's feed and the eligible-men pool belong to `discovery`; `matching` handles swipes/likes/match records. The pool is an event-fed projection (L1), not a per-request query. Deck cards are composed at the edge from `users` data (account + profile + photos, one module) via client-interface reads.

3. **`Get like and woman's profile` is drawn as a DB read.** In the backend that detail view is built by reading `users`/`profile` through their public APIs (client-interface sync call or a maintained projection) — never a cross-module DB join.

4. **Unmatch chat deletion is drawn as a direct call.** A `MatchDeleted` event (U1) is the recommended channel: `matching` announces, `messaging` purges; the unmatch response to the initiator must not block on message deletion.

5. **Ratio Service is its own module — finalized.** The gender-ratio gate + waiting queue get a dedicated `ratio` module in the backend. The admission gate stays a **sync** client-interface call from auth (R1); queue admission and queue-update notifications are **async** events (R2/R3) with `notifications` doing the push. To land it: reserve the `com.fold.backend.ratio` package (public API + `internal/`), own its tables/entities, add an `@ApplicationModuleTest`, and wire `RatioClient` as an in-process adapter. This supersedes the earlier "TBD" call-out.

6. **Naming differs.** `Chat Service` = `messaging` module. `Notification Service` = `notifications` module + external FCM/email/SMS delivery. `Database` = single shared Postgres with per-module tables (rule 7).

7. **Real-time push (WebSocket/FCM)** is a client-facing channel, not a domain event, and does not cross module boundaries the same way — server-to-client only.