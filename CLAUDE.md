# CIRRA — Agent Development Guide

## Project Overview

CIRRA is a single-file HTML/JS Progressive Web App built for women. It has two co-equal core features:
- **Gym/workout tracking** with muscle stress scoring and progressive overload
- **Menstrual cycle tracking** with phase-specific training guidance

The app is branded **TRK** in some in-app UI text. The brand green is `#95bc2b` — this must never shift to purple or any other colour.

Target users: women who train. Key differentiator: cycle-aware training guidance that competitors (Flo, Clue, Natural Cycles, Whoop) charge significant annual fees for. CIRRA offers it free.

---

## File Structure

The entire app is a **single HTML file**. There is no build step, no framework, no bundler.

```
gymtrack_v10-9-15.html   ← the entire app
```

All edits go to this one file. When making changes:
1. Copy the file to a working path before editing
2. Use `sed -n 'START,ENDp'` for targeted line-range reads on large sections
3. Use `grep -n` with specific patterns to locate functions before editing
4. Always run a JS syntax check after edits:
   ```bash
   python3 -c "
   import re, open
   scripts = re.findall(r'<script[^>]*>(.*?)</script>', open('file.html').read(), re.DOTALL)
   open('/tmp/check.js','w').write('\n'.join(scripts))
   "
   node --check /tmp/check.js
   ```
5. Output always goes to `/mnt/user-data/outputs/gymtrack_v10-9-15.html`

---

## Architecture

### Views / Screens

All screens are `<div id="v-*" class="view">` elements. Navigation via `go(viewName)`.

| View ID | Screen |
|---|---|
| `v-home` | Homepage — cycle phase, readiness, training load, muscle breakdown |
| `v-log` | Active workout logger |
| `v-build` | Workout builder / saved workouts |
| `v-lib` | Exercise library |
| `v-run` | Cardio/run logger |
| `v-cycle` | Full cycle tracking screen |
| `v-hist` | Workout history |
| `v-prog` | Progress tracking per exercise |
| `v-profile` | User profile |
| `v-settings` | App settings |
| `v-legal` | Legal/about |
| `v-session-plan` | Today's suggested session |

### Global State Variables

```js
let hist = []              // all workout + run entries (newest first)
let periodHistory = []     // period start date strings, newest first
let weightHistory = []     // [{date, kg}]
let prof = {}              // user profile
let goals = []             // training goals
let customExs = []         // user-created exercises
let cycLen = 28            // cycle length in days
let cycDay = 1             // current cycle day (1-indexed)
let trackCycles = false    // whether cycle tracking is enabled
let ac = '#95bc2b'         // accent colour (brand green — do not change default)
let appTheme = 'light'     // 'light' | 'dark'
let woMode = 'gym'         // 'gym' | 'cardio'
let curEx = []             // exercises in active workout
let gymExs = []            // gym exercises buffer during logging
let woStartTime = null     // Date.now() when workout started
let editingHistIdx = null  // hist index being edited (null = new workout)
```

---

## Storage

Three-layer persistence strategy (in order of priority):
1. **localStorage** — primary, keyed via `LS` object
2. **sessionStorage** — fallback for active workout state
3. **IndexedDB** — backup sync, recovery if localStorage is evicted

### Storage Keys (`LS` object)

```js
const LS = {
  hist:           'gt_hist',           // Array<HistEntry>
  prof:           'gt_prof',           // UserProfile
  goals:          'gt_goals',          // Array<Goal>
  nextGoalId:     'gt_nextGoalId',     // number
  weightHist:     'gt_weightHist',     // Array<{date, kg}>
  periodHist:     'gt_periodHist',     // Array<string> date strings
  trackCycles:    'gt_trackCycles',    // boolean
  cycLen:         'gt_cycLen',         // number
  settings:       'gt_settings',       // {restTimer, units, liteMode}
  ac:             'gt_ac',             // accent hex colour
  onboarded:      'gt_onboarded',      // boolean
  theme:          'gt_theme',          // 'light' | 'dark'
  savedWorkouts:  'gt_savedWorkouts',  // Array<SavedWorkout>
  curEx:          'gt_curEx',          // active workout in progress
  editingHistIdx: 'gt_editingHistIdx', // hist index being edited
  woStartTime:    'gt_woStartTime',    // workout timer start
  customExs:      'gt_customExs',      // Array<CustomExercise>
  checkins:       'gt_checkins',       // daily check-in log
};
```

One key lives outside the `LS` object: `gt_lastDci` holds the date string of the last
daily check-in. It is written directly (localStorage + sessionStorage) at submit and
checked by `hasTodaysCheckin()` — a fast guard that survives even if the main
checkins blob fails to load. `loadState()` wraps every key parse in its own
try/catch (`_safe`) so one corrupt key cannot abort the rest of the load.

