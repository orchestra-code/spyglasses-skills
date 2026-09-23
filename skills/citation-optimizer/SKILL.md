---
name: citation-optimizer
description: Run the Spyglasses AI Citation Optimizer's audit loop (score, revise, re-score) to make a web page more likely to be CITED by ChatGPT, Google AI Overviews, Google AI Mode and Claude. Use when the user wants to improve a page's AI citation readiness, score a page or draft against the searches their buyers actually run, rewrite a page so assistants will quote it, brief a page that does not exist yet, or asks why ChatGPT will not cite their page and wants it fixed. Drives the `spyglasses` MCP citation tools and knows when to STOP.
---

# Spyglasses Citation Optimizer

This skill runs an **authoring loop**. Score one page against a **set of searches** on
every AI assistant the user's plan covers, read the harmonized findings, generate a
meaning-preserving **rewrite** that addresses them, then **re-score** to confirm the lift,
repeating only until the content is publish-ready. It drives the `spyglasses` MCP server
(OAuth, no API key).

The citation tools are the part of the Spyglasses connector that **writes**. Everything
they create is the user's own: scoring runs, **draft** rewrites and briefs. Nothing is
published, no live page is edited, and no existing report or data is changed. Do not
describe this surface as read-only.

Scope: **ChatGPT, Google AI Overviews, Google AI Mode and Claude**. Which of those can be
scored at a given moment is reported by the tools themselves in `scoreablePlatforms`, and
the user's plan decides which of those they may select. Never hardcode "ChatGPT only" into
an answer; read the field.

## Two shapes of the loop

| | Audit (the default) | Single run (legacy) |
|---|---|---|
| Scores | One page against a **query set**, on every assistant the plan covers | One page against **one query**, on ChatGPT |
| Start with | `score_citation_audit` | `score_citation_pipeline` |
| Poll | `get_citation_audit` | `get_pipeline_run` |
| Revise | `revise_citation_audit` | `revise_content` |
| Verdicts | `pending`, `revise_again`, `publish_ready`, `plateaued`, `regressed` | `pending`, `revise_again`, `publish_ready`, `plateaued` |

**Use the audit unless there is a reason not to.** A page that appears for several related
searches is far likelier to be cited than one that wins a single query, and the four
pipelines disagree often enough that optimizing against one in isolation can cost score on
the others. The audit runs them together and harmonizes the findings into one score and
one list.

Use the single run only when the user explicitly wants one query on one assistant, or when
their plan covers nothing else (the free tier is ChatGPT only, and the single-run tools are
the shape it gets). The single-run tools still work unchanged.

## The audit loop

```
list_tracked_fanouts      → build the QUERY SET (per platform; read scoreablePlatforms)
        ↓
match_pages_for_fanout    → pick the closest page   (or list_property_pages / list_placements)
        ↓
score_citation_audit      → returns auditId   (background)
        ↓
get_citation_audit(id)    → poll. Composite score + range, per-platform runs,
                            mergedRecommendations, readiness, usage
        ↓  (readiness.recommendation = "revise_again")
revise_citation_audit     → returns revisionId  (SPENDS CREDITS)
        ↓
get_revision(revisionId)  → revised markdown + meta + JSON-LD + change log
        ↓
rescore_revision(revId)   → kind: "audit" + a new auditId
        ↓
get_citation_audit(new)   → loop, or STOP on the verdict
```

Scoring, revising and brief-writing are **background jobs**, so those tools return an id
immediately and you poll the companion tool.

| Kick off (returns an id) | Poll until done |
|---|---|
| `score_citation_audit` → `auditId` | `get_citation_audit(auditId)` until `completed` |
| `revise_citation_audit` → `revisionId` | `get_revision(revisionId)` until `completed` |
| `rescore_revision` → `auditId` or `runId` | `get_citation_audit` / `get_pipeline_run` |
| `generate_citation_outline` → `outlineId` | `get_citation_outline(outlineId)` until `completed` |
| `score_citation_pipeline` → `runId` | `get_pipeline_run(runId)` until `completed` / `stopped` |
| `revise_content` → `revisionId` | `get_revision(revisionId)` until `completed` |

