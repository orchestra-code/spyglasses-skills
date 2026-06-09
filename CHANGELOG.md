# Changelog

All notable changes to the Spyglasses Claude plugin are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

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
