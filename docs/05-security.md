# 5. Security & Privacy

The data here is sensitive even in a small MVP: full name, contact info,
resume, work authorization status, and (soon) a history of where someone's
applying while possibly still employed elsewhere. The controls below are the
ones that matter at this size — not a smaller version of an enterprise
checklist, just the handful of things that actually prevent the realistic
ways this could go wrong.

## 5.1 The two realistic risks

1. **A query that forgets to filter by user.** Two collections hold user data
   (`Application`, `FieldAnswer`), one rule (always filter by `userId`),
   enforced by habit and code review. See
   [backend §3.2](03-backend.md#32-keeping-data-scoped-to-the-right-user).
2. **The extension's token leaking to the page it's running on.** The
   extension holds a login token and, now that it can be enabled on any job
   site rather than a couple of named ones, runs on a much wider range of
   third-party pages. See §5.3.

Everything else in this doc is in support of those two.

## 5.2 Authentication

- Passwords hashed with bcrypt. Never stored or logged in plaintext.
- JWT with a short-ish expiry (e.g. 7 days). No refresh-token rotation or
  reuse detection yet — the token just expires and the user logs in again.
  That's a real trade-off (a stolen token works until it expires, with no way
  to detect a stolen one being replayed), acceptable for an MVP with no paid
  users yet. Add rotation before this handles anything users would consider
  high-stakes.
- HTTPS everywhere, no exceptions, including local dev pointed at a real API.

## 5.3 Extension-specific rules

- **The token lives only in the background service worker's storage
  (`chrome.storage.local`), never in a content script.** Content scripts run
  on the job site's page — if that page has an XSS bug, anything the content
  script can read, the attacker can read. Keeping the token out of the
  content script's reach means a compromised job-site page still can't steal
  a login.
- **Every message from a content script to the background worker is
  validated**: check `sender.id === chrome.runtime.id`, and don't trust the
  shape of the payload — parse it before using it.
- **Site access is requested per-domain, at runtime, not `<all_urls>` at
  install.** The manifest only requires the API's own domain up front; every
  job-site domain goes through `optional_host_permissions` and a "Enable
  ApplyFlow on this site?" prompt the first time the extension sees something
  application-shaped there — see
  [extension §Permissions](02-extension.md#permissions). This is what makes
  supporting "any site" compatible with least privilege: the extension only
  ever holds permission for domains the user has actually said yes to, one at
  a time, rather than every domain on the web by default. It's also what
  keeps Chrome Web Store review from slowing down despite the broader reach.
- **No `tabs` permission.** `activeTab` is enough for what this extension
  does, and `tabs` would show up as a scarier ask in the install prompt for no
  benefit.
- **Strict CSP, no remote code, no `eval`.** Required by the store, and
  correct anyway given the extension runs on pages we don't control.

## 5.4 Resume storage

- Resume files go to a private (non-public) S3-compatible bucket. MongoDB
  stores the storage key, never the file.
- Downloaded only through short-lived signed URLs, fetched by the backend and
  handed to whoever needs them (the dashboard, or the extension's background
  worker for the file-upload autofill step) — never a permanent public link.
- `Content-Disposition: attachment` on downloads, so an uploaded file can't
  execute in the browser if someone uploads something malicious disguised as
  a resume.

Full field-level encryption of resume/PII columns, a dedicated audit log of
access, and antivirus scanning on upload are reasonable things to add — see
[future improvements](06-future-improvements.md) — but none of them are the
thing standing between this product and a realistic breach at MVP scale. A
private bucket with signed URLs is.

## 5.5 The submit boundary

**The extension fills the form; a human clicks submit.** The extension never
programmatically submits an application, and this isn't a setting you can
turn off. Two reasons:

1. **It's simpler to build.** Detecting "the user submitted" is easier and
   safer than reliably driving a form to submission across sites that change
   their DOM — and it sidesteps any question about whether automatically
   submitting applications is consistent with a given job site's terms of
   service.
2. **A human reviewing the AI-drafted answer before it goes out is the actual
   safety mechanism for that feature** — see
   [AI usage §4.3](04-ai-architecture.md#43-everything-the-llm-writes-is-a-draft-not-a-submission).

## 5.6 Application security baseline

The unremarkable stuff, still worth listing so it doesn't get skipped:

- Input validation on every API route (a small Zod schema per route is
  enough — no need for a shared contracts package across three apps when
  there are two, and one of them is the other's frontend).
- Rate limiting on `/api/auth/*` and `/api/ai/*` (see
  [backend §3.6](03-backend.md#36-rate-limiting-the-ai-endpoint)).
- Dependency updates via Dependabot or equivalent, so this isn't a manual
  chore that gets skipped.
- No user-generated content rendered as raw HTML anywhere in the dashboard.

## 5.7 What's not here yet, on purpose

- **GDPR-style export/erasure flows.** Right now, deleting an account means
  an engineer runs a delete query. Worth automating before there are real
  users in jurisdictions that require it — see
  [future improvements](06-future-improvements.md).
- **A database-level backstop** behind the `userId` filtering habit — Mongo
  has no direct equivalent to Postgres row-level security, so the eventual
  hardening step here looks like a thin, mandatory data-access layer that
  every query goes through, rather than a policy enforced by the database
  itself. Worth adding once more than one engineer is touching this code and
  a missed filter becomes a realistic mistake rather than something one
  person would catch reviewing their own diff.
- **An audit log of security-relevant actions** (logins, profile changes).
  Add it when there's someone whose job is to look at it.
- **Google OAuth login and its consent-screen review.** Not needed for
  email/password login; comes with its own setup cost, so it's deferred
  until there's a reason users specifically need it.
