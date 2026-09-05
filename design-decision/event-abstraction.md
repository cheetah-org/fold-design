# Event Abstraction — Design Decision

**Scope:** commons-level pub/sub abstraction that decouples module code from the event transport — in-process Spring events today, Kafka after extraction, external webhooks when needed.

**Status:** abstraction lives in `commons.pubsub`; V1 impl is in-process Spring events. Kafka and webhook adapters are future additions that require zero changes to module listener code.

---

## 1. Problem

Modules publish and consume domain events. Today that's Spring's `ApplicationEventPublisher` + `@ApplicationModuleListener`. When we extract to microservices with Kafka, every publisher and consumer call site would need rewriting. The abstraction isolates module code from transport so:

- Publishers call `DomainEventPublisher.publish(payload)` — not Spring's API directly
- Consumers annotate with `@DomainEventListener` — not Spring's annotations directly
- Transport swap (Spring → Kafka) is a config change, not a code change

---

## 2. Core contracts (commons-owned)

### 2.1 `DomainEvent<T>` — generic event wrapper

A single generic record that wraps any payload type. Modules never construct this — the publisher builds it automatically.

```java
public record DomainEvent<T>(
    String topic,              // routing destination: fold.users.user-registered
    String eventType,          // type discriminator: "UserRegistered"
    T data,                    // the payload — plain record (UserRegistered, MatchCreated, etc.)
    EventMetadata metadata     // routing, correlation, partitioning
) {}
```

### 2.2 `EventMetadata` — routing + correlation

```java
public record EventMetadata(
    String eventId,            // UUID — idempotency key for consumers
    String objectId,           // partition key — typically userId; same user → same partition → ordered processing
    Instant publishedAt,       // wall-clock timestamp
    String origin,             // source module: "users", "matching"
    String traceId             // from MDC — stitches event into the originating request's trace
) {}
```

| Field | Purpose now (modulith) | Purpose later (Kafka) |
|---|---|---|
| `eventId` | Idempotency — consumers dedup on this | Same + Kafka deduplication |
| `objectId` | Logging/debugging | **Kafka partition key** — all events for the same user land in the same partition, guaranteeing ordered processing per user |
| `publishedAt` | Logging/debugging | Event time semantics, late-arrival detection |
| `origin` | Logging/debugging | Source identification across services |
| `traceId` | Log correlation within the modulith | Distributed trace correlation across services |

### 2.3 `@ObjectId` — partition key annotation

Marks the field in a payload record that provides the `objectId` (partition key). The publisher reads this via reflection when wrapping the payload into `DomainEvent<T>`.

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ObjectId {}
```

Usage in payload records:

```java
// commons.events.users
public record UserRegistered(
    @ObjectId String userId,       // partition key = userId
    String credentialId,
    String gender,
    String region,
    String status
) {}

// commons.events.matching
public record LikeReceived(
    String likeId,
    String likerId,
    @ObjectId String likedId       // partition key = the woman receiving the like
) {}

public record MatchCreated(
    String matchId,
    @ObjectId String womanId,      // partition key = woman (primary actor in the matching model)
    String manId
) {}
```

**Rule:** every event record in `commons.events.*` must have exactly one `@ObjectId` field. The publisher fails fast at startup if a record is missing the annotation.

**Choosing the `@ObjectId`:** pick the user whose experience depends on ordered processing of this event. For most events that's straightforward (`userId`). For two-party events (like, match), it's the party whose state machine matters most — in fold's women-browse model, that's typically the woman.

### 2.4 `DomainEventPublisher` — publish API

```java
public interface DomainEventPublisher {
    <T> void publish(T event);    // accepts plain payload, NOT DomainEvent<T>
}
```

One method. Module code passes a plain payload record. The publisher implementation:
1. Reads `@ObjectId` from the payload to extract `objectId`
2. Derives `topic` from the payload class (`UserRegistered` → `fold.users.user-registered`)
3. Derives `origin` from the payload's package (`commons.events.users` → `"users"`)
4. Generates `eventId` (UUID), reads `traceId` from MDC
5. Wraps into `DomainEvent<T>` and dispatches via the transport

### 2.5 `@DomainEventListener` — subscriber annotation

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DomainEventListener {
    String consumerGroup() default "";    // default: fold.{consuming-module}
    int maxRetries() default 3;
    int concurrency() default 1;          // parallel consumers (Kafka only)
    boolean batch() default false;        // receive List<DomainEvent<T>> instead of DomainEvent<T>
    boolean ordered() default true;       // respect partition ordering (Kafka only)
    String deadLetterTopic() default "";  // override DLQ topic name
}
```

