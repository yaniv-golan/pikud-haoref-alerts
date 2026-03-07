# Common Patterns by Use Case

## Polling loop — the foundation of everything

Almost every integration starts with this. Poll the real-time endpoint, deduplicate by alert ID, and do something when a new alert arrives. This is the building block for notification bots, dashboards, and monitoring tools.

```python
import requests, time, json

ENDPOINT = "https://www.oref.org.il/warningMessages/alert/alerts.json"
HEADERS = {
    "Referer": "https://www.oref.org.il/",
    "X-Requested-With": "XMLHttpRequest",
    "User-Agent": "Mozilla/5.0"
}

seen_ids = set()

def on_new_alert(alert):
    """Replace this with your notification/dashboard/logging logic."""
    print(f"ALERT: {alert['title']} in {', '.join(alert['data'])}")

while True:
    try:
        resp = requests.get(ENDPOINT, headers=HEADERS, timeout=5)
        text = resp.text.lstrip('\ufeff').strip()  # strip BOM
        if text:
            alert = json.loads(text)
            if alert.get("id") and alert["id"] not in seen_ids:
                seen_ids.add(alert["id"])
                on_new_alert(alert)
    except Exception as e:
        print(f"Poll error: {e}")
    time.sleep(2)
```

## Notification bots (Telegram, Slack, Discord)

The most common use case. The pattern is: polling loop + message send on alert.

**Telegram:**
```python
import requests

BOT_TOKEN = "your-bot-token"
CHAT_ID = "your-chat-id"

def on_new_alert(alert):
    cities = ", ".join(alert["data"])
    text = f"🚨 {alert['title']}\n📍 {cities}\n⚠️ {alert['desc']}"
    requests.post(
        f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage",
        json={"chat_id": CHAT_ID, "text": text}
    )
```

**Slack (incoming webhook):**
```python
SLACK_WEBHOOK = "https://hooks.slack.com/services/T.../B.../..."

def on_new_alert(alert):
    requests.post(SLACK_WEBHOOK, json={
        "text": f"🚨 *{alert['title']}*\n{', '.join(alert['data'])}"
    })
```

**Discord (webhook):**
```python
DISCORD_WEBHOOK = "https://discord.com/api/webhooks/..."

def on_new_alert(alert):
    requests.post(DISCORD_WEBHOOK, json={
        "content": f"🚨 **{alert['title']}**\n{', '.join(alert['data'])}"
    })
```

For location-filtered notifications (e.g., "only alert me about rockets near Netanya"), combine with the cities.json lookup and zone matching from the location-specific queries section in SKILL.md.

## Real-time dashboards and maps

For a live map showing active alerts:

**Architecture:** Polling loop on the server → push updates to browser clients via WebSocket or SSE. Don't have each browser client poll the oref.org.il endpoint directly — that's wasteful and will get blocked.

**Frontend options:**
- **Leaflet.js** with OpenStreetMap tiles — lightweight, free, good for simple alert markers on a map. Use the `lat`/`lng` from cities.json to place markers.
- **Mapbox GL** — smoother rendering, better for heatmaps and polygon overlays. Use the `polygons.json` from the pikud-haoref-api repo for zone boundaries.
- **Google Maps** — works fine, but costs money at scale.

**Enrichment with cities.json:** When an alert comes in with `data: ["אשדוד - א,ב,ד,ה"]`, look up the city in cities.json to get coordinates, zone, and countdown time. Display the marker at those coordinates with a tooltip showing time-to-shelter.

