# GHL Enrollment Widgets — Build Spec (subaccount fork)

> **Rename this project folder** to the subaccount (e.g. `ghl-enrollment-widgets-<subaccount>`).
> This is a **build-from-cold spec**: a fresh session with no other context can recreate both tools from this file alone.

## What this project is

A **standalone, self-contained** set of client-side widgets embedded into one GoHighLevel (GHL) subaccount's **enrollment form**. Three tools:

| Tool | What it does | Lookups? | Status |
| --- | --- | --- | --- |
| **A1. Medications** | Drug search (RxNorm), strength/form/frequency + "Who takes this?" applicant field. Cap 20. | Yes | Built ✓ (`medications-lookup.html`) |
| **A2. Doctors / Providers** | Doctor search (NLM NPI) with ZIP-radius filter. Cap 10. | Yes | Built ✓ (`provider-lookup.html`) |
| **B. Income & Assets table** | A multi-step, add-as-many-rows-as-you-need table for income sources and assets. Pure manual entry. | No | Spec only |
| **C. Current Coverage list** | Add-as-many-rows list of current health plans (type, carrier, start/end). Pure manual entry. | No | Built ✓ (`current-coverage-lookup.html`) |

All write their data into GHL custom fields via the **same sync bridge** (Section 1.4). **A1 and A2 are two independent embeds** (split from the original combined Rx+Provider widget) so they can be placed separately. B and C are A1/A2 with the lookup layer removed and the search→card UI swapped for a direct add-form. **All widgets default to the LPI teal `rgb(97,163,183)` / `#61a3b7`.**

**Resolved decisions (2026-06-11):**
- **Tool A output mapping = "both":** `medications_json` / `providers_json` → **new** dedicated LARGE_TEXT fields (HealthSherpa/automation); summaries → the **existing** `🏥 Prescriptions` (key `medications`) / `🏥 Doctors` (key `doctors`) LARGE_TEXT fields (agent-readable). Mapping lives in a `FIELD_KEYS` const at the top of each widget — override via `window.MEDS_CONFIG.fieldKeys` / `window.PROV_CONFIG.fieldKeys`.
- **Split delivery:** Tool A ships as **two independent embeds** — `medications-lookup.html` (`#medications-lookup-widget`, `MEDS_CONFIG`) and `provider-lookup.html` (`#provider-lookup-widget`, `PROV_CONFIG`) — so prescriptions and doctors can be placed in separate code blocks.
- **Tool C storage = new json+summary pair:** `current_coverage_json` + `current_coverage_summary`.
- **Hosting = git repo + jsDelivr CDN** (one `<script>` per widget, versioned by tag). ⚠️ jsDelivr only serves **public** repos — see Section 1.10 caveat. The widgets carry no secrets/PHI, so a dedicated **public** repo for this subaccount is fine; if private is required, host `dist/*.js` on Cloudflare Pages / Netlify / GHL file hosting instead.

> ⚠️ **Why this is a separate project (not a fork of `maxmethod/doc-rx-lookup`):** that repo is **intentionally public and CDN-shared** — its `embed.js` is pulled via jsDelivr by potentially several subaccounts. Account-specific fields/sections must NOT land there or they ship to every account on the next snapshot sync. This project is the isolation boundary.

---

## 0. Provenance — what to fork from

The working reference implementation is **`rx-provider-lookup.html` @ tag `v1.0.9`** in `github.com/maxmethod/doc-rx-lookup` (local copy at `../rx-provider-lookup/`).

**Recommended start: copy that file, don't retype it.** It's ~1,400 lines of working, mobile-tested code. The customizations in this spec are small, surgical edits to that base — far more reliable than recreating from prose.

```
cp ../rx-provider-lookup/rx-provider-lookup.html  ./rx-provider-lookup.subaccount.html
cp -r ../rx-provider-lookup/scripts               ./scripts
# us-zips.json: keep pulling from the existing CDN (Section 2.5) OR copy ../rx-provider-lookup/dist/us-zips.json
```

Reference constants in that file (top of `<script>`, ~line 513):
```js
const MAX_MEDS = 10;
const MAX_PROVIDERS = 5;
const DEBOUNCE_MS = 300;
const CT_BASE  = 'https://clinicaltables.nlm.nih.gov/api/npi_idv/v3/search';
const RX_BASE  = 'https://rxnav.nlm.nih.gov/REST';
const ZIP_BASE = 'https://api.zippopotam.us/us';
const ZIP_DATASET_VERSION = 'v1.0.9';
```

