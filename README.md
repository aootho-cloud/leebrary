# Product Requirements Document: Leebrary

**Product name:** Leebrary
**Tagline:** "a library that remembers"
**Type:** Single-file, client-only mobile web app (installable as a home-screen PWA)
**Version:** 2.2 — supersedes v2.1. Adds a hierarchical **"Date added" filter** (Year → Month → Day) to the Filters sheet, letting the user narrow the Library to books added in a specific year, a specific month within that year, or a specific day within that month. Removes nothing from v2.1's feature set.
**Audience for this document:** an engineer who has never seen the app, building it from scratch. Following this document exactly, with no deviations or personal interpretation, should produce a functionally and visually identical app.

---

## 1. Product Overview

Leebrary is a personal reading tracker. One person logs the books they own, are reading, or have finished; rates and reviews them; tracks whether a copy is owned or borrowed; tracks rereads with full history; keeps a "want to buy" list; tracks which of their own books they've lent to other people; and views their reading progress by month, year, custom range, or all-time. It is built for **personal, single-user use** — there is no login, no multi-user support, and no server or database. All data is stored locally in the browser via `localStorage`, split across three keys (books, buy-list, loans).

The app must be deliverable as **one self-contained `.html` file** with no build step, no bundler, and no external JS framework. It must run by simply opening the file (or a URL pointing to it) in a mobile browser.

### 1.1 Non-goals (explicitly out of scope)
- No user accounts, login, or authentication of any kind.
- No server, API, or database. No network calls except loading two Google Fonts.
- No multi-device sync. Data lives only in the browser that created it.
- No offline service worker / caching strategy beyond what the browser does by default.
- No support for multiple simultaneous users of the same installed instance.
- No barcode/ISBN scanning or online metadata lookup.

---

## 2. Platform & Technical Constraints

- **Single HTML file.** All CSS lives in one `<style>` block in `<head>`. All JavaScript lives in one `<script>` block at the end of `<body>`, wrapped in an immediately-invoked function expression (IIFE) so nothing leaks to the global scope.
- **No frameworks, no build tools.** Plain HTML/CSS/JavaScript (ES2017+ features are fine: `async/await`, template literals, arrow functions, `Set`, optional chaining not required).
- **Rendering approach:** the app is a single-page app with no router. A `<main id="main">` element's `innerHTML` is fully re-rendered on every state change from one of five top-level "views": `library`, `progress`, `wishlist`, `lent`, `detail`. There is no virtual DOM — re-render means rebuilding an HTML string and setting `.innerHTML`, then re-attaching event listeners.
- **Persistence:** four independent `localStorage` keys, saved after every mutation (no debounce, no batching):
  - `leebrary_books_v1` — the book library, a `JSON.stringify`'d array (schema in section 4.1).
  - `leebrary_wishlist_v1` — the buy list, a `JSON.stringify`'d array (schema in section 4.2).
  - `leebrary_loans_v1` — the lending log, a `JSON.stringify`'d array (schema in section 4.3).
  - `leebrary_last_backup_v1` — **not** a JSON array; a single raw ISO 8601 timestamp string (`new Date().toISOString()`), written only when "Export backup file" succeeds. Absent until the very first successful export. Drives the backup-freshness indicator (section 15.1).
  - On any parse failure for one of the three array-shaped keys, fall back to an empty array for that key only. A missing/unparseable `leebrary_last_backup_v1` is simply treated as "never backed up" — no error handling needed since it's read with a plain `localStorage.getItem`, not `JSON.parse`.
- **Human-readable date formatting:** every stored date-only string (`"YYYY-MM-DD"`) shown to the user — as opposed to used internally for sorting, grouping, or `<input type="date">` values — must be rendered through a single shared `fmtDate(iso)` helper as **`DD-MM-YYYY`**, zero-padded (e.g. `05-09-2026`), computed from the date's local calendar fields (`getDate()`/`getMonth()`/`getFullYear()`), not `toLocaleDateString`. This is the one and only date-display format anywhere in the app: Library/Detail dates, Lent "Lent [date]"/"Returned [date]" lines, the Settings "Last backup" line, and Custom-range progress labels all go through this same helper. This is distinct from month-only labels (e.g. Progress period headers reading "September 2026", or the month `<select>`), which keep their full month-name + year format and are unaffected by this rule.
- **Fonts:** Google Fonts, loaded via a single `<link>` tag:
  - `Source Serif 4` — weights available: 400, 600, 700, with optical-size axis `opsz` range `8..60`. Used for all headings, titles, big numbers, and anywhere a "book-ish" serif voice is wanted.
  - `Inter` — weights 400, 500, 600, 700. Used for all UI/body text.
  - Exact `<link>` href: `https://fonts.googleapis.com/css2?family=Source+Serif+4:opsz,wght@8..60,400;8..60,600;8..60,700&family=Inter:wght@400;500;600;700&display=swap`, preceded by `<link rel="preconnect" href="https://fonts.googleapis.com">`.
- **Viewport lock (critical layout requirement):** the app must fill exactly the visible browser viewport with **no page-level scrolling**. Only the app's own content region scrolls internally. Implementation:
  - `html, body { height: 100%; }`
  - `body { margin:0; height:100dvh; overflow:hidden; display:flex; justify-content:center; }` — `100dvh` (dynamic viewport height) is declared after `height:100%` so unsupported browsers keep the `100%` fallback and supporting browsers get the more accurate dynamic value that correctly handles mobile browser chrome show/hide.
  - `#app { width:100%; max-width:430px; height:100%; overflow:hidden; position:relative; display:flex; flex-direction:column; }` — this is the phone-shaped frame, centered on wider screens via the parent's `justify-content:center`, with a soft outer shadow (`box-shadow: 0 0 40px rgba(0,0,0,0.12)`) so it visually reads as a floating "device" on desktop widths.
  - `main { flex:1; overflow-y:auto; }` — this is the ONLY element that scrolls. Header, bottom tab bar, and the floating action button are all outside or fixed relative to this, so they never move.
