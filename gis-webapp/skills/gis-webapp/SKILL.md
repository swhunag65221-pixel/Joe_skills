---
name: gis-webapp
description: Use when creating, debugging, or deploying MapLibre GL JS web map applications — desktop or mobile variants. Triggers include building GeoJSON choropleth or prediction map UIs, fixing dark/black polygon rendering, timeline slider playback, popup/hover interactions, deploying static map sites to Netlify, or setting up local map dev servers. For static-data MapLibre apps, not tile pipelines or backend GIS services.
---

# GIS Webapp Development

## Overview

Build interactive geospatial prediction map webapps using MapLibre GL JS with static JSON data. Architecture: pure HTML/CSS/JS frontend (no framework), Python local server, GeoJSON boundaries + per-date JSON predictions, Netlify static deploy.

## When to Use

- Building a new map webapp for spatial prediction visualization
- Debugging MapLibre rendering issues (black polygons, null values, missing layers)
- Deploying map sites to Netlify (desktop or mobile variants)
- Setting up local dev servers for map development
- Adding time-series playback to prediction maps

## Architecture Pattern

```
[Training Script]
  -> prediction CSV (per-date x per-area)
    -> [Data Prep Script]
      -> data/boundaries.geojson    (polygon geometries, loaded once)
      -> data/predictions/{date}.json  (score per area code, one file per day)
      -> data/dates.json            (sorted date array for timeline)
      -> data/cases.json            (point data with lat/lon, optional)
        -> [Frontend app.js]
          -> MapLibre GL JS renders polygons + points
```

## Quick Reference

### Project Structure

```
webapp_name/
  index.html          # Entry point
  css/style.css       # Styles
  js/app.js           # MapLibre app (class-based)
  serve.py            # Local dev server
  data/
    boundaries.geojson
    dates.json         # ["2023-01-01", "2023-01-02", ...]
    predictions/       # One JSON per date
      2023-01-01.json  # {"AREA_CODE": 0.85, ...}
    cases.json         # [{date, lat, lon, ...}, ...]
  scripts/
    prepare_boundaries.py
    generate_predictions.py
```

### Local Dev Server (Threading Recommended)

```python
import http.server, socketserver, os

PORT = 8080
os.chdir(os.path.dirname(os.path.abspath(__file__)))

class Handler(http.server.SimpleHTTPRequestHandler):
    def end_headers(self):
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Cache-Control', 'no-cache')
        super().end_headers()
    def log_message(self, fmt, *args):
        pass

class ThreadedServer(socketserver.ThreadingMixIn, socketserver.TCPServer):
    allow_reuse_address = True

with ThreadedServer(("", PORT), Handler) as httpd:
    httpd.serve_forever()
```

**Avoid plain `TCPServer`** — single-threaded server can stall/block under concurrent browser connections (Playwright, prefetch, keepalive). Prefer `ThreadingMixIn` for local development.

### MapLibre Data-Driven Styling (Null-Safe)

```javascript
// Use coalesce to handle null/undefined properties
'fill-color': [
  'interpolate', ['linear'],
  ['coalesce', ['get', 'weighted_score'], 0],  // fallback to 0
  0,   'rgba(255,255,255,0)',   // transparent at 0
  0.1, '#fee2e2',
  0.3, '#fca5a5',
  0.5, '#f87171',
  0.8, '#dc2626',
  1.0, '#7f1d1d'
]
```

**Avoid bare `['get', 'property']` in interpolate.** When the property is null/undefined (before first data join, or missing keys), MapLibre may render an unintended fallback color (commonly black). Wrap with `['coalesce', ['get', ...], default]` to provide explicit fallback.

### Client-Side Data Join Pattern

```javascript
// Load predictions per date, join to boundary features
async renderDate(dateStr) {
  const predictions = await this.loadPredictions(dateStr);
  this.boundaries.features.forEach(f => {
    f.properties.weighted_score = predictions[f.properties.CODEBASE] ?? 0;
  });
  this.map.getSource('boundaries').setData(this.boundaries);
}
```

**Property name must match** the `['get', 'weighted_score']` in the paint expression. Mismatched names cause silent fallback to default color.

### Time-Window Case Filtering

For displaying cases relative to selected date D (left-closed, right-open intervals). Use **local Date arithmetic with string formatting** to avoid UTC timezone issues:

