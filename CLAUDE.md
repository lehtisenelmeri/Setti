# Setti — project guide

Mobile workout-log app. Single self-contained file, vanilla JS, no build step, no dependencies.

## Files

- `index.html` — the entire app: markup, `<style>`, and one inline `<script>`. This is the only source file.
- `CLAUDE.md` — this guide.

Concept experiments (e.g. a neo-brutalist redesign) live as standalone files in the scratchpad, **not** in the repo — the real app is `index.html` only.

## Architecture

- **`h(tag, props, ...kids)`** (~line 284) — tiny DOM builder. `props.style` is an object; also supports `style`-`hover`/`active` variants, event handlers (`onClick`), `class`, `title`. `svg(attrs, inner)` builds inline SVG.
- **`S`** (~line 334) — single global state object (current tab, overlay, week plan, active workout `w`, prefs, `log`, …).
- **`setState(patch)`** (~line 384) — `Object.assign` into `S`, persist any touched localStorage-backed keys, then `render()` — a **full re-render** of the phone. No diffing; every change repaints. Keep renders cheap and idempotent.
- **`render()`** (~line 1223) — rebuilds header + scroll area + tab bar + active overlay into `#phone`, restoring scroll position.
- View functions: `viewToday`, `viewPlans`, `viewPlanPicker`, `viewProgress`, `viewBuilder`, `viewWorkout`, `viewSheet`, `viewDayDetail`, `viewSettings`. `S.tab` picks the main view; `S.overlay` (`'builder'` | `'workout'` | `'plans'`), `S.sheet`, and `S.dayDetail` layer sheets on top.

## Persistence (localStorage, `setti.*` keys)

`load(k, d)` reads, `setState` writes. Persisted keys: `log`, `dark`, `unit`, `accent`. Everything else in `S` is session-only. "Reset demo data" (Settings) clears `log` back to `[]`.

## Data model

- **`EX`** (~173) — exercise catalog keyed by id (`bench`, `ohp`, …): name, muscle group `g`, `sets`, `reps`, cues.
- **`PLANS`** (~238) — ready-made splits; each has a `week` array of 7 day objects `{name, ex:[ids]}`. `usePlan(id)` clones a plan's week into `S.week`.
- **`S.log`** — real logged sessions `{date:'YYYY-MM-DD', name, sets:[{id,g,kg,reps}], vol, mins}`, written by `saveSession()`. Drives Progress (calendar, muscle balance, e1RM chart) and the week strip.
- `DAY_LBL`, `GROUPS`, `GROUP_COLOR` (**fixed, distinct per-muscle colours** `[light bg, dark text, solid dot]` — deliberately not tied to the accent palette), `HISTORY`/`SERIES` (demo fallbacks), `DEMO_LOG` (seeds all 7 weekdays of the current week so the sticker strip + day-detail have content on a fresh load).

## Key features / helpers

