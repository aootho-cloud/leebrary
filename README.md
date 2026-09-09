# Product Requirements Document: Leebrary

**Product name:** Leebrary
**Tagline:** "a library that remembers"
**Type:** Single-file, client-only mobile web app (installable as a home-screen PWA)
**Version:** 1.0
**Audience for this document:** an engineer who has never seen the app, building it from scratch. Following this document exactly, with no deviations or personal interpretation, should produce a functionally and visually identical app.

---

## 1. Product Overview

Leebrary is a personal reading tracker. One person logs the books they own, are reading, or have finished; rates and reviews them; tracks whether a copy is owned or borrowed; tracks rereads with full history; and views their reading progress by month, year, custom range, or all-time. It is built for **personal, single-user use** — there is no login, no multi-user support, and no server or database. All data is stored locally in the browser via `localStorage`, under a single key.

The app must be deliverable as **one self-contained `.html` file** with no build step, no bundler, and no external JS framework. It must run by simply opening the file (or a URL pointing to it) in a mobile browser.

### 1.1 Non-goals (explicitly out of scope)
- No user accounts, login, or authentication of any kind.
- No server, API, or database. No network calls except loading two Google Fonts.
- No multi-device sync. Data lives only in the browser that created it.
- No offline service worker / caching strategy beyond what the browser does by default.
- No support for multiple simultaneous users of the same installed instance.

---

## 2. Platform & Technical Constraints

- **Single HTML file.** All CSS lives in one `<style>` block in `<head>`. All JavaScript lives in one `<script>` block at the end of `<body>`, wrapped in an immediately-invoked function expression (IIFE) so nothing leaks to the global scope.
- **No frameworks, no build tools.** Plain HTML/CSS/JavaScript (ES2017+ features are fine: `async/await`, template literals, arrow functions, `Set`, optional chaining not required).
- **Rendering approach:** the app is a single-page app with no router. A `<main id="main">` element's `innerHTML` is fully re-rendered on every state change from one of three top-level "views": `library`, `progress`, `detail`. There is no virtual DOM — re-render means rebuilding an HTML string and setting `.innerHTML`, then re-attaching event listeners.
- **Persistence:** `window.localStorage`, single key `leebrary_books_v1`, whose value is `JSON.stringify(books)` where `books` is an array of book objects (schema in section 4). Load on startup with `localStorage.getItem`, parsed with `JSON.parse`; on any parse failure, fall back to an empty array. Save after every mutation, immediately (no debounce, no batching).
- **Fonts:** Google Fonts, loaded via a single `<link>` tag:
  - `Source Serif 4` — weights available: 400, 600, 700, with optical-size axis `opsz` range `8..60`. Used for all headings, titles, big numbers, and anywhere a "book-ish" serif voice is wanted.
  - `Inter` — weights 400, 500, 600, 700. Used for all UI/body text.
  - Exact `<link>` href: `https://fonts.googleapis.com/css2?family=Source+Serif+4:opsz,wght@8..60,400;8..60,600;8..60,700&family=Inter:wght@400;500;600;700&display=swap`, preceded by `<link rel="preconnect" href="https://fonts.googleapis.com">`.
- **Viewport lock (critical layout requirement):** the app must fill exactly the visible browser viewport with **no page-level scrolling**. Only the app's own content region scrolls internally. Implementation:
  - `html, body { height: 100%; }`
  - `body { margin:0; height:100dvh; overflow:hidden; display:flex; justify-content:center; }` — `100dvh` (dynamic viewport height) is declared after height:100% so unsupported browsers keep the `100%` fallback and supporting browsers get the more accurate dynamic value that correctly handles mobile browser chrome show/hide.
  - `#app { width:100%; max-width:430px; height:100%; overflow:hidden; position:relative; display:flex; flex-direction:column; }` — this is the phone-shaped frame, centered on wider screens via the parent's `justify-content:center`, with a soft outer shadow (`box-shadow: 0 0 40px rgba(0,0,0,0.12)`) so it visually reads as a floating "device" on desktop widths.
  - `main { flex:1; overflow-y:auto; }` — this is the ONLY element that scrolls. Header, bottom tab bar, and the floating action button are all outside or fixed relative to this, so they never move.
