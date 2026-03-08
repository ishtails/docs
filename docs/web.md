# Verity Web Architecture

**Chainlink CRE & AI Hackathon — Supporting Documentation**  
**See:** `docs/cre.md` for contract and workflow details

---

## What This Document Covers

This doc explains the **web layer** — the UI and server that facilitate the CRE-based knowledge exchange. For the core innovation (CRE workflows, AI evaluation, smart contracts), see `docs/cre.md`.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                   USER LAYER                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────────────┐  │
│  │   Teacher    │  │   Learner    │  │           Verity Web UI              │  │
│  │  (Host)      │  │  (Attendee)  │  │      (React + TanStack Router)       │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              WEB PACKAGE (Bun/Hono)                              │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                           API Server                                        │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                          │ │
│  │  │ /sessions   │ │  /meetings  │ │   /users    │                          │ │
│  │  │ CRUD        │ │  Calendar   │ │  Profile    │                          │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘                          │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                         Database (Drizzle ORM)                              │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                          │ │
│  │  │   users     │ │  sessions   │ │  meetings   │                          │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘                          │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
         ┌────────────────────────────────┼────────────────────────────────┐
         ▼                                ▼                                ▼
┌────────────────────┐      ┌────────────────────────┐      ┌────────────────────┐
│  Google Calendar   │      │      Recall.ai        │      │      Pinata        │
│  (Meeting URLs)    │      │  (Recording/Transcript)│      │   (IPFS Storage)   │
└────────────────────┘      └────────────────────────┘      └────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        BLOCKCHAIN & CRE ORACLE LAYER                             │
│  ┌────────────────────────────┐        ┌──────────────────────────────────────┐│
│  │      EVM (Sepolia)         │        │     Chainlink CRE Network            ││
│  │  ┌──────────────────────┐   │        │  ┌────────────────────────────────┐   ││
│  │  │    KXManager         │   │        │  │   Initiation Workflow          │   ││
│  │  │   - Listings         │   │◄───────┼──┤   (See docs/cre.md)            │   ││
│  │  │   - Session Requests │   │        │  └────────────────────────────────┘   ││
│  │  └──────────────────────┘   │        │  ┌────────────────────────────────┐   ││
│  │         │                    │        │  │   Settlement Workflow          │   ││
│  │         ▼                    │        │  │   (See docs/cre.md)            │   ││
│  │  ┌──────────────────────┐   │        │  └────────────────────────────────┘   ││
│  │  │ KXSessionRegistry    │   │        │                                      ││
│  │  │   - Escrow           │   │        │                                      ││
│  │  │   - AI Evaluation    │   │        │                                      ││
│  │  └──────────────────────┘   │        │                                      ││
│  └────────────────────────────┘        │                                      ││
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Key Point:** The web layer handles UI and meeting logistics. All AI evaluation and settlement logic lives in CRE workflows (see `docs/cre.md`).

---

## Frontend Layer

**Location:** `packages/web/src/app/`

**Stack:** React 19, TanStack Router, TanStack Query, shadcn/ui, Privy (auth/wallet), wagmi

**Key Screens:**


| Route           | Purpose                      |
| --------------- | ---------------------------- |
| `/`             | Landing page                 |
| `/onboarding`   | New user setup               |
| `/dashboard`    | Host dashboard with listings |
| `/listings`     | Browse available sessions    |
| `/listings/$id` | Session detail + booking     |


**Auth Flow:**

1. User clicks "Get Started"
2. Privy modal handles login
3. Embedded wallet created
4. New users → `/onboarding`, Existing → `/dashboard`

---

## Server Layer (Hono)

**Location:** `packages/web/api/`

**Runtime:** Bun  
**Framework:** Hono  
**Database:** PostgreSQL + Drizzle ORM

### Core API Routes


| Route                | Purpose              | Key Function                      |
| -------------------- | -------------------- | --------------------------------- |
| `/sessions`          | Session/listing CRUD | Teacher creates listings          |
| `/sessions/meetings` | Meeting scheduling   | Server generates Google Meet URLs |
| `/users`             | User profiles        | Auth + profile management         |


**Auth:** `requireAuth` middleware validates Privy tokens

---

## Server Responsibilities

### 1. Meeting URL Generation

**Handler:** `packages/web/api/handlers/meetings.ts`

When a learner books a session:

```typescript
// Server calls Google Calendar API
const event = await createGoogleCalendarEvent(
  input.startDate,
  input.duration,
  input.summary,
  input.attendees
);
const meetingUrl = event.hangoutLink; // Google Meet URL

// Store and return
return { meetingUrl, sessionPrice };
```

**Why Server-Generated?**

- Security: Prevents URL manipulation
- Consistency: All meetings via Google Calendar
- Tracking: Server monitors meeting lifecycle

### 2. On-Chain Data Caching

**Purpose:** Fast dashboard loads without blockchain RPC calls

**Cached:**

- Session listings
- Meeting history
- User participation

**Sync:** Server queries chain directly; frontend invalidates after contract interactions

---

## Database Schema

**Location:** `packages/web/api/lib/db/schema/`


