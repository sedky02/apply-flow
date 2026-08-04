# ApplyFlow — Technical Design (MVP)

A Chrome extension that autofills job applications on any company's career
site and saves them to a simple dashboard, with AI help only for the
free-text questions ("Why do you want to work here?") that can't be filled
from the user's profile or their own past answers.

This design is scoped for **one full-stack engineer to build in a few weeks**.
Anything that isn't needed to prove the core loop — "install the extension,
apply to a job, see it saved" — has been moved to
[Future Improvements](06-future-improvements.md). Read that doc too: it
explains what was cut and why, so nothing gets rebuilt by accident before it's
actually needed.

## What the MVP does

1. Detects that the current tab looks like a job application page — on any
   site, not a fixed list of platforms — and, the first time on a new site,
   asks the user to enable it there.
2. Pulls the job title, company, and description off the page.
3. Fills in the fields it recognizes from the user's saved profile (name,
   email, phone, resume, LinkedIn, etc.), and fields it's seen before from the
   user's own past answers (work authorization, notice period, and other
   questions the user typed once and the extension remembered).
4. For free-text questions it doesn't have an answer for either way, asks the
   AI to draft one — the user reviews and edits it before submitting.
5. Notices when the application was actually submitted, and remembers any
   answer the user typed by hand during review for next time.
6. Saves the applied job to the backend so it shows up in a simple list on the
   dashboard.

That's it. No Kanban board, no email inbox scanning, no resume tailoring, no
analytics. Those are all real ideas — they're just not what makes the first
version work, so they're written up as follow-ups instead of built now.

## Design decisions

| Decision | Choice | Why |
|---|---|---|
| Apps | **Two**, not three: one Next.js app (dashboard + API routes) and one Chrome extension | A separate backend service is a second thing to deploy, version, and keep in sync with the frontend for no MVP benefit. One engineer should run one server. |
| Data store | **MongoDB**, via Prisma's `mongodb` provider | Three small collections, every write touches exactly one document, and one column (`FieldAnswer.value`) genuinely needs a flexible shape. That's a document database's sweet spot — see [architecture §1.3](01-architecture-and-data-model.md#13-why-mongodb). |
| Background work | None. AI calls and DB writes happen inline in the API route that receives the request | No job queue, no worker process, no Redis. A single custom-question generation takes a few seconds; the extension can just wait for it. |
| AI | Used for exactly one thing: drafting an answer to a free-text application question | Everything else that might look like an "AI feature" (field mapping, email classification, resume tailoring) is done with plain code or postponed — see [AI usage](04-ai-architecture.md). |
| Field matching | A dictionary of label → field for profile data, plus an exact-match memory of the user's own past answers. No confidence scores, no fuzzy matching, no cross-user learning | Covers the common profile fields with a lookup table, and covers repeat custom questions by remembering what the user typed the first time — on any site, not just the one it was learned on. See [extension](02-extension.md#matching-fields). |
| Site coverage | **Any site**, via a generic form detector and a runtime "enable this site?" permission prompt — not a fixed adapter per platform | The value of autofill is filling forms wherever they are, not just on the handful of platforms worth writing an adapter for. Greenhouse and Lever keep a small amount of site-specific code, but only for parsing job details more reliably — see [extension](02-extension.md#extracting-job-info). |
| Submission | **Extension fills, human clicks submit** | We never auto-submit an application. Simpler to build, and it avoids any question about whether an automated apply-bot violates a job site's terms. See [security](05-security.md). |
| Data model | `User`, `Application`, and `FieldAnswer` — no separate resume versions, stage history, or notes yet | The MVP only needs to know who the user is, what they applied to, and what they've answered before. Add collections when a feature actually needs them. |

## Documents

| # | Doc | Covers |
|---|---|---|
| 1 | [Architecture & data model](01-architecture-and-data-model.md) | The two apps, how they talk to each other, the database schema |
| 2 | [Chrome extension](02-extension.md) | Folder structure, detecting job pages, extracting job info, filling forms, detecting submission |
| 3 | [Backend](03-backend.md) | API routes, auth, database access |
| 4 | [AI usage](04-ai-architecture.md) | The one place AI is used, and how it's kept simple and safe |
| 5 | [Security](05-security.md) | Auth, extension permissions, storing resumes safely |
| 6 | [Future improvements](06-future-improvements.md) | Everything cut from the MVP, and what would trigger building it |

## The two things most likely to slow this down

1. **The generic detector won't handle every form shape, and job sites change
   their HTML.** A site with an unusually structured form may confuse the
   generic detector or field matcher; Greenhouse and Lever specifically could
   also break their `JSON-LD` job-detail parsing with a markup change. Either
   way, the fix ships in the next extension update — an acceptable trade-off
   at MVP scale (one engineer) — see
   [extension](02-extension.md#matching-fields) and
   [future improvements](06-future-improvements.md) for when to invest in
   something more resilient.
2. **Chrome Web Store review takes time.** Any fix to the extension has to go
   through review before users get it. Keep permissions narrow (it reviews
   faster) and expect a few days of lag on every release.