---

## 1. Shared foundation (reuse verbatim in BOTH tools)

These five pieces are the entire reusable skeleton. Tool B keeps all of them and throws away everything else.

### 1.1 State model
```js
const state = {
  items: [],        // the rows — meds / providers / income / assets
  pending: null,    // a row being composed before "Add" is clicked
  // (Tool A also: zipCoordCache, zipDataset, zipDatasetPromise — drop for Tool B)
};
```
In the reference, `items` is split into `state.medications` and `state.providers` because there are two sections. Use one `state.<thing>` array per repeating section.

### 1.2 Utilities (copy as-is)
- `debounce(fn, ms)` — used for search-as-you-type (Tool A only).
- `escapeHtml(s)` — **always** wrap user/API text before putting it in `innerHTML`.
- `toTitleCase(s)` — name/city normalization.

### 1.3 The render + add/remove row pattern (THIS is the core of Tool B)
Every repeating section follows the exact same shape. From `renderMedications()` (ref ~line 877):

```js
function renderItems() {
  const list = document.getElementById('item-list');
  const count = state.items.length;
  document.getElementById('item-count').textContent = `${count} of ${MAX_ITEMS}`;

  // 1. redraw every row from state
  list.innerHTML = state.items.map(it => `
    <div class="item-card ${it.source === 'manual' ? 'manual' : ''}">
      <div class="item-card-main">
        <div class="item-card-name">${escapeHtml(it.name)}</div>
        <div class="item-card-meta">…</div>
      </div>
      <button class="danger-text" data-remove="${it.id}">Remove</button>
    </div>`).join('');

  // 2. wire each Remove button (re-wired every render — simplest, no leaks)
  list.querySelectorAll('[data-remove]').forEach(btn => {
    btn.onclick = () => {
      state.items = state.items.filter(it => it.id !== btn.dataset.remove);
      renderItems();
    };
  });

  // 3. hide the add UI once the limit is hit
  if (count >= MAX_ITEMS) hideAddUI();

  // 4. ALWAYS sync to GHL on every state change
  syncHiddenFields();
}
```
Adding a row = push to `state.items` (id = `` `item_${Date.now()}` ``), then `renderItems()`. Removing = filter + `renderItems()`. **Never** manipulate the DOM directly — mutate state, re-render. That single discipline is why the pattern is so portable.

### 1.4 `syncHiddenFields()` — the GHL bridge (the most important function; copy exactly)
```js
function syncHiddenFields() {
  const setAll = (key, value) => {
    // Match BOTH our own placeholder inputs (name=) AND GHL's real custom-field
    // inputs (GHL puts the field key in data-q, with a random name=).
    const selector = `[name="${key}"], [data-q="${key}"]`;
    document.querySelectorAll(selector).forEach(el => {
      el.value = value;
      el.dispatchEvent(new Event('input',  { bubbles: true }));   // <- lets GHL register the change
      el.dispatchEvent(new Event('change', { bubbles: true }));
    });
  };
  setAll('medications_json',    buildMedicationsJson());
  setAll('medications_summary', buildMedicationsSummary());
  // …one setAll per output field…
}
```
Why both events: GHL's form framework only records a value for validation / conditional logic / submission if it sees an `input`/`change` event. Setting `.value` alone is invisible to it.

### 1.5 Dual output: JSON + human summary (one pair per data domain)
Every domain emits two fields:
- `<thing>_json` — machine-readable, for HealthSherpa / automation. Shape:
  ```json
  { "version": "1.0", "count": 2, "items": [ { …flat fields… } ] }
  ```
- `<thing>_summary` — plain-text bullet list, for agent notes / SMS / email.

Builders are pure functions of `state` (ref ~line 1219): `buildMedicationsJson()`, `buildMedicationsSummary()`, etc. They return `''` when empty.

