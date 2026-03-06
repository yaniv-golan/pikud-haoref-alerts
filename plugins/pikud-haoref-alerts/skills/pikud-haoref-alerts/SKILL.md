---
name: pikud-haoref-alerts
description: >
  Comprehensive guide for working with Pikud HaOref (Israel Home Front Command) alert APIs —
  the official and community endpoints that publish real-time rocket alerts, earthquake warnings,
  and other civil defense notifications across Israel. Use this skill whenever someone wants to
  build an integration with Pikud HaOref alerts, fetch live or historical alert data, set up
  monitoring or dashboards for Israeli emergency alerts, write code that consumes oref.org.il
  endpoints, deploy an alert service, or understand the available API landscape. Also trigger
  when someone mentions "red alert API", "tzeva adom", "oref alerts", "rocket alert Israel",
  "Home Front Command API", or any Hebrew references like "פיקוד העורף" or "צבע אדום".
  Even if the user just says "I want to get alerts from Israel" or "build something with
  Israeli civil defense data", this skill is the right starting point.
  Do NOT use for US weather alerts (NWS/FEMA), UK emergency alerts, generic webhook/push
  notification frameworks, or non-Israeli civil defense systems.
license: MIT
compatibility: >
  Code examples require network access. The official oref.org.il API geo-blocks non-Israeli IPs;
  deployment examples assume GCP me-west1 (Tel Aviv) or equivalent Israeli IP.
metadata:
  author: Yaniv Golan
  version: 0.2.0
  tags: [israel, alerts, civil-defense, pikud-haoref, tzeva-adom, rockets, emergency]
---

# Pikud HaOref Alert APIs

This skill covers everything you need to know to work with the Pikud HaOref (Israel Home Front Command) alert system — from raw official endpoints to community wrapper libraries and deployment strategies.

For detailed reference material, see the `references/` directory:
- `references/community-libraries.md` — Node.js, Python, C#, Docker wrapper libraries
- `references/mcp-and-homeassistant.md` — MCP server for AI integration, Home Assistant setups
- `references/common-patterns.md` — Polling loops, notification bots, dashboards, maps, historical archiving, multi-location monitoring, smart home, accessibility

## Critical constraint: geo-blocking

The official oref.org.il API **blocks non-Israeli IP addresses**. This is the single most important thing to know before writing any code. Every approach described here either runs from an Israeli IP or uses a proxy/middleware that does. When helping someone build an integration, always surface this constraint early — it saves hours of debugging.

**Recommended workaround:** Deploy on a GCP `me-west1` (Tel Aviv) VM. The `e2-micro` instance type is free-tier eligible and provides a legitimate Israeli IP.

```bash
# Create free-tier VM with Israeli IP
gcloud compute instances create pikud-haoref \
  --zone=me-west1-a \
  --machine-type=e2-micro \
  --image-family=debian-12 \
  --image-project=debian-cloud \
  --tags=http-server

# Open ports
gcloud compute firewall-rules create allow-pikud-haoref \
  --allow=tcp:8000-8002 --target-tags=http-server
```

---

## The official endpoints

Pikud HaOref doesn't publish formal API documentation. The endpoints below are reverse-engineered from the official website and mobile app, and have been stable for years. They return JSON (sometimes with a UTF-8 BOM that needs stripping).

### Real-time alerts

```
GET https://www.oref.org.il/warningMessages/alert/alerts.json
```

Returns the currently active alert, or an empty response when no alert is active. Must be polled — typically every 1–2 seconds for real-time responsiveness.

**Response when alert is active:**
```json
{
  "id": "134168709720000000",
  "cat": "1",
  "title": "ירי רקטות וטילים",
  "data": ["תל אביב - מרכז העיר", "רמת גן - מערב"],
  "desc": "היכנסו למרחב המוגן ושהו בו 10 דקות"
}
```

**Fields:**
- `id` — Unique alert identifier (numeric string). Useful for deduplication.
- `cat` — Category number (see alert categories below).
- `title` — Human-readable alert type in Hebrew.
- `data` — Array of affected location names in Hebrew.
- `desc` — Protective instructions in Hebrew.

