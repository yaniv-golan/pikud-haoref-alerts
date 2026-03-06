# MCP Server and Home Assistant Integrations

## MCP Server: pikud-a-oref-mcp

**GitHub:** https://github.com/LeonMelamud/pikud-a-oref-mcp

A full middleware + MCP server stack for AI assistant integration. Uses a publish-subscribe architecture:

1. **FastAPI middleware** (port 8000) — Polls oref.org.il every 2 seconds, publishes via SSE
2. **MCP server** (port 8001) — Subscribes to SSE stream, exposes tools for Claude/LLMs
3. **SSE gateway** (port 8002) — Relays alerts to external clients

### MCP Tools

| Tool | Purpose |
|------|---------|
| `check_current_alerts` | Active alerts from SSE stream |
| `get_alert_history` | Recent alerts with city/limit filtering |
| `get_connection_status` | System health check |

City filtering supports exact substring matching, fuzzy matching (threshold 60), and multi-city queries. Use Hebrew city names.

### FastAPI endpoints

| Endpoint | Auth | Purpose |
|----------|------|---------|
| `GET /api/alerts-stream` | X-API-Key | SSE stream |
| `GET /api/alerts/current` | Optional | Current alert |
| `GET /api/alerts/history?city=...&limit=...` | Optional | Historical query |
| `GET /api/alerts/city/{city_name}` | Optional | City-specific |
| `GET /api/alerts/stats` | Optional | Aggregate stats |
| `GET /health` | None | Health check |

### Deployment (Docker)

```bash
git clone https://github.com/LeonMelamud/pikud-a-oref-mcp.git
cd pikud-a-oref-mcp
cp .env.example .env   # set API_KEY
make deploy             # build + start + health check
```

### MCP client config (Claude Desktop / VSCode / Cursor)

```json
{
  "servers": {
    "pikud-haoref": {
      "type": "http",
      "url": "http://localhost:8001/mcp"
    }
  }
}
```

---

## Home Assistant Integrations

Two mature options for smart home alert integration:

### oref_alert (HACS)

**GitHub:** https://github.com/amitfin/oref_alert

Monitors emergency messages via the `sensor.oref_alert` entity, auto-configured from HA's home location. Also provides `_time_to_shelter` sensors.

**Data channels:** `website-history`, `website` (real-time), `mobile` (app notifications), `tzevaadom` (third-party), `synthetic` (custom actions). When the same alert arrives on multiple channels, only the first is used.

### RedAlert (AppDaemon)

**GitHub:** https://github.com/idodov/RedAlert

AppDaemon app that monitors multiple hazard types: missiles, unauthorized aircraft, earthquakes, tsunamis, terrorist incursions, chemical emergencies.
