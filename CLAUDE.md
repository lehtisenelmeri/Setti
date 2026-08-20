# Setti — project guide

Mobile workout-log + social app. Single self-contained file, vanilla JS, no build step, no dependencies.

## Files

- `index.html` — the entire app: markup, `<style>`, and one inline `<script>`. This is the only source file.
- `CLAUDE.md` — this guide.

Concept experiments / prototypes (e.g. marker mockups) live as standalone files in the scratchpad, **not** in the repo — the real app is `index.html` only.

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

`load(k, d)` reads, `applyPrefs()`/`setState` write. Persisted keys: `log`, `dark`, `unit`, `accent`, `rest`, `mark`, `uname`, `pfp`, `unameAt`. Everything else in `S` is session-only. "Reset demo data" (Settings) clears `log` back to `[]`.

## Data model

- **`EX`** — exercise catalog keyed by id (`bench`, `ohp`, …): name `n`, muscle group `g`, `lvl`, `sets` (base 2), `reps`, `cues`.
- **`PLANS`** — ready-made **splits**; each `{name, level, days, blurb, week:[7 × {name, ex:[ids]}]}`. `usePlan(id)` clones a plan's week into `S.week`. `createPlan()` seeds `PLANS.custom` (7 rest days, name "My Split") and opens the builder. NOTE: in the **UI** every "plan" now reads **"Split"** (tab, buttons, headings), but the **code identifiers stay `PLANS`/`planId`/`planName`/`viewPlans`/`S.overlay==='plans'`** — don't rename those.
- **`S.log`** — real logged sessions `{date:'YYYY-MM-DD', name, sets:[{id,g,kg,reps}], vol, mins, photo}` (`photo` = data URL or null), written by `saveSession()`. Drives Progress (calendar, muscle balance, Most-improved), the week strip, and your own posts in the Friends feed.
- `DAY_LBL`/`DAY_FULL`, `GROUPS`, `GROUP_COLOR` (**fixed, distinct per-muscle colours** `[light bg, dark text, solid dot]` — not tied to the accent palette), `HISTORY`/`SERIES` (demo fallbacks), `DEMO_LOG` (seeds all 7 weekdays of the current week).
- **Social (demo)** — `FRIENDS` (id, name, initials, accent, level, planName, streak, thisWeek/weekGoal, `week`), `FEED` (activity posts, same shape as `S.log` + `userId`), `_LIVE_NOW` (presence ids). `_ago(n)` builds a `YYYY-MM-DD` n days back.

## Done-day marker (check / bowtie) — replaced the old stickers

`S.mark` (`'check' | 'bowtie'`, Settings toggle) picks the completion marker. The old `STK` sticker set + `stickerNode` still exist in the file but are **no longer used** for done days.

- **`doneMark(size, extraStyle)`** — the filled marker. `check` = accent circle + white check; `bowtie` = a bare cute pink bow (loops + oval knot + crease lines + top-left highlight + splayed notched tails, `bowSvg`), no circle. Used on the week strip (with `id="strip-sticker-<date>"` so the finish hook can animate it), the Progress calendar corner (18px), and the day-detail corner (52px). `_posOnly()` strips non-positional styles for the bow (which has no circle to take a border).
- **`holsterMark(size)`** — the empty "holster" shown on scheduled-but-not-done days: `check` = dashed accent ring + ghost check; `bowtie` = a **dotted** bow outline. On completion the marker **stamps** into the holster (finish hook in `render()` animates `#strip-sticker-<date>` via the `stampIn` keyframe: drop in ~1.9× → squash → settle).
- **`saveSession`** sets `S.slapDay` to the finished date + forces `tab:'today'`; the `render()` post-hook animates the stamp, then clears `slapDay`.

## Key features / helpers

