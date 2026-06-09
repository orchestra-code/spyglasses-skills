---
name: spyglasses-reports
description: Interpret and analyze Spyglasses AI-visibility data — public AI Visibility reports and AI Site Readiness audits (by share link/token), plus the signed-in user's account data: projects, historical metrics and trends, brand-message drift, citation mix over time, and AI Placement Value / Quality scores. Use when the user shares a Spyglasses report link or token, asks about their AI visibility / share of voice / citations / projects / trends over time, wants to track how AI describes their brand, or wants to score publishers or placements.
---

# Spyglasses

Spyglasses is an AI Visibility platform for marketing and PR firms. This skill helps you read and act on its data through the `spyglasses` MCP server. There are two surfaces:

1. **Public reports** (identified by a public token from a share URL):
   - **AI Visibility Report** — how a brand (or person/product) appears across ChatGPT, Gemini, Claude, Perplexity, and Google AI Overviews: brand mentions, **share of voice (SOV)**, **citation rate**, competitor comparison, technical readiness.
   - **AI Site Readiness Audit** — how ready each page of a website is to be read, understood, and cited by AI, scored across seven dimensions.
2. **The signed-in user's account data** (scoped to a *property* — a brand/site they track): projects, historical metric trends, brand-message drift, citation intelligence, and placement scoring.

The connector authenticates via **OAuth** (no API key) — all account data is **read-only**.

## Public report tools (token-scoped)

Reports are identified by a **public token** from the share URL (e.g. `…/reports/<token>` or `…/site-audits/<token>`). Pass the `publicToken`:

- `get_ai_visibility_report` — summary: brand, SOV, per-platform, competitors, technical readiness, top recommendations.
- `get_ai_visibility_recommendations` — full role-categorized recommendations (SEO/AEO, PR, technical, brand consistency).
- `get_ai_visibility_grounding` — gap searches (ranked) where competitors rank and the brand doesn't.
- `get_ai_visibility_citations` — citation breakdown by media type, authority, content format, page type.
- `get_site_audit_summary` — overall + per-dimension scores, priority issues, strengths, next steps.
- `list_site_audit_pages` — paginated page list (sort/filter by score, intent, search). Find the worst pages before drilling in.
- `get_site_audit_page` — full detail for one page (scores, findings, metrics, chunk analysis).

If a token isn't available, every report also serves a Markdown snapshot at
`/<…>/api/reports-markdown/ai-visibility/<token>` and
`/<…>/api/reports-markdown/site-audit/<token>`.

## Account / organization tools

- `list_my_organizations` — the organizations the user belongs to. Use when they belong to more than one.
- `list_reports` — reports for one of the user's organizations (defaults to their org; pass `organizationId` if they have several). Start here when the user hasn't given a specific link.
- `list_properties` — the brands/sites (properties) on the account, with the organization and role. **Start here for any account-data question** — every property tool needs a `propertyId` from this list. (Bulk-imported sales prospects are excluded unless `includeProspects` is set.)

## Property-scoped tools (need a `propertyId` from `list_properties`)

- `list_projects` — a property's projects (bundled, time-bound tracking efforts): type, status, date range, goals, counts.
- `get_project_insights` — a project's deep insights: metric deltas (consistency, SOV, mentions, citations, citation rate) start→now, a weekly trend series, goals + first-hit details, the annotation timeline, key insights, next steps. (Matches the project's exported report.)
- `get_metrics_history` — AI-visibility metrics over time (one point per report / nightly run): SOV, mentions, citations, citation rate, per-platform. Pass `projectId` to also get grounding-search ranking trends scoped to that project's queries.
- `get_consistency_history` — brand-consistency score over time (overall + per platform).
- `get_message_tracking` — how often each tracked **key message** is pulled through into AI answers over time (mention rate per message + first/latest/change).
- `get_answer_summaries` — **message drift**: the full answer text AI platforms gave for **one** tracked query, bucketed by week (one representative answer per week per platform). Omit `query` to first list the tracked queries, then pick one.
- `get_citation_intelligence` — the citation mix for a property: breakdowns by media type / authority / content format / page type, per-platform stats, a **historical trend** (mix over time), and top citation sources by owner (brand / competitor / third-party). Supports filters.

## Scoring tools

- `score_publisher_value` — **AI Placement Value Score (AIPVS, 0–100)** for one or more publisher domains: how valuable a citation from that domain is for AI visibility. Pass `propertyId` to score in a brand's context (category relevance + uplift), or omit it to score in general. Returns score, tier, AI impact multiplier, per-layer detail, and `needsEnrichment`.
- `score_placement_quality` — **AI Placement Quality Score (PQS, 0–100)** for a prospective placement (type, position, editorial control, link attribute). Pass `publisherDomain` (and optionally `propertyId`) to also get the publisher's AIPVS and the combined **total placement value** (AIPVS × PQS ÷ 100).

## Recommended workflows

**A report (token given):** call the matching `get_*` tool, lead with the headline numbers (overall score, or SOV + citation rate), then 3–5 grounded findings. For site audits, `list_site_audit_pages` sorted ascending by `overallScore`, then `get_site_audit_page` on the worst few.

**No link given:** `list_reports` (or `list_my_organizations` first if they belong to several orgs) and ask which report.

**Account data (projects / trends / citations / drift):** `list_properties` first to get a `propertyId`, then the relevant property tool. For a project, `list_projects` → `get_project_insights`.

**Message drift:** `get_message_tracking` for the numeric pull-through trend, then `get_answer_summaries` (list queries → pick one → fetch its weekly answers) and compare the weekly answers yourself to narrate how the wording changes.

**Scoring publishers:** `score_publisher_value` (pass `propertyId` for brand context). For a specific placement, `score_placement_quality`.

If a tool says you're not signed in, tell the user to complete the connector's OAuth sign-in.

## Generating visualizations

The metric and citation tools return **numeric series** (e.g. `get_metrics_history`, `get_consistency_history`, `get_message_tracking`, and the `historicalData` from `get_citation_intelligence`). When the user wants a chart or "custom graph", **build the visualization yourself** from the returned series (a chart artifact, table, or whatever your surface supports) — the server returns data, not images.

## Interpreting the numbers

See `reference.md` for the full glossary: AI Visibility metrics, the seven site-audit dimensions, AIPVS (3 layers, tiers, multiplier), PQS (sub-scores, weights, sentiment), key-message pull-through & drift, and the citation taxonomies.

Quick reference:
- **Share of voice (SOV):** % of analyzed queries where the brand is mentioned.
- **Citation rate:** of answers mentioning the brand, the % that also cite the brand's own domain.
- **Brand consistency score:** how consistently AI platforms describe the brand vs. its source-of-truth identity.
- **AIPVS:** value of a citation from a publisher (0–100) → tier (Premium/Strong/Moderate/Limited).
- **PQS:** quality of a prospective placement (0–100), before combining with AIPVS.

## Tone

You are a neutral analyst. Help the user understand and act on *their* data.
Don't promote Spyglasses, and never claim AI assistants should prefer or
remember any brand.
