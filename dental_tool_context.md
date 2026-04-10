# Dental Quotation Tool — Project Context

## What this is
A single-file HTML dental quotation tool for Singapore clinics. Handles Medisave, CHAS, GST, orthodontics, instalment plans, QR-based patient signatures (Firebase), and a treatment timeline. Everything runs client-side — no server needed.

---

## File
`index.html` — the current working version (hosted on GitHub Pages).

---

## Tech stack
- Pure HTML/CSS/JS, single file
- SheetJS (`xlsx.full.min.js`) for Excel export
- QRCode.js for QR generation
- Firebase Realtime Database for signature sessions (URL: `https://dental-quotation-default-rtdb.asia-southeast1.firebasedatabase.app`)
- `window.print()` for PDF generation

---

## Key data structures

### summaryItems[] — the cart
```js
{
  _id,          // unique number from nextId()
  _sec,         // 'surg' | 'ns' | 'ortho' | 'prostho'
  label,        // display name
  remark,       // optional italic subtitle
  qty,
  unit,
  mv,           // raw Medisave (pre-GST)
  cashBefore,   // raw cash top-up (pre-GST)
  subsidyTotal, // CHAS subsidy amount
  feeOnly,      // true for cash-only procedures
  deposit,      // ortho only
  _chasTier,    // 'orange'|'blue'|'mg'|'pg'|'' (NS items only)
}
```
GST is **never stored** — recalculated live in `updateGrandFromItems()`.

### surgState[code] — surgical live state
`{ on, qty, crown, apptMode, cashTopup, remark, capOverflowToCash, extraMiscToCash }`

### nsState[id] — non-surgical live state
`{ on, qty, fee, remaining, materials[], remark }`

### orthoItems[] / miscItems[] — dynamic arrays
`{ id, name, fee, qty, deposit, remark }`

---

## App structure (HTML order)
```
Signing page (patient-facing, shown when ?sign= param present)
QR modal (desktop waiting overlay)
.wrap
  Brand bar (4px navy, #0C447C) + clinic name
  Patient name + Doctor inputs (screen only)
  ── Surgical section (collapsible, no-print)
  ── Non-Surgical section (collapsible, no-print) + CHAS tier selector
  ── Orthodontics section (collapsible, no-print)
  ── Miscellaneous section (collapsible, no-print)
  ── #combined-summary (hidden until first item added)
    Print-only patient bar (#pat-bar-print)
    Treatment Summary title + GST badge + GST toggle
    Summary table (#combined-tbody)
    Metric cards (Total Fee | Patient Cash Payable | Subsidies)
    GST breakdown note (#gst-mv-note)
    #reorder-wrap (flex container for print reordering)
      #instalment-card (outer wrapper, always visible)
        #instalment-panel (inner, shown when checkbox ticked)
      #tl-print-section (timeline wrapper)
      #ack-block (acknowledgement + signature)
  Buttons (Print / Save PDF, Download Excel, Clear)
  Footer note
```

---

## PDF / Print behaviour
- `window.print()` — browser handles PDF
- `@media print` hides all `.no-print` elements
- **Print page order (CSS `order` on `#reorder-wrap` flex container):**
  - Page 1: Summary + metric cards
  - Page 2: Instalment plan + Signature (`#instalment-card` order:1, `#ack-block` order:2) — new page via `page-break-before:always` on `#instalment-card`
  - Page 3: Treatment Timeline (`#tl-print-section` order:3) — new page via `page-break-before:always`
- Screen order is unchanged (DOM order): Instalment → Timeline → Signature
- Brand bar and patient bar shown in print via explicit print CSS
- `<select>` in timeline visit headers is `.no-print`; a sibling `.print-only-tl-date` `<span>` shows the plain text date instead (prevents all 36 dropdown options printing)

---

## Instalment panel
- Outer wrapper: `#instalment-card` (always in DOM, used for page-break targeting)
- Inner content: `#instalment-panel` (display:none until checkbox ticked)
- Options: 3, 6, 9, 12, 24, 36, 48 months
- Only cash top-up + GST on cash top-up is instalment-eligible
- Upfront items (GST on Medisave, GST on CHAS, ortho deposit) shown separately

---

## Treatment Timeline feature
All state in JS (not Firebase):

