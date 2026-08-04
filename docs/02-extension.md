# 2. Chrome Extension Architecture

Manifest V3 + Plasmo. The extension reads the page, fills in what it can, and
reports back to the API. It doesn't decide anything an ordinary web app
couldn't — no AI calls happen inside the extension itself; it just asks the
API for an answer and shows it.

## Three things that shape every decision here

1. **The background service worker gets killed after ~30 seconds idle.**
   That's just how Manifest V3 works. Don't keep state in memory across
   messages; use `chrome.storage.session` if something needs to survive a
   worker restart, and `chrome.alarms` instead of a long `setTimeout`.
2. **No code runs that didn't ship in the extension bundle.** Prompts and the
   AI call happen on the server; the extension only ever calls our own API.
   This is a Chrome Web Store requirement, and it's also just the right place
   for that logic to live.
3. **The page itself isn't trustworthy.** A job site (or an XSS bug on it)
   should never be able to get at the user's login token. Content scripts run
   in the isolated world and never hold a token; the background worker is the
   only place that makes network calls to our API.

## Folder structure

```text
apps/extension/
├── src/
│   ├── background/
│   │   ├── index.ts              # SW entry: message handlers, alarms
│   │   ├── api-client.ts         # fetch wrapper: attaches the JWT, retries once on 401
│   │   ├── auth.ts               # stores/reads the token in chrome.storage.local
│   │   └── permissions.ts        # requests optional_host_permissions for a new domain
│   ├── contents/
│   │   ├── detector.ts           # generic: does this page look like an application form?
│   │   └── overlay.tsx           # the in-page ApplyFlow panel (fill button, review, status)
│   ├── job-detail-adapters/      # optional, bonus-only — see "Extracting job info"
│   │   ├── types.ts              # JobDetailAdapter contract
│   │   ├── registry.ts           # url → adapter resolution
│   │   ├── greenhouse.ts
│   │   └── lever.ts
│   ├── core/
│   │   ├── job-extractor.ts      # JSON-LD (via a job-detail adapter) → meta tags → heuristics
│   │   ├── field-matcher.ts      # generic: label/name → dictionary or remembered answer
│   │   ├── form-filler.ts        # framework-safe value setting — generic, any site
│   │   └── submission-detector.ts # generic — see "Submission detection"
│   └── lib/{storage,logger}.ts
├── assets/
└── package.json
```

Everything under `core/` runs on any page the extension has permission for —
none of it is specific to a job site. `job-detail-adapters/` is the one
site-specific folder, and it's small on purpose: see below.

## Detecting an application page

