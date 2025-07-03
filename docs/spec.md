## **Software Design Specification: Live Polling Application**

**Version:** 1.1
**Date:** October 27, 2023

### **1. Project Overview**

This document specifies the design for a web-based, real-time polling application. The application allows a "Poll Owner" to create and present polls to an in-person audience, and "Poll Takers" to vote from their personal devices anonymously. The system must be responsive, real-time, well-tested, and deployable as a self-contained unit.

#### **1.1. User Roles & Core Functionality**

*   **Poll Owner (Authenticated):**
    *   Logs in via an email-based magic link.
    *   Manages a dashboard of all their created polls.
    *   Creates, edits, and deletes polls, where each poll consists of one or more multiple-choice questions.
    *   Presents a live poll on a main screen, which displays the current question, a shareable link/QR code for voters, and a real-time chart of incoming results.
    *   Controls the live poll flow: starting the poll, advancing to the next question, and closing the poll.

*   **Poll Taker (Anonymous):**
    *   Joins a poll via a unique link or QR code.
    *   Sees the currently active question and its options.
    *   Casts a single vote per question and can change their vote while the poll is open.
    *   Is prevented from voting once the poll is closed.

### **2. Technical Architecture**

*   **Framework:** Next.js with the **Pages Router**.
*   **Language:** **TypeScript** for the entire project.
*   **Database:** **SQLite**.
*   **ORM:** **Prisma**.
*   **Real-time Communication:** **Socket.IO**.
*   **Authentication:** **NextAuth.js (Auth.js)** with the Email (Magic Link) Provider.
*   **Styling:** **Tailwind CSS**. Component libraries like `shadcn/ui` are recommended.
*   **Testing:** **Jest** & **React Testing Library** for unit tests; **Playwright** for E2E tests.
*   **Deployment Environment:** **Fly.io**.
*   **Persistence:** A single **Fly.io Persistent Volume**.
*   **Server Runtime:** A custom `server.ts` file to co-locate the Next.js app and the Socket.IO server.

### **3. Data Model (Prisma Schema)**

This is the definitive source of truth for the database structure. Implement this schema exactly as written in `prisma/schema.prisma`.

```prisma
// file: prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

// Models for NextAuth.js - Do not modify
model Account {
  id                String  @id @default(cuid())
  userId            String
  type              String
  provider          String
  providerAccountId String
  refresh_token     String? @db.Text
  access_token      String? @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String? @db.Text
  session_state     String?
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  @@unique([provider, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique
  userId       String
  expires      DateTime
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)
}

model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime
  @@unique([identifier, token])
}

// ---- Application-Specific Models ----
model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  accounts      Account[]
  sessions      Session[]
  polls         Poll[]
}

model Poll {
  id               String     @id @default(cuid())
  title            String
  shortCode        String     @unique @default(cuid())
  status           String     @default("DRAFT") // "DRAFT", "OPEN", "CLOSED"
  activeQuestionId String?
  ownerId          String
  owner            User       @relation(fields: [ownerId], references: [id], onDelete: Cascade)
  questions        Question[]
  createdAt        DateTime   @default(now())
  updatedAt        DateTime   @updatedAt
}

model Question {
  id      String   @id @default(cuid())
  text    String
  order   Int
  pollId  String
  poll    Poll     @relation(fields: [pollId], references: [id], onDelete: Cascade)
  options Option[]
  votes   Vote[]
}

model Option {
  id         String   @id @default(cuid())
  text       String
  questionId String
  question   Question @relation(fields: [questionId], references: [id], onDelete: Cascade)
  votes      Vote[]
}

model Vote {
  id String @id @default(cuid())
  voterIdentifier String
  questionId      String
  question        Question @relation(fields: [questionId], references: [id], onDelete: Cascade)
  optionId        String
  option          Option   @relation(fields: [optionId], references: [id], onDelete: Cascade)
  @@unique([voterIdentifier, questionId])
}
```

### **4. API and Real-Time Specification**

#### **4.1. REST API Endpoints (Next.js API Routes)**

All server-side logic must validate that the authenticated user is the owner of the resource they are trying to access.

| Method | Path | Protection | Description |
| :--- | :--- | :--- | :--- |
| `GET` | `/api/auth/[...nextauth]` | N/A | Handled by NextAuth.js for magic link login. |
| `GET` | `/api/polls` | Authenticated | Fetches all polls owned by the current user for the dashboard. |
| `POST` | `/api/polls` | Authenticated | Creates a new poll with a title, questions, and options. |
| `GET` | `/api/polls/[id]` | Authenticated | Fetches a single poll's complete data for the edit view. |
| `PUT` | `/api/polls/[id]` | Authenticated | Updates an existing poll's data. |
| `DELETE` | `/api/polls/[id]`| Authenticated | Deletes a poll and its related data. |
| `GET` | `/api/public/polls/[shortCode]` | Public | Fetches the public data for a poll for a voter. |

#### **4.2. WebSocket (Socket.IO) Events**

*   **Socket Authentication:** Implement a server-side middleware for Socket.IO. On connection, it inspects `socket.handshake.auth.token`. If the token is valid, decode it and attach the user object to the socket (`socket.data.user = user`).
*   **Rooms:** All broadcasted events must be scoped to a room named `poll_room_${pollId}`.

