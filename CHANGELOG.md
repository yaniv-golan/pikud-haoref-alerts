# Changelog

All notable changes to this plugin will be documented in this file.

## [0.5.0] - 2026-03-07

Learnings from Cowork research session analyzing the official and third-party APIs.

### Added
- Documented 3,000-record hard cap on both official history endpoints (AlertsHistory.json and GetAlarmsHistory.aspx)
- New reference: `references/alternative-data-sources.md` covering Tzofar (tzevaadom.co.il) API
  - Tzofar endpoints: alerts-history, single alert lookup, static historical archive (20K+ records)
  - Tzofar threat type mapping table (differs from oref categories)
  - Oref category to Tzofar threat mapping table
  - Tzofar's cities.json and polygons.json data sources
  - Critical caveat: Tzofar excludes pre-alerts (cat 14) and event-concluded (cat 13)
- Historical data strategy decision tree (which source to use for each time horizon)
- Community archive projects (hasadna, Meir017, Kaggle dataset, idodov/RedAlert)
- Gotcha: `mode` parameter on GetAlarmsHistory.aspx does not provide pagination

### Fixed
- AlertsHistory.json description: was "typically last 24 hours", now accurately describes 3,000-record cap
- Third-party data sources section in common-patterns.md expanded with Tzofar details and links

## [0.4.0] - 2026-03-06

Issues discovered during live testing with Claude Cowork during an actual rocket alert.

### Added
- Documented category 13 "event concluded" (האירוע הסתיים) as the authoritative signal an alert is over
- Documented category 14 pre-alert (בדקות הקרובות צפויות להתקבל התרעות) as an early warning of incoming barrage
- Added "Determining if a situation is still active" section with dual-check pattern and code example
- Documented date format differences between the two history endpoints

### Fixed
- Example 2 ("Is there an alert in Ashkelon right now?") now uses dual-check pattern instead of relying solely on alerts.json
- Clarified that alerts.json is an ephemeral snapshot (seconds to a minute), not a status indicator
- Added Hebrew titles for categories 13 and 14 in the categories table

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