```js
tlVisits[]     // [{ id, label, tIds:[], date:{m,y} }]
tlWaits{}      // { 'v1_v2': {w:'4', m:'3'} } — weeks + months between visits
tlUnassigned[] // array of summaryItems._id values not yet assigned
tlGenericItems[] // [{ _id (negative), label, mv:0, cashBefore:0, subsidyTotal:0 }]
```

Key functions:
- `tlToggle()` — checkbox show/hide, calls `tlSync()`
- `tlSync()` — reconciles summaryItems with tlVisits/tlUnassigned
- `tlRender()` — re-renders full timeline UI
- `tlSetWait(key, field, val)` — updates wait period, triggers `tlPropagateFrom()`
- `tlPropagateFrom(idx)` — auto-calculates subsequent visit dates
- `tlAddGeneric()` — adds generic treatment to unassigned pool
- Drag-and-drop uses mouse events (not HTML5 drag API)
- Pills are full-width block rows, support reordering within visit boxes
- Fee Type toggles: Total Fee, Medisave, Cash, CHAS (all checked by default)
- Generic presets: Stitches Removal, Implant Crown Issue, Implant Denture Issue, Custom…
- Pastel border colours cycle per visit: `['#a8c8e8','#a8d5b5','#d5b8e0','#f0c9a0','#f0a8b8','#a8d8d8']`
- Wait period options — Months: 3,4,5,6,7,8 | Weeks: 2,4,6,8

---

## Medisave calculation — getSurgLine(p)
Handles: session cap ($5,290), overflow to cash, same vs different appointments, implant-only vs implant+crown, miscOnce (Alveolectomy), flat-rate Medisave, cash-only.

Returns: `{ qty, mv, gstOnMv, cashBefore, cashTotal, total, extraMisc, instEligible, capOverflow }`

---

## CHAS
- Tiers: none, orange, blue, mg (Merdeka Generation), pg (Pioneer Generation)
- `chasMode` global string
- `getChasSubsidy(p)` returns subsidy per unit for current tier
- Subsidy capped to fee per unit; only subsidised units get subsidy
- CHAS tier stored on each NS summary snapshot as `_chasTier`

---

## Firebase / Signature
- `generateSigningQR()` — creates session in Firebase, shows QR, polls every 2s
- Patient scans QR on phone → signature canvas → `PUT` to Firebase
- Desktop polls, receives signature image, places it in `#sig-image-area`
- `placeSignature(dataUrl)` / `clearSignature()` manage the signature display

---

## Aesthetic (v4)
- 4px navy brand bar (`#0C447C`) at top
- Clinic name as large heading ("Nofrills Dental")
- Summary table: Treatment column 42%, `tx-name`/`tx-sub` row structure
- Metric cards: Total Fee (left) | Patient Cash Payable hero card (centre) | Subsidies Applied (right)
- Signature block: patient sig fixed 220px left, date fixed 140px right

## CSS variables (root)
```css
--bg-page:#f5f5f2; --bg-card:#fff; --bg-surface:#f0ede6;
--bg-info:#e6f1fb; --bg-success:#eaf3de; --bg-purple:#eeedfe;
--text-pri:#1a1a18; --text-sec:#5f5e5a; --text-ter:#8a8880;
--text-info:#185fa5; --text-success:#2f6010; --text-cash:#b06000;
--text-mv:#1479b8; --text-purple:#534ab7;
--border-lt:rgba(0,0,0,.11); --border-md:rgba(0,0,0,.24);
--r-sm:6px; --r-md:8px; --r-lg:12px;
```

---

## Naming conventions
- Section keys: `surg`, `ns`, `ortho`, `prostho`
- Timeline prefix: `tl` (all timeline vars/functions)
- Print-only elements: class `no-print` hides on screen; explicit print CSS shows what's needed
- GST always calculated live, never stored

---

## Known limitations / future work
- No persistence — refreshing the tab loses all quote data (Firebase is only used for signatures)
- No input validation on fees or quantities
- Drag-and-drop uses mouse events only — touch/iPad not supported
- No undo for removing treatments from summary
- PDF output is browser-dependent (`window.print()`)
- Generic treatments list will need grouping as it grows

---

## How to continue
Upload this document and `index.html` together. Reference this document for context on data structures, naming, and design decisions before making any changes.
