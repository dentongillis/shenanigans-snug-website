# Calendar guide for AI agents: Shenanigan's Snug website

Give this file to an AI coding agent that will change anything about the events calendar on the Shenanigan's Snug website: the public Events page, the event flyers, the "What's on this week" list on Home, the printable PDF calendar, or the Calendar tab in the staff dashboard.

Read the whole file before editing. The general repo guide is `AGENTS.md` in the repo root.

---

## 1. The setup in one minute

- **Repo:** `dentongillis/shenanigans-snug-website` on GitHub, branch `main`. Vercel redeploys on every push to `main`.
- **One file holds everything:** `index.html` has all the HTML, CSS and JavaScript. There is no framework, no build step and no `package.json`.
- **No backend.** Events and event types are saved in the visitor's browser with `localStorage`. Changes staff make on one device are not seen on any other device. Do not tell the owners otherwise.
- **The PDF library** is vendored at `vendor/jspdf.umd.min.js` (jsPDF 2.5.1) and loaded only when someone downloads a PDF.
- **pdf.js** is vendored at `vendor/pdf.min.js` and `vendor/pdf.worker.min.js` (3.11.174) and loaded only when staff upload a PDF flyer.
- **Time zone:** the pub is in Arizona. `azNow()` gives the current date and time in `America/Phoenix`; `today` is set from it once when the page loads.

## 2. Where the calendar appears

| Place | What visitors or staff see | Main element ids |
| --- | --- | --- |
| Events page (`#events`) | Month grid with event titles in each day, a details panel for the selected day, a Calendar/List switch, a "Download calendar (PDF)" button, a color key | `#pubPrev`, `#pubTitle`, `#pubNext`, `[data-pmode]`, `#pdfOpen`, `#pubCalView`, `#pubGrid`, `#dayBox`, `#pubList` |
| Events page, under the day details | Event flyers: flip cards with the flyer on the front and the details on the back | `#flyers`, `#flyerGrid`, `#flyMore` |
| Home page (`#home`) | "What's on this week": the next 4 events in the coming 14 days | `#weekList` |
| PDF dialog (opens from the Events page) | Month picker, live preview, Download button | `#pdfModal`, `#pdfMonth`, `#pdfGo`, `#pdfClose`, `#pdfMsg`, `#pdfPrev`, `#pdfSheet`, `#pdfPrevTitle`, `#pdfPrevGrid` |
| Downloaded PDF | One landscape US-letter page: logo, month, grid, events with time, title and details, color key, pub name | built in `buildCalendarPDF()` |
| Staff dashboard, Calendar tab (`#staff`, then the Calendar tab) | Month grid, selected day's events with Edit/Delete, the add/edit form, the Event types panel, a Reset button | `#stPrev`, `#stTitle`, `#stNext`, `#stGrid`, `#stSide`, `#stDayTitle`, `#stDayList`, `#evForm`, `#typeList`, `#typeAdd`, `#evReset`, and the flyer manager `#flyAdmin`, `#flyForm`, `#flyList` |

Every color key on the site is a `<div class="legend" data-legend></div>`, filled in by JavaScript.

## 3. Finding the code in index.html

Search for these markers instead of using line numbers (line numbers change).

**HTML**
- Events page: `<div class="page" data-page="events">`
- Home list: `<div class="week" id="weekList">`
- Staff Calendar tab: `<div data-stabp="calendar" hidden>`. The tab button is `data-stab="calendar"`, in the `.tabs.dark` row at the top of the staff page.
- PDF dialog: `<div class="modal" id="pdfModal"`, near the end of `<body>`.