## Where to start

All entry points converge on `score_citation_audit`. Only the way you pick the target and
the searches differs.

- **Fan-out-first**, "optimize for this search" or "fix our visibility on this fan-out".
  `list_tracked_fanouts` → `match_pages_for_fanout` → score the closest page. The default.
- **Page-first**, "make this page more citable" or "optimize my pricing page". The user
  names a page, not a search. `list_property_pages(propertyId, search?)` resolves it to a
  `propertyPageId`. You still need searches, because every check is relative to one, so
  offer `list_tracked_fanouts` or take the keywords the user names.
- **Placement-first**, "score and revise this PR placement". `list_placements(propertyId)`
  to find it (pick one with `hasContent: true`), `get_placement(placementId)` to read its
  `content`, then score that content as `draftMarkdown` with `pageType: "press_release"`.
  Placements rarely carry a meta title or description and that is fine; the scorer runs
  without them and simply skips the listing sub-check.
- **Keyword-first**, "we have nothing on this topic yet". The page does not exist, so score
  nothing: go to the outline flow below, then score the draft.

## Building the query set

`score_citation_audit` takes `queries`, an array of `{ query, role?, provenance?,
groundingSearchId? }`. One primary plus its fan-outs. Two to six members is the useful
range; the hard limit is 25.

- **Exactly one member must have `role: "primary"`** (the search the page is really for).
  The rest default to `fanout`.
- Build the set from `list_tracked_fanouts` where you can. Searches the user brings are
  fully supported, they just carry a different provenance label.

### Provenance is not book-keeping

| `provenance` | Means | Pass it when |
|---|---|---|
| `tracked` | This brand's assistants were **observed** running the search | It came out of `list_tracked_fanouts`. Pass its `groundingSearchId` too, which lets the SERP checks reuse stored standings |
| `freetext` | The user typed it | The user gave you the wording. This is the default |
| `generated_unverified` | You or a tool suggested it | You invented it, or a model proposed it |

The label survives into the score, the rewrite and every screen the user later sees. By the
time a brief or a results screen reaches a person there is no way left to tell a suggestion
from an observation, so **never generate fan-outs and report them as observed**. Use
`generated_unverified` and say so in your own words too.

### site:-scoped and competitor-named fan-outs

`list_tracked_fanouts` excludes two kinds by default because nothing this brand publishes
can rank for them, and reports how many exist in `siteScopedCount` and
`competitorNamedCount`.

- A `site:`-scoped fan-out (`site:example.com pricing`) is **evidence** about which sites an
  assistant already trusts. It may be added to an audit's query set as a fan-out, where it
  is kept for the named-in-searches diagnostic and marked `scoreable: false`. As the
  **primary** it is refused, and `score_citation_pipeline` rejects it outright.
- A fan-out naming a **competitor** and not this brand is informational only. A page about
  this brand's product cannot answer a question about a rival's.
- A **comparison** fan-out naming both brands ("Us vs CompetitorX") is an ordinary
  scoreable row and is always listed.

Pass `includeSiteScoped: true` / `includeCompetitorNamed: true` only to *show* the evidence,
never to pick something to score.

## Tools

### Picking the searches and the target

- `list_tracked_fanouts(propertyId, platform?, limit?, includeSiteScoped?, includeCompetitorNamed?)`
  Synchronous. **Start here.** The property's real tracked fan-outs from its latest AI
  Visibility report, highest-impact first, each with an `isGap` flag. Fan-outs are tracked
  per platform. The result carries `scoreable`, per-platform `availability`, and
  `scoreablePlatforms`, which is read live and can change without a release. If the property
  has no completed report the tool says so, and you score searches the user brings with
  provenance `freetext`.
- `match_pages_for_fanout(propertyId, fanOutQuery, limit?)` Synchronous. Ranks the
  property's pages by full-content similarity to a search. Optimize the closest existing
  page; two of the user's pages competing for one citation helps neither.
