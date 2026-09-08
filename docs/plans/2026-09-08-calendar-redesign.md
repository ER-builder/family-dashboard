# Calendar redesign — "fewer, bigger, tappable"

**Status:** planned, not built. Raised by Elul 2026-09-08: *"The calendar is
hard to read. Maybe make it clicked for more info? Change font size? Remove
the tmw section?"*

---

## Diagnosis — why it reads badly today

The calendar column is `0.55fr` of `0.55fr / 1.45fr` at 1280px ≈ **352px**,
minus card padding (20px each side) ≈ **312px of content width**. Inside a
block, `grid-template-columns: 56px 1fr auto` with `gap: 20px` and
`padding: 10px 16px 10px 20px` spends roughly **100px on chrome, time and
duration**, leaving **~210px for the title** at 19px serif. That is the root
cause; four separate symptoms fall out of it.

1. **Titles truncate constantly.** In the 2026-09-08 photo: *"Commute to…"*,
   *"Year 4 Meet the T…"*, *"Golder…"*. The 2-line clamp helps English but
   Hebrew RTL titles (*תמר – פורסט*) butt straight into the duration cell —
   which is exactly why `gap` was pushed 14 → 20px, itself taking another
   6px away from the title. We have been paying for the layout with the
   thing the layout exists to show.
2. **Tomorrow is unreadable and expensive.** Six blocks at a 13px title, on
   a 55%-opacity background, indented 14px. At kitchen distance that is
   texture, not information — and it costs roughly **220px**, about 40% of
   the column, plus it is what pushes the agenda into a scrollbar (visible
   in the photo).
3. **Everything is equally loud.** A 15:15 event and an 07:45 event are
   rendered identically. The only hierarchy is Now vs. Up Next.
4. **Truncation is unrecoverable.** `ev.location` is stashed in a `title`
   attribute — a hover tooltip, on a touch kiosk that has no hover. Dead
   affordance.

Two latent bugs found while measuring:

- **Calendar titles carry no `dir`.** `#radio-name` has `dir="rtl"`, but
  `.agenda .block .ti` has nothing, so Hebrew renders with an LTR base
  direction — trailing punctuation and digits land on the wrong side.
  `dir="auto"` fixes it per-event and costs nothing.
- **Dead code.** `.agenda .block.past` is styled but `opts.past` is never
  set by `addSection()` (past events are dropped from the agenda entirely),
  and `positionNowLine()` is a no-op still on a 60s `setInterval` left over
  from the deleted timeline view.

---

## Vertical budget (1280×800, computed — verify on device)

```
viewport                                        800
− .app padding (18 top + 18 bottom)             −36
− masthead (44px greeting + date + pad + rule)  ≈−90
− two 18px row gaps (row 2 is empty)            −36
                                                ────
  .calendar card                                ≈638
− .card padding (18 + 18)                       −36
− .eyebrow (13px line + 14px margin)            −30
                                                ────
  .calendar .body                               ≈572
− all-day chip strip, when present              ≈−44
                                                ────
  agenda, default modes                         ≈528
− routine-mode corner clearance (padding-bottom) −52
                                                ────
  agenda, morning/evening modes                 ≈476   ← design to this
```

Per repo rule: **measure at exactly 1280×800 with the webfonts loaded.**
These are computed figures, not measured ones.

---

## The design

### 1. Two-line blocks — give the title the whole column

Drop the `56px | 1fr | auto` grid. Each block becomes two stacked lines:

```
┌────────────────────────────────────┐
│ 07:45 · 30m · SCHOOL               │  meta: mono 15px, ink-soft
│ דרוס ונוע – הידו                    │  title: serif 21px, full width
└────────────────────────────────────┘
```

- Title gets **~272px instead of ~210px** — about 30% more characters —
  *and* the time gets **bigger** (13 → 15px), not smaller. Time is the
  primary scanning key on an agenda; it was the smallest text on the card.
- Meta line absorbs the duration, so the `auto` column and its 20px RTL
  safety gap both disappear.
- The source-colour `::before` bar stays exactly as is (do **not** convert
  it to a child div — old Portal Chromium auto-places abs-positioned grid
  children, which bit us 2026-05-11).
- `dir="auto"` on the title element.

Heights: `padding 9` + `meta 16` + `gap 5` + `title 24` (one line) + `9` +
`2px border` ≈ **65px**; a 2-line title ≈ 89px. "Now" block ≈ 68px.

### 2. Cap Up Next at 4, with an overflow tap

`upcoming.slice(0, 6)` → `slice(0, 4)`. If more remain, append a
`+3 more today` pill that opens the sheet in day mode. Nobody reads item 9
from across a kitchen; the ones past 4 are better served by a deliberate tap.

