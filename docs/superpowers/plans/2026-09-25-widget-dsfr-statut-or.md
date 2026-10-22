# Widget DSFR + Statut OR Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Sidebar DSFR, filtre Statut multi-checkbox OR, header acheteurs/acteurs séparés — dans `widget_carto.html` seul.

**Architecture:** Un fichier HTML autonome. CDN DSFR `@gouvfr/dsfr@1.13.0` pour la sidebar. `Statut_structure` normalisé en `statutAtoms[]` ; match OR via checkboxes. Carte Leaflet + `#detail` hors DSFR.

**Tech Stack:** HTML/CSS/JS vanilla, Leaflet, Grist plugin API, DSFR 1.13.0 (jsDelivr).

## Global Constraints

- Modify only: `widget_carto.html` (plus ce plan / spec déjà commités).
- DSFR scope: sidebar only — never restyle `#map` / `#detail` with `fr-*`.
- CDN pin: `@gouvfr/dsfr@1.13.0` (stable, widely documented).
- No npm bundler; no new dependencies beyond CDN `<link>` / `<script>`.
- Statut atoms derived dynamically from data; FR locale sort.
- Keep Grist/CSV boot, IGN basemap, export CSV, help panel behavior.

## File map

| File | Role |
|---|---|
| `widget_carto.html` | Only implementation surface: head CDN, sidebar markup, filter JS, list render |
| `docs/superpowers/specs/2026-09-25-widget-dsfr-statut-or-design.md` | Approved design (read-only reference) |
| `acteurs_reemploi_poc.csv` | Local preview data (read-only; used to verify OR filter) |

---

### Task 1: Statut atoms + OR match (logic first)

**Files:**
- Modify: `widget_carto.html` (JS: `MULTI`, `norm`, `buildFilters`, `getFilters`, `match`, `renderList`, `showDetail`, reset)

**Interfaces:**
- Consumes: existing `asList`, `DATA`, `uniq`
- Produces:
  - `d.statutAtoms: string[]` on each normalized record
  - `d.statut: string` kept as join `", "` for export/display fallback
  - `getFilters().stat: string[]` (checked values)
  - `match(d,f)` returns true when `f.stat.length===0` OR `f.stat.some(s => d.statutAtoms.includes(s))`
  - `buildStatutChecks(values: string[]): void` renders checkboxes into `#f_stat`

- [ ] **Step 1: Write a failing Node assert for OR match**

Create temporary script (delete after Task 1 passes) at `/tmp/test-statut-or.mjs`:

```javascript
function asList(v){
  if (Array.isArray(v)) { const a = v[0]==="L" ? v.slice(1) : v; return a.map(x=>String(x).trim()).filter(Boolean); }
  return String(v==null?"":v).split(";").map(s=>s.trim()).filter(Boolean);
}
function match(d, f){
  if (f.stat && f.stat.length && !f.stat.some(s => d.statutAtoms.includes(s))) return false;
  return true;
}
const d = { statutAtoms: asList("Secteur de l'ESS;Secteur de l'insertion") };
console.assert(match(d, {stat:[]}) === true, "empty = all");
console.assert(match(d, {stat:["Secteur de l'ESS"]}) === true, "atom in combo");
console.assert(match(d, {stat:["Entreprise conventionnelle"]}) === false, "missing atom");
console.assert(match(d, {stat:["Secteur de l'insertion","Entreprise conventionnelle"]}) === true, "OR");
console.log("OK");
```

Run: `node /tmp/test-statut-or.mjs`  
Expected: `OK`

- [ ] **Step 2: Add `statut` to MULTI and keep display string**

In `widget_carto.html`, change:

```javascript
const MULTI = ["cat","act","certs"];
```

to:

```javascript
const MULTI = ["cat","act","certs","statut"];
```

In `norm`, after the `for (const k in COLS)` loop, add:

```javascript
o.statutAtoms = Array.isArray(o.statut) ? o.statut.slice() : asList(o.statut);
o.statut = o.statutAtoms.join(", ");
```

