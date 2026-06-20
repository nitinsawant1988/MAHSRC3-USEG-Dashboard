# MAHSR C3 — USEG Dashboard
## Summary, Session Log & Regeneration Specification (v3 — final)

---

## 1. What This Dashboard Does

A single-file HTML dashboard tracking **Underslung Erection Gear (USEG) installation progress** across the Mumbai–Ahmedabad High Speed Rail (MAHSR) **Package C3** viaduct corridor — Km 21.150 to Km 156.600.

It is entirely self-contained (one `.html` file). No server, no build tools required to run it — though it now optionally connects to **Firebase Firestore** for live, shared data sync once deployed.

**New in this version:** a USEG Type Reference panel (parsed from a second Excel sheet), contextual export (PDF + Excel, now behind a single dropdown button) from any drill-down view, static repo-hosted reference images per type with zoom, Firestore-backed live data sync across all viewers, and a polished gradient button treatment for the two primary actions (Update Data, Export).

---

## 2. Project Scope Covered

| Category | Method | Description |
|---|---|---|
| **Viaduct USEG** | `USLG` | Standard underslung girder spans on main viaduct |
| **Station Approach USEG** | `SBS-TwinCell` | Twin-cell box spans near station approaches |
| **SMD Ramps** | `SMD No.1` … `SMD No.8` | Launching girder ramp spans at SMD locations |

### Section Boundaries

| Section | Chainage Range |
|---|---|
| Section-01 | Ch.21+150 to Ch.53+315 |
| Section-04 | Ch.53+315 to Ch.84+570 |
| Section-02 | Ch.84+570 to Ch.127+600 |
| Section-03 | Ch.127+600 to Ch.156+600 |

---

## 3. Features Built

### 3.1 Main Dashboard (Home Screen)
- **Hero card** — Grand Total scope, completed count, overall % with animated progress bar; clickable to drill all spans
- **KPI cards** — One each for Viaduct USEG (blue), Station Approach USEG (purple), SMD Ramps (orange); clickable to drill by category
- **Section cards** — One per section (Section-01/02/03/04) with:
  - Accent left border (blue/purple/green/orange per section)
  - "MAHSR C3" eyebrow label
  - Section name + chainage range
  - Total scope number (top-right)
  - Progress bar + completed/total fraction
  - Breakdown rows: Viaduct USEG / Station Approach USEG / SMD Ramps with dot colour, label, and `X / Y` count
- **Reference card row** — sits below Section cards. "Available USEG Types" stat card (top blue border, matches KPI card visual language) showing a live count; clickable to open the USEG Type Reference panel.
- **Last Updated** date shown in header
- **Update Data button** — styled blue gradient pill in the header (see §3.8); password-gated Excel upload

### 3.2 Drill-Down Modal
Opens when any card is clicked. Shows a sortable, paginated table of span records.

**Columns in drill table:**
`Span ID | Chainage (km) | Span (m) | Category | Pier Height (m) | Applicable USEG | Location / Remark | Status`

**Modal features:**
- Live search bar (filters across all columns)
- Sortable columns (click header to toggle asc/desc)
- Pagination (25 rows per page)
- Status badges: green `Completed`, grey `Pending`
- Category badges: Viaduct USEG (blue), Station Approach USEG (purple), SMD Ramps (orange)
- **Export button** — single green gradient dropdown (`⬇ Export ▾`) in the modal toolbar next to the search box. See §3.6.

### 3.3 Summary Panels (inside Modal)
Toggled via "Show/Hide summary panels" button. Four panels:

| Panel | Filters on |
|---|---|
| Span Length Mix | Span in metres (e.g. 35m, 40m, 44m) |
| Pier Height Groups | ≤10m / 10–15m / 15–20m / >20m |
| Category Summary | Viaduct / Station / SMD |
| USEG Types Available | Type-01 through Type-07 etc. (derived from the `Applicable USEG` column in span data — independent of the USEG Type Reference panel in §3.7) |

### 3.4 Multi-Select Filtering
- Click any stat row or chip to **add** it as an active filter
- **Multiple filters combine**: AND across types (span AND height AND useg AND category), OR within the same type (e.g. 35m OR 40m)
- **Active filter chips bar** appears below toggle: named chips (e.g. "Span: 35m", "Height: >20m", "USEG: Type-01") each with individual ✕ remove
- **Clear all** link removes all filters at once
- Badge on toggle bar shows count: "1 filter active", "2 filters active", etc.
- Order of filter selection does not matter
- No interaction was added between table cell values (e.g. `Applicable USEG` column) and the USEG Type Reference panel — explicitly deferred (see §8)