- `list_property_pages(propertyId, search?, limit?)` Synchronous. Pages with id, path, title
  and intent tags, optionally filtered over path and title. The page-first path.
- `list_placements(propertyId, limit?)` Synchronous. PR placements with id, title, url,
  status, mode, PQS score and `hasContent`.
- `get_placement(placementId)` Synchronous. A placement's `content` plus title and url.

### The audit

- `score_citation_audit(propertyId, queries, platforms?, pageType?, {propertyPageId | url | draftMarkdown}, metaTitle?, metaDescription?, outlineId?)`
  Enqueues the audit and returns `auditId`, `platforms`, `primaryQuery`, `queryCount`.
  Provide **exactly one** content source. `platforms` defaults to every assistant the plan
  covers that is live; naming one outside the plan is refused with the allowed list.
  `pageType` is `product | homepage | informational | press_release | general | unknown`
  (default `informational`) and drives the rewrite template. Pass `outlineId` when the draft
  was written from a brief.
- `get_citation_audit(auditId)` The poll target and the whole picture: `status`,
  `progressPercent`, `composite` (`score`, `low`, `high`, `contentScore`, `weights`),
  `queries` with their provenance and `scoreable` flags, `platforms[]` (each with its own
  `contentScore`, `overallScore`, `scoreLow`, `scoreHigh`, `haltedAtGate`, `errorMessage`
  and `checks[]`), `recommendations` grouped as `consensus` / `platformSpecific` /
  `conflict`, `revisions`, `readiness`, and `usage` (the monthly page meter).
- `reweight_citation_audit(auditId, weights)` **Free and instant.** Recomputes the combined
  score and the readiness verdict from results already collected. No re-scoring, no credits,
  no page used. See "Re-weighting" below.
- `revise_citation_audit(auditId, profile?)` **Spends credits.** Enqueues a rewrite grounded
  in every assistant's findings at once and returns `revisionId`, `iteration`, `profile` and
  the `budget`. `profile` defaults to `harmonized`, which is almost always right; naming a
  single platform optimizes for that one alone and usually costs score on the others.
- `get_revision(revisionId)` The revised `revisedMarkdown`, `metaTitle`, `metaDescription`,
  `jsonLd`, and a `changeLog` tracing each edit to the finding that motivated it. `auditId`
  tells you which loop this revision belongs to.
- `rescore_revision(revisionId)` Closes the loop and matches whatever the revision came
  from. A revision born from an audit re-scores as a **child audit** (the same query set,
  the same pinned competitor pool, every assistant again) and returns `kind: "audit"` plus
  an `auditId`. A revision from a single run returns `kind: "run"` plus a `runId`. **Read
  `kind`** to know which poll target to use. Re-scoring is always free and never counts
  against the page allowance.

Pinning the competitor pool is what makes the before-and-after mean anything: the rewrite is
measured against the identical competitors rather than against whatever happens to rank
today.

### The single-run (legacy) tools

- `score_citation_pipeline(propertyId, fanOutQuery, groundingSearchId?, pageType?, {propertyPageId | url | draftMarkdown}, metaTitle?, metaDescription?)`
  One query, ChatGPT only. Returns `runId`. Pass `groundingSearchId` when the query came from
  `list_tracked_fanouts`, which marks it `tracked` rather than hand-entered.
- `get_pipeline_run(runId)` Status, `overallScore`, `progressPercent`, `gateResults` with
  `summary` and `recommendations`, `revisionIds`, and a `readiness` verdict whose
  `bestIterationRunId` names the version to keep.
- `revise_content(runId)` **Spends credits.** A rewrite from one run's findings. Prefer
  `revise_citation_audit` whenever the page was scored as an audit; a rewrite driven by one
  assistant's findings can cost score on the others.

### Before the page exists

- `generate_citation_outline(propertyId, keyword, pageType, platforms?)` **Spends credits.**
- `get_citation_outline(outlineId)` The poll target.