### 1.6 Primary-color inheritance (copy `applyPrimaryColor` + `darken` + `isValidColor`)
Resolution order, first match wins (ref ~line 1328):
1. `window.RX_CONFIG.primaryColor` (explicit override)
2. `data-primary-color` on the `#…-widget` container ← **best for GHL:** set it to `{{ custom_values.brand_primary_color }}` and GHL substitutes the brand color server-side.
3. GHL **form** submit-button inline `background-color`
4. GHL **survey** footer Next/Submit button computed color
5. Fallback to the widget's default blue (`#1e4d8c`)

Sets `--accent` and `--accent-hover` (a 12%-darkened shade) on the container. **Rename `RX_CONFIG` → a tool-specific global** (e.g. `INCOME_CONFIG`) so the two tools don't collide if ever on the same page.

### 1.7 CSS scoping (non-negotiable inside GHL)
All widget CSS is scoped to the container id, with the id **doubled** for specificity (`#widget#widget .foo`) so it out-weighs GHL's form-framework rules **without `!important`**. `:root` custom-property declarations stay global; `body`/`html` rules are dropped. This is done automatically by `build-embed.js` (Section 1.9) — author plain CSS in the HTML, let the build scope it.

Design tokens live in `:root` (copy the whole block): `--bg, --surface, --border, --border-strong, --text, --text-muted, --accent, --accent-hover, --danger, --danger-hover, --success, --warning-bg, --warning-border, --radius, --shadow-sm, --shadow-md, --font`.

### 1.8 `embed.js` self-bootstrap (production delivery)
The thing pasted into GHL. It (ref `dist/embed.js`):
1. Guards against double-load (`window.__<tool>EmbedLoaded`).
2. Injects the scoped `<style>` into `<head>`.
3. Finds `#<tool>-widget` (or `[data-…-widget]`), or creates+appends one.
4. Sets `container.innerHTML` to the widget markup.
5. Runs the widget logic inside an IIFE.

### 1.9 `build-embed.js` (regenerate embed after every HTML edit)
`node scripts/build-embed.js` reads the source HTML and:
- extracts the `<style>` block → `scopeCss()` (doubles the id, drops body/html, keeps `:root`),
- extracts `<body>`, strips its inline `<script>`,
- escapes both for a JS template literal (`\` `` ` `` `${`),
- emits the bootstrap IIFE to `dist/embed.js`.

**Generalize it for two widgets:** parameterize input/output paths and the scope id, or run it twice. Re-run after *every* change to the source HTML — the embed is a build artifact, never hand-edited.

### 1.10 Deploy model
jsDelivr serves **immutable content per git tag**, so:
1. Commit to `main`, then `git tag vX.Y.Z && git push origin vX.Y.Z`.
2. Bump the `@vX.Y.Z` in the GHL `<script src>` (and `ZIP_DATASET_VERSION` if the ZIP file changed).
3. **Never point the embed at `@main`** — `@main` is cached up to 12h and unpredictable. Always tags.

> 🚩 **CDN caveat for this project (decide before first deploy):** jsDelivr only serves **public** GitHub repos. If you want this subaccount repo **private**, jsDelivr won't work — you must either (a) keep it public (it's just a widget, no secrets — fine), (b) host `embed.js` elsewhere (GHL file hosting, Netlify, Cloudflare Pages), or (c) skip the CDN and paste the full standalone HTML into the GHL Custom Code block. See Section 5.

---

## 2. GHL integration contract (cheat-sheet — applies to both tools)

**Placement:** the widget goes in a GHL **Custom Code / Custom HTML element**. The only hard requirement is that the widget and the form's custom fields end up in the **same rendered DOM/frame**, so `syncHiddenFields()` can find the fields. Two placements satisfy that, both fine:
- **Directly inside the form** (a Custom HTML element *within* the form) — cleanest; widget and fields share the DOM. This is how the OG `doc-rx-lookup` tool was embedded.
- **Alongside the form** on the same funnel/landing page (a page-level Custom Code element next to the form).

> ⚠️ The real gotchas (not "form builder = forbidden"): GHL's **in-builder preview** sometimes doesn't execute `<script>` — always test on the **published** form. And if the form is embedded elsewhere as an **iframe with `sandbox`**, scripts can be blocked there. A normal GHL-hosted form/funnel runs the embed fine.

**Embed snippet** (per widget; `data-primary-color` locks the LPI teal so GHL's gold brand value can't override it):
```html
<div id="medications-lookup-widget" data-primary-color="rgb(97, 163, 183)"></div>
<script src="https://cdn.jsdelivr.net/gh/maxmethod/lpi-enrollment-widgets@v1.1.0/dist/embed-medications.js"></script>
```