### 3.5 Excel Upload (Password-Gated)
- Password: `mahsr2026` (set via `UPLOAD_PASSWORD` constant in the script — change this one line to rotate the password)
- Parses **two sheets** from the same workbook in one upload:
  - `USLG_WithHeight` — span/progress records
  - `USEG-Type` — type reference attributes (see §3.7.1)
- Accepts `.xlsx` / `.xls`
- Uses native `input[type="file"]` — no JS click relay needed (works inside sandboxed iframes/previews)
- After upload: rebuilds hero, KPI cards, section cards, USEG Type Reference panel, and all drill data live
- **On successful upload, the parsed data is pushed to Firestore** (if configured — see §3.9) so every other open tab/device updates automatically via a realtime listener, with no refresh needed. If Firestore isn't configured, the dashboard works exactly as before from the data embedded in the page — nothing breaks either way.

### 3.6 Contextual Export

**Location:** Inside every drill-down modal (KPI or Section), and inside the USEG Type Reference panel header. **One button**, not two — `⬇ Export ▾` — which opens a small dropdown with `⬇ Excel (.xlsx)` and `🖨 PDF` options. Selecting either closes the dropdown and runs that export.

**Scope — exports exactly what's on screen, not the whole dataset (drill-down only):**
- If filters are active (via §3.4), only the currently filtered rows are exported.
- If no filters are active, the full table for that KPI/Section view is exported.
- The export always reflects `filteredRows` (the same in-memory array driving the visible table).

**Excel export (`exportDrillToExcel`):**
- Built with SheetJS (`XLSX.utils.aoa_to_sheet` + `sheet_add_json`), single sheet named "Export"
- Branded header block: "L&T Construction — MAHSR Package C3" / "USEG Installation Tracking Dashboard — Export", then which KPI/Section view was open, active filter summary (or "No filters applied"), export timestamp, total record count
- Then the filtered rows as a table: `# | Span ID | Chainage (km) | Span (m) | Category | Pier Height (m) | Applicable USEG | Location / Remark | Status`
- Column widths tuned for readability (Location/Remark column widened to 36 characters)
- Filename pattern: `MAHSR_C3_{ViewName}_{YYYY-MM-DD}.xlsx`
- Returns a success message string (e.g. `"Excel exported — 42 records"`) which surfaces as a toast (§3.6.1)

**PDF export (`exportDrillToPDF`):**
- Implemented via the browser's native print dialog — opens a new tab/window with a clean, print-styled HTML table, then calls `window.print()` after a short delay so the person picks "Save as PDF" themselves
- No external PDF library used; this guarantees the printed layout matches the styled HTML exactly
- Branded header (blue accent rule, "L&T Construction · MAHSR Package C3") + view name, filters, Completed/Pending breakdown, timestamp
- **Page numbering** via CSS Paged Media (`@page { @bottom-right: "Page " counter(page) " of " counter(pages); }`) — reliably renders in Chromium's print-to-PDF engine
- A small footer line at the bottom of the document: "MAHSR C3 USEG Dashboard — generated [date]"
- **Throws an `Error('popup-blocked')`** instead of alerting directly, if `window.open()` returns `null` — the calling wrapper (`runExport`, see §3.6.1) catches this and shows a descriptive toast instead of crashing

**USEG Type Reference export — different scope (one click, all types):**
- `exportAllUsegTypesToExcel()` — one workbook, **one sheet per type** (9 sheets), each with a branded header and full attribute table (Attribute | Value), sheet names truncated to Excel's 31-char limit with invalid characters stripped
- `exportAllUsegTypesToPDF()` — one print-styled document with **one page per type** (CSS `page-break-after`), each page with the branded header, attribute table, and that type's reference image (or "No image available" if not yet present in the repo); page numbers via the same `@page` rule
- Both throw `Error('no-data')` if `USEG_TYPES` is empty, and `Error('popup-blocked')` on a blocked popup — same toast-driven error handling as the drill-down exports

#### 3.6.1 Export UX Polish

- **Single dropdown button** (`toggleExportDropdown(id)` / `closeAllExportDropdowns()`) replaces the earlier two-separate-buttons layout. Clicking outside the dropdown, or pressing Escape, closes it. Only one dropdown open at a time.
- **`runExport(dropdownId, label, fn)`** wraps every export action:
  - Shows a spinner + "Generating…" on the trigger button while `fn()` runs (guards against double-clicks via a `.loading` class check)
  - On success, shows a green checkmark toast with the function's returned message
  - On failure, shows a red warning toast — with specific wording for the two known thrown error types (`popup-blocked` → "Popup blocked — please allow popups…"; `no-data` → "No data available to export yet."), falling back to a generic "export failed" message otherwise
  - Always restores the button to its original state in a `finally` block
