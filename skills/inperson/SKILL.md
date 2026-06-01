---
name: inperson
description: Plan and execute a physical/in-person cold-outreach campaign for a B2B SaaS. Walks the user through intake (product, ICP, region, salesperson contact) → target research → per-target artefact customisation (rebrand the SaaS's own output to look like the target's branding) → geographic clustering with Google Maps routes + Google My Maps KML → packaged folder ready to print and drive. Trigger phrases include "physical outreach", "in-person cold outreach", "sales blitz", "door-to-door with [product]", "hand out branded samples", "walk in with [offertes/proposals/samples]", "route around contractors", "print and deliver".
---

# inperson — physical cold-outreach campaign generator

Run this when the user wants to set up a **physical/in-person cold-outreach** campaign for a B2B SaaS — walking into prospect offices with branded printed artefacts (offertes / proposals / sample reports), driving a route around them, handing them over in person.

## When to use

Invoke when the user says any of:
- "I want to do a physical outreach campaign / sales blitz / door-to-door"
- "I'll walk in with offertes/proposals/samples"
- "Print and hand out branded [PDFs / brochures]"
- "Plan a route around [N] prospects in [area]"
- "Generate per-target [artefacts] for in-person delivery"

**Skip** when:
- The user wants pure cold email or LinkedIn DMs — this skill is about IRL artefacts + driving routes.
- The user wants generic outreach strategy advice without the artefact-generation + routing components.

## What this skill produces

By the end, the user has, in `<project>/inperson/`:
- `companies.json` — canonical target list (name, address, phone, email, website, contact, biz-reg, trigger event, cluster).
- `out/<cluster>/<slug>.<ext>` — one branded artefact per target, grouped by geographic cluster.
- `clusters.html` — one-page index: cluster names, "Open in Google Maps" route button per cluster, list of stops + links to the artefacts.
- `pins.kml` — Google My Maps import (one folder/layer per cluster, one pin per target).
- A folder ready to print + drive.

## The five phases

Walk the user through them in order. Each lives in its own reference file in this skill folder — open it before starting that phase.

1. **Intake** → [intake.md](intake.md). Ask the diagnostic questions to learn the campaign. Save to `<project>/inperson/campaign.json`.
2. **Targets** → [targets.md](targets.md). Find prospects via web search, CSV ingest, MCP-connected CRMs. Output: `companies.json`.
3. **Customise** → [customize.md](customize.md). Discover the SaaS's per-customer template; build a per-target branding pack; write the rebrand engine script that generates one artefact per target.
4. **Route** → [route.md](route.md). Cluster targets by area, build Google Maps route URLs per cluster, generate `pins.kml`, generate `clusters.html`.
5. **Package** → [package.md](package.md). Final folder layout, gitignore, commit conventions, handoff message.

## Output location convention

Default: `<current-repo>/inperson/` (a folder named `inperson` at the root of the user's current project). If not in a repo, use `<current-working-dir>/inperson/`.

Override only if the user says so during intake.

## Re-entry (when re-invoked on an existing campaign)

If `<project>/inperson/companies.json` already exists, **don't start over**. Tell the user what's already there and ask:
- "Fresh campaign — start over from intake."
- "Add more targets to the existing list."
- "Skip ahead to phase X — regenerate artefacts / re-cluster / re-route."

## Safety rules (always-on)

Non-negotiable. Surface them when relevant; push back if the user tries to bypass.

1. **Dev tenant only.** Before any write to the SaaS tenant's template/settings, confirm the tenant/org-ID is internal/dev (NOT a production customer). Read the repo's CLAUDE.md or env config if needed.
2. **Backup before any state write.** First time the script touches the tenant's template column, dump the current value to `_backup-<orgid>.json` in `<project>/inperson/`. Never overwrite an existing backup.
3. **Restore at end.** Every script run ends by restoring the original template from the backup, unless the user explicitly passes `--keep`. Provide a `--restore` flag that restores without doing anything else.
4. **Strictly sequential.** Per-tenant template columns are not versioned in any SaaS we've seen. Parallel rebrands bleed branding between targets. The rebrand loop is `for…await…`, never `Promise.all`.
5. **Don't auto-deliver.** Generated artefacts go to `out/`. Don't email or upload them anywhere — the user prints and hands them over in person.

## Honest things to be upfront about

- **~15% of B2B sites** have logos that can't be auto-fetched (inline base64 in HTML, white-only on transparent, behind a CDN with weird query params). Render the artefact without a logo and flag those targets — don't block the run.
- **Google Maps web UI doesn't have "Reorder stops"** — that feature is mobile-only. For clusters you can't TSP-optimise (>8 stops), generate the URL and tell the user to use the app's reorder feature there.
- **Windows file locks** — if you delivered a PDF via the chat preview earlier in the session and try to regenerate it, the file is locked. Either tolerate "kept previous version" or rename-old-then-write. Mention this to the user when it happens.
- **Real-customer CTA gate** — if you add a "made by us, call X" CTA to the SaaS's template (recommended), gate it behind an opt-in field. It must be invisible for actual paying customers of the SaaS sending their own artefacts to their own clients.

## Output format hints

When invoked, use TaskCreate to track the phases as tasks ("Phase 1 — intake", "Phase 2 — targets", etc.) so the user can see progress. Mark each task in_progress before working on it and completed when its outputs (the named files) exist.
