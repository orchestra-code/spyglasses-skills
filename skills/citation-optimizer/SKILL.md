---
name: citation-optimizer
description: Run the Spyglasses AI Citation Optimizer's score → revise → re-score loop to make a web page more likely to be CITED by ChatGPT for a target query. Use when the user wants to improve a page's AI citation readiness, rewrite a page to rank/get cited in ChatGPT, optimize content for a fan-out query, or asks "why won't ChatGPT cite my page" and wants it fixed. Drives the `spyglasses` MCP citation tools, generating a revised draft and confirming the improvement — and knowing when to STOP.
---

# Spyglasses Citation Optimizer

This skill runs an **authoring loop**: score a page against a real query the way ChatGPT's
retrieval pipeline would, generate a meaning-preserving **revision** that addresses what the
scorer flagged, then **re-score** to confirm the gain — repeating only until the content is
publish-ready. It drives the `spyglasses` MCP server (OAuth, no API key). Everything creates the
user's *own* runs/revisions; **nothing is published** — the user publishes the result themselves.

Scope: **ChatGPT** (ChatGPT Search) only. Google AI Overviews / AI Mode / Claude have their own
separate pipelines and are out of scope here.

## The loop

```
list_tracked_fanouts     → pick a REAL fan-out to optimize for (per platform)
        ↓
match_pages_for_fanout   → pick the closest existing page (avoid self-competition)
        ↓
score_citation_pipeline  → returns runId   (background; poll get_pipeline_run)
        ↓
get_pipeline_run(runId)  → score + gate recommendations + READINESS verdict
        ↓  (readiness = "revise_again")
revise_content(runId)    → returns revisionId  (background; poll get_revision)
        ↓
get_revision(revisionId) → revised markdown + meta + JSON-LD + change log
        ↓
rescore_revision(revisionId) → returns a new runId  (background; poll get_pipeline_run)
        ↓
get_pipeline_run(newRunId) → new score + readiness  →  loop or STOP
```

Scoring and revising run as **background jobs**, so those tools return an id immediately. **Poll**
the matching `get_*` tool every few seconds until `status` is terminal (`completed`/`stopped` for
runs; `completed`/`failed` for revisions).

### Scoring can take several minutes — set expectations and offer to notify

A scoring run has one inherently slow step. Most gates finish in seconds, but the rerank step
(`rerank-best-chunk`, feeding Gate 6 / `chatgpt.embed_and_rerank`) calls a **third-party neural
reranker hosted on an external API (Replicate)**. When that model is idle it scales to zero, so the
first run of a quiet period pays a **cold start that can take several minutes** (we've seen ~3
minutes) before the run continues. After that the model stays warm and later runs are fast. **A run
sitting on the rerank step for a few minutes is normal — not a failure or a hang.**

**Set expectations before you start, and offer to run it in the background:**

- Tell the user up front: *"Scoring runs an external AI reranker that can take a few minutes to
  spin up on the first run — I'll keep an eye on it and let you know when it's done."*
- **Offer to poll in the background and notify on completion** rather than blocking the
  conversation. If your assistant supports long-running / background tasks with a notification when
  finished (Claude can do this for long tasks), use it: kick off the score, then poll
  `get_pipeline_run(runId)` periodically and notify the user when `status` becomes
  `completed`/`stopped`. If background tasks aren't available, just poll patiently and reassure the
  user it's still working.

**While polling:** check `get_pipeline_run` every ~10–15s. As long as new entries keep appearing in
`gateResults` or `progressPercent` keeps rising, it's working — don't give up or report an error
while the run is still `running`. The early gates light up quickly; the visible pause is the
reranker cold start. Stop polling when `status` is terminal (`completed`/`stopped` for runs;
`completed`/`failed` for revisions).

## Tools

- `list_tracked_fanouts(propertyId, platform?)` — synchronous. **Start here.** Lists the property's
  REAL tracked fan-outs (the queries the platform's pipeline actually generated, from its latest AI
  Visibility report), highest-impact first, each with an `isGap` flag. Fan-outs are tracked per
  platform — **ChatGPT, Claude, Google AI Overviews** — but only **ChatGPT** can be *scored* today:
  the result's `scoreable` flag + `availability[]` say which. **Users can also bring their own
  keyword or phrase** — e.g. a query from another SEO tool or one they already know they want to
  rank for. That's a normal, fully-supported input: just pass it as `fanOutQuery` to
  `match_pages_for_fanout` / `score_citation_pipeline`. Offer the tracked list as a helpful starting
  point, but never make a user feel their own keyword is second-class — score it the same way. (A
  tracked fan-out simply adds the confirmation that the platform was observed generating it.) If the
  property has no report yet, the tool says so.
- `match_pages_for_fanout(propertyId, fanOutQuery)` — synchronous. Ranks the property's pages by
  full-content similarity to the chosen fan-out. Use it to optimize the *closest* page; two of the
  user's pages competing for one citation helps neither.
