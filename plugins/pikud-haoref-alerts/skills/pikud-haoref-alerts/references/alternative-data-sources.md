# Alternative Data Sources

## Tzofar (tzevaadom.co.il)

Tzofar is a popular third-party alert relay service. Unlike the official oref.org.il API, Tzofar endpoints are **not geo-blocked** — they work from any IP worldwide.

### Endpoints

**Recent alert groups (last 50):**
```
GET https://api.tzevaadom.co.il/alerts-history
```

Returns the last 50 alert groups. No pagination support found.

**Single alert group by ID:**
```
GET https://api.tzevaadom.co.il/alerts-history/id/{id}
```

**Full historical archive:**
```
GET https://api.tzevaadom.co.il/static/historical/all.json
```

A ~1.6MB static JSON file containing 20,000+ alert records going back to May 2021. This is the only publicly accessible source for deep historical data without running your own poller. No geo-blocking.

### Data model

Tzofar uses a compact array format, different from oref's object format:

```json
[id, threat, [cities], unix_timestamp]
```

Example:
```json
[123456, 0, ["תל אביב - מרכז העיר", "רמת גן - מערב"], 1709830200]
```

### Threat type mapping

Tzofar uses its own numeric threat types (different from oref categories):

| Threat | Type | Color |
|--------|------|-------|
| 0 | Red Alert (Rockets/Missiles) | #FF0000 |
| 1 | Hazardous Materials | #9335ee |
| 2 | Terrorist Infiltration | #FFD500 |
| 3 | Earthquake | #00FF55 |
| 4 | Tsunami | #0080FF |
| 5 | Hostile Aircraft (UAV) | #FF8000 |
| 6 | Non-conventional Missile | #ee35a8 |
| 7 | Radiological | #ee35a8 |
| 8 | General Alert | #FF0000 |

### Critical differences from oref

- **No pre-alerts** — Tzofar does NOT include oref category 14 (pre-alert / incoming alerts warning)
- **No event-concluded messages** — Tzofar does NOT include oref category 13 (event concluded)
- **Only actual threat alerts** — You cannot distinguish pre-alerts from alerts or determine when a situation has ended using Tzofar data alone

### Oref category to Tzofar threat mapping

| Oref Category | Oref Name | Tzofar Threat | Notes |
|--------------|-----------|---------------|-------|
| 1 | Missiles/Rockets | 0 | Core alert |
| 2 | Hostile aircraft | 5 | Numbering differs |
| 3 | Earthquake | 3 | Same |
| 4 | Tsunami | 4 | Same |
| 5 | Radiological | 7 | Numbering differs |
| 6 | Hazardous materials | 1 | Numbering differs |
| 7 | Terrorist infiltration | 2 | Numbering differs |
| 13 | Event concluded | — | Not in Tzofar |
| 14 | Pre-alert | — | Not in Tzofar |

### Tzofar location and polygon data

Tzofar hosts its own city and polygon datasets:
- `https://api.tzevaadom.co.il/static/cities.json`
- `https://api.tzevaadom.co.il/static/polygons.json`

Version numbers are tracked via `https://api.tzevaadom.co.il/lists-versions`. These can be useful alternatives to the eladnava/pikud-haoref-api cities.json.

---

## Community archive projects

These projects solve the historical data problem by continuously polling and archiving alerts:

- **[hasadna/oref-alarms-history](https://github.com/hasadna/oref-alarms-history)** — Continuous scraper by The Public Knowledge Workshop (Israel's civic data org)
- **[Meir017/oref-data](https://github.com/Meir017/oref-data)** — Git-based alert aggregator
- **[Kaggle dataset](https://www.kaggle.com/datasets/sab30226/rocket-alerts-in-israel-made-by-tzeva-adom)** — Downloadable dataset of historical alerts
- **[idodov/RedAlert](https://github.com/idodov/RedAlert)** — Home Assistant integration with file-based archiving