```javascript
// Timezone-safe: all arithmetic in local time, format manually
function offsetDate(dateStr, days) {
  const [y, m, d] = dateStr.split('-').map(Number);
  const dt = new Date(y, m - 1, d + days);  // local Date constructor
  const yy = dt.getFullYear();
  const mm = String(dt.getMonth() + 1).padStart(2, '0');
  const dd = String(dt.getDate()).padStart(2, '0');
  return `${yy}-${mm}-${dd}`;
}

const tomorrow = offsetDate(dateStr, 1);
const f14End   = offsetDate(dateStr, 15);   // exclusive
const p13Start = offsetDate(dateStr, -13);
const p28Start = offsetDate(dateStr, -28);

// String comparison on 'YYYY-MM-DD' — lexicographically correct
const future = cases.filter(c => c.date >= tomorrow && c.date < f14End);    // red hollow
const past14 = cases.filter(c => c.date >= p13Start && c.date < tomorrow);  // orange solid
const past28 = cases.filter(c => c.date >= p28Start && c.date < p13Start);  // yellow solid
```

Three windows are **non-overlapping and contiguous** (past28 is 15 days: D-28 to D-14 inclusive). Avoid `toISOString().slice(0,10)` — it converts to UTC and can shift the day in positive timezone offsets (e.g. UTC+8).

### FIFO Prediction Cache

```javascript
async loadPredictions(dateStr) {
  if (this.cache.has(dateStr)) return this.cache.get(dateStr);
  const resp = await fetch(`data/predictions/${dateStr}.json`);
  if (!resp.ok) return {};
  const data = await resp.json();
  if (this.cache.size >= 50) {
    this.cache.delete(this.cache.keys().next().value);
  }
  this.cache.set(dateStr, data);
  return data;
}
```

## Desktop vs Mobile Variants

| Aspect | Desktop | Mobile |
|--------|---------|--------|
| Layout | Sidebar + map | Full-screen map + bottom drawer |
| Controls | Always visible in sidebar | Hidden in collapsible drawer |
| Touch | Mouse events | Touch swipe on drawer handle |
| Port | 8080 | 8083 |
| Data | Own copy | Can symlink to desktop (local only) |

**Mobile drawer pattern:** swipe handle only (not content area) to avoid conflict with map scroll.

```javascript
handle.addEventListener('touchend', e => {
  const delta = touchStartY - e.changedTouches[0].clientY;
  if (delta > 40) openDrawer();    // swipe up
  if (delta < -40) closeDrawer();  // swipe down
});
```

## Netlify Deployment

### Prerequisites

Requires `netlify login` (or CI token via `NETLIFY_AUTH_TOKEN`) before first deploy.

### Create and Deploy

```bash
# Create site (non-interactive)
netlify sites:create --name SITE_NAME --account-slug ACCOUNT_SLUG

# Deploy static directory
netlify deploy --prod --dir=results/WEBAPP_DIR --site=SITE_ID
```

### Critical: Symlinks Not Followed

Netlify **does not follow symlinks**. If mobile variant uses `data/ -> ../desktop/data`, you must:
- Copy data directory before deploy: `rm data && cp -r ../desktop/data data`
- Or use Netlify `_redirects` to proxy from desktop site

### Cache Headers (netlify.toml)

```toml
[[headers]]
  for = "/data/dates.json"
  [headers.values]
    Cache-Control = "public, max-age=300"        # 5 min

[[headers]]
  for = "/data/predictions/*.json"
  [headers.values]
    Cache-Control = "public, max-age=86400"      # 1 day

[[headers]]
  for = "/data/boundaries.geojson"
  [headers.values]
    Cache-Control = "public, max-age=604800"     # 7 days
```

Cache duration rationale: boundaries rarely change, predictions update daily, dates list changes with new predictions.

## Ovitrap Webapp — 最小統計區未來 1-4 週預測

`results/ovitrap_webapp/` 的專用模式，預測陽性率 (pos) 和平均卵數 (egg) 各 4 週。

### Indexed Prediction Format

預測 JSON 使用 indexed 格式減少檔案大小：

