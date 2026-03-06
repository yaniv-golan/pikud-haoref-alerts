# Community Wrapper Libraries

These are useful primarily for their bundled data files and convenience methods. You don't always need to install the full library — sometimes just grabbing `cities.json` or `polygons.json` from the repo is enough.

## Node.js: pikud-haoref-api

**GitHub:** https://github.com/eladnava/pikud-haoref-api
**Install:** `npm install pikud-haoref-api --save`

**When it adds value:** Its `cities.json` and `polygons.json` are the most comprehensive location datasets available — even if you're not using Node.js, you can grab these JSON files directly from the repo. The library itself provides a clean polling interface with proxy support for running outside Israel.

```javascript
const pikudHaoref = require('pikud-haoref-api');

pikudHaoref.getActiveAlert(function(err, alert) {
  if (alert.type === 'missiles') {
    console.log('Alert in:', alert.cities.join(', '));
  }
});
```

**Alert type constants:** `none`, `missiles`, `radiologicalEvent`, `earthQuake`, `tsunami`, `hostileAircraftIntrusion`, `hazardousMaterials`, `terroristInfiltration`, `newsFlash`, `unknown`, plus drill variants like `missilesDrill`.

Note: `earlyWarning` was renamed to `newsFlash` because Pikud HaOref started using the same category for "safe to leave shelter" notifications.

## Python: python-red-alert

**GitHub:** https://github.com/Zontex/python-red-alert
**Install:** `pip install requests` (it's a single-file module)

**When it adds value:** Geolocation enrichment — it resolves alert codes to coordinates, city names, shelter times, and area codes. Also useful for generating random coordinates within affected cities for map visualizations. For simple "poll and notify" scripts, raw endpoint access is sufficient and this library is overkill.

**Key capabilities:**
- Real-time alert retrieval
- Location data from alert codes (lat/lon, city name, time to shelter)
- Random coordinate generation within affected cities (useful for map visualizations)

**Response enrichment — each city in an alert includes:**
- `label` — Location name
- `areaid`, `areaname` — Regional identifiers
- `migun_time` — Time to reach shelter (seconds)
- `city_data` — Nested object with coordinates and administrative info

## C#: RedAlert

**GitHub:** https://github.com/PwnTheStack/RedAlert

**When it adds value:** If you're in the .NET ecosystem and need an event-driven interface with continuous background sync. Also has built-in multi-language support (Hebrew, Arabic, Russian, English) and mapping capabilities. If you just need to poll and check alerts, a simple `HttpClient` call to the raw endpoint is easier.

## Docker: orefAlerts

**GitHub:** https://github.com/dmatik/orefAlerts
**Docker Hub:** `dmatik/oref-alerts`

**When it adds value:** When you need a self-hosted proxy that solves the Israeli IP problem — deploy this container on a GCP me-west1 VM, then your app anywhere in the world queries it via simple REST (`/current`, `/last_day`). If you're already running from an Israeli IP, you probably don't need this layer. Note: deprecated in favor of `oref-alerts-proxy-ms` (Java Spring Boot).