Module code annotates handler methods with this — never `@ApplicationModuleListener` directly. The method parameter type determines routing:

```java
@DomainEventListener
void onUserRegistered(DomainEvent<UserRegistered> event) {
    UserRegistered data = event.data();
    // event.metadata().objectId()  → the userId
    // event.metadata().traceId()   → for log correlation
}
```

**Type erasure handling:** at runtime, `DomainEvent<UserRegistered>` and `DomainEvent<MatchCreated>` are both `DomainEvent`. The registrar resolves routing via `Method.getGenericParameterTypes()` → `ParameterizedType` → extracts `T` at startup. Standard technique — Spring does the same internally.

---

## 3. Payload records — what changes

The existing event records in `commons.events.*` are plain Java records. They gain one thing: an `@ObjectId` annotation on the partition key field. They do **not** implement any interface.

### Before (current)

```java
public record UserRegistered(String userId, String credentialId, String gender,
                              String region, String status, Instant createdAt) {}
```

### After

```java
public record UserRegistered(@ObjectId String userId, String credentialId,
                              String gender, String region, String status) {}
```

Changes:
- Add `@ObjectId` on the partition key field
- Remove `createdAt` from payload — `EventMetadata.publishedAt` carries the timestamp (no duplication)

### `@ObjectId` mapping for all events

| Event record | `@ObjectId` field | Why |
|---|---|---|
| `UserRegistered` | `userId` | User's own lifecycle |
| `ProfileUpdated` | `userId` | User's own profile |
| `PhotoChanged` | `userId` | User's own photos |
| `UserDeactivated` | `userId` | User's own lifecycle |
| `UserShadowBanned` | `userId` | User's own moderation state |
| `UserSuspended` | `userId` | User's own moderation state |
| `UserReactivated` | `userId` | User's own moderation state |
| `ReportFiled` | `reportedId` | Partition by the user being reported — moderation pipeline processes per-user |
| `ReportResolved` | `reportedId` | Same — resolution affects the reported user's state |
| `BlockCreated` | `blockerId` | Partition by the actor — their block list must update in order |
| `BlockRemoved` | `blockerId` | Same |
| `SessionRevoked` | `credentialId` | Partition by identity — session cleanup must be ordered per credential |
| `CredentialUpserted` | `credentialId` | Same |
| `LikeReceived` | `likedId` | Partition by the woman receiving the like — her inbox ordering matters |
| `MatchCreated` | `womanId` | Partition by the woman — primary actor in matching model |
| `MatchDeleted` | `unmatchedBy` | Partition by the actor — their state machine drives the unmatch |
| `ProfileViewed` | `profileId` | Partition by the viewed user — view counts aggregate per profile |
| `AdmittedFromQueue` | `userId` | User's own admission lifecycle |
| `DailyViewCountReady` | `userId` | User's own notification |
| `WeeklySummaryReady` | `userId` | User's own notification |
| `MessageExpiring` | `conversationId` | Partition by conversation — expiry affects both participants atomically |

---

## 4. Publisher implementation

### 4.1 V1: `InProcessEventPublisher`

```java
@Component
class InProcessEventPublisher implements DomainEventPublisher {

    private final ApplicationEventPublisher springPublisher;

    @Override
    public <T> void publish(T payload) {
        DomainEvent<T> event = wrap(payload);
        springPublisher.publishEvent(event);
    }

    private <T> DomainEvent<T> wrap(T payload) {
        Class<?> type = payload.getClass();
        String objectId = extractObjectId(type, payload);
        return new DomainEvent<>(
            topicFor(type),
            type.getSimpleName(),
            payload,
            new EventMetadata(
                UUID.randomUUID().toString(),
                objectId,
                Instant.now(),
                originFor(type),
                MDC.get("traceId")
            )
        );
    }
}
```

Delegates to Spring's `ApplicationEventPublisher`. The event participates in the caller's transaction via Spring Modulith's event publication registry (outbox pattern, at-least-once, automatic retry on failure).

### 4.2 Future: `KafkaEventPublisher`

```java
@Component
@Profile("kafka")
class KafkaEventPublisher implements DomainEventPublisher {

    private final KafkaTemplate<String, String> kafka;
    private final EventSerializer serializer;

    @Override
    public <T> void publish(T payload) {
        DomainEvent<T> event = wrap(payload);
        String json = serializer.serialize(event);
        kafka.send(event.topic(), event.metadata().objectId(), json);
        //                        ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
        //                        objectId is the Kafka message key
        //                        → same user → same partition → ordered
    }
}
```