Use `_lsGet(key)` and `_lsSet(key, value)` — never call `localStorage` directly. These functions handle try/catch and sessionStorage fallback.

`saveState()` — call this after any mutation to `hist`, `prof`, `goals`, `periodHistory`, `weightHistory`, `customExs`, or settings.

`loadState()` — called once at init. Includes auto-recovery of custom exercises from history.

---

## Data Structures

### Workout Entry (`hist` array, `type: 'wo'`)

```js
{
  type: 'wo',
  date: 'YYYY-MM-DD',       // local date string
  mode: 'gym',              // 'gym' | 'cardio'
  dur: 45,                  // duration in minutes
  name: null,               // optional custom session name
  mood: null,               // optional post-workout mood
  checkin: null,            // pre-workout check-in {mood, syms:[]} — prefilled from daily check-in
  ex: [
    {
      id: 12,               // exercise ID from EX array (may be null for custom)
      n: 'Back Squat',      // exercise name
      sets: [
        { w: 80, r: 10 },   // weight (kg) + reps
        { r: 15 },          // bodyweight — reps only
        { time: 45 },       // time-based — seconds/minutes
        { dist: 5, time: 30 }, // distance + time (cardio)
        { height: 60, r: 8 },  // box height (cm) + reps — plyo
        { incline: 2, dist: 5, time: 30 } // treadmill incline %
      ]
    }
  ]
}
```

### Run Entry (`hist` array, `type: 'run'`)

```js
{
  type: 'run',
  date: 'YYYY-MM-DD',
  dur: 30,                  // minutes
  dist: 5.2,                // km
  rpe: 7,                   // rate of perceived exertion 1-10
  notes: ''
}
```

### User Profile (`prof`)

```js
{
  name: 'Alice',
  dob: 'YYYY-MM-DD',
  heightCm: 165,
  weightKg: 62,
  goal: 'strength',         // 'strength' | 'fat_loss' | 'endurance' | 'general'
  units: 'metric',          // 'metric' | 'imperial'
  restTimer: 90,            // seconds
  liteMode: false
}
```

---

## Core Constants

### Muscles

```js
const MUSCLES = ['Chest','Back','Shoulders','Legs','Glutes','Core','Biceps','Triceps'];
// Detailed mode (Settings → "Detailed muscles"): Legs → Quads/Hamstrings/Calves, plus Forearms
const MUSCLES_DETAILED = ['Chest','Back','Shoulders','Quads','Hamstrings','Calves','Glutes','Core','Biceps','Triceps','Forearms'];
```

`computeStress(detailed)` takes an optional boolean — `true` expands Legs by movement pattern
(`expandMuscle()`: squat/lunge→Quads, hinge/RDL/leg-curl→Hamstrings, calf/tibialis→Calves,
run/jump→all three) and adds Forearms from grip-heavy names (`gripLoadsForearms()`).
Only the home stress table uses detailed mode (`appSettings.detailedMuscles`); every other
caller (suggestions, cycle, dials elsewhere) uses the default 8 base groups — keep it that way.

### Muscle Decay Rates (per day, exponential)

```js
const MUSCLE_DECAY = {
  'Biceps':    0.50,   // fully recovered in ~2 days
  'Triceps':   0.50,
  'Shoulders': 0.50,
  'Core':      0.50,
  'Chest':     0.30,
  'Legs':      0.30,
  'Glutes':    0.30,
  'Back':      0.25,   // slowest — recovers over ~3 days
  // Detailed-mode submuscles
  'Quads':      0.30,
  'Hamstrings': 0.30,
  'Calves':     0.50,
  'Forearms':   0.50,
};
```

### Cycle Phases (`PHASES` array, index 0–3)

| Index | Phase | Colour | Character |
|---|---|---|---|
| 0 | Menstrual | `#E24B4A` | Rest, light movement |
| 1 | Follicular | `#EF9F27` | Building strength |
| 2 | Ovulation | `#95bc2b` | Peak strength |
| 3 | Luteal | `#7F77DD` | Moderate, endurance |

Each phase object contains: `n`, `short`, `col`, `bg`, `brd`, `str`, `end`, `en`, `read`, `intensityPct`, `intensityNote`, `loadLabel`, `adv`, `recs`, `fuel`.

---

## Key Functions

### Date Utilities
```js
parseLocalDate(s)          // 'YYYY-MM-DD' → Date (always local, avoids UTC off-by-one)
localDateKey(d)            // Date → 'YYYY-MM-DD'
daysAgo(n)                 // returns localDateKey for n days ago
```
**Always use `parseLocalDate()` — never `new Date(dateString)` directly.** The UTC/BST off-by-one bug was a hard-won fix.