```javascript
// Raw: {c:[codes], p1:[[q05,q50,q95,width,actual],...], e1:[...], ...}
// Decode to: {code: {p1:[q05,q50,q95,width,actual], e1:[...], ...}}
const codes = raw.c;
for (let i = 0; i < codes.length; i++) {
  const entry = {};
  for (const key of ['p1','p2','p3','p4','e1','e2','e3','e4']) {
    if (raw[key]?.[i]) entry[key] = raw[key][i];
  }
  data[codes[i]] = entry;
}
```

### Canvas Custom Icons (Non-SDF)

**誘卵桶（SDF icon + 黑色描邊）：**
- Canvas 畫水桶形狀（橢圓 rim + 梯形 body + 弧形 handle），全白填充
- `sdf: true` 啟用 MapLibre data-driven recoloring（卵數漸層）
- 描邊用 `icon-halo-color: '#000000'`, `icon-halo-width: 1.5`

**病例十字（固定色 icon）：**
- Canvas 畫十二頂點 cross polygon，先 stroke 黑色邊再 fill 紅色
- `sdf: false` 保留原始顏色（不需要 data-driven recoloring）

```javascript
// Cross icon: draw path then stroke-before-fill for border effect
const drawCross = () => {
  ctx.beginPath();
  ctx.moveTo(mid - arm, 0);
  ctx.lineTo(mid + arm, 0);
  // ... 12-vertex cross polygon
  ctx.closePath();
};
drawCross(); ctx.strokeStyle = '#000'; ctx.lineWidth = 2.5; ctx.stroke();
drawCross(); ctx.fillStyle = '#dc2626'; ctx.fill();
map.addImage('case-cross', ctx.getImageData(0,0,size,size), { sdf: false });
```

### Case Jitter for Overlapping Points

多個病例同一座標時，用黃金角螺旋 (golden angle) 錯位，確保均勻散開：

```javascript
const angle = count * 2.4;       // ~137.5° golden angle
const radius = count * 0.0003;   // ~30m per step
const jLon = lon + radius * Math.cos(angle);
const jLat = lat + radius * Math.sin(angle);
```

### Popup Auto-Dismiss

MapLibre `closeOnClick: true` 只監聽地圖畫布。側邊欄等外部元素需額外監聽：

```javascript
const popup = new maplibregl.Popup({ closeOnClick: true });
document.addEventListener('click', e => {
  if (!e.target.closest('.maplibregl-popup') && !e.target.closest('#map')) {
    popup.remove();
  }
});
```

### Red Gradient Color Scheme

卵數/陽性率統一使用白→紅漸層（0 = 透明白, 最大值 = 正紅 #dc2626），比多色漸層更直覺。

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Bare `['get', prop]` in interpolate | Black/dark polygon fill | Wrap with `['coalesce', ..., 0]` |
| Single-threaded `TCPServer` | Server hangs after browser disconnect | Use `ThreadingMixIn` |
| Symlink in Netlify deploy | 404 on data files | Copy real files or use `_redirects` |
| Missing `allow_reuse_address` | `Address already in use` on restart | Set `allow_reuse_address = True` on server class |
| Forgot `map.resize()` after drawer toggle | Map renders at wrong size | Call `map.resize()` after CSS transition ends |
| `setData` before map `load` event | Source not found error | Wrap layer setup in `map.on('load', ...)` |
| SDF icon without halo | Icon has no visible border | Set `icon-halo-color` + `icon-halo-width` |
| `sdf: true` on multi-color icon | Icon loses original colors | Use `sdf: false` for fixed-color icons (e.g. case cross) |
| `closeOnClick` only on map | Popup stays when clicking sidebar | Add `document.addEventListener('click')` to remove popup |
| `toISOString().slice(0,10)` in UTC+8 | Date shifts backward by 1 day | Use local `new Date(y,m-1,d)` + manual formatting |

## Verification Checklist

After deploying or modifying a GIS webapp:

- [ ] Local server starts without errors on assigned port
- [ ] Boundaries render as transparent (not black) on initial load
- [ ] Date slider/picker changes predictions correctly
- [ ] Case point layers show correct colors per time window
- [ ] Netlify deploy returns 200 on all data endpoints
- [ ] No console errors except favicon 404 (acceptable)
- [ ] Mobile drawer opens/closes without blocking map interaction
- [ ] Popup auto-dismisses when clicking sidebar controls
- [ ] Custom canvas icons render with correct border/color
- [ ] Case jitter separates overlapping points visually
