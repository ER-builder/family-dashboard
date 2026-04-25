# Family Dashboard — UX Redesign Planning Doc

**Status:** decision pending. **Audience:** Elul + family. **Scope:** dashboard layout & information architecture only — no backend/proxy/kiosk-app changes.

**Revision history:**
- v1 (initial 4 options): missed the Stars card (it was added by the parallel Pi session mid-planning) and treated Transit as already-handled. Did not consider redesigning the left column / calendar at all.
- v2 (current): Stars + Transit folded into each option's accounting; new Option E added that touches the calendar; explicit explanation for why the calendar was originally left alone.

---

## Why we're redesigning

The dashboard has accumulated five cards in the right column on top of the calendar in the left. At 1280×800 (Portal screen), there isn't enough vertical space for all of them to coexist. The CSS uses `flex-shrink: 0` on every card except `.todos` — which has `flex: 1` and absorbs all the overflow by shrinking to ~zero height. **Result: the To-Do card is unreachable any time a routine card is active** (06:00–08:59 weekdays, 15:30–20:29 every day) and badly squeezed even outside those windows.

This is a structural problem, not a tweak.

### The constraint conflict, in numbers

At 1280×800, after masthead + padding + gaps the right column has **~636px of vertical room**. Card heights when present:

| Card | Height | Can shrink? | Visibility |
| --- | --- | --- | --- |
| Weather | ~130px | no | always |
| Morning routine (9 items × 2 kids) | ~430px | no | weekdays 06:00–08:59 |
| Evening routine (10 items × 2 kids) | ~470px | no | every day 15:30–20:29 |
| Transit (Northern + Bus 102) | ~90px | no | hidden during routine windows |
| Stars (Table Stars mirror, per-kid prizes + cycle pips) | ~90px | no | hidden during routine windows |
| To-Do | absorbs leftover | yes, collapses to nothing | always present (but invisible when crushed) |

**Three real states today** (now including Stars):

```
WEEKDAY 06–09 (morning):
  weather 130 + gap 18 + morning 440 + (transit + stars hidden) + todos = ?
  → todos gets ~30px → unreachable

EVERY DAY 15:30–20:30 (evening):
  weather 130 + gap 18 + evening 470 + (transit + stars hidden) + todos = ?
  → todos gets ~0px → completely gone

DAYTIME / WEEKEND MORNING:
  weather 130 + gap 18 + transit 90 + gap 18 + stars 90 + gap 18 + todos = 374px used
  → todos gets ~262px → reachable, but tighter than v1 estimate (Stars added ~110px to off-routine accounting)
```

We've already sacrificed weather detail (Hi/Lo/Feels removed) for routine fit. Further compression hits the ≥40px kid-touch floor (AGENTS.md hard constraint). The only way out is structural.

---

## Why the calendar (left column) was originally left untouched

Honest reasoning behind the v1 scope:

- The unreachable-tasks problem is **entirely on the right column.** The calendar isn't part of it — its `.body { overflow-y: auto }` already handles its own overflow, and it sits in its own grid column unaffected by what happens next door.
- Touching the calendar would expand the redesign blast radius without solving the actual bug. "Smallest fix that solves the problem" pulled my attention right.
- The calendar IS the canonical "what's happening today" view. Shrinking it has costs (events get truncated, hour rows get tighter, future days disappear) that aren't obviously worth the gains for the right column.

**But you're right that real "redesign" should ask the bigger question:** does the calendar deserve 39% of the screen, or could it shrink/move to give the right side breathing room? That's now Option E below.

## Project context (so your future-self remembers)