- **Today** (`viewToday`) — keyed off the **real weekday** via `todayIdx()` (Mon-first), not the first training day. Rest day → "Rest day" card with an "Add a workout" button. Week strip = one bar; done days show `doneMark`, scheduled-not-done show `holsterMark`, rest days show a faint "Rest", today's cell has an accent pill label + accent top bar. Tapping a logged day opens the day-detail sheet (passes `S.dayFrom` click coords for the FLIP).
- **Day detail** (`viewDayDetail`) — centered card of that date's session(s): numbered sets `kg × reps`, plus the photo (whole image) if attached. FLIP-animates from the tap point (`S.dayFrom`).
- **Plans → "Split"** (`viewPlans`) — accent "Active split" card + "Your week" well; "Customise split" opens the builder, "Change split" opens `viewPlanPicker`, "Create split" → `createPlan()`.
- **Builder** (`viewBuilder`) — day tabs + a bar with the day name and a **"Set as rest day"** toggle (`setRestDay`). Editable **split name** input in the header (updates `PLANS[S.planId].name`); "Save" closes. Slot cards: **press-and-drag reorder** via the grip (`startExDrag`, pointer events, live-shifts siblings, `id`-based) — the old tap-based `tapEx`/`exPick` was removed. Each slot has a vertical **set stepper** (`setSets`, clamps 1–10) and corner **info**/**remove** buttons. The "Add by muscle group" drawer collapses via its grip (`S.drawerOpen`); drawer group headers use a slim vertical colour bar.
- **Workout** (`viewWorkout`) — active set screen: **weight is a typed numeric field** (mutates `S.w.kg` on `onInput`, no re-render), **reps use −/+ steppers**. Rest screen keeps the countdown ring + `spinField` mini-editors for the just-logged set. Finish screen: stats + **Add a photo** (opens the camera) + Save. `spinField` still exists for the rest-card edits only.
- **Camera** (`openCamera`, imperative overlay) — live `<video>`, opens in **selfie/front** mode with a **flip** button (mirrored preview + capture), **flash/torch** toggle (hidden if unsupported), **0.5× / 1× / 2× zoom buttons** (hardware zoom when available, digital crop otherwise), shutter, and retake/use. Preview + capture are locked to a **portrait 3:4 frame**. Falls back to the native picker (`pickPhotoFallback`) if `getUserMedia` is missing or denied. `readPhoto(file, cb)` downscales imports to ≤1080px JPEG.
- **Friends** (`viewFriends`) — friend avatar strip (live "at the gym" ring via presence, `isLive`), **gym invites** (see below), and an **activity feed**: friends' `FEED` posts + your own photo'd sessions (`userId:'me'`), newest first, **dropped after 4 days**. A post with a photo shows the whole image with the workout name overlaid + "See workout"; without a photo, a short preview + "Expand" (both inside the inner box).
- **Friend profile** (`viewFriendProfile`) — focus is their **last workout** (day-detail style: date label, big name, `min · kg`, plain-numbered set rows with a divider **only where the muscle group changes**, photo). 🔥 streak badge on the avatar; **"See plan"** opens `viewFriendPlan`.
- **Gym invites** — group invites `{id, host, time('now'|'HH:MM'), invitees:[{id, status}]}` in `S.invites` (`'me'` = you). `viewInviteCompose` multi-selects friends + Now/time → `sendInvite(ids, when)`. Incoming (`myEntry` pending/suggested) shows **host + already-accepted people** ("X and Y invited you to the gym", `joinNames`) with Accept/Decline/Suggest-time (`respondInvite`). Outgoing (host `'me'`) shows the invitee stack + going/pending/declined status. Seam-ready for a real `/api/invites`.
- **Settings** (`viewSettings`) — editable **profile** (`viewEditProfile`: photo anytime, username once per 7 days via `unameAt`; `selfAvatar` shows `S.pfp` or the initial). Accent swatches, dark mode, units, **rest timer** (`restStepper`, min/sec), **done marker** toggle (Check / Pink bowtie).
- **Social data seam** — `Social.friends()/feed()/profile(id)/sessionsFor(id)/presence()/setPresence(on)` all return Promises over the seed arrays; `loadSocial()` caches into `S.social`. To go live, swap each body for `fetch('/api/...')` — the views never change. **Presence = who's mid-workout**: `startWorkout()` calls `setPresence(true)`, `saveSession()` `setPresence(false)`.
- Date helpers near `firstTrain`/`todayIdx`: `ymd`, `parseYmd`, `weekStart`, `doneThisWeek()`.

## Re-render note

The 250ms interval only **full-renders during rest** (for the countdown ring); during active logging it just patches `#wClock` text, so constant repaints don't eat taps. `scrubbing` still freezes it while a `spinField` is open.

## Chrome / safe areas

No fake status bar / notch / home indicator (removed). `header()` pads with `env(safe-area-inset-top)`, the tab bar with `env(safe-area-inset-bottom)`. Mobile media query uses `100dvh`.

## Theming

CSS `var()` seeds live on `body` (not `:root`); `applyPrefs()` toggles `dark` and `pal-*` classes on `body`. Palettes in `PALETTES`: Clay, Forest, Ocean, **Pink** (`pal-pink`, was Graphite — saved `'graphite'` auto-migrates to `'pink'`). Fonts: heading = **Barlow Condensed** (`--font-heading`), body = Figtree. Global rule `[style*="--font-heading"]{font-weight:700}` forces inline heading elements heavy.

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

## Conventions

- Match the surrounding style: object-literal inline styles, terse comments, `var()` tokens for all colors/radii/shadows.
- No frameworks, no build, no new dependencies — keep it one self-contained file.
- UI copy says "Split"; code keeps `PLANS`/`planId`/`viewPlans`/overlay `'plans'`.