**Custom fields** — create in *Settings → Custom Fields*, type **Multi-line / Large text** (JSON payloads exceed single-line limits). The widget targets each by its **field key**, matching `[name="<key>"]` **and** `[data-q="<key>"]`.

| Tool | Field keys (must match exactly) |
| --- | --- |
| A1 (Medications) | `medications_json` (new) · `medications` (summary → existing `🏥 Prescriptions`) ✓ verified |
| A2 (Doctors/Providers) | `providers_json` (new) · `doctors` (summary → existing `🏥 Doctors`) ✓ verified |
| B (Income/Assets) | `income_json`, `income_summary`, `assets_json`, `assets_summary` *(confirm keys w/ HealthSherpa mapping owner)* |
| C (Current Coverage) | `current_coverage_json`, `current_coverage_summary` *(both new)* |

**Fields created for this rollout ✓ (LARGE_TEXT, folder `LPI - Pre-Enrollment Form Data` `QHuwbPgm36VRG0d5ZKw1`):** `medications_json`, `providers_json`, `current_coverage_json`, `current_coverage_summary`. The two summaries reuse the **existing** `🏥 Prescriptions` (key **`medications`**) / `🏥 Doctors` (key **`doctors`**) fields — verified on live 2026-06-11, no new summary fields needed.

**Verify once in DevTools:** inspect a rendered hidden field and confirm `data-q` equals the bare key (e.g. `medications_json`, no `contact.` prefix). If GHL prefixes it, either rename the field key or widen the selector in `syncHiddenFields` to also match the prefixed form.

**Constraints:** HTTPS only (GHL pages are HTTPS ✓). No PHI touches any server we control — data goes straight into GHL's own fields.

---

## 3. Tool A — RX + Provider Lookup (customized)

Base = the reference HTML, essentially unchanged except the customization in **3.3**.

### 3.1 Sections & data sources
**Medications (max 10):** type-ahead search → strength/form picker → frequency → Add. Manual-entry fallback.
- `searchMedications(q)` → `GET {RX_BASE}/approximateTerm.json?term=&maxEntries=8`, then per-candidate `…/rxcui/{id}/property.json?propName=RxNorm%20Name` for names.
- `getDrugDetails(rxcui)` → `…/rxcui/{id}/allrelated.json`; collects `SCD`/`SBD` strength-forms and `BN` brand names.
- `getNdcForRxcui(rxcui)` → `…/rxcui/{id}/ndcs.json` (first NDC).
- `parseDrugName(name)` → regex out strength (`\d+ MG/MCG/ML/G/%/UNIT`) and dosage form.

**Providers (max 5):** ZIP + radius, then doctor name search, radius-filtered client-side. Manual fallback.
- `searchProviders(name, zip, radius)` → `GET {CT_BASE}?terms=&count=50&q=addr_practice.state:{ST}&ef=addr_practice,name.full,provider_type,licenses,provider_credential_text`.
  - Response is a positional array: `[total, [npis], extrasObj, …]`. Specialty = primary taxonomy classification from `licenses`.
  - Radius filter: `getZipCoordinates(zip)` for each result, `haversineMiles()` vs the center ZIP, keep ≤ radius, sort by distance, cap at 12. Batched 10-at-a-time with an early break at 12 hits.
- `getZipCoordinates(zip)`: looks up the **bundled `us-zips.json`** (`{ "12345": [lat, lon, "ST"] }`); falls back to `GET {ZIP_BASE}/{zip}` (zippopotam) only on a miss (new/APO-FPO ZIPs).

### 3.2 Output schemas (don't change the existing fields)
`medications_json` → `items[]` of: `id, name, brand_name, strength, dosage_form, frequency, rxcui, ndc, source`.
`providers_json` → `items[]` of: `id, full_name, first_name, last_name, npi, specialty, provider_type, address{street,street2,city,state,zip}, source`.
`source` ∈ `rxnorm | clinicaltables | manual`. Summaries are bullet text with `[manually entered]` flags.

### 3.3 ⭐ CUSTOMIZATION — add new field(s) to the prescription section

