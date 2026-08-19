# KIS Primary Calendar — Documentation and Technical Handoff

**Repository:** `mfranken-ship-it/KIS-Primary-Calendar`
**Live URL:** `https://mfranken-ship-it.github.io/KIS-Primary-Calendar/`
**Maintainer:** Maarten Franken, Director of Educational Technology, KIS Almaty
**Last updated:** August 2026

---

## 1. What this is

A single-page widget that displays KIS's 8-day rotation calendar, embedded via iframe
into Google Sites "Weekly Update" pages — one Site per grade level / EY cohort.

It shows a Monday–Friday grid of the current and upcoming weeks, with each school day
labelled by its rotation day number (Day 1–8), plus which classes have PE and Library
on that day.

**The problem it solves:** parents find the 8-day rotation confusing. Google Calendar's
native embed offers only Month, Week or Agenda views — none of which show "the next two
school weeks, weekdays only, with specialist lessons". This widget does.

### Why the choices were made

| Decision | Reason |
|---|---|
| Google Sites (not PDF) | Responsive on phones; browser can machine-translate for non-English-speaking parents |
| FullCalendar reading live from Google Calendar | Single source of truth — holidays, exams and rotation shifts are handled by whoever maintains the calendar, with no duplication here |
| Rotation day → specialist mapping in static JSON | Specialist timetables are fixed for the year and not in any calendar |
| GitHub Pages | Free, zero-maintenance static hosting; the page must be publicly reachable because parents load it |

---

## 2. Architecture

```
Google Calendar (public, owned by school)
   │  all-day events titled "Day 1" … "Day 8", "Day 0: Two-Way Conferences"
   ▼
Google Calendar API  ──(API key, read-only)──▶  FullCalendar in index.html
                                                      │
                              grade JSON files ────────┤ merged at render time
                                                      ▼
                                          Google Sites <iframe> → parent's phone
```

Two independent data sources are joined in the browser:

1. **Dates → rotation day numbers** come from the Google Calendar, live.
2. **Rotation day numbers → PE/Library per class** come from a static JSON file
   chosen by the `?grade=` URL parameter.

The join key is the **rotation day number parsed out of the event title.**

---

## 3. Files in the repository

| File | Purpose |
|---|---|
| `index.html` | The entire widget — markup, CSS and JS in one file |
| `config.js` | Two constants: `API_KEY` and `CALENDAR_ID` |
| `grades.json` | Manifest of all cohorts — drives the sibling dropdown |
| `toddlers.json` | Toddlers (1 class) |
| `prek3.json` | Pre-K 3 (2 classes) |
| `prek4.json` | Pre-K 4 (2 classes) |
| `kg.json` | Kindergarten (3 classes) |
| `grade1.json` … `grade5.json` | Primary grades (3 classes each; Grade 5 has 2) |

There is no build step, no dependencies to install, and no server. Editing a file in
GitHub's web editor and committing is the entire deployment process. GitHub Pages
serves the `main` branch root and republishes within a minute or two.

---

## 4. Configuration

### `config.js`

```js
const API_KEY = '...';       // Google Cloud API key
const CALENDAR_ID = '...';   // e.g. xxxx@group.calendar.google.com
```

Loaded via `<script src>` before the main script, so both are globals.

### The API key is public by design

It is visible in page source. This is unavoidable — a browser cannot call the Google
Calendar API without shipping the key to the client. GitHub's secret scanner will flag
it; this alert can be dismissed as intentional.

**What actually protects it** is the restriction pair configured in Google Cloud Console:

- **Application restriction:** HTTP referrers → `https://mfranken-ship-it.github.io/*`
- **API restriction:** Google Calendar API only

An API key also grants **read-only** access to public data. Nobody can alter the
calendar with it.

**If the restrictions are ever missing, rotate the key** — create a new one, update
`config.js`, delete the old one in Cloud Console. Editing the file is not enough; the
old key remains in git history forever, so it must be deleted at source.

### The calendar must stay public

Anyone can read it. Keep it purpose-built for rotation days only — never put staff-only
information on it.

---

## 5. How `index.html` works

### Date range logic

- The grid always starts on **Monday of the current week**, even mid-week. This is
  deliberate: parent feedback was that they want to see the familiar Mon–Fri week with
  rotation day numbers inside it, including days already passed.
- **2 week-rows** normally; **3 rows on Thursday and Friday**, so the coming week is
  always visible by the time the Friday update goes out.
- The "Show 2 more weeks" toggle adds 2 rows on top of whichever base applies.

