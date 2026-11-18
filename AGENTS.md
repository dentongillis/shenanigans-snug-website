# AGENTS.md: Shenanigan's Snug website

A guide for AI coding agents working in this repo. Read this before changing anything.

## What this is

The public website for **Shenanigan's Snug Irish Pub & Grill** in Cave Creek, Arizona, plus a preview of a staff dashboard. The owners also run Mountain View Pub in Cave Creek; the Snug's menu and photos were copied from that pub's site as a starting point.

Current state: a **static, front-end-only preview**. There is no server, database, login or payment processing. Anything staff "save" is kept in the visitor's own browser (`localStorage`), so other visitors never see it.

The audience for the code is non-developers (the owners), so user-facing copy and staff tools must stay plain and simple.

## Repo layout

```
index.html              The entire site: HTML, CSS and JavaScript in one file (~1,200 lines)
img/                    Logo (logo.png) and 20 food/bar photos (JPG, ~800px, from Mountain View Pub)
vendor/jspdf.umd.min.js jsPDF 2.5.1, vendored (used only for the Events PDF download)
favicon.png             Browser tab icon (crop of the logo shamrock)
vercel.json             Static hosting config: cleanUrls + 1-week cache on /img
README.md               Short human readme (partly out of date: see "Known issues")
```

No `package.json`, no build step, no framework, no dependencies to install.

## Running and deploying

- **Local:** serve the folder with any static server, e.g. `python3 -m http.server 8000`, then open `http://localhost:8000`. Opening `index.html` directly as a file mostly works, but use a server when testing the PDF download (it loads `vendor/jspdf.umd.min.js` on demand).
- **Deploy:** the GitHub repo `dentongillis/shenanigans-snug-website` is connected to Vercel. Every push to `main` redeploys. Vercel settings: Framework Preset "Other", no build command, no output directory.
- **Tests:** there is no test suite. Verify changes in a browser at phone width (320 to 414px), tablet (768px) and desktop (1280px+). Check that nothing scrolls sideways (`document.documentElement.scrollWidth > innerWidth` should be false) and that the console has no errors.

## How index.html is organized

Find sections by searching for their comment markers rather than by line number.

### 1. `<head>`
Meta description, Open Graph tags, favicon, and Google Fonts (**Fraunces** for display, **Figtree** for body).

### 2. `<style>` (one stylesheet)
- `:root` holds the design tokens (see "Design system").
- Sections are marked with comments such as `/* photo bands */`, `/* menu */`, `/* order page */`, `/* calendar */`, `/* forms */`.
- Responsive breakpoints in use: `1080px` (nav collapses to the burger menu), `980px` (two-column layouts stack), `860px`, `820px`, `760px` (main phone breakpoint), `560px`, `420px`.
- A global rule `[hidden]{display:none!important}` makes the `hidden` attribute always win. The JS relies on it to show and hide panels, buttons and messages.
- `prefers-reduced-motion` turns off animations and transitions.

### 3. `<body>`: shared chrome
- `#announce`: the amber announcement bar at the top (text editable from the staff Overview tab; not saved).
- `header.top`: logo, `nav#nav` (page links), "Order online" button, `#burger` (mobile menu toggle).
- `<main>` contains every page as `<div class="page" data-page="NAME">`. Only one has the class `on` at a time.
- `footer`: logo, link columns, preview disclaimer, and the "Staff sign-in" link (`#staff`).
- `.mobilebar`: sticky bottom bar on phones (Order online / Book a table, or "View your order" on the order page).
- Overlays: `#pdfModal` (PDF download dialog) and `#lightbox` (gallery photo viewer).

### 4. Pages (`data-page` values)

| Hash | Page | Notes |
| --- | --- | --- |
| `#home` | Home | Hero with logo, open-now badge, happy hour strip, House favorites cards (`#favs`), "What's on this week" (`#weekList`), feature bands |
| `#menu` | Menu | Read-only menu with tabs, section chips and photos. Rendered into `[data-tabs="menu"]`, `[data-chips="menu"]`, `[data-body="menu"]` |
| `#order` | Order online | **"Coming soon" page.** Every Order online button points here |
| `#order-preview` | Working order page | Hidden (nothing links to it). Menu with Add buttons and a cart (`#cartBox`). Re-enable ordering by pointing the Order links back here |
| `#events` | Events | Month calendar (`#pubGrid`) with day details (`#dayBox`), Calendar/List toggle, PDF download button |
| `#reservations` | Book a table | `#bookForm` |
| `#parties` | Private parties | `#partyForm` |
| `#about` | Our story | Static copy and photos |
| `#gallery` | Gallery | Photo grid (`#gal`) with lightbox |
| `#contact` | Visit | Hours table, address placeholder, map placeholder, `#msgForm` |
| `#staff` | Staff dashboard | Three tabs: Overview, Calendar, Menu (see below). **Not protected by any login** |

### 5. `<script>` (one inline script, runs at the end of the body)

