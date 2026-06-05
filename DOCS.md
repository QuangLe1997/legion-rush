# LEGION RUSH — Technical Documentation

> Read this file to understand the **entire game without reading code**. Want to change difficulty / scoring / stages? §5, §6, §7, §15 are enough.
> Everything lives in `index.html` (zero-build, single file). All balance numbers are centralized in **§14 (`CONFIG` + `STAGES` + `genStage`)**.

---

## 0. FEATURE STATUS (read first)

| Feature | Status | Where (function / marker) |
|---|---|---|
| Core loop: auto-run, steer crowd, gates, slam barriers | ✅ | `step()`, `applyEvent()`, `resolveBarrier()` |
| **Endless** — 8 authored stretches, then procedural sectors forever | ✅ | `STAGES`, `genStage()`, `curStage()` |
| **Seamless** continuous run (no stop / confirm between stretches) | ✅ | continuous `S.dist`, `S.stageStart`, `resolveBarrier()` |
| **Carryover** crowd (persists across stretches) | ✅ | `setupStage()` (no crowd reset), `resolveBarrier()` attrition |
| **Crowd speeds up every stretch** | ✅ | `setupStage()` → `S.speed` ramp |
| 3D crowd = neon **army-men** via merged-geometry `InstancedMesh` (1 draw call) | ✅ | `buildSoldierGeometry()`, `boot()`, `updateCrowdInstances()` |
| Render ≤ `RENDER_CAP` soldiers; true count shown on HUD | ✅ | `CONFIG.RENDER_CAP`, `updateCrowdInstances()` |
| Gates: ×mul / +add / −sub / **÷div**, pick nearest lane | ✅ | `STAGES[].ev` (t:'G'), `applyGateOp()`, `opLabel/opColor` |
| Hazards: saw / spin / wall, %-cull, **some oscillate side-to-side** | ✅ | `STAGES[].ev` (t:'H'), `hazardX()`, `applyEvent()` |
| Pickups: floating +bonus | ✅ | `STAGES[].ev` (t:'P') |
| Barrier = **clash with a dynamic toll** (out-grow it to smash through) | ✅ | `stageToll()`, `resolveBarrier()` |
| No-loss streak score multiplier | ✅ | `S.stageClean`, `S.streak` |
| 3 difficulty modes (Easy/Normal/Hard) | ✅ | `CONFIG.BASE_START / RUN_SPEED / HAZARD_MULT / TOLL_FRAC` |
| Steering — drag (touch/mouse) + keyboard, **screen-correct direction** | ✅ | `pmove()`, `keydown` handler |
| Juice: particles, screen shake, slam flash, count pop, haptics | ✅ | `burst()`, `screenShake()`, `slamFX()`, `popCrowd()` |
| Biome crossfade (scenery shifts between stretches) | ✅ | `biomeTarget`, `render()` lerp |
| Synthwave music that builds with crowd size · WebAudio SFX + mute | ✅ | `Audio.startMusic()`, `Audio.sfx()` |
| Bloom + low-FX degradation + DPR cap · CDN-fail guard | ✅ | `UnrealBloomPass`, `applyLowFX()`, top-level `try/catch` |
| localStorage best score + biggest legion + furthest stage per difficulty | ✅ | `Save`, `gameOver()`, `loadBests()` |
| First-run onboarding · in-engine balance sim · selftest/`?shot=` hooks | ✅ | `#how`, `?shot=sim`, `?shot=selftest`, `applyShot()` |

**Notes:** the game is **endless** — there is no "win"; you run until a barrier out-tolls you or your legion hits 0. Stretches 1–8 are hand-authored; stretch 9+ are procedurally generated (`genStage`). "Turns" (curved track) remain deferred (straight track for steering reliability) — §15.5.

---

## 1. Overview & Concept

