# inperson

A Claude Code skill that plans and executes a **physical / in-person cold-outreach campaign** for a B2B SaaS.

You walk into prospect offices with **branded printed artefacts** (offertes, proposals, sample reports) that look like they came out of *that prospect's own* account on your SaaS, then drive a Google Maps route around them clustered by neighbourhood.

Invoke with `/inperson` (personal install) or `/inperson:inperson` (plugin install).

## What it does

Given:
- Your SaaS (one-liner + where its per-customer template lives in your codebase).
- Your ICP (industry, size, region, behavioural triggers).
- Your salesperson contact (name, phone, landing URL).
- One real sample artefact from your SaaS (e.g. a quote ID + tenant ID — must be a **dev tenant**).

The skill produces, in `<your-project>/inperson/`:
- `companies.json` — canonical list of 20–50 targets (researched from web + CSVs + connected CRMs).
- `out/<cluster>/<slug>.<ext>` — one rebranded artefact per target, grouped by area.
- `clusters.html` — one-page index. Each cluster has a green button that opens its Google Maps route.
- `pins.kml` — import to [Google My Maps](https://www.google.com/maps/d/) for one custom map with all targets as pins, colour-coded by cluster, accessible from the Google Maps app on your phone.

You print the PDFs for one cluster, open that cluster's route, drive, hand them over.

## The five phases

1. **Intake** — 10 diagnostic questions to learn the campaign.
2. **Targets** — find prospects via web research, CSV ingest, or connected MCPs (Notion / Linear / HubSpot).
3. **Customise** — discover your SaaS's per-customer template, build a per-target branding pack (logo, accent colour, company details), write a rebrand engine script.
4. **Route** — cluster by area, TSP-optimise routes ≤8 stops, build Google Maps URLs and KML.
5. **Package** — final folder layout, gitignore, handoff message.

Full details in [`skills/inperson/SKILL.md`](skills/inperson/SKILL.md) and the phase files alongside it.

## Install

### Personal use (zero-effort)

Copy the skill into your user-scope skills directory:

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/inperson
cp -r skills/inperson/* ~/.claude/skills/inperson/

# Windows PowerShell
New-Item -ItemType Directory -Force "$HOME\.claude\skills\inperson" | Out-Null
Copy-Item .\skills\inperson\* "$HOME\.claude\skills\inperson\" -Force
```

Then in any Claude Code session: `/inperson` — Claude walks you through it.

### As a plugin (for sharing across team)

This repo is already a valid Claude Code plugin (has `.claude-plugin/plugin.json` + `skills/`).

Run Claude Code with `--plugin-dir` pointing at this repo:

```bash
claude --plugin-dir /path/to/inperson-plugin
```

Then `/inperson:inperson` becomes available in every session.

### Via the Claude marketplace (public)

Submit this GitHub repo at <https://claude.ai/settings/plugins/submit>. Once approved, anyone can install via marketplace.

## Honest things to know

- **~15% of B2B sites** have logos that can't be auto-fetched (inline base64, white-only on transparent, behind a CDN with weird query params). The skill renders the artefact without a logo and flags those targets — it never blocks the run.
- **Strictly sequential rebrands.** Per-tenant template columns aren't versioned in any SaaS we've seen — parallel rebrands bleed branding between targets. The engine loop is `for…await…`, never `Promise.all`.
- **Always restores tenant state.** The engine backs up the tenant's template *once* before any write, and restores at the end. There's a `--restore` flag if anything goes sideways.
- **Dev tenant only.** The skill confirms the tenant/org-ID you supply is internal/dev before touching it. Never run this on a production customer's data.
- **Google Maps "Reorder stops" is mobile-only** on the consumer Google Maps web UI. For clusters > 8 stops where TSP brute-force isn't applied, you reorder in the Maps app on your phone.
- **`.html` over `.url`** for one-click route opens — Windows `.url` Internet Shortcuts often open as plain text in Chrome under common file-association states.
- **Real-customer CTA gate.** If you add a "made by us, call X for a demo" CTA to your SaaS's template (recommended), gate it behind an opt-in field so real customers' artefacts never carry it.

## Origin

Distilled from a 9-day revenue sprint where a founder generated 39 per-target branded artefacts for prospects in one Dutch province, clustered them into 8 driving routes, and packaged everything for a Monday-morning blitz. The methodology turned out general enough for any B2B SaaS doing in-person cold outreach, so we extracted it into this skill.

## Author

Built by **[Ammar Taher](https://www.linkedin.com/in/amtaher)**. Reach out on LinkedIn if you adapt this for your SaaS or want to swap notes on physical outreach.

## License

MIT — see [LICENSE](LICENSE).
