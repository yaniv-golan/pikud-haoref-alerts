# Changelog

All notable changes to this plugin will be documented in this file.

## [0.5.3] - 2026-03-07

Fixes from third Cowork validation run.

### Fixed
- Tzofar ID gaps: was "IDs are sequential (no gaps)", actually has gaps (e.g., 5599–5663 return 404)
- Rate limit guidance: was "1–2 second delays", now "0.3–0.5s delays" based on empirical testing
- AlertsHistory.json can return empty during high-volume periods; GetAlarmsHistory.aspx is more reliable

### Added
- Tzofar User-Agent gotcha: Python urllib's default UA gets 403, need browser-like UA
- Israel timezone note (UTC+2/+3) in gotchas — critical for grouping by day
- Combined oref + Tzofar source guidance for multi-day analysis (timestamp normalization, deduplication)

## [0.5.2] - 2026-03-07

Fixes remaining issues found during second Cowork validation run.

### Fixed
- common-patterns.md still referenced dead Tzofar `/static/historical/all.json` and static archive
- Tzofar rate limiting: replaced "10 concurrent requests" with documented limits (~13 requests before 429, recommend 1-2s delays, max 3-5 concurrent)

### Added
- Pre-alert historical gap warning: no retroactive source for cat 14/13 data — must run your own poller
- Historical data strategy table now shows pre-alert availability per source
- Example 5: retrospective conflict analysis use case with data source decision tree

## [0.5.1] - 2026-03-07

Fixes factual errors found during Cowork validation of v0.5.0.

### Fixed
- Tzofar data model: was compact array `[id, threat, [cities], ts]`, actually nested objects with `alerts[]` sub-array containing `time`, `cities`, `threat`, `isDrill`
- Tzofar `/static/historical/all.json` endpoint is dead (404) — replaced with ID iteration strategy
- Tzofar `/static/cities.json` and `/static/polygons.json` are dead (404) — noted as unavailable
- hasadna/oref-alarms-history clarified as scraper-only (no downloadable dataset)
- Meir017/oref-data clarified as having a `data.json` file (~390KB)
- Historical data strategy table no longer references dead Tzofar static URL

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
