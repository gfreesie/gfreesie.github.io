# Mortgage Pipeline Tracker — Project Recap & Handoff

> Last updated: April 23, 2026
> **Context-clear handoff doc — read this first at the start of every new session.**

---

## What This Is

An internal team client tracker for mortgage files in process. Built as a single self-contained HTML file — `mortgage-tracker.html` — no installs, no server, no dependencies. Open it in any browser. Data persists via localStorage.

**File location:** `C:\Users\Crypt\OneDrive\Desktop\Claude Projects\LandHome\Deal Tracker\mortgage-tracker.html`

**Team users:** GA (Garry A.), SG (Steve G.), RH (Roxanne H.)

---

## Architecture At a Glance

- Pure vanilla HTML / CSS / JavaScript — zero CDN, zero frameworks, works fully offline
- Single `<script>` block; all logic is pure immutable functions
- State lives in a single `S` object; `commit(changes)` applies changes + calls `render()`
- `localStorage` key: `'pt_v3'`
- **Test suite runs on every page load** — if any test fails, the app blocks render and shows an error. Currently 38 tests, all passing.
- Canvas 2D used for: constellation background (90-node neural net), 3D fractal user icons, black hole animation

---

## Pipeline Stages (current STAGES array order)

| Index | ID | Label | Color |
|---|---|---|---|
| 0 | `unqual` | Unqual / Cancelled | `#dc2626` red |
| 1 | `app-docs` | App / Docs | `#d97706` orange |
| 2 | `processing` | Processing | `#0891b2` cyan |
| 3 | `uw-conditional` | UW / Conditional | `#6d28d9` purple |
| 4 | `ctc` | Clear to Close | `#059669` green |
| 5 | `funded` | Funded | `#16a34a` green |

**Important:** `unqual` is NOT part of the linear pipeline. `ACTIVE_STAGE_IDS` excludes it. `moveClient()` fwd/back cannot reach or leave unqual — only drag-and-drop or explicit reassignment via the popup can send files there.

---

## Key Constants & State

```javascript
const STAGES        // full array (6 stages including unqual)
const STAGE_IDS     // ['unqual','app-docs','processing','uw-conditional','ctc','funded']
const ACTIVE_STAGE_IDS  // ['app-docs','processing','uw-conditional','ctc','funded']
const USERS = { GA, SG, RH }  // color, glow, name per user
const STORAGE_KEY = 'pt_v3'

S = {
  clients:[],         // array of client objects
  user: null,         // 'GA'|'SG'|'RH' — set on home screen
  search: '',
  filterLO: 'All',
  modal: null,        // null | {type:'detail'|'add'|'edit'|'unqual-list', id?}
  dragId: null,
  dragOver: null,
  noteInputOpen: false,
  noteGlitchPhase: 'idle',  // 'idle'|'glitching'|'who'
  pendingNote: '',
  showOlderNotes: false,
}
```

## Client Object Schema

```javascript
{
  id: Number,           // Date.now() at creation
  name: String,         // borrower name
  loanAmount: Number,
  loanType: String,     // 'Conventional'|'FHA'|'VA'|'USDA'|'Jumbo'|'HELOC'|'Non-QM'
  propertyAddress: String,
  rateLocked: Boolean,
  stage: String,        // one of STAGE_IDS
  closingDate: String,  // 'YYYY-MM-DD' or ''
  lo: String,           // 'GA'|'SG'|'RH'
  notes: Array,         // [{id, text, author, ts}] — newest first
  createdAt: String,    // ISO timestamp
}
```

---

## Features Built (complete list)

**Board & Cards**
- Kanban board — 6 columns (unqual + 5 pipeline), each color-coded with stage glow
- Client cards: borrower name, loan amount (grey bold with green glow), loan type (green), lock status (green=locked, yellow=floating). No LO tag or note preview on quick cards.
- Funded column: replaces lock tag with "✦ FUNDED" green tag; shows funded date; no Back button
- Urgency dots: red pulse when closing ≤7 days, yellow when ≤14 days
- Drag & drop between any columns; `setClientStage()` on drop
- Fwd → / ← Back buttons on each card (disabled at pipeline endpoints); Back is subdued red, Fwd is green
- Click anywhere on a card → opens detail modal

**Unqual / Cancelled Column (leftmost)**
- Renders an animated black hole canvas instead of individual cards
- Black hole shows file count in the event horizon center; slowly churning accretion disk with stars
- **Click the black hole** → opens the Unqual List modal showing all files with inline reassignment
- Unqual List modal: each file has a stage dropdown + "Restore →" button + "Detail" link
- Files in unqual: detail modal shows "⚫ Removed from Pipeline" + "← Restore to Pipeline" button

**Detail Modal**
- Stage indicator, borrower name, loan amount (grey bold green glow), full data grid
- Pipeline progress bar (shows active pipeline stages only, not unqual)
- Notes & Activity section (see Notes System below)
- Actions: Back / Advance / Edit / Delete (unqual files: Restore to Pipeline / Edit / Delete)