(If `MULTI` already makes `o.statut` an array via `asList`, the first line is enough; always set `statutAtoms` and string `statut` for tags/export.)

- [ ] **Step 3: Replace `#f_stat` select with checkbox fieldset**

Replace markup:

```html
<div class="field"><label for="f_stat">Statut</label><select id="f_stat"></select></div>
```

with:

```html
<div class="field full" id="f_stat_wrap">
  <fieldset id="f_stat" class="fr-fieldset">
    <legend class="fr-fieldset__legend fr-text--regular fr-fieldset__legend--regular">Statut</legend>
    <div class="fr-fieldset__content" id="f_stat_checks"></div>
  </fieldset>
</div>
```

- [ ] **Step 4: Implement `buildStatutChecks` + wire filters**

Replace `fillSelect("f_stat",...)` in `buildFilters` with:

```javascript
function buildStatutChecks(values){
  const box = document.getElementById("f_stat_checks");
  const prev = new Set(
    [...box.querySelectorAll('input[type="checkbox"]:checked')].map(i=>i.value)
  );
  box.innerHTML = values.map((v,i)=>{
    const id = "f_stat_"+i;
    const checked = prev.has(v) ? " checked" : "";
    return `<div class="fr-checkbox-group">
      <input type="checkbox" id="${id}" name="f_stat" value="${escapeHtml(v)}"${checked}>
      <label class="fr-label" for="${id}">${escapeHtml(v)}</label>
    </div>`;
  }).join("");
  box.querySelectorAll("input").forEach(inp=>{
    inp.addEventListener("change", apply);
  });
}
```

In `buildFilters`:

```javascript
buildStatutChecks(uniq(DATA.flatMap(d=>d.statutAtoms)));
```

Update `getFilters`:

```javascript
stat: [...document.querySelectorAll('#f_stat_checks input[type="checkbox"]:checked')].map(i=>i.value),
```

Update `match` statut line:

```javascript
if (f.stat.length && !f.stat.some(s => d.statutAtoms.includes(s))) return false;
```

Remove `"f_stat"` from `FILTER_IDS` input listeners (checkboxes bind in `buildStatutChecks`). Keep reset clearing:

```javascript
document.querySelectorAll('#f_stat_checks input[type="checkbox"]').forEach(i=>{ i.checked=false; });
```

- [ ] **Step 5: Tags in list + detail use atoms**

In `renderList`, replace single tag with:

```javascript
${(d.statutAtoms||[]).map(s=>`<span class="tag">${escapeHtml(s)}</span>`).join(" ")}
```

In `showDetail`, badge:

```javascript
<div class="badge">${escapeHtml((d.statutAtoms||[]).join(" · ")||"")}</div>
```

Export already uses `d.statut` string — OK (comma-joined atoms).

- [ ] **Step 6: Manual verify with CSV preview**

Run: open `widget_carto.html` via local server (or file) so CSV loads.  
Check: 4 checkboxes only; check « Secteur de l'ESS » → count includes combo rows (ESS;insertion, ESS;handicap).  
Expected: no `;` options in UI; count rises vs old exact-match behavior for ESS alone.

- [ ] **Step 7: Commit**

```bash
git add widget_carto.html
git commit -m "feat: Statut multi-checkbox with OR match on atoms"
```

---

### Task 2: Split header — acheteurs vs acteurs

**Files:**
- Modify: `widget_carto.html` (header HTML + minimal CSS / DSFR utility classes)

**Interfaces:**
- Consumes: `#cta`, `QUESTIONNAIRE_URL` wiring unchanged
- Produces: `#sidebar-buyers` and `#sidebar-actors` sibling blocks; `#help` stays below both

- [ ] **Step 1: Restructure header markup**

Replace the single `<header>...</header>` content so structure is:

