# lpi-enrollment-widgets

Client-side widgets for the **Lion's Pride Insurance (LPI Advising)** GoHighLevel pre-enrollment form. Each widget is an independent embed: drop its `<div>` + `<script>` into a GHL Custom Code block, it collects structured data and writes it into GHL custom fields for HealthSherpa handoff.

**This repo is intentionally public.** The code runs entirely in the browser and is visible via View Source on any page that embeds it. There are no secrets, API keys, or PHI here — every widget writes straight into GHL's own custom fields. Do not commit anything you would not want indexed by Google.

> Forked from [`maxmethod/doc-rx-lookup`](https://github.com/maxmethod/doc-rx-lookup) — this repo is the LPI-specific isolation boundary so account-specific fields never ship to the shared CDN that other subaccounts pull from.

## Widgets — three independent embeds

| Widget | File | Embed | Container id | GHL fields written |
| --- | --- | --- | --- | --- |
| **Medications** | `medications-lookup.html` | `dist/embed-medications.js` | `medications-lookup-widget` | `medications_json` · `medications` (summary → existing `🏥 Prescriptions`) |
| **Doctors / Providers** | `provider-lookup.html` | `dist/embed-providers.js` | `provider-lookup-widget` | `providers_json` · `doctors` (summary → existing `🏥 Doctors`) |
| **Current Coverage** | `current-coverage-lookup.html` | `dist/embed-coverage.js` | `coverage-lookup-widget` | `current_coverage_json` · `current_coverage_summary` |

Medications and Doctors are **separate embeds** — place them in the same code block, different blocks, or different form pages. Each uses a distinct container id / global / load-guard, so any combination can coexist on one page.

## Brand color

All three default to the LPI teal **`rgb(97, 163, 183)` (`#61a3b7`)** for buttons and accents. The embed snippets also pass `data-primary-color="rgb(97, 163, 183)"` to **lock** it (so GHL's gold brand value doesn't override it). To use a different color on a page, change that attribute, or set `window.<WIDGET>_CONFIG = { primaryColor: '...' }` before the embed loads.

## GHL embed snippets

Paste each into a Custom Code / Custom HTML block. The widget can sit **directly inside the form** (it then shares the DOM with the form's custom fields — the cleanest placement) or alongside the form on the same page. Either way the widget finds the fields by `name=` and GHL's `data-q=` and keeps them synced.

**Medications**
```html
<div id="medications-lookup-widget" data-primary-color="rgb(97, 163, 183)"></div>
<script src="https://cdn.jsdelivr.net/gh/maxmethod/lpi-enrollment-widgets@v1.3.0/dist/embed-medications.js"></script>
```

**Doctors / Providers**
```html
<div id="provider-lookup-widget" data-primary-color="rgb(97, 163, 183)"></div>
<script src="https://cdn.jsdelivr.net/gh/maxmethod/lpi-enrollment-widgets@v1.3.0/dist/embed-providers.js"></script>
```

**Current Coverage**
```html
<div id="coverage-lookup-widget" data-primary-color="rgb(97, 163, 183)"></div>
<script src="https://cdn.jsdelivr.net/gh/maxmethod/lpi-enrollment-widgets@v1.3.0/dist/embed-coverage.js"></script>
```

> The form's matching custom fields must be present on the same rendered page (hidden inputs are fine). Note: GHL's in-builder **preview** may not run `<script>` — test on the **published** form.

## Build

The `dist/*.js` embeds are **build artifacts** — never hand-edit them. After any change to a widget's HTML:

```bash
node scripts/build-embed.js                 # rebuild all widgets
node scripts/build-embed.js medications     # or just one (medications | providers | coverage)
```

The build scopes each widget's CSS to its container id (doubled for specificity, no `!important`) and inlines markup + logic into a self-bootstrapping IIFE.

## ZIP dataset (Doctors/Providers widget only)

The provider radius filter needs ZIP coordinates. To avoid duplicating a 1.2 MB blob, this repo **reuses the dataset hosted by the OG tool**:

```
https://cdn.jsdelivr.net/gh/maxmethod/doc-rx-lookup@v1.0.9/dist/us-zips.json
```

(set in `ZIP_DATASET_URL` inside `provider-lookup.html`). On `localhost` it auto-resolves to a local `./dist/us-zips.json` if present. No need to commit the dataset here.

## Field mapping note — match by GHL field ID (important gotcha)

**On a live GHL form, the custom-field input's `name=` is the GHL field ID; its `data-q` is a slug of the field's *label*, which GHL caches inside the form when the field is added.** Consequences (learned the hard way, 2026-06-11):
- Renaming the global custom field does **not** update the form's `data-q`.
- Any punctuation/extra words in the label leak into `data-q` — a field labeled "… (widget)" rendered `data-q="current_coverage_json_(widget)"`, so matching by a clean key never connected.

So each widget targets its destination field by **both** the clean key (standalone/testing placeholders) **and** the stable **GHL field ID** (what actually connects on the live form). `setAll()` accepts an array. Field IDs verified on the live LPI location:

| Output | Clean key | GHL field ID | GHL field |
| --- | --- | --- | --- |
| meds JSON | `medications_json` | `9Ntdoe7rPRtvesiIpwet` | Medications Json |
| meds summary | `medications` | `bfueEINiKDtEIP5CdE98` | 🏥 Prescriptions |
| providers JSON | `providers_json` | `aAWx4APM5udep77XC9gw` | Providers Json |
| providers summary | `doctors` | `kn0LprMsJXOzXEGY9KXY` | 🏥 Doctors |
| coverage JSON | `current_coverage_json` | `Cuj5yUIHcBAuqcL7y9oG` | Current Coverage Json |
| coverage summary | `current_coverage_summary` | `Fuz39fEJ4mGL6ZKuZbQC` | Current Coverage Summary |

All six destination fields are **LARGE_TEXT** (do not confuse `🏥 Prescriptions`/`🏥 Doctors` with the single-line `Prescription List`/`Provider List` legacy fields). To retarget, edit `FIELD_KEYS` at the top of a widget, or override per-page via `window.<WIDGET>_CONFIG = { fieldKeys: { <output>: ['<key>', '<fieldId>'] } }`. Get a field's ID from `GET /locations/{loc}/customFields` (see `scripts/create_widget_fields.js`).

## Versioning & deploys

jsDelivr serves **immutable content per git tag** and caches `@main` for up to 12h. **Always deploy by tag:**

1. Commit changes to `main`.
2. `git tag vX.Y.Z && git push origin vX.Y.Z`.
3. Bump the `@vX.Y.Z` in the GHL embed snippets (above).

See `BUILD-SPEC.md` for the full architecture, the Income & Assets widget spec, and the GHL integration contract.
