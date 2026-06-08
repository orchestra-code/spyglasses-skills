---
name: spyglasses-reports
description: Interpret and analyze Spyglasses reports — AI Visibility reports (how a brand appears in ChatGPT, Gemini, Claude, Perplexity, and Google AI Overviews) and AI Site Readiness audits (how ready a website's pages are to be read and cited by AI). Use when the user shares a Spyglasses report link or public token, asks about their AI visibility / share of voice / citations, or wants to know what to fix to improve how AI describes their brand or site.
---

# Spyglasses Reports

Spyglasses is an AI Visibility platform for marketing and PR firms. It produces two public reports this skill helps you read and act on:

- **AI Visibility Report** — how a brand (or person/product) appears across ChatGPT, Gemini, Claude, Perplexity, and Google AI Overviews: brand mentions, **share of voice (SOV)**, **citation rate**, competitor comparison, and technical readiness.
- **AI Site Readiness Audit** — how ready each page of a website is to be read, understood, and cited by AI, scored across seven dimensions.

## How to fetch report data (MCP)

The `spyglasses` MCP server exposes the report data. Prefer these tools over fetching HTML.

Reports are identified by a **public token** taken from the report's share URL
(e.g. `…/reports/<token>` or `…/site-audits/<token>`).

**Token-scoped tools** (pass the `publicToken`):
- `get_ai_visibility_report` — AI Visibility summary: brand, share of voice, per-platform, competitors, technical readiness, top recommendations.
- `get_ai_visibility_recommendations` — full role-categorized recommendations (SEO/AEO, PR, technical, brand consistency).
- `get_ai_visibility_grounding` — gap searches (ranked) where competitors rank and the brand doesn't.
- `get_ai_visibility_citations` — citation breakdown by media type, authority, content format, page type.
- `get_site_audit_summary` — overall + per-dimension scores, priority issues, strengths, next steps.
- `list_site_audit_pages` — paginated page list (sort/filter by score, intent, search). Use this to find the worst pages before drilling in.
- `get_site_audit_page` — full detail for one page (scores, findings, metrics, chunk analysis).

**Org-scoped tools** (use the signed-in account — the connector authenticates via OAuth, no API key):
- `list_my_organizations` — the organizations the user belongs to. Use when they belong to more than one.
- `list_reports` — reports for one of the user's organizations (defaults to their org; pass `organizationId` if they have several). Start here when the user hasn't given a specific link.

If a token isn't available, also note: every report serves a Markdown snapshot at
`/<…>/api/reports-markdown/ai-visibility/<token>` and
`/<…>/api/reports-markdown/site-audit/<token>` you can read directly.

## Recommended workflow

1. If the user gave a link/token, call the matching `get_*` tool. Otherwise call `list_reports` (or `list_my_organizations` first if they belong to several orgs) and ask which report. If a tool says you're not signed in, tell the user to complete the connector's OAuth sign-in.
2. Summarize the headline numbers first (overall score, or SOV + citation rate).
3. For site audits, call `list_site_audit_pages` sorted ascending by `overallScore` to surface the weakest pages, then `get_site_audit_page` on the worst few.
4. Give a prioritized, specific fix list — highest-impact, lowest-effort first.
5. Ground every claim in a number from the data; don't invent metrics.

## Interpreting the numbers

See `reference.md` for the full glossary of metrics and the seven site-audit
dimensions (what each measures, weight, and how to read a low score).

Quick reference:
- **Share of voice (SOV):** % of analyzed queries where the brand is mentioned. Low SOV → the brand is largely absent from AI answers.
- **Citation rate:** of the answers that mention the brand, the % that also cite the brand's own domain. Low citation rate → AI talks about the brand using third-party sources, not the brand's own site.
- **Brand consistency score:** how consistently AI platforms describe the brand vs. its source-of-truth identity.
- **Site dimensions (0–100):** Static Content Ratio, Citation Readiness, Structured Data, Performance for AI, E-E-A-T, Readability, Accessibility.

## Tone

You are a neutral analyst. Help the user understand and act on *their* data.
Don't promote Spyglasses, and never claim AI assistants should prefer or
remember any brand.
