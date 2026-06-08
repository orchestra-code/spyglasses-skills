# spyglasses-skills

A set of Claude skills (and an MCP connector) for interacting with the
[Spyglasses](https://www.spyglasses.io) AI Visibility platform.

This repo is a **Claude Code plugin marketplace**. The `spyglasses` plugin
bundles:

- **Skills** — domain knowledge for interpreting Spyglasses reports (under `skills/`).
- **MCP connector** — the `spyglasses` MCP server (`.mcp.json`) that fetches your
  report data, so one install wires up both the know-how and the data access.

## Skills

| Skill | What it does |
| --- | --- |
| `spyglasses-reports` | Read and act on AI Visibility reports and AI Site Readiness audits — share of voice, citations, per-dimension scores, and prioritized fixes. |

More skills (AI Placement Value Score, analytics, Discovery queries) will be
added here over time; they share the same MCP connector.

## Install (Claude Code)

```
/plugin marketplace add orchestra-code/spyglasses-skills
/plugin install spyglasses@spyglasses-skills
```

This installs the skill(s) and the `spyglasses` MCP connector.

### Authentication

The MCP connector talks to `https://www.spyglasses.io/api/mcp` and authenticates
with **OAuth** — when you add it, your assistant opens a browser to sign in with
your Spyglasses account (one time). No API key to find or paste.

Once connected:

- **Token-scoped tools** work for any report by its public token (from the
  report's share URL).
- **Org-scoped tools** (`list_my_organizations`, `list_reports`) use your
  account's organization membership.

## Use with Claude.ai / ChatGPT / other assistants

Add `https://www.spyglasses.io/api/mcp` as a custom MCP connector in your
assistant's settings and complete the sign-in (OAuth) prompt. Then load /
reference the `spyglasses-reports` skill.

> A `?q=` "Chat with this report" deep-link button (in the Spyglasses report UI)
> opens a one-shot conversation using the report's Markdown snapshot. It does
> **not** auto-connect this MCP — connectors are added manually, once, here.

## MCP prompts (start here)

Prompts are the easiest entry point — pick one from your assistant's prompt menu
and give it a report's public token; it drives the right tools for you:

- `analyze_report` — summarize a report and surface key findings
- `what_to_fix_first` — a prioritized, role-based action plan (SEO/AEO, PR, technical)
- `competitor_gaps` — where competitors rank/are cited and the brand isn't
- `audit_worst_pages` — surface and explain a site audit's weakest pages
- `list_my_reports` — list your org's reports (uses your signed-in account)

## MCP tools

Token-scoped (pass a report `publicToken`): `get_ai_visibility_report`,
`get_ai_visibility_recommendations`, `get_ai_visibility_grounding`,
`get_ai_visibility_citations`, `get_site_audit_summary`,
`list_site_audit_pages`, `get_site_audit_page`.
Org-scoped (uses your signed-in account): `list_my_organizations`, `list_reports`.

## Local development & testing

Test the plugin without publishing it. The MCP `url` is environment-overridable
(`${SPYGLASSES_MCP_URL:-https://www.spyglasses.io/api/mcp}`).

```bash
# Point at a dev MCP endpoint (kept out of the repo). Use a publicly reachable
# HTTPS URL (e.g. an ngrok tunnel) — the OAuth browser redirect must reach it.
export SPYGLASSES_MCP_URL=https://<your-dev-host>/api/mcp

# Fastest loop — load the plugin from this directory (no install/marketplace).
# Edit files, then run /reload-plugins inside Claude Code to pick up changes.
claude --plugin-dir .

# Inside Claude Code:
#   /mcp                         → connect (completes the OAuth sign-in once)
#   /spyglasses:spyglasses-reports
```

To exercise the full marketplace + install flow against a local checkout:

```bash
# Inside Claude Code (absolute path works):
/plugin marketplace add /absolute/path/to/spyglasses-skills
/plugin install spyglasses@spyglasses-skills
```

See `.env.example` for the variables. Never commit a real dev URL.

## Releasing

```bash
claude plugin validate .
```

Bump `version` in `.claude-plugin/plugin.json` (semver) and record changes in
`CHANGELOG.md` for every release. To submit to the community directory, see
https://clau.de/plugin-directory-submission.