| Event Name | Direction | Payload | Auth | Description |
| :--- | :--- | :--- | :--- | :--- |
| `client:join_poll` | C -> S | `{ pollId: string }` | No | Joins the socket to the specified poll's room. |
| `presenter:change_question` | C -> S | `{ pollId: string, nextQuestionId: string \| null }` | Yes | Changes the poll's `activeQuestionId`. Must verify sender is owner. |
| `presenter:change_status` | C -> S | `{ pollId: string, status: "OPEN" \| "CLOSED" }` | Yes | Changes the poll's `status`. Must verify sender is owner. |
| `client:submit_vote` | C -> S | `{ pollId: string, questionId: string, optionId: string, voterIdentifier: string }` | No | Creates/updates a vote using Prisma `upsert` on `[voterIdentifier, questionId]`. |
| `server:question_changed` | S -> C | `{ activeQuestion: Question & { options: Option[] } }` | N/A | Broadcast to room. Informs clients of the new active question. |
| `server:poll_status_changed` | S -> C | `{ status: "OPEN" \| "CLOSED" }` | N/A | Broadcast to room. Informs clients to enable/disable voting. |
| `server:results_update` | S -> C | `{ questionId: string, results: { optionId: string, count: number }[] }` | N/A | Broadcast to room after a vote. Provides new vote counts for real-time chart updates. |

### **5. Page & Component Breakdown**

*   **Pages (`/pages`)**: `/index.tsx`, `/dashboard.tsx`, `/poll/create.tsx`, `/poll/[id]/edit.tsx`, `/present/[pollId].tsx`, `/v/[shortCode].tsx`, `/_app.tsx`, `/_document.tsx`.
*   **Components (`/components`)**: `Layout.tsx`, `PollForm.tsx`, `LiveResultsChart.tsx`, `QRCodeDisplay.tsx`, `VoterView.tsx`, `PresenterControls.tsx`.
*   **Contexts (`/contexts`)**: `SocketContext.tsx` to provide the Socket.IO client instance via a `useSocket` hook.

### **6. Deployment & Operations**

#### **6.1. Dockerfile**

Implement the provided multi-stage `Dockerfile` to create a production-ready container image. It must correctly copy source files, build the Next.js app, and prepare the custom server for execution.

#### **6.2. Fly.io Configuration (`fly.toml`)**

The `fly.toml` file must be configured as follows to ensure persistence and an always-on instance.

```toml
app = "live-polling-app-placeholder" # To be replaced by the end user
primary_region = "iad" 

[http_service]
  internal_port = 3000
  force_https = true
  auto_stop_machines = false
  auto_start_machines = true
  min_machines_running = 1

[mounts]
  source = "poll_data"
  destination = "/data"

[deploy]
  release_command = "npm run db:deploy"
```

#### **6.3. Environment Variables & Secrets**

The following secrets must be set in the Fly.io environment:
*   `DATABASE_URL`: `file:/data/livepoll.db`
*   `NEXTAUTH_URL`: `https://<final-app-name>.fly.dev`
*   `NEXTAUTH_SECRET`: A securely generated random string.
*   `EMAIL_SERVER`: SMTP server connection string.
*   `EMAIL_FROM`: The "From" address for magic link emails.

#### **6.4. `package.json` Scripts**

The following scripts must be present:
```json
"scripts": {
  "dev": "ts-node-dev --respawn --transpile-only server.ts",
  "build": "next build && tsc --project tsconfig.server.json",
  "start": "NODE_ENV=production node dist/server.js",
  "test": "jest",
  "test:e2e": "playwright test",
  "db:deploy": "prisma migrate deploy"
}
```
*(Note: Build script must compile `server.ts` to JavaScript for production.)*

### **7. Testing Strategy**

#### **7.1. Unit & Integration Testing**

*   **Tools:** Jest, React Testing Library, Prisma Mocking, Socket.IO Mocking.
*   **Scope:**
    *   **Frontend Components:** Test form validation, state changes from mocked socket events, and correct event emissions on user interaction.
    *   **Backend Logic:** Test API route authorization (owner-only access), data validation (bad requests), and business logic using a mocked Prisma client.

#### **7.2. End-to-End (E2E) Testing**

*   **Tool:** Playwright.
*   **Scope:** Automate full user journeys in a browser. Tests will run against a production-like build.
*   **Critical Scenarios to Implement:**
    1.  **Poll Owner Lifecycle:** Log in, create a poll, see it on the dashboard, delete it, and assert it's gone.
    2.  **Anonymous Voter Journey:** Visit a poll, vote, change the vote, and verify that controls become disabled when the poll is closed by the backend.
    3.  **Real-time Presenter/Voter Interaction:** Use two concurrent browser contexts. Have the presenter start the poll and advance questions. Assert that the voter's screen updates automatically in real-time without a page refresh, and that votes cast by the voter appear on the presenter's chart instantly.

#### **7.3. CI/CD Integration**

*   A CI/CD pipeline (e.g., GitHub Actions) will be configured.
*   Unit tests (`npm test`) and E2E tests (`npm run test:e2e`) will run on every pull request.
*   A failing test in either suite will block the pull request from being merged.