## Reading the results

**Every score has a range, and the range is not decoration.** These pipelines do not make
the same decision every time they see the same page, so the score comes with a band (never
narrower than plus or minus 8 points). A change **inside** the range has not been shown to
be a change. Never report a few points as an improvement; say the score did not move
meaningfully.

**The composite drops what did not finish.** An assistant whose run failed is removed from
the average, top and bottom, never counted as zero, because a failed run says nothing about
the page. There is **no floor**: a page that does well on three assistants and fails
outright on the fourth is reported as a good score with a warning beside it. Say the warning
out loud; do not let the headline number stand alone.

**Before-and-after uses the content-only sub-score** (`composite.contentScore`). It is the
only figure that stays comparable between a published page, which has a search ranking, and
a draft, which does not.

**Every finding carries a lever**, which says what can move it.

| Lever | Means |
|---|---|
| `content` | The words on the page can change it. This is what a rewrite works on |
| `metadata` | The title, description, dates or snippet directives |
| `technical` | How the page is served: rendering, robots rules, source order |
| `offpage` | Nothing on the page moves it. Context only, outside the rewrite's objective |

Some checks are reported and deliberately left out of the number. **Named in searches**, how
often an assistant writes the brand or domain into its own searches, is the strongest signal
visible and no rewrite moves it, so it sits next to the score rather than in it. Present it
as a diagnostic, never as something the next pass will fix.

**Recommendations arrive grouped.** `get_citation_audit` returns them under
`recommendations.consensus`, `recommendations.platformSpecific` and
`recommendations.conflict` (the `kind` field on each item reads `consensus`,
`platform_specific`, `conflict`).

| Kind | What it means | What to do |
|---|---|---|
| `consensus` | Several assistants asked for the same thing | Do these first; they pay off everywhere |
| `platform_specific` | One assistant asked alone | Worth doing, weighted by how much that assistant matters to this brand |
| `conflict` | Two asks that cannot both be satisfied in the same passage | **Follow `resolution.allocations`**, which gives each ask a different part of the page. Averaging two incompatible instructions produces a rewrite that satisfies neither |

## When to STOP

`readiness.recommendation` on `get_citation_audit`:

| Verdict | What it means | What to do |
|---|---|---|
| `pending` | Nothing has finished scoring yet, so the verdict is not meaningful | Keep polling. Do **not** act on a pending verdict |
| `revise_again` | There is real headroom; at least one assistant's selection checks are not passing | Run one `revise_citation_audit` → `rescore_revision` pass, then re-check |
| `publish_ready` | Every assistant that finished passes its selection checks and the combined score clears the bar | **STOP.** Tell the user to publish it; content has to be live to climb |
| `plateaued` | The last pass did not move the score beyond its range | **STOP.** Present the best version (`readiness.bestAuditId`) and suggest a manual edit |
| `regressed` | The rewrite made at least one assistant meaningfully worse, even though the combined score held up | **STOP.** The version was held back. Present the **previous** one, named by `readiness.bestAuditId`, and say which assistants fell (`readiness.regressedPlatforms`) |

`regressed` is the one that is easy to miss, because the combined number alone will not tell
you. The rewrite's objective is "raise the combined score **without making any assistant
worse**", so a version that traded Claude for the average has not met it. Nothing is lost:
the live page is untouched and the draft is still there if the user disagrees.

**Whenever `readiness.assistantInstruction` is present it is an explicit STOP.** Honour it.
Do not re-revise a `publish_ready`, `plateaued` or `regressed` audit unless the user asks.
A reasonable cap is **2 to 3 revise passes**; if the page is not publish-ready by then, hand
back the best version and explain what still needs a human edit.

The single-run loop uses the same verdicts minus `regressed`, and names its best version
`readiness.bestIterationRunId`.

## Re-weighting

Weights are equal by default. When the user tells you where their audience actually is
("most of our buyers still start on Google", "our people live in ChatGPT"), call
`reweight_citation_audit(auditId, weights)`.

