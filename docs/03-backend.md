# 3. Backend Architecture

One Next.js app. Dashboard pages and API routes live in the same codebase and
deploy together. There's no separate API service, no job queue, no cache
layer — just route handlers that read the request, talk to MongoDB through
Prisma, and (for one endpoint) call the LLM.

## 3.1 Folder structure

```text
apps/web/src/
├── app/
│   ├── (dashboard)/
│   │   ├── login/
│   │   ├── applications/          # the list page
│   │   └── profile/               # name, phone, resume upload, work-auth answers
│   └── api/
│       ├── auth/{register,login}/route.ts
│       ├── me/route.ts             # GET/PATCH profile
│       ├── applications/
│       │   ├── route.ts            # GET (list), POST (create)
│       │   └── [id]/route.ts       # PATCH (status), DELETE
│       ├── resume/route.ts         # POST — upload, returns storage key
│       ├── field-answers/route.ts  # GET (list), POST (upsert) — the learned-answer bank
│       └── ai/answer/route.ts      # POST — draft a custom-question answer
├── lib/
│   ├── prisma.ts
│   ├── auth.ts                    # sign/verify JWT, password hashing
│   ├── require-user.ts            # pulls userId off the request, throws 401 if missing
│   └── ai.ts                      # the one LLM call — see AI usage doc
└── prisma/schema.prisma
```

That's the whole backend. No `core/`, no `modules/`, no `queues/` — those
folders exist to organize code across many features and many engineers.
With eight endpoint files and one engineer, a flat `app/api` tree is easier to work
in, not less "architected."

## 3.2 Keeping data scoped to the right user

The realistic way this product leaks data is a query that forgets to filter
by user. The MVP's answer to that is a habit, not a framework:

```ts
// lib/require-user.ts
export async function requireUser(req: Request): Promise<string> {
  const token = getBearerToken(req) ?? getSessionCookie(req)
  const payload = verifyJwt(token)
  if (!payload) throw new ApiError(401, 'unauthenticated')
  return payload.userId
}
```

```ts
// app/api/applications/route.ts
export async function GET(req: Request) {
  const userId = await requireUser(req)
  const applications = await prisma.application.findMany({ where: { userId } })
  return Response.json(applications)
}
```

Every handler that touches user data starts with `requireUser` and every
Prisma call includes `where: { userId }` (or `userId` in the `data` on
create). Grep for `prisma.application.` in review and make sure every result
has it. That's the whole tenant-isolation strategy at this size — a
repository base class, row-level security policies, and a custom AST lint rule
are real hardening steps for when there's a team enforcing this across many
PRs at once (see [future improvements](06-future-improvements.md)), not a
day-one requirement for one engineer who can just read their own code.

## 3.3 API design

Plain REST, no versioning prefix yet (`/api/v1` is a rename away if it's ever
needed — not worth doing before there's a second version to distinguish from).

```text
POST   /api/auth/register           { email, password }
POST   /api/auth/login              { email, password } → JWT

GET    /api/me                      profile fields used for autofill
PATCH  /api/me                      update profile fields

POST   /api/resume                  multipart upload → { storageKey }

GET    /api/applications            list (dashboard)
POST   /api/applications            create — used by the extension on submit, and
                                     by "add manually" on the dashboard
PATCH  /api/applications/:id        update status
DELETE /api/applications/:id

GET    /api/field-answers           the user's whole learned-answer bank, fetched
                                     once per page load by the extension
POST   /api/field-answers           upsert { signature, rawLabel, value }[]

POST   /api/ai/answer               { question, jobDescription } → { answer }
```

Fourteen routes. No cursor pagination (a user's application list, and their
field-answer bank, are each at most a few hundred rows — a plain
`sort + limit` is plenty), no `202 Accepted` + poll pattern for AI (see
below), no `Idempotency-Key` header (the `@@unique` constraints in the schema
do that job for the two endpoints that need it — an upsert on conflict).

`GET /api/field-answers` returns the user's whole bank in one call rather
than exposing a per-signature lookup — the extension fetches it once when a
page loads and matches locally against whatever fields it finds. One request
per page beats one request per field, and the whole bank is small enough
(tens to low hundreds of entries even for a heavy user) that shipping all of
it is cheaper than the round trips a narrower endpoint would save.

## 3.4 Why the AI call is synchronous

`POST /api/ai/answer` calls the LLM and returns the drafted answer in the same
response — no job id, no polling, no webhook. A single-question generation
takes a few seconds. The extension shows a loading state on the button and
gets a result back on the same request. This only stops being fine if a
single call starts taking tens of seconds or the traffic volume makes
inline calls expensive to hold connections open for — neither is a concern at
the traffic an MVP sees. See [AI usage](04-ai-architecture.md) for the actual
call, and [future improvements](06-future-improvements.md) for when to move
this behind a queue.

## 3.5 Authentication

Email + password only for the MVP. No Google OAuth yet — one login path is
less to build and less to debug, and it's a small addition later if it turns
out to matter (see [future improvements](06-future-improvements.md)).

- Passwords hashed with bcrypt.
- `POST /api/auth/login` returns a JWT (short expiry, e.g. 7 days). The
  dashboard stores it in an httpOnly cookie; the extension stores the same
  token in `chrome.storage.local`, readable only by its background worker.
- The extension gets the token by calling `/api/auth/login` directly from its
  own login form (in the popup) — the same endpoint the dashboard uses. There's
  no separate pairing handshake: since there's only one app and one API, on
  the same domain, a normal cross-origin API call from the extension with
  CORS allowed for the extension's origin is all that's needed.
- No refresh-token rotation, no reuse detection. The token just expires and
  the user logs in again. That's a real gap for a product with sensitive
  session semantics at scale; for an MVP, it's an acceptable trade for not
  building a token-family tracking system yet.

## 3.6 Rate limiting the AI endpoint

The one cost worth guarding on day one: nothing stops a bug (or an impatient
user double-clicking) from firing the same generation repeatedly. A single
check covers it —

```ts
const countToday = await prisma.application.count({
  // or a small dedicated counter if you want it independent of applications
  where: { userId, createdAt: { gte: startOfDay() } },
})
if (countToday > DAILY_AI_LIMIT) throw new ApiError(429, 'daily limit reached')
```

— a plain count query against MongoDB, no Redis token bucket. Good enough
until there's enough traffic for a count per request to be worth optimizing.

## 3.7 Observability

- Structured request logs (method, path, status, duration, userId). Scrub the
  resume/job-description text and the `Authorization` header before logging
  anything.
- Whatever error tracking is fastest to wire up (e.g. Sentry's Next.js SDK) —
  the goal is finding out about a 500 before a user emails you, not a full
  tracing pipeline.
- Log the AI call's token counts and rough cost per call from day one, even as
  just a log line. It's the one number worth having a history of once you
  start caring about unit economics — see
  [future improvements](06-future-improvements.md).