**Notes System**
- "+ Add Note" → opens textarea; first focus auto-inserts `•` bullet; Enter auto-continues bullet
- Submit → 300ms hacker glitch text-scramble → "Says who?" prompt slides in
- GA / SG / RH buttons → finalizes note with author tag and timestamp; prepends to history
- Shows most-recent 3 notes; "Show X older…" toggle for full history
- State machine: `noteGlitchPhase` = `'idle'` → `'glitching'` → `'who'` → `'idle'`

**Add / Edit File**
- Full form modal: all fields, rate lock toggle, stage selector
- `submitForm()` calls `addClient()` or `updateClient()` then `commit()`

**Home Screen**
- Full-screen constellation canvas (90 nodes, mouse-attraction, edge drawing)
- "PIPELINE" title with gradient
- 3 user cards: GA (Merkaba star-tetrahedron), SG (3D atom with 3 orbital rings + electrons), RH (4D tesseract hypercube with stereographic projection)
- All icons are live Canvas 2D animations using real 3D/4D rotation matrices
- Clicking a user card → 320ms glitch-fade → board screen

**Header**
- Live stats: active files, total volume, rate locked count
- Search input (filters by name or address)
- LO filter dropdown
- User badge (click to go home)

---

## Pure Logic Functions (all immutable)

```javascript
migrateClients(clients)      // handles legacy string notes → array
moveClient(clients, id, dir) // uses ACTIVE_STAGE_IDS; unqual clients untouched
addClient(clients, data)
deleteClient(clients, id)
updateClient(clients, id, changes)
setClientStage(clients, id, stageId)  // use this to move to/from unqual
addNote(clients, id, note)
filterClients(clients, search, lo)
fmt(n)           // '$485,000'
esc(s)           // XSS-safe HTML escaping
fmtDate(d)       // 'Apr 22, 2026'
fmtDateTime(ts)  // 'Apr 22 10:30 AM'
daysUntil(d)     // days from today, null if empty
getStage(id)
getClient(id)
```

---

## CSS Architecture

- All CSS variables in `:root` — `--bg`, `--panel`, `--card`, `--border`, `--accent`, `--uv`, `--text`, `--text-s`, `--text-d`, `--red`, `--green`, `--yellow`, user colors `--ga/--sg/--rh`
- Glassmorphism: `backdrop-filter: blur() saturate()` on all panels, modals, cards, header
- Animations: `uv-border-pulse` (columns/cards), `neon-pulse`, `slide-up`, `modal-in`, `overlay-in`, `glitch-shake`, `says-who-slide`, `who-btns-slide`
- Card tags: `.tag.lt` (loan type, green), `.tag.locked` (green), `.tag.float` (yellow)
- Card amount: `.card-amt` — grey bold, green text-shadow glow
- Black hole canvas: `.bh-canvas` — cursor pointer, hover brightens

---

## What's Next (prioritized backlog)

### 1. Data Export / Import *(highest priority)*
One-click JSON export button in the header saves the full `S.clients` array as a `.json` file. Import button (file picker) reads JSON back and replaces the board. Keeps it self-contained with zero backend. Useful for backups and syncing between machines.

**Implementation notes:**
- Export: `URL.createObjectURL(new Blob([JSON.stringify(S.clients,null,2)],{type:'application/json'}))` → create `<a>` and click it
- Import: `<input type="file" accept=".json">` → `FileReader.readAsText` → `JSON.parse` → run through `migrateClients()` → `commit({clients: parsed})`
- Add two small icon buttons to the header (next to `+ New File`)

### 2. Priority / Flag System *(medium priority)*
A "flagged" state per file — a quick star or `!` icon on the card. Flagged cards float to the top of their column. Useful for files needing urgent attention regardless of stage or closing date.

**Implementation notes:**
- Add `flagged: Boolean` to client schema
- `renderCard` renders a `⭐` or `!` icon button; `onclick` toggles the flag via `updateClient`
- `renderBoard` sorts each column's cards: flagged first, then by closing date
- Add flag toggle to detail modal actions
- CSS: flagged card gets a subtle gold border pulse

### 3. Closing Date Sort Within Columns *(low priority)*
Cards currently render in creation order. Sort by closing date ascending (soonest first) within each column. Cards with no closing date go to the bottom.

---

## Session 5 — Completed Work

### Features Added
- **Data Export / Import** — header ↓ Export (JSON download) and ↑ Import (file picker + `migrateClients` validation)
- **Priority / Flag System** — `★` / `☆` button per card; flagged cards float to top; gold border glow; flag toggle in detail modal
- **Closing Date Sort** — cards sort within columns: flagged first → soonest closingDate → no-date last
- **Inline Detail Edit** — clicking the data grid makes all fields editable (name, amount, type, LO, date, address, rate lock toggle); ✓ Save / ✕ Discard actions; `toggleEditLock()` uses direct DOM to avoid re-render stomping typed values
- **Funded Column — Green Nebula Canvas** — replaces card stack with `drawFundedStar()` green star/nebula animation (counter-rotating particle rings, corona rays); click opens Funded Files modal
- **Funded Files Modal** — lists funded files; red ↩ Revert button reverts to `previousStage` (stored on transition to funded)
- `previousStage` tracked in both `moveClient()` and `setClientStage()` whenever a client moves TO funded

