# LEGION RUSH — Technical Documentation

> Read this file to understand the **entire game without reading code**. Want to add a stage / change difficulty / change scoring? §5, §6, §7, §15 are enough.
> Everything lives in `index.html` (zero-build, single file). All balance numbers are centralized in **§14 (`CONFIG` + `STAGES`)**.

---

## 0. FEATURE STATUS (read first)

| Feature | Status | Where (function / marker) |
|---|---|---|
| Core loop: auto-run, steer crowd, gates, slam barrier | ✅ | `step()`, `applyEvent()`, `resolveBarrier()` |
| 3D crowd via `THREE.InstancedMesh` (1 draw call) | ✅ | `boot()` (crowdMesh), `updateCrowdInstances()` |
| Steering — drag (touch/mouse) + keyboard | ✅ | `pdown/pmove/pup`, `keydown` handler |
| Gates: ×mul / +add / −sub, pick nearest lane | ✅ | `STAGES[].ev` (t:'G'), `chooseGateSide()`, `applyGateOp()` |
| Hazards: saw / spin / wall, %-cull on contact | ✅ | `STAGES[].ev` (t:'H'), `applyEvent()` |
| Pickups: floating +bonus | ✅ | `STAGES[].ev` (t:'P') |
| End-of-stage CHARGE + barrier slam | ✅ | `resolveBarrier()`, `slamFX()` |
| 8 discrete stages, rising difficulty + biome color | ✅ | `STAGES`, `setupStage()` |
| No-loss streak score multiplier | ✅ | `S.stageClean`, `S.streak`, `resolveBarrier()` |
| 3 difficulty modes (Easy/Normal/Hard) | ✅ | `CONFIG.START_MULT / TARGET_MULT / HAZARD_MULT / RUN_SPEED` |
| Juice: particles, screen shake, slam flash, count pop, haptics | ✅ | `burst()`, `screenShake()`, `slamFX()`, `popCrowd()`, `haptic()` |
| Synthwave music that builds with crowd size | ✅ | `Audio.startMusic()`, `Audio.setIntensity()` |
| WebAudio SFX + mute | ✅ | `Audio.sfx()`, `#muteBtn` |
| Bloom + graceful low-FX degradation + DPR cap | ✅ | `UnrealBloomPass`, `applyLowFX()`, `resize()` |
| CDN-fail guard (Reload message, no black screen) | ✅ | top-level `try/catch` around `import('three')` |
| localStorage: best score + biggest legion + best stage per difficulty | ✅ | `Save`, `gameOver()`, `loadBests()` |
| Onboarding "How to play" (first run) | ✅ | `#how`, `playBtn` handler |
| In-engine balance sim (bots) | ✅ | `?shot=sim`, `_simAll()` |
| Headless screenshot + selftest hooks | ✅ | `?shot=…`, `applyShot()` |

**Backlog / deferred:** **Turns** (curved track) — the data model + steering are built for a straight axis; turns are documented as a future extension in §15.5 (kept straight in v1 for steering reliability). Coins/meta-progression not implemented (score-attack design). Split-flow gates (crowd straddling two gates) — v1 uses centroid lane selection (whole crowd takes one gate).

---

## 1. Overview & Concept

### Market context (research 2026-06-05)
- **Genre hot:** crowd-runner / "count master" is a proven hybrid-casual hook (Count Masters, Crowd City, Count Control Legends on Poki/CrazyGames). Hybrid-casual was the fastest-growing mobile sector in 2025 (~$4.2B). The "steer a crowd through math gates" loop is a cheap, high-retention hook.
- **Mechanics that work:** ×/＋/− gates, choosing the better gate even off your path, dodging red gates & traps, a boss/king slam at the finish.
- **Competitor gaps:** the web/H5 versions are ad-heavy, flat stickman art, and endless-shallow.
- **Hook:** a **neon-synthwave, instant-play (no ads/install)** version with **authored discrete stages** of rising difficulty and a **no-loss streak multiplier** that rewards clean play — a score-attack arcade runner, not an endless ad-runner. The CHARGE slam (crowd→damage) is the signature beat; **percentage-cull hazards** stay threatening at any crowd size.

