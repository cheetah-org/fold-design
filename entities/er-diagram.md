# Entity Relationship Diagram

```mermaid
erDiagram
    %% ============ NORTH: AUTH SERVICE ============
    AUTH_CREDENTIAL {
        string id PK
        string uid UK "Firebase UID"
        string user_id FK
        string phone
        datetime created_at
    }
    OTP_CHALLENGE {
        string id PK
        string phone
        string code_hash
        int attempts
        datetime expires_at
        boolean consumed
    }

    %% ============ NORTHWEST: RATIO SERVICE ============
    ADMISSION_QUEUE {
        string id PK
        string user_id FK
        int rank "f(onboarding_time, referral_count)"
        datetime admitted_at
        datetime created_at
    }
    RATIO_EVENT_LOG {
        string id PK
        enum type "ADMITTED | SOFT_PAUSED | DEACTIVATED"
        string user_id FK
        datetime at
    }

    %% ============ WEST: ANALYTICS SERVICE ============
    RATIO_STATE {
        string id PK
        string city
        int active_women
        int active_men
        boolean admission_open
        datetime updated_at
    }
    PROFILE_VIEW_EVENT {
        string id PK
        string viewer_id FK
        string profile_id FK
        datetime viewed_at
        string source
    }
    ENGAGEMENT_FACT {
        string id PK
        string bucket "hour | day"
        string city
        int likes
        int matches
        int new_users
        json ratio_snapshot
    }

    %% ============ CENTER: USER SERVICE ============
    USER {
        string id PK
        string auth_credential_id FK
        enum gender "FEMALE | MALE"
        date dob
        string city
        enum status "ACTIVE | QUEUED | SOFT_PAUSED | SUSPENDED | SHADOW_BANNED"
        int referrals
        datetime created_at
    }
    PROFILE {
        string user_id PK, FK "1:1 USER"
        string bio
        string description
        json preferences "multiple-choice"
    }
    PHOTO {
        string id PK
        string profile_id FK
        int position "0-5"
        string url
        string prompt "optional"
    }
    REPORT {
        string id PK
        string reporter_id FK
        string reported_id FK
        enum category "HARASSMENT | FAKE_PROFILE | UNDERAGE | INAPPROPRIATE | BLOCK"
        enum status "PENDING | NOT_UPHELD | UPHELD"
        string reviewed_by
        datetime created_at
    }
    BLOCK {
        string id PK
        string blocker_id FK
        string blocked_id FK
        string report_id FK, UK "1:1 REPORT"
        datetime created_at
    }
    REFERRAL {
        string id PK
        string referrer_id FK "USER.id - owns the code"
        string referee_id FK "USER.id - optional, null until used"
        string referralCode UK "8-char alphanumeric"
        int usageCount "fixed usage limit"
        datetime expiryDate "created_at + 28 days"
        datetime created_at
    }

    %% ============ EAST: MATCHING SERVICE ============
    LIKE {
        string id PK
        string liker_id FK "woman"
        string liked_id FK "man"
        enum status "PENDING | ACCEPTED | PASSED"
        datetime expires_at
        datetime created_at
    }
    MATCH {
        string id PK
        string woman_id FK
        string man_id FK
        enum status "ACTIVE | UNMATCHED"
        string unmatched_by
        datetime unmatched_at
        datetime created_at
    }
    FEED_CURSOR {
        string user_id PK, FK
        string last_seen_profile_id
        int limit
    }

    %% ============ SOUTHEAST: CHAT SERVICE ============
    CONVERSATION {
        string id PK
        string match_id FK, UK "1:1 MATCH"
        enum status "ACTIVE | DELETED"
    }
    CONVERSATION_MEMBER {
        string id PK
        string conversation_id FK
        string user_id FK
        boolean visible
    }
    MESSAGE {
        string id PK
        string conversation_id FK
        string sender_id FK
        string body
        datetime sent_at
        datetime expires_at "sent_at + 48h"
    }

    %% ============ SOUTH: NOTIFICATION SERVICE ============
    DEVICE_TOKEN {
        string id PK
        string user_id FK
        string fcm_token
        string platform
        datetime last_seen_at
    }
    NOTIFICATION {
        string id PK
        string user_id FK
        enum type "LIKE | MATCH | EXPIRY | QUEUE_UPDATE | WEEKLY_SUMMARY"
        json payload
        enum status "PENDING | SENT | FAILED"
        datetime created_at
    }
    NOTIFICATION_PREFERENCE {
        string user_id PK, FK
        enum type
        boolean enabled
    }

    %% ============ RELATIONSHIPS (interleaved so dagre spreads N/S/E/W) ============

    %% North: Auth -> User
    AUTH_CREDENTIAL ||--o| USER : "creates"

    %% Northwest: Ratio -> User
    USER ||--o{ ADMISSION_QUEUE : "queued"
    USER ||--o{ RATIO_EVENT_LOG : "status event"

    %% West: Analytics -> User
    USER ||--o{ PROFILE_VIEW_EVENT : "viewed"
    USER ||--o{ PROFILE_VIEW_EVENT : "views"

    %% East: Matching -> User
    USER ||--o{ LIKE : "likes (woman)"
    USER ||--o{ LIKE : "is liked (man)"
    USER ||--o{ MATCH : "as woman"
    USER ||--o{ MATCH : "as man"
    USER ||--o{ FEED_CURSOR : "discovery feed"

    %% Southeast: Chat -> Matching/User
    MATCH ||--|| CONVERSATION : "opens"
    CONVERSATION ||--o{ CONVERSATION_MEMBER : "has"
    USER ||--o{ CONVERSATION_MEMBER : "member"
    CONVERSATION ||--o{ MESSAGE : "contains"
    USER ||--o{ MESSAGE : "sends"

    %% South: Notification -> User
    USER ||--o{ DEVICE_TOKEN : "registers"
    USER ||--o{ NOTIFICATION : "receives"
    USER ||--o{ NOTIFICATION_PREFERENCE : "sets"

    %% Center: User Service internal (kept last so they hug USER)
    USER ||--|| PROFILE : "has"
    PROFILE ||--o{ PHOTO : "owns (max 6)"
    USER ||--o{ REPORT : "files as reporter"
    USER ||--o{ REPORT : "is reported"
    USER ||--o{ BLOCK : "blocks as blocker"
    REPORT ||--o| BLOCK : "block creates report"
    USER ||--o{ REFERRAL : "refers (referrer)"
    USER ||--o{ REFERRAL : "referred (referee, nullable)"
```

