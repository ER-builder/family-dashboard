# Family Dashboard — UX Redesign Planning Doc

**Status:** decision pending. **Audience:** Elul + family. **Scope:** dashboard layout & information architecture only — no backend/proxy/kiosk-app changes.

---

## Why we're redesigning

The dashboard has accumulated five cards in the right column on top of the calendar in the left. At 1280×800 (Portal screen), there isn't enough vertical space for all of them to coexist. The CSS uses `flex-shrink: 0` on every card except `.todos` — which has `flex: 1` and absorbs all the overflow by shrinking to ~zero height. **Result: the To-Do card is unreachable any time a routine card is active** (06:00–08:59 weekdays, 15:30–20:29 every day) and badly squeezed even outside those windows.

This is a structural problem, not a tweak.

### The constraint conflict, in numbers

At 1280×800, after masthead + padding + gaps the right column has **~636px of vertical room**. Card heights when present:

| Card | Height | Can shrink? |
| --- | --- | --- |
| Weather | ~130px | no |
| Morning routine (9 items × 2 kids) | ~430px | no |
| Evening routine (10 items × 2 kids) | ~470px | no |
| Transit | ~90px (hidden during routines) | no |
| To-Do | absorbs leftover | yes, collapses to nothing |

**Three real states today:**

```
WEEKDAY 06–09 (morning):
  weather 130 + gap 18 + morning 440 + gap 18 + transit hidden + todos = ?
  → todos gets ~30px → unreachable

EVERY DAY 15:30–20:30 (evening):
  weather 130 + gap 18 + evening 470 + gap 18 + transit hidden + todos = ?
  → todos gets ~0px → completely gone

DAYTIME / WEEKEND MORNING:
  weather 130 + gap 18 + transit 90 + gap 18 + todos = 256px used
  → todos gets ~380px → fine
```

We've already sacrificed weather detail (Hi/Lo/Feels removed) for routine fit. Further compression hits the ≥40px kid-touch floor (AGENTS.md hard constraint). The only way out is structural.

---

## Project context (so your future-self remembers)

- **Family-only project**, not for distribution. Discovery / onboarding is not a concern — Elul can teach the family any new gesture once.
- **Always-on kitchen dashboard**, viewed across the room. Glanceability is the prime UX value.
- **Kids are the heaviest interactors** with routines. They tap items in sequence each morning + evening.
- **To-Do is dual-purpose**: glanceable (parents see what's left) AND interactive (mark items done, add new ones from the dashboard or from phones via the linked Google Sheet).
- **Calendar is read-only** in the left column.

---

## Four design options

Each option has a one-line idea, an ASCII sketch, and honest pros/cons. **No recommendation buried inside — see "What I'd pick" at the bottom.**

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

## Comparison matrix

| | A — Scheduling | B — Swipe | C — Scroll | D — Hybrid |
| --- | --- | --- | --- | --- |
| To-Do always reachable | ✓ (as strip) | ✓ (full page) | ✗ (below fold) | ✓ (full panel default) |
| Single-screen feel | ✓ | ✗ | ✓ | ✓ |
| No new gestures | ✓ | ✗ | ✓ | ✓ |
| Routine card uncompromised | ✓ | ✓ | ✓ | ✓ |
| Implementation effort | medium | medium-large | trivial | large |
| Risk on old Portal Chromium | low | medium (touch) | low | low |
| Future extensibility | medium (more states) | high (more pages) | low | medium |
| Aesthetic cohesion | high | medium | low | high |

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

**Recommendation: Option D** (hybrid: sticky context strip + rotating primary panel).

Reasoning:

1. **Kitchen behavior favors zero-tap glanceability.** During the 7am rush you don't want to swipe to check tasks. D keeps everything visible.
2. **To-Do deserves to be the DEFAULT view.** It's the most-edited card and currently the least reachable. Flipping that priority is the single biggest UX win.
3. **B's discovery argument got weaker** once you confirmed it's family-only — but the touch-handling risk on Portal's old Chromium is still real, and the "swipe to add a task while cooking" friction is still there.
4. **D doesn't preclude B.** Once the hybrid layout is shipped, you can add a swipe-to-page-2 later if you want a photos page or similar — without redoing the main layout.
5. **A is the lighter alternative to D.** Same destination, less rebuild, but adds the complexity of a new "masthead strip + overlay" pattern that's neither here nor there.
6. **C is a band-aid** — doesn't solve the real problem, just shifts it.

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

Reply with **A / B / C / D** (or a hybrid like "D but with the swipe-to-page-2 included now") and I'll write the detailed implementation plan + start building.

If you want to think on it overnight, this doc is committed to the repo at `UX_REDESIGN_PLAN.md` — readable on GitHub from any device.