### Concept
- **Genre:** 3D crowd-multiplier runner (single player).
- **Core loop (1 sentence):** *"Steer your whole neon legion left/right down a track, pass ×/＋ gates to multiply your numbers and dodge culling hazards, then slam an end-of-stage barrier whose strength = a target count — a bigger crowd clears it and scores more; clear all 8 stages."*
- **Win:** clear all 8 stages (smash the OVERLORD GATE). **Lose:** crowd hits 0 mid-stage, OR reach a barrier with `crowd < target`.
- **Fantasy:** command an ever-growing army; feel the power spike as your legion balloons from a handful to thousands, then the screen-shaking slam.
- **2D or 3D:** **3D (Three.js)** — depth + a trailing camera + a crowd surging down a track is the "wow"; the crowd is one `InstancedMesh` for performance.
- **Layout (mobile #1):** **A — Mobile-frame.** A runner only needs a narrow vertical track, so the game is framed in a portrait column `width:min(100vw,480px)`, centered with letterbox on desktop. No responsive breakpoints needed.

## 2. Tech stack
- **Render:** Three.js **r0.169** via importmap CDN (jsDelivr). Post: `EffectComposer` + `UnrealBloomPass` (strength 0.62, radius 0.7, threshold 0.82) + `OutputPass`. Bloom is wrapped in its own try/catch — if the addons fail to load, the game runs without bloom.
- **Build:** zero-build, single `index.html`, ES module inline, no framework/bundler/npm.
- **External resources:** Google Fonts (Orbitron + Space Grotesk) + Three.js CDN. Nothing else.
- **Storage:** `localStorage` (namespace `legion-rush.*`). **Audio:** WebAudio synth (no audio files).
- **Crowd:** one `InstancedMesh` of `CapsuleGeometry` capsules (cap `RENDER_CAP=700` drawn; logical crowd can exceed and is shown via the HUD/barrier). Phyllotaxis formation packing.

## 3. State machine
`mode ∈ menu → playing → (paused) → dead/won → gameover → menu/playing`
- **menu:** title, difficulty pills, best stats. `crowdMesh.count=0`.
- **playing:** fixed-tick `step()` runs; camera follows; events apply as `S.dist` crosses each `ev.at`.
- **paused:** tick halted, scene still renders, overlay shown (HUD hidden via `.overlay-open`).
- **dead / won:** short FX beat (shake/burst/flash), then `gameOver()`.
- **gameover:** end panel (hero score + best + biggest legion + stage reached + retry/menu).
- **frozen** (screenshots/dev only): renders a static scene, does not advance the run.

## 4. Gameplay & rules
- **Field:** straight track, half-width `TRACK_HALF=6`. The crowd's centroid `S.cx` is clamped to `±(TRACK_HALF − STEER_MARGIN) = ±5.5`. The crowd auto-advances `S.dist` forward at `RUN_SPEED` (×`CHARGE_MULT` in the final 16 units before the barrier).
- **Steering:** drag moves the centroid relative to drag start (`world = dx/canvasWidth × TRACK_HALF×2 × STEER_SENS`); the centroid chases the target via lerp (`STEER_LERP`). Keyboard nudges the target at `KEY_SPEED`.
- **Gate selection:** a gate row has 2–3 options at lanes (2 → x=±TRACK_HALF/2; 3 → x=−1/0/+1 × TRACK_HALF·⅔). When the crowd crosses the row's `at`, the option whose lane is **nearest the centroid** applies to the whole crowd (`chooseGateSide()`).
- **Hazard contact:** if `|centroid − hazard.x| < hazard.w/2` when crossed, the crowd loses a fraction (`cull × HAZARD_MULT[diff]`). Hazards span only part of the track so they're dodgeable.
- **Barrier:** at `stage.len` the crowd CHARGES; `crowd ≥ target` → **SMASH** (clear, score, advance); `crowd < target` → **fail**.
- **Death:** any gate/hazard that brings `crowd ≤ 0` ends the run immediately.

---

## 5. STAGE STRUCTURE ⭐

### 5.1 Level model
- [x] **Discrete stages** — 8 authored tracks, each an ordered list of events + an end barrier target. No carryover: **each stage starts fresh** at its own start-crowd (see §5.2). Rising difficulty comes from bigger starts, bigger multipliers, harder hazards, and much bigger targets.

### 5.2 Where stages are defined
- Data: the **`STAGES[]`** array in `index.html` (search `const STAGES`).
- One stage entry:
```javascript
{ name:'NEON DAWN', hue:330, len:120, start:20, target:130, ev:[
    {at:22, t:'G', g:[{side:-1,op:'add',v:20},{side:1,op:'mul',v:2}]},  // gate row: pick a lane
    {at:64, t:'H', kind:'saw', x:-2.6, w:3.0, cull:0.25},               // hazard: %-cull on contact
    {at:82, t:'P', x:0, op:'add', v:60},                                // pickup: drive over it for +bonus
] }
```
- `hue` = biome color (HSL hue). `len` = track length (world units). `start` = base start crowd. `target` = base barrier strength.
- **Per-difficulty effective values:** `startCrowd = max(4, round(start × START_MULT[diff]))`, `target = round(target × TARGET_MULT[diff])`, hazard cull `= cull × HAZARD_MULT[diff]`. (See §6 + §14.)

### 5.3 The 8 stages (base values; biome hue in °)
| # | Name | Biome (hue) | `len` | `start` | `target` | Greedy @barrier (Normal, from sim) |
|---|------|-------------|------:|-------:|--------:|-----:|
| 1 | NEON DAWN | pink-red 330 | 120 | 20 | 130 | 320 |
| 2 | PULSE GRID | violet 280 | 135 | 26 | 320 | 656 |
| 3 | VOLT CANYON | blue 200 | 150 | 32 | 1,150 | 2,304 |
| 4 | MAGMA RUN | red-orange 18 | 160 | 42 | 2,300 | 4,536 |
| 5 | CRYO VAULT | teal 165 | 165 | 55 | 2,800 | 5,535 |
| 6 | ECHO REACTOR | green 120 | 175 | 70 | 11,500 | 22,700 |
| 7 | STORM SPIRE | amber 50 | 185 | 90 | 80,000 | 157,000 |
| 8 | OVERLORD GATE | magenta 300 | 200 | 120 | 700,000 | 1,680,000 |

"Greedy @barrier" = crowd a perfect bot reaches at the barrier (Normal) — the **headroom** over `target`. Designed so perfect play clears every stage on every difficulty (margin ≥ ~1.3× even on Hard), while sloppy play falls short. Re-measure any time you edit a stage: open `?shot=sim`.

### 5.4 Progression & milestones
- Each stage = a new biome color (fog/sun/rails recolor in `setupStage()`), a longer track, and a far bigger target.
- Stage 8 "OVERLORD GATE" is the boss-barrier finale (target 700K base). Clearing it = **VICTORY**.
- Difficulty multipliers (§6) reshape every stage at once.

---

## 6. DIFFICULTY STRUCTURE ⭐

### 6.1 Modes — global multipliers (×) over the base stage values
| Mode | `START_MULT` (start crowd) | `RUN_SPEED` | `HAZARD_MULT` (cull) | `TARGET_MULT` (barrier) |
|------|--------------------------:|-----------:|---------------------:|------------------------:|
| Easy   | 1.50 | 15 | 0.70 | 0.60 |
| Normal | 1.00 | 18 | 1.00 | 1.00 |
| Hard   | 0.75 | 21 | 1.15 | 1.12 |

Easy = more starting runners, gentler hazards, lower targets, slower track (more reaction time). Hard = fewer starters, harsher hazards, higher targets, faster track.

### 6.2 In-run ramp
Difficulty rises across the 8 stages by design (start/target/hazard curves in §5.3), not by a per-tick speed ramp. Within a stage, speed is constant except the final **CHARGE** (`×CHARGE_MULT=2.3`) in the last 16 units.

### 6.3 Verified curve (in-engine sim, `?shot=sim`)
A **greedy** bot (always best gate, always dodges) clears **8/8 on Easy, Normal, and Hard**. A **careless** bot (random gate incl. −, ignores hazards) averages ~2.1 / 1.9 / 1.4 stages — i.e. perfect play always wins; sloppy play fails progressively earlier on harder modes. Re-run after any balance edit.

### 6.4 How to change difficulty → §15.2.

---

## 7. SCORING ⭐

### 7.1 Points (awarded once per stage cleared, in `resolveBarrier()`)
```
gain      = floor(crowd × SCORE_SLAM)            // slam power  (SCORE_SLAM = 4)
          + SCORE_CLEAR                          // flat clear bonus (250)
          + floor((crowd − target) × SCORE_SURPLUS)   // surplus bonus (2 / extra runner)
streakMult = 1 + min(streak, STREAK_MAX) × STREAK_STEP   // 1 + min(streak,10)×0.20  → up to ×3.0
scoreAdded = floor(gain × streakMult)
```
`crowd` is the legion size at the moment of the slam → **bigger crowd = much bigger score.** Score only comes from clearing stages.

### 7.2 No-loss streak
- `S.stageClean` starts true each stage and flips false on any **loss event**: picking a `sub` gate, or contacting a hazard (any crowd decrease).
- On a clear: `stageClean` true → `streak = min(streak+1, 10)`; otherwise `streak = 0`.
- Streak multiplier shown in HUD as `🔥×N.N` (max ×3.0). Rewards flawless gate-picking + dodging.

### 7.3 Records
- `legion-rush.best.<diff>` (best score), `legion-rush.bestCrowd.<diff>` (biggest legion ever), `legion-rush.bestStage.<diff>` (furthest stage). Shown on menu + game-over. Numbers formatted K/M/B by `fmt()`.

### 7.4 How to change scoring → §15.3.

## 8. Economy
None — pure score-attack (no coins/shop). Deliberate (see §1 hook). If added later, namespace under `legion-rush.*`.

## 9. Gates / Hazards / Pickups (the "items")
| Type | code | Effect | Visual | Color |
|---|---|---|---|---|
| Multiply gate | `op:'mul'` | `crowd ×= v` | translucent panel + label `×v` | green |
| Add gate | `op:'add'` | `crowd += v` | translucent panel + label `+v` | blue |
| Subtract gate | `op:'sub'` | `crowd −= v` (loss event) | translucent panel + label `−v` | red |
| Saw hazard | `kind:'saw'` | `crowd ×= (1 − cull·mult)` on contact | spinning cross bars | red |
| Spin hazard | `kind:'spin'` | same, spins on 2 axes | spinning cross bars | red |
| Wall hazard | `kind:'wall'` | same | spinning bars | red |
| Pickup | `t:'P'` | `crowd += v` if driven over | floating gold gem `+v` | gold |

Gate/barrier labels are `CanvasTexture` planes rotated 180° on Y so the trailing camera reads them un-mirrored (see §LESSONS link in repo guide).

## 10. Audio
- **SFX** (`Audio.sfx`): `add` (rising blip), `mul` (3-note arpeggio up), `bad`/`cull` (low saw), `charge`, `slam` (deep boom), `clear` (5-note fanfare), `win` (7-note), `over`. Mute via 🔊 button (persists).
- **Music** (`Audio.startMusic` / `setIntensity`): a 128-BPM synth bed (bass always; arpeggio when intensity > 0.33; lead+hat when > 0.66). Intensity rises with `log10(crowd)` + stage index → the track **thickens as your legion grows**.
- iOS/Android: `AudioContext` is created/resumed inside the first user gesture (`Audio.ensure()` in every input handler).

## 11. Controls
- **Mobile:** drag anywhere on the canvas left/right to steer the whole crowd (pointer capture; `touch-action:none` prevents scroll/tap-leak).
- **Desktop:** drag with mouse, or **← / →** (or **A / D**) to steer; **Space / Esc** to pause/resume.

## 12. State object `S` (key fields)
```
mode, diff, crowd, score, streak, stageIdx, stageClean,
dist (forward distance in stage), cx (centroid X), targetX (steer target), keyDir,
speed, charging, slammed, best, bestCrowd, bestStage, shake, t
```

## 13. localStorage keys (namespace `legion-rush.`)
| key | meaning |
|---|---|
| `best.<diff>` | best score for that difficulty |
| `bestCrowd.<diff>` | biggest legion reached |
| `bestStage.<diff>` | furthest stage reached |
| `muted` | SFX/master mute |
| `music`, `sfx`, `haptic` | settings toggles |
| `onboarded` | How-to-play shown once |
| `plays` | total games played |

---

## 14. BALANCE NUMBERS (single source of truth) ⭐
All tunables are in `CONFIG` and `STAGES` near the top of the `<script>` in `index.html`.
```javascript
const CONFIG = {
  RENDER_CAP: 700, MAX_LOGIC: 99999999,
  START_MULT: { easy:1.50, normal:1.0, hard:0.75 },   // × stage.start
  RUN_SPEED:  { easy:15,   normal:18,  hard:21   },   // world units/sec
  CHARGE_MULT: 2.3,                                    // speed-up in final 16u
  TRACK_HALF: 6.0, STEER_MARGIN: 0.5, STEER_SENS: 1.7, STEER_LERP: 9, KEY_SPEED: 11,
  SPACING: 0.62, RUNNER_R: 0.30, BOB_RATE: 9.0,        // crowd formation/anim
  HAZARD_MULT: { easy:0.70, normal:1.0, hard:1.15 },   // × hazard cull fraction
  TARGET_MULT: { easy:0.60, normal:1.0, hard:1.12 },   // × stage target
  STREAK_STEP: 0.20, STREAK_MAX: 10,                   // mult = 1 + min(streak,10)×0.20
  SCORE_SLAM: 4, SCORE_CLEAR: 250, SCORE_SURPLUS: 2,   // scoring
};
// STAGES[] — see §5. Each: { name, hue, len, start, target, ev:[ {at,t,...} ] }
```

---

## 15. HOW-TO recipes ⭐

### 15.1 Add a new stage
1. Append an entry to **`STAGES[]`** (§5.2 structure): set `name`, `hue` (0–360), `len`, `start`, `target`, and an ordered `ev` list (`at` increasing, last well before `len`).
2. Mix gate rows (`t:'G'`), hazards (`t:'H'`), and pickups (`t:'P'`). Keep 3–4 multiply opportunities so a perfect run grows enough to beat `target`.
3. Run **`?shot=sim`** → read the new stage's **greedy @barrier**; set `target ≈ 0.5 × that` (so perfect play clears ~2× on Normal, ≥~1.3× on Hard).
4. Re-run the sim: greedy must show **CLEAR** on Easy/Normal/Hard.
5. Update **§5.3 table** here. Commit code + DOCS together.

### 15.2 Tune difficulty
1. Edit **`CONFIG.START_MULT / RUN_SPEED / HAZARD_MULT / TARGET_MULT`** (§14) and/or per-stage `start`/`target` (§5.3).
2. Run `?shot=sim`: confirm greedy clears 8/8 on all modes and careless fails progressively. Constraint: keep Hard's greedy margin ≥ ~1.25 (Hard greedy ≈ 0.71 × Normal greedy; Hard target = 1.12 × base).
3. Update **§6.1 table**. Commit code + DOCS together.

### 15.3 Change scoring
1. Edit `SCORE_SLAM / SCORE_CLEAR / SCORE_SURPLUS / STREAK_STEP / STREAK_MAX` (§14).
2. Update **§7.1 formula** here. Verify HUD score + best-score persist.
3. Commit code + DOCS together.

### 15.4 Add a gate op / hazard kind / pickup
1. Gate op: extend `applyGateOp()` + `opLabel()`/`opColor()`. Hazard kind: handle in `spawnHazardVisual()` + `render()` spin logic. Pickup already generic.
2. Update **§9** + **§0** here. Commit with DOCS.

### 15.5 Add turns (deferred feature)
v1 runs a straight track for steering reliability. To add turns: introduce a `{t:'turn', at, dir, deg}` event that rotates the world group (or yaws the forward axis) over a span, and keep steering relative to the current heading. Add to §5.2 model + §9, re-verify the sim and steering, then update this section.

---

## 16. Update history
- **2026-06-05** (initial): LEGION RUSH shipped — 3D Three.js crowd-multiplier runner, 8 stages, 3 difficulties, InstancedMesh crowd, no-loss streak scoring, synthwave audio, in-engine balance sim. Accent `#ff3344` (requested `#ff4d6d` was already taken by BEATFALL — substituted a distinct hot-scarlet).

> **Last updated:** 2026-06-05 · branch `main`
