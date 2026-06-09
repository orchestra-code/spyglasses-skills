# Spyglasses metrics reference

## AI Visibility Report

- **Queries analyzed** — the set of representative questions Spyglasses ran across the AI platforms for this subject.
- **Brand mentions** — count of analyzed answers that mention the brand.
- **Share of voice (SOV)** — % of analyzed queries where the brand is mentioned. The single best "are we present in AI answers?" number. Reported overall and per platform.
- **Citations** / **citation rate** — citations is the count of answers that link the brand's own domain; citation rate is citations ÷ mentions. High SOV but low citation rate means AI describes the brand using *other people's* pages — a content/authority gap on the brand's own site.
- **Per-platform breakdown** — mentions / SOV / citations for ChatGPT, Gemini, Claude, Perplexity, and Google AI Overview. Big gaps between platforms point to platform-specific issues (e.g. strong on ChatGPT, absent on Perplexity).
- **Brand consistency score (0–100)** — how consistently the platforms describe the brand against its source-of-truth identity (tagline, category, ICP, features). Low = AI is telling an inconsistent or outdated story.
- **Competitor share of voice** — SOV/mentions/citations for competitors, with **gap queries**: questions where a competitor appears and the brand does not. Gap queries are the most actionable content targets.
- **Technical readiness** — overall + structure/accessibility/readability/performance sub-scores for the brand's site (a lighter-weight cousin of the full site audit).
- **Recommendations & gap opportunities** — prioritized openings to improve presence.

## AI Site Readiness Audit

Overall score is a weighted average of seven dimensions (0–100 each). Pages are
scored individually; the summary aggregates them and flags priority issues.

| Dimension | Weight | What it measures | A low score means |
| --- | --- | --- | --- |
| Static Content Ratio | 22% | How much content is visible without JavaScript | AI crawlers see an incomplete page; move key content into server-rendered HTML |
| Citation Readiness | 20% | How well content is structured for AI to cite (clear headings, quotable, self-contained passages) | AI is unlikely to select the page as a source; add clear headings and standalone, quotable statements |
| Structured Data | 18% | Whether the right JSON-LD schemas are present and complete | AI lacks rich context; add/complete schema.org markup |
| Performance for AI | 14% | How fast and efficiently content is delivered to AI crawlers (TTFB, HTML size) | crawlers may skip or truncate; reduce TTFB and HTML payload |
| E-E-A-T Signals | 14% | Trust and authority signals at brand and page level | add author attribution, organizational context, credibility indicators |
| Readability | 10% | Whether content is written at an appropriate reading level | simplify overly complex prose so AI can summarize cleanly |
| Accessibility | 10% | Whether the page meets core accessibility standards | fix semantic HTML / alt text / heading hierarchy (also helps AI parse structure) |

### Per-page detail (`get_site_audit_page`)
- **Findings** — specific issues with severity (`high`/`medium`/`low`), status (`red`/`yellow`/`green`), and a recommendation.
- **Technical metrics** — performance (TTFB, HTML size), token counts, content stats.
- **Chunk / citation-readiness analysis** — how the page splits into citable chunks: chunk quality score, the best self-contained chunk, **structural deserts** (long stretches with no headings/structure), and 128-token window alignment. Structural deserts are a common, high-impact fix.

### Status bands
Scores map to red / yellow / green bands. Treat **red** items as priority fixes,
**yellow** as improvements, **green** as strengths to preserve.

## Projects & historical metrics

- **Property** — a brand/site the user tracks (the unit of account-scoped data). Get a `propertyId` from `list_properties`.
- **Project** — a bundled, time-bound tracking effort over a set of tracked queries ("prompts") with **goals** (page-group or publisher citations). `get_project_insights` returns:
  - **Metric deltas** — start vs. current for consistency, SOV, mentions, citations, citation rate (each with a change and direction).
  - **Weekly trend** — a per-week series of consistency / SOV / mentions / citations for charting.
  - **Goals** — type (`PAGE_GROUP` / `PUBLISHER`), status (pending / hit / missed), and first-hit details (date, citation, platform).
  - **Annotations** — a timeline of events (manual or auto-generated when a goal is first hit).
