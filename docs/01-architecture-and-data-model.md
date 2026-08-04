# 1. Architecture & Data Model

## 1.1 System design

Two things talk to each other: the Chrome extension, and one Next.js app that
serves both the dashboard pages and the API routes the extension calls.

```text
┌───────────────────────┐          ┌─────────────────────────────┐
│  Chrome Extension      │  HTTPS   │        Next.js App           │
│  content script        │ ───────► │  Dashboard pages (React)     │
│  background worker     │  JWT     │  API routes  (/api/*)        │
│  (holds the token)     │          │  Prisma Client               │
└───────────────────────┘          └───────────────┬──────────────┘
                                                     ▼
                                        ┌─────────────────────────┐
                                        │  MongoDB                 │
                                        │  S3-compatible storage   │
                                        │  (resume files)          │
                                        └───────────┬──────────────┘
                                                     ▼
                                        ┌─────────────────────────┐
                                        │  LLM provider (one)      │
                                        │  called directly from    │
                                        │  the API route            │
                                        └─────────────────────────┘
```

That's the whole system. No queue, no worker process, no cache layer, no
separate API service. If you've built a typical Next.js + MongoDB app before,
you already know this shape.

## 1.2 Why one app instead of three

The earlier draft of this design had a separate NestJS API, a separate Next.js
dashboard, Redis, BullMQ, and a pool of worker processes. That's a reasonable
shape for a team with dozens of engineers and millions of users. For one
engineer building the first version, it's mostly overhead:

- A separate API service means two things to deploy, two sets of environment
  variables, and CORS to configure between them. A single Next.js app serves
  pages and API routes from the same origin.
- A job queue exists to keep slow work off the request thread. The only slow
  work here is one AI call per custom question, which takes a few seconds —
  slow enough to show a spinner, not slow enough to need a queue and a poll
  endpoint. See [AI usage](04-ai-architecture.md) for why this is fine.
- Everything that queues, workers, and events were protecting against
  (a degraded LLM provider blocking other traffic, events getting lost on
  crash) matters at a scale this product isn't at yet. It's cheaper to add a
  queue later, when there's a real job worth queuing, than to build and
  maintain one now for jobs that don't exist yet.

## 1.3 Why MongoDB

The app has three collections and every write to any of them touches exactly
one document — create a user, create an application, upsert a remembered
field answer. Nothing here needs a multi-row transaction. That, plus one
column whose shape genuinely varies (see `FieldAnswer.value` below), is what
tips the choice toward MongoDB over a relational database for this design:

- **`FieldAnswer.value` is the actual feature being asked for.** A remembered
  answer might be a string, a yes/no, or a list of selected checkboxes. A
  document database stores that as-is. A relational table would need either a
  fixed set of nullable columns per possible shape, or a serialized blob
  column that's schemaless anyway — Mongo just gives you that directly.
- **No feature here needs a cross-collection transaction.** Reminders,
  billing, and the other things that would want one are explicitly out of
  scope (see [future improvements](06-future-improvements.md)). Every write
  in the MVP is naturally atomic as a single-document operation, which is
  what Mongo is good at without reaching for its (heavier) multi-document
  transaction API.
- **One database to run, not two.** A hybrid (Postgres for `User`/
  `Application`, Mongo for `FieldAnswer`) would mean two connections, two
  backup procedures, and joining across them in application code instead of
  in one query. Not worth it when the whole schema is three collections.
- **Prisma, not a separate ODM.** The design still uses Prisma as the client —
  its `mongodb` provider gives typed models and `include` for relational-style
  lookups (e.g. `application.findMany({ include: { user: true } })`), which is
  the same thing `populate` gives you in Mongoose, without adding a second
  library to learn.

**One real trade-off worth knowing up front:** Mongo has no
`prisma migrate` history. Schema changes sync with `prisma db push` instead —
you lose a versioned migration log and easy rollback, in exchange for not
having to write a migration file for every small schema tweak. For a schema
that's expected to keep changing while the product is being figured out,
that's a fair trade; revisit if the schema stabilizes and rollback safety
starts to matter more than edit speed.

## 1.4 Components

**Chrome Extension.** Detects an application form on the current page, pulls
job details off it, fills in known and remembered fields, asks the API for
AI-drafted answers to custom questions, and reports back when the application
was submitted. Holds no business logic beyond reading and writing the page's
DOM — everything it needs comes from the API, and everything it learns goes to
the API. Details in [extension](02-extension.md).

**Next.js App.** Dashboard pages (login, profile, application list) plus API
routes under `/api/*` that both the dashboard and the extension call. One
Prisma client, one MongoDB connection. Details in [backend](03-backend.md).

**MongoDB.** The only datastore. Holds users, applications, and remembered
field answers. No vector store, no separate cache — there's nothing here big
enough to need one yet.

**S3-compatible storage.** Resume files only. MongoDB stores the storage key,
never the file itself. Accessed through short-lived signed URLs, never a
public bucket.

## 1.5 How the pieces talk to each other

