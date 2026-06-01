# Phase 1 — Intake

Ask the questions below in order. Don't proceed to phase 2 until you have answers (or explicit "skip" markers) for each.

Save the answers to `<project>/inperson/campaign.json` so future phases (and re-invocations) can read them.

## campaign.json schema

```json
{
  "saas":        { "name": "...", "oneLiner": "...", "outputDescription": "..." },
  "icp":         { "industry": "...", "sizeBand": "...", "geo": "...", "triggers": ["..."] },
  "salesperson": { "name": "...", "phone": "+...", "url": "https://..." },
  "region":      { "centre": "City, Country", "radius": "30 min" },
  "artefactFormat": "printed PDF | brochure | USB | demo unit | ...",
  "rebrand": {
    "perCustomerOutput": true,
    "templateLocation": "DB JSON column at X / file template at Y / API endpoint Z",
    "renderFunction":   "where in the code the artefact is rendered, if known",
    "sampleArtefactId": "a real artefact ID we can reuse as the canvas",
    "tenantId":         "the dev tenant/org-ID we'll write into"
  },
  "targetSources":   ["CSV: /path/to/file.csv", "Notion: db-id", "fresh research"],
  "clustering":      "city | neighbourhood | postcode-prefix | road-corridor",
  "outputLocation":  "<project>/inperson/",
  "devTenantConfirmed": true
}
```

## The questions

1. **The SaaS — one-liner.** What does it do? What does its public output look like? (e.g. "AI office for Dutch construction contractors — generates offertes from RFQ emails"; "data observability platform — generates incident reports"; "telehealth scheduler — generates appointment confirmations".)

2. **ICP — who you're targeting.** Industry, employee band (e.g. 10–50), geographic scope. Any **behavioural triggers** that signal pain *right now*? (e.g. "currently hiring a calculator" implies their estimating pipeline is overflowing — perfect moment.) Triggers double the conversion of cold visits — ask for them.

3. **The salesperson.** Name + phone (E.164 with `+`) + landing URL. These go onto every artefact's CTA so prospects can call after.

4. **Region.** A city + driving radius (e.g. "Utrecht, 30 min") or a province / metropolitan area. The skill will geo-filter targets to this and build the routes inside it.

5. **Artefact format.** What gets printed/handed over? Most common: **printed PDF** (offertes, proposals, sample reports). Could also be: branded brochure, USB stick, demo unit, swag bundle. The skill assumes "something a single per-target file can produce" — for non-file artefacts ask the user how to script the per-target customisation.

6. **Per-customer rebrand-ability — the most important question.**
   - Does the SaaS already render per-customer output (with the customer's logo/branding)?
   - **Where does that template live?** Common shapes: a JSON column in a tenants table (e.g. `organizations.settings.pdf_template`), a file template (Handlebars/Puppeteer HTML), an image overlay (Sharp/ImageMagick on a base PNG), a Figma file + API, a runtime render endpoint behind auth.
   - **Can the user supply ONE sample artefact** they've already generated (e.g. a quote-ID + tenant-ID)? This is the canvas we'll clone N times.

7. **Target sources.** Do they already have a CSV / Notion DB / CRM export of prospects? Path or DB ID. Otherwise we research from scratch (phase 2 covers both paths).

8. **Clustering preference.** When grouping targets so the user can hit one area per trip — by **city**, **neighbourhood**, **postcode prefix**, or **road corridor**? If unsure, default to **city** (or postcode prefix in dense urban areas).

9. **Output location.** Default: `<current-project>/inperson/`. Override only if they want elsewhere (rare).

10. **Dev-vs-prod gate.** Confirm the tenant/org-ID they'll supply is **internal/dev**, not a production customer. The rebrand engine writes to this tenant's template column. **Never** do this on prod data. Read the repo's CLAUDE.md if there's any ambiguity about which env an ID belongs to.

## After intake

- Write `<project>/inperson/campaign.json` with the answers.
- Print a one-paragraph recap and ask the user to confirm.
- Then open [targets.md](targets.md) for phase 2.