**CSS** (all inside the one `<style>` block)
- Shared grid and day cells: `/* calendar */`, then rules for `.cal-top`, `.navb`, `.cal-layout`, `.grid`, `.day`, `.grid.big .day`, `.day .num`, `.day .evs`, `.evt`
- Events page only: rules starting `.pubcal`, plus `.seg` (the Calendar/List switch), `.monthlist`, `.calcta`, `.daybox`, `.legend`, `.mk` (the colored dot)
- Staff calendar: `.staff-cal`, `#stSide`, `.types-col`, `.dayevs`, `.types`, `.mini`, `.mini.del`, `.tabs.dark`
- PDF dialog: `.modal`, `.pdfprev`, `.pdfprev .sheet`
- Home list: `.week`, `.ev`, `.ev .date`
- Phone layout: the `@media (max-width:760px)` block that contains `.pubcal .evt` and `.staff-cal .evt`
- Leftover, unused: `.k-music`, `.k-trivia`, `.k-sport`, `.k-special`, `.e-music` and similar, and `.day .marks`. Colors now come from inline styles set by the event types. These rules can be deleted.

**JavaScript** (one inline `<script>` at the end of `<body>`)
- Everything calendar-related runs from `/* ---------- Events ---------- */` up to `/* ---------- Forms ---------- */`. The PDF part starts at `/* ---------- Calendar PDF ---------- */`.
- `azNow()` is in `/* ---------- Arizona time ---------- */`.
- Shared helpers at the top of the script: `$` (querySelector), `$$` (querySelectorAll as an array), `esc()` (HTML escaping).

## 4. Data

### Event
```js
{
  id: 5,                  // unique number; new events take nextId++
  date: "2026-11-20",     // YYYY-MM-DD, built with dkey(y, m, d) where m is 0-based
  title: "Friday live music",
  time: "19:00",          // 24-hour "HH:MM", or "" for an all-day event
  kind: "music",          // id of an event type
  note: "Local bands, no cover."   // "Details shown to guests"; may be ""
}
```
All events live in one array, `EVENTS`. There is no recurring-rule object: "Repeats every week" makes separate copies, one per week, and editing one copy does not change the others.