- **Week strip** (in `viewToday`) — one horizontal **bar** (`weekBar`, thin dividers between days, 6px corners, `flexShrink:0`); each trained day this week shows a **sticker**, a **rest day** (`S.week[i].ex.length===0`) is darker + blank, else a small hollow dot. Today's cell is accent-tinted. Tapping a logged day opens the day-detail sheet.
- **Stickers** (`STK`, ~360) — the completion icons are **inline SVG line-art** in a `0 0 100 100` box, not emoji: thin coloured stroke + **opaque white fill** (so nothing shows through — no see-through holes) drawn via `_LN`/`_LO`/`_CF` helpers. Set of 12 (`STK_KEYS`): Trophy, Medal, Dumbbell, Barbell, Kettlebell, Bolt, Star, Heart, Crown, Check, Diamond, Rosette. `stickerForDate(dateStr)` returns a key; `stickerInkForDate(dateStr)` a colour from `STK_INK`. Both are **random per day, re-rolled each page load** (`STICKER_SALT`) but cached (`_stickerCache`) so they stay put across re-renders; the icon never repeats within a **4-day span** (`_stickerIndex` greedy no-repeat). **`stickerNode(dateStr, size, extraStyle)`** builds the positioned `<svg>` node (colour + white fill baked in; caller adds transform/`STICKER_FILTER` die-cut rim). Used by the week strip (40px, per-weekday rotation/offset/scale, spills past the bar so it isn't `overflow:hidden`), Progress calendar (20px, top-right corner), and the day-detail corner (62px).
- **Sticker slap** — finishing a workout (`saveSession`) sets `S.slapDay` to that date and forces `tab:'today'`; the strip sticker is tagged `id="strip-sticker-<YYYY-MM-DD>"`, and the `render()` post-hook (next to the `dayFrom` FLIP) animates a **slap-on** (drops in at ~1.9× → squash → settle) on that node, then clears `slapDay`.
- **Day detail** (`viewDayDetail`, overlay via `S.dayDetail` = a `YYYY-MM-DD`) — centered card listing that date's session(s): numbered/dotted sets with `kg × reps`. Opened by tapping a logged day in the week strip **or** the Progress calendar; both pass `S.dayFrom = {x,y}` (click coords). `render()` then FLIP-animates the card from that point to screen-centre with a spring overshoot ("jump"), clearing `dayFrom` so it doesn't replay.
- **Today card** — the day header (`DAY_FULL` + `name + ' day'`) lives in an accent-coloured band at the top of the exercise card; a separate small "Today" label sits above the card. Exercises are grouped by muscle (stable sort on `GROUPS` order) in both `viewToday` and `startWorkout`.
- **Plans** (`viewPlans`) — one green "Active plan" card holds the plan name/level and a recessed "Your week" well; each training day row taps to expand its exercises (`S.planOpen` = day index) as a dotted list with thin dividers where the muscle group changes. "Customise plan" opens the builder; "Change plan" opens `viewPlanPicker` (bottom sheet, `S.overlay==='plans'`).
- **Builder** (`viewBuilder`) — day tabs + move bar (both `flexShrink:0`, or they squish in the flex column and text spills). Slot cards + drawer rows show a small muscle-group tag + big exercise name (no sets×reps). The "Add by muscle group" drawer collapses to a bottom bar via its grip handle (`S.drawerOpen`). **Exercise reorder is tap-based** (`tapEx`/`moveEx`, `S.exPick`): tap a slot's grip (⣿) to pick it up, tap another to drop it there — HTML5 drag was removed (dead on touch). The drawer's exercises still drag-to-add on desktop.
- **`spinField(opts)`** (~475) — spin-wheel number editor (tap number → drag strip 1:1). Module vars `scrubbing` (freezes the 250ms rest re-render while open) and `activeSpin` (only one open at a time). Used for kg/reps on main log + rest card. Weight step 0.25, reps 1.
- **`logSet`** / **`saveSession`** — log a set / commit a finished session to `S.log`.
- **Reordering** — days: `moveDay(from,to)`/`tapDay(i)` on `S.week` (builder move bar). Exercises within a day: `moveEx(from,to)`/`tapEx(i)` with `S.exPick` (builder slots).
- Date helpers near `firstTrain`: `ymd`, `parseYmd`, `weekStart`, `doneThisWeek()` (boolean[7], Mon-first, from the log).

## Theming

CSS `var()` seeds live on `body` (not `:root`); `applyPrefs()` toggles `dark` and `pal-*` classes on `body`. Palettes in `PALETTES`. Fonts loaded from Google Fonts in `<head>`: heading = **Barlow Condensed** (`--font-heading`), body = Figtree (`--font-body`). A global rule `[style*="--font-heading"]{font-weight:700}` forces all inline-`--font-heading` elements heavy (Barlow Condensed's regular reads too light).

## Verify an edit

Extract the inline script and syntax-check with node:

```bash
cd "C:/Users/el0821/Desktop/Setti" && node -e "const fs=require('fs');let h=fs.readFileSync('index.html','utf8');let m=h.match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync(process.env.TEMP+'/setti.js',m[1]);require('child_process').execSync('node --check \"'+process.env.TEMP+'/setti.js\"',{stdio:'inherit'});console.log('OK')"
```

## Run / preview

Static file — serve and open in a real browser (localStorage works there; the in-app browser pane treats `file:`/`data:` URLs as static snapshots that block localStorage and flake on clicks):

```bash
cd "C:/Users/el0821/Desktop/Setti" && (python -m http.server 8777 >/dev/null 2>&1 &)
```

Then open `http://localhost:8777/index.html`. Use the browser device toolbar (F12) for a mobile/touch view. If the week strip looks empty after a change to demo seeding, old stored data may override it — run `localStorage.removeItem('setti.log')` in the console and reload.

## Conventions

- Match the surrounding style: object-literal inline styles, terse comments, `var()` tokens for all colors/radii/shadows.
- No frameworks, no build, no new dependencies — keep it one self-contained file.
