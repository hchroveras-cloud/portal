# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Bacchus Beverages — Producer Portal** (`https://portal-bacchus.web.app`)

A single-file HTML producer onboarding portal. Producers log in, fill out sections (company info, products, tech sheets, regulatory, pricing, sustainability, brand, agreements, documents), and submit for admin review. Admins approve, flag, or delete producers.

**Everything lives in `index.html`** — all CSS, HTML structure, and JavaScript in one file. No build step, no bundler, no framework.

## Deploy

```bash
firebase deploy          # push live to https://portal-bacchus.web.app
git add index.html && git commit -m "..." && git push   # save to GitHub
```

## Firebase Backend

- **Project:** `portal-bacchus`
- **Auth:** Email/Password. Admin role determined by `ADMIN_EMAILS` array (line ~645). To add a new admin, add their email to that array and create their account in Firebase Console → Authentication.
- **Firestore:** `producers/{uid}` stores profile + `completedSecs[]`. Sub-collection `producers/{uid}/sections/{sectionId}` stores form field data per section. Sub-collection `producers/{uid}/documents/` stores file metadata.
- **Storage:** Files at `producers/{uid}/doc-upload/…` (documents) and `producers/{uid}/brand-upload/…` (brand assets).

## Architecture

### Screens
Four screens toggled via `showScreen(n)` which adds class `active` to `#s-{n}`:
- `s-login` — login page
- `s-portal` — producer portal (sidebar + section panes)
- `s-admin` — admin dashboard (producer table)
- `s-admin-detail` — admin detail view for a single producer

### Script Initialisation Order
1. Firebase init runs immediately (outside DOMContentLoaded) — `fbAuth`, `fbDb`, `fbStorage` are global.
2. `ADMIN_EMAILS`, `currentUser`, `getRoleForEmail()` are global.
3. Everything else runs inside `DOMContentLoaded`.
4. `fbAuth.onAuthStateChanged` restores session on page reload → calls `handleAuthSuccess(user)`.

### Section System (`SECS`)
`SECS` is an array of `{id, label, req}` objects that drives everything:
- Sidebar nav buttons: `#n-{id}`
- Section panes: `#sec-{id}.spane`
- Save buttons: `#save-{id}`
- Progress tracking via `completedSecs` object

Pre-computed constants derived from `SECS`:
- `SEC_IDS` — array of all section id strings
- `SEC_REQ` — array of required section id strings (used for progress + submit unlock)

**To add a new section:** add entry to `SECS`, add `<button class="ni" id="n-{id}">` to sidebar HTML, and add `<div id="sec-{id}" class="spane">` to portal main.

### Producer Data Flow
- `producers[]` — in-memory array. Seeded from hardcoded demo data, overridden by `loadProducersFromFirestore()` which merges Firestore producers + any demo producers not already in Firestore (matched by email).
- `saveState()` — writes `producers[]` to localStorage as a local cache backup.
- `saveSec(id)` — marks section complete, calls `collectSection(id)` to serialise all form inputs, writes to Firestore.

### Key Helpers
| Function | Purpose |
|---|---|
| `findProducer(id)` | Lookup producer by id from `producers[]` |
| `showModal(id)` | Add `show` class to overlay element |
| `showToast(msg, type)` | Toast notification (`"ok"`, `"err"`, `"info"`) |
| `showScreen(n)` | Switch active screen |
| `showSec(id)` | Switch active section pane in producer portal |
| `collectSection(secId)` | Serialise all inputs in a section to a plain object |
| `updateProducerStatus(id, status, msg, toastType)` | Update producer status, re-render table |
| `makeRow(title, fields, cols)` | Create a dynamic form row (addresses, contacts, banks) |
| `addFileToList(list, ...)` | Render an uploaded file item in the UI |

### Dynamic Product Cards
`addProduct()` and `addTech()` build product/tech-sheet cards by string-concatenating HTML. Each card has an inline Product Type `<select class="prod-type-sel">` that on `change`:
- Repopulates the Category `<select class="prod-cat-sel">` with type-specific options
- Shows/hides ABV fields (`display:contents`) and food fields

### Admin View
- `renderTable()` — rebuilds the full producer list from `producers[]`
- `loadProducersFromFirestore()` — fetches Firestore, merges with demo data, then calls `renderTable()`
- `openDetail(id)` / `showDS()` — renders a producer's full detail view using renderer functions (`rCompany`, `rProducts`, `rTechnical`, etc.)
- Edit mode toggled via `editMode` boolean; `toggleEdit()` / `saveEdit()` handle inline editing

## CSS Layout
- **Fonts:** `Cormorant Garamond` (headings/brand), `Outfit` (body)
- **CSS variables:** defined in `:root` — use `--navy`, `--gold`, `--text`, `--bord`, `--bg`, `--sh` etc.
- **Grid helpers:** `.fg` (2-col), `.fg3` (3-col), `.fg4` (4-col), `.fgroup` (label+input stack)
- **Responsive breakpoints:** `@media(max-width:900px)` collapses sidebar to icon strip; `@media(max-width:600px)` hides sidebar and enables hamburger overlay (`#mob-menu-btn` / `#portal-sidebar` / `#mob-overlay`)

## Product Categories by Type
- **With ABV:** Still Wine, Sparkling Wine, Spirit, Liqueur, Aperitif, Beer, Cider, Vermouth
- **No ABV:** Soft Drink/Soda, Tonic/Mixer, Juice, Water, Kombucha
- **Food:** Pizza & Bakery, Tomatoes & Sauces, Pasta Rice & Grains, Oils Vinegars & Condiments, Snacks & Antipasti, Cheese & Dairy, Meat Fish & Protein, Frozen & Dessert, Ready Meals & Meal Solutions, Dry Goods & Pantry, Beverage Ingredients, Non-Food/POS/Equipment