# user-service

Owns identity, profile, moderation, and referrals. CRUD for `USER`, `PROFILE`, `PHOTO`, `REPORT`, `BLOCK`, `REFERRAL`. Reads `AUTH_CREDENTIAL` (created by Auth Service); applies `status` changes emitted by Ratio Service as events.

```mermaid
erDiagram
    %% Owned entities
    USER {
        string id PK
        string auth_credential_id FK
        enum gender "FEMALE | MALE"
        date dob
        string city
        enum status "ACTIVE | QUEUED | SOFT_PAUSED | SUSPENDED | SHADOW_BANNED"
        int referrals
        datetime created_at
    }
    PROFILE {
        string user_id PK, FK "1:1 USER"
        string bio
        string description
        json preferences "multiple-choice"
    }
    PHOTO {
        string id PK
        string profile_id FK
        int position "0-5"
        string url
        string prompt "optional"
    }
    REPORT {
        string id PK
        string reporter_id FK
        string reported_id FK
        enum category "HARASSMENT | FAKE_PROFILE | UNDERAGE | INAPPROPRIATE | BLOCK"
        enum status "PENDING | NOT_UPHELD | UPHELD"
        string reviewed_by
        datetime created_at
    }
    BLOCK {
        string id PK
        string blocker_id FK
        string blocked_id FK
        string report_id FK, UK "1:1 REPORT"
        datetime created_at
    }
    REFERRAL {
        string id PK
        string referrer_id FK "USER.id - owns the code"
        string referee_id FK "USER.id - optional, null until used"
        string referralCode UK "8-char alphanumeric"
        int usageCount "fixed usage limit"
        datetime expiryDate "created_at + 28 days"
        datetime created_at
    }

    %% External reference (owned by Auth Service)
    AUTH_CREDENTIAL {
        string id PK
    }

    %% Relationships
    AUTH_CREDENTIAL ||--o| USER : "creates (external)"
    USER ||--|| PROFILE : "has"
    PROFILE ||--o{ PHOTO : "owns (max 6)"
    USER ||--o{ REPORT : "files as reporter"
    USER ||--o{ REPORT : "is reported"
    USER ||--o{ BLOCK : "blocks as blocker"
    REPORT ||--o| BLOCK : "block creates report"
    USER ||--o{ REFERRAL : "refers (referrer)"
    USER ||--o{ REFERRAL : "referred (referee, nullable)"
```
# ratio-service