### 3. Mode-aware Tomorrow (chosen over deleting it)

- **Default / spotify / morning modes** — one collapsed strip, ~44px:
  `TOMORROW · WED 9 SEPT   5 events · first 07:15  ›`
  Reclaims ~180px from a section nobody could read anyway.
- **Evening mode (15:30–20:30)** — expands to **4 blocks**, title 13 → 15px,
  plus `+n more` → sheet. Tomorrow matters at 19:00 and not at 09:00, so the
  space follows the need instead of being permanently allocated.
- Either state is tappable → day-mode sheet with the full list.

CSS-gated on `body[data-mode]`, never `:has()` (risky on Portal Chromium).

### 4. Tap for detail — `#cal-sheet`

Tapping any block, the tomorrow strip, or a `+n more` pill opens a centered
overlay, modelled on the existing `.celebration` overlay (same fixed-full-
viewport + fade-out pattern, so it is a known-good shape on this WebView).

```
        ┌──────────────────────────────────┐
        │  SCHOOL                          │  source chip, source colour
        │  07:45 – 08:15        30m        │  mono 28px
        │                                  │
        │  Year 4 Meet the Teacher         │  serif clamp(28,3.2vw,40)
        │                                  │  dir="auto", no clamp
        │  📍 Room 4B                      │  location, if present
        └──────────────────────────────────┘
```

- Dismiss: tap anywhere, `×` button, **15s auto-dismiss** (kitchen: nobody
  closes things), and forced-closed by `updateMode()` on a mode change.
- Clear the auto-dismiss timer on re-open, or a fast second tap inherits the
  first timer and shuts the sheet early.
- **Solid `rgba` scrim, no `backdrop-filter`** — unproven on this Chromium.
- `z-index` above `#exit-corner` / `#spotify-corner` (both `position: fixed`).
- **Day mode:** same sheet, listing every event for the chosen day at 17px.
  This sheet *may* scroll — it is a deliberate modal, not the routine card.
- **Single `click` handlers only.** No `click` + `touchend` pair (2026-05-17
  hardening: `preventDefault()` on `touchend` suppresses the synthetic click
  on Portal Chromium → intermittent dead taps). `touch-action: manipulation`
  in CSS instead.

### 5. Type floor and cleanup

- Nothing in the calendar below 13px except the eyebrow / section labels.
  Tomorrow titles 13 → 15px in the expanded state.
- Delete `.agenda .block.past` (dead), `positionNowLine()` and its
  `setInterval` (dead since the timeline was removed).
- Drop the `title=` location tooltip; the sheet replaces it.
- Add **`checkAgendaFit()`**, mirroring `checkRoutineFit()`: warn to console
  when `#agenda` scrollHeight exceeds clientHeight, and paint it under
  `?debug=fit`. `.calendar .body` keeps `overflow-y: auto` as a safety valve
  — unlike the routine card, a scrollbar here is ugly, not broken — but the
  layout should be budgeted never to need it.

### Budget check (morning mode, ~476px)

```
section-head "Now"                  20
now block                           68
gap                                  8
section-head "Up Next"              20
4 × 65px block + 3 × 8px gap       284
tomorrow strip (collapsed) + margin 66
                                   ───
                                   466   ← fits, ~10px slack
```

Slack is thin. If it overruns on device, the first thing to give is the
`Up Next` cap (4 → 3), not the type sizes — the type sizes are the point.

---

## Open dependency

The detail sheet is only as good as the proxy payload. The dashboard
currently consumes `title, start, end, allDay, source, location`. **Check
whether `family-dashboard-proxy` also passes through the iCal `DESCRIPTION`
field** — if it does, the sheet should show it; if not, that is a one-line
change in the proxy repo and worth doing before phase 2. (Could not be
verified from this session — the proxy host is outside the network policy.)

---

## Phasing — each step independently shippable

1. **Layout, type and budget** (§1, §2, §5). The whole readability win, no
   new interaction surface, lowest risk. Ship and look at it for a day.
2. **Detail sheet** (§4). Makes remaining truncation recoverable.
3. **Mode-aware tomorrow** (§3). Depends on the sheet for its expanded view.

## Risks

- **Vertical budget is computed, not measured.** Verify at 1280×800 with
  real webfonts before shipping; a fallback-font dev browser will lie.
- **Old Chromium**: no `:has()`, no abs-positioned grid children, avoid
  `backdrop-filter`. All three are already avoided above.
- **A new tap target during routine windows.** The calendar is on screen in
  morning/evening mode, so a kid can open the sheet mid-checklist. The 15s
  auto-dismiss and mode-change close are what keep that harmless.
