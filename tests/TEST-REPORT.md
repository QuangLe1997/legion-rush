# LEGION RUSH — Test Report (QA Evidence)

- **Date:** 2026-06-05 (v2 — endless / carryover / army-men)
- **Build:** `index.html` (single file, zero-build), served via local `http.server`
- **Viewports:** desktop **1280×800** (letterbox) · mobile-frame **480×900** (frame width — see note)
- **Console errors:** **0** (in-page collector `window.__errs`, reported by `?shot=selftest`)
- **Automated:** in-engine balance sim (`?shot=sim`) + self-test (`?shot=selftest`)

> Mobile-frame note: per the dev-guide LESSONS, headless Chrome ignores `device-width`, so this `width:min(100vw,480px)` game is captured at **480** (its real phone-frame width); desktop at 1280 shows the centered letterbox.

## Automated results

**`?shot=selftest`** → `SELFTEST OK`
| Check | Result |
|---|---|
| Drag-right moves the legion **screen-right** (worldX → −5.50) — control direction fixed | ✅ PASS |
| Keyboard ArrowRight → screen-right (keyDir −1) | ✅ PASS |
| Greedy carryover bot reaches stage 6 via real `step()` | ✅ PASS |
| **Console errors = 0** | ✅ PASS |

**`?shot=sim`** (bots play the real `step()` logic, full carryover):
- **Greedy (perfect) bot survives 60+ stretches on Easy, Normal AND Hard** — the math is survivable forever (toll clamped to the crowd cap); rising speed is the real limiter in human play.
- Careless bot (random gate incl. ÷/−, ignores hazards) averages ~1.4–1.6 stretches — choices + dodging matter.
- Greedy carryover growth table prints per stretch (enter → toll → slam → after), confirming the legion snowballs while each toll stays beatable.

## Screenshot matrix

| ID | Category | Test | Viewport | Image | Result |
|----|----------|------|----------|-------|--------|
| T01 | UI | Menu — title, difficulty pills, best stats | desktop | [01-menu-d.jpg](screenshots/01-menu-d.jpg) | ✅ PASS |
| T02 | Stage | Stretch 1 NEON DAWN — soldier crowd + clear gates (+15 / ×2) | desktop | [02-stage1-d.jpg](screenshots/02-stage1-d.jpg) | ✅ PASS |
| T03 | Crowd | Big army (Stretch 3, streak ×2.0) — dense soldier legion | desktop | [03-army-d.jpg](screenshots/03-army-d.jpg) | ✅ PASS |
| T04 | Logic | Barrier slam — toll number, huge legion | desktop | [04-slam-d.jpg](screenshots/04-slam-d.jpg) | ✅ PASS |
| T05 | Endless | Procedural SECTOR (stretch 12) — endless gen, new biome | desktop | [05-sector-d.jpg](screenshots/05-sector-d.jpg) | ✅ PASS |
| T06 | UI | Game over — score + best + biggest legion + stretch reached | desktop | [06-gameover-d.jpg](screenshots/06-gameover-d.jpg) | ✅ PASS |
| T10 | Responsive | Menu (mobile frame) | mobile | [10-menu-m.jpg](screenshots/10-menu-m.jpg) | ✅ PASS |
| T11 | Stage | Stretch 1 gameplay — army-men + gates + sawblade | mobile | [11-stage1-m.jpg](screenshots/11-stage1-m.jpg) | ✅ PASS |
| T12 | Crowd | Big army (Stretch 3, 4.8K legion, streak ×2.0) | mobile | [12-army-m.jpg](screenshots/12-army-m.jpg) | ✅ PASS |
| T13 | Gates | ÷ gate row + oscillating sawblade (Stretch 4 MAGMA) | mobile | [13-divgate-m.jpg](screenshots/13-divgate-m.jpg) | ✅ PASS |
| T14 | Logic | Barrier slam (toll vs legion) | mobile | [14-slam-m.jpg](screenshots/14-slam-m.jpg) | ✅ PASS |
| T15 | Endless | Procedural SECTOR (stretch 12), new biome | mobile | [15-sector-m.jpg](screenshots/15-sector-m.jpg) | ✅ PASS |
| T16 | Transition | SMASH! banner + streak multiplier (seamless clear) | mobile | [16-smash-m.jpg](screenshots/16-smash-m.jpg) | ✅ PASS |
| T17 | UI | How-to-play onboarding overlay | mobile | [17-howto-m.jpg](screenshots/17-howto-m.jpg) | ✅ PASS |
| T18 | UI | Pause overlay — Music/SFX/Haptics (HUD hidden) | mobile | [18-pause-m.jpg](screenshots/18-pause-m.jpg) | ✅ PASS |
| T19 | UI | Game over (mobile) | mobile | [19-gameover-m.jpg](screenshots/19-gameover-m.jpg) | ✅ PASS |
| T20 | Endless | Deep-run game-over (stretch 23, 12.5M score) | mobile | [20-deeprun-m.jpg](screenshots/20-deeprun-m.jpg) | ✅ PASS |
| T21 | Difficulty | HARD gameplay (Stretch 7) | mobile | [21-hard-m.jpg](screenshots/21-hard-m.jpg) | ✅ PASS |
| T22 | Persist | Best 12.5M / Biggest legion 3.40M **survive a reload** (separate session, shared profile) | mobile | [22-persist-m.jpg](screenshots/22-persist-m.jpg) | ✅ PASS |

## Summary
- **Test cases:** 19 screenshots + 2 automated suites · **PASS:** all · **FAIL:** 0
- **Coverage:** menu / HUD / how-to / pause / game-over · soldier-army rendering at scale · ×/＋/÷ gates + oscillating sawblades · barrier slam + toll · endless procedural sectors + biome variety · carryover snowball (deep-run) · scoring + streak · localStorage persistence across reload · Easy/Normal/Hard · **fixed control direction** (drag-right → screen-right) · mobile + desktop.
- **Console errors:** 0.
- **DOCS.md matches code:** ✅ (endless/carryover model, toll formula, CONFIG, controls, ÷ gates, speed ramp all verified against `index.html` + `?shot=sim`).

## ✅ VERDICT: **PASS — GAME DONE (v2)**
