# Alternative Data Sources

## Tzofar (tzevaadom.co.il)

Tzofar is a popular third-party alert relay service. Unlike the official oref.org.il API, Tzofar endpoints are **not geo-blocked** — they work from any IP worldwide.

### Endpoints

**Required headers:** Tzofar blocks Python's default `User-Agent: Python-urllib/X.Y` with HTTP 403. Set a browser-like User-Agent:

```python
# Works
requests.get(url)  # requests uses its own UA, not blocked
urllib.request.Request(url, headers={"User-Agent": "Mozilla/5.0"})

# Fails with 403
urllib.request.urlopen(url)  # sends Python-urllib/3.x
```

**Recent alert groups (last 50):**
```
GET https://api.tzevaadom.co.il/alerts-history
```

Returns the last 50 alert groups. No pagination support found.

**Single alert group by ID:**
```
GET https://api.tzevaadom.co.il/alerts-history/id/{id}
```

**Historical data via ID iteration:**

There is no bulk download endpoint. To build historical data, iterate alert group IDs backwards from the latest known ID:

```
GET https://api.tzevaadom.co.il/alerts-history/id/5913
GET https://api.tzevaadom.co.il/alerts-history/id/5912
GET https://api.tzevaadom.co.il/alerts-history/id/5911
...
```

**Rate limiting:** Tzofar rate-limits burst traffic — rapid-fire requests with no delay trigger HTTP 429 after ~13 requests. Adding short delays (0.3–0.5 seconds between requests) avoids 429s entirely and allows fetching hundreds of groups in a few minutes.

The `/alerts-history` endpoint returns the most recent ~50 groups — use the lowest `id` from that response as your starting point for backward iteration. **IDs are mostly sequential but gaps exist** — e.g., IDs 5599–5663 all return 404 while surrounding IDs work. When iterating, skip 404s and stop after a tolerance limit (e.g., 100 consecutive 404s).

### Data model

Tzofar groups alerts into "alert groups." Each group contains one or more sub-alerts (typically one per threat wave).

**`/alerts-history` response** (array of alert groups):
```json
[
  {
    "id": 5913,
    "description": null,
    "alerts": [
      {
        "time": 1772857423,
        "cities": ["תל אביב - מרכז העיר", "רמת גן - מערב"],
        "threat": 0,
        "isDrill": false
      }
    ]
  }
]
```

**`/alerts-history/id/{id}` response** (single alert group, same structure as one array element above).

**Key fields:**
- `id` — Sequential alert group ID. Can be iterated backwards for historical data.
- `alerts[].time` — Unix timestamp of the alert.
- `alerts[].cities` — Array of affected location names in Hebrew.
- `alerts[].threat` — Threat type number (see mapping table below).
- `alerts[].isDrill` — Boolean indicating if this is a drill.
- `description` — Usually null; occasionally contains a text description.

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

Tzofar tracks city and polygon data versions via `https://api.tzevaadom.co.il/lists-versions` (returns e.g. `{"cities": 10, "polygons": 5}`). However, the static download URLs (`/static/cities.json`, `/static/polygons.json`) are no longer available (404 as of March 2026). Use the eladnava/pikud-haoref-api `cities.json` instead.

---

## Community archive projects

These projects solve the historical data problem by continuously polling and archiving alerts:

- **[hasadna/oref-alarms-history](https://github.com/hasadna/oref-alarms-history)** — Scraper scripts by The Public Knowledge Workshop (Israel's civic data org). No downloadable dataset — you run the scraper yourself to collect data.
- **[Meir017/oref-data](https://github.com/Meir017/oref-data)** — Git-based alert aggregator with a `data.json` file (~390KB) containing aggregated alerts.
- **[Kaggle dataset](https://www.kaggle.com/datasets/sab30226/rocket-alerts-in-israel-made-by-tzeva-adom)** — Downloadable dataset of historical alerts
- **[idodov/RedAlert](https://github.com/idodov/RedAlert)** — Home Assistant integration with file-based archiving