**Response when no alert:** Empty body or empty JSON object.

**Required headers:** The endpoint expects a browser-like `Referer` and `X-Requested-With` header:
```
Referer: https://www.oref.org.il/
X-Requested-With: XMLHttpRequest
```

### Alert history

```
GET https://www.oref.org.il/warningMessages/alert/History/AlertsHistory.json
```

Returns alerts from the recent past (typically last 24 hours). Useful for historical analysis and catching alerts you may have missed between polls.

**Important:** The commonly documented path `/WarningMessages/History/AlertsHistory.json` (capital W/M, no `/alert/` segment) is blocked by Akamai WAF with a 403. The working path uses lowercase `warningMessages` and includes `/alert/` — matching what the Angular SPA actually requests.

**Response format:**
```json
[
  {
    "alertDate": "2024-10-15 14:32:00",
    "title": "ירי רקטות וטילים",
    "data": "אשדוד - א,ב,ד,ה",
    "category": 1
  }
]
```

Note that `data` here is a string (not an array like the real-time endpoint), and `category` is a number (not a string).

### Alert history (alternative endpoint)

```
GET https://alerts-history.oref.org.il/Shared/Ajax/GetAlarmsHistory.aspx?lang=he&mode=1
```

An ASP.NET endpoint on a dedicated subdomain that returns richer history data including `matrix_id`, `rid`, and `category_desc` fields. Useful as a fallback or when you need the extra metadata. Supports `lang=he` (Hebrew) and `lang=en` (English).

### Alert categories

```
GET https://www.oref.org.il/alerts/alertCategories.json
```

Returns the mapping of category numbers to alert types. Known categories:

| Category | Type | Hebrew |
|----------|------|--------|
| 1 | Missiles / Rockets | ירי רקטות וטילים |
| 2 | Hostile aircraft intrusion | חדירת כלי טיס עוין |
| 3 | Earthquake | רעידת אדמה |
| 4 | Tsunami | צונאמי |
| 5 | Radiological event | אירוע רדיולוגי |
| 6 | Hazardous materials | חומרים מסוכנים |
| 7 | Terrorist infiltration | חדירת מחבלים |
| 13 | Event conclusion | — |
| 14 | Pre-alert / news flash | — |

Categories 101–107 are the drill equivalents of 1–7.

---

## When to use raw endpoints vs. libraries

**Default to raw endpoints.** The official JSON files are simple, stable, and well-understood. For most use cases — polling for alerts, checking history, building a notification service — direct HTTP requests are all you need.

**Reach for a library when it provides data you'd otherwise have to maintain yourself.** The most common reason is the **cities.json database** — it contains ~1,500 locations with Hebrew/English/Russian/Arabic names, GPS coordinates, zone mappings, and time-to-shelter values. See `references/community-libraries.md` for details on each library.

---

## Location-specific queries

A very common use case is checking whether a specific city or area has an active alert. The challenge is that the official API returns location names in Hebrew, but users often ask in English or transliterated Hebrew.

### The cities database