> **DONE (2026-06-03):** the LPI fork added a **"Who takes this?" (`applicant_name`)** optional field to both the search and manual medication paths — wired through the entry object, the JSON + summary builders, and the card meta. Caps were also raised (`MAX_MEDS` 10→20, `MAX_PROVIDERS` 5→10). The recipe below remains the template for adding any *further* field end-to-end; the worked example uses **`quantity`** (e.g. "30 tablets / 90-day supply").

Adding one field touches **exactly four places** (all in the medications flow; `syncHiddenFields` needs **no** change):

**① Markup** — add the input to the strength block *and* the manual block.
In `#med-strength-block` (ref ~line 344), next to the Frequency `<select>`:
```html
<div>
  <label for="med-quantity">Quantity / days supply</label>
  <input type="text" id="med-quantity" placeholder="e.g. 30 tablets / 90-day">
</div>
```
Mirror it in `#med-manual-block` with id `med-manual-quantity`.

**② Read it in the Add handlers** — `#med-add` (ref ~line 811) and `#med-manual-add` (ref ~line 856):
```js
const quantity = document.getElementById('med-quantity').value.trim() || null;        // search path
// const quantity = document.getElementById('med-manual-quantity').value.trim() || null; // manual path
```
(Also clear `med-manual-quantity` in the manual-path reset list, ref ~line 872.)

**③ Add to the `entry` object** pushed into `state.medications`:
```js
const entry = { id:`med_${Date.now()}`, name, /* …existing… */, quantity, source:'rxnorm', display_name };
```

**④ Add to both builders** (ref ~line 1223):
- `buildMedicationsJson()` — add `quantity: m.quantity` to the mapped item.
- `buildMedicationsSummary()` — append `if (m.quantity) parts.push(m.quantity);` (or its own line).

Then `node scripts/build-embed.js`, tag, bump the snippet. If the new field must also reach HealthSherpa as its **own** GHL field, add `setAll('medications_<field>', …)` in `syncHiddenFields` and create the matching custom field — otherwise it just rides inside `medications_json`.

---

## 4. Tool B — Income & Assets multi-step table (NEW; build next session)

> **Status: spec skeleton only.** The user will prompt the full build next session. Captured here so this doc covers both tools.

### 4.1 Concept
A **multi-step table** for two domains — **income sources** and **assets** — where the user adds **as many rows as needed**. **No external lookups.** This is Tool A's repeating-row engine (1.3) with the lookup layer deleted and the search→select→card flow replaced by **directly editable table rows**.

### 4.2 What to reuse vs. strip
**Reuse:** state array + `renderItems` + add/remove + `syncHiddenFields` + JSON/summary builders + color inheritance + CSS tokens + `embed.js`/`build-embed.js`.
**Strip entirely:** RxNorm + Clinical Tables + ZIP dataset code — `searchMedications`, `getDrugDetails`, `getNdcForRxcui`, `searchProviders`, `getZipCoordinates`, `loadZipDataset`, `haversineMiles`, `debounce` (no type-ahead), all `*_BASE`/`ZIP_DATASET_*` constants, and the search/results/strength/manual-toggle markup.

### 4.3 Draft data model (CONFIRM next session)
```
income row: { id, type, source_name, amount, frequency }      // frequency: weekly|biweekly|monthly|annually
asset  row: { id, type, description, value }
```
Output fields (draft): `income_json` / `income_summary` / `assets_json` / `assets_summary`, same `{version,count,items[]}` envelope. **Confirm the exact field keys with whoever maps HealthSherpa.**

### 4.4 New UI work (the only genuinely new build)
- "Multi-step": decide **two steps** (income table → assets table, with Back/Next) **vs. one page, two tables**. Open design decision.
- Rows should be **inline-editable** (a real `<table>` or a grid of row inputs with an "+ Add row" button and a per-row Remove), not the search→select→card pattern. This row-editor is the main net-new component; everything around it is copied.
- On any input/blur/add/remove → re-read rows from the DOM into `state`, re-render counts/totals, `syncHiddenFields()`. Consider showing a computed **total** (sum of amounts / values) as a nicety.

---

## 4C. Tool C — Current Coverage list (BUILT)

File: **`current-coverage-lookup.html`** → builds to **`dist/embed-coverage.js`**. It's Tool A's repeating-row engine (1.3) with the lookup layer deleted and the search→card flow replaced by a single always-visible **add-form**.

