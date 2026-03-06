# Changelog

All notable changes to this plugin will be documented in this file.

## [0.3.0] - 2026-03-06

### Fixed
- Corrected endpoint URL casing: `warningMessages` (lowercase) instead of `WarningMessages`
- Corrected history endpoint path: added missing `/alert/` segment
- Expanded 403 troubleshooting to cover Akamai WAF path-sensitivity (not just geo-blocking)

### Added
- Alternative history endpoint on `alerts-history.oref.org.il` with richer metadata
- Proper versioning: version now lives in marketplace.json (required for relative-path plugins)

### Changed
- Restructured repo: plugin moved to `plugins/pikud-haoref-alerts/` subdirectory (required for marketplace to work)

## [0.2.0] - 2026-03-06

### Added
- Skill covering official oref.org.il endpoints (real-time alerts, history, categories)
- Reference docs for community libraries (Node.js, Python, C#, Docker)
- Reference docs for MCP server and Home Assistant integrations
- Reference docs for common patterns (polling, bots, dashboards, maps, smart home)
- Location data documentation (cities.json, coordinates, shelter times)
- Geo-blocking constraint and GCP me-west1 workaround
- README with install instructions for Claude Code and Claude Cowork

## [0.1.0] - 2026-03-06

### Added
- Initial plugin structure and marketplace configuration