`visibleRange` (not the view's `duration`) drives this — `duration` is a view-level
option and does not respond to `setOption()`. **After changing `visibleRange` you must
call `calendar.refetchEvents()`**, or the new rows render empty: the plugin only
fetched events for the original range.

### Specialist rendering

`renderSpecialists()` runs on the `eventsSet` callback (wrapped in `setTimeout(…, 0)`
so FullCalendar has finished painting). For each event it:

1. Regexes `/day\s*(\d+)/i` out of the title to get the rotation day number.
2. Looks up that number in the active grade JSON's `rotation` block.
3. Injects a coloured dot + text row into the matching `.fc-daygrid-day` cell.

It removes all `.specialists` elements first, so it is safe to call repeatedly.

### Class filter

- Pills are built from the active grade's `classes` block — a cohort with 2 classes
  gets 2 pills automatically.
- Tapping a pill **isolates** that class; tapping the same pill again returns to
  showing all. Only one class can be isolated at a time.
- Once isolated, an "Also show:" dropdown appears (on its own row, via the
  `.filter-break` flex spacer) letting a parent add up to **2 classes from any other
  cohort** — for families with siblings across grades.
- Added sibling classes render in **orange (`#E65100`)** then **blue (`#0277BD`)**,
  not their own class colour, because every cohort reuses the same A/B/C palette and
  two dark-red dots would be indistinguishable.
- Selections do **not** persist across page loads. `localStorage` was deliberately
  avoided: browsers increasingly block storage in third-party iframes, so it would work
  for some parents and silently fail for others.

### Long event titles

Event titles containing a colon are split: text before the colon renders as the day
pill label, text after it renders smaller beneath (class `.day-desc`). This is how
"Day 0: Two-Way Conferences" displays. FullCalendar's `white-space: nowrap` is
overridden so long titles wrap rather than clip.

### Cell alignment

`.cell-date` has a **fixed `height: 3em`**, not `min-height`. FullCalendar injects
extra markup into certain cells (today's cell, the first of a month), which made the
day pills sit at different heights. A fixed height plus zeroed padding/margin on
`.fc-daygrid-day-top`, `.fc-daygrid-day-number` and `.fc-daygrid-day-events` means the
pills always start on the same horizontal line regardless of what FullCalendar adds.

---

## 6. Embedding in Google Sites

**Insert → Embed → "By URL"**, using the full address including the grade parameter:

```
https://mfranken-ship-it.github.io/KIS-Primary-Calendar/index.html?grade=1
```

The `grade` value matches an `id` in `grades.json` — `1`…`5`, or `toddlers`, `prek3`,
`prek4`, `kg`.

### Duplicating Sites

Embed blocks store a URL reference, so duplicating a Site copies it intact. Each
duplicate needs its `?grade=` value edited to match its cohort.

**Use the page picker, not a pasted URL, for internal links** (e.g. the "full
timetable" link). Internal references are remapped to the duplicate's own pages;
pasted URLs stay pointed at the original template, so every grade's link would lead
back to Grade 1's page.

### Known limitation: iframe height

The height you drag in Sites is the **iframe's** height, not the content's. On mobile,
Sites overrides it with its own responsive sizing while the content simultaneously
gets taller (five columns wrap heavily on a narrow screen), so parents may need to
scroll within the frame.

This cannot be fixed from inside the widget: the page is cross-origin to Sites, so it
cannot resize its parent frame, and Sites does not permit adding the `postMessage`
listener that would normally solve it. Switching to "Embed code" does not help — that
is still rendered inside a Sites-managed iframe.

If it becomes a real problem, the options are (a) a media query tightening spacing and
font sizes below ~600px, or (b) switching to a vertical list layout on phones. Option
(b) is the real fix but abandons the week-grid shape parents asked for.

---

## 7. Caching

GitHub Pages sends `Cache-Control: max-age=600` and this cannot be overridden — there
is no `.htaccess` equivalent, and `_headers` files are a Netlify/Cloudflare feature,
not a GitHub one. A stale `index.html` therefore self-corrects within ten minutes.

The real risk is a **new `index.html` pulling an old cached `config.js` or grade JSON**.
Both are therefore fetched with a version parameter:

```html
<script src="config.js?v=1"></script>
```
```js
fetch('grades.json?v=1')
fetch(file + '?v=1')
```

**Bump these numbers whenever you edit `config.js`, `grades.json` or any grade file.**
There are three places in `index.html` to change.

---

## 8. Known fragilities

Ordered by likelihood of biting.

**1. Event title format.** The regex `/day\s*(\d+)/i` needs a digit after "Day".
"Day 3", "day 3" and "Day 3: Sports Day" all work; "D3", "Rotation 3" or "Third day"
do not. If titles are renamed, specialist rows silently vanish while the grid still
looks fine. Anyone taking over the calendar needs to know this.

**2. Day numbers not in the JSON.** "Day 0" has no `rotation` entry, so it renders no
specialists. This is correct and intentional. But if a new day number were ever
introduced, it would fail the same silent way.

**3. Timetable changes mid-year.** The grade JSONs are a snapshot. Nothing detects
drift between them and the real timetables — a specialist swap in October will show
stale data until someone updates the file.

**4. Two Grade 4 entries were never confirmed** against source timetables: a "TBD"
slot (4A, day 8) and an ambiguous "Library/" cell (4B, day 6). Worth verifying.

**5. Silent failure by design.** Missing or malformed JSON produces a plain calendar
with no error message — good for parents, but it means a broken file looks like a
missing feature. **When something seems absent, open the browser console first**
(F12 → Console). A syntax error in `index.html` prevents the entire script running,
so the symptom is "everything is broken", not "one feature is broken".

---

## 9. Start-of-year checklist

1. Confirm the rotation calendar has been populated for the new academic year and is
   still public.
2. Confirm event titles still follow the `Day N` format.
3. Collect the new specialist timetables for every cohort (see
   `UPDATING-SCHEDULES.md`).
4. Regenerate every grade JSON. **Do not assume last year's data carries over** —
   rotation patterns change with staffing.
5. Update `grades.json` if cohorts have been added, removed, or renamed (e.g. a grade
   gaining a D class, or Grade 5 returning to three classes).
6. Bump the `?v=` cache-busting numbers in `index.html`.
7. Load each `?grade=` URL and check the pills, the specialists, and the sibling
   dropdown.
8. Check the Google Cloud API key restrictions are still in place.
9. Update the guiding text on each Site — **EY Sites must not mention PE uniform**,
   as PE is deliberately excluded from EY data (EY children do not change for PE).
