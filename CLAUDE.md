# Fit Daily Routines — CLAUDE.md

Personal fitness PWA ("Borrowing Time, FMNV" in the header). One self-contained `index.html`, vanilla HTML/CSS/JS, no build step, no frameworks. Used daily on an iPhone (Chrome) as a home-screen shortcut.

- Live: https://kevinshieldsgrp.github.io/Fit-Daily-Routines/
- Repo: https://github.com/kevinshieldsgrp/Fit-Daily-Routines (deploys from `main`, live ~60s after push)
- The file to edit is always `index.html` at the repo root.

## Source of truth (read this first)

Work from the checked-out repo, never from a pasted or cached copy. Two regressions happened in chat sessions because a fix was layered onto a stale copy of the file and silently reverted earlier work. Before editing: `git pull`, confirm `git status` is clean. After editing: review `git diff` to confirm only intended lines changed.

## Deploy

Commit and push to `main`. GitHub Pages serves it. No build. Test on the live URL on the phone, because localStorage, Google Fonts, audio and touch behavior do not work properly in a desktop preview. Do not ask the user to change the shortcut URL (no `?v=N`).

**Bump `APP_VERSION` on every change to `index.html`** (format `YYYY-MM-DD.N`, near the end of the script). The auto-update block (`checkForUpdate`) fetches the live `index.html` on open and whenever the app returns to the foreground; if the live `APP_VERSION` differs, it refreshes the cache and reloads. It only reloads on the home screen with no timer running. If the version isn't bumped, the phone may keep showing the old copy for a while.

## How the user wants me to work

- Concise. No apologies; fix it and say what changed.
- If a request is ambiguous, ask one clarifying question before acting.
- Do not change anything that wasn't asked for without asking first.
- Batch related tool calls. For complex choices, show a short pros/cons table.
- Small cosmetic choices: decide and mention them. Significant ones (new dependency, storage schema change, architecture shift): ask first.
- Finish with a short summary: what changed, which functions/sections, what to check on the phone.
- The user's longer-term app-building standard (TypeScript front end, Python back end, strict tooling, tests, PR per feature) applies to new production projects. It does NOT apply to this single-file app unless the user decides to migrate it. Do not add tooling, frameworks or dependencies here without asking.

## Tech and constraints

- Vanilla JS, localStorage for all state, Web Audio API, SpeechSynthesis, Pointer Events. Fonts: Bebas Neue and DM Sans from Google Fonts. Dark theme, mobile-first, touch-optimized.
- Edit with targeted replacements (Edit/str_replace); the file is ~3,700 lines. After every edit, check syntax: extract the `<script>` block and run it through `new Function(...)` in node, and `grep -n` to confirm placement.
- iOS gotchas:
  - AudioContext and SpeechSynthesis must be created/resumed inside a direct user gesture (e.g. `toggleTimer()`, `startRoutine()`).
  - The iPhone silent switch blocks WebAudio regardless of volume.
  - Screen wake lock is faked with a 40Hz, gain 0.003 oscillator plus an 8s heartbeat to revive a suspended AudioContext.
  - HTML5 drag-and-drop does not work on iOS; use pointer events + `elementFromPoint` (see `attachBodyDragHandlers`).
  - Clearing Chrome cache wipes Strava tokens and forces re-auth.

## Routines (data lives in the `ROUTINES` object in the script)

Each routine: `{ name, label, subtitle, color, exercises: [{ name, detail, timer, instructions, swingFocus? }] }`. `timer` is seconds; `0` means no timer.

| Key | Card | Style |
|---|---|---|
| `hip` | Hip & Leg | Timed, auto-advance |
| `golf` | Golf Flex | Timed, auto-advance, Swing Focus panel per exercise |
| `body` | Total Body | Gym checklist. No timer. Tap check circle to complete, drag handle to reorder (order saved in `body_order`), FINISH WORKOUT logs the session. Last item "End with Bend Routine" links to Bend (`isBendLink`) |
| `glutes` | Glutes | Timed, auto-advance, same shell as Bend. Exercises named "Exercise 1/2/3" for now; user is adding his own coaching tips himself |
| `bend` | Bend | Timed stretch flow |

Timed flow: `openRoutine` → `startRoutine` → `showExercise(true)` → `toggleTimer` counts down (beeps in last 5s, ding + vibrate at 0) → `nextExercise` after a short pause → `showDone` on the last one. `showDone` calls `incrementCompletion`, `updateCategoryStreak('streak_<key>')` and `autoCheckExercise` (auto-checks the AM Exercise habit).

Only `type === 'body'` takes the checklist path in `openRoutine`, `goToList`, `showDone` and `restartRoutine`. Everything else uses the generic timer engine.

## Checklist: adding a new routine

Use `glutes` as the worked example (search for `glutes`). Touch every one of these or the routine will half-work:

