# Changelog

All notable changes to the Spyglasses Claude plugin are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

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
