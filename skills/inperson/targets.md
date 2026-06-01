# Phase 2 — Find targets

Build `<project>/inperson/companies.json` — the canonical target list. Every downstream phase reads from this one file.

## companies.json schema

```json
{
  "tenantId":         "<from campaign.json: the dev tenant>",
  "sampleArtefactId": "<from campaign.json: the canvas artefact>",
  "companies": [
    {
      "slug":            "kebab-case-unique-id",
      "company":         "Display name (legal-ish)",
      "logo_url":        "<source URL or '<paste'>",
      "logo_base64":     "<filled by branding step>",
      "accent_color":    "#hex (filled by branding step)",
      "address":         "Street + postcode + city",
      "city":            "City",
      "phone":           "+...",
      "email":           "info@...",
      "website":         "https://...",
      "linkedin":        "...",
      "contact_person":  "...",
      "biz_reg":         "KvK / VAT / EIN / etc",
      "trigger_event":   "currently hiring X / recent expansion / ...",
      "cluster":         "<filled by phase 4>",
      "logo_collected":  false,
      "notes":           ""
    }
  ]
}
```

## Sources, in priority order

### 1. CSV ingest (if user has a CSV)

LinkedIn export / Apollo / Clay / hand-curated CSV. Parse with a minimal quoted-field parser — handle:
- Quoted fields containing commas: `"Some, address"`.
- Embedded newlines inside quoted fields.
- Double-quote escapes inside quoted fields: `""`.

Mini Node parser (drop into a one-off script):

```js
function parseCsv(text) {
  const rows = [], row = [];
  let field = "", inQ = false;
  for (let i = 0; i < text.length; i++) {
    const c = text[i];
    if (inQ) {
      if (c === '"' && text[i+1] === '"') { field += '"'; i++; continue; }
      if (c === '"') { inQ = false; continue; }
      field += c;
    } else {
      if (c === '"') { inQ = true; continue; }
      if (c === ',') { row.push(field); field = ""; continue; }
      if (c === '\n') { row.push(field); rows.push([...row]); row.length = 0; field = ""; continue; }
      if (c === '\r') continue;
      field += c;
    }
  }
  if (field.length || row.length) { row.push(field); rows.push(row); }
  return rows;
}
```

Map CSV columns into the schema. Drop rows that don't pass the geographic + ICP filter (below).

### 2. Connected MCPs

Check what MCP servers the user has connected. Look for prospect data in:
- **Notion** (`notion-search`, `notion-fetch`) — existing prospect databases / boards.
- **Linear** (`list_issues`) — tickets tagged as leads.
- **HubSpot / Salesforce / Pipedrive** MCPs if connected.
- **Supabase / Postgres** MCPs — direct queries against the user's CRM tables.

### 3. Web research (when no list exists)

Spawn a **background research agent** (general-purpose or Explore) with a focused brief covering:
- The ICP details from `campaign.json` (industry, size, geography).
- **Vertical directories** for the industry — examples: Trustoo.nl, Yelp, Google Maps Places, the industry's trade-association directory, local Chamber of Commerce lists, niche aggregators.
- **LinkedIn job posts** as a trigger-event signal — filter by ICP industry + geography + job titles that imply the pain the SaaS solves. (e.g. for construction: "calculator", "werkvoorbereider", "projectleider". For data tools: "data engineer", "analytics engineer". For sales tools: "RevOps", "BDR manager".) **Hiring = capacity strain = perfect moment to pitch.**
- **Trade-show / event** websites — exhibitor lists are gold.
- **Per-target fields to collect** (the full schema above). **Never invent** phone numbers, emails, or biz-reg IDs — leave blank if not found.

Run the agent in the background while you start drafting the next phase. The agent should return a markdown table the user can eyeball before you commit it to `companies.json`.

## Filter + dedupe

- **Geographic predicate**: a `cityInRegion(addr)` helper. Build the set of accepted city names from the user's region answer + driving-radius reverse lookup (mention common towns within the radius). For dense urban / structured-postcode countries, prefix-range matching works too (e.g. NL postcodes `3400`–`3960` ≈ Utrecht province).
- **ICP predicate**: employee-band match (LinkedIn employee count or KVK / public registry data), industry tag, ICP traits from `campaign.json`.
- **Dedupe**: by website URL (canonicalise: strip `https?://`, `www.`, trailing slash; lowercase) OR by company name (lowercase trim). When dup found, **merge fields** (prefer non-empty values from either source).

## Output + summary

- Write everything to `<project>/inperson/companies.json`.
- Print a count summary: "32 targets total. Top 7 trigger-event signals: …" — those are the user's first-priority visits.
- Tag the strongest signals (e.g. `notes: "🔥 hot — hiring estimator"`) so phase 4 can sort the route by priority within a cluster.

## After targets

Open [customize.md](customize.md) — phase 3.