- **Toast notifications** (`showExportToast(message, isError)`) — small dark pill at the bottom-center of the screen, auto-dismisses after ~2.4 seconds, styled red for errors

### 3.7 USEG Type Reference Panel

A dedicated reference panel for the 9 USEG gear types used on C3, parsed from a second worksheet in the same uploaded Excel workbook.

**Entry point:** "Available USEG Types" card on the main dashboard (§3.1), below the Section cards.

**Desktop layout:** Left sidebar (230px) listing all type names as clickable buttons; right-hand content panel (up to 1280px wide) shows the selected type's attribute table + reference image.

**Mobile layout (<900px):** Sidebar is hidden; a horizontal scrollable chip bar at the top of the modal takes its place.

#### 3.7.1 Source Data & Parsing

Sheet name: **`USEG-Type`** (case-insensitive match), read from the same workbook as `USLG_WithHeight`.

**Layout (fixed, address-based — not array-index-based, because the sheet's used-range can start at any column):**
- **Row 3** (0-indexed row 2): type names, scanning from **column E** (0-indexed col 4) onward to the sheet's last used column. Column D itself is excluded because it holds the row-label "C3 Nomenclature".
- **Row 2** (0-indexed row 1): "C4 Nomenclature" — same column alignment as row 3. Injected as the **first attribute row** for every type.
- **Column D** (0-indexed col 3), from **row 4** onward: attribute labels (Main Beam, Beam-2/Top Beam, Beam-3/Secondary Beam, Trestle Column, Trestle Bracings, Trestle Dimension, Trestle Height, Footings).
- **Data cells:** intersection of each attribute row and each type column.

**Parser implementation note:** Uses `XLSX.utils.decode_range` + direct cell addressing (`XLSX.utils.encode_cell({r,c})`) rather than `sheet_to_json(ws, {header:1})`, because the latter drops leading empty columns from its array output when the sheet's `!ref` range doesn't start at column A.

**Confirmed real data, 9 types:**
`Type-1`, `Type-3`, `Type-4 (SMD)`, `Type-5 / Hybrid-01`, `Type-6 / Hybrid-02`, `Type-7`, `Type-8`, `Type-9 (SMD)`, `Type-10 (SMD)`

These are embedded as seed data (`USEG_TYPES` JS array) so the panel works immediately without requiring an upload.

#### 3.7.2 Display

- Sidebar/chip item per type → clicking loads that type's full attribute table on the right
- Table: two columns (Attribute | Value), full grid borders on every cell, alternating row shading, hover highlight
- No attribute-count badge — banner shows just the type name
- Below the table: a Reference Image section (§3.7.3)
- **Export dropdown** in the panel header (same component as §3.6.1)

#### 3.7.3 Reference Images — Static, Repo-Hosted

Images are plain static files served from the same GitHub repository as the HTML file — not uploaded through the dashboard UI, not stored in the browser.

- **Base path constant:** `const USEG_IMAGE_BASE_PATH = 'Images/';` — update this single line once the actual repo folder structure is confirmed.
- **Filename convention:** `Type-0X.<ext>` — the number is extracted from the type name (digits immediately following "Type-"), zero-padded to 2 digits:

  | Type name (from sheet) | Image filename expected |
  |---|---|
  | Type-1 | `Type-01` |
  | Type-3 | `Type-03` |
  | Type-4 (SMD) | `Type-04` |
  | Type-5 / Hybrid-01 | `Type-05` |
  | Type-6 / Hybrid-02 | `Type-06` |
  | Type-7 | `Type-07` |
  | Type-8 | `Type-08` |
  | Type-9 (SMD) | `Type-09` |
  | Type-10 (SMD) | `Type-10` |

- **Mixed extensions handled automatically:** the `<img>` tries `.jpg` first; on load failure (`onerror`), retries `.png`, then `.jpeg`, in sequence (`usegImageTryNextExt()`). Falls back to a black "No image available" placeholder if none load — visible to everyone, no password gate.
- **Click-to-zoom:** clicking the loaded image opens a full-screen lightbox overlay (click anywhere, or press Escape, to close).
- **PDF export compatibility:** the "export all types to PDF" feature reuses the same path-resolution + extension-fallback logic inside the print popup's own inline script (escaped as `<\/script>` to avoid the HTML-parser bug described in §5), with a `<base href="...">` tag so relative image paths resolve correctly.

**Deployment note:** drop the 9 reference images into an `Images/` folder alongside the HTML file, named per the table above.

#### 3.7.4 What Was Tried and Reverted

An earlier iteration used a password-gated, per-type image **upload** control with images stored in browser-local IndexedDB (`usegTypeImagesDB`). This was fully removed in favour of the static repo-image approach once the GitHub + Firebase deployment plan was confirmed, since static files work identically for every viewer rather than only on the device that uploaded them.

### 3.8 Button Styling — Update Data & Export

Both primary action buttons share a consistent "polished gradient pill" treatment, colour-coded by function:

| Button | Colour | Meaning |
|---|---|---|
| **⬆ Update Data** (header) | Blue gradient (`#1D6FD8 → #1559B0`) | Bringing data *into* the dashboard |
| **⬇ Export ▾** (modal toolbar / type panel header) | Green gradient (`#1E9E6B → #167A53`) | Sending data *out* of the dashboard |

Both buttons share:
- Soft drop shadow tinted to their own colour, with an inset highlight for depth
- A gentle lift + brighter glow on hover; settles back down on click (`:active`)
- White icon + label, bold 12.5px text, fully rounded pill corners (`border-radius: 8px`)
- The Export button's loading spinner is white-on-translucent-white so it stays visible against the green fill
- The Export dropdown menu items get a faint green-tinted hover state, visually tying the flyout back to the trigger button's colour without competing with it

### 3.9 Live Data Sync — Firebase Firestore

Replaces the earlier browser-local IndexedDB persistence cache entirely. No Firebase Authentication is used — **viewing the dashboard is public**, and **editing remains gated by the existing UI-level password** (§3.5), not by any database-level permission.

**Setup (in `<head>`, before the main script):**
- A `<script type="module">` block loads the Firebase modular Web SDK (v10.12.0) via CDN:
  - `firebase-app.js` → `initializeApp`
  - `firebase-firestore.js` → `getFirestore`, `doc`, `setDoc`, `onSnapshot`
- A `firebaseConfig` object placeholder (`apiKey`, `authDomain`, `projectId`, `storageBucket`, `messagingSenderId`, `appId`) — ships with dummy `"YOUR_..."` values
- **Graceful no-Firebase mode:** if `firebaseConfig.apiKey` is still the placeholder value, the module script logs a console warning and sets `window.__firebaseEnabled = false` — every downstream Firestore call checks this flag and silently no-ops, so the dashboard runs exactly as it did before Firebase was wired in (seed data only, no cross-device sync) until a real config is pasted in.
- On success, the Firestore handle, document reference (`dashboard/latest`), and the `setDoc`/`onSnapshot` functions are exposed on `window.__firestoreDb` / `window.__firestoreDoc` / `window.__firestoreSetDoc` / `window.__firestoreOnSnapshot` — this is the bridge between the `type="module"` script (which can use `import`) and the main classic `<script>` (which can't).

**Writing (`saveDashboardCache()`):**
- Called after every successful Excel upload
- Includes a defensive polling wait (up to 2 seconds, checking every 100ms) for `window.__firebaseEnabled` to be defined, in case this fires before the module script in `<head>` has finished initializing — this guards against a script-execution-order race condition between the deferred module script and the classic script
- Writes `{ records, usegTypes, lastUpdated }` to the single Firestore document via `setDoc()`
- Sets an `isApplyingRemoteUpdate` flag around the write so the realtime listener (which will also fire from our own write) doesn't re-process data we just applied locally

**Reading / live sync (`initDashboardLiveSync()`):**
- Called once on page load (after `buildHero()`, `buildSections()`, `updateUsegTypeCard()`)
- Same defensive polling wait as the write path
- Subscribes via `onSnapshot()` to the single Firestore document — every time it changes (by anyone, from anywhere), the callback fires
- If the document doesn't exist yet (first deploy, nobody has uploaded anything), the callback returns early and the dashboard stays on embedded seed data
- If the change is the echo of our own just-completed write (`isApplyingRemoteUpdate` is true), it's skipped — already applied locally, no need to re-render
- Otherwise, `applyRemoteSnapshot(data)` runs: replaces `RECORDS` / `USEG_TYPES` / `LAST_UPDATED`, rebuilds the hero/sections/type-count, and shows a toast — `"Dashboard updated — new data received"` — so viewers know something changed without needing to refresh

**Firestore security rule** (set once, in the Firebase Console, not in this file):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /dashboard/{document=**} {
      allow read: if true;
      allow write: if true;
    }
  }
}
```
Public read, public write — matches the "anyone with the link can view; shared password for editing" scope decision. **Known tradeoff:** since there's no Firebase Auth, this rule can't actually distinguish who is writing — a person with browser dev tools could write to Firestore directly, bypassing the UI password entirely. Accepted as reasonable for a small internal team tool; the fix path (if ever needed) is enabling Firebase Auth and changing `allow write` to `if request.auth != null;`, without needing to restructure anything else.

---

## 4. Excel File Format Required

### 4.1 Sheet 1 — `USLG_WithHeight`

Data starts from **row 3** (row 1 = main headers, row 2 = sub-headers for merged cells).

#### Required Column Headers (case-insensitive, partial match)

| Column | Header keyword to match | Notes |
|---|---|---|
| Superstructure Erection Method | `superstructure erection method` or `erection method` | Values: `USLG`, `SBS-TwinCell`, `SMD No.X` |
| Chainage From | `chainage` → sub-header `from` | Decimal km (e.g. 30.095) |
| Chainage To | next column after Chainage From | Optional |
| Span (m) | `span (m)` or `span(m)` | Numeric |
| Pier Height | `height of pier above existing ground` | Numeric, metres |
| Applicable USEG | `applicable useg` | Text e.g. `Type-01`, can be blank |
| Status | `erection status` or `status` | `Completed` or `Pending` (or blank → Pending) |
| Location/Remark | `location/remark`, `location / remark`, or `remark` | Free text |
| Pier From | 2 columns before Chainage From | Pier ID e.g. `30P02` |
| Pier To | 1 column before Chainage From | Pier ID e.g. `30P03` |

##### Method Filter Logic
Only rows with these method values are imported:
- `USLG` → Viaduct USEG
- `SBS-TwinCell` → Station Approach USEG
- `SMD No.1` … `SMD No.8` → SMD Ramp (left/right determined by `-L` / `-R` suffix on pier ID)

All other rows (header rows, blank rows, other methods) are skipped automatically.

### 4.2 Sheet 2 — `USEG-Type`

See §3.7.1 for full parsing rules. Summary:

| Cell range | Content |
|---|---|
| Row 3, col E onward | Type names (e.g. `Type-1`, `Type-4 (SMD)`) |
| Row 2, col E onward (same columns) | C4 Nomenclature equivalents — injected as first attribute |
| Col D, row 4 onward | Attribute labels |
| Intersection cells (row 4+, col E+) | Attribute values per type |

Sheet name match is case-insensitive (`useg-type`, `USEG-Type`, etc. all match).

---

## 5. Session Changes Log

### Earlier sessions (carried forward)

**Issue 1 — File Upload Not Opening.** Hidden-input `.click()` blocked in sandboxed iframes → replaced with native visible `input[type="file"]` styled via `::file-selector-button`.

**Issue 2 — Single Filter Only, No Active Label.** Replaced single `statFilter` object with `activeFilters = { span: Set, height: Set, useg: Set, category: Set }` for multi-select AND/OR filtering with named removable chips.

**Issue 3 — Hint Text Showing on Collapse.** Removed conditional hint text rendering.

**Issue 4 — Section Cards Visual Cleanup.** Accent borders, eyebrow label, larger scope number, thinner progress bar, redesigned breakdown rows.

**Feature — USEG Type Reference Panel (initial build).**
Added a second-sheet parser for `USEG-Type`, a modal with type chips and an attribute table, wired into the Excel upload pipeline alongside the existing span-data parser.

**Bug — USEG-Type Sheet Column Index Shift.**
**Problem:** `sheet_to_json(ws, {header:1})` silently drops leading empty columns (A–C) from its returned row arrays when the sheet's `!ref` range starts at column D, causing all column-index assumptions in the parser to be wrong by a fixed offset — confirmed against the real uploaded workbook, which returned only 6 garbled "types" instead of 9.
**Fix:** Rewrote the parser to use `XLSX.utils.decode_range` + direct cell addressing (`encode_cell({r,c})`) instead of array-index assumptions. Verified against the real file: now correctly returns all 9 types with correct names.

**Feature — Reference Card on Main Dashboard.**
Moved the panel's entry point from a header button ("📋 USEG Types") to a dedicated stat-style card below the Section cards.

**Feature — Desktop Sidebar Layout.**
Replaced the horizontal chip-bar-only layout with a left sidebar (230px) + larger right-hand content panel (up to 1280px) on desktop (>900px). Mobile retains the horizontal chip bar. Also: removed the attribute-count badge, added full grid borders to the attribute table, and added "C4 Nomenclature" as an injected first attribute row.

**Feature — Per-Type Images, Take 1 (built then reverted).**
Built a password-gated, per-type image upload/replace/remove UI backed by browser-local IndexedDB, plus a click-to-zoom lightbox. Worked, but was browser/device-local only.

**Feature — Per-Type Images, Take 2 (final).**
Once the GitHub + Firebase deployment plan was confirmed, the IndexedDB upload subsystem was fully removed and replaced with static `<img>` tags pointing at a configurable `Images/` folder path, automatic extension fallback, and a black "no image available" placeholder. Zoom lightbox kept, simplified to work directly off the resolved `<img>` src.

**Feature — Contextual Export (PDF + Excel), initial build.**
Added two separate export buttons to every drill-down modal's toolbar, scoped to exactly the currently filtered (or unfiltered) table. Added a separate "export all types" pair of buttons to the USEG Type Reference panel header.

**Bug — Literal `</script>` Inside Inline Script Breaks the Page.**
**Problem:** The "export all types to PDF" function's `document.write()` call included an inline `<script>...</script>` block. Because this was written as a plain string inside the *page's own* `<script>` block, the literal substring `</script>` inside that string caused the HTML parser to terminate the page's real script tag early — an HTML tokenizer behaviour that ignores JS string/template-literal context entirely. A naive regex-based syntax check missed this because it also stopped at the first `</script>` match.
**Fix:** Escaped the embedded closing tag as `<\/script>`. Verified by extracting the *actual* script boundary (real opening tag to real last closing tag before `</body>`) rather than a naive first-match regex.

**Bug — PDF Export Crash: "Cannot read properties of undefined (reading 'document')".**
**Problem:** `window.open('', '_blank')` can return `null` when a popup is blocked; the code unconditionally called `.document.write(...)` on the result.
**Fix (initial):** Added a guard — alert the person and return early if `win` is falsy.
**Fix (final, this version):** Changed to `throw new Error('popup-blocked')` instead of alerting directly, so the new `runExport()` wrapper (§3.6.1) can catch it and show a proper toast with specific wording.

**Known limitation — Claude App In-App Preview (Android).**
Confirmed during testing: exporting (both PDF and Excel) does nothing when the dashboard is opened inside the Claude mobile app's in-app artifact preview (a sandboxed webview) — `window.open()` and the `download`-attribute-based file-save pattern used by `XLSX.writeFile()` are both commonly no-op'd inside app-embedded webviews, with no error surfaced. This is a platform constraint of testing inside an in-app preview, not a bug in the dashboard code. **Expected to work normally once deployed and opened in a real mobile browser** (Chrome on Android, Safari on iOS).

### This version's sessions

**Feature — Export Buttons Consolidated to a Single Dropdown.**
Replaced the two separate "⬇ Excel" / "🖨 PDF" buttons (in both the drill-down modal and the USEG Type panel) with one "⬇ Export ▾" button that opens a small dropdown menu with both options. Added `toggleExportDropdown()` / `closeAllExportDropdowns()`, an outside-click handler, and Escape-key support.

**Feature — Export UX Polish: Toasts, Loading States, Branded Output.**
Added `showExportToast()` (success/error toast notifications), `runExport()` (wraps every export call with a loading spinner on the button, double-click guarding, and toast feedback on completion or failure). All four export functions (`exportDrillToExcel`, `exportDrillToPDF`, `exportAllUsegTypesToExcel`, `exportAllUsegTypesToPDF`) were updated to: return a descriptive success-message string instead of nothing; `throw` typed errors (`'popup-blocked'`, `'no-data'`) instead of alerting directly; include a branded "L&T Construction — MAHSR Package C3" header in both Excel and PDF output; and (PDF only) include page numbers via a CSS `@page` rule.

**Feature — Stylish Gradient Buttons.**
Restyled the "Update Data" button (blue gradient, upload-arrow icon, hover lift + glow) and the "Export" button (green gradient, same hover treatment, white spinner for visibility) to read as the dashboard's two clear primary actions, colour-coded by direction of data flow (blue = in, green = out). Dropdown menu item hover states tinted to match.

**Feature — Firebase Firestore Live Sync (replaces IndexedDB persistence).**
Removed the browser-local IndexedDB dashboard-data cache entirely. Added a `<script type="module">` block in `<head>` for the Firebase Web SDK (v10.12.0, CDN-loaded) with a `firebaseConfig` placeholder, and a `window.__firebaseEnabled` flag bridging the module script to the main classic script. `saveDashboardCache()` now writes to a single Firestore document (`dashboard/latest`) on every successful upload; `initDashboardLiveSync()` subscribes to that same document via `onSnapshot()` so every open tab/device updates **live**, with a toast notification, the instant anyone else uploads new data — no refresh needed. No Firebase Authentication is used; viewing stays public and editing stays gated by the existing UI-level password, per explicit scope decision. Both the write and read paths include a short defensive polling wait to avoid a script-load-order race against the deferred module script. Graceful no-Firebase fallback preserved: if `firebaseConfig` is left as placeholder values, the dashboard behaves exactly as it did before Firebase was introduced.

---

## 6. Regeneration Prompt

Use this prompt to rebuild the dashboard from scratch in a new Claude conversation:

---

> Build a single-file HTML dashboard for MAHSR C3 USEG installation tracking. No frameworks, no server required to run it locally; optionally connects to Firebase Firestore for live shared data sync once deployed. Pure HTML + CSS + JS.
>
> **Data source 1 — span records:** Parse a `.xlsx` file (sheet `USLG_WithHeight`) using SheetJS (CDN: `https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.18.5/xlsx.full.min.js`). Import rows where `Superstructure Erection Method` column is `USLG`, `SBS-TwinCell`, or `SMD No.X`. Map them to categories: USLG → Viaduct USEG, SBS-TwinCell → Station Approach USEG, SMD → SMD Ramps. Parse pier IDs, chainage from/to, span (m), pier height (m), applicable USEG type, status (Completed/Pending), location/remark. Derive section from chainage: Section-01 (21.15–53.315), Section-04 (53.315–84.57), Section-02 (84.57–127.6), Section-03 (127.6–156.601).
>
> **Data source 2 — USEG type reference (second sheet, same workbook):** Parse sheet `USEG-Type` using direct cell addressing via `XLSX.utils.decode_range` + `encode_cell` (NOT `sheet_to_json` array-index reads, which break when the sheet's used range doesn't start at column A). Row 3 (0-indexed row 2), columns E onward = type names. Row 2 (0-indexed row 1), same columns = "C4 Nomenclature" — inject as the first attribute for each type. Column D (0-indexed col 3), row 4 onward = attribute labels (Main Beam, Beam-2/Top Beam, Beam-3/Secondary Beam, Trestle Column, Trestle Bracings, Trestle Dimension, Trestle Height, Footings). Embed the 9 real types as seed data so the panel works without upload.
>
> **Seed data:** Embed JSON of ~90 span records as `SEED_RECORDS`, and the 9 USEG types as `USEG_TYPES`, so the dashboard works before any upload or Firebase connection.
>
> **Upload:** Password-gated via native `input[type="file"]` (no hidden input tricks). Parses both sheets from one uploaded workbook in a single action. On successful parse, rebuild all cards live, and push the parsed data to a single Firestore document if Firebase is configured (see below), so other viewers update automatically.
>
> **Firebase (optional, graceful fallback):** In `<head>`, add a `<script type="module">` loading the Firebase Web SDK v10.12.0 from `https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js` and `firebase-firestore.js`. Include a `firebaseConfig` object with placeholder `"YOUR_API_KEY"`-style values. If the placeholder is still present, set `window.__firebaseEnabled = false` and skip Firestore entirely — the dashboard must run identically to a no-Firebase version in that case. Otherwise initialize Firestore, expose the doc reference and `setDoc`/`onSnapshot` functions on `window.__firestore*` globals (since the main script is a classic script and can't use `import` directly), write to a `dashboard/latest` document on every upload, and subscribe to it with `onSnapshot()` on page load for live updates — applying a toast notification when a remote update is received. Use no Firebase Authentication; the Firestore security rule should allow public read and public write, since editing is gated by the in-app password instead, not by auth.
>
> **Home screen:**
> - Hero card (Grand Total scope, completed, % + progress bar) — clickable to drill all
> - KPI cards row: Viaduct USEG (blue), Station Approach USEG (purple), SMD Ramps (orange) — each clickable
> - Section cards grid: one per section, accent left border (blue/purple/green/orange), eyebrow "MAHSR C3", section name, chainage label, total scope (large), progress bar, breakdown rows (dot + label + completed/total)
> - Reference card row below sections: "Available USEG Types" stat card with live count, clickable to open the USEG Type Reference panel
> - Header "Update Data" button styled as a blue gradient pill with an upload icon, hover-lift and glow
>
> **Drill-down modal:**
> - Searchable, sortable, paginated (25/page) table
> - Columns: Span ID, Chainage (km), Span (m), Category, Pier Height (m), Applicable USEG, Location/Remark, Status
> - Status badges: Completed (green), Pending (grey)
> - Collapsible summary panels (Show/Hide toggle): Span Length Mix, Pier Height Groups (≤10m/10–15m/15–20m/>20m), Category Summary, USEG Types Available
> - Multi-select filters: clicking any stat row/chip toggles a Set; AND across types, OR within type; named active filter chips appear below toggle with individual remove (✕) and "Clear all"; badge shows "N filters active"
> - **One** export button styled as a green gradient pill ("⬇ Export ▾") opening a small dropdown with Excel and PDF options. Wrap the export call in a loading-spinner + toast-notification helper. Excel via SheetJS `aoa_to_sheet`/`writeFile` with a branded header block; PDF via a new window with print-styled HTML (branded header, page numbers via CSS `@page`) and `window.print()`. On failure (blocked popup, no data), throw a typed error and show a specific toast instead of crashing or alerting directly.
>
> **USEG Type Reference panel:**
> - Desktop: left sidebar (230px) listing type names, right content panel (wide, up to ~1280px) showing the selected type's attribute table (full grid borders, no count badge) and a reference image below it
> - Mobile (<900px): horizontal chip bar instead of sidebar
> - Reference images are static files served from a configurable base path (e.g. `Images/Type-01.jpg`), filename = zero-padded number extracted from the type name; `<img onerror>` tries `.jpg` → `.png` → `.jpeg` in sequence, falling back to a black "No image available" placeholder. Click image to open a full-screen zoom lightbox.
> - Same single export dropdown button in the panel header, covering all types in a single click (one sheet per type for Excel, one page per type with image for PDF) regardless of which type is currently selected.
>
> **Design:** Navy/white professional engineering aesthetic; Inter font; blue `#1D6FD8`/`#2D6CDF`, purple `#7B3FE4`, orange `#E08A1E`, green `#1E9E6B`/`#1C9E5E`; card `border-radius: 10px`; subtle hover lift on cards; the two primary action buttons (Update Data, Export) use gradient fills in their respective colours with soft shadows and a hover-lift/glow effect, distinct from the flatter neutral styling used elsewhere.