| Table      | Purpose             | Key Columns                                 |
| ---------- | ------------------- | ------------------------------------------- |
| `users`    | Profiles            | `id`, `email`, `walletAddress`              |
| `sessions` | Listings            | `id`, `hostId`, `topic`, `price`, `dataCID` |
| `meetings` | Scheduled meets     | `id`, `sessionId`, `meetingUrl`, `eventId`  |
| `goals`    | Learning objectives | `id`, `sessionId`, `name`, `weight`         |


---

## Complete Session Flow

```
┌─────────────┐     1. Create Listing      ┌─────────────┐
│   Teacher   │────────────────────────────►│  Web UI     │
│   (Host)    │                             │  /dashboard │
└─────────────┘                             └──────┬──────┘
                                                   │
                                                   │ 2. POST /sessions
                                                   ▼
                                           ┌─────────────┐
                                           │   Server    │
                                           │  (Hono API) │
                                           └──────┬──────┘
                                                  │
                                                  │ 3. Store metadata
                                                  ▼
                                           ┌─────────────┐
                                           │  Database   │
                                           └─────────────┘

┌─────────────┐     4. Book Session        ┌─────────────┐
│   Learner   │────────────────────────────►│  Web UI     │
│  (Attendee) │                             │  /listings  │
└─────────────┘                             └──────┬──────┘
                                                   │
                                                   │ 5. POST /meetings
                                                   ▼
                                           ┌─────────────┐
                                           │   Server    │
                                           │  (Calendar) │
                                           └──────┬──────┘
                                                  │
                                                  │ 6. Google Calendar API
                                                  │    Creates Meet URL
                                                  ▼
                                           ┌─────────────┐
                                           │  Meeting    │
                                           │   URL       │
                                           └──────┬──────┘
                                                  │
                                                  │ 7. Return { meetingUrl }
                                                  ▼
┌─────────────┐     8. Join Meeting        ┌─────────────┐
│   Teacher   │◄────────────────────────────│  Learner    │
│   Learner   │    (Google Meet)            │  pays USDC  │
└─────────────┘                             └─────────────┘
       │
       │ 9. Meeting happens
       │    Recall bot joins
       ▼
[From here, CRE workflows take over — see docs/cre.md]
       │
       ▼
┌─────────────┐
│   CRE       │ 10. Initiation Workflow
│  Workflow   │     (creates bot, stores data)
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   CRE       │ 11. Settlement Workflow  
│  Workflow   │     (AI eval via Gemini,
│             │      calculates payout)
└──────┬──────┘
       │
       ▼
[Contracts settle payment — see docs/cre.md]
       │
       ▼
┌─────────────┐     12. Claim funds        ┌─────────────┐
│   Teacher   │◄───────────────────────────│   Learner   │
│   claims    │    (via Web UI)            │   claims    │
└─────────────┘                             └─────────────┘
```

**Steps 10-11:** Happen autonomously via Chainlink CRE — no server involvement.

---

## External Services


| Service             | Used By      | Purpose                   |
| ------------------- | ------------ | ------------------------- |
| **Google Calendar** | Server       | Generate Google Meet URLs |
| **Recall.ai**       | CRE Workflow | Recording, transcription  |
| **Gemini**          | CRE Workflow | AI evaluation             |
| **Pinata**          | CRE Workflow | IPFS storage              |


---

## File Structure

```
packages/web/
├── src/
│   ├── app/                   # TanStack Router pages
│   │   ├── index.tsx          # Landing
│   │   ├── onboarding/          # User setup
│   │   └── _authenticated/    # Protected routes
│   │       ├── dashboard/
│   │       ├── listings/
│   │       └── history/
│   ├── components/ui/         # shadcn components
│   └── lib/
│       ├── hooks/api/          # API hooks (TanStack Query)
│       └── context/            # Auth, EVM, Theme
│
└── api/                       # Hono server
    ├── routes/
    │   ├── sessions.route.ts  # Session CRUD
    │   ├── meetings.route.ts  # Calendar integration
    │   └── users.route.ts     # Auth
    ├── handlers/
    │   ├── sessions.ts
    │   ├── meetings.ts        # Google Calendar API
    │   └── users.ts
    └── lib/
        ├── db/                # Drizzle ORM + schema
        └── utils/
            └── calendar.ts    # Google Calendar client

packages/contracts/            # See docs/cre.md
└── workflows/
    ├── initiation-workflow/   # See docs/cre.md
    └── settlement-workflow/   # See docs/cre.md
```

---

## Deployment

### Sepolia Testnet

- **KXManager:** `0x6ee50eeafa56f269a8875bb09e2a3ab9608bdf4b`
- **KXSessionRegistry:** `0x42c105b36825778ca323bf850df6e007b0407dca`

**Full deployment details in `docs/cre.md`**

---

## Key Design Decisions

### Why Separate Web Layer from CRE Workflows?


| Layer             | Responsibility                                 | Trust Model                 |
| ----------------- | ---------------------------------------------- | --------------------------- |
| **Web (Server)**  | UI support, meeting logistics, caching         | Standard app server         |
| **CRE Workflows** | AI evaluation, settlement, oracle verification | Trustless via Chainlink DON |


The server facilitates the exchange. The workflows guarantee fair settlement.

---

*Last updated: March 8, 2026*