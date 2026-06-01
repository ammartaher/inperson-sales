# Phase 4 — Cluster + route + KML

Group targets by geography so the user can hit one area per trip. Build Google Maps routes per cluster and a KML for Google My Maps.

## Step 1 — Cluster

Match the user's clustering preference from `campaign.json`. Common approaches:

### By city / neighbourhood
Build a `clusterFor(address)` function — substring match on city names. **Order matters**: most-specific city first; the central city is the fallback last.

```ts
function clusterFor(addr: string | null): string | null {
  if (!addr) return null;
  const a = addr.toLowerCase();
  // Most-specific clusters first
  if (/\b(bunnik|houten|maurik)\b/.test(a))      return "bunnik-houten-maurik";
  if (/\b(zeist|driebergen|doorn)\b/.test(a))    return "zeist";
  if (/\b(amersfoort|bunschoten|woudenberg)\b/.test(a)) return "amersfoort-region";
  // ... more clusters ...
  if (/\b(utrecht|vleuten)\b/.test(a))           return "utrecht-stad";  // fallback
  return null;  // = not in the campaign region
}
```

### By postcode prefix
For countries with structured postcodes (NL 4-digit, US 5-digit), prefix-range matching is cleaner:
```ts
function clusterFor(addr: string): string | null {
  const m = addr.match(/\b(\d{4})\s*[A-Z]{2}\b/i);   // NL example
  if (!m) return null;
  const pc = +m[1];
  if (pc >= 3400 && pc <= 3499) return "nieuwegein-houten";
  if (pc >= 3500 && pc <= 3589) return "utrecht-stad";
  // ...
  return null;
}
```

**Tag each company** in `companies.json` with its cluster slug. **Re-organise `out/`** into `out/<cluster>/<slug>.<ext>` (move the existing flat files into subfolders).

## Step 2 — Google Maps route URL per cluster

**Use the path-segment format only** — pipe-form (`?api=1&waypoints=…|…`) breaks markdown link parsers in chat and some browsers:

```
https://www.google.com/maps/dir/<addr1>/<addr2>/<addr3>/.../
```

- Each address: URL-encode spaces as `+` (or `%20`). Leave commas as-is — Google handles them.
- **Stop limit**: ~9 stops per URL works reliably. For clusters with more, **split into A/B sub-routes** and produce two URLs.

## Step 3 — TSP optimisation (for clusters ≤ ~8 stops)

Brute-force the shortest Hamiltonian path through a cluster's stops:

1. Get approximate lat/lng for each stop. **Postcode centroids** work well for countries with precise postcodes (NL 4-digit ≈ 1km accuracy). For others, geocode via Nominatim (free, rate-limited 1/sec) or a Google Geocoding API call.
2. Distance matrix: Euclidean with longitude correction:
   ```
   dx = (lng2 - lng1) * 111 * cos(lat_rad)
   dy = (lat2 - lat1) * 111
   dist = sqrt(dx² + dy²)
   ```
3. `n!` permutations (8! = 40 320; instant). For each perm, sum pairwise distances. Keep the min.
4. The min-distance permutation = the optimal one-way order. Build the URL with stops in that order.

For clusters > 8: skip TSP. Use natural address order or alphabetical. Tell the user: *"Open this URL in **Google Maps on mobile** → tap the three dots → **Reorder stops** to auto-optimise. That feature is mobile-only."*

## Step 4 — KML for Google My Maps

Generate `<project>/inperson/pins.kml` with one `<Folder>` per cluster and one `<Placemark>` per target:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<kml xmlns="http://www.opengis.net/kml/2.2">
<Document>
  <name>[Campaign name] — targets</name>
  <description>N targets across M clusters.</description>
  <Folder>
    <name>[Cluster name]</name>
    <Placemark>
      <name>[Company]</name>
      <address>[full address], [Country]</address>
      <description>[phone] • [email] • [website]</description>
    </Placemark>
    <!-- one Placemark per target in this cluster -->
  </Folder>
  <!-- one Folder per cluster -->
</Document>
</kml>
```

- XML-escape `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;` in names/descriptions.
- Address-only is fine (Google geocodes on import). Coords optional.

**Recipe to give the user**:
1. Open https://www.google.com/maps/d/ (My Maps).
2. **+ Create a new map**.
3. Layer panel → **Import** → upload `pins.kml`.
4. *"Choose columns to position your placemarks"* → tick **address**. *"Choose a column to title your markers"* → **Name**. Done.
5. (Optional) Right-click each cluster layer → change colour for visual differentiation.
6. **On phone**: Google Maps app → **Saved** → **Maps** → your custom map appears with all pins.

## Step 5 — clusters.html index

Generate `<project>/inperson/clusters.html`. One `<section>` per cluster:

```html
<section style="margin:30px 0;padding:20px;border:1px solid #ddd;border-radius:6px;">
  <h2>[Cluster name] <span style="font-weight:normal;color:#888;font-size:14px;">([N] stops)</span></h2>
  <p><a href="[google-maps-url]" target="_blank"
        style="background:#15803d;color:#fff;padding:10px 18px;border-radius:4px;text-decoration:none;font-size:16px;">
    Open this cluster in Google Maps
  </a></p>
  <ol>
    <li><b>[Company]</b><br>
        <span style="color:#555;">[Address] — [phone]</span> · 
        <a href="./out/[cluster]/[slug].pdf">[slug].pdf</a></li>
    <!-- one li per target -->
  </ol>
</section>
```

Top of the page: a link to download `pins.kml` and a one-line "import to Google My Maps to get all pins on one map" hint.

## Use `.html`, never `.url`

Windows `.url` Internet Shortcut files often open as plain text in Chrome (file-association quirks). For one-click route opens, use a tiny HTML with auto-refresh instead:

```html
<!DOCTYPE html><html><head>
<meta charset="utf-8">
<meta http-equiv="refresh" content="0; url=[the google maps URL]">
<title>Open route</title>
</head><body style="font-family:sans-serif;padding:40px;text-align:center;">
<h1>Opening Google Maps…</h1>
<p><a href="[the URL]">Click here if it doesn't redirect</a></p>
</body></html>
```

## After route

Open [package.md](package.md) — phase 5.
