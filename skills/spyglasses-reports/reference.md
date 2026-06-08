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