- **Family-only project**, not for distribution. Discovery / onboarding is not a concern — Elul can teach the family any new gesture once.
- **Always-on kitchen dashboard**, viewed across the room. Glanceability is the prime UX value.
- **Kids are the heaviest interactors** with routines. They tap items in sequence each morning + evening.
- **To-Do is dual-purpose**: glanceable (parents see what's left) AND interactive (mark items done, add new ones from the dashboard or from phones via the linked Google Sheet).
- **Calendar is read-only** in the left column.

---

## Five design options

Each option has a one-line idea, an ASCII sketch, and honest pros/cons. **No recommendation buried inside — see "What I'd pick" at the bottom.**

**How Stars + Transit fit under each option** (called out explicitly per the v2 update):

| Option | Stars | Transit |
| --- | --- | --- |
| A — Scheduling | Joins the time-conditional rotation. Visible during commute hours + late-day "look back at what kids earned today." Hidden during routine windows like today. | Same as today: hidden during routines, visible otherwise. Rules unchanged. |
| B — Swipe pages | Both stay on Page 1 (right column). Page 2 = full-screen To-Do. No change to Stars/Transit visibility. | Same. |
| C — Scroll | Both join the scrollable stack. End up below the fold during routines just like today. | Same. |
| D — Hybrid | **Stars + Transit fold into the top "context strip"** as small inline elements: weather emoji+temp, transit one-line, stars-mini (`🎁 Eitan 4 · Tamar 2`). All glanceable from one row. | Same — strip element. |
| E — Rebalance left/right | Both keep their current cards, just with more horizontal room. Stars could split into 2 columns (Eitan / Tamar side-by-side instead of stacked). | Same. |

---

### Option A — Smarter time-windowed scheduling

> Single screen, content rotates with the time of day. The dashboard always shows the smallest set of cards that fit.

```
06–09 weekdays:   Weather (compact)  ·  Morning routine
15:30–20:30:                            Evening routine          ·  todo strip in masthead
Daytime weekday:  Weather  ·  Transit  ·  To-Do (full card)
Weekend morning:  Weather  ·  To-Do (full card, big)
Late night:       Weather  ·  Quotes / family fact / off-duty
```

Tasks become a **single-line ticker in the masthead** during routine windows — visible, taps to expand to a full overlay. Outside routine windows, the full To-Do card returns.

**Pros**
- Stays one-screen, glanceable, no swipe gesture for kids to fight
- Every card gets the space it actually needs
- Feels "intelligent" — Terry knows what matters right now
- To-Do still reachable as a strip during routines

**Cons**
- Cards "appear and disappear" through the day — can be disorienting first week
- Most complex visibility logic (4–5 distinct states instead of 2)
- Masthead-strip overlay is new UI we haven't built
- Late-night "off-duty" mode is creative scope expansion

**Implementation footprint:** medium — new state machine in `updateRoutineVisibility()`, new strip + overlay HTML/CSS, no JS framework changes.

---

### Option B — Swipe between two pages

> Add horizontal swipe + page dots. Page 1 = current dashboard. Page 2 = full-screen To-Do.

```
PAGE 1 (default):                     PAGE 2 (swipe left):
┌──────────────────────────────┐      ┌──────────────────────────────┐
│  Masthead w/ clock           │      │  ← Family To-Do              │
├────────────┬─────────────────┤      │                              │
│  Calendar  │  Weather        │      │  [+ add task]                │
│            │  Routine/Transit│      │                              │
│            │                 │      │  ☐ Buy milk                  │
│            │  • • (page dots)│      │  ☐ Call school               │
└────────────┴─────────────────┘      │  ☑ Pay gas bill              │
                                      └──────────────────────────────┘
```

**Pros**
- To-Do is GUARANTEED reachable, full size, comfortable
- Each page optimized for its job — no constraint conflicts
- Future-extensible: page 3 = photos, page 4 = recipes, page 5 = something else
- Discovery isn't a concern in this family project

**Cons**
- Old Portal Chromium touch handling is unproven for swipe — laggy/missed swipe risk
- If kids accidentally swipe during routine, they lose their place
- Adds touch-event JS complexity (~30 lines + state mgmt)
- The "look once and see everything" magic dilutes — info is now scattered across two pages
- Tap-to-add-task during cooking requires swipe → tap → type → swipe-back

**Implementation footprint:** medium-large — touch handling, page state, transitions, CSS slide keyframes.

---

### Option C — Vertical scroll on the right column

> Add `overflow-y: auto` to `.right`. Cards below the fold get reached by scrolling that column.

```
┌────────────┬─────────────────┐
│  Calendar  │  Weather        │
│            │  Routine        │
│            │  Transit        │
│            │  ─ scroll ─     │  ← scrollbar appears when overflowing
│            │  To-Do          │
└────────────┴─────────────────┘
```

**Pros**
- ~5 lines of CSS, ships in 10 minutes
- No new gestures, no new components
- Easily reversible

**Cons**
- Family doesn't think to scroll a kitchen display — UX antipattern for ambient screens
- Scrollbars on a kitchen display look like an unfinished web app
- Active routine cards still push To-Do below the fold every morning + evening — same "unreachable" problem with scroll-required-to-find
- **Doesn't solve the real problem — just shifts it** from invisible to "scroll to find"

**Implementation footprint:** trivial. But probably the wrong instinct.

---

### Option D — Hybrid: sticky context strip + rotating primary panel

> Restructure the right column into two regions: a thin always-visible context strip on top, and a big always-visible primary panel below that swaps content based on the time.

- **Top "context strip"** (~120px): time-aware. Shows weather + (during routines) routine preview + (otherwise) transit. ONE compact horizontal row, never two.
- **Bottom "primary panel"** (~500px): the dominant task of the moment.
  - Routine windows → big routine checklist (full kids card)
  - Otherwise → big To-Do
  - Late evening → ambient (rotating quote, family fact)

```
DURING MORNING ROUTINE:                OUTSIDE WINDOWS:
┌─ context strip ─────────────┐       ┌─ context strip ──────────┐
│ Weather  ·  10°  ·  ☀ 06:30  │       │ Weather  ·  10°  ·  Bus 5m│
└──────────────────────────────┘       └────────────────────────────┘
┌─ primary panel ─────────────┐       ┌─ primary panel ──────────┐
│  ☀ Morning Routine          │       │  ✎ Family To-Do          │
│  Eitan │ Tamar              │       │  [add a task...]         │
│  ☐ Breakfast    ☐ Breakfast │       │  ☐ Buy milk              │
│  ☐ Get dressed  ☐ Get dressed│      │  ☐ Doctor appointment    │
│  ...                        │       │  ...                     │
└─────────────────────────────┘       └────────────────────────────┘
```

To-Do becomes the DEFAULT primary panel — visible most of the day. Routines take it over only during their windows.

**Pros**
- Single screen, no gestures — "look once and see everything" preserved
- To-Do is the default and dominant — most visible card by default
- Routines get the space they need without crowding
- Weather + transit compressed to a horizontal strip (they're small-info anyway)
- Cleanest mental model: "context up top, what-to-do-now in the body"
- Walk past Terry making coffee, see the next 3 todos at a glance — no tapping

**Cons**
- Biggest restructure — touches the most code
- Loses the airy feel of separate cards — denser layout
- Compressing transit + weather into one strip means some info gets cropped (route name truncated, no hourly forecast strip — would need a separate solution)
- Calendar in the left column stays unchanged but feels more dominant by comparison

**Implementation footprint:** large — new top/bottom structure, weather + transit need a horizontal "compact" mode, primary panel needs a state machine to swap between routine/todo/ambient.

---

### Option E — Rebalance the left/right column split (touch the calendar)

> Address the unreachable-tasks problem by giving the right column **more width**, taken from the calendar. The right column gets breathing room without changing its structure.

Today: `0.78fr / 1.22fr` → calendar 39% wide / right 61% wide. Roughly the calendar gets 478px and the right gets 750px at 1280-44 padding-18 gap = 1218px.

**Three sub-variants of this option, depending on how aggressive you want to be:**

**E.1 — Light shrink (calendar narrower):** `0.55fr / 1.45fr` → calendar ~340px, right ~860px. Calendar event blocks get shorter titles (longer ones truncate with ellipsis), but the timeline + future days still work. Right column has ~110px more width — Stars can become a horizontal row instead of stacked, freeing ~40px vertical.

**E.2 — Calendar today-only (drop "future days" and timeline shrinks):** Calendar shows today only — timeline 06:00–21:00 plus all-day chips, no future-days list. Maybe also tighten hour rows from 22px to 18px. That's a vertical save (~70-150px) but doesn't directly help the right column unless we ALSO shrink width. So pair with E.1.

**E.3 — Calendar as horizontal strip on top (radical):** Calendar becomes a wide horizontal "today only" timeline above the right column. Right column gets the full screen width.
```
┌─────────────────────────────────────────────┐
│ Masthead w/ clock                           │
├─────────────────────────────────────────────┤
│ ▣ Today  06─07─08─09─10─11─12─...─21        │ ← calendar strip (~120px tall)
├─────────────────────────────────────────────┤
│ Weather │ Routine/To-Do │ Stars │ Transit  │ ← right cards in 2x2 grid
├─────────┼────────────────┼───────┼──────────┤
│         │                │       │          │
└─────────────────────────────────────────────┘
```
Big remodel. Future days disappear (or move to a popover on tap).

**Pros (E in general)**
- Solves the right-column crush by addressing the root cause: not enough width was allocated
- Smallest of the structural changes (E.1 alone is ~10 lines of CSS)
- Touches the dimension we never questioned — the column ratio
- Preserves single-screen, no gestures
- Compatible with A or D layered on top later

**Cons**
- Calendar event titles get tighter — Hebrew titles already long, ellipsis hits sooner
- Future-days view (E.2/E.3) loses information you may want
- E.3 is a "the dashboard now looks completely different" change
- Doesn't solve the **routine-window** crush — even at 860px wide, weather+routine+todos still don't all fit vertically. Need to combine E with A or D for full fix.

**Implementation footprint:** small (E.1) → medium (E.2) → large (E.3). E.1 alone won't fully solve the routine-window problem — best paired with A or D.

---

## Comparison matrix

| | A — Scheduling | B — Swipe | C — Scroll | D — Hybrid | E — Rebalance columns |
| --- | --- | --- | --- | --- | --- |
| To-Do always reachable | ✓ (as strip) | ✓ (full page) | ✗ (below fold) | ✓ (full panel default) | partial — only if combined with A or D |
| Solves routine-window crush | ✓ | ✓ | ✗ | ✓ | partial (helps off-routine; routine-window still tight) |
| Single-screen feel | ✓ | ✗ | ✓ | ✓ | ✓ |
| No new gestures | ✓ | ✗ | ✓ | ✓ | ✓ |
| Routine card uncompromised | ✓ | ✓ | ✓ | ✓ | ✓ |
| Touches calendar | ✗ | ✗ | ✗ | ✗ | ✓ |
| Implementation effort | medium | medium-large | trivial | large | small (E.1) → large (E.3) |
| Risk on old Portal Chromium | low | medium (touch) | low | low | low |
| Future extensibility | medium | high | low | medium | low |
| Aesthetic cohesion | high | medium | low | high | depends on variant |

(Discovery / onboarding deliberately omitted — family-only project, not a concern.)

---

## Decision criteria — questions to weigh

1. **How important is "single screen, no interaction needed to see everything"?** If essential → A or D. If "interactive is fine" → B is open.
2. **How often does the family edit/check the To-Do?** If multiple times daily → To-Do should be the default primary view (D). If just glance occasionally → strip in A is fine.
3. **Are you OK adding a swipe gesture kids might trigger by accident?** No → rule out B.
4. **Do you want this dashboard to grow in the future** (photos page, recipes, message board)? Yes → B is most extensible. No → A or D win on focus.
5. **How disruptive can the rebuild be?** Low effort → A or C. Bigger remodel OK → D.

---

## What I'd pick — and why

**Recommendation: D + E.1 layered.** Hybrid layout (D) plus a small calendar-narrowing (E.1 — `0.55fr / 1.45fr` column ratio) underneath it.

Reasoning:

1. **D solves the routine-window crush** — the headline problem. To-Do becomes the default primary panel, visible most of the day. Routines take it over only during their windows. No card competes for vertical space.
2. **E.1 gives D more horizontal room to breathe** — at 860px wide instead of 750px, the primary panel can show a 2-column todo layout (next 5 active, recently checked) instead of single column. Stars in the context strip can show both kids inline instead of stacked.
3. **Together they touch both columns** so the redesign feels intentional, not lopsided. Just D would leave the calendar feeling oversized; E.1 alone wouldn't fix the routine crush.
4. **Stars + Transit get a real home** in the context strip (under D) — they're small-info elements anyway, ideal for a horizontal row.
5. **E.2 / E.3 are too aggressive** for v1 — losing the future-days view or making the calendar a horizontal strip both have real costs. Easier to ship D + E.1 first, then iterate to E.2 if calendar still feels too big.
6. **B (swipe) stays available as a future addition** — if you want a photos page or recipes page someday, layer it on top of D + E.1 without redoing the main layout.
7. **C is a band-aid** — doesn't solve the real problem.

**Honest caveat:** D + E.1 is the largest combined change. If you want to ship something this weekend, **A alone** is the lighter fix (medium effort, single-screen, no calendar disruption). It just doesn't make To-Do the visual default the way D does.

---

## If you pick D — what gets built

(Filling this in for completeness; same outline applies to whichever option you pick — I'll write a fuller implementation plan once you decide.)

**Files touched:**
- `/Users/elul/Projects/family-dashboard/index.html` (single-file dashboard)
- `/Users/elul/Projects/family-dashboard/AGENTS.md` (architecture update)
- `/Users/elul/Projects/family-dashboard/ROADMAP.md` (close stale items)

**Major changes to `index.html`:**
- Restructure `.right` flex column into two regions: `.context-strip` (top) + `.primary-panel` (bottom)
- New compact horizontal weather + transit "context strip" component (~120px)
- New "primary panel" container with `data-mode` attribute swapping between `routine-morning`, `routine-evening`, `todo` (default), `ambient` (late night)
- JS: extend `updateRoutineVisibility()` → `updatePrimaryMode()` covering all states
- Migrate existing routine + todo card content into the primary panel
- Delete old separate routine cards / todo card markup; they become primary-panel modes
- Preserve all existing IDs/classes the JS depends on (`#cal-body`, `#w-temp`, `#w-icon`, `#morning-card`, `#evening-card`, `#todo-list`, `#todo-form`, `#todo-input`, `.kid[data-kid]`, etc.) — wrap the old markup, don't replace it

**Major changes to JS:**
- New `updatePrimaryMode()` state machine — runs every minute, sets `body[data-mode="..."]`
- CSS controls which panel content is shown for each mode via `body[data-mode] .primary-panel ...`
- Existing `setCardVisible()` repurposed for mode switching

**Verification (any option):**
1. Test all four real states on Terry: weekday morning (06:00–08:59), weekday daytime (09:00–15:29), every day evening (15:30–20:29), every day late (after 20:30). Each must show a usable layout where To-Do is reachable in some form.
2. Specifically: try adding a To-Do task during EACH state. Currently impossible during routines — must be possible after.
3. Routine constraint preserved: morning shows all 9 items per kid without scrolling; evening shows all 10 items per kid without scrolling. Touch targets ≥40px.
4. Hebrew calendar event titles still render correctly.
5. Weekend morning behavior: morning routine hidden, weekend "what's primary?" answered correctly.
6. Force-restart Terry, confirm 5-min meta-refresh picks up the new layout cleanly.

---

## Next step

Reply with **A / B / C / D / E** (or a layered combo like "D + E.1" or "A then add B later") and I'll write the detailed implementation plan + start building.

---

## Implementation Plan: D + E.1 (chosen 2026-04-26)

### Decisions baked into v1 (override before I start if you disagree)

- **Transit in context strip:** minimal — Northern Line and Bus 102 each as a small badge + status pill (Good/Minor/Severe). Bus 102 inline shows next arrival as `Due / 4 min / 12:34` if available, otherwise just status. The expanded arrivals row (`#tfl-102-arrivals`) is dropped from v1 — too much info for the strip. If you miss it, we promote Transit to its own primary-panel mode in v1.5.
- **Weather hourly strip:** dropped from v1 context strip. Glanceable now (temp + icon + condition word) is enough for casual checks. If you miss the hourly trend, we add a "weather details" panel mode later.
- **Stars in context strip:** per-kid compact `🎁 Name N` (no pips) — saves horizontal room. Full Stars card with pips becomes a primary-panel mode you can pin manually if desired (v2).
- **Late-night ambient mode:** skipped from v1. Primary panel just shows To-Do all night. Add ambient mode later if it bugs you.
- **To-Do layout:** stays single-column for v1. Easy to bump to 2 columns later if the list typically has 8+ items.
- **Column ratio (E.1):** `0.55fr / 1.45fr` (calendar 27.5% / right 72.5%). At 1280px viewport this gives calendar ~335px (was ~478px), right ~880px (was ~750px). Calendar event titles will truncate sooner — Hebrew titles especially. If the calendar feels too cramped, we relax to `0.65fr / 1.35fr`.

### Architecture

**Right column restructured into two regions** — context strip on top, primary panel below:

```
.right (flex column)
├── .context-strip (fixed ~110px, always visible, horizontal flex row)
│    ├── .cs-weather   (icon · big temp · desc)
│    ├── .cs-divider   (vertical hairline)
│    ├── .cs-transit   (N · status pill   |   102 · next/status)
│    ├── .cs-divider
│    └── .cs-stars     (🎁 Eitan N · 🎁 Tamar N)
└── .primary-panel (flex: 1, fills remaining, data-mode swaps content)
     ├── .card.routine.morning  (#morning-card preserved)
     ├── .card.routine.evening  (#evening-card preserved)
     └── .card.todos             (preserved)
```

CSS-only mode switching via `body[data-mode="..."]`:
```css
.primary-panel > * { display: none !important; }
body[data-mode="morning"] .primary-panel > #morning-card { display: flex !important; }
body[data-mode="evening"] .primary-panel > #evening-card { display: flex !important; }
body[data-mode="todo"]    .primary-panel > .todos        { display: flex !important; }
```

`updateRoutineVisibility()` becomes `updateMode()` — picks one of `morning | evening | todo` and sets `body[data-mode]`. Wraps the same `nowMinutesLondon()` + `isWeekendLondon()` logic.

### Files touched

- `/Users/elul/Projects/family-dashboard/index.html` — CSS (grid, context strip, primary panel modes), markup restructure, JS rename + state machine
- `/Users/elul/Projects/family-dashboard/AGENTS.md` — update architecture section to reflect context-strip + primary-panel design
- `/Users/elul/Projects/family-dashboard/ROADMAP.md` — close "layout outside routine windows" + "to-do unreachable" entries; add "v1.5 / v2" follow-ups for the deferred decisions above

### IDs/classes that MUST survive (JS depends on them)

- `#cal-body`, `.cal-allday`, `.cal-today`, `.cal-future` (calendar — left column unchanged except width)
- `#w-icon`, `#w-temp`, `#w-desc` (weather — relocated to `.cs-weather`)
- `#w-hi`, `#w-lo`, `#w-feels` (already removed from markup — JS guards against null)
- `#w-hourly` (weather hourly strip — being dropped from markup; null-guard the JS that populates it OR delete that JS block)
- `#tfl-northern`, `#tfl-102` (transit line rows — relocated, kept as inline elements)
- `#tfl-102-arrivals` (bus arrivals — being dropped from v1; remove the JS function that populates it OR keep as no-op if element is absent)
- `#morning-card`, `#evening-card` (routine cards — relocated into primary panel)
- `.kid[data-kid="eitan-morning"|"tamar-morning"|"eitan-evening"|"tamar-evening"]` (routine kid columns — preserved)
- `#prog-eitan-morning` etc. and `#bar-eitan-morning` etc. (progress counters/bars — preserved)
- `#todo-list`, `#todo-form`, `#todo-input` (todo — relocated)
- Stars per-kid IDs (whatever Pi added — preserve)
- `#exit-corner` and its children (exit button — fixed position, unaffected)

### JS changes

- Rename `updateRoutineVisibility()` → `updateMode()`. Same trigger frequency (60s + on load).
- Replace `setCardVisible("morning-card", ...)` calls with `document.body.setAttribute("data-mode", pickMode())`.
- Drop `body.routine-active` class — the new `body[data-mode]` does the same job for other selectors. Update transit/stars CSS rules that referenced `body.routine-active`.
- Drop `loadHourlyForecast()` (or just stop calling it) since hourly markup is gone.
- Drop the bus-102 expanded arrivals render. Keep `loadTfl()` for line status — populates inline `#tfl-northern` and `#tfl-102` status pills.
- Update transit JS to write a "next arrival" snippet inline next to the 102 badge if live data is available.

### Build order (incremental, each step shippable)

1. **CSS grid ratio change (E.1 alone)** — change `.app` columns to `0.55fr / 1.45fr`. Verify calendar doesn't break, right column has more breathing room. Roughly 1 line of CSS, instantly verifiable. SAFE TO SHIP ALONE.
2. **Context strip skeleton** — add empty `.context-strip` markup at top of `.right`, define CSS for it, leave existing cards in place. No visual change yet.
3. **Move weather into context strip** — relocate `#w-icon`, `#w-temp`, `#w-desc` into `.cs-weather`. Delete old `.card.weather` wrapper. Style as inline.
4. **Move transit into context strip** — relocate `#tfl-northern`, `#tfl-102` into `.cs-transit`. Style as inline pills. Drop `#tfl-102-arrivals` markup + populate JS.
5. **Move stars into context strip** — relocate Stars elements into `.cs-stars`. Style as inline.
6. **Primary panel skeleton + mode CSS** — wrap remaining cards (`#morning-card`, `#evening-card`, `.todos`) in `<section class="primary-panel">`. Add the `body[data-mode] .primary-panel > X` show/hide rules.
7. **JS: updateMode()** — rename function, replace setCardVisible calls, drop the body.routine-active class.
8. **Polish + verification** — test all four real states on Terry, fix any layout bugs, update AGENTS.md and ROADMAP.md.
9. **Commit + push at logical breakpoints** — each numbered step above is its own commit. Allows easy rollback if any step breaks something on Terry.

### Verification matrix (run on Terry after step 8)

| State | Time | Expected primary panel | Expected context strip | Expected calendar |
| --- | --- | --- | --- | --- |
| Weekday morning | 06:00–08:59 | Morning routine (full, 9 items × 2 kids, no scroll) | Weather + Transit + Stars all visible | Today's events |
| Weekday daytime | 09:00–15:29 | To-Do (default) | Weather + Transit + Stars | Today's events |
| Every day evening | 15:30–20:29 | Evening routine (full, 10 items × 2 kids, no scroll) | Weather + Transit + Stars | Today's events |
| Late evening | 20:30+ | To-Do | Weather + Transit + Stars | Today's events |
| Weekend morning | Sat/Sun 06:00+ | To-Do (NOT morning routine — that's weekday only) | Weather + Transit + Stars | Today's events |

For each state: confirm a To-Do can be added from the dashboard. (This was the original failure mode and must be fixed in every state outside routine windows.)

### Estimated effort

End-to-end: ~2 hours of focused build + iteration. Steps 1–2 are minutes. Steps 3–5 are the bulk (each card needs new CSS for the inline form). Step 6 is small. Step 7 is small. Step 8 is the long tail.

**Heads up on size of diff:** this redesign touches a lot of CSS rules and reshuffles markup, but doesn't change any of the data-fetching JS (calendar, weather, transit, todo, stars, routine). Risk concentrated in CSS layout, not in functionality.

If you want to think on it overnight, this doc is committed to the repo at `UX_REDESIGN_PLAN.md` — readable on GitHub from any device.