**WebSocket relay for multi-client broadcast:** The [red-alert-websocket](https://github.com/YakirOren/red-alert-websocket) project polls in the background and exposes a WebSocket that multiple browser clients can subscribe to, so you don't hammer the official endpoint.

## Historical data analysis

The history endpoint (`AlertsHistory.json`) is hard-capped at 3,000 records — during low-activity periods this may cover weeks, but during heavy conflicts it can be exhausted in hours. For longer-term analysis, consider iterating Tzofar alert group IDs (see `references/alternative-data-sources.md`) or build your own archive by continuously polling and storing alerts.

**Storage approach:**
```python
import sqlite3, json, datetime

def store_alert(alert, db_path="alerts.db"):
    conn = sqlite3.connect(db_path)
    conn.execute("""CREATE TABLE IF NOT EXISTS alerts (
        id TEXT PRIMARY KEY,
        timestamp TEXT,
        category TEXT,
        title TEXT,
        locations TEXT,
        description TEXT
    )""")
    conn.execute(
        "INSERT OR IGNORE INTO alerts VALUES (?, ?, ?, ?, ?, ?)",
        (alert["id"], datetime.datetime.now().isoformat(),
         alert["cat"], alert["title"],
         json.dumps(alert["data"], ensure_ascii=False), alert["desc"])
    )
    conn.commit()
    conn.close()
```

**Analysis patterns people commonly want:**
- Alert frequency by city/zone over time (heatmap)
- Peak hours / days of week for alerts
- Average number of simultaneous locations per alert
- Time between successive alerts (escalation detection)
- Category breakdown (rockets vs. drones vs. other)

All of these are straightforward SQL queries once you have the data in a database. Enrich with cities.json to get zone/coordinates for geographic analysis.

## Multi-location monitoring

For organizations with offices or personnel in several cities — monitor all locations and route alerts to the right people.

**Pattern:** Load a config of watched locations (with Hebrew names from cities.json), and when an alert comes in, check which watched locations are affected:

```python
WATCHED = {
    "Tel Aviv HQ": ["תל אביב"],
    "Haifa R&D": ["חיפה"],
    "Beer Sheva warehouse": ["באר שבע"],
}

def on_new_alert(alert):
    affected = alert.get("data", [])
    for office, hebrew_names in WATCHED.items():
        for name in hebrew_names:
            if any(name in loc for loc in affected):
                notify_team(office, alert)
                break
```

## Push vs. poll: SSE and WebSockets

The official API is poll-only. But if you're building a system where multiple consumers need alerts, polling once and fanning out is better than each consumer polling independently.

**Options for fan-out:**
- **SSE (Server-Sent Events):** The pikud-a-oref-mcp FastAPI middleware (port 8000) already does this — poll once, publish to `/api/alerts-stream`. Clients connect with `EventSource` in the browser or `httpx`/`aiohttp` in Python.
- **WebSocket:** The [red-alert-websocket](https://github.com/YakirOren/red-alert-websocket) project does this. Good for bidirectional communication if you need it (e.g., clients acknowledging receipt).
- **Message queue (Redis pub/sub, MQTT):** For larger systems where consumers are microservices. Poll → publish to a topic → each service subscribes.

## Smart home automation

Beyond the Home Assistant integrations (oref_alert, RedAlert AppDaemon) described in `references/mcp-and-homeassistant.md`, common automations include:
- Flash smart lights red when alert is active in your area
- Lock smart locks and close smart shutters
- Pause media playback and announce alert via smart speakers
- Send push notification to family members' phones
- Activate security cameras

These all follow the same pattern: polling loop (or HA sensor trigger) → condition check (is my area affected?) → action.

## Accessibility

The [RedAlerts-For-Hearing-Impaired](https://github.com/Bnux256/RedAlerts-For-Hearing-Impaired) project uses a Raspberry Pi with vibrating motors and LEDs for physical notification — useful reference for any hardware alert system.

For software accessibility: the alert data is text-based, so it works naturally with screen readers. The main challenge is real-time delivery — a desktop notification or system tray app that announces alerts via text-to-speech is straightforward to build on top of the polling loop.

Multi-language support is important here — many Israeli residents are more comfortable in Russian or Arabic. The cities.json `name_ru`/`name_ar` fields and the C# RedAlert library's multi-language support help with this.

## Third-party data sources

Besides the official oref.org.il endpoints, there are alternative channels. See `references/alternative-data-sources.md` for full details including API endpoints, data models, and category mappings.

- **Tzofar (tzevaadom.co.il)** — A third-party alert relay service with **no geo-blocking**. Provides a recent alerts API and single-alert lookup by ID (iterate backwards for historical data). Critical caveat: does NOT include pre-alerts (oref cat 14) or event-concluded messages (oref cat 13). The oref_alert Home Assistant integration uses it as one of its data channels.
- **Community archive projects** — hasadna/oref-alarms-history, Meir017/oref-data, and others continuously poll and archive alerts, solving the 3,000-record cap problem. See `references/alternative-data-sources.md` for links.

The official oref.org.il endpoint should be treated as the authoritative source for real-time alerting. Tzofar and community archives are valuable for historical analysis and as redundant/fallback sources.
