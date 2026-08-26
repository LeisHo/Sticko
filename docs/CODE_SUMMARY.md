# STICKO — Code Summary

Status: see `../README.md` for how to run it and the full feature list; this
file is a quick orientation pointer into `deliverable/index.html`, not a
duplicate.

## File Map
Everything lives in `deliverable/index.html` (single-file architecture, see
`../.claude/CLAUDE.md`). Inside the one `<script>` block, top to bottom:

- **settings** — the object every dev-panel slider writes into; read live
  by the physics/behavior code every tick.
- **world setup** — `engine`, `platformTop`/`platformBottom` (static Matter
  bodies, rebuilt on height/thickness slider change), collision-filter
  category constants.
- **Figure** — the class for one stick figure. Constructor builds a 10-part
  Matter body group (torso, head, upperArm/foreArm × 2, thigh/shin × 2) + 9
  joint constraints, and gives every part extra rotational inertia
  (`Body.setInertia`, torso ×8, limbs ×4) so real joint/collision reaction
  torque can't spin a part out on its own. Key methods:
  - `applyBehavior(dt)` — the per-tick state machine (`walking`, `idle`,
    `fleeing`, `gettingUp`, `falling`, `flying`). `walking`/`idle`/
    `fleeing` are **fully kinematic** (see below) — `gettingUp` is a brief
    real-physics recovery phase (angle correction + real collision, no
    limb forces) that hands off to kinematic walking once upright.
    `falling`/`flying` are full ragdoll physics throughout.
  - `poseKinematic(moving, speedMul)` / `placeLegKinematic(...)` /
    `placeArmKinematic(...)` — while walking/idle/fleeing, every part's
    position and angle is computed by direct forward kinematics from
    `this.kinX` (a plain tracked number, moved by `dir * px/sec * dt`) and
    `this.phase`, then **set outright** (`Body.setPosition`/`setAngle`,
    velocity zeroed) every tick — no force is ever applied for these
    states, so there's nothing to fight collision or the joint solver.
    This replaced an earlier force/spring-based walk controller (sessions
    1-4) that could never fully stop sliding/floating no matter how it was
    tuned — see PROJECT_PROGRESS.md session 5 for why the kinematic
    approach is the real fix, not a workaround.
  - `settleAngle(body, angErr, decay, gain, maxAV)` (module-level helper,
    not on Figure) — the shared, clamped angular-velocity corrector, now
    only used for the torso during `gettingUp`. Don't reintroduce a raw
    `Body.setAngularVelocity(body, prev*decay + err*gain)` without a clamp
    — Matter expresses angular velocity per engine step, not per second,
    so an unclamped gain against a large error can spin a figure out in a
    handful of ticks.
  - `sendFalling` / `sendCatapult` / `punch` — the three click-triggered
    physics events; each flips the figure to the `CAT_FIGURE_RAGDOLL`
    collision category (`setAirborne(true)`) and/or sets body velocities,
    nothing else.
  - `checkLanding` — proximity+low-speed landing detection, checked against
    *both* platforms (not just the intended one — a catapult that falls
    short still needs to land back on its origin platform), plus a `lost`
    fallback that snaps the whole figure (all parts together, preserving
    pose) back near the nearer platform if it ends up far past either one.
  - `draw(ctx)` — all rendering; reads body positions/angles only, never
    writes them.
- **spawning** — `spawnFigure()`, called from `stepBehavior()` (see render
  loop below) on the spawn-rate slider's interval, soft-capped at
  `MAX_FIGURES = 40`.
- **click handling** — `figureAt(x,y)` (Matter `Query.point` hit-test over
  every figure's parts), `triggerFlee(clicked)` (reaction/run-distance
  logic), and the canvas `click` listener that dispatches to
  sendFalling/sendCatapult/punch based on the clicked figure's state.
- **slider wiring** — `bindSlider(...)`, one call per dev-panel control.
- **render loop** — `stepBehavior(dt)` (spawn accumulator + every figure's
  `applyBehavior`) runs first using **real elapsed time** (`dtMs/1000`,
  capped at 100ms), then `Engine.update`, then a draw pass. All three run
  inside one `requestAnimationFrame` `tick(now)` loop. Earlier this used a
  Matter `beforeUpdate` event with a hardcoded `dt = 1/60` regardless of
  actual frame rate — under any throttling that assumption silently ran
  the whole simulation in slow motion; don't reintroduce a fixed-dt
  assumption here.

## Architecture
Rendering (`draw`) and physics/behavior (`applyBehavior` and everything it
calls) are kept as separate concerns in the code even though they're one
file, per `../.claude/CLAUDE.md`. `draw` never mutates a body; behavior code
never touches `ctx`.

The core trick the whole toy rests on: every body part always has its
physics joints, all the time — but a "controlled" figure (walking/idle/
fleeing) has its position/angle **directly set** every tick (kinematic,
no force involved), while a "ragdoll" figure (falling/flying) has that
direct control switched off entirely, leaving gravity/collision/joints to
move it. `gettingUp` is real physics too (a brief recovery phase), not
kinematic. There's no in-between state where forces chase a target — that
hybrid was tried across sessions 1-4 and is what caused the
sliding/floating that session 5 replaced.

## Untouchable Systems
None formally designated — still early, single-developer. That said, prior
sessions verified the following the hard way and they shouldn't be
casually reworked without re-testing live: the O(1) `normAngle`, the
`settleAngle` clamp, the two-collision-category design (`CAT_FIGURE_WALK`/
`CAT_FIGURE_RAGDOLL`, see `setAirborne`), `checkLanding`'s both-platforms
check, and — the newest and most load-bearing — the fact that
walking/idle/fleeing never apply a force to move a figure, only direct
position/angle sets. Reintroducing a spring/force for these states is
almost certainly reintroducing the sliding bug.

## Gotchas
- **`window.innerWidth`/`innerHeight` can read `0` at initial script
  execution** in some embedding contexts. `resize()` is called through a
  small `initialResize()` retry loop (`requestAnimationFrame` until a
  nonzero size shows up) specifically because of this — don't replace it
  with a bare `resize()` call.
- **Matter.js angular/linear velocity is per engine step, not per second.**
  A raw `Body.setAngularVelocity`/large unclamped force gain that looks
  small on paper can spin or launch a figure out within a handful of ticks
  (measured: 0 → 2.8 rad in 11 ticks, and separately a thigh to -94 rad
  before each was fixed). Any new *physics* control (only relevant to
  `gettingUp`/ragdoll states now) should go through `settleAngle`, not a
  raw `applyForce`/`setAngularVelocity` call with an untested gain.
- **Arms are built horizontal** (local +x is the long axis) — a baseline
  angle of `sign * PI/2` is what makes them hang down at rest; angle `0`
  is a T-pose. This was wrong for 4 sessions before anyone actually looked
  at it running (see PROJECT_PROGRESS.md session 5) — a reminder that this
  project's verification has always been data-only (positions/velocities),
  never a real rendered screenshot, so purely-visual mistakes like this
  can survive a long time undetected.
- Changing the stickman-size slider does **not** resize already-spawned
  figures — it only affects ones spawned after the change. Live-rescaling a
  constrained Matter body group is nontrivial (joint anchor points are in
  local coordinates and would all need re-deriving) and was out of scope.
- A walking/idle figure is kinematic, which means it's effectively
  immovable by collision — a ragdoll figure bumping into one will bounce
  off it, but the walker itself won't react. Intentional (see
  PROJECT_PROGRESS.md session 5), but worth knowing if it ever looks wrong.
