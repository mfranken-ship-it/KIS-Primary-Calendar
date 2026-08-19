# Updating the schedule JSON files

Paste everything below into a fresh LLM conversation, then attach the timetable
screenshots for one cohort at a time.

---

## LLM Prompt

````
I maintain a calendar widget for our school that shows which classes have PE and
Library on each day of our 8-day rotation. I need you to produce JSON data files
from class timetables, which I'll attach as screenshots — one cohort at a time.

**Read only PE and Library from the timetables. Ignore every other subject**
(Music, Art, Kazakh, Russian, UOI, Literacy, Numeracy, and so on).

**For Early Years cohorts — Toddlers, Pre-K 3, Pre-K 4, KG — record Library only
and omit PE entirely**, even where PE appears on the timetable. EY children do not
change into PE kit, so parents don't need it, and including it adds noise.

### Output format

One file per cohort:

```json
{
  "classes": {
    "A": { "label": "1A", "colour": "#8E2323" },
    "B": { "label": "1B", "colour": "#1B5E20" },
    "C": { "label": "1C", "colour": "#4A148C" }
  },
  "rotation": {
    "1": { "A": ["Library"], "B": ["Library"], "C": ["Library"] },
    "2": { "A": ["PE"],      "B": ["PE"],      "C": ["PE"] },
    "3": { "A": [],          "B": ["Library"], "C": [] },
    "4": { "A": ["PE"],      "B": ["PE"],      "C": ["PE"] },
    "5": { "A": ["Library"], "B": [],          "C": ["Library"] },
    "6": { "A": ["PE"],      "B": ["PE"],      "C": ["PE"] },
    "7": { "A": [],          "B": [],          "C": [] },
    "8": { "A": ["PE"],      "B": ["PE"],      "C": ["PE"] }
  }
}
```

### Rules

- Every rotation day 1–8 must appear, even when no class has anything: use `[]`.
- A class with both on one day gets both: `["PE", "Library"]`.
- Subject names are exactly `"PE"` and `"Library"` — this text is shown to parents.
- Class keys are `"A"`, `"B"`, `"C"` (or `"T"` for a single-class cohort).
  Include only the classes that exist — Grade 5 has just A and B.
- Colours are always the same by position: A = `#8E2323`, B = `#1B5E20`,
  C = `#4A148C`. Do not vary these between cohorts.
- `label` is what parents see. Primary grades use the short form (`1A`, `5B`).
  Early Years cohorts use the full form (`Pre-K 3A`, `KG B`, `Toddlers`), because
  all EY classes share one dropdown heading and `3A` would be ambiguous against
  Grade 3A.
- Valid JSON only — no trailing commas, no comments.

### After each cohort, tell me:

1. Which classes are identical to each other, and which is the outlier.
2. Any rotation day where no class has anything.
3. Any cell you couldn't read confidently, or that looked provisional
  ("TBD", a trailing slash, an ambiguous merged cell) — I'll verify those against
  the source rather than have you guess.

These summaries matter: last year a genuine reading error was caught this way, and
an apparent pattern across four grades ("PE is always on even days") turned out to
be false once Grade 5 arrived. Don't state cross-cohort patterns as fact until
every cohort is in.
````

---

## File names

The name must match `grades.json` — either `grade<id>.json`, or whatever the manifest
entry's `file` key specifies.

| Cohort | File | `?grade=` value |
|---|---|---|
| Toddlers | `toddlers.json` | `toddlers` |
| Pre-K 3 | `prek3.json` | `prek3` |
| Pre-K 4 | `prek4.json` | `prek4` |
| KG | `kg.json` | `kg` |
| Grades 1–5 | `grade1.json` … `grade5.json` | `1` … `5` |

## If cohorts change

Edit `grades.json`. Order in this file is the order in the dropdown — Early Years
first, then primary.

```json
{ "id": "prek3", "label": "Pre-K 3", "file": "prek3.json", "group": "Early Years" },
{ "id": "1", "label": "Grade 1" }
```

- `file` is optional; without it the widget looks for `grade<id>.json`.
- `group` is optional; entries sharing a value are merged under one dropdown heading.
  Without it, the entry's own `label` becomes the heading.
- Only list a cohort once its JSON file exists. The widget skips missing files
  silently, so listing one early leaves you unable to tell "not built yet" from
  "broken".

## Finally

Bump the `?v=` numbers in `index.html` (three places) so parents don't get served
cached copies of the old data.