The `pikud-haoref-api` Node.js library ships a comprehensive `cities.json` file (available at https://github.com/eladnava/pikud-haoref-api) that maps every alertable location. Each entry has:

```json
{
  "id": 511,
  "name": "אבו גוש",
  "name_en": "Abu Ghosh",
  "name_ru": "Абу Гош",
  "name_ar": "أبو غوش",
  "zone": "שפלת יהודה",
  "zone_en": "Judean Lowlands",
  "lat": 31.80686,
  "lng": 35.11038,
  "countdown": 90,
  "value": "אבו גוש"
}
```

**Key fields:** `name`/`name_en` for matching, `zone`/`zone_en` for regional queries, `countdown` for time-to-shelter in seconds, `lat`/`lng` for proximity and maps, `value` for matching against alert `data` array.

### Matching strategies

Alert location names in the `data` array don't always exactly match the city name. A location might appear as `"תל אביב - מרכז העיר"` rather than just `"תל אביב"`. Three strategies:

1. **Substring match** — `"תל אביב" in "תל אביב - מרכז העיר"` → True. Simple, works for most cases.
2. **Fuzzy match** — Use `fuzzywuzzy` or similar with threshold ~60-70. Handles transliteration variations.
3. **Zone-based match** — Match against `zone`/`zone_en` for broader regional queries.

### English-to-Hebrew name resolution

When a user asks about a city by English name, resolve it to the Hebrew `value`:
- **cities.json lookup** — Search `name_en`. Most reliable.
- **Fuzzy English search** — "Gan Hayim", "Gan Chaim", "Gan Hayyim" all → גן חיים.
- **Multi-language** — `name_ru` and `name_ar` also available.

### Proximity-based lookups

Using `lat`/`lng` coordinates from the cities database:

```python
from math import radians, cos, sin, asin, sqrt

def haversine(lat1, lon1, lat2, lon2):
    """Distance in km between two GPS coordinates."""
    lat1, lon1, lat2, lon2 = map(radians, [lat1, lon1, lat2, lon2])
    dlat, dlon = lat2 - lat1, lon2 - lon1
    a = sin(dlat/2)**2 + cos(lat1) * cos(lat2) * sin(dlon/2)**2
    return 2 * 6371 * asin(sqrt(a))

def cities_near(lat, lon, radius_km, cities_db):
    return [c for c in cities_db
            if haversine(lat, lon, c["lat"], c["lng"]) <= radius_km]
```

### Time-to-shelter

Every location has a `countdown` value (seconds). Ranges from 0 seconds (border communities near Gaza) to 180 seconds (central Israel). Always include this when reporting location-specific alerts — it's life-safety critical.

---

## Examples

**Example 1: "Build me a Telegram bot for rocket alerts"**
- Explain geo-blocking constraint upfront
- Use the polling loop pattern from `references/common-patterns.md`
- Add Telegram `sendMessage` in the `on_new_alert` callback
- Suggest GCP me-west1 deployment
- Result: Working bot that sends alerts to a Telegram chat

**Example 2: "Is there an alert in Ashkelon right now?"**
- Load cities.json to resolve "Ashkelon" → "אשקלון"
- Poll the real-time endpoint with required headers
- Substring-match "אשקלון" against the alert `data` array
- Report result including time-to-shelter (15 seconds for Ashkelon)

**Example 3: "Set up a live alert map dashboard"**
- Server-side: polling loop → fan out via SSE or WebSocket (see `references/common-patterns.md`)
- Client-side: Leaflet.js + OpenStreetMap, markers from cities.json coordinates
- Enrich markers with zone and countdown data
- Don't have browsers poll oref.org.il directly

**Example 4: "Integrate alerts into Home Assistant"**
- Point to oref_alert HACS integration (see `references/mcp-and-homeassistant.md`)
- Auto-configures from HA's home location
- Provides `sensor.oref_alert` and `_time_to_shelter` entities
- Common automations: flash lights, lock doors, pause media, announce via speakers

---

## Gotchas and troubleshooting

1. **UTF-8 BOM** — The official endpoint sometimes returns a BOM (`\ufeff`) before the JSON. Always strip it: `text.lstrip('\ufeff')`.
2. **Empty vs. no alert** — An empty response body means no active alert. Don't parse it as JSON.
3. **History corruption** — The history endpoint occasionally returns malformed JSON. Retry after a few seconds.
4. **Rate limiting** — Poll every 1–2 seconds for real-time. Going faster doesn't help and may get you blocked.
5. **Hebrew encoding** — All location names and descriptions are in Hebrew. Use UTF-8 throughout your stack.
6. **Multiple simultaneous alerts** — During heavy barrages, multiple alert types can be active simultaneously. Your code should handle arrays, not assume single alerts.
7. **Drill alerts** — Categories 101–107 are drills. Filter them unless you specifically want them.
8. **403 Forbidden** — Two common causes: (a) geo-blocking — deploy from an Israeli IP (GCP me-west1) or use a proxy; (b) Akamai WAF blocking the URL path — the endpoint paths are case-sensitive and must match exactly what the Angular SPA uses (lowercase `warningMessages`, include `/alert/` segment). The commonly documented uppercase paths return 403 even from Israeli IPs.