The data lives at the very top: `let MENU = [...]` is **one very long line (~21,000 characters)** holding the seed menu as JSON. Edit it with a script or a precise find-and-replace, not by eye.

Sections, in order (search for these comments):

| Marker | What it does |
| --- | --- |
| (top of script) | `MENU` seed, helpers `$`, `$$`, `money`, `esc`, `slug`, the `PHOTO` map (dish name to image file), menu loading/saving (`MENU_KEY`, `saveMenu`, `itemById`) |
| `/* ---------- Router ---------- */` | `PAGES` list and `route()`: shows the page matching `location.hash`, highlights the nav link, closes the mobile menu, scrolls to top |
| `/* ---------- Arizona time ---------- */` | `azNow()` returns the current date/time in `America/Phoenix`. `status()` drives the open-now badge and the live happy hour pill (runs every minute) |
| `/* ---------- Menu ... ---------- */` | `TABS` (menu keys and their labels), `renderMenu('menu'|'order')`, `itemHTML()`, section chip row with arrow buttons (`chipState()`) |
| `/* ---------- Favorites ---------- */` | `FAVS` (six dish names shown on Home), `renderFavs()` |
| `/* ---------- Cart ---------- */` | In-memory `cart`, `addToCart`, `renderCart`, pickup times, fake checkout. `log` collects orders, bookings and party requests for the staff Overview |
| `/* ---------- Events ---------- */` | Event types (`TYPES`), events (`EVENTS`), date helpers, `gridHTML()` (shared month grid), public calendar, home "this week", staff calendar editor, event type editor, staff tab switching |
| `/* ---------- Calendar PDF ---------- */` | PDF dialog, live preview, lazy-loads jsPDF, `buildCalendarPDF(y,m)` draws a one-page landscape letter calendar |
| `/* ---------- Forms ---------- */` | Booking, party and contact forms. They show a confirmation but **send nothing** |
| `/* ---------- Gallery ---------- */` | `GAL` list of photos and captions, lightbox |
| `/* ---------- Staff dashboard ---------- */` | Overview counters and inbox (`renderDash`), announcement bar controls |
| `/* ---------- Staff: menu editor and sold-out ---------- */` | Menu tab: list, search, add/edit/delete form, photo upload, sold-out toggles, "Sold out today" lists |
| `route();` | Last line: renders the page for the current hash |

## Data model

### Menu item (`MENU` array)
```js
{
  id: 12,                         // number, unique; new items use nextItemId
  menu: "Main Menu",              // one of the TABS keys (exact string, see below)
  cat: "Shareables",              // section name; sections appear in first-seen order
  note: "",                       // optional section note, e.g. "Served with fries or potato chips"
  name: "Pub's Big Pretzel",
  desc: "Topped with garlic butter...",
  addons: "",                     // free text, e.g. "Add bacon +$3.00"
  opts: [{ label: "", price: 18 }],   // one or more prices; label is blank when there is one price,
                                      // or a size/option such as "Half Pound" / "12 Wings"
  img: "img/pretzel.jpg",         // relative path, a data: URL (staff upload), or "" for none
  out: false                      // sold out today
}
```
The three `TABS` keys are exact strings and must match `menu` values:
`'Main Menu'`, `'Happy Hour (Mon–Fri 1–6 PM)'` (note the en dashes), `'Breakfast Libations'`.

