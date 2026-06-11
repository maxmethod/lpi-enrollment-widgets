# lpi-enrollment-widgets

Client-side widgets for the **Lion's Pride Insurance (LPI Advising)** GoHighLevel pre-enrollment form. Each widget embeds in a GHL funnel page (Custom Code block), collects structured data, and writes it into GHL custom fields for HealthSherpa handoff.

**This repo is intentionally public.** The code runs entirely in the browser and is visible via View Source on any page that embeds it. There are no secrets, API keys, or PHI here — every widget writes straight into GHL's own custom fields. Do not commit anything you would not want indexed by Google.

> Forked from [`maxmethod/doc-rx-lookup`](https://github.com/maxmethod/doc-rx-lookup) — this repo is the LPI-specific isolation boundary so account-specific fields never ship to the shared CDN that other subaccounts pull from.

## Widgets

| Widget | File | Embed | GHL fields written |
| --- | --- | --- | --- |
| **Rx + Provider lookup** | `rx-provider-lookup.html` | `dist/embed.js` | `medications_json`, `providers_json` (JSON) · `medications`, `doctors` (summaries → existing `🏥 Prescriptions` / `🏥 Doctors`) |
| **Current Coverage list** | `current-coverage-lookup.html` | `dist/embed-coverage.js` | `current_coverage_json`, `current_coverage_summary` |

## GHL embed snippets

Paste into a Custom Code / Custom HTML block on the funnel page where each widget should appear. The `data-primary-color` template var lets GHL substitute the brand color server-side.

**Rx + Provider lookup**
```html
<div id="rx-lookup-widget" data-primary-color="{{ custom_values.brand_primary_color }}"></div>
<script src="https://cdn.jsdelivr.net/gh/maxmethod/lpi-enrollment-widgets@v1.0.0/dist/embed.js"></script>
```

**Current Coverage list**
```html
<div id="coverage-lookup-widget" data-primary-color="{{ custom_values.brand_primary_color }}"></div>
<script src="https://cdn.jsdelivr.net/gh/maxmethod/lpi-enrollment-widgets@v1.0.0/dist/embed-coverage.js"></script>
```

On the same page, the GHL form must contain the matching custom fields (as hidden inputs is fine). The widget finds them by `name=` **and** GHL's `data-q=` and keeps them synced as the user makes selections. Both widgets use distinct container ids / globals, so they can live on the same page.

## Build

The `dist/*.js` embeds are **build artifacts** — never hand-edit them. After any change to a widget's HTML:

```bash
node scripts/build-embed.js            # rebuild all widgets
node scripts/build-embed.js rx         # or just one
node scripts/build-embed.js coverage
```

The build scopes each widget's CSS to its container id (doubled for specificity, no `!important`) and inlines markup + logic into a self-bootstrapping IIFE.

## ZIP dataset (Rx widget only)

The provider radius filter needs ZIP coordinates. To avoid duplicating a 1.2 MB blob, this repo **reuses the dataset already hosted by the OG tool**:

```
https://cdn.jsdelivr.net/gh/maxmethod/doc-rx-lookup@v1.0.9/dist/us-zips.json
```

(set in `ZIP_DATASET_URL` inside `rx-provider-lookup.html`). On `localhost` it auto-resolves to a local `./dist/us-zips.json` if present. No need to commit the dataset here.

## Versioning & deploys

jsDelivr serves **immutable content per git tag** and caches `@main` for up to 12h. **Always deploy by tag:**

1. Commit changes to `main`.
2. `git tag vX.Y.Z && git push origin vX.Y.Z`.
3. Bump the `@vX.Y.Z` in the GHL embed snippets (above).

## Field mapping note

`🏥 Prescriptions` resolves to GHL key **`medications`** (not `prescriptions`) and `🏥 Doctors` to **`doctors`** on the live LPI location — verified 2026-06-11. These are configured in the `FIELD_KEYS` const at the top of `rx-provider-lookup.html`; override per-page via `window.RX_CONFIG = { fieldKeys: {...} }` before the embed loads.

See `BUILD-SPEC.md` for the full architecture, the third (Income & Assets) widget spec, and the GHL integration contract.
