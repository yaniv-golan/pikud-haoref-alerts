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
  version: 0.5.6
  tags: [israel, alerts, civil-defense, pikud-haoref, tzeva-adom, rockets, emergency]
---

# Pikud HaOref Alert APIs

This skill covers everything you need to know to work with the Pikud HaOref (Israel Home Front Command) alert system — from raw official endpoints to community wrapper libraries and deployment strategies.

For detailed reference material, see the `references/` directory:
- `references/community-libraries.md` — Node.js, Python, C#, Docker wrapper libraries
- `references/mcp-and-homeassistant.md` — MCP server for AI integration, Home Assistant setups
- `references/common-patterns.md` — Polling loops, notification bots, dashboards, maps, historical archiving, multi-location monitoring, smart home, accessibility
- `references/alternative-data-sources.md` — Tzofar (tzevaadom.co.il) API, oref-to-Tzofar category mapping, community archive projects

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

**Critical: this is a snapshot, not a status check.** Alerts appear on this endpoint only briefly (typically seconds to a minute) while the siren is actively sounding. An empty response does NOT mean "all clear" — it only means no siren is sounding right now. To determine if a situation is still active, you must also check the history endpoint for recent alerts without a matching category 13 "event concluded" message (see [Determining if a situation is still active](#determining-if-a-situation-is-still-active) below).

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

### Alert history (JSON file — unreliable under load)

```
GET https://www.oref.org.il/warningMessages/alert/History/AlertsHistory.json
```

Returns the most recent alerts, **hard-capped at 3,000 records with no pagination**. During low-activity periods this may cover weeks; during high-intensity conflicts a single day can exceed 3,000 records (e.g., 1,147 pre-alerts + 483 missile alerts + 22 aircraft + 1,348 concluded = 3,000 exactly). Useful for catching alerts you may have missed between polls, but not reliable for historical analysis during escalation periods.

**Degradation under load:** During high-volume periods (active conflict), `AlertsHistory.json` can return HTTP 200 with an empty body (~2 bytes) while `GetAlarmsHistory.aspx` continues to return full data (600KB+). **Prefer `GetAlarmsHistory.aspx` as the primary history source** and treat `AlertsHistory.json` as a fallback.

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

### Alert history (recommended)

```
GET https://alerts-history.oref.org.il/Shared/Ajax/GetAlarmsHistory.aspx?lang=he&mode=1
```

An ASP.NET endpoint on a dedicated subdomain that returns richer history data including `matrix_id`, `rid`, and `category_desc` fields. Useful as a fallback or when you need the extra metadata. Supports `lang=he` (Hebrew) and `lang=en` (English).

**Same 3,000-record cap applies.** The `mode` parameter accepts values 1–3 (modes 4–5 return empty), but all modes return the same 3,000 most recent records. No date-range query parameters are supported. This is not a pagination mechanism.

**Date format differences between history endpoints:**
- `AlertsHistory.json`: `"alertDate": "2026-03-06 19:33:53"` (space-separated, with seconds)
- `GetAlarmsHistory.aspx`: `"alertDate": "2026-03-06T19:35:00"` (ISO 8601 T-separator, seconds always `:00`)

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
| 13 | Event conclusion | האירוע הסתיים |
| 14 | Pre-alert / incoming alerts warning | בדקות הקרובות צפויות להתקבל התרעות באזורך |

Categories 101–107 are the drill equivalents of 1–7.

### Category 13: event conclusion

When Pikud HaOref determines the threat to a location has passed, it sends a category 13 alert with the title "האירוע הסתיים" (the event has ended) or "ניתן לצאת מהמרחב המוגן" (you may exit the protected space). This is the **authoritative signal** that a specific alert event is over. It appears per-location in the history endpoint.

### Category 14: pre-alert (incoming alerts warning)

A category 14 alert with the title "בדקות הקרובות צפויות להתקבל התרעות באזורך" warns that alerts are expected in the coming minutes for a given area. This is an early warning that a barrage is incoming — valuable for preparation time beyond the standard time-to-shelter countdown. If a pre-alert doesn't escalate to an actual alert (cat 1–7) within ~20 minutes, it can be considered expired.

### Matching pre-alerts to actual alerts

A common analysis task is determining which pre-alerts (cat 14) escalated to actual alerts (cat 1–7). The challenge: location strings don't always match cleanly between categories, and multiple alerts may follow a single pre-alert.

```python
from datetime import timedelta

def match_prealerts_to_alerts(history, source="oref_history", window_minutes=20):
    """Match cat 14 pre-alerts to subsequent cat 1-7 alerts by location overlap.

    Uses normalize_alert_time() for proper datetime comparison.
    Returns list of (prealert, [matching_alerts]) tuples.
    """
    prealerts = [e for e in history if e.get("category") == 14]
    alerts = [e for e in history if e.get("category") in (1, 2, 3, 4, 5, 6, 7)]
    window = timedelta(minutes=window_minutes)
    matches = []

    for pa in prealerts:
        pa_time = normalize_alert_time(pa["alertDate"], source)
        pa_locations = pa.get("data", "")
        matched = []
        for a in alerts:
            a_time = normalize_alert_time(a["alertDate"], source)
            if a_time <= pa_time or a_time > pa_time + window:
                continue
            a_locations = a.get("data", "")
            # Token overlap: handles partial location name matches
            pa_words = set(pa_locations.replace(",", " ").split())
            a_words = set(a_locations.replace(",", " ").split())
            if pa_words & a_words:
                matched.append(a)
        matches.append((pa, matched))

    return matches
```

**Notes:** Location strings between cat 14 and cat 1 don't always match exactly — cat 14 often uses zone/area names while cat 1 uses specific city names. Substring or token overlap matching works better than exact matching. The 20-minute window reflects the typical pre-alert-to-alert escalation time; adjust based on observed patterns.

### Determining if a situation is still active

**Do not rely solely on `alerts.json` being empty.** The real-time endpoint is a narrow snapshot — alerts disappear within seconds. The correct approach is a dual-check:

1. **Check `alerts.json`** for the instant snapshot (is a siren sounding right now?)
2. **Check the history endpoint** for recent cat 1–7 alerts for the location
3. **Check if a matching cat 13** "event concluded" has followed the alert for that location
4. If no cat 13 has been received yet → the situation is **still active**

```python
def is_situation_active(location_hebrew, history):
    """Check if a location still has an active alert (no cat 13 received yet)."""
    last_alert_time = None
    last_concluded_time = None

    for entry in history:
        if location_hebrew not in entry.get("data", ""):
            continue
        cat = entry.get("category", 0)
        alert_time = entry.get("alertDate", "")
        if cat in (1, 2, 3, 4, 5, 6, 7):
            if last_alert_time is None or alert_time > last_alert_time:
                last_alert_time = alert_time
        elif cat == 13:
            if last_concluded_time is None or alert_time > last_concluded_time:
                last_concluded_time = alert_time

    if last_alert_time is None:
        return False  # no recent alert
    if last_concluded_time is None:
        return True   # alert with no conclusion yet
    return last_alert_time > last_concluded_time  # alert after last conclusion
```

### Historical data strategy

The 3,000-record cap on both official history endpoints means you need different sources depending on your time horizon:

| Need | Source | Notes |
|------|--------|-------|
| Last few minutes | `alerts.json` (real-time) | Poll every 1–2 seconds |
| Last ~3,000 records (all categories) | `AlertsHistory.json` or `GetAlarmsHistory.aspx` | Includes pre-alerts + concluded; hours to weeks depending on activity |
| Last 50 alert groups (no pre-alerts) | `api.tzevaadom.co.il/alerts-history` | No geo-blocking |
| Months/years (no pre-alerts) | Iterate Tzofar alert group IDs backwards | No geo-blocking; watch rate limits (see `references/alternative-data-sources.md`) |
| Complete historical with pre-alerts | Your own continuous poller | **No retroactive source exists** — must be running before the period you need |

**Pre-alert historical gap (critical limitation):** There is no retroactive source for pre-alert data (cat 14) or event-concluded messages (cat 13). Tzofar excludes them entirely. The official oref history endpoints include them but are capped at 3,000 records. During active conflict, those 3,000 records may cover only hours. **If you need historical pre-alert data, you MUST set up your own continuous poller before the period you want to analyze.** There is no way to recover this data after the fact. Community archives (hasadna, Meir017) also lack pre-alerts unless their specific scraper captures them.

**Combining oref + Tzofar data:** For multi-day conflict analysis, you'll typically need both sources. Key alignment points: (1) normalize timestamps — oref uses Israel local time strings, Tzofar uses Unix timestamps (UTC); (2) use the category mapping table in `references/alternative-data-sources.md` to align threat types; (3) deduplicate by matching on timestamp + city name, since both sources report the same underlying alerts.

```python
from datetime import datetime, timezone
from zoneinfo import ZoneInfo  # stdlib in Python 3.9+

IST = ZoneInfo("Asia/Jerusalem")  # handles DST automatically

def normalize_alert_time(raw, source):
    """Normalize any alert timestamp to a Python datetime in Israel time.

    source: 'oref_history' | 'oref_aspx' | 'tzofar'
    """
    if source == "oref_history":
        # "2026-03-07 19:33:53" (space-separated, Israel local time)
        return datetime.strptime(raw, "%Y-%m-%d %H:%M:%S").replace(tzinfo=IST)
    elif source == "oref_aspx":
        # "2026-03-07T19:35:00" (ISO T-separator, Israel local time)
        return datetime.strptime(raw, "%Y-%m-%dT%H:%M:%S").replace(tzinfo=IST)
    elif source == "tzofar":
        # Unix timestamp (UTC)
        return datetime.fromtimestamp(raw, tz=timezone.utc).astimezone(IST)
```

**Oref category to Tzofar threat mapping:** When working with both data sources, note that the numeric IDs differ. See the full mapping table in `references/alternative-data-sources.md`.

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
- **Dual-check**: poll `alerts.json` AND check history for recent cat 1–7 alerts without a matching cat 13
- Substring-match "אשקלון" against the alert `data` fields
- Report result including time-to-shelter (15 seconds for Ashkelon)
- If alerts.json is empty but history shows a recent alert with no cat 13 → situation is still active

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

**Example 5: "Analyze alert patterns over the last week of conflict"**
- First question: do you need pre-alerts (cat 14)? If yes, you need your own poller archive — no retroactive source exists
- For actual threat alerts only: iterate Tzofar alert group IDs backwards with conservative pacing (1–2s delays)
- For the most recent data (if within 3,000 records): use `GetAlarmsHistory.aspx` which includes all categories
- Normalize Tzofar and oref data into a common schema using the category mapping in `references/alternative-data-sources.md`
- Enrich with cities.json for coordinates and zone data for geographic analysis

---

## Gotchas and troubleshooting

1. **UTF-8 BOM** — The official endpoint sometimes returns a BOM (`\ufeff`) before the JSON. Always strip it: `text.lstrip('\ufeff')`.
2. **Empty vs. no alert** — An empty response body means no active alert. Don't parse it as JSON.
3. **History corruption** — The history endpoint occasionally returns malformed JSON. Retry after a few seconds.
4. **Rate limiting** — Poll every 1–2 seconds for real-time. Going faster doesn't help and may get you blocked.
5. **Hebrew encoding** — All location names and descriptions are in Hebrew. Use UTF-8 throughout your stack.
6. **Multiple simultaneous alerts** — During heavy barrages, multiple alert types can be active simultaneously. Your code should handle arrays, not assume single alerts.
7. **Drill alerts** — Categories 101–107 are drills. Filter them unless you specifically want them.
8. **3,000-record history cap** — Both `AlertsHistory.json` and `GetAlarmsHistory.aspx` are hard-capped at 3,000 records with no pagination or date-range filtering. During the March 2026 conflict, 3,000 records were exhausted in ~97 minutes (854 pre-alerts + 480 missiles + 38 aircraft + 1,628 concluded). The `mode=1,2,3` parameter on `GetAlarmsHistory.aspx` does NOT provide pagination — all modes return the same 3,000 most recent records (`mode=4,5` return empty). For deeper history, use Tzofar's archive or a community poller (see `references/alternative-data-sources.md`).
9. **Israel timezone** — All oref timestamps are in Israel local time (UTC+2, or UTC+3 during DST which runs late March to late October). Tzofar uses Unix timestamps (UTC). When combining sources or grouping by day, normalize to a consistent timezone first.
10. **403 Forbidden** — Two common causes: (a) geo-blocking — deploy from an Israeli IP (GCP me-west1) or use a proxy; (b) Akamai WAF blocking the URL path — the endpoint paths are case-sensitive and must match exactly what the Angular SPA uses (lowercase `warningMessages`, include `/alert/` segment). The commonly documented uppercase paths return 403 even from Israeli IPs.
