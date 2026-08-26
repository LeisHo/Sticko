# STICKO — Stick-Figure Physics Playground

A single self-contained HTML page: stick figures walk onto two platforms
(one near the top of the browser, one near the bottom), idle and wander on
their own, and react to clicks with real Matter.js physics — no keyframed
"falling" or "flying" animation, no scripted reactions. Falling, getting
catapulted, mid-air punches, and getting back up are all forces/impulses
handed to the physics engine. Walking/idling itself is **kinematic, not
physics-simulated** — the figure's position and jointed-limb pose are
computed directly each frame (real forward kinematics for the legs/arms,
not a physics spring), which is what keeps it standing exactly on the
platform instead of sliding or floating; the moment a figure is clicked,
it becomes a full ragdoll and physics takes over completely.

See `PROJECT_PROGRESS.md` for the honest state-of-the-project narrative —
this file is the how-to-run reference.

## Run it

Open `deliverable/index.html` directly in a browser (double-click it, or
`start deliverable/index.html` from a terminal on Windows). No build step,
no server, no install — it's one file that pulls Matter.js from a CDN.

## What it does

- Stick figures walk in from either edge of the screen onto whichever
  platform they're heading for, then wander/idle indefinitely on their own.
- New figures spawn every N seconds (dev-panel slider), alternating between
  the two platforms, up to a soft cap of 40 on-screen at once (kept as a
  fixed internal limit, not a slider, to protect frame rate on longer runs).
- Click a figure standing on the **top** platform → it ragdolls and falls
  all the way to the **bottom** platform.
- Click a figure standing on the **bottom** platform → it gets catapulted
  up to the **top** platform.
- Click a figure while it's mid-air (falling or flying) → it gets punched
  in the direction away from your click point.
- Either way, once it lands it's a physics-driven ragdoll the whole trip
  down/up — it settles, gets back on its feet (a short physics-applied
  "stand up" recovery, not a canned animation), and resumes walking/idling
  on whichever platform it landed on. Landing reassigns its "home" platform,
  so the toy is cyclical: click it again from there and it goes the other way.
- Clicking any walking/idling figure also scares nearby ones — figures
  within the reaction-distance slider's range flee a run-distance away
  (both randomized per the randomization slider), then settle back into
  idling/walking.

## Dev panel

Top-right, always visible (toggle with the "Hide/Show dev panel" button).
Every slider is live — no reload, no apply button:

- **Platforms**: top platform height, bottom platform height, platform
  thickness.
- **Figures**: stickman size (applies to newly-spawned figures only —
  live-rescaling already-spawned ragdoll joints isn't supported), rate at
  which new men enter frame.
- **Physics**: rate of fall, rate of catapult, mid-air click (punch) effect
  scale, overall physics engine intensity (scales gravity and most applied
  forces/impulses together).
- **Shadows**: shadow size (blur radius) and shadow distance (vertical
  offset) for the drop shadow cast by platforms and figures onto the
  white background.
- **Reactions**: reaction distance (how close a click has to land before a
  nearby figure flees), run distance (how far it flees), randomization
  (variance applied to both distances).

## Project structure

```
STICKO/
├── deliverable/
│   └── index.html         The entire app — HTML/CSS/JS in one file, single-
│                           file architecture on purpose (see .claude/CLAUDE.md)
├── docs/
│   ├── PROJECT_SUMMARY.md   short status pointer
│   └── CODE_SUMMARY.md      file map / architecture / gotchas pointer
├── .claude/
│   ├── CLAUDE.md           project-specific instructions
│   └── ETA History.md      timing log
└── PROJECT_PROGRESS.md     detailed session log
```

## Known limitations

See `PROJECT_PROGRESS.md` session 2 for the full account. Short version:
a real debugging session (spawning, walking, falling, catapulting, and
landing all manually driven and inspected live) found and fixed several
real bugs — a canvas-sizing race that meant nothing ever rendered, a
sign-inverted force that pushed figures through the platform instead of
onto it, an angular-velocity controller that could spin a figure out
within a fraction of a second, and a couple of physics-tuning
instabilities. All of that is now verified mechanically stable (a 9-figure,
33-simulated-second stress test with a fall and a catapult mid-run settled
every figure near its platform, upright, no runaway/NaN/lost figures).
What's still unverified is purely how it *looks* — this session's browser
tooling never composited a live frame for the page, so the walk cycle,
shadow appearance, and catapult arc were checked as physics data, not as
something anyone actually watched. Open it in a real browser and see.