- **Container id:** `coverage-lookup-widget` · **config global:** `COVERAGE_CONFIG` · **embed guard:** `__coverageWidgetEmbedLoaded` · **style attr:** `data-coverage-lookup`. All distinct from Tool A so both embeds can sit on the same page.
- **Row model:** `{ id, coverage_type, coverage_level, insurance_company, member_id, who_covered, start_date, end_date }`.
  - `coverage_type` — select (Employer/Group, Marketplace/ACA, Medicaid, Medicare, MA, Med Supp, COBRA, Individual/Private, Military/VA/TRICARE, Short-Term, IHS, None/Uninsured, Other). **Required** to add a row.
  - `coverage_level` — Primary / Secondary (shows a badge on the card).
  - `start_date` / `end_date` — `<input type="month">` → stored `YYYY-MM`; blank end = "present".
  - `who_covered` — free text (e.g. "John Smith" / "Whole household"). Mirrors Tool A's `applicant_name`.
- **Cap:** `MAX_COVERAGES = 15` (add-form hides at the cap).
- **Output:** `current_coverage_json` `{version,count,items[]}` + `current_coverage_summary` (bullet text). Keys overridable via `window.COVERAGE_CONFIG.fieldKeys`.
- **Embed snippet:**
  ```html
  <div id="coverage-lookup-widget" data-primary-color="{{ custom_values.brand_primary_color }}"></div>
  <script src="https://cdn.jsdelivr.net/gh/maxmethod/lpi-enrollment-widgets@vX.Y.Z/dist/embed-coverage.js"></script>
  ```
- **Build:** `node scripts/build-embed.js coverage` (or no arg = all widgets). The build script is now config-driven (a `WIDGETS` map) and emits one `dist/*.js` per widget.
- **Test harness:** `test-embed-coverage.html` (simulates the GHL form's hidden fields + submit).

## 5. New-project setup checklist

1. `git init` the renamed folder. **Decide public vs private** (Section 1.10 caveat) — drives your hosting choice.
2. Layout:
   ```
   <tool-a>.html            ← copied + customized reference
   <tool-b>.html            ← new income/assets tool
   scripts/build-embed.js   ← generalized for 2 widgets
   dist/embed-rx.js
   dist/embed-income.js
   dist/us-zips.json        ← Tool A only (or keep pulling from existing CDN)
   README.md
   ```
3. Hosting (pick one): **public repo + jsDelivr** (matches the existing tool), or **Netlify/Cloudflare Pages** (works with private), or **paste standalone HTML** into GHL (no CDN, but manual re-paste on every update).
4. Create the GHL custom fields, embed per Section 2, verify `data-q` in DevTools, test end-to-end into a contact record.

## 6. Open questions
- [x] Tool A: new prescription field(s) → **`applicant_name` ("Who takes this?")** done (Section 3.3).
- [x] Tool A: output mapping → **JSON to new fields, summaries to existing `🏥 Prescriptions`/`🏥 Doctors`** (Section 2).
- [x] Tool C: current-coverage data model + storage → **built; `current_coverage_json` + `current_coverage_summary`** (Section 4C).
- [x] Hosting → **public repo `maxmethod/lpi-enrollment-widgets` + jsDelivr**, tag **`v1.0.0`** live (both embeds HTTP 200).
- [x] **Confirmed `data-q` keys** on the live LPI location (2026-06-11): `🏥 Prescriptions` → **`medications`**, `🏥 Doctors` → **`doctors`** (set in `FIELD_KEYS`).
- [x] **Created the 4 new LARGE_TEXT fields** (`medications_json`, `providers_json`, `current_coverage_json`, `current_coverage_summary`) in folder `QHuwbPgm36VRG0d5ZKw1` via `scripts/create_widget_fields.js`.
- [x] **Repo stood up + tagged `v1.0.0`**; embed snippets point at `@v1.0.0`.
- [ ] **Embed all three Custom Code blocks** into the GHL form pages + verify each field lands end-to-end into a test contact (the only step left — user does it in the GHL builder).
- [ ] Tool B (Income & Assets): still spec-only — data model + exact GHL field keys (HealthSherpa mapping owner); multi-step vs. single page.