### Session 5 — Nebula Transitions + Dead Code Removal

**Nebula Warp Transitions:**
- `@keyframes modal-in` — replaced simple scale with perspective/blur/hue-rotate warp (0%: scale 0.85, rotateX 12°, blur 12px, hue-rotate 80°; sweeps to crisp at 100%)
- `@keyframes overlay-in` — hue-rotate sweep (55° → 0°) as overlay fades in
- `.nebula-ripple` CSS class + `@keyframes nebula-ripple` — radial gradient (purple→cyan→transparent) that scales from 0 to 2.4× and fades; `mix-blend-mode:screen`
- Global JS click listener (`document.addEventListener('click', ...)`) spawns `.nebula-ripple` div at click coordinates, auto-removes after 600ms
- `:active` warp flash on `.card`, `.btn`, `.cbtn`, `.ucard`, `.btn-io`, `.btn-add-note`, `.uq-restore-btn`, `.who-btn` — brightness+hue-rotate on press

**Dead Code Removed:**
- `@keyframes neon-pulse`, `spin-cw`, `spin-ccw`, `ga-wrap-rot`, `sg-wrap-rot`, `rh-wrap-rot` (all from old SVG icon approach)
- CSS classes `.finner`, `.fsv`, `.ga-finner`, `.sg-finner`, `.rh-finner`, `.ga-r1–r4`, `.ga-core`, `.sg-r1–r5`, `.sg-core`, `.rh-r1–r3`, `.rh-l`, `.rh-core` (all dead SVG icon CSS)
- CSS vars `--panel`, `--card`, `--card-h`, `--border-hi`, `--accent-d` (never referenced)
- Duplicate `backdrop-filter:blur(12px)` in `.ucard`
- CSS classes `.card-funded`, `.cf-l`, `.cf-v` (funded column renders canvas, not cards)
- Dead `isFunded` branches in `renderCard()` (funded cards never render on board)

**Test suite:** 48 tests, all passing. File: 1960 lines.

---

## Session 6 — Completed Work

### Features Added
- **LO Tools Suite** — right-side slideout panel (300px, glassmorphism, z-index 24); toggle tab on right edge shows ⚡ Tools vertically; appears only on board screen, hidden on home screen; panel slide-in bug fixed (was keeping `hidden`/`display:none` class on open — now removes `hidden` first, adds `open` on next rAF, restores `hidden` after 370ms close transition)
- **10-Year Treasury + Spread Tracker** — live Canvas chart (last 7 trading days via Yahoo Finance `%5ETNX` API, synthetic fallback `[4.41,4.38,4.42,4.35,4.29,4.33,4.31]`); cyan area fill + line for 10-yr; dashed green overlay for estimated 30-yr; pulsing live dot animation (`chartPulse` sine wave); adjustable spread input (default 175bps); 7D high/low/change stats panel
- **30-Year Fixed National Avg** — Freddie Mac PMMS rate pulled from FRED public CSV (`MORTGAGE30US`); cached in `localStorage` key `pt_30yr` and refreshed once daily at 9am; displays rate + "Freddie Mac PMMS {date}" label in the 10-YR tool panel; falls back gracefully if API unavailable
- **Payment Calculator** — loan amount + rate + term → live-calculated monthly P&I, total interest, total paid (standard amortization formula)
- **DTI Calculator** — gross income + housing PITI + other debts → front-end and back-end DTI with animated color bars; pass/fail color coding (green ≤28/45, yellow ≤31/57, red above); reference line `Conv: 28/45  FHA: 31/57  VA: 41`
- **Max Qualifying Loan** — income + rate + term + other debts + target DTI → max loan amount, max payment, resulting back DTI
- **FHA/Conforming Limit Lookup** — 27-market searchable table; **2026 limits** (updated from official FHFA/HUD sources); baseline conforming $832,750, FHA floor $541,287, high-cost ceiling $1,249,125; key market specifics: Seattle $1,063,750, San Diego $1,104,000, Denver $862,500, Boston/LA/SF/NYC/DC/HI/AK $1,249,125; high-cost markets highlighted in purple, elevated in green; searches by market name or state abbreviation

### Architecture
- `toggleTools()` / `showTool(id)` — panel open/close and tab switching
- `fetchChartData()` — Yahoo Finance fetch with synthetic fallback; `startChartAnim()` — rAF loop; `redrawChart()` — full Canvas 2D redraw each frame
- `fetch30YrRate()` / `show30YrRate()` — FRED CSV fetch, localStorage cache keyed `pt_30yr`, staleness check against 9am daily threshold
- `calcPayment()`, `calcDTI()`, `calcMaxQual()` — all live on `oninput`, no button required
- `LIMITS_DATA` array (27 markets), `renderLimits(data)`, `searchLimits()` — filtered on keypress
- `selectUser()` now shows `#tools-tab`; `goHome()` hides it, closes panel, cancels rAF loop

**Test suite:** 51 tests, all passing. File: ~131KB.

---

*Open `mortgage-tracker.html` in any browser to use the tracker. All data auto-saves to localStorage.*