- **Mobile home-screen installability:**
  - Static meta tags in `<head>`: `apple-mobile-web-app-capable=yes`, `apple-mobile-web-app-status-bar-style=default`, `apple-mobile-web-app-title=Leebrary`, `mobile-web-app-capable=yes`, `theme-color=#5B3A5C`.
  - A static `<link rel="apple-touch-icon">` and `<link rel="icon">`, both pointing to the **same baked-in base64 PNG data URI** (see section 3.4 for the icon's visual spec) — baked in statically (not generated at runtime) so it's available the instant the page loads, with no dependency on JavaScript execution timing.
  - At runtime (on load), the app also dynamically constructs a Web App Manifest object (`name`/`short_name: "Leebrary"`, `start_url: "."`, `display: "standalone"`, `background_color` = the `--paper` token, `theme_color` = the `--spine` token, `icons` array referencing the same baked-in icon at 192×192 and 512×512), serializes it, wraps it in a `Blob` of type `application/manifest+json`, and appends a `<link rel="manifest">` pointing at `URL.createObjectURL(blob)`. This is best-effort progressive enhancement for Android's "Install app" flow; wrap the whole thing in try/catch and fail silently since it's non-critical.
- **No native `confirm()`/`alert()`/`prompt()`.** These are unreliable or fully blocked in some hosting/sandbox contexts. The app must implement its own in-app modal for both "yes/no" confirmations and single-button "OK" notices (spec in section 15). Every place that would naturally want a browser confirm dialog (delete a book, confirm an import, report a failed image load, remove a wishlist or loan entry) must use this custom modal instead.

---

## 3. Visual Design System

### 3.1 Color tokens
Define these as CSS custom properties on `:root`. Every color used anywhere in the app must trace back to one of these — no other hard-coded colors except pure white (`#fff`) for text-on-solid-color.

| Token | Hex | Used for |
|---|---|---|
| `--paper` | `#F1E7E9` | App background |
| `--paper-deep` | `#E3D2D6` | Page background behind the app frame; inactive segmented-control track |
| `--card` | `#FBF6F5` | Card / input / button surfaces |
| `--ink` | `#2B2230` | Primary text |
| `--ink-soft` | `#7C6B75` | Secondary/muted text |
| `--spine` | `#5B3A5C` | Primary brand color — active states, primary buttons, cover placeholder gradient start, wordmark "Lee" |
| `--spine-deep` | `#3B2640` | Darker brand shade — big numbers, cover placeholder gradient end |
| `--brass` | `#B8863A` | Gold accent — star ratings only (rating picker, filled stars, canvas star drawing), and the "on" state of the borrowed-copy toggle switch in the Add/Edit sheet |
| `--brass-bg` | `#F1E2C4` | Unused by any tag/pill (reserved token; kept for the toggle switch's related styling) |
| `--rust` | `#A6483B` | Destructive actions (delete), filter-count badge, overdue-loan accent |
| `--line` | `#E0CCD1` | Borders, dividers, inactive/empty elements, unfilled stars |
| `--shadow` | `0 2px 10px rgba(43,34,48,0.08)` | Standard soft card/element shadow |
| `--tag-red` | `#B14A3D` | "TBR" status tag (text) |
| `--tag-red-bg` | `#F5DBD7` | "TBR" status tag (background); also the Overdue badge/pill background |
| `--tag-yellow` | `#8A6A1D` | "Reading" status tag (text) |
| `--tag-yellow-bg` | `#F8ECC6` | "Reading" status tag (background) |
| `--tag-green` | `#3F7A4E` | "Done" status tag (text) |
| `--tag-green-bg` | `#DCEBDD` | "Done" status tag (background) |
| `--tag-blue` | `#3A5F8C` | "Owned" ownership tag (text) |
| `--tag-blue-bg` | `#DCE6F2` | "Owned" ownership tag (background) |
| `--tag-violet` | `#6F4FA0` | "Borrowed" ownership tag (text) |
| `--tag-violet-bg` | `#EBE1F7` | "Borrowed" ownership tag (background) |

**Semantic tag-color rule (applies everywhere a status/ownership tag appears — list pills, grid dots, the detail page, the shareable book-card image, and the ownership toggle button):** status maps `to-read → red`, `reading → yellow`, `done → green`; ownership maps `owned → blue`, `borrowed → violet`. There is no other color mapping for these two dimensions anywhere in the app. These are the internal status *values* — the on-screen **labels** for them are **"TBR"**, **"Reading"**, and **"Done"** respectively (see the status-label rename in the version note above); only the label text changed, not the value, the color, or the CSS class name (`status-to-read`, `status-reading`, `status-done` are unchanged). The ownership toggle button on the detail page (`.tag-owner-btn`) reflects the *current* state via color — plain style (`--tag-blue-bg` / `--tag-blue`) when the book is currently owned (button offers "Mark borrowed"), solid `--tag-violet` fill with white text when currently borrowed (button offers "Mark owned").

### 3.2 Typography
- Headings, titles, brand name, big stat numbers, sheet `<h2>` titles: `'Source Serif 4', serif`, weight 600–700.
- All body/UI text, buttons, labels, inputs: `'Inter', system-ui, sans-serif`.
- The brand wordmark "Leebrary" is rendered as `<span class="script">Lee</span>brary` where **both parts use the identical font/weight/size** (no different typeface) — the only difference is that `.script` is colored `--spine` while the rest of the word inherits `--ink`.

### 3.3 Core components (describe visual spec precisely; exact CSS should mirror this)
- **Segmented control** (`.segmented`): a full-width, rounded-11px track (`--paper-deep` background, 3px padding), containing flex buttons with no visible border; the active button gets a white-ish (`--card`) pill background and the standard shadow token, inactive buttons are transparent with `--ink-soft` text. A `.scroll` modifier variant allows horizontal scrolling with non-stretching buttons, used for the rating-filter row.
- **Pills** (`.pill`): small fully-rounded (`border-radius:100px`) tags, 11px bold text, a small 5×5px solid dot before the label, used for status (TBR/Reading/Done) and ownership (owned/borrowed), colored per the semantic rule in 3.1. An additional `.overdue-pill` variant (`--tag-red-bg` background, `--tag-red` text, no leading dot) reads "Overdue" and appears only on active, overdue loan cards.
- **Star rating display**: five `★` glyphs; filled stars colored `--brass`, unfilled colored `--line`. Two sizes: normal (14px, used on the detail page) and `.small` (12px, used on library tiles and grid tiles).
- **Buttons** (`.btn`): rounded-9px, 1px `--line` border, `--card` background, `--ink` text, 12.5px bold. Modifiers: `.primary` (solid `--spine` bg, white text), `.tag-owner-btn` (see 3.1 for its two color states), `.danger-outline` (transparent bg, `--rust` border+text — used for Delete/Remove actions), `.danger-solid` (solid `--rust` bg, white text — used for the destructive confirm button in the custom confirm modal).
- **Cards** (`.book-card`, `.wish-card`, `.loan-card`): `--card` background, 1px `--line` border, 14px radius, standard shadow, 14–15px padding. `.book-card` is fully tappable (`cursor:pointer`, subtle `scale(0.985)` active-press feedback, visible focus ring `:focus-visible { outline: 2px solid var(--spine) }`, `tabindex="0"` and `role="button"`, responds to `Enter`/`Space`). `.loan-card.returned` drops to `opacity:0.55` with no shadow (a visually "disabled" look for completed loans); `.loan-card.overdue` gets a `--tag-red` border instead of `--line`.
- **Grid tiles** (`.grid-tile`, used only in the Library's grid view): a 3-column CSS grid (`repeat(3, 1fr)`, `gap:14px 10px`). Each tile is a cover image (or the gradient+📕 placeholder) at a `3/4.2` aspect ratio, rounded 8px, with a small colored ownership dot overlaid on the cover's top-right corner (10px, 2px `--card` border, colored per 3.1's blue/violet rule) and, below the image: the title (12px serif, 2-line clamp), a centered status dot (7px, colored per 3.1's red/yellow/green rule), and a small star row if rated. The whole tile is tappable exactly like a list card (same click/keyboard handling).
- **Bottom sheets / modals** (`.sheet-backdrop` + `.sheet`): full-screen semi-transparent scrim (`rgba(43,34,48,0.4)`) that is `display:none` by default and `display:flex; align-items:flex-end` when given an `.open` class; the sheet itself slides up from the bottom (`translateY(30px)→0` with fade, 0.22s ease), rounded 20px top corners only, `max-height:88vh` with internal scroll, `position:relative` (so a close-icon button can be absolutely positioned inside it). Clicking the backdrop itself (not the sheet) closes it. A reusable `.sheet-close-btn` — a small 28px circular ✕ button, `--paper-deep` background, positioned `top:14px; right:14px` — is used by sheets that want an icon-close instead of/alongside a text "Cancel"/"Close" button (currently only the Surprise Me popup).
- **Floating Action Button** (`.fab`): 52×52px, 16px radius, solid `--spine` background, white "+", fixed to the bottom-right of the app frame (`position:absolute; right:20px; bottom:92px` — sitting just above the tab bar), with a colored drop shadow tinted `--spine`. Hidden (via JS `style.display`) whenever the current view is `detail`. Its action and tooltip (`title` attribute) depend on the active tab: **Library or Progress** → opens the Add/Edit book sheet, tooltip "Add a book"; **List** → opens the "Add to your list" sheet, tooltip "Add to your list"; **Lent** → opens the "Lend a book" sheet, tooltip "Lend a book".
- **Bottom tab bar** (`nav.tabbar`): fixed to the bottom of the app frame, `--card` background, top border `--line`, **four** flex buttons, each `position:relative` (emoji above label, stacked vertically), active tab text colored `--spine`, in this exact order: **📖 Library, 📈 Progress, 🛒 List, 🤝 Lent**. The Lent button additionally contains a small notification dot (`.tab-dot`, 8px, `--tag-red`, `--card` border, `display:none` unless given the `.show` class) positioned near its top-right — see section 16.4 for when it appears.

### 3.4 App icon (home-screen logo)
A square image (generate at 192×192 and 512×512), **not pre-rounded** (let the OS apply its own mask):
1. Fill the entire square with a linear gradient from `--spine` (top-left) to `--spine-deep` (bottom-right).
2. Draw a solid horizontal bar in `--brass` across the full width at the very bottom, with height equal to 4.5% of the icon size (a "bookmark ribbon" accent).
3. Centered in the square, draw a bold serif capital letter **"L"** (font: `700 {58% of icon size}px serif`), fill color `--paper` (the light paper tone, not pure white), vertically positioned so its optical center sits slightly above true center (baseline `middle`, drawn at `y = size/2 - size*0.04` roughly, i.e. nudged up to account for the letter's descender-free shape).

This same image is what both the static `<link rel="apple-touch-icon">`/`<link rel="icon">` and the dynamically-built manifest's `icons` array must reference.

---

## 4. Data Model

### 4.1 Book
A single book is a plain JS object with this exact shape. Every field except `id`, `title`, `author`, `status`, `dateAdded` is nullable/optional and must default sensibly.

```
{
  id: string,            // e.g. "b_173703421123_f8x2q1" — see ID scheme below
  title: string,         // required, non-empty
  author: string,        // required, non-empty
  status: "to-read" | "reading" | "done",
  borrowed: boolean,     // true = borrowed copy, false = owned (default false)
  cover: string | null,  // a base64 JPEG data URI, or null if no cover image
  rating: number,        // 0-5 integer; 0 = unrated
  review: string,        // free text, "" if none
  dateAdded: "YYYY-MM-DD",         // date-only ISO string, set once at creation, never changes
  dateFinished: "YYYY-MM-DD" | null, // date-only ISO string of the CURRENT/most recent read's completion, or null if not currently marked done
  dateFinishedApprox: boolean,     // true only if dateFinished was captured via the "not sure of the exact date" year picker (add mode only — see 9.4); default false
  dateFinishedApproxYear: number | null, // the year chosen in that picker, or null when dateFinishedApprox is false; used ONLY for display (section 10), never for sorting/grouping — dateFinished itself (a real "YYYY-07-02" placeholder date, see 9.4) is what sorting/grouping/progress stats use
  readHistory: string[]  // array of "YYYY-MM-DD" strings, one per completed read, in chronological order of when they were recorded (append-only; see section 9.3 for exact rules on when entries are added vs. corrected)
}
```

**ID scheme:** `uid()` returns `'b_' + Date.now() + '_' + Math.random().toString(36).slice(2,8)` — i.e. `b_<millisecond-timestamp>_<6-char-random-base36>`. This same `uid()` is reused for wishlist and loan record ids too. Beyond uniqueness, the millisecond timestamp embedded in a book's id is reused as a **sort tiebreaker** (section 7.2) whenever two books share the same date-only value, since a date-only field can't distinguish which of several same-day entries came first.

**`readHistory` vs. `dateFinished` back-compatibility:** a helper `getReadDates(book)` must be used everywhere read-completion dates are needed for aggregate purposes (progress stats, sort-by-done-date, month-grouping): if `readHistory` has entries, return it; otherwise, if `dateFinished` is set, return a single-element array `[dateFinished]`; otherwise return `[]`. This lets the rest of the app treat every book uniformly even though `readHistory` was added after `dateFinished` conceptually.

### 4.2 Wishlist item (buy list)
```
{
  id: string,
  title: string,        // required, non-empty
  author: string,       // required, non-empty
  publisher: string,    // optional, "" if not given
  dateAdded: "YYYY-MM-DD"
}
```
Stored under `leebrary_wishlist_v1` as a flat array, independent of the books array.

### 4.3 Loan record (lending log)
```
{
  id: string,
  bookId: string,              // the id of the book in the main books array at the time it was lent
  bookTitle: string,           // snapshot of the book's title at lend time
  bookAuthor: string,          // snapshot of the book's author at lend time
  personName: string,          // who it was lent to; required, non-empty
  dateLent: "YYYY-MM-DD",
  returned: boolean,           // false = still out, true = returned
  dateReturned: "YYYY-MM-DD" | null
}
```
Stored under `leebrary_loans_v1` as a flat array. The `bookTitle`/`bookAuthor` snapshot means a loan record remains fully readable even if the underlying book is later deleted from the library — the Lent tab never needs to look the book up to render a card, though the book's own detail page does look loans up by `bookId` to display them (section 12).

---

## 5. Information Architecture

Bottom tab bar has exactly four destinations, in order: **Library, Progress, List, Lent**. A fifth internal "view" — **Book Detail** — is reached only by tapping a book tile from the Library list, or via the Surprise Me popup's "Start reading" action which does NOT navigate to it (see section 17) — in practice the only navigational entry point is tapping a Library tile. Detail has its own dedicated back-navigation; the tab bar's active-tab highlighting still reflects whichever tab you came from.

Modal "sheets" (all using the same bottom-sheet visual pattern from section 3.3, each independent and never nested inside another):
1. **Add / Edit book** — opened by the FAB on Library/Progress (add mode), or the Detail page's "Edit" button (edit mode, pre-filled).
2. **Filters** — opened by the funnel icon on the Library page.
3. **Confirm/Alert** — a generic reusable modal, opened programmatically wherever the app needs a yes/no confirmation or a single-button notice.
4. **Share** — opened after generating a shareable image, shows a preview and a download link.
5. **Backup & restore (Settings)** — opened by the gear icon in the header.
6. **Add to your list** — opened by the FAB while on the List tab.
7. **Lend a book** — opened by the FAB while on the Lent tab.
8. **Surprise Me popup** — opened by the "🎲 Choose a book to read" button on the Library tab. Visually a sheet, but functionally distinct: it never sets `view`/`selectedBookId`, so the Library page underneath is untouched and still there the moment it closes.

---

## 6. Header

Fixed at the top of the app frame, not part of the scrolling content. Contains, left-to-right:
- The wordmark (section 3.2) as a heading, with a tagline directly beneath it in small muted text: **"a library that remembers"**.
- A circular-cornered icon button on the far right (gear/settings icon, using a standard 24×24 viewBox "settings" SVG glyph — two concentric shapes: a small circle plus the classic 8-notch gear outline, stroked not filled, `currentColor`), which opens the Backup & Restore sheet. This button also carries a small red notification dot (`.header-dot`, same visual language and positioning convention as the Lent tab's `.tab-dot`, but anchored to the icon button's own top-right corner rather than a tab-bar item) whenever the backup is stale or has never happened — see section 15.1 for the exact rule and how it's kept in sync.

A 1px bottom border (`--line`) separates the header from the content area.

---

## 7. Library Tab (default view on load)

Rendered top-to-bottom inside `<main>`:

### 7.1 Quick filter row
A `.quick-filter-row` containing:
- A 3-option segmented control: **All / TBR / Done**. This controls only the `status` dimension and is a convenience shortcut for the two most common single-status views (it intentionally does NOT include "Reading" as a quick option — that's only reachable via the Filters sheet).
  - Clicking **All** clears the status filter set entirely.
  - Clicking **TBR** sets the status filter to exactly `{"to-read"}`.
  - Clicking **Done** sets the status filter to exactly `{"done"}`.
  - The segmented control's active button reflects the CURRENT filter state: "All" is active only when the status-filter set is empty; "TBR"/"Done" are active only when the set is exactly that single value. If the set holds any other combination (e.g. "Reading" alone, or multiple statuses together, set via the Filters sheet), none of the three quick buttons show as active — this is expected and correct.
- A funnel-icon button (40×40 rounded-square) that opens the Filters sheet. If any filter is active that ISN'T representable by the quick row, show a small red numeric badge in the button's top-right corner. The badge count = (number of selected ownership filters) + (number of selected rating filters) + (1 if "Reading" is among the selected statuses, else 0). This is an approximate "how much extra filtering beyond the quick row is active" indicator, not a strict total.
- A view-mode toggle button (same 40×40 style), showing a grid glyph when currently in list view (tapping switches to grid) or a list glyph when currently in grid view (tapping switches to list). State: `libraryViewMode`, `"list" | "grid"`, default `"list"`.

### 7.2 Sort row
A `.sort-row` with a label on the left reading **"Sort by added date"** or **"Sort by done date"**, and a pill-shaped toggle button on the right reading **"↓ Recent first"** or **"↑ Oldest first"**.

**Which field is used is fully automatic, never user-chosen directly:**
- If the current status-filter set is EXACTLY `{"done"}` (i.e., the user is viewing only Done books — whether via the quick "Done" button or via the Filters sheet), sort by **done date** (the most recent entry in `getReadDates(book)`, or empty string if none).
- In every other case (All, TBR, Reading, or any other combination), sort by **added date** (`dateAdded`).

Clicking the direction toggle flips between `desc` (recent-first) and `asc` (oldest-first) and re-renders; it does not affect which field is used.

**Sort algorithm, exact:** compare the two books' string sort-keys (either both `dateAdded` values or both "most recent done date" values, per the rule above) with `localeCompare`, in the direction implied by `sortDir`. **Critically, if the two keys are equal** (e.g. two books both added "today", since the field only has day-level granularity) **fall back to comparing the millisecond timestamp embedded in each book's `id`** (section 4.1), in the same direction as the primary sort. Without this tiebreaker, same-day entries would silently fail to respond to the direction toggle at all (this was a real, user-reported bug during development — do not omit this fallback).

### 7.3 Surprise me button
A full-width button directly below the sort row: **"🎲 Choose a book to read"**. See section 17 for its behavior (the button's label is the only user-facing wording change from the original "Surprise me from your to-read pile" copy — its pool and randomization logic are unchanged: it still only draws from TBR-status books).

### 7.4 Search box
A single text input, placeholder **"Search title or author"**, filtering case-insensitively against the concatenation of `title + ' ' + author`. Typing must not lose focus or cursor position on re-render (re-focus the input and restore cursor to the end after each keystroke's re-render).

### 7.5 Month grouping (applies identically to list and grid view)
After filtering and sorting, books are further grouped into month sections before rendering:
- **A book with status `"done"`** groups under the calendar month of its most recent completion — `getReadDates(book)`'s last entry, sliced to `"YYYY-MM"`.
- **A book with status `"to-read"` or `"reading"`** always groups under the **current real-world month** (today's `"YYYY-MM"`), regardless of when it was actually added to the app. (This is a deliberate, explicitly-requested rule: an in-progress or not-yet-started book should surface under "what I'm doing right now," not get buried under whatever month it happened to be logged.)

Each month group renders a header — the month's full name and year (e.g. "September 2026") in bold serif on the left, and a muted "N book(s)" count on the right — followed by that group's books (in list-card or grid-tile form per the current view mode). Groups themselves are ordered by their month key using the same `sortDir` direction as the book-level sort (`desc` = most recent month first).

### 7.6 List view (`libraryViewMode === "list"`)
Each book renders as a full-width card (`.book-card`, section 3.3) containing:
- A 42×58px cover thumbnail (`object-fit:cover`, rounded 5px) if `cover` is set; otherwise the gradient+"📕" placeholder.
- Title (serif, 16.5px, weight 600) and, if present, author (13px, `--ink-soft`) stacked to the right of the thumbnail.
- Below title/author: a pill row with exactly two pills — the status pill and the ownership pill (**always show exactly one of "Owned"/"Borrowed", never both, never neither**), colored per section 3.1.
- If `rating > 0`, a small star row beneath the pills.

**Nothing else appears on the tile** — no action buttons, no menu, no dates. The entire card is the tap target; tapping it (or pressing Enter/Space while it's focused) navigates to the Book Detail view for that book.

### 7.7 Grid view (`libraryViewMode === "grid"`)
Each book renders as a `.grid-tile` (section 3.3): cover (or placeholder) with a small ownership dot on its corner, title beneath (2-line clamp), a status dot beneath that, and a small star row if rated. Same tap/keyboard behavior as a list card.

### 7.8 Empty states
- If there are zero books at all: glyph "📚", title **"Your shelf is empty"**, subtitle **"Tap the + button to add your first book."**
- If there are books but none match the current filters/search: glyph "📚", title **"No books match"**, subtitle **"Try different filters or search."**

### 7.9 Floating Action Button
Bottom-right "+" button, opens the Add/Edit sheet in "add" mode (no book passed in, all fields blank/default).

---

## 8. Filters Sheet

Opened via the funnel icon. Title: **"Filters"**. Three independently multi-selectable chip groups (tapping a chip toggles its membership in the corresponding `Set` state — multiple chips within a group can be active simultaneously, and combining criteria across groups is a logical AND), plus a fourth, hierarchical date-drill-down control:

1. **"Reading status"** — chips: TBR / Reading / Done.
2. **"Ownership"** — chips: Owned / Borrowed.
3. **"Rating"** — chips: ★1 / ★2 / ★3 / ★4 / ★5. Selecting a rating chip means "books rated exactly this many stars" (selecting several means "rated any of these values" — it is NOT a minimum threshold).
4. **"Date added"** — three side-by-side `<select>` dropdowns (Year / Month / Day), filtering on `dateAdded` and letting the user drill down from a whole year, to a specific month within that year, to a specific day within that month:
   - **Year** — defaults to **"Any year"**. Its options are every distinct year present across all books' `dateAdded` values, descending (most recent first), plus the current real-world year even if no book was added in it yet (so the option is never simply missing on a near-empty library). Selecting a year immediately filters the Library to books added in that year, with month/day left at "Any" — this alone satisfies "see the books added in a particular year."
   - **Month** — disabled and shows only **"Any month"** until a specific year is chosen; once a year is picked, it's enabled and populated with all 12 full month names. Selecting a month further narrows the filter to that year+month — "books from a month on a particular year."
   - **Day** — disabled and shows only **"Any day"** until a specific month is chosen; once a month is picked, it's enabled and populated `1..N`, where `N` is the actual number of days in the selected year+month (so February correctly offers 28 or 29 depending on the year, April offers 30, etc.) — never a flat, sometimes-wrong 31. Selecting a day narrows the filter to that exact year+month+day — "a particular day of month of year."
   - Choosing a coarser value resets everything finer beneath it back to "Any": picking a different year clears both month and day back to "Any"; picking a different month clears day back to "Any". This keeps the three selects always in a valid, non-contradictory state (there's no way to end up with a day selected but no month, for instance).
   - A book with no `dateAdded` at all can never match a non-"Any" year (there is no legacy data shape where this happens in practice, since `dateAdded` is required at creation, but the filter treats it defensively as "doesn't match" rather than throwing).

Below the four groups, two full-width action buttons:
- **"Clear all"** — empties all three chip filter sets, resets Year/Month/Day back to "Any"/disabled, and immediately updates the chip active-states, the date selects, and the live count (does not close the sheet).
- **"Show N book(s)"** (this button's own label is dynamic, always reflecting the live count of books that would match the CURRENTLY toggled — not yet applied — filter state, combined with the current search term) — tapping it closes the sheet and re-renders the Library list with the new filters applied.

Filtering logic used both for the live count in this sheet and for the actual Library list: a book matches if (status set is empty OR its status is in the set) AND (ownership set is empty OR its owned/borrowed state is in the set) AND (rating set is empty OR its rating is in the set) AND (Year is "Any" OR `dateAdded`'s year matches, AND — only if Year is set — Month is "Any" OR `dateAdded`'s month matches, AND — only if Month is set — Day is "Any" OR `dateAdded`'s day matches) AND (search term is empty OR title+author contains it, case-insensitive). The funnel icon's badge count (section 7.1) also increments by 1 whenever Year is set to something other than "Any", the same way it already does for "Reading" among the status chips — it's an approximate "extra filtering beyond the quick row" indicator, not a strict total, so Month/Day being set on top of an already-counted Year does not add further to the badge.

---

## 9. Add / Edit Book Sheet

Title: **"Add a book"** (add mode) or **"Edit book"** (edit mode, all fields pre-filled from the existing book).

Fields, in this exact order:
1. **Title*** — required text input, placeholder "The Left Hand of Darkness".
2. **Author*** — required text input, placeholder "Ursula K. Le Guin". (Only these two fields are marked with a red `*` and only these two block saving if empty — every other field below is optional with a sensible default.)
3. **Cover image (optional)** — a file input (`accept="image/*"`, hidden) triggered by a dashed-border "Choose a photo" label-button. On selection:
   - Read the file, load it into an `Image`, and **resize/compress it client-side before storing**: scale so the longer edge is at most **320px** (preserve aspect ratio), draw to an off-screen `<canvas>`, and export via `canvas.toDataURL('image/jpeg', 0.72)`. This keeps stored covers small since everything lives in `localStorage`.
   - Once set, show a 52×72px preview thumbnail plus a "Remove" button (which clears the cover and re-shows the "Choose a photo" control).
4. **Status** — a 3-button single-select group: TBR / Reading / Done (internal value `"done"`). Choosing "Done" reveals field 5.
5. **Finished-date block** (only visible when status = Done) — see section 9.4 for its full, order-sensitive spec. In short: a "Not sure of the exact date" toggle sits ABOVE the date field it controls, and — critically — this whole toggle is only ever offered in **add mode**; in **edit mode** it is hidden outright and the field always behaves as a plain date input, regardless of whether the book being edited currently has an approximate date on file.
6. **Borrowed copy** — a toggle switch (default off = owned).
7. **Your rating (optional)** — five tappable star buttons (unfilled `--line`, filled `--brass` up to the chosen value). Tapping the currently-selected star again clears the rating back to 0 (there is no separate "clear rating" control).
8. **Notes / review (optional)** — a multi-line textarea, placeholder "What stood out to you?".

Footer: **"Cancel"** (closes without saving) and **"Save book"** (primary).

### 9.1 Save validation
Block saving (and focus the offending field) if Title or Author is empty after trimming whitespace. Every other field may be left at its default.

### 9.2 Save logic — new book
Push a new book object with a fresh `uid()`, `dateAdded` = today, and: if status is "done", `dateFinished` = the resolved finish date (section 9.4 — either the exact date typed, or the `YYYY-07-02` placeholder derived from the chosen year), `dateFinishedApprox`/`dateFinishedApproxYear` set per 9.4, AND `readHistory` = `[thatDate]`; otherwise `dateFinished = null`, `dateFinishedApprox = false`, `dateFinishedApproxYear = null`, and `readHistory = []`.

### 9.3 Save logic — editing an existing book (exact rule, do not simplify)
- Update title/author/status/borrowed/cover/rating/review directly from the form.
- Because the approximate-date toggle is never offered in edit mode (field 5 above), the finish date resolved on save is always the plain date input's value — treat it exactly as an exact date, never as approximate, no matter what the book's stored `dateFinishedApprox` was before this edit.
- If the new status is **"done"**:
  - If the book's status was ALREADY "done" before this edit (i.e. you're editing an already-finished book, not freshly completing it) AND it already has at least one `readHistory` entry: **overwrite the LAST entry** in `readHistory` with the chosen finish date, and set `dateFinished` to that same date. (Rationale: the user is correcting the date of the read they're currently editing, not logging a brand-new read.)
  - Otherwise (the book is transitioning INTO "done" status via this edit, from "to-read" or "reading"): **append** the chosen finish date as a NEW entry to `readHistory`, and set `dateFinished` to it. (Rationale: this is a genuinely new completion event.)
  - Either way, set `dateFinishedApprox = false` and `dateFinishedApproxYear = null` — an edit always resolves to a concrete date, even if the book previously carried an approximate one from when it was first added.
- If the new status is NOT "done": set `dateFinished = null`, `dateFinishedApprox = false`, `dateFinishedApproxYear = null`. **Never delete or modify `readHistory` in this branch** — past completions must remain on permanent record even if the book's current status is reset back to "reading" or "to-read".
- After saving an edit, return the user to the Book Detail view (not the Library list) — editing is only ever reached FROM the detail page in this app (there is no edit entry point on the library tile itself), so it should feel like "the detail page updated," not "you navigated away."
- The "Mark done" quick action on the Detail page (section 10, item 8) is a separate code path from this sheet and always produces a fresh, exact completion — it also resets `dateFinishedApprox = false` / `dateFinishedApproxYear = null` for the same reason.

### 9.4 The "not sure of the exact date" option (add mode only)
This exists for logging backlogged books — ones already finished, sometimes years ago, where the user has no idea of the specific day.

- **Only shown when adding a new book**, never when editing (see field 5 above and section 9.3). The rationale: once a book is in the library, editing it is about correcting/updating a record you can already see in full, including whatever date is on file — there's no reason to re-offer "I don't know the date" for a book that already has *some* date, exact or not.
- Layout, top to bottom, inside the "Done" branch of field 5: a toggle switch row reading **"Not sure of the exact date"** first, then — depending on its state — either:
  - **Off (default):** a plain **"Date finished"** label + native date input, defaulting to today's date.
  - **On:** the date input (and its label) are hidden entirely and replaced by a **"Year finished"** `<select>`, populated with the current year down through the previous 60 years (current year first, descending), defaulting to the current year.
- **On save, if the toggle is on:** resolve the finish date as a placeholder string `` `${chosenYear}-07-02` `` (July 2nd — an arbitrary, roughly mid-year date chosen so the entry still sorts/groups into the correct calendar year everywhere: sort-by-done-date, month/year grouping on the Library tab, and the Progress tab's charts all key off this real date string exactly like any exact one). Set `dateFinishedApprox = true` and `dateFinishedApproxYear = chosenYear` on the book.
- **On save, if the toggle is off:** resolve the finish date from the date input as normal (fallback to today if somehow empty). Set `dateFinishedApprox = false` and `dateFinishedApproxYear = null`.
- **Display implication (see section 10):** wherever this specific finish date would otherwise be shown as "[date]", show **"sometime in [year]"** instead, whenever `dateFinishedApprox` is true AND the date being displayed is the book's current `dateFinished` value. This only ever applies to the single most-recent completion recorded on the book — a book's older `readHistory` entries are never shown as approximate even if the book's current entry is, since the approximate flag is a single per-book field, not per-history-entry.

---

## 10. Book Detail Page

Reached only by tapping a library tile (list or grid). Not a modal — it's a top-level "view" that fully replaces the Library/Progress/List/Lent content in `<main>`, with its own back-navigation.

Layout, top to bottom:

1. **"‹ Back to library"** text button — returns to whichever tab was active before entering detail.
2. **Prev/Next pager** (only rendered if the current filtered/sorted Library list is non-empty): "‹ Prev", a centered "N of M" count, "Next ›". This pager iterates over **the exact same filtered + sorted list currently shown on the Library page** (recomputed fresh each time detail renders, using the live filter/sort/search state — the month-grouping from section 7.5 is a display-only layer and does not affect this flat list's order). Buttons disable themselves at the start/end of the list. If the current book isn't found in that list at all, default the index to 0 rather than erroring.
3. **Hero row**: a large (104×148px) cover image or gradient-placeholder, next to the title (21px serif bold) and author (13.5px muted) stacked beside it.
4. **Pill row**: status pill + ownership pill (identical rules and colors to the library tile).
5. **Star row** (only if rated).
6. **Dates line** — normally reads **"Added [date] · Finished [date]"** (omitting the "· Finished" part if not currently done). **Special case:** if the book's `dateFinished` is earlier than its `dateAdded`, show **only** "Finished [date]", omitting the "Added" part entirely. **Approximate-date case:** wherever this "[date]" placeholder is the book's `dateFinished` value AND `dateFinishedApprox` is true, render it as **"sometime in [dateFinishedApproxYear]"** instead of a formatted date (section 9.4). `dateAdded` itself is never approximate — only the finished date can be.
7. **Reread note** (only shown when the book's CURRENT status is not "done" AND it has at least one prior read in `readHistory`): a small italic line, e.g. *"Read 2 times before · last finished [date]"* — the same approximate-date substitution from item 6 applies here too, when the "last finished" date being shown equals the book's (approximate) `dateFinished`.
8. **Status action row** — buttons depend on current status:
   - `to-read`: "Start reading" + "Mark done" (primary)
   - `reading`: "Mark done" (primary) only
   - `done`: "Reopen" only
   - Always also present: the ownership toggle button (`.tag-owner-btn`, section 3.1), reading "Mark borrowed" (if currently owned) or "Mark owned" (if currently borrowed).
   - **"Mark done" behavior:** set status to "done", set `dateFinished` to today's date, clear `dateFinishedApprox`/`dateFinishedApproxYear` back to `false`/`null` (a fresh completion is always exact, never approximate), and **append today's date as a new `readHistory` entry** — always a fresh completion event regardless of prior history.
   - **"Reopen" behavior:** set status back to "reading" and clear `dateFinished` to `null`. **Do not touch `readHistory`.**
   - **"Start reading"**: sets status to "reading", nothing else changes.
   - **Ownership toggle**: flips the boolean, nothing else changes.
9. **Review** (only if present): a labeled "Notes" section with the review text as a paragraph.
10. **Read history list** (only rendered if `readHistory.length > 1`): a small ordered list, one line per entry, formatted **"1st time — [date]"**, **"2nd time — [date]"**, etc. (proper English ordinals). The same approximate-date substitution applies per-entry: only the entry whose date equals the book's current `dateFinished`, on a book with `dateFinishedApprox` true, renders as "sometime in [year]" — every other entry in the list always renders as a normal formatted date, since the approximate flag is a single per-book field and cannot describe older, already-superseded entries.
11. **Lent out section** (only rendered if at least one loan record references this book's id): a labeled "Lent out" list, one line per matching loan record sorted most-recent-lent-first, each formatted either **"[person] — still out (lent [date])"** or **"[person] — returned [date]"**.
12. **Footer row** — three equal-width buttons: **"Share"**, **"Edit"**, **"Delete"** (styled `.danger-outline`).
    - Share → generates and offers the book's shareable image (section 13.1).
    - Edit → opens the Add/Edit sheet pre-filled with this book.
    - Delete → opens the custom confirm modal with message **`Remove "[title]" from your shelf?`**, confirm button labeled "Remove"; on confirm, actually remove the book from the array, persist, and navigate back to the Library tab. (Native `confirm()` was found to silently do nothing in some hosting contexts during development — the custom modal exists specifically to avoid that failure mode.)

---

## 11. Progress Tab

### 11.1 Period selector
A 4-option segmented control: **Month / Year / Custom / All time**.

### 11.2 Month view
- A month `<select>` dropdown (all 12 months, full names, e.g. "March"), defaulting to the current real-world month.
- Below it, the exact same year prev/next arrow control used by the Year view (see 11.3) — sharing state, so switching between Month and Year views keeps the same year in context.
- Shows the count of books completed in that specific month+year (using every entry from `getReadDates()` across all books, not just books whose CURRENT status is "done").
- No chart in this view.
- Below the stat, a list titled **"Finished in this period"**, one row per completed-read EVENT (not per book) showing title, author, and that specific date.

### 11.3 Year view
- A year switcher: "‹" button, the year number (serif, bold), "›" button, defaulting to the current real-world year.
- Total count for that year, plus a 12-bar chart (Jan–Dec), each bar's height proportional to that month's count within the year (zero-count bars render at a fixed minimum height in `--line`; non-zero bars show their count above them, in `--spine`).
- Same "Finished in this period" event list below, scoped to the year.

### 11.4 Custom view
- Two native date inputs, "From" and "To".
- Once both are set: total count, a chart broken down by month across the range (label each bar with the month abbreviation, plus the 2-digit year if the range spans more than one calendar year), and the same event list.
- Before both dates are set: show only the prompt "Choose a date range" as the period label, no stat/chart/list.

### 11.5 All time view
- Total count across every read event ever recorded, no date filter.
- A chart broken down by YEAR — one bar per calendar year that has at least one completion, spanning from the earliest to the latest year with data.
- Same event list, unscoped.

### 11.6 Shared elements across all four period views
- A stat hero: large serif number (52px) + a label below it reading **"[N] book(s) finished · [period label]"**.
- A **"Share progress"** button directly under the stat hero label, which generates the progress share image (section 13.2) for whatever is currently on screen.
- If the event list for the period is empty (and, for Custom view, only once both dates ARE set), show: glyph "🔖", title **"Nothing finished yet"**, subtitle **"Mark a book done to see it here."**

There is no "recap"/"year in review" feature — this was tried during development and deliberately removed. Do not add a wrap-up or summary card beyond the plain Share progress image described in 13.2.

---

## 12. List Tab (buy list)

A simple, standalone wishlist of books the user intends to buy — entirely separate from the main library and never automatically synced to it except via the explicit "Mark as bought" action.

- **Search box** at the top: placeholder "Search title, author, or publisher", filters case-insensitively against `title + ' ' + author + ' ' + publisher`. Same focus/cursor-preservation behavior as the Library search.
- **Count row** directly beneath the search box, shown only when the wishlist is non-empty: **"N book(s) on your list"**, where N is the TOTAL wishlist count regardless of the search term — when a search is active and narrows the visible set, append **" · M shown"** (M = the filtered count), so the user can always see both "how many total" and "how many currently visible" at once. Not shown at all when the wishlist is empty (the empty state below covers that).
- **Empty states:** if the wishlist is empty outright: glyph "🛒", title **"Your list is empty"**, subtitle **"Tap the + button to add a book you want to buy."** If items exist but none match the search: glyph "🛒", title **"No books match"**, subtitle **"Try a different search."**
- **List items**, sorted most-recently-added first, each a `.wish-card` showing: title (serif, 16.5px), author (muted, 13px) if present, publisher (muted, 11.5px, *italic*) if present, and three action buttons:
  - **"Mark as bought"** (primary) — pushes a brand-new book into the main library array: `title`/`author` copied from the wishlist entry, `status: "to-read"`, `borrowed: false`, `cover: null`, `rating: 0`, `review: ""`, `dateAdded` = today, `dateFinished: null`, `dateFinishedApprox: false`, `dateFinishedApproxYear: null`, `readHistory: []`. **The publisher field is discarded entirely** — it never travels to the book record. The wishlist entry is then removed from the wishlist array. Both arrays are persisted.
  - **"Edit"** — opens the same "Add to your list" sheet described below, but in edit mode: title becomes **"Edit list entry"**, the primary footer button's label changes from "Add to list" to **"Save changes"**, and all three fields are pre-filled from the entry being edited. Saving updates that entry's `title`/`author`/`publisher` in place (its `id` and `dateAdded` never change) rather than pushing a new one; the same required-field validation as add mode applies. This is the only way to correct a mistyped title/author/publisher on an existing list entry — there is no inline editing on the card itself.
  - **"Remove"** (`.danger-outline`) — opens the custom confirm modal, message `Remove "[title]" from your list?`, confirm label "Remove". On confirm, the wishlist entry is deleted outright. **This never touches the library** — a manually removed wishlist item is gone, full stop, and is never added to the book library under any circumstance.
- **Add to your list sheet** (opened by the FAB while on this tab, in add mode — the FAB always adds, never edits): fields **Title*** (required), **Author*** (required), **Publisher (optional)**. Same red-asterisk / trim-and-block-on-empty validation as the book sheet, but only for Title and Author. Footer: "Cancel" / "Add to list" (primary). On save, pushes `{ id: uid(), title, author, publisher, dateAdded: today }` onto the wishlist array. See "Edit" above for this same sheet's edit-mode behavior.

---

## 13. Lent Tab (lending tracker)

Tracks books the user has physically handed to other people — the mirror image of the "borrowed" ownership tag (which tracks copies the user borrowed FROM someone else and is expected to return promptly, and therefore needs no separate tracking UI). Only the user's own, currently-unlent copies can be logged here.

### 13.1 Eligible-books rule
A book can be selected for a new loan only if **both**: (a) `borrowed === false` (you can't lend out a copy that isn't yours), and (b) it has no existing loan record with `returned === false` (a book already out with one person can't simultaneously be logged as lent to someone else). This pool is recomputed fresh every time the Lend sheet opens.

### 13.2 Search + quick filter
A 3-option segmented control **All / Active / Returned** filters by loan status. A search box (placeholder "Search book or person") filters case-insensitively against `bookTitle + ' ' + bookAuthor + ' ' + personName`. Both apply together (logical AND) before the loans are split into the three display sections below.

Directly beneath the search box, shown only when at least one loan record exists (regardless of the current search/filter): a count row reading **"N book(s) currently lent out"**, where N = `loans.filter(l => !l.returned).length` computed over the **full, unfiltered** loans array — same "always the true total, independent of whatever search/quick-filter is currently applied" principle as section 13.7's tab-bar dot, and the same principle behind the List tab's count row (section 12).

### 13.3 Overdue rule
A loan is **overdue** if it is not yet returned AND at least **30 days** (`OVERDUE_DAYS = 30`, measured as whole calendar days between `dateLent` and today) have passed since it was lent. This is purely a display concept — it does not change any stored data, only how a loan renders.

### 13.4 Sections
After the search/quick-filter pass, loans split into three groups, each shown only if non-empty:
1. **"Overdue"** — non-returned loans past the 30-day threshold, sorted oldest-lent-first (the most urgent first).
2. **"Currently lent out"** — non-returned, non-overdue loans, sorted most-recently-lent-first.
3. **"Returned"** — returned loans, sorted most-recently-returned-first (falling back to `dateLent` if no return date somehow present). Cards here render in the muted "disabled" visual style (`.loan-card.returned`, section 3.3) — greyed out but still fully present and readable, never deleted.

**Empty states:** if there are no loan records at all: glyph "🤝", title **"No loans yet"**, subtitle **"Tap the + button to note who you've lent a book to."** If records exist but none match the current search/filter: glyph "🤝", title **"No loans match"**, subtitle **"Try a different search or filter."**

### 13.5 Loan card content and actions
Each `.loan-card` shows: book title, author (if present), "Lent to **[person]**", a status line (**"Lent [date]"** or **"Returned [date]"**), an inline "Overdue" pill (section 3.3) appended to that status line when applicable, and two buttons:
- **"Mark as returned"** (primary; hidden once already returned) — sets `returned = true` and `dateReturned` = today. The card immediately re-renders into the muted "Returned" section.
- **"Remove"** (`.danger-outline`) — custom confirm modal, message `Remove this loan record for "[bookTitle]"?`, confirm label "Remove". On confirm, deletes the loan record outright (available on both active and returned records, for correcting mistaken entries).

### 13.6 Lend a book sheet
Opened by the FAB while on the Lent tab. Fields:
1. **Book** — a `<select>` populated from the eligible-books pool (13.1), each option labeled `"[title] — [author]"` (or just the title if no author). If the pool is empty, the select shows a single disabled option "No owned books available right now" and the Save button is inert.
2. **Author** (read-only preview, shown only once a book is selected and it has an author) — a plain text display of that book's author, updated automatically on selection change. This satisfies "author auto-loads" — the field is never manually typed.
3. **Lent to*** — required text input, placeholder "Friend's name".

Footer: "Cancel" / "Save" (primary). On save (blocked if no book selected or person name is empty after trimming): pushes a new loan record (schema in section 4.3) with `dateLent` = today, `returned: false`, `dateReturned: null`, using the selected book's live `id`/`title`/`author` as the snapshot fields.

### 13.7 Overdue tab-bar indicator
Independent of and in addition to the in-tab "Overdue" section (13.4): the Lent tab's bottom-bar icon shows a small red dot (`.tab-dot.show`, section 3.3) whenever `loans.some(isOverdue)` is true — computed from the **full, unfiltered** loans array, not whatever search/quick-filter is currently applied on the Lent tab itself. This check must run and update **every time the main view re-renders for any tab** (i.e., call the dot-update function from the top-level render dispatcher, not only from the Lent tab's own render function), so the user can see there's an overdue loan without ever opening the Lent tab. It must also update immediately after any action that changes loan data (marking returned, removing a record, adding a new loan) even when that action re-renders only the Lent tab in place.

---

## 14. Shareable Images

Both share features generate a PNG via an off-screen `<canvas>`, then either hand it to the native OS share sheet or fall back to an in-app preview+download modal. **No canvas drawing may hard-code colors** — always read the live CSS custom property values at draw time (via `getComputedStyle(document.documentElement).getPropertyValue('--token')`) so the generated image always matches the app's current theme.

### 14.1 Book share card — 1080×1350px canvas
Top to bottom, all centered horizontally:
1. The wordmark (drawn manually with two `fillText` calls in the same font — "Lee" in `--spine`, "brary" in `--ink` — immediately adjacent, matching the on-screen wordmark treatment).
2. The cover (320×452px, rounded 16px) or the same gradient+"📕" placeholder used elsewhere, scaled up.
3. Title, serif bold 54px, word-wrapped up to 3 lines with a trailing ellipsis if it would need a 4th.
4. Author, 34px, muted.
5. The status pill + ownership pill, drawn as actual rounded-rect shapes with centered text (not images), colored per section 3.1's red/yellow/green and blue/violet rules (this must stay in sync with the on-screen pill colors — do not let this canvas drawing drift to older or different colors).
6. Star row, if rated.
7. Review, if present, rendered in italic serif as a quoted line (curly quotes “ ”), word-wrapped up to 5 lines with ellipsis truncation.
8. Footer: small muted text, **"Shared from Leebrary"**.

### 14.2 Progress share card — 1080×1080px canvas
1. Wordmark (same treatment as above).
2. The big count number, huge serif (220px).
3. Label: "book(s) finished".
4. The period label (e.g. "March 2026", "2026", a date range, or "All time").
5. If chart data exists for the current period (i.e., not the Month view): the same bar chart shown on-screen, redrawn to canvas — bars, per-bar counts above non-zero bars, axis labels below, thin baseline.
6. Footer: **"Shared from Leebrary"**.

There is no third "recap" share card — see the closing note in section 11.6.

### 14.3 Share/download mechanics (shared by both card types)
1. Render the canvas, then `canvas.toBlob(...)` to get a PNG blob.
2. If `navigator.canShare` exists AND returns true for `{files: [aFileWrappingThatBlob]}`, call `navigator.share({files, title:'Leebrary', text: <a one-line caption specific to what's being shared>})`. If the user completes or cancels this, stop here either way (wrap in try/catch; a thrown/rejected share is treated as "fell through to the fallback," not an error to surface).
3. Otherwise (or if the above throws), fall back: create an object URL from the blob and open the **Share modal** — a bottom sheet titled "Share", showing the image in a preview (`max-height:56vh`), a hint line **"Tap and hold the image to save it, or use the button below."**, and two footer actions: "Close" and a "Download image" link (a real `<a download>` element, filename slugified to lowercase-with-hyphens, e.g. `leebrary-the-left-hand-of-darkness.png` or `leebrary-progress-march-2026.png`).

---

## 15. Settings — Backup & Restore

Opened via the header's gear icon. Title: **"Backup & restore"**, with an explanatory paragraph:

> "Your library is saved only in this browser. To use it somewhere else — another browser, or your phone as well as a computer — export a backup file here, then import it there. Importing adds any books not already in your library; it won't duplicate ones that are."

Directly below that paragraph (before the two action buttons):
- A **"Last backup: [date]"** line, or **"Last backup: never"** if `leebrary_last_backup_v1` has never been written. Uses the same `fmtDate` DD-MM-YYYY formatting as everywhere else (section 2), applied to the date portion of the stored ISO timestamp.
- Immediately after it, shown only when the backup is stale (section 15.1): a red-tinted inline warning banner (⚠️ icon + text) reading, if a backup exists, **"It's been over 7 days since your last backup. Export one now to keep your library safe."**, or, if none has ever happened, **"You haven't backed up your library yet. Export a backup file to keep it safe."**
- Both this line and the warning banner are refreshed every time the sheet is opened (not just once on app load), so exporting and then immediately reopening Settings always reflects the just-completed backup with no stale state.

Two full-width stacked buttons:
- **"Export backup file"** — builds `{ app: "leebrary", version: 1, exportedAt: <today's date>, books: <the full current books array> }`, serializes it with 2-space indentation, wraps it in a `Blob` (`application/json`), and triggers a download named `leebrary-backup-<today's date>.json`. (This export currently covers the book library only — the wishlist and loans arrays are not included in the backup payload.) **On a successful export**, also write `new Date().toISOString()` to `leebrary_last_backup_v1` and immediately refresh both the "Last backup" line/warning banner in this sheet and the gear icon's dot (section 15.1) — all three must update together, in the same click handler, with no separate save step.
- **"Import backup file"** — opens a hidden file input (`accept="application/json,.json"`). On file selection, read it as text, `JSON.parse` it (accepting either a raw array, or an object with a `.books` array), and:
  - If parsing fails or the result isn't array-shaped: show the confirm/alert modal with a single "OK" button reading **"That file could not be read as a Leebrary backup."**
  - Otherwise, compute which incoming books are genuinely new (must have an `id`, a `title`, and an `id` not already present among current books). If zero are new: show a single-button notice **"All N book(s) in that file are already in your library — nothing new to import."** Otherwise show a yes/no confirmation: **"Found N book(s) in that file — X new, Y already in your library. Import the X new one(s)?"**, confirm label "Import" — on confirm, push all the new ones into the array, persist, close the settings sheet, and re-render.
  - **Importing does NOT count as a backup** — it never writes `leebrary_last_backup_v1`. Only a successful export does. (Rationale: the freshness indicator specifically tracks "do you have an up-to-date copy of your data safely exported elsewhere," which importing doesn't establish.)
- Below both, a plain **"Close"** button.

### 15.1 Backup-freshness rule (gear icon dot + in-sheet warning)
- Constant: `BACKUP_WARNING_DAYS = 7`.
- **Stale** = `leebrary_last_backup_v1` is absent, OR at least 7 whole calendar days have passed between the stored timestamp and now (same day-counting approach as the Lent tab's `OVERDUE_DAYS` rule in section 13.3, just against a full timestamp rather than a date-only string).
- Whenever the app considers itself stale, show the gear icon's red dot (section 6) AND the in-sheet warning banner (above). Whenever it's fresh (a backup within the last 7 days exists), both are hidden.
- The gear icon's dot must be computed and shown/hidden **on initial app load** (so opening the app after a long gap immediately shows it, without requiring the user to open Settings first) and **re-checked every time an export succeeds** (so exporting immediately clears it). It does not need to be re-checked on a timer while the app sits open — a fresh page load or a successful export are the only two triggers.

---

## 16. Custom Confirm/Alert Modal

A single, generic, reusable modal (not tied to any one feature) that every part of the app calls into for confirmations and notices — used by book deletion, wishlist "Remove", loan "Remove", backup import results, and the cover-photo-failed-to-load notice. It supports two shapes:
- **Confirmation** (yes/no): a message, a "Cancel"-labeled button (customizable text), and a confirm-labeled button (customizable text, e.g. "Remove" or "Import") that runs a supplied callback when tapped.
- **Notice** (single button only): pass `cancelLabel: null` to hide the cancel button entirely, leaving just one button (conventionally labeled "OK") that simply closes the modal with no callback.

Tapping the backdrop (outside the sheet) or the cancel button both simply close it without running any callback. This modal must be the ONLY mechanism used anywhere in the app for confirmations or notices — never call the browser's native `confirm()`/`alert()`.

---

## 17. Surprise Me Popup

Triggered by the "🎲 Choose a book to read" button on the Library tab (section 7.3).

1. Compute the pool: every book in the library with `status === "to-read"`.
2. If the pool is empty: show the custom Confirm/Alert modal (single "OK" button) reading **"Your to-read pile is empty — add some books first!"** — do not open the popup.
3. Otherwise, pick one book **at random** from the pool and open the popup, showing: a close-icon button (`.sheet-close-btn`, top-right) with no separate text-labeled close/cancel button; the picked book's cover (or placeholder), centered; its title (serif, bold, centered); its author (muted, centered) if present; and two footer buttons, **"Try again"** and **"Start reading"** (primary).
4. **"Try again"** re-rolls: recompute the to-read pool fresh (in case something changed), and — **if the pool has more than one book** — exclude the currently-shown book from the reroll so it's guaranteed to show something different; if only one book exists, it's shown again (nothing else to pick). The popup stays open and its contents update in place; it does not close and reopen.
5. **"Start reading"**: sets the shown book's `status` to `"reading"` (no other fields change — this does NOT touch `dateFinished` or `readHistory`), persists, and closes the popup. **This never navigates to the Book Detail view** — the Library tab underneath remains exactly where the user left it, simply re-rendered so the status pill updates if that book happens to be visible.
6. The **✕ close icon** (or tapping the backdrop) closes the popup at any point with no side effects — the picked book's status is left untouched.

This popup is functionally and visually distinct from Book Detail: it is a lightweight, disposable suggestion surface, not a navigation destination.

---

## 18. Full Interaction/State Summary (for implementers building the JS)

Track these pieces of top-level state (plain variables, no external state library):
- `books`: array, the book library, loaded from and saved to `leebrary_books_v1` on every mutation.
- `wishlist`: array, the buy list, loaded from and saved to `leebrary_wishlist_v1` on every mutation.
- `loans`: array, the lending log, loaded from and saved to `leebrary_loans_v1` on every mutation.
- `currentTab`: `"library" | "progress" | "wishlist" | "lent"` — which bottom tab was last chosen.
- `view`: `"library" | "progress" | "wishlist" | "lent" | "detail"` — what's actually rendered right now (differs from `currentTab` only while viewing a book's detail page).
- `selectedBookId`: the book currently shown in detail view, or null.
- `libraryViewMode`: `"list" | "grid"`.
- `statusFilters`, `ownerFilters`, `ratingFilters`: `Set` instances for the multi-select Library filters.
- `filterYear`, `filterMonth`, `filterDay`: number or `null` each (`null` = "Any") — the hierarchical Date-added drill-down filter (section 8, item 4). `filterMonth` is only meaningful when `filterYear` is set, and `filterDay` only when `filterMonth` is set; the app enforces this by resetting finer fields to `null` whenever a coarser one changes.
- `searchTerm`: string — Library search.
- `sortDir`: `"desc" | "asc"` — Library sort direction.
- `wishSearchTerm`: string — List tab search.
- `lentSearchTerm`: string — Lent tab search.
- `lentStatusFilter`: `"all" | "active" | "returned"` — Lent tab quick filter.
- `progressPeriod`: `"month" | "year" | "custom" | "all"`.
- `progressYear`, `progressMonth`: numbers, shared between the Month and Year progress views.
- `customFrom`, `customTo`: date strings for the Custom progress view.
- `editingId`: the id of the book currently being edited via the Add/Edit sheet, or null when adding a new one.
- `editingWishId`: the id of the List entry currently being edited via the "Add to your list" sheet's edit mode (section 12), or null when adding a new one — same pattern as `editingId` above, one level down for the wishlist.
- `surpriseBookId`: the id of the book currently shown in the Surprise Me popup, or null when it's closed.
- `OVERDUE_DAYS`: constant, `30`.

Tapping any bottom tab button always sets both `currentTab` and `view` to that tab (i.e., always exits detail view back to a tab). Opening a book's detail sets `view = "detail"` and `selectedBookId`, without changing `currentTab`. The detail page's "back" link sets `view = currentTab`. The Surprise Me popup never touches `view` or `selectedBookId` at all.

---

## 19. Acceptance Checklist

An implementation is complete when all of the following hold:

- [ ] Opening the file directly (no server) works fully offline except for the two Google Fonts.
- [ ] Adding a book with only a title and author succeeds; every other field can be left untouched. The same holds for adding a wishlist entry (title/author required, publisher optional) and lending a book (book + person required).
- [ ] The book tile — in both list and grid view — shows exactly: cover/placeholder, title, author, one status tag, one ownership tag, and stars only if rated.
- [ ] Status tags read **TBR** / **Reading** / **Done** and are colored red (to-read) / yellow (reading) / green (done); ownership tags are blue (owned) / violet (borrowed) — consistently across list pills, grid dots, the detail page, the ownership toggle button, the shareable book-card image, the status-choice buttons in the Add/Edit sheet, and the Filters sheet's status chips.
- [ ] Toggling list/grid view preserves the current filters, search, sort, and month grouping.
- [ ] A "to-read"/"reading" book always appears under the current month's group, never under the month it was added, no matter how old its `dateAdded` is; a "done" book appears under its most recent completion's month.
- [ ] Tapping a tile (list or grid) opens its detail page; Prev/Next there step through the same filtered/sorted set as the Library list (unaffected by month grouping); Back returns to Library.
- [ ] Marking a book done today, reopening it, and marking it done again produces TWO entries in `readHistory` and the detail page's "Read history" section lists both with correct ordinals and dates.
- [ ] A book logged with a finish date earlier than today (its add date) shows only "Finished [date]" with no "Added" text.
- [ ] Adding a NEW book with status Done and the "Not sure of the exact date" toggle on shows a year picker instead of a date input, saves a `dateFinished` of `YYYY-07-02` for the chosen year, and the detail page shows "sometime in [year]" instead of a formatted date wherever that finish date would otherwise appear. The same toggle is completely absent when EDITING an existing book, regardless of whether that book currently has an approximate date on file.
- [ ] Every human-readable date shown anywhere in the app (Library/Detail dates, Lent lend/return dates, Settings' last-backup date, Custom-range progress labels) renders as `DD-MM-YYYY`, e.g. `05-09-2026` — never a month-name format, and never the raw `YYYY-MM-DD` storage format.
- [ ] Deleting a book, removing a wishlist entry, and removing a loan record all show the custom confirm modal (never a native browser dialog) and actually perform the deletion on confirm.
- [ ] Sorting toggles direction correctly even when several books share the exact same added (or done) date, thanks to the id-timestamp tiebreaker.
- [ ] Switching the quick filter to "Done" changes the sort label to "Sort by done date" automatically; every other filter state sorts by added date.
- [ ] The Filters sheet supports selecting multiple chips per group simultaneously and its "Show N books" button count updates live as chips are toggled.
- [ ] The Filters sheet's "Date added" Year/Month/Day selects correctly drill down: picking only a year shows every book added in that year; additionally picking a month narrows to that year+month; additionally picking a day narrows to that exact date. Month is disabled until a year is chosen and Day is disabled until a month is chosen; picking a new year resets Month/Day back to "Any", and picking a new month resets Day back to "Any". The Day select's option count correctly adapts to the chosen month/year (e.g. February shows 28 or 29 days depending on leap year).
- [ ] Progress → Month has a working month dropdown and shares the year arrows with the Year view; Year shows a 12-bar chart; Custom requires both dates before showing results; All time buckets by year. No "Year in reading"/recap feature exists anywhere.
- [ ] Both "Share" buttons (book detail, progress) produce a themed PNG, colored consistently with the current tag palette, and either open the native share sheet or the in-app preview/download modal.
- [ ] Export produces a valid, re-importable JSON file; importing it into a fresh/different browser storage merges by id without duplicating.
- [ ] The whole app fits the viewport with no page-level scroll; only the content area scrolls when a list is long.
- [ ] A home-screen "Add to Home Screen"/install shows the custom "L" monogram icon, not a generic browser icon or webpage screenshot.
- [ ] On the List tab, "Mark as bought" creates a TBR library book from title+author only (publisher discarded) and removes the wishlist entry; manually removing a wishlist entry never creates a library book. The tab shows a live "N books on your list" count (plus "· M shown" while a search narrows the view), and each entry's "Edit" button reopens the same sheet pre-filled, updating that entry in place rather than creating a duplicate.
- [ ] On the Lent tab, the book picker in "Lend a book" excludes borrowed copies and books already out on an active loan; selecting a book auto-fills its author preview with no typing.
- [ ] Marking a loan "returned" visually mutes its card and moves it to the Returned section without deleting it; the book's own detail page reflects both "still out" and "returned" states correctly under "Lent out".
- [ ] A loan lent 30+ days ago and not yet returned shows in the "Overdue" section with an "Overdue" pill, AND makes the Lent tab's bottom-bar icon show a red dot even while the user is on a different tab. The Lent tab also shows a live "N books currently lent out" count, computed from all active loans regardless of the current search/quick-filter.
- [ ] Tapping "Choose a book to read" with an empty TBR pile shows a friendly notice and does not open the popup; with books available, it opens a popup (not a navigation) showing a random pick; "Try again" swaps to a different book when more than one is available; "Start reading" marks it reading and closes without navigating to its detail page; the ✕ icon closes without changing anything.
- [ ] The gear icon shows a red dot, and the Settings sheet shows a matching warning banner, whenever the last successful backup is 7+ days old or has never happened; both clear immediately after a successful export, and the "Last backup: [date]" line updates in the same action.
