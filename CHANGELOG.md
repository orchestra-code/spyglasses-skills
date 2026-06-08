# Changelog

All notable changes to the Spyglasses Claude plugin are documented here.
This project adheres to [Semantic Versioning](https://semver.org/).

## [0.1.0] - Unreleased

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