- `score_citation_pipeline(propertyId, fanOutQuery, {propertyPageId | url | draftMarkdown}, pageType?, groundingSearchId?)`
  — enqueue scoring; returns `runId`. Provide exactly one content source. **When the fan-out came
  from `list_tracked_fanouts`, pass its `groundingSearchId`** — this marks the query as a verified
  tracked fan-out (otherwise the pipeline treats it as a hand-entered keyword) and lets the SERP gate
  reuse stored standings. Omit it for a custom keyword the user brought.
- `get_pipeline_run(runId)` — status, `overallScore` (0–100), per-gate `status` + `summary` +
  `recommendations`, and a **`readiness`** verdict (see below). Poll target after scoring/re-scoring.
- `revise_content(runId)` — enqueue a revision of that run's content; returns `revisionId`.
- `get_revision(revisionId)` — the revised `revisedMarkdown`, `metaTitle`, `metaDescription`,
  `jsonLd`, and a `changeLog` (each entry traces an edit to the recommendation that motivated it).
- `rescore_revision(revisionId)` — score the revised content as a new run; returns the new `runId`.

## Reading the score

`get_pipeline_run` returns gate results. The gates that matter most for "will ChatGPT cite this":

- **embed_and_rerank (6)** / **deep_read_audition (7)** — is your best passage relevant + competitive
  enough to be read? Their `poolRank` is your best chunk's rank against the pages that already rank.
- **synthesis_readiness (8)** — is the passage answer-first, self-contained, and entity-dense enough
  to be quoted? Gate 8 warnings are the most common revision target.

Each gate's `recommendations` carry a `signal`, a `message` (what to change), and a `rationale`
(why). `revise_content` consumes these automatically — you don't hand-edit; you call the tool.

## When to STOP (do not loop forever)

`get_pipeline_run` returns `readiness.recommendation`:

- **`pending`** — the run is **still scoring** (not yet `completed`/`stopped`). The verdict isn't
  meaningful yet. **Do NOT act on readiness while it's pending** — keep polling until the run is
  terminal, THEN read the real verdict. (Readiness is only computed once the run finishes; a
  mid-run verdict would be based on partial gates and a null score.)
- **`publish_ready`** — gates 6–8 are green and the page is competitive (or there's no SERP pool to
  compare against). **STOP.** Tell the user it's ready to publish (content must be published to climb).
  `readiness.assistantInstruction` will say so explicitly.
- **`plateaued`** — the last revision didn't improve the score. **STOP.** Present the best version
  (`readiness.bestIterationRunId`) and suggest a manual edit; another automated pass won't help.
- **`revise_again`** — there's a clear opportunity. Run one `revise_content` → `rescore_revision`
  pass, then re-check readiness.

Honor `readiness.assistantInstruction` when present — it is an explicit STOP. Don't re-revise a
`publish_ready` / `plateaued` run unless the user asks. A reasonable cap is **2–3 revise passes**;
if not publish-ready by then, hand back the best version and explain what still needs manual work.

## Guardrails

- The revision is **meaning-preserving**: it front-loads, tightens, and restructures what the page
  already says; it does **not** invent facts, figures, quotes, or credentials. If the user wants new
  claims added, that's their edit to make and substantiate — not the optimizer's.
- **Superlative hygiene** (especially legal / medical / financial): matching a "best…" or
  "top-rated…" query does **not** mean asserting the page is the best. The optimizer keeps a
  superlative only if the original substantiated it. Don't override this.
- **Any query is welcome — tracked or the user's own.** A tracked fan-out (from the user's AI
  Visibility report) is a query the platform was observed generating, which adds confidence. A
  custom keyword the user brings (e.g. from another SEO tool) is equally valid and scored the same
  way — don't gatekeep it or treat it as inferior. At most, briefly note that a tracked fan-out
  would add that observed-behavior confirmation if they have one; keep it encouraging, not a warning.

## A typical run

1. `list_properties` → get the `propertyId` (this lives in the broader `spyglasses` toolset).
2. `list_tracked_fanouts(propertyId)` → pick a fan-out `query` (defaults to ChatGPT, the scoreable
   pipeline). If the user is interested in Claude or Google AI Overviews, pass that `platform` to
   show their tracked fan-outs — but note those can't be scored yet, only ChatGPT. **Or** skip the
   list entirely and use a keyword the user brings — that's a normal, equally-supported path.
3. `match_pages_for_fanout(propertyId, query)` → pick `propertyPageId` of the closest page.
4. `score_citation_pipeline(propertyId, query, { propertyPageId })` → `runId`; poll `get_pipeline_run`.
   **If the query came from `list_tracked_fanouts`, also pass its `groundingSearchId`** so it scores
   as a verified tracked fan-out (not a hand-entered keyword).
5. Read the score + readiness. If `revise_again`: `revise_content(runId)` → poll `get_revision` →
   show the user the change log → `rescore_revision(revisionId)` → poll the new run.
6. Repeat step 5 until `publish_ready` or `plateaued` (or the 2–3 pass cap), then present the final
   revised markdown + meta + JSON-LD and tell the user to publish it.