---

## 7. File Locations

| File | Description |
|---|---|
| `index.html` (or `MAHSR_C3_USEG_Dashboard_v3.html`) | Current live dashboard (all fixes and features applied) |
| `README.md` | Repo front page — quick start, deployment, password, image folder convention |
| `MAHSR_C3_USEG_Dashboard_SPEC.md` | This document — full technical spec, change log, regeneration prompt |
| `Images/Type-01.jpg` … `Images/Type-10.jpg` | USEG Type reference images, path configurable via `USEG_IMAGE_BASE_PATH` |

---

## 8. Known Open Items / Deferred Decisions

1. **Cross-linking** between the `Applicable USEG` column in span records and the USEG Type Reference panel (e.g. clicking a `Type-01` value in the drill-down table to jump to its spec) was suggested but explicitly deferred. Note the two systems use slightly different naming conventions (`Type-01` in span data vs. `Type-1` in the type reference sheet) — this mapping would need to be defined if/when this is built.
2. **Firebase Authentication** is intentionally not used. Viewing is public; editing is gated by a UI-level shared password only, not by any database-level permission. The Firestore rule is public-read/public-write as a direct consequence. If tighter control is ever needed, the documented fix path is enabling Firebase Auth and changing the Firestore rule to require `request.auth != null` for writes.
3. **Mobile export verification** — export (PDF + Excel) is confirmed broken inside the Claude app's in-app preview webview; not yet verified end-to-end in a real mobile browser (Chrome/Android or Safari/iOS) post-deployment. Recommended as a pre-launch check.
4. **Actual `Images/` folder path** — `USEG_IMAGE_BASE_PATH` should be confirmed against the real GitHub repo structure once deployed and adjusted if the folder is named or located differently.
5. **Firestore document size** — all records and USEG types are currently stored in a single Firestore document (`dashboard/latest`). Firestore documents have a 1MiB size limit; this is very unlikely to be hit at the current data volume (~90–100 span records + 9 type specs) but worth knowing if the dataset grows substantially, in which case splitting into a `records` collection (one document per span) would be the next step.

---

*Last updated: June 2026 — MAHSR C3, L&T Construction*
