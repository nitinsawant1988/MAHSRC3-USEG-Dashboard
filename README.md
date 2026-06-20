# MAHSR C3 — USEG Dashboard

A single-file HTML dashboard tracking **Underslung Erection Gear (USEG) installation progress** across the Mumbai–Ahmedabad High Speed Rail (MAHSR) **Package C3** viaduct corridor — Km 21.150 to Km 156.600.

No backend, no build step, no framework. One `index.html` file, opened in any browser.

---

## What it does

- Tracks erection progress for three categories of spans — **Viaduct USEG**, **Station Approach USEG**, and **SMD Ramps** — across four corridor sections (Section-01 to Section-04)
- Drill down from any summary card into a searchable, sortable, filterable table of individual spans
- Reference panel for the 9 **USEG gear types** used on C3, with full spec sheets and site photos
- Export any filtered view to **Excel or PDF**, on demand
- Data updates **live** — when one person uploads a new Excel sheet, everyone viewing the dashboard sees it instantly, no refresh needed

---

## Live dashboard

Once deployed (see [Deployment](#deployment) below):

```
https://<your-username>.github.io/<repo-name>/
```

---

## Updating the data

1. Click **⬆ Update Data** in the header
2. Enter the password (see `UPLOAD_PASSWORD` in the HTML source — currently `mahsr2026`)
3. Choose your `.xlsx` file and upload

The workbook must contain two sheets:

| Sheet name | Contents |
|---|---|
| `USLG_WithHeight` | Span-by-span erection records (chainage, span length, pier height, status, etc.) |
| `USEG-Type` | The 9 USEG type spec sheets (attributes per type) |

Full column-by-column format requirements are in [`MAHSR_C3_USEG_Dashboard_SPEC.md`](./MAHSR_C3_USEG_Dashboard_SPEC.md).

On a successful upload, the parsed data is pushed to Firestore (if configured — see below) so every open browser tab updates automatically. If Firestore isn't configured, the dashboard falls back to working entirely from the data embedded in the page (no sharing across devices, but otherwise fully functional).

---

## Reference images

The USEG Type Reference panel shows a photo/diagram for each of the 9 types. These are **not** uploaded through the dashboard — they're static files in this repo.

Drop them into the `Images/` folder, named to match the type number:

```
Images/
├── Type-01.jpg   (or .png / .jpeg)
├── Type-03.jpg
├── Type-04.jpg
├── Type-05.jpg
├── Type-06.jpg
├── Type-07.jpg
├── Type-08.jpg
├── Type-09.jpg
└── Type-10.jpg
```

The dashboard automatically tries `.jpg`, then `.png`, then `.jpeg` for each — you don't need to tell it which extension you used. If an image is missing, it shows a placeholder instead of breaking.

> The base folder path is set via `USEG_IMAGE_BASE_PATH` near the top of the HTML's `<script>` section — change this one line if you use a different folder name or path.

---

## Deployment

### 1. Host on GitHub Pages

1. Push this repo to GitHub (keep it **public** — required for free Pages hosting)
2. Rename the dashboard file to `index.html` if it isn't already
3. Repo → **Settings → Pages** → Source: **Deploy from a branch** → Branch: `main`, folder: `/ (root)` → Save
4. Your dashboard is live at `https://<your-username>.github.io/<repo-name>/` within about a minute

### 2. Connect Firebase (for live, shared data sync)

This is optional but recommended — without it, an upload only updates the browser that did the uploading.

1. Create a project at [console.firebase.google.com](https://console.firebase.google.com)
2. Enable **Firestore Database** (any region; Production mode)
3. Set the Firestore rules to allow public read/write (no Firebase Auth is used — viewing is public, and editing stays gated by the in-app password instead):
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
4. Register a **Web app** in your Firebase project to get a config object
5. Open `index.html`, find the `firebaseConfig` placeholder near the top of `<head>` (search for `YOUR_API_KEY`), and paste in your real values
6. Commit and push — live sync is now active

> **Security note:** because there's no Firebase Auth, the public-write rule means anyone with the live URL could write to Firestore directly (bypassing the in-app password) using browser dev tools. The password protects the normal upload flow, not the database itself. This is an accepted tradeoff for an internal team tool with a small, trusted audience — not a public-facing system.

---

## Architecture notes

- **No frameworks** — vanilla HTML/CSS/JS, single file
- **SheetJS** (`xlsx.full.min.js`, via CDN) parses the uploaded Excel workbook entirely in-browser
- **Firebase Firestore** (via the modular Web SDK, also CDN-loaded) is the only backend dependency, and it's entirely optional — the dashboard degrades gracefully to "seed data only" if not configured
- **PDF exports** use the browser's native print dialog (`window.print()`), not a PDF library — this guarantees the output matches the on-screen styling exactly
- **Excel exports** use SheetJS's write functions, client-side, no server round-trip

---

## Project structure

```
.
├── index.html                          ← the dashboard itself
├── Images/                             ← USEG type reference photos (you provide these)
│   └── Type-01.jpg ... Type-10.jpg
├── README.md                           ← this file
└── MAHSR_C3_USEG_Dashboard_SPEC.md      ← full technical specification & regeneration prompt
```

---

## Full specification

For the complete column-by-column Excel format, parsing logic, every feature in detail, and the session-by-session change log (including bugs found and fixed), see [`MAHSR_C3_USEG_Dashboard_SPEC.md`](./MAHSR_C3_USEG_Dashboard_SPEC.md).

---

*MAHSR Package C3 — L&T Construction*