### Exercise Lookup
```js
resolveExDef(e)            // resolve exercise def from hist entry {n, id}
findEx(idOrName)           // find in EX array by id or name
getPBForExercise(exName)   // returns estimated 1RM PB using Brzycki formula
exRawLoad(e)               // raw load for an exercise entry (weight × reps × ir²)
```

### Muscle Stress Scoring
```js
computeStress()
// Returns: { scores, lastTrained, exByMuscle, totalScore }
// scores: { Chest: 42, Back: 78, ... }  (0-100)
// Frequency-aware ceiling: 2.5× multiplier for new users → 1.5× for established
// Score 65+ = High, 35-64 = Med, 10-34 = Low, <10 = Fresh
```

Score labels and colours:
```js
stressLabel(p)    // 'High' | 'Med' | 'Low' | 'Fresh'
stressColor(p)    // hex colour per threshold
stressBadgeSty(p) // inline style string for badge
```

### Cycle Functions
```js
getCycBounds()             // returns [[start,end], ...] day ranges per phase
getCycPhaseIdx()           // current phase index 0-3
renderCycSvgChart()        // renders the cycle performance chart
renderCycToday()           // renders today's cycle card on home screen
renderCycPhases()          // renders all 4 phase cards
renderCycInsights()        // renders performance insights chart section
```

### Navigation & Rendering
```js
go(viewName)               // navigate to a view
renderHome()               // re-render homepage
renderDials()              // re-render training load dials
renderStressBlock()        // re-render muscle stress table
saveState()                // persist all state to storage
setTheme(mode, skipSave)   // 'light' | 'dark'
isLight()                  // returns true if light mode active
```

### Theme-Aware Colour Helpers
```js
tPrimary()    // primary text colour (adapts to light/dark)
tSecondary()  // secondary text
tTertiary()   // tertiary text
tMuted()      // muted text
```

---

## Muscle Stress Scoring — How It Works

The scoring system is **frequency-aware**, not just volume-based.

1. For each muscle, collect all historical sessions where that muscle was trained
2. Calculate `avgLoad` per session and average gap between sessions
3. Compute a **steady-state ceiling** — what accumulated load looks like if training continues at this exact frequency indefinitely: `ss = avgLoad / (1 - e^(-k × avgGap))`
4. **Variable multiplier**: starts at 2.5× for new users (physiological rationale: beginners operate at <40% 1RM, neural adaptation dominates), blends to 1.5× by session 12+
5. Blend between `simpleCeiling` and `steadyCeiling` based on session count (n≤4 = simple, n≥12 = fully steady-state)
6. Score 65 = at your own baseline. Score 100 = 1.5× above baseline (genuine overload)
7. Rest days always pull scores down via exponential decay

**Do not revert** the `ceilMult` variable back to a flat 1.5. The 2.5→1.5 blend is intentional.

---

## Cycle Chart — How It Works

`renderCycSvgChart()` renders an SVG performance chart in the Cycle tab.

- **Session performance score** per workout = `Σ(weight × reps × intensityRatio²) × sessionTypeWeight`
- **Session type weighting**: heavy lifting = 1.0, cardio = 0.6, pilates/yoga/mobility = 0.3, bodyweight/circuits = 0.75
- **Rolling average** across all past cycles per cycle-day (not just current cycle)
- **Buffer band** = ±1 standard deviation, filled per phase using phase colour at low opacity
- **Gap days** (no session logged) interpolated at 80% of surrounding values — creates natural dip not a null gap
- **McNulty ghost** (research reference line): full opacity with <3 sessions, 30% opacity at 3-8 sessions, hidden at 8+ sessions
- Ghost stroke adapts to light/dark mode via `isLight()`

Data quality thresholds:
- `hasEnoughData` = `totalSessions >= 3`
- `isEstablished` = `totalSessions >= 8`

---

## Exercise Library (`EX` array)

Each exercise object:
```js
{
  id: 12,
  n: 'Back Squat',
  m: 'Legs',              // primary muscle
  ms: ['Glutes','Core'],  // secondary muscles (optional)
  t: 'weight',            // 'weight' | 'time' | 'run'
  c: 'gym',               // category: 'gym' | 'run' | 'custom'
  anim: 'anim-squat',     // CSS animation class
  tags: ['Compound','Legs'],
  cols: ['kg','reps'],    // columns shown in set logger
  tip: '...',             // coaching cue
  s: ['Barbell'],         // equipment
  custom: true            // only present on custom exercises
}
```

Cardio exercises muscle mappings (important — do not revert):
- Spin Bike / Spin Bike Intervals → `m:'Legs', ms:['Glutes','Core']`
- Assault Bike / Assault Bike Intervals → `m:'Legs', ms:['Shoulders','Core']`
- Rowing Erg / Rowing Erg Intervals → `m:'Back', ms:['Legs','Core']`

