# Torch — project guide

Mobile workout-log + social app. Single self-contained file, vanilla JS, no build step, no dependencies.

**The app is named Torch.** It used to be "Setti"; that name is gone from all UI. Two deliberate leftovers, do **not** "fix" them:
- **localStorage keys stay `setti.*`** (`load()`/`applyPrefs()`). Renaming them would orphan every existing user's logged sessions. Migrate properly or leave alone.
- The **repo and Pages URL are still `Setti`** (https://lehtisenelmeri.github.io/Setti/).

## Files

- `index.html` — the entire app: markup, `<style>`, and one inline `<script>`. This is the only source file.
- `README.md` — public-facing readme (Finnish).
- `CLAUDE.md` — this guide.

Concept experiments / prototypes (e.g. marker mockups, brand sheets) live as standalone files in the scratchpad, **not** in the repo — the real app is `index.html` only.

## Architecture

- **`h(tag, props, ...kids)`** — tiny DOM builder. `props.style` is an object; also supports `style`-`hover`/`active` variants, event handlers (`onClick`, `onPointerdown`, `onInput`, …), `class`, `title`, `src`, `value`, etc. `svg(attrs, inner)` builds inline SVG.
- **`S`** — single global state object (current tab, overlay, week plan, active workout `w`, prefs, `log`, social, invites, …).
- **`setState(patch)`** — `Object.assign` into `S`, persist any touched localStorage-backed keys via `applyPrefs()`, then `render()` — a **full re-render** of the phone. No diffing; every change repaints. Keep renders cheap and idempotent. Avoid `setState` during a text-typing gesture (mutate `S.w.kg` directly on `onInput`, commit on next render).
- **`render()`** — builds `header()` + the **swipe pager** + `tabBar()` + any active overlay into `#phone`.
- Main views: `viewToday`, `viewPlans`, `viewFriends`, `viewProgress`, `viewSettings`. `S.tab` (`'today' | 'plans' | 'friends' | 'progress' | 'settings'`) picks the main view.
- Overlays / sheets (layered on top, own z-index): `viewBuilder` (`S.overlay==='builder'`), `viewWorkout` (`'workout'`), `viewPlanPicker` (`'plans'`), `viewSheet` (`S.sheet`), `viewDayDetail` (`S.dayDetail`), `viewFriendProfile` (`S.friendProfile`), `viewFriendPlan` (`S.friendPlan`), `viewInviteCompose` (`S.invitePanel`), `viewEditProfile` (`S.editProfile`). The camera (`openCamera`) is **imperative** — it appends its own fixed overlay to `document.body` outside the render cycle (it holds a live `<video>` stream).

### Swipe pager (`buildPager`)
`TAB_ORDER = ['progress','plans','today','friends','settings']`. Three panels (prev · current · next, each its own vertical scroller) ride a flex track offset by `translateX(-100%)`. A horizontal drag moves the track 1:1 with the finger and snaps to a neighbour (commits the tab after the animation) or springs back; vertical drags fall through to scroll. Skips when a gesture starts on an inner horizontal scroller (`.sx`) or when any overlay/sheet is open. Per-tab scroll positions cached in `_scrollByTab`. `viewFor(tab)` returns the view for a tab.

## Persistence (localStorage, `setti.*` keys)

`load(k, d)` reads, `applyPrefs()`/`setState` write. Persisted keys: `log`, `dark`, `unit`, `rest`, `mark`, `uname`, `pfp`, `unameAt`. (There is **no** `accent` key: one fixed coral brand, no palette switcher.) Everything else in `S` is session-only. "Reset demo data" (Settings) clears `log` back to `[]` **behind a confirm dialog**. Full data is portable via **Export/Import** (`exportBackup()` downloads a JSON of log+week+prefs; `importBackup()` reads one back, both gated by `viewConfirm`).

## Data model

- **`EX`** — exercise catalog keyed by id (`bench`, `ohp`, …): name `n`, muscle group `g`, `lvl`, `sets` (base 2), `reps`, `cues`. **Cardio** entries (`run`, `walk`, `cycle`, …) have `cardio:true`, `sets:1`, no weight; `GROUPS` includes `Cardio`.
- **Progress** (`viewProgress`) — month calendar with **prev/next nav** (`S.calMonth` offset, next disabled at the current month), "Most improved" (est. 1RM gain), muscle balance (**Cardio is excluded** from the balance bars — `GROUPS.filter(g => g !== 'Cardio')`).
- **`PLANS`** — ready-made **splits**; each `{name, level, days, blurb, week:[7 × {name, ex:[ids]}]}`. `usePlan(id)` clones a plan's week into `S.week`. `createPlan()` seeds `PLANS.custom` (7 rest days, name "My Split") and opens the builder. NOTE: in the **UI** every "plan" now reads **"Split"** (tab, buttons, headings), but the **code identifiers stay `PLANS`/`planId`/`planName`/`viewPlans`/`S.overlay==='plans'`** — don't rename those.
- **`S.log`** — real logged sessions `{date:'YYYY-MM-DD', name, sets:[{id,g,kg,reps}], vol, mins, photo}` (`photo` = data URL or null), written by `saveSession()`. Drives Progress (calendar, muscle balance, Most-improved), the week strip, and your own posts in the Friends feed.
- `DAY_LBL`/`DAY_FULL`, `GROUPS`, `GROUP_COLOR` (**fixed, distinct per-muscle colours** `[light bg, dark text, solid dot]` — not tied to the accent palette), `HISTORY`/`SERIES` (demo fallbacks), `DEMO_LOG` (seeds all 7 weekdays of the current week).
- **Social (demo)** — `FRIENDS` (id, name, initials, accent, level, planName, streak, thisWeek/weekGoal, `week`), `FEED` (activity posts, same shape as `S.log` + `userId`), `_LIVE_NOW` (presence ids). `_ago(n)` builds a `YYYY-MM-DD` n days back.

## Done-day marker (check / bowtie) — replaced the old stickers

`S.mark` (`'check' | 'bowtie'`, Settings toggle) picks the completion marker. (The old `STK` sticker set + `stickerNode` + `STICKER_FILTER*` were dead and have been **removed**.)

- **`doneMark(size, extraStyle)`** — the filled marker. `check` = accent circle + white check; `bowtie` = a bare cute pink bow (loops + oval knot + crease lines + top-left highlight + splayed notched tails, `bowSvg`), no circle. Used on the week strip (with `id="strip-sticker-<date>"` so the finish hook can animate it), the Progress calendar corner (18px), and the day-detail corner (52px). `_posOnly()` strips non-positional styles for the bow (which has no circle to take a border).
- **`holsterMark(size)`** — the empty "holster" shown on scheduled-but-not-done days: `check` = dashed accent ring + ghost check; `bowtie` = a **dotted** bow outline. On completion the marker **stamps** into the holster (finish hook in `render()` animates `#strip-sticker-<date>` via the `stampIn` keyframe: drop in ~1.9× → squash → settle).
- **`saveSession`** sets `S.slapDay` to the finished date + forces `tab:'today'`; the `render()` post-hook animates the stamp, then clears `slapDay`.

## Key features / helpers

- **Today** (`viewToday`) — date line + big condensed "Today" with your **streak badge** in the top right (`streakBadge(48, myStreak())`, hidden at 0), keyed off the **real weekday** via `todayIdx()`. `myStreak()` counts consecutive days back from today that have a logged session; **scheduled rest days are skipped** (they don't break it) and an unfinished today doesn't end the run. While a session is live, the start buttons read **"Back to workout"** and reopen it rather than clobbering `S.w`. Layout top-to-bottom: **accepted-invite cards** (see Gym invites — one solid-accent card per invite you've accepted: "`<workout>` with `<names>` at `<time>`"), a **quick-invite card** (friends with no logged session today and not mid-workout → avatar stack + "Invite to the gym", which opens `viewInviteCompose` **pre-selected** with those friends), a **day-pill row** (`weekBar`: done → `doneMark`, scheduled-not-done → `holsterMark`, rest → grey dot; today's pill is raised/accent-labelled), then the **main card**. Training day = a **solid-accent hero card** (day name + count + white "Start workout") over a white exercise list + **Swap / Empty** ghost buttons; rest day = a white card with a **random-per-date encouragement** (`restQuote()` picks from `REST_QUOTES` by a hash of the date) + **"Start an empty workout"** (`startEmptyWorkout`). `DEMO_LOG` leaves two weekdays undone (`skip`) so holsters show on a fresh load.
- **Day detail** (`viewDayDetail`) — centered card of that date's session(s): numbered sets `kg × reps` (or a cardio duration `m:ss`), plus the photo if attached. FLIP-animates from `S.dayFrom`.
- **Split** (`viewPlans`, formerly "Plans") — accent "Active split" card (tonal gradient) + week well; "Customise split" → builder, "Change split" → `viewPlanPicker` (titled **Templates**; each preset card has **Customise** + **Use**), "Create split" → `createPlan()`.
- **Builder** (`viewBuilder`) — day tabs + a bar with the day name and a **"Set as rest day"** toggle (`setRestDay`). Editable split-name input in the header. Slot cards: **press-and-drag reorder** via the grip (`startExDrag`), a vertical **set stepper** (`setSets`), corner info/remove. The "Add by muscle group" drawer is **drag-to-resize** by its grip (`startDrawerDrag`, snaps open/closed; tap still toggles).
- **Workout** (`viewWorkout`) — weight = **typed field** with a centered number + pen (mutates `S.w.kg` on `onInput`); reps = **−/+ steppers**. **Start empty** or add exercises live via the in-workout picker (`viewWorkoutPick`, `S.wPick` → `addExToWorkout`); **Add exercise** / **Finish** (`finishWorkout`) buttons on every set screen. **Cardio** exercises (`EX[id].cardio`, group `Cardio`) show a stopwatch instead: **Start → Stop (red) → Continue / End**, tracked on `w.cardioRunning`/`cardioStart`/`cardioAccum` (`cardioBegin`/`cardioStop`/`cardioEnd`, logs `{secs}`). Rest screen keeps the ring + `spinField` mini-editors. **Minimising**: the top-left control is a **chevron-down, not an X** — `minimiseWorkout()` only clears `S.overlay`, so `S.w` (and presence) keep running and the whole app stays browsable mid-session. `viewWorkoutBubble()` then floats a coral bubble above the tab bar on every tab: elapsed time + current exercise, or, while resting, the rest countdown + `Next · <exercise>` on the blue secondary. Tap it to reopen; its small × discards the session behind a `viewConfirm`. `bubbleLines()` is shared by the view and the interval's text patcher so they can't drift. **Finish screen**: stat tiles, set-value **bubbles**, an **"Update [Day]'s split"** toggle (`S.saveToPlan` writes the session's exercises into `S.week[today]`), **Add a photo** (camera), Save. Session name = `w.name` (custom) else the day name.
- **Camera** (`openCamera`, imperative overlay) — live `<video>`, opens in **selfie/front** mode with a **flip** button (mirrored preview + capture), **flash/torch** toggle (hidden if unsupported), **0.5× / 1× / 2× zoom buttons** (hardware zoom when available, digital crop otherwise), shutter, and retake/use. Preview + capture are locked to a **portrait 3:4 frame**. Falls back to the native picker (`pickPhotoFallback`) if `getUserMedia` is missing or denied. `readPhoto(file, cb)` downscales imports to ≤1080px JPEG.
- **Friends** (`viewFriends`) — friend avatar strip (live "at the gym" ring via presence, `isLive`; a **🔥 streak count** under each name), **gym invites** (see below), and an **activity feed**: friends' `FEED` posts + your own photo'd sessions (`userId:'me'`), newest first, **dropped after 4 days**. A post with a photo shows the whole image with the workout name overlaid + "See workout"; without a photo, a short preview + "Expand" (both inside the inner box).
- **Friend profile** (`viewFriendProfile`) — focus is their **last workout** (day-detail style: date label, big name, `min · kg`, plain-numbered set rows with a divider **only where the muscle group changes**, photo). 🔥 streak badge on the avatar; **"See plan"** opens `viewFriendPlan`.
- **Gym invites** — group invites `{id, host, time('now'|'HH:MM'), invitees:[{id, status}]}` in `S.invites` (`'me'` = you). `viewInviteCompose` multi-selects friends + Now/time → `sendInvite(ids, when)`. Incoming (`myEntry` pending/suggested) shows **host + already-accepted people** ("X and Y invited you to the gym", `joinNames`) with Accept/Decline/Suggest-time (`respondInvite`). Outgoing (host `'me'`) shows the invitee stack + going/pending/declined status. Seam-ready for a real `/api/invites`.
- **Settings** (`viewSettings`) — editable **profile** (`viewEditProfile`: photo anytime, username once per 7 days via `unameAt`; `selfAvatar` shows `S.pfp` or the initial). **Dark mode** toggle, units, **rest timer** (`restStepper`, min/sec), **done marker** toggle (Check / Pink bowtie). Account card: **Export** / **Import** JSON backup (`exportBackup`/`importBackup`). **Reset demo data** and destructive restores route through `viewConfirm` (generic `S.confirm = {title, body, ok, run, cancel?}` dialog). No accent-colour swatches (single brand).
- **Social data seam** — `Social.friends()/feed()/profile(id)/sessionsFor(id)/presence()/setPresence(on)` all return Promises over the seed arrays; `loadSocial()` caches into `S.social`. To go live, swap each body for `fetch('/api/...')` — the views never change. **Presence = who's mid-workout**: `startWorkout()` calls `setPresence(true)`, `saveSession()` `setPresence(false)`.
- Date helpers near `firstTrain`/`todayIdx`: `ymd`, `parseYmd`, `weekStart`, `doneThisWeek()`.

## Re-render note

`render()` swaps every node, so a full render landing mid-press destroys the button the finger is on and **the click never fires**. This is a real bug that shipped once ("Join the competition" was dead because the leaderboard repainted every 250ms). Three guards keep it from coming back, all in the 250ms interval:

- **`touching`** — a global flag set by capture-phase `pointerdown`/`pointerup`/`pointercancel` on `document`. The interval bails while a pointer is down. Any new timer-driven repaint must respect it.
- **`scrubbing`** — freezes it while a `spinField` is open.
- **Scope** — it only full-renders when the workout overlay is actually open (rest countdown ring, or the leaderboard sheet). Minimised, it just patches the bubble's two text nodes; during active logging it patches `#wClock` only.

## Chrome / safe areas

**No fake device chrome at all.** The old `.island` / `.statusbar` / "9:41" / `.homebar` CSS **and** the `statusBar()` function were dead code and are **removed** - a real app never draws a fake iOS around itself. Do not re-add them. Instead the head declares a **PWA manifest + icon** (inline `data:` URIs, see Brand), `theme-color`, and `apple-mobile-web-app-*` meta so it installs standalone. `header()` pads with `env(safe-area-inset-top)`, the tab bar with `env(safe-area-inset-bottom)`. Mobile media query uses `100dvh`. The `.phone` frame is a **desktop-preview shell only** (full-screen under the 640px media query) - it is not simulated OS UI.

The **tab bar** is a **solid `--color-surface` with a 1px divider** - no blur/glass (`backdrop-filter` was removed). The active tab's icon sits in a coral **pill**; that pill is one of the very few sanctioned pill shapes (see Design guardrails).

## Brand

**One mark: the flame.** `FLAME_PATH` (a single const, 24-unit grid, ink bbox `x 4.6 y 1.8 w 14.825 h 20.196`) is the *only* flame geometry in the codebase. It is reused verbatim by:

- `flameSvg(size, color)` — every streak badge in the UI (Friends strip, friend profile).
- The **favicon**, **apple-touch-icon** and **manifest icon** — all three are inline `data:` URIs of a coral tile (`rx 5.4` on a 24 grid) plus the same path at `translate(2.72 2.9) scale(.772)`.

If the flame changes, change it in **all four** places (plus `404.html` and the OG script) or they drift apart. The shape is **faceted**: straight angled planes, a tall dominant peak with a shorter second peak beside it, and a wide weighted base that nods at a kettlebell. Ink bbox `x 4.6 y 1.5 w 14.8 h 20`.

**Two failure modes this mark already hit, do not walk back into them:**
- A **smooth round bottom** (a semicircle base) makes it read as the **Tinder** logo. That is what the first two drafts did. The faceted top alone does not save it; the giveaway is the base.
- A **literal kettlebell** (handle arch with a hole over a wide bell) reads as a **padlock, handbag or a person's head and shoulders** at icon size. Tried and rejected.

Check any edit at **24px and 40px** (where detail collapses) *and* at 200px+ (where a too-wide base starts to read as a sack). The OG card deliberately renders the flame small for that reason.

**`streakBadge(size, count, color)`** — the flame with the streak count set **inside its belly** (`<text>` centred at `12, 17.2`, font-size steps down as the number gets more digits). No pill, no wrapper: one shape carries both the metaphor and the number. Used on the Friends avatar strip (28px), the friend profile (40px) and the Today header (48px). `flameSvg()` is the plain, numberless flame.

**Wordmark** (not used in-app; lives as a brand asset): `TORCH` set in Barlow Condensed 700 with the flame occupying the O slot. Measured, not eyeballed — at `font-size 200`: cap height 141, advances `T 93.60 · O 94.60 · R 94.21 · C 92.80 · H 96.00`, O ink `83 × 143`. The flame gets its own slightly wider slot (advance 102, ink 90 × 150, so it overshoots the cap line by 7 like a pointed glyph should) via `translate(71.674 38.63) scale(6.0708 7.4272)`, with `RCH` starting at `x 195.6`. The non-uniform scale is intentional: a condensed wordmark needs a condensed flame.

## Theming

CSS `var()` seeds live on `body` (not `:root`); `applyPrefs()` toggles the single `dark` class on `body`. **One committed brand colour: coral** (`--a-base #FB5A34`) with a supporting **electric-blue** secondary (`--b*`) used only for functional accents (finish tiles etc.). There is **no palette switcher and no `PALETTES` array** - do not reintroduce swappable accents. Light ground: `--color-bg #F4F3F0`, `--color-surface #FFFFFF`, cool-neutral ramp. Big accent surfaces (Today hero card, split card) are a **solid coral fill** (no gradients).

**Dark mode** (`S.dark`, Settings toggle) stays: `body.dark` flips bg/surface/text, inverts the seed ramp step-for-step (so any `accent-200 bg + accent-800 text` pair stays readable), and mutes the accent so big bands don't glare. It reuses the same coral/blue seeds - it is a theme, not a palette.

Fonts: heading + all big numbers = **Barlow Condensed** (condensed sport display, weight 700), body = **Figtree**. Global rule `[style*="--font-heading"]{font-weight:700; letter-spacing:0.005em}`. Barlow Condensed is deliberately narrow - don't add tight negative tracking back.

## Verify an edit

Extract the inline script and syntax-check with node:

```bash
cd "C:/Users/el0821/Desktop/Setti" && node -e "const fs=require('fs');let h=fs.readFileSync('index.html','utf8');let m=h.match(/<script>([\s\S]*?)<\/script>/);fs.writeFileSync(process.env.TEMP+'/setti.js',m[1]);require('child_process').execSync('node --check \"'+process.env.TEMP+'/setti.js\"',{stdio:'inherit'});console.log('OK')"
```

## Run / preview

Static file — serve and open in a real browser (localStorage works there; the in-app browser pane treats `file:`/`data:` URLs as static snapshots that block JS/localStorage and flake on clicks):

```bash
cd "C:/Users/el0821/Desktop/Setti" && (python -m http.server 8777 >/dev/null 2>&1 &)
```

Then open `http://localhost:8777/index.html`. Use the browser device toolbar (F12) for a mobile/touch view. The **camera** needs an HTTPS origin + permission, so it only works on the deployed URL (or a real phone), not `file:`/localhost snapshots. If the week strip looks empty after a demo-seeding change, run `localStorage.removeItem('setti.log')` in the console and reload.

## Deploy

Public repo `lehtisenelmeri/Setti` on GitHub; **GitHub Pages** serves `main`/root at **https://lehtisenelmeri.github.io/Setti/**. Pushing to `main` auto-rebuilds (~1 min). (Vercel project creation is blocked for this account's token — Pages is the deploy path.)

## Design guardrails (keep it from looking AI-generated)

The app was deliberately de-"AI-templated". Hold this line; don't regress toward the generic look:

- **No fake device chrome.** No status bar, notch/island, "9:41", or home indicator. Ever. (See Chrome / safe areas.)
- **One brand colour (coral). No palette switcher, no `PALETTES`.** Commit to it. Dark mode is a theme toggle, not a colour picker.
- **Pills are rationed.** Full `borderRadius:'999px'` is reserved for genuinely circular things (avatars, dots, icon buttons) and the **one** sanctioned rectangular pill: the active-tab chip. Every other button / chip / badge / card uses a **moderate radius** (`~10-12px`, or the `--radius-*` tokens). Some elements may be squarer still. Don't pill-ify new buttons by default.
- **No eyebrow-label spam.** Avoid the reflex of putting a tiny **ALL-CAPS, wide-letter-spaced** micro-label over every section/stepper. `textTransform:'uppercase'` + `letterSpacing` was stripped from ~40 spots. If a section truly needs a name, use a plain, normal-case label (heading font, no tracking). One typographic decision, not five.
- **Icons, not emoji, in the UI.** No 🔥 / 💪 / ⚡ etc. Draw them as inline SVG in the same stroke/fill family as the other icons. The streak flame is `flameSvg(size, color)` off the shared `FLAME_PATH` (see Brand); the camera torch is a hand-drawn bolt path. `svg(attrs, inner)` is the builder.
- **Check new marks at real size.** Anything that becomes an icon or badge gets looked at at 40px and 24px, not just at hero size. That is where over-detailed shapes collapse.
- **Solid surfaces, not glass.** No `backdrop-filter`/blur bars.
- **Restrained motion.** Two signature animations only: the completion **check-stamp** (`stampIn` / `#strip-sticker-*`) and the day-detail **bloom** (FLIP from the tapped pill). Don't sprinkle pop/spring scale-ins on every element (the workout-bubble pop was removed).
- **Handle the unhappy path.** Destructive actions confirm first (`viewConfirm`); persistence has export/import; `load()`/save wrap `localStorage` in try/catch. Edge-state handling is what separates "product" from "demo".

## Conventions

- Match the surrounding style: object-literal inline styles, terse comments, `var()` tokens for all colors/radii/shadows.
- Follow the **Design guardrails** above for any new UI (no fake chrome, one brand colour, rationed pills, no eyebrow labels, SVG not emoji, restrained motion).
- No frameworks, no build, no new dependencies. Keep it one self-contained file.
- UI copy says "Split"; code keeps `PLANS`/`planId`/`viewPlans`/overlay `'plans'`.
- **Never use an em dash (`—`) anywhere in this app** (copy, comments, data). Use a period, comma, or ` · ` instead. Hyphen `-` for ranges (`8-10 reps`).