- **Mobile home-screen installability:**
  - Static meta tags in `<head>`: `apple-mobile-web-app-capable=yes`, `apple-mobile-web-app-status-bar-style=default`, `apple-mobile-web-app-title=Leebrary`, `mobile-web-app-capable=yes`, `theme-color=#5B3A5C`.
  - A static `<link rel="apple-touch-icon">` and `<link rel="icon">`, both pointing to the **same baked-in base64 PNG data URI** (see section 3.4 for the icon's visual spec) — baked in statically (not generated at runtime) so it's available the instant the page loads, with no dependency on JavaScript execution timing.
  - At runtime (on load), the app also dynamically constructs a Web App Manifest object (`name`/`short_name: "Leebrary"`, `start_url: "."`, `display: "standalone"`, `background_color` = the `--paper` token, `theme_color` = the `--spine` token, `icons` array referencing the same baked-in icon at 192×192 and 512×512), serializes it, wraps it in a `Blob` of type `application/manifest+json`, and appends a `<link rel="manifest">` pointing at `URL.createObjectURL(blob)`. This is best-effort progressive enhancement for Android's "Install app" flow; wrap the whole thing in try/catch and fail silently since it's non-critical.
- **No native `confirm()`/`alert()`/`prompt()`.** These are unreliable or fully blocked in some hosting/sandbox contexts. The app must implement its own in-app modal for both "yes/no" confirmations and single-button "OK" notices (spec in section 8.9). Every place that would naturally want a browser confirm dialog (delete a book, confirm an import, report a failed image load) must use this custom modal instead.

---

## 3. Visual Design System

### 3.1 Color tokens
Define these as CSS custom properties on `:root`. Every color used anywhere in the app must trace back to one of these — no other hard-coded colors except pure white (`#fff`) for text-on-solid-color and a small number of pill-background tints noted in section 3.3.

| Token | Hex | Used for |
|---|---|---|
| `--paper` | `#F1E7E9` | App background |
| `--paper-deep` | `#E3D2D6` | Page background behind the app frame; inactive segmented-control track |
| `--card` | `#FBF6F5` | Card / input / button surfaces |
| `--ink` | `#2B2230` | Primary text |
| `--ink-soft` | `#7C6B75` | Secondary/muted text |
| `--spine` | `#5B3A5C` | Primary brand color — active states, primary buttons, "Reading" status pill, cover placeholder gradient start, wordmark "Lee" |
| `--spine-deep` | `#3B2640` | Darker brand shade — big numbers, cover placeholder gradient end, "Owned" pill text |
| `--brass` | `#B8863A` | Gold accent — star ratings, "Borrowed" pill |
| `--brass-bg` | `#F1E2C4` | "Borrowed" pill background |
| `--leather` | `#8C5A3A` | "Read"/done status pill text |
| `--leather-bg` | `#F0DDC8` | "Read"/done status pill background |
| `--rust` | `#A6483B` | Destructive actions (delete), filter-count badge |
| `--line` | `#E0CCD1` | Borders, dividers, inactive/empty elements |
| `--shadow` | `0 2px 10px rgba(43,34,48,0.08)` | Standard soft card/element shadow |

Two additional inline (non-tokenized) tints used only for pills, chosen to sit visually between the tokens above:
- "To read" status pill background: `#ECD9D3` (text uses `--ink-soft`)
- "Owned" ownership pill background: `#EDE2EC` (text uses `--spine-deep`)

### 3.2 Typography
- Headings, titles, brand name, big stat numbers, sheet `<h2>` titles: `'Source Serif 4', serif`, weight 600–700.
- All body/UI text, buttons, labels, inputs: `'Inter', system-ui, sans-serif`.
- The brand wordmark "Leebrary" is rendered as `<span class="script">Lee</span>brary` where **both parts use the identical font/weight/size** (no different typeface) — the only difference is that `.script` is colored `--spine` while the rest of the word inherits `--ink`.

### 3.3 Core components (describe visual spec precisely; exact CSS should mirror this)
- **Segmented control** (`.segmented`): a full-width, rounded-11px track (`--paper-deep` background, 3px padding), containing flex buttons with no visible border; the active button gets a white-ish (`--card`) pill background and the standard shadow token, inactive buttons are transparent with `--ink-soft` text. A `.scroll` modifier variant allows horizontal scrolling with non-stretching buttons, used for the rating-filter row.
- **Pills** (`.pill`): small fully-rounded (`border-radius:100px`) tags, 11px bold text, a small 5×5px solid dot before the label, used for status (to-read/reading/read) and ownership (owned/borrowed).
- **Star rating display**: five `★` glyphs; filled stars colored `--brass`, unfilled colored `--line`. Two sizes: normal (14px, used on the detail page) and `.small` (12px, used on library tiles).
- **Buttons** (`.btn`): rounded-9px, 1px `--line` border, `--card` background, `--ink` text, 12.5px bold. Modifiers: `.primary` (solid `--spine` bg, white text), `.ghost-brass` (light `--brass-bg` bg, `--brass` text; `.active` state inverts to solid `--brass` bg with white text — used for the borrowed/owned toggle button), `.danger-outline` (transparent bg, `--rust` border+text — used for Delete), `.danger-solid` (solid `--rust` bg, white text — used for the destructive confirm button in the custom confirm modal).
- **Cards** (`.book-card`): `--card` background, 1px `--line` border, 14px radius, standard shadow, 14px/15px padding, `cursor:pointer` (the whole card is tappable) with a subtle `scale(0.985)` active-press feedback and a visible focus ring for keyboard users (`:focus-visible { outline: 2px solid var(--spine) }`) — the card also has `tabindex="0"` and `role="button"` for keyboard accessibility, responding to both `Enter` and `Space`.
- **Bottom sheets / modals** (`.sheet-backdrop` + `.sheet`): full-screen semi-transparent scrim (`rgba(43,34,48,0.4)`) that is `display:none` by default and `display:flex; align-items:flex-end` when given an `.open` class; the sheet itself slides up from the bottom (`translateY(30px)→0` with fade, 0.22s ease), rounded 20px top corners only, `max-height:88vh` with internal scroll. Clicking the backdrop itself (not the sheet) closes it.
- **Floating Action Button** (`.fab`): 52×52px, 16px radius, solid `--spine` background, white "+", fixed to the bottom-right of the app frame (`position:absolute; right:20px; bottom:92px` — sitting just above the tab bar), with a colored drop shadow tinted `--spine`. Hidden (via JS `style.display`) whenever the current view is `detail`.
- **Bottom tab bar** (`nav.tabbar`): fixed to the bottom of the app frame, `--card` background, top border `--line`, two flex buttons ("📖 Library", "📈 Progress" — emoji above label, stacked vertically), active tab text colored `--spine`.

### 3.4 App icon (home-screen logo)
A square image (generate at 192×192 and 512×512), **not pre-rounded** (let the OS apply its own mask):
1. Fill the entire square with a linear gradient from `--spine` (top-left) to `--spine-deep` (bottom-right).
2. Draw a solid horizontal bar in `--brass` across the full width at the very bottom, with height equal to 4.5% of the icon size (a "bookmark ribbon" accent).
3. Centered in the square, draw a bold serif capital letter **"L"** (font: `700 {58% of icon size}px serif`), fill color `--paper` (the light paper tone, not pure white), vertically positioned so its optical center sits slightly above true center (baseline `middle`, drawn at `y = size/2 - size*0.04` roughly, i.e. nudged up to account for the letter's descender-free shape).

This same image is what both the static `<link rel="apple-touch-icon">`/`<link rel="icon">` and the dynamically-built manifest's `icons` array must reference.

---

## 4. Data Model

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
  readHistory: string[]  // array of "YYYY-MM-DD" strings, one per completed read, in chronological order of when they were recorded (append-only; see section 8.10 for exact rules on when entries are added vs. corrected)
}
```

**ID scheme:** `uid()` returns `'b_' + Date.now() + '_' + Math.random().toString(36).slice(2,8)` — i.e. `b_<millisecond-timestamp>_<6-char-random-base36>`. This is important beyond uniqueness: the millisecond timestamp embedded in the id is later reused as a **sort tiebreaker** (section 8.5) whenever two books share the same date-only value, since a date-only field can't distinguish which of several same-day entries came first.

**`readHistory` vs. `dateFinished` back-compatibility:** a helper `getReadDates(book)` must be used everywhere read-completion dates are needed for aggregate purposes (progress stats, sort-by-done-date): if `readHistory` has entries, return it; otherwise, if `dateFinished` is set, return a single-element array `[dateFinished]`; otherwise return `[]`. This lets the rest of the app treat every book uniformly even though `readHistory` was added after `dateFinished` conceptually (i.e. it must gracefully handle old-shaped data that only has `dateFinished`).

---

## 5. Information Architecture

Bottom tab bar has exactly two destinations: **Library** and **Progress**. A third internal "view" — **Book Detail** — is reached only by tapping a book card from the Library list (never from a tab), and has its own dedicated back-navigation; the tab bar's active-tab highlighting still reflects whichever of Library/Progress you came from.

Five modal "sheets" exist, all using the same bottom-sheet visual pattern (section 3.3), each independent and never nested inside another:
1. **Add / Edit book** — opened by the FAB (add mode) or the Detail page's "Edit" button (edit mode, pre-filled).
2. **Filters** — opened by the funnel icon on the Library page.
3. **Confirm/Alert** — a generic reusable modal, opened programmatically wherever the app needs a yes/no confirmation or a single-button notice.
4. **Share** — opened after generating a shareable image, shows a preview and a download link.
5. **Backup & restore (Settings)** — opened by the gear icon in the header.

---

## 6. Header

Fixed at the top of the app frame, not part of the scrolling content. Contains, left-to-right:
- The wordmark (section 3.2) as a heading, with a tagline directly beneath it in small muted text: **"a library that remembers"**.
- A circular-cornered icon button on the far right (gear/settings icon, using a standard 24×24 viewBox "settings" SVG glyph — two concentric shapes: a small circle plus the classic 8-notch gear outline, stroked not filled, `currentColor`), which opens the Backup & Restore sheet.

A 1px bottom border (`--line`) separates the header from the content area.

---

## 7. Library Tab (default view on load)

Rendered top-to-bottom inside `<main>`:

### 7.1 Quick filter row
A `.quick-filter-row` containing:
- A 3-option segmented control: **All / To read / Done**. This controls only the `status` dimension and is a convenience shortcut for the two most common single-status views (it intentionally does NOT include "Reading" as a quick option — that's only reachable via the Filters sheet).
  - Clicking **All** clears the status filter set entirely.
  - Clicking **To read** sets the status filter to exactly `{"to-read"}`.
  - Clicking **Done** sets the status filter to exactly `{"done"}`.
  - The segmented control's active button reflects the CURRENT filter state: "All" is active only when the status-filter set is empty; "To read"/"Done" are active only when the set is exactly that single value. If the set holds any other combination (e.g. "Reading" alone, or multiple statuses together, set via the Filters sheet), none of the three quick buttons show as active — this is expected and correct.
- A funnel-icon button (same 40×40 rounded-square style as other icon buttons) that opens the Filters sheet. If any filter is active that ISN'T representable by the quick row, show a small red numeric badge in the button's top-right corner. The badge count = (number of selected ownership filters) + (number of selected rating filters) + (1 if "Reading" is among the selected statuses, else 0). This is an approximate "how much extra filtering beyond the quick row is active" indicator, not a strict total.

### 7.2 Sort row
A `.sort-row` with a label on the left reading **"Sort by added date"** or **"Sort by done date"**, and a pill-shaped toggle button on the right reading **"↓ Recent first"** or **"↑ Oldest first"**.

**Which field is used is fully automatic, never user-chosen directly:**
- If the current status-filter set is EXACTLY `{"done"}` (i.e., the user is viewing only Done books — whether via the quick "Done" button or via the Filters sheet), sort by **done date** (the most recent entry in `getReadDates(book)`, or empty string if none).
- In every other case (All, To read, Reading, or any other combination), sort by **added date** (`dateAdded`).

Clicking the direction toggle flips between `desc` (recent-first) and `asc` (oldest-first) and re-renders; it does not affect which field is used.

**Sort algorithm, exact:** compare the two books' string sort-keys (either both `dateAdded` values or both "most recent done date" values, per the rule above) with `localeCompare`, in the direction implied by `sortDir`. **Critically, if the two keys are equal** (e.g. two books both added "today", since the field only has day-level granularity) **fall back to comparing the millisecond timestamp embedded in each book's `id`** (see section 4), in the same direction as the primary sort. Without this tiebreaker, same-day entries would silently fail to respond to the direction toggle at all (this was a real, user-reported bug during development — do not omit this fallback).

### 7.3 Search box
A single text input, placeholder **"Search title or author"**, filtering case-insensitively against the concatenation of `title + ' ' + author`. Typing must not lose focus or cursor position on re-render (re-focus the input and restore cursor to the end after each keystroke's re-render).

### 7.4 Book list
For each book passing all active filters and the search term, in the current sort order, render a tappable card (section 3.3) containing, top-to-bottom / left-to-right:
- A 42×58px cover thumbnail (`object-fit:cover`, rounded 5px) if `cover` is set; otherwise a same-sized placeholder — a rounded rectangle filled with the `--spine`→`--spine-deep` gradient, centered with a plain "📕" glyph.
- Title (serif, 16.5px, weight 600) and, if present, author (13px, `--ink-soft`) stacked to the right of the thumbnail.
- Below title/author: a pill row with exactly two pills — the status pill (label "To read" / "Reading" / "Read", colored per section 3.1) and the ownership pill ("Owned" or "Borrowed" — **always show exactly one of these two, never both, never neither**).
- If `rating > 0`, a small star row beneath the pills.

**Nothing else appears on the tile** — no action buttons, no menu, no dates. The entire card is the tap target; tapping it (or pressing Enter/Space while it's focused) navigates to the Book Detail view for that book.

### 7.5 Empty states
- If there are zero books at all: glyph "📚", title **"Your shelf is empty"**, subtitle **"Tap the + button to add your first book."**
- If there are books but none match the current filters/search: glyph "📚", title **"No books match"**, subtitle **"Try different filters or search."**

### 7.6 Floating Action Button
Bottom-right "+" button, opens the Add/Edit sheet in "add" mode (no book passed in, all fields blank/default).

---

## 8. Filters Sheet

Opened via the funnel icon. Title: **"Filters"**. Three independently multi-selectable chip groups (tapping a chip toggles its membership in the corresponding `Set` state — multiple chips within a group can be active simultaneously, and combining criteria across groups is a logical AND):

1. **"Reading status"** — chips: To read / Reading / Done.
2. **"Ownership"** — chips: Owned / Borrowed.
3. **"Rating"** — chips: ★1 / ★2 / ★3 / ★4 / ★5. Selecting a rating chip means "books rated exactly this many stars" (selecting several means "rated any of these values" — it is NOT a minimum threshold).

Below the three groups, two full-width action buttons:
- **"Clear all"** — empties all three filter sets and immediately updates the chip active-states and the live count (does not close the sheet).
- **"Show N book(s)"** (this button's own label is dynamic, always reflecting the live count of books that would match the CURRENTLY toggled — not yet applied — filter state, combined with the current search term) — tapping it closes the sheet and re-renders the Library list with the new filters applied.

Filtering logic used both for the live count in this sheet and for the actual Library list: a book matches if (status set is empty OR its status is in the set) AND (ownership set is empty OR its owned/borrowed state is in the set) AND (rating set is empty OR its rating is in the set) AND (search term is empty OR title+author contains it, case-insensitive).

---

## 9. Add / Edit Book Sheet

Title: **"Add a book"** (add mode) or **"Edit book"** (edit mode, all fields pre-filled from the existing book).

Fields, in this exact order:
1. **Title*** — required text input, placeholder "The Left Hand of Darkness".
2. **Author*** — required text input, placeholder "Ursula K. Le Guin". (Only these two fields are marked with a red `*` and only these two block saving if empty — every other field below is optional with a sensible default.)
3. **Cover image (optional)** — a file input (`accept="image/*"`, hidden) triggered by a dashed-border "Choose a photo" label-button. On selection:
   - Read the file, load it into an `Image`, and **resize/compress it client-side before storing**: scale so the longer edge is at most **320px** (preserve aspect ratio), draw to an off-screen `<canvas>`, and export via `canvas.toDataURL('image/jpeg', 0.72)`. This keeps stored covers small since everything lives in `localStorage`.
   - Once set, show a 52×72px preview thumbnail plus a "Remove" button (which clears the cover and re-shows the "Choose a photo" control).
4. **Status** — a 3-button single-select group: To read / Reading / Read (internal value `"done"`). Choosing "Read" reveals field 5.
5. **Date finished** (only visible when status = Read) — a native date input, defaulting to today's date when first shown.
6. **Borrowed copy** — a toggle switch (default off = owned).
7. **Your rating (optional)** — five tappable star buttons (unfilled `--line`, filled `--brass` up to the chosen value). Tapping the currently-selected star again clears the rating back to 0 (there is no separate "clear rating" control).
8. **Notes / review (optional)** — a multi-line textarea, placeholder "What stood out to you?".

Footer: **"Cancel"** (closes without saving) and **"Save book"** (primary).

### 9.1 Save validation
Block saving (and focus the offending field) if Title or Author is empty after trimming whitespace. Every other field may be left at its default.

### 9.2 Save logic — new book
Push a new book object with a fresh `uid()`, `dateAdded` = today, and: if status is "done", `dateFinished` = the chosen finish date AND `readHistory` = `[thatDate]`; otherwise `dateFinished = null` and `readHistory = []`.

### 9.3 Save logic — editing an existing book (exact rule, do not simplify)
- Update title/author/status/borrowed/cover/rating/review directly from the form.
- If the new status is **"done"**:
  - If the book's status was ALREADY "done" before this edit (i.e. you're editing an already-finished book, not freshly completing it) AND it already has at least one `readHistory` entry: **overwrite the LAST entry** in `readHistory` with the chosen finish date, and set `dateFinished` to that same date. (Rationale: the user is correcting the date of the read they're currently editing, not logging a brand-new read.)
  - Otherwise (the book is transitioning INTO "done" status via this edit, from "to-read" or "reading"): **append** the chosen finish date as a NEW entry to `readHistory`, and set `dateFinished` to it. (Rationale: this is a genuinely new completion event.)
- If the new status is NOT "done": set `dateFinished = null`. **Never delete or modify `readHistory` in this branch** — past completions must remain on permanent record even if the book's current status is reset back to "reading" or "to-read".
- After saving an edit, return the user to the Book Detail view (not the Library list) — editing is only ever reached FROM the detail page in this app (there is no edit entry point on the library tile itself), so it should feel like "the detail page updated," not "you navigated away."

---

## 10. Book Detail Page

Reached only by tapping a library tile. Not a modal — it's a third top-level "view" that fully replaces the Library/Progress content in `<main>`, with its own back-navigation.

Layout, top to bottom:

1. **"‹ Back to library"** text button — returns to whichever tab (Library or Progress — in practice always Library, since that's the only entry point) was active before entering detail.
2. **Prev/Next pager** (only rendered if the current filtered/sorted Library list is non-empty): "‹ Prev", a centered "N of M" count, "Next ›". This pager iterates over **the exact same filtered + sorted list currently shown on the Library page** (recomputed fresh each time detail renders, using the live filter/sort/search state) — so paging through books here mirrors paging through the list, without ever returning to the list itself. Buttons disable themselves at the start/end of the list. If the current book isn't found in that list at all (e.g. it was just edited to no longer match the active filters), default the index to 0 rather than erroring.
3. **Hero row**: a large (104×148px) cover image or gradient-placeholder (same visual treatment as the library tile's placeholder, scaled up, with a bigger "📕" glyph), next to the title (21px serif bold) and author (13.5px muted) stacked beside it.
4. **Pill row**: status pill + ownership pill (identical rules to the library tile).
5. **Star row** (only if rated).
6. **Dates line** — normally reads **"Added [date] · Finished [date]"** (omitting the "· Finished" part if not currently done). **Special case:** if the book's `dateFinished` is earlier than its `dateAdded` (this happens when someone logs a book they actually finished reading in the past, weeks/months before adding it to the app today) — show **only** "Finished [date]", omitting the "Added" part entirely, since showing "Added today · Finished months ago" reads as confusing/backwards.
7. **Reread note** (only shown when the book's CURRENT status is not "done" AND it has at least one prior read in `readHistory`): a small italic line, e.g. *"Read 2 times before · last finished [date]"*.
8. **Status action row** — buttons depend on current status:
   - `to-read`: "Start reading" + "Mark done" (primary)
   - `reading`: "Mark done" (primary) only
   - `done`: "Reopen" only
   - Always also present: a borrowed/owned toggle button reading "Mark borrowed" (if currently owned) or "Mark owned" (if currently borrowed), styled `.ghost-brass` (and `.active` when borrowed).
   - **"Mark done" behavior:** set status to "done", set `dateFinished` to today's date, and **append today's date as a new `readHistory` entry** — this is always treated as a fresh completion event, regardless of any prior history.
   - **"Reopen" behavior:** set status back to "reading" and clear `dateFinished` to `null`. **Do not touch `readHistory`** — this preserves the permanent record of the read that's being reopened/redone.
   - **"Start reading"**: sets status to "reading", nothing else changes.
   - **Borrowed toggle**: flips the boolean, nothing else changes.
9. **Review** (only if present): a labeled "Notes" section with the review text as a paragraph.
10. **Read history list** (only rendered if `readHistory.length > 1` — a single read is already covered by the dates line above, so this section only earns its place once there's more than one completion to compare): a small ordered list, one line per entry, formatted **"1st time — [date]"**, **"2nd time — [date]"**, etc. (proper English ordinals: 1st/2nd/3rd/4th...).
11. **Footer row** — three equal-width buttons: **"Share"**, **"Edit"**, **"Delete"** (styled `.danger-outline`).
    - Share → generates and offers the book's shareable image (section 12.1).
    - Edit → opens the Add/Edit sheet pre-filled with this book.
    - Delete → opens the custom confirm modal with message **`Remove "[title]" from your shelf?`**, confirm button labeled "Remove"; on confirm, actually remove the book from the array, persist, and navigate back to the Library tab. (During development, this specific action surfaced a real bug: native `confirm()` silently does nothing in some hosting contexts, so nothing appeared to happen and nothing was deleted. The custom in-app modal exists specifically to avoid this failure mode — do not substitute the browser's native `confirm()` here or anywhere else.)

---

## 11. Progress Tab

### 11.1 Period selector
A 4-option segmented control: **Month / Year / Custom / All time**.

### 11.2 Month view
- A month `<select>` dropdown (all 12 months, full names, e.g. "March"), defaulting to the current real-world month.
- Below it, the exact same year prev/next arrow control used by the Year view (see 11.3) — sharing state, so switching between Month and Year views keeps the same year in context.
- Shows the count of books completed in that specific month+year (using every entry from `getReadDates()` across all books, not just books whose CURRENT status is "done" — a book that was read in March and later reopened still counts for March).
- No chart in this view (a single month has nothing to break down further).
- Below the stat, a list titled **"Finished in this period"**, one row per completed-read EVENT (not per book — if a book was read twice in the same month, it appears twice, once per actual completion date) showing title, author, and that specific date.

### 11.3 Year view
- A year switcher: "‹" button, the year number (serif, bold), "›" button, defaulting to the current real-world year.
- Total count for that year, plus a 12-bar chart (Jan–Dec), each bar's height proportional to that month's count within the year (bars with zero count render at a fixed minimum height in the muted `--line` color rather than being invisible; non-zero bars show their count above them, in `--spine`).
- Same "Finished in this period" event list below, scoped to the year.

### 11.4 Custom view
- Two native date inputs, "From" and "To".
- Once both are set: total count, a chart broken down by month across the range (label each bar with the month abbreviation, and additionally the 2-digit year if the range spans more than one calendar year), and the same event list.
- Before both dates are set: show only the prompt "Choose a date range" as the period label, no stat/chart/list.

### 11.5 All time view
- Total count across every read event ever recorded, no date filter.
- A chart broken down by YEAR (not month) — one bar per calendar year that has at least one completion, spanning from the earliest to the latest year with data.
- Same event list, unscoped.

### 11.6 Shared elements across all four period views
- A stat hero: large serif number (52px) + a label below it reading **"[N] book(s) finished · [period label]"**.
- A **"Share progress"** button directly under the stat hero label, which generates the progress share image (section 12.2) for whatever is currently on screen (the exact count, period label, and chart data visible at the moment of tapping).
- If the event list for the period is empty (and, for Custom view, only once both dates ARE set — don't show this empty-state while still prompting for a range), show: glyph "🔖", title **"Nothing finished yet"**, subtitle **"Mark a book done to see it here."**

---

## 12. Shareable Images

Both share features generate a PNG via an off-screen `<canvas>`, then either hand it to the native OS share sheet or fall back to an in-app preview+download modal. **No canvas drawing may hard-code colors** — always read the live CSS custom property values at draw time (via `getComputedStyle(document.documentElement).getPropertyValue('--token')`) so the generated image always matches the app's current theme.

### 12.1 Book share card — 1080×1350px canvas
Top to bottom, all centered horizontally:
1. The wordmark (drawn manually with two `fillText` calls in the same font — "Lee" in `--spine`, "brary" in `--ink` — immediately adjacent, matching the on-screen wordmark treatment).
2. The cover (320×452px, rounded 16px) or the same gradient+"📕" placeholder used elsewhere, scaled up.
3. Title, serif bold 54px, word-wrapped up to 3 lines with a trailing ellipsis if it would need a 4th.
4. Author, 34px, muted.
5. The status pill + ownership pill, drawn as actual rounded-rect shapes with centered text (not images) — same color rules as the on-screen pills.
6. Star row, if rated.
7. Review, if present, rendered in italic serif as a quoted line (wrap the text in curly quotes “ ”), word-wrapped up to 5 lines with ellipsis truncation.
8. Footer: small muted text, **"Shared from Leebrary"**.

### 12.2 Progress share card — 1080×1080px canvas
1. Wordmark (same treatment as above).
2. The big count number, huge serif (220px).
3. Label: "book(s) finished".
4. The period label (e.g. "March 2026", or "2026", or a date range, or "All time").
5. If chart data exists for the current period (i.e., not the Month view, which has none): the same bar chart shown on-screen, redrawn to canvas — bars, per-bar counts above non-zero bars, and axis labels below, with a thin baseline.
6. Footer: **"Shared from Leebrary"**.

### 12.3 Share/download mechanics (shared by both card types)
1. Render the canvas, then `canvas.toBlob(...)` to get a PNG blob.
2. If `navigator.canShare` exists AND returns true for `{files: [aFileWrappingThatBlob]}`, call `navigator.share({files, title:'Leebrary', text: <a one-line caption specific to what's being shared>})`. If the user completes or cancels this, stop here either way (wrap in try/catch; a thrown/rejected share is treated as "fell through to the fallback," not an error to surface).
3. Otherwise (or if the above throws), fall back: create an object URL from the blob and open the **Share modal** — a bottom sheet titled "Share", showing the image in a preview (`max-height:56vh`), a hint line **"Tap and hold the image to save it, or use the button below."**, and two footer actions: "Close" and a "Download image" link (a real `<a download>` element, `href` = the object URL, filename derived from the book title or the period label, slugified to lowercase-with-hyphens, e.g. `leebrary-the-left-hand-of-darkness.png` or `leebrary-progress-march-2026.png`).

---

## 13. Settings — Backup & Restore

Opened via the header's gear icon. Title: **"Backup & restore"**, with an explanatory paragraph:

> "Your library is saved only in this browser. To use it somewhere else — another browser, or your phone as well as a computer — export a backup file here, then import it there. Importing adds any books not already in your library; it won't duplicate ones that are."

Two full-width stacked buttons:
- **"Export backup file"** — builds `{ app: "leebrary", version: 1, exportedAt: <today's date>, books: <the full current array> }`, serializes it with 2-space indentation, wraps it in a `Blob` (`application/json`), and triggers a download named `leebrary-backup-<today's date>.json` via a programmatically-clicked, invisible `<a download>` element.
- **"Import backup file"** — opens a hidden file input (`accept="application/json,.json"`). On file selection, read it as text, `JSON.parse` it (accepting either a raw array, or an object with a `.books` array — support both shapes), and:
  - If parsing fails or the result isn't array-shaped: show the confirm/alert modal with a single "OK" button reading **"That file could not be read as a Leebrary backup."**
  - Otherwise, compute which incoming books are genuinely new (must have an `id`, a `title`, and an `id` not already present among current books). If zero are new: show a single-button notice **"All N book(s) in that file are already in your library — nothing new to import."** Otherwise show a yes/no confirmation: **"Found N book(s) in that file — X new, Y already in your library. Import the X new one(s)?"**, confirm label "Import" — on confirm, push all the new ones into the array, persist, close the settings sheet, and re-render.
- Below both, a plain **"Close"** button.

---

## 14. Custom Confirm/Alert Modal

A single, generic, reusable modal (not book- or settings-specific) that every other feature calls into. It supports two shapes:
- **Confirmation** (yes/no): a message, a "Cancel"-labeled button (customizable text, e.g. defaults to "Cancel"), and a confirm-labeled button (customizable text, e.g. "Remove" or "Import") that runs a supplied callback when tapped.
- **Notice** (single button only): pass `cancelLabel: null` to hide the cancel button entirely, leaving just one button (conventionally labeled "OK") that simply closes the modal with no callback.

Tapping the backdrop (outside the sheet) or the cancel button both simply close it without running any callback. This modal must be the ONLY mechanism used anywhere in the app for confirmations or notices — never call the browser's native `confirm()`/`alert()`.

---

## 15. Full Interaction/State Summary (for implementers building the JS)

Track these pieces of top-level state (plain variables, no external state library):
- `books`: array, the full dataset, loaded from and saved to `localStorage` on every mutation.
- `currentTab`: `"library" | "progress"` — which bottom tab was last chosen.
- `view`: `"library" | "progress" | "detail"` — what's actually rendered right now (differs from `currentTab` only while viewing a book's detail page).
- `selectedBookId`: the book currently shown in detail view, or null.
- `statusFilters`, `ownerFilters`, `ratingFilters`: `Set` instances for the multi-select filters.
- `searchTerm`: string.
- `sortDir`: `"desc" | "asc"`.
- `progressPeriod`: `"month" | "year" | "custom" | "all"`.
- `progressYear`, `progressMonth`: numbers, shared between the Month and Year progress views.
- `customFrom`, `customTo`: date strings for the Custom progress view.
- `editingId`: the id of the book currently being edited via the Add/Edit sheet, or null when adding a new one.

Tapping any bottom tab button always sets both `currentTab` and `view` to that tab (i.e., always exits detail view back to a tab). Opening a book's detail sets `view = "detail"` and `selectedBookId`, without changing `currentTab`. The detail page's "back" link sets `view = currentTab`.

---

## 16. Acceptance Checklist

An implementation is complete when all of the following hold:

- [ ] Opening the file directly (no server) works fully offline except for the two Google Fonts.
- [ ] Adding a book with only a title and author succeeds; every other field can be left untouched.
- [ ] The book tile shows exactly: cover/placeholder, title, author, one status pill, one ownership pill, and stars only if rated — nothing else, and the whole tile is tappable.
- [ ] Tapping a tile opens its detail page; Prev/Next there step through the same filtered/sorted set as the Library list; Back returns to Library.
- [ ] Marking a book done today, reopening it, and marking it done again produces TWO entries in `readHistory` and the detail page's "Read history" section lists both with correct ordinals and dates.
- [ ] A book logged with a finish date earlier than today (its add date) shows only "Finished [date]" with no "Added" text.
- [ ] Deleting a book shows the custom confirm modal (not a native browser dialog) and actually removes the book on confirm.
- [ ] Sorting toggles direction correctly even when several books share the exact same added (or done) date, thanks to the id-timestamp tiebreaker.
- [ ] Switching the quick filter to "Done" changes the sort label to "Sort by done date" automatically; every other filter state sorts by added date.
- [ ] The Filters sheet supports selecting multiple chips per group simultaneously and its "Show N books" button count updates live as chips are toggled.
- [ ] Progress → Month has a working month dropdown and shares the year arrows with the Year view; Year shows a 12-bar chart; Custom requires both dates before showing results; All time buckets by year.
- [ ] Both "Share" buttons (book detail, progress) produce a themed PNG and either open the native share sheet or the in-app preview/download modal.
- [ ] Export produces a valid, re-importable JSON file; importing it into a fresh/different browser storage merges by id without duplicating.
- [ ] The whole app fits the viewport with no page-level scroll; only the content area scrolls when a list is long.
- [ ] A home-screen "Add to Home Screen"/install shows the custom "L" monogram icon, not a generic browser icon or webpage screenshot.