- Weights can be any non-negative numbers (shares, percentages, plain counts). They are
  normalised.
- **The verdict follows the weights.** The score, the range and the readiness verdict
  recompute together, so re-weighting can turn `revise_again` into `publish_ready` or the
  other way round. Read the returned `readiness` rather than the one from before.
- Naming an assistant the audit did not run is refused rather than quietly ignored.
- The returned `contributing` and `dropped` lists say which platforms entered the average.
- The new mix is saved on the audit, so a shared link shows the same score and verdict the
  sender saw, and it becomes the property's default for its next audit.

It costs nothing, uses no page from the allowance and is instant, so offer it freely.

## Briefing a page that does not exist yet

Scoring answers "why was this page not cited". A brief answers "what should the page say in
the first place".

1. `generate_citation_outline(propertyId, keyword, pageType, platforms?)` **Spends credits**,
   on the same monthly allowance as a rewrite. `pageType` is one of `homepage`, `product`,
   `informational`, `press_release`; there is no fallback, because each has its own template.
   A `site:`-scoped keyword is refused. Returns `outlineId` and the `budget`.
2. Poll `get_citation_outline(outlineId)` until `status` is `completed`. It returns the
   `querySet` the brief was built for (each member labelled with where it came from), the
   `gaps` (what the ranking pages cover and what none of them answers), and the `brief`
   itself plus `briefMarkdown`: sections with target searches, word budgets, must-include
   terms and the rules that decide whether a passage can be quoted, with the FAQ and the meta.
3. Present the brief. **Keep the provenance labels.** A search marked
   `generated_unverified` is one we suggested; presenting it as one this brand's assistants
   were observed running is the single mistake this flow exists to prevent.
4. "Score the draft" means `score_citation_audit` in draft mode: pass the written text as
   `draftMarkdown`, the brief's searches as the query set, the same `pageType`, and
   `outlineId`. It then scores against the same searches the brief was built from. A draft
   scored against a different set answers a different question.

## What it costs

**Scoring costs no credits.** It is metered instead, by **distinct pages per month per
organization**.

- A page counts **once** a month however many times it is re-run, with different searches,
  different assistants or a different query set.
- **Re-scoring a rewrite never counts at all**, so once you are in the loop on a page, the
  loop is free.
- **Re-weighting never counts.**
- Over the allowance the tool returns a 402-style error carrying `usage` (`used`, `cap`,
  `remaining`, `enforced`). Report those numbers and say whether to wait for the reset at the
  start of next month, re-score a page already counted this month, or upgrade.
- `get_citation_audit` returns the same `usage` on every poll, so you can warn the user
  before they hit the cap.
- The **free plan is ChatGPT only**. Asking for a platform outside the plan is refused with
  the list they can use, alongside `planType` and `scoreable`.

**Rewrites and briefs spend credits.** `revise_citation_audit`, `revise_content` and
`generate_citation_outline` draw on the plan's monthly rewrite allowance first, then
credits. When neither covers it, the refusal carries the allowance and the balance, so say
exactly what is needed rather than "it failed". Tell the user a rewrite will spend credits
before you call it.

## Performance and polling

An audit runs up to four pipelines **in parallel**, so it is not four times slower than a
single run. Most checks finish in seconds. The slow step is the relevance check, which calls
a third-party neural reranker on an external API (Replicate) that scales to zero when idle,
so the **first run of a quiet period can pay a cold start of a few minutes** (about three
has been observed). After that the model stays warm.

- **Set expectations up front.** Tell the user the first score of the day can take a few
  minutes to spin up, and that you will watch it.
- **Poll every 10 to 15 seconds.** While `progressPercent` rises or new entries appear in a
  platform's `checks`, it is working. A run sitting on the relevance step is **normal, not a
  failure or a hang**. Never report an error while `status` is still running.
- **Offer to poll in the background** and notify on completion rather than blocking the
  conversation, if the assistant supports long-running tasks.
