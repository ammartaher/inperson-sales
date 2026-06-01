# Phase 5 — Package + handoff

Final folder layout. Commit the right files. Hand off to the user with a clear next-action message.

## Standard layout

```
<project>/inperson/
├── campaign.json                # tracked — config from intake
├── companies.json               # tracked — canonical target list
├── clusters.html                # tracked — one-page index
├── pins.kml                     # tracked — Google My Maps import
├── rebrand.<ts|mjs|py>          # tracked — the engine script
├── .gitignore                   # tracked — ignores out/ and _backup-*
├── out/                         # gitignored — generated artefacts
│   ├── <cluster-1>/
│   │   ├── <slug>.<ext>
│   │   └── ...
│   └── <cluster-2>/
│       └── ...
└── _backup-<tenantid>.json      # gitignored — tenant template safety net
```

## .gitignore contents

```
out/
_backup-*.json
```

## What to commit and why

**Track**:
- `campaign.json`, `companies.json` — small, valuable, the canonical campaign state.
- `clusters.html`, `pins.kml` — generated but small + useful artefacts; committing them means future sessions (including claude.ai/code on the web) inherit a usable starting state.
- `rebrand.<ext>` — the engine script; obviously code.
- `.gitignore` — guards the rest.

**Don't track**:
- `out/*` — generated, regeneratable by re-running `rebrand`. Can be large and binary.
- `_backup-*.json` — contains tenant template state which may include sensitive config (logos, internal addresses, etc).

## Optional Downloads mirror

**Only if the user asks.** Copy `out/` into `<HOME>/Downloads/<campaign-name>/` for one-click access from File Explorer.

**Warn about Windows file locks** when you mirror: the chat preview holds handles on PDFs you've delivered. Re-running the engine may fail to overwrite locked files with `EBUSY`. Workarounds:
- Rename the old file before regenerating: `Move-Item old.pdf old.locked.pdf` then run the engine, then `Remove-Item *.locked.pdf`.
- Or just tolerate the partial failure ("kept previous version for X") and tell the user.

Also note: bulk `Remove-Item Downloads\*` with wildcards is blocked by the sandbox in many setups. Use specific filenames or `Copy-Item -Force` to overwrite instead of deleting first.

## Handoff message

End the campaign run with a single concise message to the user, like:

> Campaign packaged at `<project>/inperson/`.
>
> - **`clusters.html`** — double-click to open the index. One big green button per cluster opens that cluster's route in Google Maps.
> - **`pins.kml`** — import to https://www.google.com/maps/d/ (+ Create a new map → Import → upload). Gives you one custom map with all pins, one layer per cluster, accessible in Google Maps app under Your Places → Maps.
> - **`out/<cluster>/`** — print-ready artefacts, one per target. Print the cluster you're driving today.
> - [N] firms had no logo on their site — flagged in `companies.json` (`logo_collected: false`). Their artefacts render without a logo but are otherwise complete.
> - Pushed to `<branch>` (`<commit-sha>`) — re-runnable from a web Claude Code session.
>
> Optional: when you want to publish the campaign template as a Claude Code plugin so others can install via marketplace, the manifest scaffold is in this skill's docs.

## Sanity checklist before saying "done"

- [ ] `campaign.json` exists with all 10 intake answers filled in.
- [ ] `companies.json` has ≥ 1 target with full schema (`logo_base64`, address, phone, …).
- [ ] `out/<cluster>/<slug>.<ext>` exists for at least one target.
- [ ] You verified ONE artefact visually (read the PDF / open the image) — branding right, sample data intact, CTA renders.
- [ ] `clusters.html` opens in a browser → clicking the cluster button opens Google Maps with that cluster's stops.
- [ ] `pins.kml` import to My Maps creates a map with pins in the right cluster folders.
- [ ] The tenant template is restored to its original (check the dev tenant's settings — should match the `_backup-*.json` contents).
- [ ] `.gitignore` keeps `out/` and `_backup-*` out of any commits.
