# 4. AI Usage

AI does exactly one job in this product: **drafting an answer to a free-text
application question** that isn't in the user's profile and isn't in the
field dictionary — things like "Why do you want to work here?" or "Describe a
time you solved a hard problem."

Everything that isn't that — field mapping, email classification, resume
scoring, resume tailoring — either doesn't exist in the MVP or is done with
plain code. If you're looking for a LangGraph pipeline, a provider router, a
vector store, or a fabrication-detection critic model: none of that is here.
It's all in [future improvements](06-future-improvements.md), because none of
it is needed to answer one text question well.

## 4.1 The one call

```ts
// lib/ai.ts
export async function draftAnswer(input: {
  question: string
  jobDescription: string
  resumeText: string
}): Promise<string> {
  const response = await anthropic.messages.create({
    model: 'claude-sonnet-5',
    max_tokens: 400,
    system: SYSTEM_PROMPT,
    messages: [{
      role: 'user',
      content: `Job description:\n${input.jobDescription}\n\nResume:\n${input.resumeText}\n\nQuestion:\n${input.question}`,
    }],
  })
  return extractText(response)
}
```

One provider, one model, one prompt, called directly from the API route. No
provider abstraction layer, no "task → tier → model" router. Those exist to
let you swap providers or price-optimize across dozens of different AI
features — with one feature, picking one provider and calling its SDK is the
entire abstraction you need. If a second provider becomes worth supporting
later, that's a small, contained change to this one function, not a redesign.

## 4.2 Facts are answered from the profile, not generated

Work authorization, sponsorship needs, and similar eligibility questions are
**facts about the user, not something to compose text about** — and getting
one wrong on a real application is a real problem for the user, not just a
bad paragraph.

The fix is simple: a short list of keyword patterns catches these questions
before they ever reach the LLM.

```ts
const FACT_QUESTIONS: Array<{ pattern: RegExp; answer: (user: User) => string | null }> = [
  { pattern: /authorized to work/i, answer: (u) => boolToYesNo(u.workAuthorized) },
  { pattern: /require.*sponsorship/i, answer: (u) => boolToYesNo(u.needsSponsorship) },
]

function findFactAnswer(question: string, user: User): string | null {
  const match = FACT_QUESTIONS.find((f) => f.pattern.test(question))
  return match ? match.answer(user) : null
}
```

If a question matches, answer it directly from `User` fields and skip the LLM
call entirely. If the relevant field is unset, tell the user to fill it in on
their profile rather than guessing. This is plain pattern matching — no
classifier model deciding what "kind" of question this is, because a wrong
guess here is exactly the failure mode worth spending five minutes of regex
instead of an AI call to avoid.

## 4.3 Everything the LLM writes is a draft, not a submission

The extension shows the generated answer in a text box the user can edit
before it goes into the form, and nothing is auto-submitted regardless (see
[security](05-security.md#the-submit-boundary)). That review step is the
actual safety net here — not a second AI model checking the first one's work.
An earlier version of this design had a dedicated "critic" model whose only
job was fact-checking the writer model's output before anything reached the
user. That's a legitimate technique, but it's a second model call, a second
prompt to maintain, and a second thing that can be wrong — for one text box
that a human reads before submitting anyway, it's solving a problem the review
step already solves. Revisit this if answers start going out unreviewed
somehow, or once there's a feature (like tailoring a whole resume) where a
human skimming the output isn't a reliable enough check — see
[future improvements](06-future-improvements.md).

## 4.4 The prompt

```text
You are helping a job applicant answer a question on a job application.

Rules:
- Only use facts present in the resume text provided below. Do not invent
  work experience, skills, or accomplishments that aren't in it.
- Keep the answer concise and in the first person, as if the applicant wrote it.
- If the question asks about something not covered by the resume, write a
  general but honest answer rather than fabricating specifics.

<job_description>
{{jobDescription}}
</job_description>

<resume>
{{resumeText}}
</resume>

Question: {{question}}
```

One prompt, versioned by being in source control like any other code — no
separate prompt-versioning table, no semver export. If you change the prompt
and the answers get worse, `git log` tells you what changed and when; that's
enough history for one prompt.

The job description and the question both come from a page we don't control,
so they're untrusted text. The mitigation for that is short, because there's
no agent here for an injected instruction to hijack: **the model has no tools
and no ability to take any action** — it reads text and returns text. Wrap the
job description and resume in tags as shown above so the model can tell "data
to read" apart from "the actual instruction," and that's the whole prompt
injection story at this scope. The elaborate version of this (tool
allowlists, an operator-only instruction channel, an injection eval suite) is
worth building once there's a graph with tools that can mutate state — see
[future improvements](06-future-improvements.md). There's nothing for an
injection to do here yet.

## 4.5 No memory, no learning, and that's fine for now

Each call gets exactly the resume text, the job description, and the
question — nothing persisted between calls, nothing learned from past edits,
no retrieval step. That means the model doesn't get better at matching the
user's writing voice over time. It's also one prompt and zero data pipelines,
which is the right trade until there's evidence the plain version isn't good
enough. See [future improvements](06-future-improvements.md) for what a
"learns your voice from your edits" version would need.