Owns the male admission waitlist and ratio audit trail. CRUD for `ADMISSION_QUEUE`, `RATIO_EVENT_LOG`. Reads `RATIO_STATE` (owned by Analytics Service) to decide `admission_open`; emits status events that User Service applies to `USER`.

```mermaid
erDiagram
    %% Owned entities
    ADMISSION_QUEUE {
        string id PK
        string user_id FK
        int rank "f(onboarding_time, referral_count)"
        datetime admitted_at
        datetime created_at
    }
    RATIO_EVENT_LOG {
        string id PK
        enum type "ADMITTED | SOFT_PAUSED | DEACTIVATED"
        string user_id FK
        datetime at
    }

    %% External references (owned by User / Analytics Service)
    USER {
        string id PK
    }
    RATIO_STATE {
        string id PK
        boolean admission_open
    }

    %% Relationships
    USER ||--o{ ADMISSION_QUEUE : "queued (external)"
    USER ||--o{ RATIO_EVENT_LOG : "status event (external)"
    RATIO_STATE ||--|| ADMISSION_QUEUE : "drives admission (external, no FK)"
```

# auth-service

Thin credential store. CRUD for `AUTH_CREDENTIAL`, `OTP_CHALLENGE`. Mints `AUTH_CREDENTIAL` on OTP verify and hands a `user_id` to User Service; never owns `USER`.

```mermaid
erDiagram
    %% Owned entities
    AUTH_CREDENTIAL {
        string id PK
        string uid UK "Firebase UID"
        string user_id FK
        string phone
        datetime created_at
    }
    OTP_CHALLENGE {
        string id PK
        string phone
        string code_hash
        int attempts
        datetime expires_at
        boolean consumed
    }

    %% External reference (owned by User Service)
    USER {
        string id PK
    }

    %% Relationships
    AUTH_CREDENTIAL ||--o| USER : "maps to (external)"
```

# matching-service

Owns discovery, likes, and matches. CRUD for `LIKE`, `MATCH`, `FEED_CURSOR`. Emits `match_created` events → Chat Service creates `CONVERSATION`.

```mermaid
erDiagram
    %% Owned entities
    LIKE {
        string id PK
        string liker_id FK "woman"
        string liked_id FK "man"
        enum status "PENDING | ACCEPTED | PASSED"
        datetime expires_at
        datetime created_at
    }
    MATCH {
        string id PK
        string woman_id FK
        string man_id FK
        enum status "ACTIVE | UNMATCHED"
        string unmatched_by
        datetime unmatched_at
        datetime created_at
    }
    FEED_CURSOR {
        string user_id PK, FK
        string last_seen_profile_id
        int limit
    }

    %% External references (owned by User / Chat Service)
    USER {
        string id PK
    }
    CONVERSATION {
        string id PK
    }

    %% Relationships
    USER ||--o{ LIKE : "likes (woman, external)"
    USER ||--o{ LIKE : "is liked (man, external)"
    USER ||--o{ MATCH : "as woman (external)"
    USER ||--o{ MATCH : "as man (external)"
    USER ||--o{ FEED_CURSOR : "discovery feed (external)"
    MATCH ||--|| CONVERSATION : "opens (external, event)"
```