### Event type
```js
{ id: "music", name: "Live music", color: "#f0a93b" }
```
All types live in `TYPES`. The defaults are in `DEFAULT_TYPES`: `music` (Live music, #f0a93b), `trivia` (Trivia, #7cc7ff), `sport` (Game day, #b9f27c), `special` (Special event, #ff7a8a). Types staff add get ids like `t1731958000000`.

### Saving and loading

| Key in localStorage | Holds | Saved by | Loaded |
| --- | --- | --- | --- |
| `snug-events-v2` (`EV_KEY`) | `EVENTS` | `saveEvents()` | on page load; falls back to `seedEvents()` |
| `snug-flyers-v1` (`FLY_KEY`) | `FLYERS` | `saveFlyers()` | on page load; falls back to `FLYER_SEED` |
| `snug-types-v1` (`TYPE_KEY`) | `TYPES` | `saveTypes()` | on page load; falls back to `DEFAULT_TYPES` |

All reads and writes are wrapped in try/catch, so the page still works when storage is blocked.

**Important:** once a browser has saved events or types, it ignores the seed data. If you change `seedEvents()` or `DEFAULT_TYPES` and want every browser to pick up the change, either tell staff to press "Reset to the example events and types", or bump the key (for example `snug-events-v3`). Bumping the key throws away events staff added in that browser, so say so to the user first.

### Real events: `REAL_EVENTS`
Confirmed events are listed by month, keyed `'YYYY-MM'`, in the `REAL_EVENTS` object just above `seedEvents()`. Each entry has the event shape above without `id`. A month listed there shows only its real events; the example weekly events are skipped for that month. To publish a new month for every visitor, add a key with its events and bump `EV_KEY`.

### Example events: `seedEvents()`
Runs when no saved events exist. It loops over every day from the first of last month to the end of the month 12 months ahead, relative to `today`, and adds:
- Wednesday: "Pub trivia night", 19:00, `trivia`
- Friday: "Friday live music", 19:00, `music`
- Saturday: "Saturday live music", 19:00, `music`
- Sunday: "Sunday football", 11:00, `sport`
- March 17: "St. Patrick's Day party", all day, `special`

To change the starting schedule, edit this function. Days of the week use JavaScript's numbering (0 = Sunday, 6 = Saturday).

## 5. Functions and what they do

**Lookups and formatting**
| Function | Purpose |
| --- | --- |
| `dkey(y, m, d)` | Builds `"YYYY-MM-DD"` (month `m` is 0-based) |
| `pad(n)` | Two-digit zero padding |
| `fmtTime("19:00")` | Returns "7 PM", "6:30 PM", or "All day" for "" |
| `niceKey("2026-11-20")` | Returns "Fri, November 20" |
| `eventsOnKey(key)` / `eventsOn(y, m, d)` | Events on one day, sorted by start time (all-day first) |
| `tget(id)`, `tname(id)`, `tcolor(id)` | Type lookup; an unknown id falls back to "Event" in grey |
| `rgb(hex)`, `fgFor(hex)` | Color helpers; `fgFor` picks dark or white text for contrast |
| `tstyle(id)` | Inline style string (background and text color) for an event label |
| `shiftMonth(view, n)` | Moves a `{y, m}` view object forward or back by n months |
| `MONTHS`, `DOW` | Month names and 3-letter day names |

**Shared month grid: `gridHTML(view, selectedKey, interactive)`**
Returns the HTML for a 7-column month: day-name headers, blank cells before the 1st, then one cell per day with the day number and a label for each event (`<span class="evt">` with **bold time** and title). Today gets the class `today`. With `interactive` true each day is a `<button data-date="YYYY-MM-DD">` and the selected day gets `aria-pressed="true"`; otherwise each day is a plain `<div>`. Used by the Events page, the staff calendar and the PDF preview, so a change here shows in all three.

**Events page**
| Name | Purpose |
| --- | --- |
| `pview` | Month being shown, `{y, m}` |
| `psel` | Selected day key (starts as today) |
| `pmode` | `'cal'` or `'list'` |
| `renderPubCal()` | Draws the title, grid, day details and the month list |
| `renderDayBox()` | Fills `#dayBox` with the selected day's full details and a "Book a table for this day" button. If the selected day is in another month it shows "Pick a day..." |
| `#pubGrid` click handler | Sets `psel`; on screens 980px or narrower it scrolls to the details |
| `[data-pmode]` click handler | Switches between `#pubCalView` and `#pubList` |

**Home**
- `renderWeek()`: the next 4 events within 14 days of today, into `#weekList`.

**Staff Calendar tab**
| Name | Purpose |
| --- | --- |
| `sview`, `ssel` | Month shown and selected day |
| `editing` | Id of the event being edited, or `null` when adding |
| `renderStaffCal()` | Draws the grid and the selected day's list with Edit/Delete buttons, and switches the form between "Add an event" and "Edit this event" (title, button text, Cancel button, Repeats field) |
| `fillForm(event or null)` | Loads an event into the form, or resets it for a new one (date = selected day, time = 19:00, type = first type) |
| `#stGrid` click | Selects a day and resets the form |
| `#stDayList` click | `data-edit` loads that event into the form; `data-del` deletes it immediately (there is no confirm step for events) |
| `#evForm` submit | Adds (with optional weekly repeats) or saves edits, saves, jumps the staff view to that date, and re-renders everything |
| `#evCancel` | Leaves edit mode |
| `#evReset` | Restores `DEFAULT_TYPES` and `seedEvents()` and clears both storage keys |
| `calSay(text)` | Shows a message in `#calOk` |

**Event types panel**
| Name | Purpose |
| --- | --- |
| `renderTypeList()` | Draws one row per type: color picker, name box, Delete |
| `renderTypeUses()` | Refreshes every `[data-legend]` color key and the Type dropdown `#eKind` |
| `#typeList` input | Live rename or recolor; saves and re-renders |
| `#typeList` click on `data-tdel` | Deletes a type, but refuses if any event uses it or if it is the last type |
| `#typeAdd` | Adds "New type" with the next color from a small palette and focuses its name |
| `typeSay(text)` | Shows a message in `#typeOk` |

**Re-render everything:** `renderAllEvents()` calls `renderPubCal()`, `renderWeek()` and `renderStaffCal()`. After changing `EVENTS` or `TYPES`, call the matching save function and then `renderAllEvents()`, plus `renderTypeUses()` when types changed.

**Staff tab switching:** a document click handler on `[data-stab]` sets `aria-selected` on the tab buttons and toggles `hidden` on the `[data-stabp]` panels.

**Start-up order** (at the end of the Events section): `renderTypeList(); renderTypeUses(); fillForm(null); renderAllEvents();`

## 6. Event flyers

Staff upload a flyer (photo or PDF) with a title, date, start time and details. On the Events page it appears as a card in the right-hand column under the day details. Hovering (mouse) or tapping (touch) flips the card to show the details, a "Book a table" button (it also pre-fills the booking date) and a "See the day" button (jumps the calendar to that date).

**Flyer shape**
```js
{ id: "f1731958000000", img: "data:image/jpeg;base64,...", title: "Rhonda Street Band", date: "2026-11-21", time: "17:00", note: "Until 9 PM on the patio." }
```
`img` is a JPEG data URL for staff uploads (resized to 1,000px on the long side) or a path like `img/flyers/rhonda.jpg` for flyers that ship with the site. Flyers are separate from `EVENTS`: adding a flyer does not add a calendar event.

**Where the code is**
- HTML: `<section class="flyers" id="flyers">` inside `.cal-side` on the Events page; `<div class="panel fly-admin" id="flyAdmin">` at the bottom of the staff Calendar tab.
- CSS: `/* event flyers: flip cards */` and `/* staff flyer manager */`.
- JavaScript: `/* ---------- Event flyers ---------- */`.

**How it works**
| Name | Purpose |
| --- | --- |
| `FLYER_SEED` | Flyers that ship with the website and show for every visitor. Put the image in `img/flyers/` and add an entry (example in the code comment) |
| `FLYERS`, `saveFlyers()` | All flyers; saved per browser under `snug-flyers-v1`. `saveFlyers()` returns false when storage is full |
| `upcomingFlyers()` | Flyers dated today or later, sorted by date and time. Past flyers never show publicly |
| `renderFlyers()` | Draws the cards. Shows 2 at a time in the desktop sidebar (wider than 980px) and 4 on smaller screens, with a "Show all N flyers" button. Hides the section when there are no upcoming flyers |
| `flyerCard(f)` | HTML for one card: `.flyer` > `.flyer-inner` > `.ffront` (the flyer over a blurred copy of itself, date pill, flip icon) and `.fback` (calendar block, title, date and time, details, buttons, faint logo watermark) |
| Flip behaviour | CSS 3D flip (`rotateY(180deg)`, 0.75s ease). On mouse devices it flips on hover (`@media (hover:hover) and (pointer:fine)`); on touch devices a tap toggles `.flipped`; Enter or Space toggles it from the keyboard; focusing a button on the back keeps it flipped. Reduced-motion users get a fade instead of a flip |
| `renderFlyAdmin()` | Staff list with thumbnails, a Past tag, Edit and Delete (two taps), and "Remove past flyers" |
| `fillFlyForm(f)` | Loads a flyer into the staff form, or resets it (date defaults to the selected staff calendar day) |
| `pdfFirstPage(file)` | Lazy-loads pdf.js, renders page 1 of a PDF to a canvas, returns a JPEG data URL |
| `toJpeg(source)` | Resizes an image or canvas and returns a JPEG data URL |

**Watch out for**
- Each flyer adds roughly 50 to 250 KB to browser storage, which is limited to about 5 MB and shared with the menu and events. The form warns when it is full; "Remove past flyers" frees space.
- Card text sits on the paper-colored back, so long details scroll inside `.fb-note`. Keep the height math out of it: the card is a fixed 4:5 shape.
- Test hover on a desktop browser and tap on a real phone; desktop browsers emulating touch don't always switch the hint text.

## 7. The PDF

| Name | Purpose |
| --- | --- |
| `pdfv` | Month chosen in the dialog |
| `#pdfMonth` options | This month plus the next 11 (change `length:12` to offer more) |
| `#pdfOpen` click | Opens the dialog on the month currently shown on the Events page (if it is in the list) |
| `renderPdfPrev()` | Fills the preview using `gridHTML(pdfv, null, false)` |
| `fitPdfPrev()` | Scales the 700px-wide preview sheet to fit the dialog |
| `loadJsPDF()` | Loads `vendor/jspdf.umd.min.js` once |
| `buildCalendarPDF(y, m)` | Draws the PDF and returns the jsPDF document |
| `#pdfGo` click | Builds and saves `Shenanigans-Snug-Events-<Month>-<Year>.pdf` and shows a message in `#pdfMsg` |
| `closePdf()` | Closes the dialog (also on Escape or a click outside) |

How `buildCalendarPDF` lays out the page (units are points, page 792 x 612, margin 36):
1. Logo (taken from the header `<img>`) top left, month and year top right, "Events calendar" under it, a green rule.
2. A dark green bar with full day names.
3. One cell per day; days outside the month are shaded.
4. In each day, events are drawn as colored rounded boxes in their type color: bold "time  title", and the details in smaller text underneath **only if every event that day fits with details**. If not, the day shows time and title only, and if even that overflows the last line says "+N more".
5. Footer: a color key for the types (stops before it would collide with the pub name) and "Shenanigan's Snug Irish Pub & Grill, Cave Creek, Arizona".

Font sizes: day number 10, event title 7.5, details 6.5, footer 9. jsPDF only has its built-in fonts here (helvetica, times), so the site's web fonts are not used in the PDF.

Downloads are blocked inside the Claude artifact preview. Test the PDF on the Vercel site or a local server.

## 8. Phone and desktop behavior

- At 760px and narrower (phones), Events page labels drop the time and show the title only in very small text; the selected day's details appear below the grid; the Calendar/List switch fills the width.
- At 980px and narrower, the details panel moves under the grid (Events page) and the staff day editor moves under the staff grid.
- On phones the staff grid shows colored dots instead of text (`.staff-cal .evt` rule), because the editor below shows the full details.
- The PDF preview always shows text, scaled down.
- After any change, check widths 320, 375, 414, 768, 1024 and 1440. Nothing may scroll sideways: `document.documentElement.scrollWidth > innerWidth` must be false.

## 9. Recipes for common changes

**Change the example schedule**
Edit `seedEvents()`. Remember existing browsers keep their saved events (see section 4).

**Change the default event types or their colors**
Edit `DEFAULT_TYPES`. Keep the ids `music`, `trivia`, `sport`, `special` unless you also update `seedEvents()`, which refers to them.

**Add a new field to events** (example: an end time, a cover charge or a ticket link)
1. HTML: add a labelled input inside `#evForm`, with a unique id (search the whole file for the id first; every page shares one document).
2. `fillForm()`: set the new input from the event, and a sensible default for new events.
3. `#evForm` submit handler: add the field to the `data` object.
4. Show it where it belongs: `renderDayBox()` (Events page details), the list built in `renderPubCal()` (List view), `renderStaffCal()` (staff day list), `renderWeek()` (Home) if relevant, and optionally `gridHTML()` (it appears in every grid, so keep it short).
5. PDF: add it to the text measured and drawn in the `blocks` code of `buildCalendarPDF()`. Keep the height calculation (`hOf`) in step with what is drawn, or boxes will overlap.
6. Old saved events will not have the field. Treat a missing value as empty.
7. Pass the value through `esc()` before putting it in HTML. For links, only accept `http://` or `https://` URLs.

**Change the repeat options**
Edit the `<option>` values in `#eRepeat` (the value is the number of weeks). For other patterns such as every other week or monthly, change the loop in the submit handler; it currently adds `7*i` days.

**Add a confirm step to deleting events**
The staff Menu tab already has `twoStep(button, label)`. Use it in the `data-del` branch of the `#stDayList` click handler: `if(!twoStep(dl,'Tap again to delete')) return;`. Do not use `confirm()`; dialogs are blocked in some places the site runs.

**Start the week on Monday**
`gridHTML()` uses `getDay()` (0 = Sunday) for blank cells and `DOW` for headers; `buildCalendarPDF()` has its own day-name list and the same `first` calculation. Change both places together.

**Show more or fewer events on Home**
`renderWeek()`: `i<14` is the number of days searched and `out.length<4` the number of events.

**Change calendar colors or sizes**
Use the CSS listed in section 3. Desktop cell height is `.pubcal .grid.big .day{min-height:104px}`; phone cell height is in the 760px block. Label size is `.evt`. Keep text readable on every type color; `fgFor()` handles text color automatically.

**Change the PDF look**
Edit `buildCalendarPDF()`. Brand colors as RGB: ink `[22,38,28]`, dark green bar `[15,51,35]`, shamrock rule `[35,160,90]`, muted text `[76,91,80]`, shaded cells `[244,240,230]`.

**Move the calendar to a real backend later**
Replace the bodies of `saveEvents()`, `saveTypes()` and the two `localStorage` loads with API calls, keep the same object shapes, and keep calling `renderAllEvents()` after each change. Staff access must then sit behind a login (planned: Clerk). Nothing else in the rendering code needs to change.

## 10. Rules that prevent bugs

- **Unique ids.** All pages live in one HTML document. Check for duplicates after editing: `grep -o 'id="[^"]*"' index.html | sort | uniq -d` must print nothing.
- **Escape text.** Run titles, notes and type names through `esc()` before inserting them into HTML.
- **Use `hidden` to show and hide.** A global CSS rule `[hidden]{display:none!important}` makes it reliable. Don't toggle `style.display`.
- **No `alert`, `confirm` or `prompt`.** Show messages with `calSay`, `typeSay` or in `#pdfMsg`.
- **Dates are strings.** Build them with `dkey()` and compare strings. Don't store `Date` objects in `EVENTS`.
- **Keep saving and re-rendering together.** Every change to `EVENTS` or `TYPES` must save and then re-render, or the page and storage drift apart.
- **Don't break the other tabs.** The staff page also has Overview and Menu tabs that share the `[data-stab]` switching.
- **Plain language.** The owners are not technical. Messages say what happened ("Trivia night added to Wed, November 18."), and buttons say what they do.

## 11. Testing checklist

Serve the folder (`python3 -m http.server 8000`) and check, at phone and desktop widths:

- [ ] The Events page opens on the current month in Calendar view, with titles in the day cells
- [ ] Tapping a day shows its full details; the arrows change month; the List switch shows every event in the month
- [ ] Staff Calendar tab: add an event, add a weekly repeating event, edit one, delete one; each change shows straight away on the Events page and Home
- [ ] Event types: rename, recolor, add, and try deleting a type that is in use (it should refuse)
- [ ] Reload the page: events and types are still there
- [ ] "Reset to the example events and types" restores the defaults
- [ ] Event flyers: upload a JPG and a PDF in the staff Calendar tab, check they flip on hover (desktop) and tap (phone), and that past flyers don't show
- [ ] The PDF dialog preview matches the month picked, and the downloaded PDF has the logo, every event's time and title, and the color key
- [ ] No sideways scrolling at any width and no errors in the browser console

## 12. Publishing a change

Commit with a short, plain message describing what changed for the pub, then push to `main`:

```
git add index.html
git commit -m "Calendar: <what changed>"
git push origin main
```

Vercel redeploys automatically within a minute or two. Remind the owners that calendar changes made in the staff dashboard are still saved only on the device they were made on.
