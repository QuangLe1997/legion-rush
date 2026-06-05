# 🏃 LEGION RUSH — Neon Crowd Runner

Steer a glowing **legion** of neon runners down a synthwave track. Blast through **×2 / +30** gates to multiply your numbers, dodge spinning hazards that cull your crowd, then **CHARGE** the end-of-stage barrier — a bigger legion smashes it for a bigger score. Eight stages of rising difficulty, a no-loss **streak multiplier** for flawless runs, and a screen-shaking slam at every finish. A 3D crowd-multiplier ("count master") runner, built for mobile portrait.

**▶ Live:** https://quangle1997.github.io/legion-rush/ · part of [QUANG ARCADE](https://quangle1997.github.io/arcade/)

## Features
- **Core gameplay:** auto-running crowd, drag to steer, multiply/add/subtract gates, %-cull hazards, end-of-stage barrier slam (crowd ≥ target = smash).
- **8 discrete stages** of rising difficulty, each its own neon biome, ending in the OVERLORD GATE boss-barrier.
- **3 difficulty modes** (Easy / Normal / Hard) — verified by an in-engine balance sim so perfect play always clears and sloppy play falls short.
- **No-loss streak** score multiplier (up to ×3.0) rewards flawless gate-picking + dodging.
- **Look & feel:** Three.js 3D, `InstancedMesh` crowd (one draw call for the whole legion), weak bloom, particle bursts, screen shake, slam flash, synthwave music that thickens as your crowd grows.
- **Controls:** touch drag · mouse drag · arrow keys / A-D · Space-Esc to pause.
- **Systems:** localStorage best score + biggest legion + furthest stage (per difficulty), first-run onboarding, mute & settings.
- Single `index.html`, zero build, mobile-first portrait, CDN-fail guard, deployed on GitHub Pages.

## Run locally
```bash
python3 -m http.server 8773
# open http://localhost:8773
```
(ES modules / importmap need an HTTP origin — `file://` won't work.)

### Dev / QA hooks (harmless in production)
- `?shot=sim` — run the in-engine balance sim (bots play every stage via the real logic) and print the table.
- `?shot=selftest` — input + keyboard + logic + console-error self-check.
- `?shot=play:N` / `big:N` / `slam:N` / `clear:N` / `pause` / `how` / `win` / `lose` (+ `&diff=hard`) — drive a state for screenshots.

---
Built by [QuangLe1997](https://github.com/QuangLe1997) · crafted with ♥ & Claude Code.