- **`get_metrics_history`** — one trend point per report / nightly run: overall SOV, mentions, citations, citation rate, plus a per-platform breakdown. With a `projectId`, also returns grounding-search ranking trends scoped to that project's queries (brand vs. best-competitor SERP position, gap flag).
- **`get_consistency_history`** — overall brand-consistency score over time, plus per-platform scores.

## Message tracking & drift

- **Key message** — a tracked phrase/claim with an intended **sentiment**. `get_message_tracking` returns, per message, the **mention rate** (% of scanned AI executions that pulled the message through) over time, plus first/latest values and the change in percentage points.
- **Answer summary** — the LLM's main answer text for one tracked query on one platform in one run. `get_answer_summaries` returns the **full** text for one query, bucketed by week (one representative answer per week per platform). Compare weeks to detect **message drift** — how the wording, claims, or framing AI uses for the brand changes over time. (The MCP returns raw text; you do the comparison and summary.)

## AI Placement Value Score (AIPVS) — `score_publisher_value`

A composite **0–100** score of how valuable a citation from a publisher domain is for AI visibility. Three layers (weights):

- **OAB — Organic Authority Baseline (40%)** — the domain's search authority (PageRank percentile, estimated traffic value, top-position quality, commercial value, category relevance to the brand).
- **ACA — AI Citation Accessibility (35%)** — how accessible the domain is to AI citation crawlers (citation-bot market share it allows vs. blocks).
- **ATI — AI Training Influence (25%)** — the domain's influence on AI training data, with brand-domain uplift when scored in a brand's context.

Outputs: composite **AIPVS** (0–100), **tier** + label (**Premium / Strong / Moderate / Limited**), **AI impact multiplier** (~0.40–1.00, the citation/training multiplier applied to organic authority), **confidence** (`exact` | `estimated`) with a reason, and per-layer detail. `needsEnrichment: true` means the publisher's organic metrics haven't been fetched yet, so the score is an estimate. Scored **in general** (publisher only) or **in a brand's context** (pass `propertyId` → category relevance + ATI uplift).

## AI Placement Quality Score (PQS) — `score_placement_quality`

A **0–100** score of how much citation value a *placement* earns, independent of the publisher. Weighted sub-scores (prospective default weights): **position 30%**, **chunk 20%**, **type 20%**, **editorial 15%**, **link 15%**; then multiplied by a **sentiment multiplier**.

- **Position** — where on the page (higher = better): top <10% → 1.0 down to ≥90% → 0.20.
- **Type** — `dedicated_article` 1.0, `roundup_top` 0.85, roundup top/middle/bottom third 0.75/0.60/0.45, `passing_mention` 0.40, `sidebar_widget_biobox` 0.30, `quote_only` 0.25.
- **Editorial control** — `full_authorship` 1.0, `collaborative` 0.85, `quoted` 0.70, `mentioned` 0.50, `unpredictable` 0.35.
- **Link attribute** — `dofollow` 1.0, `nofollow` 0.6, `sponsored` 0.5, `ugc` 0.4, `no_link` 0.3.
- **Sentiment multiplier** — strongly negative 0.0–0.2, mildly negative 0.5, neutral 1.0, mildly positive 1.15, strongly positive 1.3. (Prospective scoring assumes neutral chunk 0.7 and neutral sentiment 1.0.)

**Total placement value** = AIPVS × PQS ÷ 100 — the combined worth of placing on a specific publisher in a specific way.

## Citation taxonomies (`get_citation_intelligence`)

Each cited source is classified along several axes (counts roll up into the breakdowns):

- **Media type** — `owned`, `earned_press`, `earned_mention`, `review_platform`, `social_community`, `reference`, `video_platform`, `third_party`, `unclassified`.
- **Authority type** — `academic`, `government`, `major_media`, `trade_media`, `analyst`, `professional_org`, `review_aggregator`, `encyclopedia`, `company`, `unclassified`.
- **Content format** — `listicle`, `how_to_guide`, `review`, `research_data`, `case_study`, `video`, `podcast`, `news`, `reference`, `standard`, `unclassified`.
- **Owner type** (of a citation source domain) — `brand` (the property's own domain), `competitor` (a tracked competitor), or `third_party`.

`historicalData` is the trend series — one point per report/day with total/owned/listicle counts and per-axis breakdowns — use it to chart how the citation mix shifts over time. A high **listicle rate** or low **share of owned** signals AI leans on third-party roundups rather than the brand's own pages.
