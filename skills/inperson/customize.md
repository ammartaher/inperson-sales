# Phase 3 — Customise the artefact (the heart)

For each target in `companies.json`, generate one artefact (PDF / image / report) that looks like it came out of **that target's own** account on the SaaS — their logo, their brand colour, their company name/address/phone/etc. Then append an opt-in CTA pointing to the salesperson.

This is the most product-specific phase — the SaaS dictates the shape. Read the codebase first.

## Step 1 — Discover the template

Open the SaaS codebase the user pointed at (from `campaign.json`'s `rebrand.templateLocation`). Find:

- **Where customer branding is stored**: which DB column / file path / API endpoint holds the per-customer rendering template? Look for shapes like:
  - JSON column in a tenants/orgs table (e.g. `organizations.settings.pdf_template`).
  - Per-org config files (e.g. `tenants/<id>/branding.json`).
  - Per-customer CSS / SCSS theme files.
  - Figma file + Figma API.
  - Runtime render endpoint (e.g. `POST /api/render`) that takes a template payload.
- **What fields it has**: logo (base64? URL?), accent colour (hex?), company name/address/phone/email/website/biz-reg, header title, footer text, payment terms, …
- **How the artefact gets rendered**: which function takes the template + a sample data ID and returns the file? (e.g. `generateQuotePdf(quoteId)` in pdfmake, or a Puppeteer HTML route, or an image-overlay script using Sharp.)
- **Whether render runs standalone** without the full app server. If it does (just needs DB credentials), great — we script it. If not, fall back to the HTTP render endpoint via a local auth token.

Read 1–3 relevant source files and confirm: (a) template field shape, (b) the render function path, (c) whether it's standalone. Update `campaign.json.rebrand.renderFunction` with the verified path.

## Step 2 — Build a per-target branding pack

For each target in `companies.json`, populate a `pdf_template` (or whatever the SaaS calls the template object) with:

### Logo
1. **Find the source URL** — WebFetch the target's website, look for `<img>` with `logo` in `class`/`src`, or the `og:image` meta tag.
2. **Download + resize** with `sharp`: `sharp(buf).resize(MAX, MAX, { fit: 'inside', withoutEnlargement: true }).png().toBuffer()` where `MAX` is the SaaS template's max dimension (commonly 512 or 360).
3. **Base64-encode** as `data:image/png;base64,…`.
4. **Persist** in the company's `logo_base64`. Set `logo_collected: true`.

**Logo gap tolerance**: if any step fails (404, inline base64 on the source HTML with no file URL, white-on-transparent that'd be invisible, etc.), don't block — leave `logo_base64` empty and set `logo_collected: false` with a `notes: "site uses inline base64 logo — manual fetch needed"`. The artefact will render without a logo; still useful.

### Accent colour
Derive from the logo's dominant colour:
```ts
const { dominant } = await sharp(input).stats();   // {r, g, b}
const lum = (0.299 * dominant.r + 0.587 * dominant.g + 0.114 * dominant.b) / 255;
const hex = `#${[dominant.r, dominant.g, dominant.b].map(n => Math.round(n).toString(16).padStart(2, '0')).join('')}`;
const accent = lum > 0.82 ? "#1f2937" : hex;       // luminance guard: near-white falls back to slate
```

### Company fields
Copy from `companies.json` straight into the template's named fields (`company_name`, `company_address`, `company_phone`, `company_email`, `company_website`, `company_kvk`/`company_btw`/etc.).

## Step 3 — Add an opt-in CTA template field (one-time SaaS code change)

This is a small, additive change to the SaaS's template renderer to support a "made by us, call X for a demo" block on the artefact. **It must be opt-in via a template field** so real customers' artefacts never carry it.

### Interface change
Add an optional field to the template type:
```ts
inperson_cta?: {
  text?: string;
  phone?: string;
  url?: string;
  name?: string;        // e.g. "Bel Ammar +3163..." rather than just "Bel +3163..."
  logo_base64?: string; // SaaS's own logo above the message
};
```

### Renderer change
At the end of the artefact body — after the totals / summary, before any per-page footer — conditional on `template.inperson_cta?.phone || template.inperson_cta?.url`, render:
1. A thin horizontal divider.
2. The SaaS's own logo (small, ~90px wide) above the message text.
3. The CTA text (e.g. "This [artefact] was made by [SaaS] in under 5 minutes from one of our users' real data. Want one based on yours? Call [Name] for a demo.").
4. A one-line `Bel <Name> <Phone> · <url>` with the phone as a `tel:` link and the URL clickable in the target's accent colour.

### Engine sets it
The rebrand script sets `inperson_cta` from `campaign.json`'s `salesperson` info, merged into every per-target template object. Real customers' templates never have this field → no CTA renders for them.

## Step 4 — Write the rebrand engine script

Place at `<project>/inperson/rebrand.<ts|mjs|py>` matching the SaaS's stack. Skeleton (TypeScript / pdfmake-style SaaS):

```ts
import "dotenv/config";
import fs from "fs";
import path from "path";
// import the SaaS's render function + DB client (paths vary)
// import sharp from "sharp"; // only if you size logos at runtime

const ROOT     = "<project>/inperson";
const CAMPAIGN = JSON.parse(fs.readFileSync(`${ROOT}/campaign.json`, "utf8"));
const CONFIG   = JSON.parse(fs.readFileSync(`${ROOT}/companies.json`, "utf8"));
const BACKUP   = `${ROOT}/_backup-${CONFIG.tenantId}.json`;
const OUT      = `${ROOT}/out`;

function assertDevTenant(id: string) {
  // implement a guard: read CLAUDE.md, server/.env, or a `prod_tenants.json` block-list
  // throw if `id` matches any known production tenant
}

async function main() {
  const args    = process.argv.slice(2);
  const only    = args.includes("--only")    ? args[args.indexOf("--only") + 1] : null;
  const keep    = args.includes("--keep");
  const restore = args.includes("--restore");

  assertDevTenant(CONFIG.tenantId);

  if (!fs.existsSync(BACKUP)) {
    const current = await getTenantTemplate(CONFIG.tenantId);
    fs.writeFileSync(BACKUP, JSON.stringify({ savedAt: new Date().toISOString(), template: current }, null, 2));
  }

  if (restore) {
    const { template } = JSON.parse(fs.readFileSync(BACKUP, "utf8"));
    await setTenantTemplate(CONFIG.tenantId, template);
    console.log("Restored.");
    return;
  }

  const targets = only ? CONFIG.companies.filter(c => c.slug === only) : CONFIG.companies;
  const cta     = { ...CAMPAIGN.salesperson }; // text/phone/url/name/logo_base64 prebuilt

  let ok = 0, fail = 0;
  for (const c of targets) {                            // STRICTLY SEQUENTIAL — never Promise.all
    try {
      const tmpl = { ...c.pdf_template, inperson_cta: cta };
      await setTenantTemplate(CONFIG.tenantId, tmpl);
      const { buffer, ext } = await renderArtefact(CAMPAIGN.rebrand.sampleArtefactId);
      const cluster = c.cluster ?? "unclustered";
      const outPath = `${OUT}/${cluster}/${c.slug}.${ext}`;
      fs.mkdirSync(path.dirname(outPath), { recursive: true });
      fs.writeFileSync(outPath, buffer);
      console.log(`  OK    ${c.company}  ->  ${outPath}`);
      ok++;
    } catch (err) {
      console.error(`  FAIL  ${c.company}: ${err instanceof Error ? err.message : err}`);
      fail++;
    }
  }

  if (!keep) {
    const { template } = JSON.parse(fs.readFileSync(BACKUP, "utf8"));
    await setTenantTemplate(CONFIG.tenantId, template);
    console.log("Restored original template.");
  }
  console.log(`Done. ${ok} ok, ${fail} fail.`);
}

main().catch(e => { console.error(e); process.exit(1); });
```

For non-TS stacks (Python, Ruby, …): same shape — config → dev guard → backup → sequential loop → restore.

## Step 5 — Verify with ONE target first

Always run with `--only <one-slug>` first. Read the generated artefact (use the PDF/image reader). Confirm:
- Logo appears (or is documented missing).
- Accent colour matches the target's brand.
- Company name, address, phone, email, website are the target's, NOT the dev tenant's.
- The salesperson CTA renders with the right phone + URL + clickable links.
- Real artefact content (line items, sample data) is intact.

If it looks right, run the full batch. If not, fix and retry.

## After customise

Open [route.md](route.md) — phase 4.