Generic, not tied to a list of known sites: the content script looks at the
current page's forms and checks for a handful of application-shaped signals —
a file input alongside a handful of {name, email, phone} fields is a strong
one. If it looks like an application form and the extension doesn't have
permission for this domain yet, the overlay offers to enable it (see
[Permissions](#permissions)). Once enabled, the overlay shows up on every
visit to that domain going forward.

No allow-list of ATS platforms to detect against — the same check runs
everywhere. This is what makes "the forms can be on any website" true.

## Extracting job info

Job details (title, company, description) matter for what shows up on the
dashboard, and they're extracted the same way regardless of which site the
form is on. Try each source in order and use the first one that has data:

1. **`JSON-LD JobPosting`**, if the page emits it — Greenhouse and Lever both
   do, so `job-detail-adapters/greenhouse.ts` and `lever.ts` exist purely to
   parse `title`, `hiringOrganization`, `jobLocation`, `description` out of
   that structured, machine-readable script tag on those two sites. This is
   the one place site-specific code earns its keep: the data's already there
   for free, and parsing it beats guessing.
2. **Meta tags** (`og:title`, etc.) — the generic fallback for every other
   site.
3. **A generic heuristic** as a last resort — the page's `<h1>` for the
   title, `document.title` parsing for the company name.

No confidence scores, no "surface low-confidence fields for correction." If
the job title comes back empty, show an empty field in the review panel and
let the user type it in. That's simpler than scoring confidence and just as
effective at this scale.

Adding a job-detail adapter for a third site (say, Ashby, which also emits
`JSON-LD`) is a small, optional addition under `job-detail-adapters/` — it
improves extraction quality on that one site but changes nothing about
whether forms can be filled there, since filling already works everywhere.

## Matching fields

This is the part that decides whether autofill actually saves the user time,
so it's worth being explicit about how it works. Two tiers, tried in order,
for every field the page exposes:

**Tier 1 — the profile dictionary.** Build a small signature from the field's
label, `name`, `id`, and `placeholder`. Normalize it (lowercase, strip
punctuation) and look it up in a table of canonical fields and their known
synonyms:

```ts
const FIELD_DICTIONARY: Record<CanonicalField, string[]> = {
  firstName: ['first name', 'given name'],
  lastName:  ['last name', 'surname', 'family name'],
  email:     ['email', 'email address'],
  phone:     ['phone', 'phone number', 'mobile', 'cell'],
  linkedin:  ['linkedin', 'linkedin url', 'linkedin profile'],
  resume:    ['resume', 'cv', 'resume/cv'],
  // ...a couple dozen entries covers the fields that repeat across every ATS
}
```

A match here fills straight from the `User` profile columns (name, email,
phone, resume, etc.) — the same handful of fields every application asks for.

**Tier 2 — the user's own remembered answers.** If a field doesn't match the
dictionary, normalize its signature the same way and check it against the
`FieldAnswer` bank fetched from `GET /api/field-answers` when the page loaded
(see [backend](03-backend.md)). This is where "are you authorized to work in
the US?" or "what's your notice period?" gets filled in automatically the
second time the extension sees a version of that question, on any site —
because the first time, the user typed it by hand and the extension
remembered it (see [Remembering answers](#remembering-answers) below).

The lookup is an **exact match on the normalized signature** — no fuzzy or
similarity matching. "Are you authorized to work in the US?" and "Do you have
US work authorization?" normalize to different signatures and won't match
each other yet. That's a real limit on how often tier 2 actually fires, and a
deliberate one: a wrong autofill on an eligibility question is worse than a
blank one, and the review panel is where a near-miss gets caught, not a
similarity threshold trying to guess at it. See
[future improvements](06-future-improvements.md#smarter-field-matching) for
what a fuzzier version would need.

A field that matches something in `FieldAnswer` is filled and **visibly
marked "from memory"** in the review panel, not indistinguishable from a
fresh fill — the user typed this once for a specific question and should
glance at it again before trusting it on a new one.

**Anything that matches neither tier** is left blank and listed in the review
panel as "not filled — please complete." The user fills it by hand, same as
they would without the extension — and that manual entry is exactly what
feeds tier 2 for next time.

**Free-text questions ("Why do you want to work here?") are handled
separately, by AI, not by either tier above** — see
[AI usage](04-ai-architecture.md). A field only reaches the AI path once both
matching tiers have missed.

## Remembering answers

The capture point is the same review panel that already exists for
[Filling the form](#filling-the-form) — not a separate, silent process
watching every keystroke on every page. When the user reviews a fill (before
confirming it or hitting submit), any field that:

- didn't come from tier 1 (the profile dictionary), and
- has a non-empty value — either the user typed it themselves, or it was
  filled from tier 2 and the user left it as-is or edited it,

gets sent to `POST /api/field-answers` as `{ signature, rawLabel, value }`,
upserted by `[userId, signature]`. A second answer to a question already in
the bank just updates the stored value — there's no history of past answers,
only the latest one.

This deliberately only fires at the one moment the user has already agreed to
share this data with the product (reviewing an application they're about to
submit through it) — it isn't collected from forms the user fills in and
abandons, or from fields the extension didn't even attempt to fill.

## Filling the form

Setting `input.value = x` directly doesn't trigger React's `onChange` — React
wraps the native value setter, so the value visually appears and then gets
discarded on submit. This silently breaks autofill on any React-based form
(Lever included, but this is common across career sites generally, not a
Lever-specific quirk). The fix:

```ts
function setNativeValue(el: HTMLInputElement | HTMLTextAreaElement, value: string) {
  const proto = el instanceof HTMLTextAreaElement
    ? HTMLTextAreaElement.prototype
    : HTMLInputElement.prototype
  Object.getOwnPropertyDescriptor(proto, 'value')!.set!.call(el, value)
  el.dispatchEvent(new Event('input', { bubbles: true }))
  el.dispatchEvent(new Event('change', { bubbles: true }))
}
```

A couple of other input types need real interaction instead of a value
assignment:

- **Native `<select>`** — set the value, dispatch `change`.
- **File input (resume)** — fetch the resume file (via a signed URL, fetched
  by the background worker so the page never sees the storage URL), build a
  `File`, wrap it in a `DataTransfer`, assign `input.files`, dispatch `change`.

After filling, re-read each field and compare it to what was set. If it
doesn't match, leave it flagged in the review panel — a field the extension
*thinks* it filled but didn't is worse than one it left blank, because the
user won't think to check it.

## Submission detection

Two generic signals, not four, and not gated to specific sites:

| Signal | What it means |
|---|---|
| URL change | Navigation to a path that looks like a confirmation page (`/confirmation`, `/thank-you`, `?submitted=true`, etc.), or a same-page view swap on a single-page app |
| DOM text | A confirmation-shaped message appears — a small, generic keyword check ("thank you for applying," "application received," "your application has been submitted") against newly-added text on the page |

Both are pattern checks against generic phrasing, not selectors tied to one
site's markup — the same two checks run everywhere the extension is enabled.
They're necessarily a little fuzzier than a hardcoded selector would be for
one specific site, which is exactly why there's a fallback:

```text
IDLE ──user clicks Submit──► WAITING ──URL or DOM signal appears──► CONFIRMED → report to API
                                │
                                └──10s pass, nothing appears───────► ask the user directly
```

If nothing confirms within about 10 seconds, the overlay just asks: *"Did that
go through?"* with Yes/No buttons. We never guess and save a record on a
guess — a wrong "applied" is worse than a missing one, because the user starts
trusting a dashboard that's lying to them.

On confirmation (automatic or user-confirmed), the background worker calls
`POST /api/applications` with the job data it already extracted. See
[backend](03-backend.md).

## Permissions

Supporting "any site" doesn't mean requesting every site upfront. The
manifest asks for nothing beyond the API's own domain at install time, and
every job-site domain is requested **at runtime, one at a time, with the
user's consent**:

```jsonc
{
  "permissions": ["storage", "activeTab", "scripting"],
  "host_permissions": ["https://api.applyflow.com/*"],
  "optional_host_permissions": ["https://*/*"]
}
```

The flow: `detector.ts` recognizes an application-shaped form on a domain the
extension doesn't have permission for yet → the overlay shows "Enable
ApplyFlow on this site?" → accepting calls
`chrome.permissions.request({ origins: [thisDomain] } )` → Chrome shows its
own native permission prompt scoped to that one domain → granted access is
remembered by Chrome for that domain going forward, no need to ask again on
future visits.

- **No `<all_urls>` at install.** The broadest permission Chrome offers,
  requested upfront, is the thing that most spooks users at install and slows
  Chrome Web Store review. Asking per-domain, only when there's an actual
  form to fill, is both a better trust signal and a faster review.
- **No `tabs` permission** — `activeTab` plus the sender's tab on a message is
  enough.
- Every message handler checks `sender.id === chrome.runtime.id` before doing
  anything with the payload.
- Strict CSP, no remote code, no `eval` — required by the store and also just
  correct given the extension runs on pages we don't control.
- Because the extension can now legitimately end up running on a very wide
  range of third-party domains (not just two named ones), the token-isolation
  rule in [security](05-security.md#extension-specific-rules) — content
  scripts never hold the token, only the background worker does — matters
  even more than it would with a short, curated site list.

## Testing

- **Generic-path tests against saved HTML fixtures from several different
  career sites** — not just Greenhouse/Lever, since the generic detector,
  field matcher, and submission detector are what most users actually run.
  Commit real (anonymized) page HTML from a handful of different company
  career pages and run detection, extraction, and matching against them in
  CI. This is what catches "the generic heuristic doesn't handle this common
  form shape" in a test run instead of in a user's bug report.
- **Job-detail adapter tests against Greenhouse and Lever fixtures**
  specifically, since that's the one piece of genuinely site-specific code —
  a `JSON-LD` parsing regression there is easy to miss otherwise.
- **Manual pass before each release** on a handful of live postings across
  different sites. A synthetic daily monitoring job (loading known postings in
  a headless browser and alerting on drift) is a good idea — see
  [future improvements](06-future-improvements.md) — but it's infrastructure
  that isn't worth building before there are enough users for silent breakage
  to matter.