### Event (`EVENTS` array)
```js
{ id: 5, date: "2026-11-20", title: "Friday live music", time: "19:00", kind: "music", note: "Local bands, no cover." }
```
`time` is `"HH:MM"` (24-hour) or `""` for all day. `kind` is an event type id. On first load `seedEvents()` generates example weekly events (trivia Wednesdays, live music Fridays and Saturdays, football Sundays, St. Patrick's Day) from last month to 12 months ahead, relative to today.

### Event type (`TYPES` array)
```js
{ id: "music", name: "Live music", color: "#f0a93b" }
```
Defaults are in `DEFAULT_TYPES`. Staff can rename, recolor, add (ids like `t1731...`) and delete types (blocked while any event uses the type). Label text color is chosen automatically by `fgFor(hex)` for contrast.

### Where data is kept

| Data | Storage | Key | Survives reload? |
| --- | --- | --- | --- |
| Menu (incl. sold out, uploaded photos) | `localStorage` | `snug-menu-v1` | Yes, this browser only |
| Events | `localStorage` | `snug-events-v1` | Yes, this browser only |
| Event types | `localStorage` | `snug-types-v1` | Yes, this browser only |
| Cart, orders/bookings log, announcement text | memory | none | No |

Every `localStorage` read and write is wrapped in try/catch, and the site falls back to the seed data. **Important:** once a browser has saved data under a key, changes to the seed (`MENU` line, `seedEvents`, `DEFAULT_TYPES`) are not visible in that browser until staff press the Reset button or the key is bumped (e.g. `snug-menu-v2`). Uploaded photos are stored as JPEG data URLs (resized to 640px max), so `localStorage`'s ~5 MB limit can fill up; `saveMenu()` returns false and the form says so.

## Rendering pattern

There is no framework. Each feature has a `render...()` function that rebuilds its HTML with template strings and sets `innerHTML`. State changes call the relevant renderers:
- Menu changes call `menuChanged()`, which saves and re-renders the Menu page, the order page, Home favorites, the cart, the sold-out lists and the staff Menu tab.
- Event or type changes call `saveEvents()` / `saveTypes()` then `renderAllEvents()` (public calendar, home "this week", staff calendar) and `renderTypeUses()` (legends, type dropdown).

Clicks are mostly handled with **event delegation** on `document` or a container, keyed by `data-*` attributes: `data-tab`, `data-jump`, `data-chipnav`, `data-add`, `data-q`, `data-date`, `data-pmode`, `data-stab`, `data-edit`, `data-del`, `data-type`, `data-tdel`, `data-mtab`, `data-medit`, `data-mdel`, `data-mout`, `data-unsell`, `data-lb`.

Rules to keep:
- **Always pass user or staff text through `esc()`** before putting it in HTML.
- **Element ids must be unique across the whole file**, since every page is in one document. (A duplicate id between the contact form and the staff menu form once broke the menu editor.) Check with: `grep -o 'id="[^"]*"' index.html | sort | uniq -d`.
- Destructive staff actions (delete dish, reset menu) use `twoStep(button, label)`: the first tap changes the label, the second tap acts.
- The page runs in strict browser environments (including a sandboxed preview), so avoid `alert`/`confirm`/`prompt`; show messages in the page instead (`say`, `calSay`, `typeSay`, `mSay`).

## Design system

Tokens in `:root`:

| Token | Value | Use |
| --- | --- | --- |
| `--bottle` | `#0f3323` | Main dark green background |
| `--cellar` | `#092116` | Darker green: header, staff bands, cart |
| `--shamrock` | `#23a05a` | Logo green, accents, check marks |
| `--paper` | `#f4ead3` | Cream "menu paper" sections |
| `--ink` | `#16261c` | Text and buttons on paper |
| `--amber` | `#f0a93b` | Primary action buttons, announcement bar |
| `--cream` | `#f7f0df` | Text on green |
| `--muted` | `#b9c9bd` | Secondary text on green |
| `--display` / `--body` | Fraunces / Figtree | Headings / everything else |

Look and feel: an Irish pub in the Arizona desert. Dark bottle-green "rooms" alternate with cream paper bands; pages open with a photo header (`.phead`, or `.hero` on Home); the menu reads like a printed menu with dotted price leaders. Buttons are pills (`.btn-amber`, `.btn-ghost`, `.btn-ink`, `.btn-line`, size `.btn-sm`). Keep the copy plain, short and friendly, in sentence case; buttons say exactly what they do ("Book a table", "Add to calendar").

## Things that are hard-coded (update in every place)

- **Hours** (Mon to Fri 11 AM to 10 PM, Sat and Sun 8 AM to 10 PM): in `status()`, `pickupTimes()`, the booking time list (`#bTime` builder in Forms), and the hours table on `#contact`.
- **Happy hour** (Mon to Fri 1 to 6 PM): in `status()`, the Home strip (`#hhPill`), the `#contact` table and the Happy Hour menu key.
- **Address, phone, email:** placeholders ("coming soon") on `#contact` and in the footer.
- **Dish photos:** the `PHOTO` map links seed dish names to `img/*.jpg`. Home favorites come from `FAVS` (names must match a Main Menu dish).
- **Gallery:** the `GAL` list.

## Known issues and limits

- **Nothing is shared between visitors.** Staff edits only exist in the browser that made them. This is the main thing the next phase fixes.
- **The staff dashboard has no login.** Anyone can open `#staff`; changes only affect their own browser.
- **Forms and checkout send nothing.** Each shows a "This is a preview" note.
- **Example content:** hours, events, story text and contact details are placeholders until the owners confirm them. Photos belong to Mountain View Pub.
- **README.md is out of date:** it says staff changes reset on reload, but menu, events and event types now persist per browser.
- **Hash routing:** pages are `#name`, so they share one URL for SEO purposes and cannot have their own meta tags.
- On phones, long event titles wrap tightly inside the month grid; the List view is the readable alternative.

## Planned next phase (not started)

The agreed direction is to rebuild as a real app with a shared backend:
- **Next.js (App Router) on Vercel**, with the same design tokens and page structure.
- **Clerk** for invite-only staff login with roles (Owner, Manager, Staff).
- **Postgres** (e.g. Neon via the Vercel Marketplace) for the menu, events, event types, orders, reservations, party inquiries and site settings. The localStorage shapes above map directly onto tables.
- **Vercel Blob** for staff photo uploads.
- **Stripe** for online pickup orders and party deposits; email (e.g. Resend) and text (e.g. Twilio) confirmations.
- Turn `#order` back into the real ordering page once payments work.

When doing that migration, keep the public look identical and move one feature at a time (menu first, then events, then bookings, then ordering).
