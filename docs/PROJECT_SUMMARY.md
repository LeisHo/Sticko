# STICKO — Project Summary

Status: live-verified and mechanically stable (spawning, walking, falling,
catapulting, landing, recovering all confirmed via a manually-driven
simulation run) as of 2026-08-26, session 2; visual/gait polish (does the
motion actually *look* good) is still unjudged — see Known Limitations.
See `../PROJECT_PROGRESS.md` for the detailed, honest session log — this
file stays a short pointer, not a duplicate.

## Objective
A single-file HTML physics toy: stick figures walk/idle on two platforms
(top and bottom of the browser) on their own, and clicking them triggers
Matter.js-driven falls, catapults, mid-air punches, and flee reactions —
no scripted/keyframed animation for any of that.

## Current State
`deliverable/index.html` — one file, Matter.js via CDN. Each figure is a
6-part Matter.js body group (torso, head, 2 arms, 2 legs) joined by
constraints; a small per-tick force controller holds the pose and drives the
walk cycle while "controlled" (walking/idle/fleeing/getting up), and is
switched off entirely while ragdolled (falling/flying), letting gravity and
collisions take over. 12 live dev-panel sliders per the user's spec (2
platform heights, thickness, size, fall rate, catapult rate, mid-air punch
scale, physics intensity, spawn rate, reaction distance, run distance,
randomization).

## Decisions
- Single self-contained HTML file, no build step — matches the project's
  small scope and the existing Quiz Game / Chess Prompt Writer / Living
  Globe single-file convention (`J:\CLAUDE\PROJECTS\CLAUDE.md` §11).
- Hybrid kinematic/ragdoll control (not a full walking physics simulation,
  not keyframed clips) — a per-tick PD-style force pulls each limb toward a
  procedurally-computed target while "controlled," released completely on
  ragdoll. This is the standard "active ragdoll" pattern games use for
  exactly this walk-then-get-knocked-down-then-recover shape.
- Collision-filter mask toggling (not teleportation) makes figures fall
  through their departure platform and land on the target one.
- Stickman size slider applies to newly-spawned figures only — live
  rescaling of an existing constrained body group wasn't attempted (real
  risk of broken joints, out of scope for this pass).

## Known Limitations
See `../PROJECT_PROGRESS.md`'s session 2 "Known limitations" update — the
short version: mechanically verified stable (no NaN, no runaway, no
stranded figures across a 9-figure/33-second stress test including a fall
and a catapult), but real-time visual smoothness was never actually seen —
this session's browser tooling never composited a live frame for the page,
so verification was data-only (positions/velocities/angles), not visual.

## Next Action
Open `deliverable/index.html` in an actual browser and just look at it —
judge the walk cycle, the shadow look, the catapult arc, and whether the
160px drift-recovery safety net (see PROJECT_PROGRESS.md) is ever visibly
noticeable as a "pop." Report anything that looks off so it can be tuned
against real observed behavior.