```html
<header class="sidebar-top">
  <div id="sidebar-buyers" class="sidebar-buyers">
    <div class="brand">
      <!-- existing logo SVG -->
      <div class="brand-text">
        <div class="brand-top">
          <h1>Les acteurs du réemploi pour une commande publique circulaire</h1>
          <button type="button" class="info-btn" id="help_toggle" aria-expanded="false" aria-controls="help" title="Aide et notice d'utilisation">i<span class="sr-only"> Afficher l'aide</span></button>
        </div>
        <p class="fr-text--sm">Pour les acheteurs publics — SPASER / art. 58 loi AGEC</p>
      </div>
    </div>
  </div>
  <div id="sidebar-actors" class="sidebar-actors fr-background-alt--grey">
    <p class="actors-title fr-text--sm fr-mb-1w"><strong>Vous êtes un acteur du réemploi ?</strong></p>
    <a id="cta" class="fr-btn fr-btn--secondary fr-btn--sm" href="#" target="_blank" rel="noopener noreferrer">Se référencer <span class="sr-only">(nouvel onglet)</span></a>
  </div>
</header>
```

Keep `#help` immediately after `</header>` (unchanged contents).

- [ ] **Step 2: Spacing CSS for the two blocks**

Add (can stay in custom `<style>` until Task 3 trims):

```css
.sidebar-top{padding:0;background:#fff}
.sidebar-buyers{padding:12px 16px;border-bottom:1px solid var(--border)}
.sidebar-actors{padding:12px 16px;border-bottom:1px solid var(--border)}
.actors-title{margin:0 0 8px}
#cta{display:inline-flex;width:auto;text-decoration:none}
```

Remove old `.cta` green button styles once `fr-btn` is loaded (Task 3) — for now keep fallback if CDN not yet present:

```css
.cta{display:block;margin-top:10px;padding:8px 10px;background:var(--accent);color:#fff;border-radius:8px;font-size:12px;font-weight:600;text-align:center;text-decoration:none}
```

After Task 3, delete `.cta` rules if unused.

- [ ] **Step 3: Verify CTA still opens questionnaire**

Confirm `QUESTIONNAIRE_URL` IIFE still sets `#cta.href`. Click CTA in preview → Grist form URL. Help toggle still expands `#help`.

- [ ] **Step 4: Commit**

```bash
git add widget_carto.html
git commit -m "feat: separate buyer header from actor référencement CTA"
```

---

### Task 3: DSFR CDN + sidebar component classes

**Files:**
- Modify: `widget_carto.html` (`<head>`, sidebar filters/list/bar markup, prune conflicting custom CSS)

**Interfaces:**
- Consumes: Tasks 1–2 markup ids (`#f_stat_checks`, `#sidebar-buyers`, `#cta`, filter ids)
- Produces: DSFR-styled sidebar; layout `#app` / `#mapwrap` unchanged

- [ ] **Step 1: Add DSFR CDN in `<head>` (after charset/viewport, before Leaflet)**

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@gouvfr/dsfr@1.13.0/dist/dsfr/dsfr.min.css" />
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/@gouvfr/dsfr@1.13.0/dist/utility/utility.min.css" />
```

Before `</body>` (after Leaflet scripts is fine; DSFR JS after DOM):

```html
<script type="module" src="https://cdn.jsdelivr.net/npm/@gouvfr/dsfr@1.13.0/dist/dsfr/dsfr.module.min.js"></script>
<script nomodule src="https://cdn.jsdelivr.net/npm/@gouvfr/dsfr@1.13.0/dist/dsfr/dsfr.nomodule.min.js"></script>
```

On `<html>` add: `data-fr-scheme="light"` (widget: force light, avoid dark flash in iframe).

- [ ] **Step 2: Remap filter fields to DSFR**

For each text filter (pattern):

```html
<div class="fr-input-group">
  <label class="fr-label" for="f_nom">Nom</label>
  <input class="fr-input" id="f_nom" type="search" placeholder="Nom de la structure…" autocomplete="off" />
</div>
```

For each select:

```html
<div class="fr-select-group">
  <label class="fr-label" for="f_cat">Catégorie de produit</label>
  <select class="fr-select" id="f_cat"></select>