1. CSS vars `--<key>-primary` and `--<key>-light` in `:root`
2. Streak pill CSS: `.streak-pill.s-<key>` (base, `:not(.zero)` border, count color)
3. Card CSS: `.routine-card.<key>`, `.<key> .card-accent`, `.card-tag`, `.card-meta-item`, `.card-arrow`
4. Helper classes: `.c-<key>`, `.bg-<key>`, `.bg-<key>-soft`, `.fill-<key>`
5. Home streak pill markup `#spill-<key>` / `#spill-<key>-count`
6. Home routine card markup with ids `<key>CountWeek`, `<key>CountPrev`, `<key>CountTotal` and `onclick="openRoutine('<key>')"` (card order on screen is DOM order)
7. The `ROUTINES.<key>` entry
8. Color maps in JS: `COLOR_VAR` (openRoutine), `exColorVar` (renderExerciseList), `softBgMap` (showExercise), and `colorVars` / `bgColors` / `icons` (showDone)
9. `loadStats()` routine array
10. `renderStreakGrid()` categories array

## Home screen layout (top to bottom)

Weekly Goals (editable, with its own weekly streak) → Streak grid (Hip, Golf, Strava, Strength, Bend, Glutes, Fuel) → Routine cards: Fuel, Bend, Hip & Leg, Golf Flex, Total Body, Glutes → Strava card (activity list, 20 most recent, tap opens Strava app).

Screens are `div.screen` elements toggled by `showScreen(id)`: `home`, `list`, `exercise`, `done`, `habits`, `goalsEditor`, `fuel`.

The Water and Habits home tiles (and the "Featured" section) were removed on 2026-10-17. The `habits` screen and its JS still exist but nothing on the home screen links to it any more.

## Fuel (food + water log)

Not a routine: the Fuel card calls `openFuel()`, not `openRoutine`. Code lives in the `FUEL` JS block (`FUEL_MEALS`, `getFuelDay`, `saveFuelDay`, `renderFuelScreen`, `renderFuelCard`, `getFuelStreak`). Stored in `food_YYYYMMDD` as `{ water, meals: { breakfast, lunch, dinner, snacks: [{ text, at }] } }`; old water-only records still load via `normalizeFuelDay`. A streak day = breakfast, lunch and dinner logged AND water >= `WATER_GOAL` (10). The streak is computed from stored days (not `updateCategoryStreak`), so back-filling earlier days via the ‹ › date buttons works. Recent items (`getFuelRecents`) are one-tap chips per meal. Each chip has a small × that hides it (stored lowercase per meal in `fuel_recents_hidden`; logged history is untouched). Typing a hidden food again un-hides it. Entry text is rendered with `textContent`, never `innerHTML`.

## Habits

Three habits in `HABITS`: `exercise` (AM Exercise, auto-completes when any routine finishes), `mindful` (intermittent fast 8pm–10:30am), `espanol` (Duolingo lesson). Habits streak updates when all three are done.

## Streaks and stats

- Per-category daily streaks use `updateCategoryStreak(key)` with keys `streak_<name>_count`, `_last`, `_best`. Active only if last date is today or yesterday (`readCategoryStreak`).
- Completion counts: `completions_<type>` (all time) and `completions_<type>_<weekKey>`.
- Week key: `getWeekKey()` returns the Monday of the week as `week-YYYYMMDD`; previous week via `getWeekKey(-1)`.
- Daily keys: `food_YYYYMMDD` (Fuel: water + meals), `habits_YYYYMMDD`.
- Goals: `goals_template`, `goals_progress_<weekKey>`, `goals_streak`.
- Body order: `body_order`. Strava: `strava_access_token`, `strava_refresh_token`, `strava_expires_at`, `completions_strava*`.
- Adding, removing or reordering UI never touches stored data, because keys are date or week based, not position based. Do not rename existing keys; that would orphan the user's history.

## Strava

- OAuth via AWS: API Gateway → Lambda `strava-oauth` (us-west-2) → client secret in SSM Parameter Store `/strava/client_secret`. Client ID `227459`, scope `read,activity:read_all`. Never put the client secret in this repo.
- Activities load on every open and on pull-to-refresh, using `fetchWithTimeout` (12s) and `cache: 'no-store'`.
- The Strava streak counts any activity logged today. `updateStravaStats` must use `getTodayKey()` (a past bug referenced an undefined `todayKey` inside a try/catch, which silently skipped the streak update).
- Walk/Hike counts are computed in `updateStravaStats` and rendered by `renderWalkDisplay`. The Walk card and pill were removed from the UI; some Walk CSS and helpers remain as dead code.

## Known state and loose ends

- Glutes exercises are named "Exercise 1/2/3" with estimated timers (120s / 90s / 100s). The user will rename and add his own tips.
- Dead Walk CSS and helpers remain. Remove only if asked.
- Next planned learning topic: AWS (S3 → CloudFront → Route 53 → DynamoDB for cloud sync → Cognito). Not started in code.
- The home streak pill row now has seven pills; check it still fits on a small phone after any further additions.

## Content boundaries

Exercise copy stays standard fitness and mobility content (muscle engagement, breathing, pacing, sets/reps/holds).