Custom exercise filter logic:
```js
if(catF==='custom') return e.custom===true;
if(catF==='gym')    return e.c==='gym'||e.custom;  // custom exercises appear under gym too
```

---

## Common Bug Patterns

### Z-index / overlay blocking taps
Full-screen backdrop divs silently blocking UI are a recurring issue. If taps stop working on a screen, check for a `position:fixed` or `position:absolute` overlay with a high z-index that has been left open.

### UTC vs local date
Never use `new Date('YYYY-MM-DD')` — it parses as UTC midnight, which in UK BST (UTC+1) gives the previous day. Always use `parseLocalDate()` which explicitly constructs `new Date(y, m-1, d)` in local time.

### Duplicate function definitions
The file is ~11,000 lines. Before adding a new function, `grep -n "function functionName"` to confirm it doesn't already exist. Duplicates cause silent override bugs.

### Custom exercises not appearing in library
Two conditions must both be true: the exercise must be in `customExs` AND `mergeCustomExs()` must have been called to add it to the live `EX` array. `loadState()` handles this on boot including auto-recovery from history.

### Storage eviction (Safari)
The three-layer strategy (localStorage → sessionStorage → IndexedDB) exists because Safari aggressively evicts localStorage for non-installed PWAs. `navigator.storage.persist()` is called at `init()` to request persistent storage from the browser. Do not remove this call.

---

## Theming

Light/dark mode is toggled via `body.light-mode` class. CSS uses this class for overrides.

```js
setTheme('light')  // adds body.light-mode, updates stored preference
setTheme('dark')   // removes body.light-mode
isLight()          // check current mode in JS
```

Brand colours:
- **Brand green**: `#95bc2b` — accent, CTAs, active states. Never changes.
- **Menstrual**: `#E24B4A`
- **Follicular**: `#EF9F27`
- **Ovulation**: `#95bc2b` (same as brand green)
- **Luteal**: `#7F77DD`

Logo variants toggle via CSS class with `!important` to prevent specificity conflicts.

---

## What NOT To Do

- **Do not introduce a build step** — the app is intentionally single-file with no tooling
- **Do not add external dependencies** beyond the existing Fuse.js CDN import
- **Do not change `#95bc2b`** to any other default accent colour
- **Do not revert the frequency-aware stress ceiling** to the old `peakDayLoad` approach
- **Do not revert the `ceilMult` 2.5→1.5 blend** — it's physiologically justified for newcomers
- **Do not use `new Date('YYYY-MM-DD')`** — always use `parseLocalDate()`
- **Do not call `localStorage` directly** — always use `_lsGet()` / `_lsSet()`
- **Do not add a React rewrite** — modularisation is acceptable, full framework rewrite is not planned
- **Do not name competitors directly** in any marketing copy baked into the app

---

## Testing Approach

There is no automated test suite. Verification is manual and via simulation scripts.

For muscle stress algorithm changes, write a Python simulation:
```python
import math
# Simulate 3 profiles: Newcomer (3×/wk), Regular (PPL 4×/wk), Pro Athlete (5-6×/wk)
# Check that:
# - Newcomer oscillates High on session days, Med on rest — not pinned at 100
# - Regular shows muscle specificity (Chest High on push day, Low on pull day)
# - Pro athlete deload week shows arc: High → Med → Low → Fresh
```

For the cycle chart, verify:
- Ghost visible when no data (opacity 0.80)
- Ghost fades at 3+ sessions (opacity 0.30)
- Ghost gone at 8+ sessions (opacity 0)
- Phase band colours match PHASES array
- Gap interpolation produces 80% of surrounding values (not zero)

---

## Planned / In Progress

- **Capacitor wrapping** — iOS/Android app store distribution. The storage layer (`_lsGet`/`_lsSet`) is already abstracted for easy swap to `Capacitor.Preferences`
- **File modularisation** — splitting the single file into modules (no framework rewrite). Medium-term improvement
- **Trademark clearance** — CIRRA name has minor conflicts in unrelated categories (umbrellas, an ISP). Formal search recommended before full commitment. Backup name: Orbita

---

## Research Basis

Phase-appropriate training recommendations are grounded in published research:
- McNulty 2021 — strength/endurance performance across menstrual cycle phases
- Hackney 2017 — hormonal effects on exercise performance
- Wikström-Frisén 2017 — resistance training and menstrual cycle

Do not alter phase recommendations without a scientific basis. Exercise suggestions must be defensible, not arbitrary.

All work must be performed within `C:\Users\seankeane\Documents\Personal\v15` only.
Do not read from, write to, modify, or reference any files outside this directory.