</div>
```

Keep `#f_stat` fieldset from Task 1. Wrap grid in `.filters` with `fr-grid-row` optional; if grid fights DSFR, use simple stacked `fr-mb-2w` groups (preferred for narrow 400px sidebar — **stack all fields**, drop `.grid2` two-column layout).

- [ ] **Step 3: Sections + list + bar**

Replace custom `<details>` summaries with DSFR-friendly structure — keep native `<details open>` for simplicity (no accordion JS dependency), but style summary with DSFR text utilities:

```html
<details open class="fr-mb-2w">
  <summary class="fr-h6">Filtres</summary>
  ...
</details>
<details open>
  <summary class="fr-h6">Liste des acteurs (<span id="count2">0</span>)</summary>
  <ul id="list" class="fr-raw-list" tabindex="-1"></ul>
</details>
```

Update `renderList` item button classes:

```html
<button type="button" class="fr-btn fr-btn--tertiary-no-outline fr-btn--block item${cur?' sel':''}" ...>
```

Bar:

```html
<div class="bar">
  <span aria-live="polite"><b id="count">0</b> structure(s)</span>
  <div class="bar-actions">
    <button type="button" class="fr-btn fr-btn--tertiary-no-outline fr-btn--sm" id="export" disabled>Exporter la recherche (Excel)</button>
    <button type="button" class="fr-btn fr-btn--tertiary-no-outline fr-btn--sm" id="reset">Réinitialiser</button>
  </div>
</div>
```

- [ ] **Step 4: Prune CSS that fights DSFR**

Delete or neutralize custom rules for: `.field label` uppercase muted, `.field input/select` borders, `.cta`, `.quick` (if unused), heavy card chrome on `.item` if `fr-btn` covers it.  
**Keep:** `#app` flex layout, `#sidebar` width 400px, `#mapwrap`/`#map`, `#detail` card styles, `.sr-only` / skip-link, list scroll, map z-index.

Ensure Marianne loads via DSFR CSS (do not force `-apple-system` on `body` for sidebar — scope legacy font to `#mapwrap`/`#detail` only if needed):

```css
body{margin:0;height:100%;width:100%;color:var(--ink);background:var(--bg)}
/* DSFR sets font on fr-root; ensure html uses DSFR */
```

- [ ] **Step 5: Visual + regression check**

Preview CSV mode:
1. Sidebar looks DSFR (Marianne, blue focus, secondary CTA).
2. Two header blocks distinct.
3. Statut OR still works.
4. Map markers / detail card / export / reset / help unchanged.
5. Mobile `@media (max-width:720px)` still stacks.

- [ ] **Step 6: Commit**

```bash
git add widget_carto.html
git commit -m "feat: apply DSFR components to carto sidebar"
```

---

### Task 4: Acceptance pass + cleanup

**Files:**
- Modify: `widget_carto.html` only if bugs found
- Delete: `/tmp/test-statut-or.mjs` if still present

- [ ] **Step 1: Walk spec acceptance criteria**

From `docs/superpowers/specs/2026-09-25-widget-dsfr-statut-or-design.md`:

1. No `;` statut combinations in UI  
2. ESS checkbox includes combo rows  
3. ESS + insertion = union  
4. Header split visible  
5. Sidebar DSFR; map not DSFR-themed  
6. Export/reset/Grist/CSV/map OK  

- [ ] **Step 2: Fix any miss; commit if needed**

```bash
git add widget_carto.html
git commit -m "fix: polish DSFR sidebar after acceptance pass"
```

(Skip commit if nothing to fix.)

---

## Self-review (plan vs spec)

| Spec requirement | Task |
|---|---|
| Statut atoms + OR multi-checkbox | Task 1 |
| Header two blocks acheteurs / acteurs | Task 2 |
| DSFR CDN sidebar `fr-*` | Task 3 |
| Detail/map hors DSFR | Task 3 Step 4 keep |
| Dynamic atoms, FR sort via `uniq` | Task 1 (`uniq` already localeCompare fr) |
| Acceptance criteria | Task 4 |

No TBD placeholders. Single file scope preserved.
