# Cloud City — project notes

A single-file, browser-based **3D open-world game that teaches AWS**. You roam a
city on foot or by vehicle; each building is an AWS service you enter to do a
hands-on console-style lab. Stylized low-poly look, day/night cycle, no quizzes.

## Files
- **`cloud-city.html`** — the entire game (HTML + CSS + JS in one file). This is the deliverable.
- `.claude/launch.json` — preview config (serves the folder via `python -m http.server 8777`).
- `NOTES.md` — this file.
- `Test/` — a stray empty nested git repo, unrelated; **not** part of the game (left untracked).

## Run / preview
- Just open `cloud-city.html` in a browser (double-click) — it works offline-ish.
- **Only external dependency:** Three.js **r128** from cdnjs (one `<script>`). Needs
  internet the first time, or it shows a blank screen. To make it fully self-contained
  for sharing on locked-down networks, inline `three.min.js` into the file.
- For a dev server: it's wired for `python -m http.server 8777` (see launch.json).

## Sharing with a team
- Email **just `cloud-city.html`** — one file, double-click to play (needs internet for the CDN).
- Azure does NOT improve the graphics (rendering is client-side); it'd only host a URL
  (Static Web Apps / Blob static website). Better graphics = improve the Three.js code.

## Architecture (where things live in cloud-city.html)
- **Boot:** character creator → `beginGame()` → `initWorld()` → `animate()` loop
  (`updatePlayer` → `updateWorld` → `updateCamera` → `updateHUD`).
- **Data:** `DISTRICTS` (4), `LANDMARKS` (11 AWS services), `PLACES` (special non-service
  spots: Cloud Investigation, Zanardi Project), `CONSOLE_LABS` (the hands-on content),
  `GIGS`, `OUTFITS`, `VEHICLES`.
- **Hands-on labs:** `showConsoleLab()` / `gradeConsoleLab()` render an AWS-console-style
  board (scenario + real-AWS dropdowns) per building; selections are validated.
- **Map:** `openMap()` draws `#fullMap` (numbered pins + clean district labels) plus a
  numbered side list; near full-page static panel. `fastTravelTo()` teleports.
- **World systems in `updateWorld()`:** multi-phase sky + sun/moon/stars/god-rays, fog,
  night window-lighting, moving traffic (`trafficCars`), ambient pedestrians, district
  signs/neon, fountain + hologram, Zanardi cranes/excavators.
- **Collision:** `obstacles[]` (circles) + `resolveCollisions()` — radii kept small enough
  that you can still get within ~13u to press E.
- **Crashes:** `checkVehicleCrash()` / `triggerCrash()` → flash + shake + sound, respawn at
  last safe spot.
- **Audio:** `AudioFX` = fully synthesized Web Audio (no asset files); city hum, horns,
  crowd murmur, crash, UI chimes. Mute button top-right (or `0`).
- **Sign text:** `fitFont()` shrinks canvas text so signs never clip — use it for any new sign.

## Controls
WASD move · Shift sprint · **arrow keys** orbit/tilt camera · `,`/`.` zoom · E enter/interact ·
V vehicle · M or F map · B badges · O outfit · J gigs · **I** toggle controls panel ·
H honk · **0** mute · `[`/`]` time speed · `\` pause · T skip 6h.

## Conventions / gotchas
- **Commits are local only** (no git remote). Commit when asked. Author currently
  auto-detected as `Palraj <2401169@cognizant.com>` — set `git config user.name/email` to change.
- **Preview limitation:** the dev preview uses software WebGL — **screenshots time out**, and
  full-FPS measurement is unreliable. Verify changes by inspecting DOM/state via eval
  (element rects, scene mesh counts, `renderer.info`), not by eye.
- **Performance rule:** keep total scene meshes low. Building windows are **baked into
  textures** (not per-window meshes) — an earlier version hit ~58k meshes and tanked FPS;
  it's now ~2k. Don't reintroduce thousands of small meshes.
- `#fullMap` must stay `position:relative` (a global `canvas{position:absolute}` rule will
  otherwise pin it to the corner and break the map layout).

## Status / ideas not yet done
- **Cloud Investigation** is a reserved "coming soon" plot — experience not built yet.
- Possible next steps: capstone wiring, gig-requires-lab-solve, self-contained (inlined
  Three.js) build for offline sharing.