### Market context (research 2026-06-05)
Crowd-runner / "count master" is a proven hybrid-casual hook (Count Masters, Crowd City). The web/H5 versions are ad-heavy, flat stickman art, and shallow. **LEGION RUSH's hook:** a neon-synthwave, instant-play, **endless** crowd-runner with a real army of little soldiers, a **carryover legion** that snowballs into the millions/billions, ÷ traps & swinging sawblades, and a track that **never stops and keeps speeding up** — a score-attack arcade runner.

### Concept
- **Genre:** 3D endless crowd-multiplier runner (single player).
- **Core loop (1 sentence):** *"Steer your whole legion down a synthwave track, take ×/＋ gates (avoid ÷/− and swinging sawblades) to out-grow each barrier's toll, smash through, and keep running ever faster forever."*
- **Lose:** reach a barrier with `crowd ≤ toll`, or your legion hits 0 mid-stretch. **No win** — it's endless; the goal is the furthest stretch + biggest score.
- **Fantasy:** command an army that snowballs from a handful to billions; the screen-shaking slam at every barrier; the rising-speed gauntlet.
- **2D or 3D:** **3D (Three.js)** — depth, a trailing camera, and a surging army of soldiers. The crowd is **one merged-geometry `InstancedMesh`** (single draw call) capped at `RENDER_CAP` representatives; the true count is shown on the HUD.
- **Layout (mobile #1):** **A — Mobile-frame** (`width:min(100vw,480px)`, centered + letterbox on desktop). No responsive breakpoints.

## 2. Tech stack
- **Render:** Three.js **r0.169** via importmap CDN. `EffectComposer` + `UnrealBloomPass` (strength 0.55, threshold 0.82) + `OutputPass`. `BufferGeometryUtils.mergeGeometries` builds the soldier. Each addon is wrapped in try/catch (graceful fallback: no bloom / capsule soldiers).
- **Build:** zero-build, single `index.html`, ES module inline.
- **Storage:** `localStorage` (`legion-rush.*`). **Audio:** WebAudio synth (no files).

## 3. State machine
`mode ∈ menu → playing → (paused) → dead → gameover → menu/playing` (+ `frozen` for screenshots; `sim*` for the bot sim).
The run is **continuous**: `playing` advances `S.dist` forever; clearing a barrier seamlessly starts the next stretch (no stop). A barrier failure or crowd-0 → short FX → `gameOver()`.

## 4. Gameplay & rules
- **Field:** straight track, half-width `TRACK_HALF=6`. Crowd centroid `S.cx` clamps to `±5.5`. Crowd auto-advances `S.dist` at `S.speed` (×`CHARGE_MULT` in the last 16 units before a barrier).
- **Steering (direction is screen-correct):** drag (relative to drag start) or **← / →** (A/D). Because the camera trails behind looking +Z, screen-right = world −x, so the drag/key delta is inverted internally — **drag right ⇒ the legion goes right on screen.** (This was a v1 bug; see §16.)
- **Gate selection:** a row has 2–3 lane options; when crossed, the option whose lane is **nearest the centroid** applies to the whole legion (`chooseGateSide`). 2 lanes → x=±TRACK_HALF/2; 3 → −1/0/+1 × TRACK_HALF·⅔.
- **Hazard contact:** if `|centroid − hazardX| < w/2` when crossed, lose `cull × HAZARD_MULT` of the crowd. Some hazards **oscillate** (`osc`) so their x swings over distance — dodge the moving gap.
- **Barrier (clash):** at the stretch end the legion **CHARGES**. If `crowd > toll` → **SMASH**: score, then `crowd −= toll` (attrition), and the survivors run straight into the next stretch. If `crowd ≤ toll` → fail.
- **Toll** scales to the stretch (see §7/§14) so carryover stays challenging at any size; the **rising speed** is what eventually ends a great run.

---

## 5. STAGE STRUCTURE ⭐

### 5.1 Model — endless, continuous, carryover
- [x] **Discrete stretches, played back-to-back forever.** 8 authored stretches (`STAGES`), then procedural sectors (`genStage(i)`, i≥8). The world distance `S.dist` is continuous; each stretch begins at `S.stageStart` (set to the previous barrier). The **crowd carries over** (minus each barrier's toll). Difficulty ramps via rising speed, bigger multipliers, harsher hazards and more ÷ traps.

### 5.2 Where stretches are defined
- Authored: **`STAGES[]`** (search `const STAGES`). Procedural: **`genStage(i)`** (deterministic per index).
- One entry: `{ name, hue, len, ev:[ … ] }` (the legacy `start`/`target` fields are unused — the toll is dynamic, §7).
```javascript
{ name:'MAGMA RUN', hue:18, len:160, ev:[
   {at:22, t:'G', g:[{side:-1,op:'mul',v:3},{side:1,op:'mul',v:2}]},      // gate row
   {at:80, t:'G', g:[{side:-1,op:'mul',v:2},{side:0,op:'mul',v:3},{side:1,op:'div',v:3}]}, // 3-lane w/ ÷ trap
   {at:100,t:'H', kind:'saw', x:2.4, w:3.4, cull:0.40, osc:1.8, phase:1.1}, // oscillating sawblade
   {at:120,t:'P', x:0, op:'add', v:200},                                   // pickup
] }
```
- Positions are scaled by `EV_SCALE=1.4` at runtime (`evZ`, `stageLenW`) → wider spacing, less cramped.

### 5.3 The 8 authored stretches (biome hue °, base length)
| # | Name | Biome (hue) | `len` | Flavor |
|---|------|-------------|------:|--------|
| 1 | NEON DAWN | pink-red 330 | 120 | gentle intro (×2 vs +) |
| 2 | PULSE GRID | violet 280 | 135 | first pickup + −sub trap |
| 3 | VOLT CANYON | blue 200 | 150 | ×3 appears |
| 4 | MAGMA RUN | red-orange 18 | 160 | ÷ traps + oscillating hazards |
| 5 | CRYO VAULT | teal 165 | 165 | 3-lane gates, wall hazards |
| 6 | ECHO REACTOR | green 120 | 175 | ×4, ÷3 traps |
| 7 | STORM SPIRE | amber 50 | 185 | ÷ traps + swinging saws |
| 8 | OVERLORD GATE | magenta 300 | 200 | boss-length finale |
| 9+ | SECTOR n | cycling | 150+ | procedural (`genStage`): scaling ×, ÷ traps, oscillating hazards, bonus pickups |

### 5.4 Progression
Each stretch = a new biome (fog/rails/sun crossfade), wider/longer track, and **faster** run speed (§6.2). The legion snowballs via carryover; the toll grows with it; rising speed is the long-term limiter.

---

## 6. DIFFICULTY STRUCTURE ⭐

### 6.1 Modes — global knobs
| Mode | `BASE_START` (stretch-1 crowd) | `RUN_SPEED` (base) | `HAZARD_MULT` (cull) | `TOLL_FRAC` (barrier) |
|------|------------------------------:|-------------------:|---------------------:|----------------------:|
| Easy   | 40 | 14 | 0.72 | 0.30 |
| Normal | 26 | 17 | 1.00 | 0.42 |
| Hard   | 16 | 20 | 1.15 | 0.55 |

`TOLL_FRAC` sets the perfect-play margin (toll = `greedyEnd × TOLL_FRAC`, so margin = `1/TOLL_FRAC`): Easy ≈ 3.3× headroom (forgiving), Normal ≈ 2.4×, Hard ≈ 1.8× (near-optimal required). Lower `BASE_START` + faster speed make Hard tighter.

### 6.2 Speed ramp (the real endless limiter)
`S.speed = RUN_SPEED[diff] × min(SPEED_CAP=2.7, 1 + stageIdx × SPEED_RAMP=0.05)`.
Perfect play out-grows every toll indefinitely (math is survivable), so what eventually ends a great run is **reaction time** as the track keeps accelerating each stretch.

### 6.3 Verified curve (in-engine sim, `?shot=sim`)
A **greedy** bot (best gate + dodge) survives **60+ stretches on Easy/Normal/Hard** (speed-limited in real play). A **careless** bot (random gate incl. ÷/−, ignores hazards) averages ~1.4–1.6 stretches — i.e. choices and dodging matter. Re-run after any balance edit.

### 6.4 Change difficulty → §15.2.

---

## 7. SCORING ⭐

### 7.1 Toll (barrier strength), computed per stretch in `setupStage()`
```
greedyEnd = best-case crowd if you take the best gate every row from your CURRENT crowd (stageGreedyEnd)
toll      = max(8, round( min(greedyEnd, MAX_LOGIC) × TOLL_FRAC[diff] ))
```
You must out-grow the toll to smash through. Clamping to `MAX_LOGIC` ensures a maxed legion can always still clear (no false wall).

### 7.2 Points (per barrier smashed, in `resolveBarrier()`)
```
add = floor( (slamCrowd × SCORE_SLAM(3) + SCORE_CLEAR(200) + toll × SCORE_TOLL(2)) × streakMult )
```
`slamCrowd` is the legion at the slam → bigger legion = bigger score. After scoring, `crowd −= toll` (clash attrition; carries to the next stretch).

### 7.3 No-loss streak
`S.stageClean` starts true each stretch, flips false on any **loss event**: a `sub` or `div` gate, or a hazard hit. On a smash: clean → `streak = min(streak+1, 10)`, else `0`. **Attrition is not a loss.** `streakMult = 1 + min(streak,10)×0.20` (≤ ×3.0), shown as `🔥×N.N`.

### 7.4 Records
`legion-rush.best.<diff>` (score), `bestCrowd.<diff>` (biggest legion), `bestStage.<diff>` (furthest stretch). Numbers formatted K/M/B/T/Qa… by `fmt()`.

### 7.5 Change scoring → §15.3.

## 8. Economy
None — pure endless score-attack (deliberate, see §1).

## 9. Gates / Hazards / Pickups
| Type | code | Effect | Color | Loss event? |
|---|---|---|---|---|
| Multiply gate | `op:'mul'` | `crowd ×= v` | green | no |
| Add gate | `op:'add'` | `crowd += v` | blue | no |
| Subtract gate | `op:'sub'` | `crowd −= v` | red | yes |
| Divide gate | `op:'div'` | `crowd = floor(crowd/v)` (scales with size!) | orange | yes |
| Saw / Spin / Wall hazard | `kind:…` | `crowd ×= (1 − cull·HAZARD_MULT)` on contact; `osc` = swings side-to-side | red | yes |
| Pickup | `t:'P'` | `crowd += v` if driven over | gold | no |

Gate/barrier number labels are bigger `CanvasTexture` planes rotated 180° on Y so the trailing camera reads them un-mirrored.

## 10. Audio
- **SFX:** add / mul (arpeggio) / bad / cull / slam / clear / over. Mute via 🔊 (persists).
- **Music:** 128-BPM synth bed; intensity (bass→arp→lead) rises with `log10(crowd)` + stretch → thickens as your legion grows. iOS/Android: resumed inside first gesture.

## 11. Controls
- **Mobile:** drag anywhere left/right (pointer capture; `touch-action:none`). **Drag right ⇒ legion right on screen.**
- **Desktop:** drag, or **← / →** (A / D); **Space / Esc** pause.

## 12. State object `S` (key fields)
```
mode, diff, crowd, score, streak, stageIdx, stageClean,
dist (absolute forward), stageStart (this stretch's start), toll, cx, targetX, keyDir,
speed, charging, slammed, best, bestCrowd, bestStage, shake, t
```

## 13. localStorage keys (namespace `legion-rush.`)
`best.<diff>` · `bestCrowd.<diff>` · `bestStage.<diff>` · `muted` · `music` · `sfx` · `haptic` · `onboarded` · `plays`.

---

## 14. BALANCE NUMBERS (single source of truth) ⭐
```javascript
const CONFIG = {
  RENDER_CAP: 100, MAX_LOGIC: 1e15,
  BASE_START: { easy:40, normal:26, hard:16 },     // stretch-1 crowd (then carries over)
  RUN_SPEED:  { easy:14, normal:17, hard:20 },      // base units/sec
  SPEED_RAMP: 0.05, SPEED_CAP: 2.7,                 // speed ×= min(CAP, 1+stageIdx·RAMP)
  CHARGE_MULT: 2.0,
  TRACK_HALF: 6.0, STEER_MARGIN: 0.5, STEER_SENS: 1.8, STEER_LERP: 9, KEY_SPEED: 11,
  SPACING: 0.60, RUNNER_R: 0.30, BOB_RATE: 9.0,
  EV_SCALE: 1.4,                                    // widen gate spacing + stretch length
  HAZARD_MULT: { easy:0.72, normal:1.0, hard:1.15 },// × hazard cull fraction
  TOLL_FRAC:   { easy:0.30, normal:0.42, hard:0.55 }, // toll = min(greedyEnd,MAX_LOGIC)×TOLL_FRAC
  STREAK_STEP: 0.20, STREAK_MAX: 10,                // mult = 1 + min(streak,10)·0.20
  SCORE_SLAM: 3, SCORE_CLEAR: 200, SCORE_TOLL: 2,
};
// STAGES[] (§5) + genStage(i) for endless sectors.
```

---

## 15. HOW-TO recipes ⭐

### 15.1 Add / edit an authored stretch
1. Edit `STAGES[]` (§5.2): `name`, `hue` (0–360), `len`, ordered `ev` (`at` increasing). Mix gate rows, hazards (`osc` to make them swing), pickups. Keep ≥3 multiply opportunities so good play out-grows the toll.
2. Run **`?shot=sim`** → check the greedy carryover growth table and that greedy "reached 60+".
3. Update **§5.3** + commit code + DOCS together.

### 15.2 Tune difficulty
1. Edit `CONFIG.BASE_START / RUN_SPEED / SPEED_RAMP / HAZARD_MULT / TOLL_FRAC` (§14). `TOLL_FRAC` = perfect-play margin (1/frac); `SPEED_RAMP` = how fast the endless gauntlet accelerates.
2. Run `?shot=sim`: greedy should still survive 60+ on all modes; careless should fail fast. Update **§6.1** + commit with DOCS.

### 15.3 Change scoring
Edit `SCORE_SLAM / SCORE_CLEAR / SCORE_TOLL / STREAK_STEP / STREAK_MAX` (§14); update **§7.2** + commit with DOCS.

### 15.4 Tune the endless generator
Edit `genStage(i)` — multiplier ceiling (`maxMul`), ÷-trap chance, hazard frequency/oscillation, pickup chance, length, hue cycle. Re-verify with `?shot=sim` (it runs full carryover greedy/careless to 60).

### 15.5 Add turns (deferred)
Still a straight track for steering reliability. To add: a `{t:'turn',at,dir,deg}` event yawing the world group while steering stays relative to heading; re-verify the sim + steering.

---

## 16. Update history
- **2026-06-05 (v2):** went **endless** (procedural sectors after 8 authored stretches) + **carryover** crowd with a dynamic **barrier toll** (out-grow it; clamped to the cap so no false wall) + **rising speed every stretch**. Crowd is now neon **army-men** (merged-geometry InstancedMesh, helmet/rifle/boots) capped at 100 representatives with the true count on the HUD. Added **÷ gates** + **oscillating sawblades**, bigger/clearer gate labels, biome crossfade, wider spacing (`EV_SCALE`). **Fixed reversed steering** (drag-right now moves the legion right on screen — the +Z trailing camera makes screen-right = world −x, so the input is inverted). Seamless stretch transitions (no confirm).
- **2026-06-05 (v1):** initial — 8 fixed stages, per-stage fresh start, fixed targets, capsule crowd. Accent `#ff3344` (requested `#ff4d6d` was already BEATFALL's).

> **Last updated:** 2026-06-05 (v2) · branch `main`
