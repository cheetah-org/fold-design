# Design Document — V1

**Dating app where women date younger men. Bangalore first.**

---

## What Is This

A dating app with one rule: the woman must always be older than the man she matches with. Women browse and choose. Men wait to be chosen. Women always outnumber men on the platform.

---

## Who It's For

- Women, 21+, Bangalore
- Men, 21+, Bangalore
- The woman in any match is always older than the man — the app enforces this structurally

---

## The Rules

### 1. Age Gate
- Minimum age to register: 21 for everyone
- A woman is only shown men who are strictly younger than her
- A man is only shown to women who are strictly older than him
- Age is taken from the date of birth fetched from the user's Google account at registration

### 2. Women Outnumber Men — Always
- At any point, the number of active women must be greater than or equal to the number of active men
- Enforced at city level (Bangalore for V1)
- If a man tries to register when counts are equal, he enters a waiting queue
- If ratio tips because women leave, new male registrations are paused and the most recently admitted men are soft-paused — notified, not deleted
- Active conversations are never interrupted by ratio enforcement

### 3. Women Browse, Men Receive
- Only women have a discovery feed — they scroll through men's profiles
- Men cannot browse, search, or discover women's profiles at all
- Men open the app to one screen: a list of women who liked them
- When a man receives a like, he can view her profile and either accept or pass
- Accept = mutual match, chat opens
- Pass = she disappears, she is not notified

### 4. Disappearing Chats
- All messages auto-delete 48 hours after they are sent
- The clock is per-message
- No extensions, no recovery
- Before the last message expires, both users get a notification: "Your conversation is expiring — made plans yet?"

### 5. No Screenshots, No Sharing
- Screenshots and screen recording are blocked on all profile and chat screens
- Copy-paste is disabled in chat
- Photos carry an invisible watermark tied to the viewer's account so leaks can be traced

### 6. No Unsend
- Neither party can unsend a message once sent
- Messages only disappear when the 48-hour timer runs out

---

## Onboarding

### Registration Flow
```
Download → Sign in with Google 
→ Fetch name, DOB, email from Google → Age check (must be 21+) → Profile creation → Done
```

### For Men — Queue Experience
- If the ratio doesn't allow admission, the man enters a waiting queue
- He sees his position and estimated wait time
- While waiting, he can complete his profile
- When admitted: push notification "You're in"

---

## Profiles

- Up to 6 photos
- Bio: free text, anything they want
- Description: free text, anything they want
- No mandatory fields beyond the above, no curated prompts, no tags

---

## How Matching Works

### Step 1: She Browses
- Woman opens the app and sees a feed of men's profiles
- Only men strictly younger than her are shown
- Discovery algorithm to be decided at build time

### Step 2: She Likes
- She finds a profile she's interested in and sends a like
- The like appears in his inbox

### Step 3: He Decides
- He opens the app and sees her like
- He views her full profile
- He accepts (mutual match, chat opens) or passes (she disappears, not notified)

### Step 4: Chat
- On mutual match, both can see each other's profiles and a chat window opens
- Text only
- All messages disappear after 48 hours per message
- No read receipts
- No unsend

---

## Unmatching

Either party can unmatch at any time.

**When someone unmatches:**
- The person who unmatched: the other person's chat disappears immediately, completely gone
- The person who was unmatched: the chat window remains but shows "User no longer exists" — profile is not accessible, messages cannot be sent. They can delete this conversation from their end whenever they want, like clearing a chat in WhatsApp

The same message — "User no longer exists" — is shown whether someone unmatched or deleted their account entirely. The other person never knows which one happened.

---

## Reporting & Moderation

- Both men and women can report any user
- Report categories: harassment, fake profile, underage suspicion, inappropriate behavior
- On report: reported user is immediately shadow-banned — they can still use the app but are not shown to anyone and their messages don't deliver
- Moderation team reviews the report:
  - **Not upheld:** user is unbanned, no record
  - **Upheld:** user is permanently suspended
- If a user files multiple reports that are consistently found to be false, that user is also permanently suspended

### Blocking
- Either user can block the other at any time
- Instant, no notification to the blocked person
- Blocked person sees the profile as if it doesn't exist

---

## Men's Experience When Inbox Is Empty

Since men can't browse, an empty inbox means nothing to do. To prevent uninstalls:

- Anonymous count: "X women viewed your profile today" — no names, no photos, just a number
- Push notification the moment a new like arrives
- Weekly summary: "Your profile was viewed Y times this week"

This is intentionally minimal. Men who leave due to a slow inbox are replaced by the queue. The app is built around the woman's experience.

---

## Monetization (V1)

Free tier is fully functional. No core feature is paywalled.

**Free for everyone:**
- Full matching and chatting
- Women: unlimited browsing and likes
- Men: see all incoming likes, accept or pass, full chat

**Premium for men:**
- Queue priority
- Spotlight — boost profile visibility in women's feeds for a short window
- Detailed profile view analytics

**Premium for women:**
- Like delivery insights — how many of her sent likes were seen vs ignored (counts only, no names)
- Advanced filters on the discovery feed

No ads. Ever.

---

## Decisions Locked

| Question | Decision |
|---|---|
| Age gate | 21+ for everyone |
| Younger only or same-age? | Strictly younger only |
| Age verification | DOB fetched from Google account at registration |
| Can men browse? | No |
| Dwell-time auto-like | Removed |
| Chat duration | 48 hours per message, no extend |
| Unsend | No, for either party |
| Profiles | Free-form only, no curated tags or prompts |
| Photos | Up to 6, no mandatory selfie |
| Chat features | Text only, all unlocked from first message, no gating |
| Invisible mode | Removed |
| Vouches | Removed |
| Unmatch | Either party can unmatch |
| Unmatch UX | Initiator: chat gone immediately. Other party: sees "User no longer exists", can delete conversation |
| Profile access post-unmatch | Gone — same message whether unmatched or account deleted |
| Reporting | Immediate shadow ban on report, unban if not upheld, permanent suspend if upheld, false reporters also suspended |
| Platform | Mobile only, no web app |
| Tech stack | Flutter + Java Spring Boot Modulith + PostgreSQL + Firebase (Auth, FCM, Storage — all-Firebase) + Railway |
| Architecture | Modulith — single deployment, strict module boundaries |
| Discovery algorithm | To be decided at build time |

---

## Platform & Tech Stack

- **Platform:** Mobile only (iOS + Android), no web app
- **Mobile:** Flutter
- **Backend:** Java 21+ with Spring Boot, Modulith architecture
- **Database:** PostgreSQL
- **Real-time chat:** Spring WebSocket
- **Push notifications:** Firebase Cloud Messaging (free tier)
- **File storage:** Firebase Storage (free tier, 5GB)
- **Auth:** Firebase Auth with Google OAuth — sign-in via Google account, fetch name/DOB/email
- **Hosting:** Railway or Render (free/low-cost tier to start)

---

## Out of Scope for V1

- Multiple cities (Bangalore only)
- Web app
- Media in chat (photos, voice notes, video)
- In-app date planning or venue suggestions
- Events or real-world meetups
- Vouches
- Any AI-powered features
- Revenue beyond basic premium subscriptions
