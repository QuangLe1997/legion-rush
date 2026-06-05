# 🏃 LEGION RUSH — Neon Crowd Runner

Steer a glowing **legion** of neon soldiers down a synthwave track. Blast through **×2 / +30** gates to multiply your army, dodge **÷2** traps and swinging sawblades that cull your crowd, then **CHARGE** each barrier — out-grow its toll to **SMASH** through and roll straight on. Your legion **carries over and snowballs** into the millions; the track **never ends and keeps speeding up**. A no-loss **streak multiplier** rewards flawless runs. A 3D crowd-multiplier ("count master") runner, built for mobile portrait.

**▶ Live:** https://quangle1997.github.io/legion-rush/ · part of [QUANG ARCADE](https://quangle1997.github.io/arcade/)

## Features
- **Endless** — 8 hand-authored stretches, then procedurally-generated sectors forever. Seamless: no stop or confirm between stretches, and the track **speeds up every stretch**.
- **Carryover army** — your legion persists and snowballs across stretches (into the millions/billions). Each barrier is a **clash with a toll** you must out-grow to smash through.
- **Core gameplay:** auto-running crowd, drag to steer, ×/＋/−/÷ gates, %-cull sawblades (some swing side-to-side), barrier slams.
- **Neon army-men:** a real soldier crowd (helmet, rifle, boots) as one merged-geometry `InstancedMesh` (single draw call); renders ~100 representatives, true count shown on the HUD.
- **3 difficulty modes** (Easy / Normal / Hard) — an in-engine balance sim verifies perfect play survives 60+ stretches while sloppy play falls fast.
- **No-loss streak** score multiplier (up to ×3.0); biome crossfades; bloom, particle bursts, screen shake, slam flash; synthwave music that thickens as your crowd grows.
- **Controls:** touch drag · mouse drag · arrow keys / A-D · Space-Esc to pause.
- **Systems:** localStorage best score + biggest legion + furthest stretch (per difficulty), first-run onboarding, mute & settings.
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
