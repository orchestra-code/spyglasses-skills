# Changelog

All notable changes to the Spyglasses Claude plugin are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## [0.5.0]

### Added
- **The audit loop** for the citation optimizer. Scoring is no longer one query on
  one assistant: an audit scores one page against a whole **query set** on every
  assistant the plan covers (ChatGPT, Google AI Overviews, Google AI Mode, Claude)
  and harmonizes four sets of findings into one score and one list.
  - `score_citation_audit`, which scores a page, URL, or draft against a query set on
    several assistants (fire-and-poll, returns an `auditId`). Each query carries a
    `provenance` label (`tracked` / `freetext` / `generated_unverified`) that
    survives into the score, the rewrite and every screen.
  - `get_citation_audit`, the poll target. It returns the combined score **with the range it
    is measured within**, each assistant's own score and checks, the merged
    recommendation list (`consensus` / `platformSpecific` / `conflict`, conflicts
    carrying a `resolution.allocations`), the readiness verdict, and the monthly
    page `usage`.
  - `revise_citation_audit`, a rewrite grounded in every assistant's findings at
    once, conflicts resolved by allocation rather than by averaging. Spends credits.
  - `reweight_citation_audit`, which changes how much each assistant counts toward the
    combined score. Free, instant, uses no page, and the readiness verdict
    recomputes with the weights.
  - `regressed`, a fifth readiness verdict. A rewrite that made any single
    assistant meaningfully worse is held back and the previous version offered,
    even when the combined score held up.
- **Keyword briefs**, the step before the page exists.
  - `generate_citation_outline`, which turns a keyword and a page type into a
    section-by-section brief with per-section target searches, word budgets,
    must-include terms and quotable-passage rules. Spends credits.
  - `get_citation_outline`, the poll target, including the searches the brief was
    built for with their provenance labels and what none of the ranking pages answers.

### Changed
- `citation-optimizer` SKILL.md rewritten around the audit loop: the query set and
  its provenance rules, the grouped recommendations, the five STOP verdicts,
  re-weighting, the outline flow, and how to read a score that comes with a range.
- `rescore_revision` now follows whatever the revision came from and returns a
  `kind` of `"audit"` or `"run"`; read it to know which poll target to use.
- The single-run loop (`score_citation_pipeline`, `get_pipeline_run`,
  `revise_content`) is documented as the legacy shape and the one the free plan
  gets. It still works unchanged.
- Metering is documented: scoring costs no credits but counts distinct **pages** per
  month per organization. A page counts once however many times it is re-run,
  re-scoring a rewrite and re-weighting never count, and over the cap the tools
  refuse with `usage` (`used`, `cap`, `remaining`, `enforced`). Rewrites and briefs
  spend credits from a shared monthly allowance.
- Performance note updated: an audit runs the pipelines in parallel, and the
  reranker cold start on Replicate can still take a few minutes on the first run of
  a quiet period. Poll every 10 to 15 seconds and never treat a run sitting on the
  relevance check as failed.
- Plugin description now names the four assistants and the audit loop.

## [0.4.0]

### Added
- **Context-first entry points for the citation optimizer** — new MCP tools so an
  assistant can start the score → revise → re-score loop from a specific page or a
  PR placement, not just from a fan-out query:
  - `list_property_pages` — list/search a property's pages (id, path, title, tags)
    to resolve a page the user names ("optimize my pricing page") to a
    `propertyPageId`. The **page-first** path.
  - `list_placements` — list a property's PR placements (with a `hasContent` flag).
  - `get_placement` — read a placement's content + title/url to score it as a
    `draftMarkdown` (use `pageType: "press_release"`). The **placement-first** path
    ("score and revise this placement I'm working on").

### Changed
- `citation-optimizer` SKILL.md documents the three entry points (fan-out-first,
  page-first, placement-first), the new tools, and that placements score fine
  without a meta title/description.

## [0.3.0]

### Added
- **`citation-optimizer` skill** + MCP tools for the AI Citation Optimizer's
  score → revise → re-score authoring loop (ChatGPT pipeline):
  - `match_pages_for_fanout` — rank a property's pages against a query by content.
  - `score_citation_pipeline` — score a page/URL/draft (fire-and-poll, returns a runId).
  - `get_pipeline_run` — status, gate recommendations, and a **readiness verdict**
    (`publish_ready` / `revise_again` / `plateaued`) with an explicit STOP signal so
    an assistant terminates the loop instead of revising indefinitely.
  - `revise_content` — generate a meaning-preserving, template-grounded revision
    (fire-and-poll, returns a revisionId).
  - `get_revision` — the revised markdown + meta + JSON-LD + a traceable change log.
  - `rescore_revision` — re-score the revision as a new draft run to confirm the gain.

## [0.2.1]

### Fixed
- `spyglasses-reports` SKILL.md frontmatter: the `description` contained an
  unquoted `": "` (colon-space), which YAML parses as a mapping value and
  rejected the whole block. Rephrased so the description parses cleanly — fixes
  the plugin frontmatter error and lets skill loaders (e.g. `npx skills add`)
  read the skill's name and description.

## [0.2.0]

### Added
- **Account data tools** (read-only, scoped to the signed-in user's properties):
  - `list_properties` — discover the brands/sites on the account.
  - `list_projects`, `get_project_insights` — projects and their deep insights
    (metric deltas, weekly trends, goals, annotations).
  - `get_metrics_history`, `get_consistency_history` — AI-visibility and
    brand-consistency trend lines (with an optional `projectId` filter on the
    grounding-search trend).
  - `get_message_tracking` — key-message pull-through over time.
  - `get_answer_summaries` — per-query, week-bucketed full answer text for
    message-drift analysis.
  - `get_citation_intelligence` — citation mix over time, breakdowns, and top
    sources by owner.
- **Scoring tools**:
  - `score_publisher_value` — AI Placement Value Score (AIPVS) for one or more
    publishers, in a brand's context or in general (read-only; never enriches).
  - `score_placement_quality` — AI Placement Quality Score (PQS) for a
    prospective placement, optionally combined with AIPVS into a total
    placement value.
- **Prompts**: `analyze_project`, `track_message_drift`, `citation_mix_trends`,
  `evaluate_publishers`.
- `reference.md` glossary sections for projects/metrics, message tracking &
  drift, AIPVS, PQS, and the citation taxonomies; SKILL.md guidance on the
  property-scoped workflow and building visualizations client-side from the
  returned series.

## [0.1.0]

### Added
- `spyglasses-reports` skill — interpret AI Visibility reports and AI Site
  Readiness audits.
- `spyglasses` MCP connector (`.mcp.json`) pointing at
  `https://www.spyglasses.io/api/mcp`, with tools:
  - `get_ai_visibility_report`, `get_site_audit_summary`,
    `list_site_audit_pages`, `get_site_audit_page` (token-scoped)
  - `list_reports` (org-scoped via `x-api-key`)
- Plugin marketplace manifest so the plugin installs via
  `/plugin marketplace add orchestra-code/spyglasses-skills`.
