# Modulith Structure — Design Decision

**Scope:** how the Spring Boot Modulith codebase is organized — package layout, module boundaries, inter-module contracts, data ownership, configuration, and testing.

**Status:** locked. All module code must follow the rules in this document. Violations are caught by `@ApplicationModuleTest`.

---

## 1. Base package & module detection

**Base package:** `com.fold.backend`

Spring Modulith auto-detects modules as **direct sub-packages** of the base package. Each sub-package = one module. No registration, no annotations needed (except `commons`, which needs `@ApplicationModule(type = OPEN)`).

```
com.fold.backend/
  FoldApplication.java           — @SpringBootApplication
  commons/                       — shared kernel (OPEN)
  auth/                          — module
  users/                         — module
  matching/                      — module
  messaging/                     — module
  notifications/                 — module
  ratio/                         — module
  analytics/                     — module
```

---

## 2. Module visibility rule

Spring Modulith enforces this automatically:

| Location | Visibility | Example |
|---|---|---|
| Module root package | **PUBLIC** — accessible from other modules | `com.fold.backend.users.InProcessUserClient` |
| Any sub-package | **PRIVATE** — only accessible within the owning module | `com.fold.backend.users.service.UserService` |

There is no `internal/` wrapper package — Spring Modulith treats **all sub-packages** as module-private by default. The directory name is irrelevant; the depth is what matters. Only classes directly in the module's root package are public API.