- Stop polling when `status` is terminal: `completed` for audits, briefs and revisions,
  `completed` or `stopped` for single runs, `failed` for a revision.
- A single platform can fail while the audit completes. Its `errorMessage` and
  `haltedAtGate` say why, and it is dropped from the composite rather than scored as zero.

## Guardrails

- **The rewrite is meaning-preserving.** It front-loads, tightens and restructures what the
  page already says. It does **not** invent facts, figures, quotes, customers, dates or
  credentials. If a check asks for a number the page does not contain, the rewrite says to
  add it rather than making one up. New claims are the user's to add and substantiate.
- **Superlative hygiene**, especially legal, medical and financial. Matching a "best" or
  "top-rated" search does not mean asserting the page is the best. A superlative survives
  only if the original substantiated it. Do not override this.
- **Off-page findings are never rewritten.** They appear as context, clearly marked, and sit
  outside the rewrite's objective. No rewrite changes what Brave returns or whether an
  assistant names the brand in its own searches.
- **Any search is welcome, tracked or the user's own.** A tracked fan-out adds the
  confirmation that the platform was observed generating it. A keyword the user brings from
  another tool is equally valid and scored the same way. Do not gatekeep it or treat it as
  second-class; at most note, briefly and not as a warning, that a tracked fan-out would add
  that confirmation.
- **Never generate fan-outs and present them as observed.** Use the `provenance` field, and
  keep the label when you report results back.
- **Scores from different scoring versions are not comparable.** Each run records the version
  that produced it; do not compare across versions.
- Clearing every check does not guarantee a citation. It means the factors that can be seen
  and measured are in place. The answers themselves stay nondeterministic.

## A typical run

1. `list_properties` → the `propertyId` (from the wider `spyglasses` toolset).
2. `list_tracked_fanouts(propertyId)` → build the query set: one `primary` plus two to five
   related fan-outs, each with `provenance: "tracked"` and its `groundingSearchId`. Note
   `scoreablePlatforms`. Or take the searches the user brings, labelled `freetext`.
3. `match_pages_for_fanout(propertyId, primaryQuery)` → the closest page's `propertyPageId`.
4. `score_citation_audit({ propertyId, queries, propertyPageId, pageType })` → `auditId`.
   Warn about the cold start, then poll `get_citation_audit` every 10 to 15 seconds.
5. Read the composite **with its range**, the per-platform scores, the grouped
   recommendations and `readiness`. Always report the range, and any dropped platform.
6. If `revise_again`: say it will spend credits, call `revise_citation_audit(auditId)`, poll
   `get_revision`, show the change log, then `rescore_revision(revisionId)` and poll
   `get_citation_audit` on the returned `auditId`.
7. Repeat step 6 until `publish_ready`, `plateaued` or `regressed`, or the 2 to 3 pass cap.
   Then present the final markdown, meta and JSON-LD from the best version and tell the user
   to publish it.

### Page-first variant

1. `list_property_pages(propertyId, search)` → the `propertyPageId`.
2. `list_tracked_fanouts(propertyId)` → the searches this page should target, or take the
   keywords the user names. A query set is required; every check is relative to one.
3. `score_citation_audit({ propertyId, queries, propertyPageId })`, then the normal loop.

### Placement-first variant

1. `list_placements(propertyId)` → a placement with `hasContent: true`.
2. `get_placement(placementId)` → its `content`.
3. Pick the searches (`list_tracked_fanouts`, or the user's own).
4. `score_citation_audit({ propertyId, queries, draftMarkdown: <content>, pageType: "press_release", metaTitle: <title?> })`,
   then the normal loop. A missing meta title or description is fine.

### Keyword-first variant

1. `generate_citation_outline({ propertyId, keyword, pageType })` → `outlineId` (spends credits).
2. Poll `get_citation_outline(outlineId)` and present the brief with its provenance labels.
3. The user writes the draft.
4. `score_citation_audit({ propertyId, queries: <the brief's searches>, draftMarkdown, pageType, outlineId })`,
   then the normal loop.