| Path | Mechanism | Notes |
|---|---|---|
| Extension → API | HTTPS, JWT bearer token | Token lives in the extension's background worker only. See [security](05-security.md). |
| Dashboard → API | Next.js API routes, same origin | Regular cookie-based session; no cross-origin concerns because it's the same app. |
| API → DB | Prisma Client (`mongodb` provider) | Every query for user data includes `where: { userId }`. No repository framework — just a habit enforced in code review. |
| API → LLM | Direct HTTPS call from inside the API route | Request comes in, API calls the LLM, response goes out. No intermediate job record needed for a single-shot call. |

---

## 2. Database design

### Principles

- **Every table with user data has a `userId` column directly on it.** Not
  reachable through a join — directly on the row. That's the one rule worth
  keeping strict, because getting it wrong means one user can see another
  user's data.
- **Three collections for the MVP: `User`, `Application`, and
  `FieldAnswer`.** Add a collection when a feature needs it, not before. The
  earlier draft had thirteen tables at MVP scope (OAuth accounts, candidate
  profile, notes, reminders, documents, versioned resumes, AI generation
  logs, email integration, email messages...) — almost all of it in support
  of features that aren't in this version. See
  [future improvements](06-future-improvements.md) for what gets added back
  and when.

### Schema

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "mongodb"
  url      = env("DATABASE_URL")
}

model User {
  id           String   @id @default(auto()) @map("_id") @db.ObjectId
  email        String   @unique
  passwordHash String
  name         String?
  phone        String?
  linkedinUrl  String?
  location     String?

  // Facts the AI must never guess — answered directly from these fields
  // instead of generated. See AI usage doc.
  workAuthorized      Boolean?
  needsSponsorship    Boolean?

  resumeStorageKey String?   // S3 key of the current resume file
  resumeFileName   String?

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  applications  Application[]
  fieldAnswers  FieldAnswer[]
}

model Application {
  id             String            @id @default(auto()) @map("_id") @db.ObjectId
  userId         String            @db.ObjectId
  companyName    String
  jobTitle       String
  jobUrl         String?
  jobDescription String?
  location       String?
  atsPlatform    AtsPlatform       @default(OTHER)
  status         ApplicationStatus @default(APPLIED)
  source         ApplicationSource @default(EXTENSION)
  appliedAt      DateTime          @default(now())
  createdAt      DateTime          @default(now())
  updatedAt      DateTime          @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, jobUrl])   // re-submitting the same job updates, doesn't duplicate
  @@index([userId, appliedAt])
}

enum AtsPlatform { GREENHOUSE  LEVER  OTHER }

enum ApplicationStatus { APPLIED  INTERVIEWING  OFFER  REJECTED }

enum ApplicationSource { MANUAL  EXTENSION }

// A question the user has answered by hand at least once, remembered so the
// extension can fill it in automatically next time it sees the same
// question — on any site, not just the one it was learned on.
model FieldAnswer {
  id         String   @id @default(auto()) @map("_id") @db.ObjectId
  userId     String   @db.ObjectId
  signature  String   // normalized label, e.g. "are_you_authorized_to_work_in_the_us"
  rawLabel   String   // the exact label as seen on the page, for display/debugging
  value      Json     // string, string[], boolean — shape varies by field type
  updatedAt  DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([userId, signature])
}
```

That's the whole schema. No soft deletes (hard-delete is fine at this size —
add `deletedAt` back if you ship an undo feature), no append-only audit trail
of status changes (the dashboard just lets the user change `status` directly),
no AI generation log (drafted question answers aren't persisted — see
[AI usage](04-ai-architecture.md); `FieldAnswer` is for values the user typed
themselves, not for AI output).

### Why the `@@unique` constraints

Both compound-unique indexes exist for the same reason: to turn a
"has this already happened?" check into a database constraint instead of
application logic.

- **`Application [userId, jobUrl]`** — if the extension reports the same
  submission twice (a retry, a duplicate click), the second write updates the
  existing row instead of creating a second application. No separate
  idempotency-key table required.
- **`FieldAnswer [userId, signature]`** — typing a new answer to a question
  already in the bank updates the remembered value instead of creating a
  duplicate entry. This is what makes saving an answer a plain upsert.

### What's deliberately not here yet

- **Resume versions / tailoring.** One resume file per user for now. A
  `Resume` table with versions and per-job tailoring is a real feature, just
  not this one — see [future improvements](06-future-improvements.md).
- **Notes, reminders, stage history.** The dashboard shows a list with a
  status dropdown. No sub-tables until there's a feature that needs them.
- **Email integration tables.** No inbox connection in the MVP at all.
- **Any AI output table.** A generated answer is shown to the user once and
  never saved. If caching or reuse becomes worth it, that's the trigger to add
  a table for it.
- **Fuzzy matching or cross-user sharing on `FieldAnswer`.** Matching is exact
  on the normalized signature, and answers are private to the user who typed
  them. See [future improvements](06-future-improvements.md).