**Rule: keep the root package minimal.** Only the `InProcessXClient` bean (the module's public API for sync calls) lives there. Everything else — entities, services, repositories, controllers, listeners — goes in sub-packages.

---

## 3. commons — the shared kernel

```
commons/                                      — @ApplicationModule(type = OPEN)
  client/                                     — sync client interfaces
    users/UserClient.java
    ratio/RatioClient.java
    matching/MatchingClient.java
    analytics/AnalyticsClient.java
  events/                                     — event records, grouped by producer
    users/UserRegistered.java
    users/ProfileUpdated.java
    users/UserDeactivated.java
    users/UserShadowBanned.java
    users/UserSuspended.java
    users/UserReactivated.java
    users/ReportFiled.java
    users/ReportResolved.java
    users/BlockCreated.java
    users/BlockRemoved.java
    users/PhotoChanged.java
    auth/SessionRevoked.java
    auth/CredentialUpserted.java
    matching/LikeReceived.java
    matching/MatchCreated.java
    matching/MatchDeleted.java
    matching/ProfileViewed.java
    ratio/AdmittedFromQueue.java
    analytics/DailyViewCountReady.java
    analytics/WeeklySummaryReady.java
    messaging/MessageExpiring.java
  exceptions/                                 — typed families + ErrorCode enum
    ApiException.java
    UnauthorizedException.java
    ForbiddenException.java
    NotFoundException.java
    ConflictException.java
    ValidationException.java
    TooManyRequestsException.java
    TemporaryException.java
    ErrorCode.java                            — single enum, globally unique
  error/
    ErrorDto.java                             — canonical envelope
  web/
    PagedDto.java
    RequestContextFilter.java                 — MDC stamping (observability.md)
  async/
    MdcTaskDecorator.java                     — MDC propagation to async threads
  security/
    JwtVerifier.java                          — RS256 verification, JWKS cache
    AuthenticatedPrincipal.java               — record: sub, oid, sid, dfp, onb, email, roles

```

### commons rules (locked)

| Rule | Why |
|---|---|
| Depends on **nothing** — no module imports | `commons → module` closes a cycle since every module imports commons |
| Contains **only**: interfaces, records, enums, exceptions, security verifier, logging utilities | No controllers, no repositories, no services, no business logic, no DB access |
| `@ApplicationModule(type = OPEN)` | All sub-packages must be accessible from every module |
| Adding/changing a commons contract is **doc-first** | Update `api-design/*.md` row, then the code artifact |
| Records and enums are **additive-only** | No breaking renames — same rule as notification types |
| Jackson annotations allowed | DTOs may carry serialization hints |

---

## 4. Module internal structure (standard layout)

Every module follows the same sub-package convention:

```
{module}/
  InProcessXClient.java             — root package, PUBLIC (only if the module exposes a client)
  domain/                            — JPA entities, enums, value objects
  repository/                        — Spring Data repositories
  service/                           — business logic
  web/                               — REST controllers, WebSocket handlers
  listener/                          — @ApplicationModuleListener event consumers
  config/                            — @ConfigurationProperties, module-specific beans
  sweep/ or batch/                   — scheduled jobs (if any)
  pool/                              — event-fed projections (if any, e.g. matching's eligible-men pool)
```

Sub-packages are created only when needed — a module with no scheduled jobs has no `sweep/` package.

### Per-module breakdown

**auth** — no client interface (modules never call back to auth); publishes `SessionRevoked`, `CredentialUpserted`

```
auth/
  domain/AuthCredential.java, RefreshSession.java, RefreshToken.java
  repository/
  service/AuthService.java
  web/AuthController.java
  config/AuthModuleConfig.java
```

**users** — exposes `InProcessUserClient`; publishes `UserRegistered`, `ProfileUpdated`, `UserDeactivated`, `UserShadowBanned`, `UserSuspended`, `UserReactivated`, `ReportFiled`, `ReportResolved`, `BlockCreated`, `BlockRemoved`, `PhotoChanged`; consumes `AdmittedFromQueue`

```
users/
  InProcessUserClient.java
  domain/User.java, Profile.java, Photo.java, Report.java, Block.java
  repository/
  service/UserService.java, ProfileService.java, ModerationService.java
  web/UserController.java, ProfileController.java, ReportController.java
  listener/RatioEventListener.java
  config/
```

**matching** — exposes `InProcessMatchingClient`; publishes `LikeReceived`, `MatchCreated`, `MatchDeleted`, `ProfileViewed`; consumes `UserRegistered`, `UserDeactivated`, `ProfileUpdated`, `PhotoChanged`, `BlockCreated`, `BlockRemoved`

```
matching/
  InProcessMatchingClient.java
  domain/Like.java, Match.java, FeedCursor.java
  pool/EligibleMenPool.java, PoolEventListener.java
  repository/
  service/LikeService.java, MatchService.java, FeedService.java
  web/LikeController.java, MatchController.java, FeedController.java
  listener/BlockEventListener.java
  sweep/LikeExpirySweep.java
```

**messaging** — no client interface (no other module sync-calls messaging); publishes `MessageExpiring`; consumes `MatchCreated`, `MatchDeleted`, `UserDeactivated`, `BlockCreated`, `BlockRemoved`

```
messaging/
  domain/Conversation.java, ConversationMember.java, Message.java
  repository/
  service/ConversationService.java, MessageService.java
  web/ConversationController.java, WebSocketHandler.java
  listener/MatchEventListener.java, BlockEventListener.java
  sweep/MessageTtlSweep.java
```

**notifications** — no client interface; publishes nothing; consumes `LikeReceived`, `MatchCreated`, `MatchDeleted`, `AdmittedFromQueue`, `SessionRevoked`, `CredentialUpserted`, `DailyViewCountReady`, `WeeklySummaryReady`, `MessageExpiring`, `UserShadowBanned`

```
notifications/
  domain/Notification.java, DeviceToken.java, NotificationPreference.java
  repository/
  service/NotificationService.java, FcmDeliveryService.java, EmailDeliveryService.java
  web/NotificationController.java, DeviceTokenController.java
  listener/MatchEventListener.java, RatioEventListener.java, AuthEventListener.java,
           AnalyticsEventListener.java, MessagingEventListener.java
  config/FcmConfig.java, SmtpConfig.java
```

**ratio** — exposes `InProcessRatioClient`; publishes `AdmittedFromQueue`; consumes `UserDeactivated`, `UserSuspended`, `UserReactivated`

```
ratio/
  InProcessRatioClient.java
  domain/AdmissionQueue.java, RatioEventLog.java
  repository/
  service/GateService.java, QueueService.java
  web/RatioAdminController.java
  listener/UserStatusEventListener.java
  sweep/RatioEvaluationSweep.java
```

**analytics** — exposes `InProcessAnalyticsClient`; publishes `DailyViewCountReady`, `WeeklySummaryReady`; consumes `UserRegistered`, `UserDeactivated`, `UserSuspended`, `UserShadowBanned`, `UserReactivated`, `ProfileViewed`, `LikeReceived`, `MatchCreated`, `ReportFiled`, `ReportResolved`

```
analytics/
  InProcessAnalyticsClient.java
  domain/RatioState.java, ProfileViewEvent.java, EngagementFact.java
  repository/
  service/CounterService.java, ViewPipelineService.java, EngagementService.java
  web/AnalyticsAdminController.java
  listener/UserEventListener.java, MatchingEventListener.java
  batch/DailyViewBatch.java, WeeklySummaryBatch.java
  config/BatchScheduleConfig.java
```

---

## 5. Client interface wiring

**Interface in commons, implementation in the producer.** Callers inject the commons interface; Spring DI wires the producer's bean. No module ever imports another module.

```
commons/client/users/UserClient.java          — interface (commons owns)
users/InProcessUserClient.java                — @Component implements UserClient (producer owns)
matching/service/MatchingService.java          — injects UserClient (Spring DI binds it)
```

The caller (`matching`) only imports `commons.client.users.UserClient`. It has no knowledge of the `users` module. Spring finds the `@Component` bean and injects it.

### Client catalog

| Interface | Producer | Bean | Consumed by |
|---|---|---|---|
| `UserClient` | `users` | `InProcessUserClient` | auth, matching, messaging, notifications, analytics |
| `RatioClient` | `ratio` | `InProcessRatioClient` | users |
| `AnalyticsClient` | `analytics` | `InProcessAnalyticsClient` | ratio |
| `MatchingClient` | `matching` | `InProcessMatchingClient` | messaging |

### After extraction to microservices

The interface stays. A new `HttpUserClient implements UserClient` is added **in commons** (safe — real HTTP needs no producer import). Swap via Spring profile:

```yaml
# application-microservices.yml
fold.users.client: http    # activates HttpUserClient, deactivates InProcessUserClient
```

Callers never change.

---

## 6. Cross-module data ownership

### Own-tables-only rule

Each module owns its tables exclusively. No module reads or writes another module's tables. Cross-module references are **plain ID strings** — never JPA `@ManyToOne` or `@JoinColumn`.

```java
// users module — User entity
@Entity
public class User {
    @Id private String id;
    private String credentialId;   // plain string, NOT @ManyToOne AuthCredential
}

// matching module — Like entity
@Entity
public class Like {
    @Id private String id;
    private String likerId;        // plain string, NOT @ManyToOne User
    private String likedId;        // plain string
}
```

If a module needs data from another module's table, it calls the client interface — never a DB join.

### Table ownership map

| Module | Tables |
|---|---|
| auth | `AUTH_CREDENTIAL`, `REFRESH_SESSION`, `REFRESH_TOKEN` |
| users | `USER`, `PROFILE`, `PHOTO`, `REPORT`, `BLOCK` |
| matching | `LIKE`, `MATCH`, `FEED_CURSOR` |
| messaging | `CONVERSATION`, `CONVERSATION_MEMBER`, `MESSAGE` |
| notifications | `NOTIFICATION`, `DEVICE_TOKEN`, `NOTIFICATION_PREFERENCE` |
| ratio | `ADMISSION_QUEUE`, `RATIO_EVENT_LOG` |
| analytics | `RATIO_STATE`, `PROFILE_VIEW_EVENT`, `ENGAGEMENT_FACT` |

Single Postgres datasource. All tables in one schema. Module ownership is a convention enforced by code review and `@ApplicationModuleTest`.

---

## 7. Event contract

All async inter-module communication uses Spring application events backed by the **Spring Modulith event publication registry** (JPA-persisted, at-least-once, automatic retry).

### Publishing

```java
// In the service that owns the event
applicationEventPublisher.publishEvent(
    new UserRegistered(userId, credentialId, gender, region, status, Instant.now())
);
```

The event record (`UserRegistered`) lives in `commons.events.users` — both publisher and consumer import the same record. The event is persisted in the same transaction as the business operation (outbox pattern).

### Consuming

```java
// In the consuming module's listener package
@ApplicationModuleListener
void onUserRegistered(UserRegistered event) {
    // react — e.g. add to eligible-men pool
}
```

`@ApplicationModuleListener` = `@Async` + `@TransactionalEventListener(phase = AFTER_COMMIT)`. The listener runs asynchronously after the publishing transaction commits. If it fails, the event publication registry retries it.

### Rules

| Rule | Why |
|---|---|
| Event records are Java `record` types in `commons.events` | Immutable, serializable, single source of truth |
| Producers publish only their own events | `users` publishes `users.*` events, never `matching.*` |
| Consumers must be **idempotent** | At-least-once delivery means duplicate deliveries are possible |
| No event chaining across more than 2 hops | If A → B → C, consider whether A should publish directly to C |
| Events carry IDs, not full objects | Consumers refetch via client interfaces if they need current state |

---

## 8. Configuration

Single `application.yml` with module-specific sections. Each module reads its own section via `@ConfigurationProperties` inside its `config/` sub-package.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/fold
  modulith:
    events:
      jdbc:
        schema-initialization: create-if-not-found

fold:
  auth:
    jwks-uri: https://api.wiingman.in/auth/.well-known/jwks.json
    access-token-ttl: 15m
    refresh-token-ttl: 30d
  ratio:
    evaluation-interval: 5m
    region: Bangalore
  analytics:
    reconciliation-interval: 1h
    daily-batch-time: "08:00"
    weekly-batch-day: MONDAY
  messaging:
    message-ttl: 48h
    sweep-interval: 1m
    expiry-warning-before: 2h
  notifications:
    fcm:
      project-id: fold-prod
    smtp:
      host: smtp.gmail.com
```

Environment overrides via Spring profiles: `application-dev.yml`, `application-prod.yml`.

---

## 9. Testing

### Module tests

```java
@ApplicationModuleTest
class UsersModuleTest {
    // bootstraps: users + commons only
    // verifies: no illegal access to other modules' internals
    // runs against: real database (no mocks)
}
```

Each module has its own `@ApplicationModuleTest`. This test:
1. Starts a minimal Spring context with only that module + commons
2. Verifies the module doesn't access another module's sub-packages
3. Verifies the module doesn't directly access another module's tables
4. Runs integration tests against a real DB (Testcontainers for Postgres)

### Architecture verification

```java
@AnalyzeClasses(packages = "com.fold.backend")
class ModulithArchitectureTest {
    @Test
    void verifyModularity() {
        ApplicationModules.of(FoldApplication.class).verify();
    }
}
```

This single test verifies the entire module dependency graph — catches any illegal cross-module access across the whole codebase.

---

## 10. Rules summary

These rules must be followed across the entire repository to maintain modulith consistency:

| # | Rule | Enforcement |
|---|---|---|
| 1 | Commons depends on nothing | `ApplicationModules.verify()` fails on any module import |
| 2 | Modules only import commons, never each other | `ApplicationModules.verify()` + `@ApplicationModuleTest` |
| 3 | Only root-package classes are module-public | Spring Modulith default — all sub-packages are private |
| 4 | Root package contains only the `InProcessXClient` bean (if any) | Code review |
| 5 | No cross-module JPA relations — plain ID strings only | Code review + `@ApplicationModuleTest` table ownership |
| 6 | Cross-module sync = client interface in commons | No direct service imports across modules |
| 7 | Cross-module async = event record in commons | No `@Async` calls across module boundaries |
| 8 | Event listeners are idempotent | At-least-once delivery guarantee requires it |
| 9 | One table owner per module — no shared writes | Table ownership map (§6) is authoritative |
| 10 | Module-specific config under `fold.{module}.*` | `@ConfigurationProperties` in each module's `config/` |
| 11 | Every module has an `@ApplicationModuleTest` | CI gate — no module ships without boundary verification |
| 12 | Consistent sub-package naming: `domain/`, `repository/`, `service/`, `web/`, `listener/`, `config/` | Code review |
| 13 | No `System.out.println` — SLF4J only | Code review + static analysis |
| 14 | Event records are `record` types, additive-only, snake_case JSON | Commons contract rule |