Swap via Spring profile — `InProcessEventPublisher` is the default, `KafkaEventPublisher` activates with `--spring.profiles.active=kafka`. Module publish calls don't change.

---

## 5. Subscriber implementation

### 5.1 V1: `InProcessListenerRegistrar`

A `BeanPostProcessor` that scans for `@DomainEventListener` methods and registers them as Spring `@ApplicationModuleListener` handlers:

```java
@Component
class InProcessListenerRegistrar implements BeanPostProcessor {

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        for (Method method : bean.getClass().getDeclaredMethods()) {
            if (method.isAnnotationPresent(DomainEventListener.class)) {
                // Resolve T from DomainEvent<T> via ParameterizedType
                // Register as Spring ApplicationModuleListener equivalent:
                // - @Async execution
                // - @TransactionalEventListener(phase = AFTER_COMMIT)
                // - Event publication registry handles retry
            }
        }
        return bean;
    }
}
```

The registrar translates `@DomainEventListener` semantics into Spring Modulith's mechanisms. Module code only sees `@DomainEventListener`.

### 5.2 Future: `KafkaListenerRegistrar`

Replaces the in-process registrar under the `kafka` profile. For each `@DomainEventListener` method:

| Annotation field | Kafka behavior |
|---|---|
| `consumerGroup` | Kafka consumer group ID. Default: `fold.{consuming-module}` (derived from the listener's package) |
| `maxRetries` | Retry with exponential backoff before sending to DLQ |
| `concurrency` | Number of concurrent Kafka consumers for this listener |
| `batch` | `true`: batch consumer receives `List<ConsumerRecord>`, handler gets `List<DomainEvent<T>>` |
| `ordered` | `true` (default): single consumer per partition, in-order processing. `false`: concurrent processing within a partition (for handlers that don't need ordering) |
| `deadLetterTopic` | Override the DLQ topic name. Default: `fold.{module}.{event}.dlq` |

The registrar handles:
- **Deserialization**: JSON → `DomainEvent<T>` (resolves `T` from the handler method's generic parameter)
- **Partition key routing**: `objectId` → Kafka message key → same user on same partition
- **Offset management**: commit after successful handler execution (at-least-once)
- **Backpressure**: configurable concurrency per listener
- **Dead letter queue**: exhausted retries → DLQ topic with the original event + failure metadata

### 5.3 Ordering guarantee via `objectId`

```
Producer publishes:
  LikeReceived { likedId = "user-A" }   →  objectId = "user-A"  →  partition 3
  MatchCreated { womanId = "user-A" }   →  objectId = "user-A"  →  partition 3
  LikeReceived { likedId = "user-B" }   →  objectId = "user-B"  →  partition 7

Consumer (ordered = true, default):
  partition 3 → single thread → LikeReceived then MatchCreated (in order for user-A)
  partition 7 → single thread → LikeReceived (for user-B, independent)
```

Events for the same user are always processed in publication order. Events for different users are processed in parallel across partitions.

### 5.4 Subscriber contract rules

| Rule | Enforcement |
|---|---|
| Handlers must be **idempotent** | At-least-once delivery (both Spring and Kafka). Dedup on `eventId` if the operation isn't naturally idempotent |
| One event type per handler method | Method parameter type `DomainEvent<T>` determines routing — one `T` per method |
| Handlers must not publish back to the same event type they consume | Prevents infinite loops. Caught by code review |
| Handler failures don't roll back the publisher | Events are post-commit (Spring) or separate transactions (Kafka) |

---

## 6. Webhook adapter — external event ingestion

External systems (FCM, SMTP bounce, future payment gateway) push events to us via HTTP. The webhook adapter normalizes them into plain payload records, then publishes through the same `DomainEventPublisher` — the rest of the system handles them identically to internal events.

### 6.1 Flow

```
External system → POST /webhooks/{provider}
                       │
                  WebhookController (commons)
                       │
                  ┌─────────────┐
                  │ route by     │
                  │ {provider}   │
                  └──────┬──────┘
                         │
                  WebhookAdapter impl (module-owned)
                    1. verify signature
                    2. parse provider payload
                    3. normalize → plain event record
                         │
                  DomainEventPublisher.publish(record)
                    → wraps into DomainEvent<T> with metadata
                         │
                  @DomainEventListener (same as internal events)
```

### 6.2 Contracts

```java
// commons — interface
public interface WebhookAdapter<P, E> {
    String provider();                          // e.g. "fcm", "smtp-bounce"
    boolean verifySignature(String payload,
                            String signatureHeader);
    E normalize(P providerPayload);             // external format → plain event record
    Class<P> payloadType();                     // for deserialization of incoming JSON
}
```

```java
// commons — generic controller
@RestController
@RequestMapping("/webhooks")
@ConditionalOnBean(WebhookAdapter.class)
class WebhookController {

    private final Map<String, WebhookAdapter<?, ?>> adapters;
    private final DomainEventPublisher publisher;

    @PostMapping("/{provider}")
    ResponseEntity<Void> handle(@PathVariable String provider,
                                 @RequestBody String rawPayload,
                                 @RequestHeader HttpHeaders headers) {
        WebhookAdapter adapter = adapters.get(provider);
        if (adapter == null) return ResponseEntity.notFound().build();

        String sig = headers.getFirst(signatureHeaderFor(provider));
        if (!adapter.verifySignature(rawPayload, sig)) {
            return ResponseEntity.status(401).build();
        }

        Object parsed = objectMapper.readValue(rawPayload, adapter.payloadType());
        Object event = adapter.normalize(parsed);
        publisher.publish(event);       // wraps into DomainEvent<T>, same pipeline

        return ResponseEntity.ok().build();
    }
}
```

### 6.3 Module-owned adapter example

```java
// notifications/webhook/FcmWebhookAdapter.java (module-private)
@Component
class FcmWebhookAdapter implements WebhookAdapter<FcmDeliveryReceipt, PushDeliveryFailed> {

    @Override
    public String provider() { return "fcm"; }

    @Override
    public boolean verifySignature(String payload, String sig) {
        // HMAC-SHA256 verification against FCM shared secret
    }

    @Override
    public PushDeliveryFailed normalize(FcmDeliveryReceipt receipt) {
        return new PushDeliveryFailed(receipt.getRegistrationToken(), receipt.getError());
    }

    @Override
    public Class<FcmDeliveryReceipt> payloadType() {
        return FcmDeliveryReceipt.class;
    }
}
```

The adapter returns a plain event record (`PushDeliveryFailed`) with an `@ObjectId` field. The publisher wraps it into `DomainEvent<PushDeliveryFailed>` automatically — same as any internal event.

### 6.4 Webhook sources relevant to fold

| Provider | Event | Internal event record | Consuming module |
|---|---|---|---|
| FCM | Delivery failure/feedback | `PushDeliveryFailed` | notifications |
| SMTP | Email bounce | `EmailBounced` | notifications |
| Payment gateway (future) | Payment confirmed/failed | `PaymentCompleted` / `PaymentFailed` | future payments module |

Webhook adapters are added only when a provider is integrated — none exist in V1.

---

## 7. Wire format (JSON)

The on-the-wire representation when events cross process boundaries (Kafka, DLQ storage, webhook delivery):

```json
{
  "topic": "fold.users.user-registered",
  "event_type": "UserRegistered",
  "data": {
    "user_id": "9f3a1c4e-...",
    "credential_id": "abc-...",
    "gender": "MALE",
    "region": "Bangalore",
    "status": "ACTIVE"
  },
  "metadata": {
    "event_id": "d7c4e6a8-...",
    "object_id": "9f3a1c4e-...",
    "published_at": "2026-09-05T10:00:00.123Z",
    "origin": "users",
    "trace_id": "8f2c1a3d4b5e4c6f9a0d1e2f3a4b5c6d"
  }
}
```

In the modulith, the `DomainEvent<T>` is an in-process Java object — not serialized to JSON. The wire format is defined now so Kafka serialization produces a known shape from day 1.

---

## 8. Topic & consumer group naming (Kafka)

Defined now so the `eventType` / `topic` convention is consistent from the start.

| Convention | Pattern | Example |
|---|---|---|
| Topic | `fold.{module}.{event-name-kebab}` | `fold.users.user-registered` |
| Consumer group | `fold.{consuming-module}` | `fold.matching` |
| Dead letter topic | `fold.{module}.{event-name-kebab}.dlq` | `fold.users.user-registered.dlq` |

One topic per event type. One consumer group per consuming module (all instances of that module share the group). A module consuming 5 different events has 5 listeners but 1 consumer group.

---

## 9. Package structure in commons

```
commons/pubsub/
  DomainEvent.java                    — generic record: DomainEvent<T>(topic, eventType, data, metadata)
  EventMetadata.java                  — record: eventId, objectId, publishedAt, origin, traceId
  ObjectId.java                       — field annotation: marks the partition key in payload records
  DomainEventPublisher.java           — publish API interface: <T> void publish(T event)
  DomainEventListener.java            — subscriber annotation
  EventSerializer.java                — DomainEvent<T> ↔ JSON
  webhook/
    WebhookAdapter.java               — interface: provider, verify, normalize, payloadType
    WebhookController.java            — generic POST /webhooks/{provider} endpoint
  support/
    InProcessEventPublisher.java      — V1: wraps payload → DomainEvent<T>, delegates to Spring
    InProcessListenerRegistrar.java   — V1: translates @DomainEventListener → Spring listeners
```

Future additions (not in V1):

```
  support/
    KafkaEventPublisher.java          — wraps payload → DomainEvent<T>, serializes, sends to Kafka with objectId as message key
    KafkaListenerRegistrar.java       — Kafka consumer: deser, offset, retry, DLQ, partition-ordered processing
```

---

## 10. Module usage — complete example

### Publishing (in the users module)

```java
// users/service/UserService.java
@Service
class UserService {

    private final DomainEventPublisher eventPublisher;

    void register(CreateUserRequest req) {
        User user = userRepository.save(/* ... */);

        eventPublisher.publish(
            new UserRegistered(user.getId(), req.credentialId(),
                               req.gender(), req.region(), "ACTIVE")
        );
        // publisher wraps into DomainEvent<UserRegistered>:
        //   topic     = "fold.users.user-registered"
        //   eventType = "UserRegistered"
        //   data      = the UserRegistered record
        //   metadata  = { eventId=UUID, objectId=user.getId(), publishedAt=now,
        //                  origin="users", traceId=MDC.get("traceId") }
    }
}
```

### Consuming (in the matching module)

```java
// matching/listener/UserEventListener.java
@Component
class UserEventListener {

    @DomainEventListener
    void onUserRegistered(DomainEvent<UserRegistered> event) {
        UserRegistered data = event.data();
        if ("MALE".equals(data.gender())) {
            eligibleMenPool.add(data.userId(), data.region());
        }
        // event.metadata().objectId()     → "9f3a1c4e-..." (same as data.userId())
        // event.metadata().traceId()      → traces back to the registration request
        // event.metadata().publishedAt()  → when the event was published
    }
}
```

---

## 11. What to build now vs later

| Component | V1 (modulith) | Later |
|---|---|---|
| `DomainEvent<T>` record | Build | — |
| `EventMetadata` record | Build | — |
| `@ObjectId` annotation | Build | — |
| `DomainEventPublisher` interface | Build | — |
| `@DomainEventListener` annotation | Build | — |
| `EventSerializer` | Build (used for DLQ logging, debugging) | Used for Kafka ser/deser |
| `InProcessEventPublisher` | Build | Deactivated by profile |
| `InProcessListenerRegistrar` | Build | Deactivated by profile |
| `WebhookAdapter` interface | Define | Build when FCM/SMTP integrated |
| `WebhookController` | — | Build with first webhook adapter |
| `KafkaEventPublisher` | — | Build on extraction |
| `KafkaListenerRegistrar` | — | Build on extraction |
| Batch consumption (`batch = true`) | — | Analytics high-volume consumers |
| Concurrency config | — | Kafka parallel consumers |

V1 surface: 7 classes. The contract is designed so Kafka and webhook adapters slot in without changing any module code.

---

## 12. Migration checklist (modulith → Kafka)

When extracting to microservices:

1. Add `spring-kafka` dependency
2. Implement `KafkaEventPublisher` + `KafkaListenerRegistrar` in `commons.pubsub.support`
3. Create Kafka topics matching the naming convention (§8)
4. Activate via `--spring.profiles.active=kafka`
5. `InProcessEventPublisher` and `InProcessListenerRegistrar` deactivate
6. Module code: **zero changes** — same `eventPublisher.publish(payload)`, same `@DomainEventListener`
7. `objectId` automatically becomes the Kafka message key → partition affinity → ordered processing per user
8. Verify idempotency — at-least-once semantics remain, but duplicates may arrive from different partitions
9. Add webhook adapters if external push integrations are live

---

## 13. Non-rules

- Modules never import `ApplicationEventPublisher` or `@ApplicationModuleListener` directly — always through the abstraction.
- Payload records in `commons.events.*` are plain Java records with `@ObjectId` — they do not implement any interface.
- Every payload record must have exactly one `@ObjectId` field — the publisher validates this at startup.
- Modules publish plain payloads (`eventPublisher.publish(payload)`), never construct `DomainEvent<T>` or `EventMetadata` directly — the publisher builds the wrapper.
- The `WebhookController` lives in commons but only activates when at least one `WebhookAdapter` bean exists (`@ConditionalOnBean`).
- Webhook secrets are external config (`fold.webhooks.{provider}.secret` in `application.yml`) — never hardcoded.
- Event records remain additive-only and snake_case JSON — same contract as before this abstraction.