# chat-service

Owns conversations, membership visibility, and per-message TTL. CRUD for `CONVERSATION`, `CONVERSATION_MEMBER`, `MESSAGE`. On `match_created` (Matching Service) creates a `CONVERSATION`; on unmatch, Matching flips `MATCH` to `UNMATCHED` and Chat tears down the thread.

```mermaid
erDiagram
    %% Owned entities
    CONVERSATION {
        string id PK
        string match_id FK, UK "1:1 MATCH"
        enum status "ACTIVE | DELETED"
    }
    CONVERSATION_MEMBER {
        string id PK
        string conversation_id FK
        string user_id FK
        boolean visible
    }
    MESSAGE {
        string id PK
        string conversation_id FK
        string sender_id FK
        string body
        datetime sent_at
        datetime expires_at "sent_at + 48h"
    }

    %% External references (owned by Matching / User Service)
    MATCH {
        string id PK
    }
    USER {
        string id PK
    }

    %% Relationships
    MATCH ||--|| CONVERSATION : "opens (external, event)"
    CONVERSATION ||--o{ CONVERSATION_MEMBER : "has"
    USER ||--o{ CONVERSATION_MEMBER : "member (external)"
    CONVERSATION ||--o{ MESSAGE : "contains"
    USER ||--o{ MESSAGE : "sends (external)"
```

# notification-service

Owns push delivery and device registration. CRUD for `DEVICE_TOKEN`, `NOTIFICATION`, `NOTIFICATION_PREFERENCE`. Consumes domain events (like, match, expiry, queue update) to enqueue push payloads.

```mermaid
erDiagram
    %% Owned entities
    DEVICE_TOKEN {
        string id PK
        string user_id FK
        string fcm_token
        string platform
        datetime last_seen_at
    }
    NOTIFICATION {
        string id PK
        string user_id FK
        enum type "LIKE | MATCH | EXPIRY | QUEUE_UPDATE | WEEKLY_SUMMARY"
        json payload
        enum status "PENDING | SENT | FAILED"
        datetime created_at
    }
    NOTIFICATION_PREFERENCE {
        string user_id PK, FK
        enum type
        boolean enabled
    }

    %% External reference (owned by User Service)
    USER {
        string id PK
    }

    %% Relationships
    USER ||--o{ DEVICE_TOKEN : "registers (external)"
    USER ||--o{ NOTIFICATION : "receives (external)"
    USER ||--o{ NOTIFICATION_PREFERENCE : "sets (external)"
```

# analytics-service

Reads domain events, owns aggregates and gender-ratio state. CRUD for `RATIO_STATE`, `PROFILE_VIEW_EVENT`, `ENGAGEMENT_FACT`. `RATIO_STATE` is consumed read-only by Ratio Service; `PROFILE_VIEW_EVENT` powers "X women viewed your profile today" and women's like-delivery insights.

```mermaid
erDiagram
    %% Owned entities
    RATIO_STATE {
        string id PK
        string city
        int active_women
        int active_men
        boolean admission_open
        datetime updated_at
    }
    PROFILE_VIEW_EVENT {
        string id PK
        string viewer_id FK
        string profile_id FK
        datetime viewed_at
        string source
    }
    ENGAGEMENT_FACT {
        string id PK
        string bucket "hour | day"
        string city
        int likes
        int matches
        int new_users
        json ratio_snapshot
    }

    %% External reference (owned by User Service)
    USER {
        string id PK
    }

    %% Relationships
    USER ||--o{ PROFILE_VIEW_EVENT : "viewed (external)"
    USER ||--o{ PROFILE_VIEW_EVENT : "views (external)"
```