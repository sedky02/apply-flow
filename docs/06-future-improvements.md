# 6. Future Improvements

Everything here was cut from the MVP on purpose. None of it is a bad idea —
most of it was in the original design — it's just not needed to answer the
one question that matters right now: *does autofill actually save someone
time, and do they keep using it?* Each item below says what it is, why it's
not in the MVP, and roughly what would tell you it's time to build it.

Treat this as a backlog to revisit, not a to-do list to work through in
order. Build the thing your actual users are asking for, not the thing that
was cut most recently.

## Per-site quirk handling

**What:** Dedicated code for sites where the generic detector, field matcher,
or submission detector under-performs — most notably Workday, which runs a
multi-step wizard behind an iframe that a generic same-page heuristic isn't
built to follow.

**Why not now:** Generic detection already covers arbitrary sites at MVP —
that's the whole point of the current design (see
[extension](02-extension.md#detecting-an-application-page)) — so this isn't
about *adding* site support the way it would have been under a
one-adapter-per-ATS model. It's specifically about the sites where "generic"
turns out not to be good enough, and you won't know which those are until
there's usage data showing where fill success drops.

**Build it when:** Fill-success telemetry (see below) or user reports point
at a specific site where the generic approach reliably struggles.

## Smarter field matching

**What:** Two upgrades beyond what's in the MVP (a profile dictionary, plus
an exact-normalized-match memory of the user's own past answers — see
[extension](02-extension.md#matching-fields)):

1. **Fuzzy/similarity matching**, so "Are you authorized to work in the US?"
   and "Do you have US work authorization?" are recognized as the same
   question instead of needing an exact normalized match.
2. **Cross-user sharing** of learned answers for non-personal questions
   ("How did you hear about us?"), so the first user to answer a given
   question benefits everyone else who sees it — the crowd-sourced cache idea
   from the original design, scoped down today to per-user-only memory.

**Why not now:** Both raise the same risk in a different shape: a wrong
autofill on a question the user didn't type themselves is worse than a blank
field, and fuzzy matching in particular means autofill can now confidently
fill in the *wrong* answer to a question that only looks similar. The current
design leans on exact match plus the review panel to keep that risk low;
either upgrade needs real thought about false-match rate before it ships, not
just a bigger dictionary.

**Build it when:** You have data showing how often tier 2 (the exact-match
memory) *should* have fired but didn't because of wording differences — that
tells you whether fuzzy matching is worth the added risk. Cross-user sharing
specifically also needs a policy for which questions are safe to share
(clearly not eligibility/personal ones) before it's worth building.

## Remote-config selectors and drift monitoring

**What:** Fetching selector maps from the API instead of shipping them in the
extension bundle, so a site's HTML change is a config push instead of a
Chrome Web Store release. Paired with a daily job that loads known postings
in a headless browser and alerts if extraction or fill success drops.

**Why not now:** This is real operational infrastructure — worth it once a
selector break affects enough users that finding out from a bug report is too
slow. At MVP scale, a manual check before each release and fixing it in the
next update is an acceptable trade for not building a config-serving system
and a canary job yet.

**Build it when:** You have enough users that a broken site would go
unnoticed for more than a day or two, or shipping an extension update takes
long enough that a config push would meaningfully help.

## Email inbox integration

**What:** Connecting Gmail/Outlook to detect recruiter emails, interview
invites, and rejections, and using that to move applications between statuses
automatically.

**Why not now:** This is close to a second product on top of the first one —
OAuth for a restricted mail scope (which requires a third-party security
review before Google will approve it — start that process early once you
decide to build this, it takes weeks), a sync worker, a classification step,
and a confidence-gated decision about when it's safe to auto-update a status
versus just suggesting one. None of it is needed to prove autofill works.

**Build it when:** The application list is well-used enough that manually
updating status is a real point of friction users complain about.

## Resume tailoring, cover letters, follow-up emails, interview prep

**What:** The rest of the "AI career assistant" ideas — rewriting resume
bullets per job, drafting cover letters, writing follow-up emails, generating
interview prep material.

**Why not now:** Each is its own feature with its own prompt, its own review
UI, and (for resume tailoring specifically) a real safety concern — rewritten
bullets need to be checked against the source resume so nothing gets
fabricated, which is more than a human glancing at a text box should be
trusted to catch reliably. The MVP's one AI feature (drafting a single
question's answer) is deliberately the simplest, lowest-risk slice of this
whole category.

**Build it when:** The core autofill loop is validated and you're looking for
the next feature to build, not before.

## Compatibility / fit scoring

**What:** Scoring how well a job matches the user's profile and explaining
the gaps.

**Why not now:** Needs a structured representation of the job description and
the user's experience to compare against, which doesn't exist yet in this
design — the MVP just stores raw job description text. Worth building once
there's a reason users need a score rather than reading the job description
themselves.

## Multi-resume versions and tailoring history

**What:** Multiple named resumes, versions tailored per application, lineage
between them.

**Why not now:** The MVP has one resume file per user. Versioning is only
useful once there's a tailoring feature (above) producing versions worth
keeping.

## Application history, notes, and reminders

**What:** A status-change audit trail, freeform notes per application,
scheduled reminders (e.g. "nudge me if I haven't heard back in a week").

**Why not now:** The MVP dashboard is a list with a status dropdown. These are
all legitimate, low-risk additions — they just don't need to exist before
there's a list of applications worth taking notes on.

**Build it when:** Users are tracking these things in a separate notes app or
spreadsheet next to the dashboard — a good sign the dashboard should just do
it.

## Analytics (funnel, response rate, time-in-stage)

**What:** Dashboards showing how applications move through stages over time.

**Why not now:** Meaningless without enough applications and enough of a
status-change history to compute from — and the MVP doesn't keep a
status-change history (§ above). Comes after that data exists and after
there's enough volume for the numbers to mean something.

## Background jobs and a queue

**What:** Redis + a job queue (e.g. BullMQ), separate worker processes,
retries and backoff, a dead-letter queue.

**Why not now:** The only slow operation in the MVP is one AI call per
question, handled inline in the API route (see
[backend §3.4](03-backend.md#34-why-the-ai-call-is-synchronous)). A queue
solves problems — a provider outage blocking unrelated traffic, work that
needs to survive a server restart, jobs slow enough that users shouldn't wait
on them — that don't exist yet.

**Build it when:** You add a feature with a genuinely slow operation (e.g.
email sync, or generating a full tailored resume), or the AI endpoint's
latency or cost profile changes enough that holding a request open for it
stops being fine.

## Tenant-isolation hardening

**What:** A repository base class that all user-scoped queries go through, a
database-level backstop (Mongo has no direct row-level-security equivalent,
so this would mean a mandatory data-access layer rather than a DB policy),
and a CI check that fails the build on a raw Prisma call outside that layer.

**Why not now:** With one engineer, "always filter by `userId`, and review
your own diff for it" is a real and sufficient control — see
[backend §3.2](03-backend.md#32-keeping-data-scoped-to-the-right-user). The
extra layers exist to catch a mistake slipping past review when multiple
people are shipping code without seeing each other's changes.

**Build it when:** A second engineer joins.

## Auth hardening

**What:** Refresh-token rotation with reuse detection, Google OAuth login,
account lockout backoff, a security-relevant audit log.

**Why not now:** Email/password with a short-lived JWT is a normal,
sufficient starting point. Rotation-with-reuse-detection specifically matters
once a stolen token being replayed is a scenario worth detecting rather than
just outliving via expiry.

**Build it when:** Before charging money or storing anything more sensitive
than what's here today — whichever comes first.

## Compliance automation (GDPR export/erasure)

**What:** Self-serve "export my data" and "delete my account" flows that
actually purge MongoDB, object storage, and any third-party grants.

**Why not now:** At MVP user counts, an engineer running a delete script by
hand when someone emails asking is a real, honest answer. Automate it before
it's the only thing standing between you and a compliance obligation you
can't handle by hand anymore.

## Evals and prompt versioning

**What:** A golden set of test cases for the AI answer-drafting prompt, run in
CI on every prompt change; a table recording which prompt version produced
which output.

**Why not now:** There's one prompt. `git log` on one file is a sufficient
version history, and eyeballing a handful of outputs after a prompt edit is a
sufficient eval process. This becomes real infrastructure once there are
several prompts that interact, or once a bad prompt change reaching
production is expensive enough to justify a gate.

## Referral matching, compensation intelligence, and outcome analytics

**What:** Using inbox history to surface people the user knows at a target
company; anonymized, opt-in salary data at the offer stage; correlating
resume version and application patterns against real outcomes to tell users
something like "your callback rate is higher at smaller companies."

**Why not now:** All three need data this MVP doesn't collect yet (inbox
access, an opted-in cohort, enough volume of tracked outcomes per user to say
anything statistically honest), and the last two specifically require real
privacy engineering — k-anonymity thresholds, a separate data pipeline — to
do without accidentally leaking something about a specific person. Not a
weekend addition once the data exists.

## Billing

**What:** Paid plans, usage-based limits on AI generations, Stripe
integration.

**Why not now:** No paid product yet. Build it right before you actually
charge someone, informed by which features people used enough to pay for —
guessing at plan tiers now would be designing against imagined usage instead
of real usage.

## Mobile companion

**What:** A PWA for checking status and reminders from a phone.

**Why not now:** Applying happens at a desk. This is a nice-to-have once
there's enough going on (reminders, status changes) to be worth checking
on the go — which, per the items above, isn't built yet either.
