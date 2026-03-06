<p align="center">
  <img src="assets/banner.png" alt="Pikud HaOref Alerts — Claude Skill for Israel's Home Front Command" width="100%" />
</p>

# Pikud HaOref Alerts — Claude Plugin

A Claude plugin that provides comprehensive knowledge for working with Israel's Pikud HaOref (Home Front Command) alert system.

## What it does

When installed, Claude gains deep knowledge of how to build integrations with Israel's civil defense alert infrastructure — from raw API endpoints to community libraries, deployment patterns, and common use cases.

This covers:

- **Official oref.org.il endpoints** — real-time alerts, history, categories, required headers, BOM handling
- **Geo-blocking workarounds** — the API blocks non-Israeli IPs; the skill explains the standard GCP me-west1 deployment pattern
- **Community libraries** — Node.js (`pikud-haoref-api`), Python (`python-red-alert`), C# (`RedAlert`), Docker (`orefAlerts`)
- **MCP server** — `pikud-a-oref-mcp` for AI assistant integration
- **Home Assistant** — `oref_alert` (HACS) and `RedAlert` (AppDaemon)
- **Location data** — cities.json database with ~1,500 locations, coordinates, shelter times, multi-language names
- **Common patterns** — polling loops, Telegram/Slack/Discord bots, dashboards, maps, historical archiving, multi-location monitoring, smart home automation

## Install

### Claude Code

Add the marketplace, then install the plugin:

```
/plugin marketplace add yaniv-golan/pikud-haoref-alerts
/plugin install pikud-haoref-alerts
```

### Claude Cowork

1. Open the Claude Desktop app and switch to the **Cowork** tab
2. Click **Customize** in the left sidebar
3. Click **Browse plugins** → **Personal** → **+** →**Add marketplace from GitHub**
4. Enter `yaniv-golan/pikud-haoref-alerts`
5. Click **Sync**

## When it triggers

The skill activates when you ask Claude about topics like:

- Building a Telegram bot for rocket alerts
- Fetching data from oref.org.il
- Setting up a real-time alert dashboard
- Checking alerts for a specific city
- Integrating Home Front Command alerts into Home Assistant
- Working with the pikud-haoref-api npm package
- Debugging 403 errors from oref.org.il
- Historical alert data collection

It also recognizes Hebrew references like "פיקוד העורף", "צבע אדום", and related terms.

May we never need this. Wishing for quiet skies and lasting peace.

## License

MIT

---