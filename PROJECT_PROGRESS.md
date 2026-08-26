# STICKO — Project Progress

Stick-figure physics playground: figures walk/idle on two platforms on
their own, and clicking them triggers Matter.js-driven falls, catapults,
mid-air punches, and flee reactions — no scripted/keyframed animation
anywhere in the reaction system. This log covers session 1 (2026-08-26,
project setup + first build), session 2 (2026-08-26, restyle + a real
debugging pass that found and fixed why figures never appeared/landed),
and session 3 (2026-08-26, a two-category collision redesign + splitting
every limb into two jointed segments + a real size-vs-blur shadow rework).
Session 4 (2026-08-26) found and fixed the root cause of the floating bug
session 3 left as a known limitation — see below. Session 5 (2026-08-26)
replaced force-based walking with fully kinematic walking, eliminating the
sliding/floating problem class outright rather than continuing to tune it.

**Single self-contained HTML file, Matter.js from a CDN.** No backend, no
build step, no bundler — see `.claude/CLAUDE.md` for why this is a
deliberate exception to the project's default folder skeleton.

---

## Session 1 — 2026-08-26 — project setup + first build

### Project setup
Created the STICKO folder structure, mirroring DOGGO's conventions per
`J:\CLAUDE\PROJECTS\CLAUDE.md` §11 (own `.claude/CLAUDE.md` importing
`Working_With_Me.md`/`Technical_Background.md`, own `.claude/ETA History.md`,
`README.md`, `PROJECT_PROGRESS.md`, `docs/PROJECT_SUMMARY.md`,
`docs/CODE_SUMMARY.md`) — with the deliberate single-file-architecture
exception explained in `.claude/CLAUDE.md` (no `src/`, no `data/`, no
`scripts/`, since none of those apply to a build-step-free client-side toy).

### What was implemented

**Physics/behavior architecture** — every figure is a 6-part Matter.js
rigid-body group (torso, head, 2 arms, 2 legs) connected by
`Matter.Constraint` joints that exist permanently. A figure is never
literally "in ragdoll mode" as a separate code path — it's controlled by
whether a per-tick corrective force (`applyLimbControl` /
`moveTowardLocal`, a small PD-style spring pulling each limb toward a
procedurally-computed target) is being applied that frame:

- **Controlled states** (`entering`, `walking`, `idle`, `fleeing`,
  `gettingUp`) — the force is applied. Legs/arms swing via a phase
  variable advanced by `dt * speed` (so faster walking = faster cadence,
  not a fixed-speed clip); idle state uses a slower phase + small torso
  bob for a "breathing/weight-shift" idle rather than a frozen pose.
- **Ragdoll states** (`falling`, `flying`) — the force is skipped
  entirely. Gravity, joint constraints, and collisions are the only thing
  moving the body from that point on.

This means the actual fall/catapult/punch motion is 100% physics-engine
output — nothing about *how* a figure tumbles, flails, or lands is
authored; only the initial impulse (fall: small nudge + an extra
fall-rate-scaled downward force; catapult: a strong upward velocity set on
every part; punch: a velocity kick away from the click point) is
code-driven, same as giving a real ragdoll a shove.

**Falling through platforms**: rather than teleporting a figure or
disabling all collision, each figure's collision mask toggles off just the
one platform category it's currently leaving (`setCollidesWithPlatform`),
so it falls straight through that platform's own collider while everything
else (the other platform, other figures) still collides normally. Restored
automatically once it's clearly clear of that platform, so return trips
(bottom → top → bottom → ...) keep working.

**Landing + recovery**: `checkLanding` watches for the torso settling near
the target platform's height at low vertical speed for a few consecutive
frames, then reassigns `homePlatform` to wherever it actually landed (so
the toy is cyclical — click a landed figure again and it goes the other
way), restores full platform collision, and hands control back
(`gettingUp` state) with a brief timer plus the same upright-angle
correction used everywhere else — a physics-applied "stand up," not an
animation.

**Click interactions**: `Query.point` hit-tests every figure's body parts
directly (not a bounding-circle approximation) to find what was clicked.
Clicking a figure standing on a platform triggers its fall/catapult and
also runs `triggerFlee`, which checks every other walking/idling figure's
distance to the click and sends the ones inside the reaction-distance
threshold off to a random point `runDistance` away (both jittered by the
randomization slider) before they settle back into idling/walking on their
own. Clicking an already-airborne figure applies a punch impulse away from
the click point instead.

**Dev panel** — 12 live sliders exactly matching the request: top/bottom
platform height, platform thickness, stickman size (new spawns only), rate
men enter frame, rate of fall, rate of catapult, mid-air click effect
scale, physics engine intensity (scales gravity + most applied
forces/impulses together), reaction distance, run distance, randomization.
Every slider writes straight into a shared `settings` object read live by
the simulation each tick — no reload, no apply button. Platform
height/thickness changes rebuild the two static platform bodies in place.

**Rendering**: custom Canvas 2D drawing (Matter's own debug renderer was
not used) — rounded-rect torso, circular head with two eye dots, thick
round-capped limb strokes, per-figure randomized skin tone + shirt/pants
color pulled from small fixed palettes, so figures read as varied small
people rather than identical stick men.

### Verification

Syntax-checked via `node --check` against the script block extracted from
`index.html` (with `Matter`/`window`/`document`/`canvas` stubbed out) — no
syntax errors. Manually re-derived the spawn-height geometry (platform
surface → leg length → torso height → head radius) and confirmed the same
arithmetic is reused consistently between figure spawning and the
walking/idle target-height spring, so a figure's rest pose should line up
with the platform surface.

**Not verified**: actual behavior in a browser. The user explicitly asked
not to browser-test this session, so force/impulse magnitudes (walk force
gain, catapult launch speed, punch impulse, joint stiffness/damping,
gravity scale) are sized from typical Matter.js example scale and internal
consistency only — not observed or tuned against a real running page.
Whether the catapult actually reaches the top platform, whether the
ragdoll looks floppy-but-controlled rather than exploding or too stiff, and
whether the walk cycle reads as natural are all open questions.

### Known limitations
- Force/impulse tuning unverified live (see above) — the physics-intensity,
  fall-rate, and catapult-rate sliders exist specifically so this can be
  adjusted without touching code, but a genuinely broken interaction
  (never reaching a platform, ragdoll flying apart) hasn't been ruled out.
- Stickman-size slider only affects newly-spawned figures; no live
  rescaling of already-built joint groups.
- No despawn/exit logic — figures accumulate up to the fixed `MAX_FIGURES`
  cap (40, not exposed as a slider since it wasn't requested) and then stop
  spawning until the count drops, which currently never happens (nothing
  removes a figure). Not raised as a problem yet since it wasn't in scope,
  but worth knowing for a long-running session.

### What should be built/checked next
1. Open `deliverable/index.html` in a real browser and watch it run.
2. Tune `catapultRate`/`intensity`/`fallRate` defaults if the flight looks
   too weak/strong, and `applyLimbControl`'s force gain if the walk cycle
   looks jittery or too stiff.
3. Decide whether a long-running session needs figures to eventually leave
   the scene (despawn at the edges, or a hard swap-out past the cap) —
   not requested yet, flagging only because unbounded accumulation was a
   deliberate scope cut this session, not an oversight.

---

## Session 2 — 2026-08-26 — restyle + real debugging pass

User request: white background / black platforms+figures with a
drop-shadow slider, plus "I don't see any stickmen, I don't think they're
landing" — and explicit permission to browser-test this time. Restyling
was quick; the bug report turned into a real multi-bug debugging session,
since the app had never actually been run before (session 1 was built and
shipped without browser verification, per the user's explicit instruction
at the time).

### Restyle (as requested)
- Background white, platforms/figures solid black (dropped the random
  per-figure skin/shirt/pants palette entirely), head's face dots switched
  to white for contrast against the now-black head.
- Added `ctx.shadowColor/shadowBlur/shadowOffsetY` around the
  platform+figure draw calls, driven by two new sliders — **shadow size**
  and **shadow distance** — bringing the dev panel to the requested 12
  sliders.

### Debugging: why nothing appeared / nothing landed
Set up a local static server (`.claude/launch.json` + Python
`http.server`, since `file://` previews render as static snapshots in this
tool) so the page could actually run and be inspected live. Found and
fixed, in the order discovered:

1. **Canvas stayed 0×0.** `resize()` read `window.innerWidth`/`innerHeight`
   directly at first script execution; in at least one real environment
   those were still `0` at that exact moment. Fixed with a
   `requestAnimationFrame`-retrying `initialResize()` that waits for a real
   nonzero size before sizing the canvas. This alone explained "I don't see
   any stickmen" outright — the whole simulation had been running against a
   zero-sized world.
2. **`normAngle` could hang the tab.** It un-wrapped an angle into
   `[-π, π]` with a linear `while` loop — fine for normal angles, but if a
   figure's angle ever grew large (see #4), the loop could run for a very
   long time and freeze the renderer. Rewrote it as an O(1) modulo. Caught
   this by driving the simulation manually (see below) and hitting a real
   45-second CDP timeout.
3. **The vertical "hold at platform height" force had an inverted sign**
   (`y: -dyv * gain` instead of `y: dyv * gain`), in all three places that
   used it (walking/fleeing/gettingUp, idle, and the now-removed entering
   state). It was pushing figures *away* from the platform, compounding
   with gravity — a textbook positive-feedback bug. This is the direct
   cause of "not landing." Fixed the sign in all three, then later removed
   the force entirely (see #6) once it was clear real collision could hold
   figures up on its own.
4. **The torso's upright-correction controller was wildly unstable.** It
   directly `Body.setAngularVelocity`'d a `0.75*prev + 0.12*angErr`
   result every tick — reasonable-looking on paper, but Matter.js
   expresses angular velocity per *engine step*, not per second, so a gain
   of `0.12` against a possible `π` radian error is enormous. A freshly
   spawned figure was measured spinning from angle 0 to ~2.8 rad in 11
   ticks (183ms). Fixed by (a) adding a shared `settleAngle()` helper with
   a much smaller gain and a hard clamp on the resulting angular velocity,
   used at all 4 call sites (torso × 3 states, limb-to-torso tracking), and
   (b) giving the torso a much higher rotational inertia
   (`Body.setInertia(torso, torso.inertia * 40)`) so limb-swing reaction
   torque can't push it around as easily — the standard "stiffen the core"
   trick for this kind of hybrid kinematic/ragdoll controller.
5. **The limb position-control spring (`moveTowardLocal`) could shock the
   system.** A freshly-spawned figure's walk phase starts at a random
   value, so on the very first tick the legs could be asked to jump to a
   full-amplitude swing target instantly. Combined with a stiff, lightly
   damped spring (`k=0.028`, `damp=0.006`), this injected a real energy
   spike that grew into oscillation over a few ticks — measured torso
   vertical velocity climbing to ±30+ within 8 ticks. Softened the spring
   (`k=0.01`, `damp=0.018`), added a hard per-tick force clamp, and reduced
   the limb-to-torso angle-tracking gain the same way as #4.
6. **Removed the artificial vertical "hold height" spring entirely** for
   walking/idle/fleeing/gettingUp once collision was confirmed to hold
   figures up correctly on its own (verified in isolation: a figure with
   zero custom forces settles cleanly onto a platform via real Matter.js
   resting contact). Letting real collision do the job, instead of a hand-
   tuned force fighting it, removed an entire class of bugs at once.
7. **Catapults that fell short had no way to land.** `checkLanding` only
   ever checked proximity to the *intended* target platform (top, while
   `flying`) — a catapult that didn't have enough energy to clear the gap
   and fell back down near its own launch platform was never recognized as
   landed, and could sit there in `flying` limbo indefinitely. Rewrote
   `checkLanding` to check proximity to *both* platforms and land on
   whichever one it actually settled near; also raised the base catapult
   power (14 → 22) so falling short is rarer at default settings.
8. **General safety nets**, added after live testing kept surfacing edge
   cases faster than each root cause could be chased individually:
   velocity is now hard-clamped every tick (±16 while controlled, ±45
   while airborne — generous enough for normal physics, tight enough to
   stop a runaway or platform tunneling); a figure that ends up more than
   160px from its home platform's expected height while in a controlled
   state gets gently translated back (whole ragdoll together, preserving
   relative pose, not just the torso) rather than left stranded.

### Verification
All of the above were found and confirmed fixed by actually running the
page — `Matter.Engine.update` was monkey-patched to prove `rAF` never
fires in this tool's own backgrounded browser tab (`document.hidden` stays
`true` even in "real Chrome" via the Claude-in-Chrome connector, which is
a tooling limitation, not an app bug), so verification was done by
temporarily exposing the closured `tick`/`figures`/`engine` objects on
`window` and manually driving the simulation forward with synthetic
timestamps — the same production code path a real `requestAnimationFrame`
loop would call, just invoked directly. That hook was removed before
finishing; nothing debug-only shipped in `deliverable/index.html`.

Final stress test: ~33 simulated seconds, continuous spawning (9 figures
accumulated), including one figure sent falling and one catapulted
mid-run. All 9 finished within roughly -3 to +44px of their home
platform's natural resting height, angles within ±0.17 rad of upright, no
NaN/Infinity, no runaway velocity, no figure lost off-screen.

**Not verified**: real-time visual smoothness/game-feel (walk-cycle look,
shadow appearance, exact catapult arc shape) — this tool's browser tab
never composites a live frame for this page (see above), so no real
screenshot of the running simulation was possible this session, only the
underlying physics data. The user should open `deliverable/index.html`
directly to judge how it actually looks and feels; report anything that
looks off; the manual-drive testing proves the mechanics are sound, not
that the motion looks good.

### Known limitations (session 2 update)
- Visual/gait polish is unverified (see above) — mechanically stable now,
  but whether the walk cycle *looks* natural rather than merely "doesn't
  explode" hasn't been judged by eye.
- The 160px drift-recovery safety net is a real fix for a real observed
  problem (figures do drift, most likely from figure-on-figure collisions
  physically shoving each other, though that specific mechanism wasn't
  isolated further given time already spent), but it treats the symptom
  more than a fully diagnosed root cause. If figures are still seen
  visibly "popping" back into position after a shove, that's this
  mechanism catching a drift it didn't prevent — worth a closer look if it
  turns out to happen often/visibly.

---

## Session 3 — 2026-08-26 — collision redesign, jointed limbs, shadow rework

Six requests in one batch: size default 0.55 + slider max 1; figures
shouldn't collide with each other while walking, only when one is
clicked/airborne; a top-platform figure falls through when clicked; an
airborne figure under the top platform must never hit its underside;
2-segment arms/legs (elbows/knees); body 25% thinner; and "shadow size"
redefined to mean actual geometric size (not blur), with blur split out
into a new "shadow sharpness" slider. Plus 3 follow-ups sent mid-turn:
spawn rate default 1s/max 3, shadow size+distance slider max ×4, remove
the bottom-left hint text.

### Collision redesign
Replaced the single `CAT_FIGURE` category with two: `CAT_FIGURE_WALK` and
`CAT_FIGURE_RAGDOLL`. A figure's parts switch category (and mask) via
`setAirborne(on)` at exactly two points — `sendFalling`/`sendCatapult` (on)
and `checkLanding`'s landed branch (off) — nowhere else needs to touch
collision filters:
- **Walking** mask includes both platforms + `CAT_FIGURE_RAGDOLL`, but
  *not* `CAT_FIGURE_WALK` — so two calm walkers never push each other, but
  a ragdoll figure colliding into a walker still works.
- **Airborne** mask includes the bottom platform + both figure categories,
  but never `CAT_PLATFORM_TOP` — so a falling or catapulted figure can
  never hit the top platform from any direction, for the entire time it's
  airborne, satisfying "falls through when clicked" and "never collides
  with the underside" as the same mechanism, not two separate fixes.
- The top platform's own `collisionFilter.mask` only ever includes
  `CAT_FIGURE_WALK`; the bottom platform's includes both — bottom is the
  universal floor, top is only solid for its own walkers.
This let the old `departFrom`/`awayFrom` collision-restore logic in
`checkLanding` (session 2) be deleted outright — it's no longer needed
once the category itself, not a per-platform toggle, decides collision.

### Two-segment limbs
Each arm/leg is now two Matter bodies (`upperArmL`/`foreArmL`,
`thighL`/`shinL`, and the R-side mirrors), doubling the joint count from 5
to 9 per figure. Rather than inverse kinematics, each lower segment
(`shinL`, `foreArmL`, etc.) targets a position computed directly off the
*upper* segment's real current end point (`localEndPoint`, hoisted to a
shared helper also used by `draw()`), offset by a small procedural
knee/elbow-bend angle tied to the walk phase — `chainLeg`/`chainArm` in
the code. Every new segment uses the exact same clamped, damped spring
controller (`springTo`) and angle clamp (`settleAngle`) that session 2
proved stable for the single-segment version — same gains, same force
caps, nothing re-tuned from scratch. That carried over cleanly: a
live-driven stress test (11 figures, ~12 simulated seconds) landed
everyone within 12-24px of their platform, angles ≤0.07 rad, on the first
attempt — no repeat of session 2's multi-hour instability hunt.

### Thinner body
`torsoW` now includes a flat `* 0.75` factor (25% thinner). Nothing else
(torso height, limb thickness, head) was touched — "body" was read as the
torso specifically.

### Shadow rework
The built-in `ctx.shadowBlur/shadowColor/shadowOffset` mechanism (which
only offers blur, not a real size control) was replaced with an explicit
shadow-drawing pass, drawn before the real black shapes each frame:
- Platform shadow: a separate rect below the platform, `height =
  thickness + shadowSize*2` (grows outward from the real platform as
  "size" increases), offset down by `shadowDistance`.
- Figure shadow: a squashed ellipse at the figure's actual current foot
  position (tracks it while airborne too, not just standing), radius tied
  to `shadowSize`.
- Both use `ctx.filter = "blur(Npx)"` for softness, computed from the new
  **shadow sharpness** slider as `blur = 30 - sharpness` — higher
  sharpness slider value = crisper edge, matching the intuitive direction.

### Mid-turn follow-ups
Applied directly: `spawnRate` default 2.0→1.0, slider max 10→3;
`shadowSize` slider max 40→160, `shadowDistance` slider max 50→200 (both
×4 per request); removed the `#hint` div and its CSS entirely (was the
"click a stick figure..." text, bottom-left).

### Verification
Same manually-driven technique as session 2 (this tool's browser tab still
never composites a live frame). Confirmed directly:
- Collision filter bit math for both a walking figure (`category=8,
  mask=22`) and a just-triggered-falling one (`category=16, mask=28`)
  matches the intended constants exactly.
- A figure sent falling from the top platform passed straight through
  `y≈140` (the top platform's own height) with no deflection, continuing
  smoothly down to the bottom.
- A catapulted figure's natural ballistic apex landed almost exactly at
  the top platform's height (`y≈137` vs `topY=140`) with `vy≈0` — and it
  sailed through that height with no collision interference on the way up
  either, confirming the underside-collision fix works for both directions
  of travel, not just falling.
- 11-figure spawn stress test (~12 simulated seconds): all figures within
  12-24px of their platform, angles ≤0.07 rad, no NaN, no runaway.

**Not verified**: real-time visual appearance, same limitation as session
2 — nobody has watched this run in a real composited browser frame yet.

### Known limitations (session 3 update)
- **A small residual landing overshoot was observed and not chased down.**
  In the catapult-apex test above, once the figure's category flipped back
  to `WALK` and platform collision re-engaged near the top platform, it
  ended up settling noticeably higher than its computed target height
  (~103px off — under the 160px drift-recovery threshold, so the safety
  net didn't correct it) rather than resting exactly at the expected
  torso height. The figure was still upright, on the correct platform, and
  fully interactive — this is a positioning polish issue, not a broken
  interaction — but it's a real, observed imperfection worth a closer look
  if it turns out to be common or visually obvious, rather than something
  confirmed fixed.
- Visual/gait polish for the new 2-segment limbs specifically (does the
  knee/elbow bend read as natural, or too subtle/too much) is unverified
  the same way session 2's single-segment walk cycle was — open it in a
  real browser and look.

---

## Session 4 — 2026-08-26 — root-caused the floating bug, single-entry gating, proportions

User reported figures "walking above the platforms" and "cannot float" —
confirming session 3's disclosed-but-unconfirmed catapult overshoot was in
fact a general, always-present bug, not a rare edge case. Also: only one
figure should ever be walking into frame at a time (spawn rate shared
across both platforms, serialized); torso 25% shorter with the removed
height added to the legs instead; spawn rate default back to 2s.

### Root-causing the float (the real story, not the first theory)
Measured the actual foot-to-platform gap directly (a live-driven test, same
technique as sessions 2-3): **19.67px**, on a leg only 16.5px long — the
foot wasn't just sagging, it was floating higher than its own leg length,
confirming this was never "natural sag."

The investigation went through three wrong turns before finding the real
fix, worth recording so a future session doesn't repeat them:

1. **First suspect: the thigh had spun to -94 radians** — a literal
   windmill, inherited by the shin through the chain. Real joint/spring
   reaction torque was overpowering the angular-velocity clamp
   (`settleAngle`'s `maxAV`) over hundreds of ticks — the clamp bounds
   *speed* of rotation, not *whether* it happens, so a small sustained bias
   can still accumulate into many full rotations. Fixed by giving every
   limb segment the same rotational-inertia boost session 2 used for the
   torso (`Body.setInertia(part, part.inertia * N)`) — this stopped the
   spinning, but the float (now ~14-19px, not a symptom of spinning) was
   still there.
2. **Second attempt: reintroduced a vertical "hold torso at ground height"
   force** (`holdGroundHeight`), carefully re-deriving the sign against the
   session-2 postmortem to avoid repeating that exact bug. The sign was
   correct. It still made things *worse* — isolated by selectively
   disabling pieces of the control code live: with `holdGroundHeight`
   *and* limb control both off, the figure was rock-solid via real
   collision alone (`vy` settling to exactly 0). Re-enabling
   `holdGroundHeight` alone reintroduced a hard oscillation (`vy` flipping
   between the ±12 clamp bounds every few frames, y ranging 96-118).
   **Lesson generalized beyond the sign bug session 2 found**: *any*
   custom force targeting a position right at a real collision boundary
   fights the (effectively infinitely stiff) collision response, sign
   notwithstanding. Removed it again, permanently this time.
3. **The actual cause**: with `holdGroundHeight` gone and limb control
   back on, the persistent upward drift remained (`vy` still pinned at the
   clamp). Bisecting further (disabling just the chained segments' own
   position-spring, leaving only their angle control) helped some but
   didn't fully resolve it. The real fix was that the *root* segments'
   spring (`springTo`, used for thigh/upperArm's torso-relative
   positioning) was simply too strong for a multi-joint chain — reducing
   its gain sharply (`k` 0.008→0.002, `damp` 0.05→0.08, `maxF`
   0.3*mass→0.15*mass) resolved the great majority of it. Average gap
   across a 15-figure test dropped to **-6.5 to 2.0px** (essentially
   correct), worst case ~16px (down from the original 19.67px baseline
   *and* down from an intermediate 100+px-worst-case regression hit along
   the way) — a real, large improvement, not a full mathematical zero.

Also, while chasing this: removed the chained segments' (shin/forearm)
redundant position-spring entirely — they now only get an angle nudge
(`chainLeg`/`chainArm`), leaving position to the real knee/elbow joint,
since driving position from *both* a spring and a real joint for the same
relationship was itself part of what was fighting. And added a hard
post-physics velocity clamp (`clampVelocities()`, called after
`Engine.update`, not just at the top of each figure's `applyBehavior`) —
the original clamp only caught velocity from the *previous* frame, since
it ran before that frame's own forces were applied.

### Single-entry gating
`spawnFigure` sets `hasArrived = false`; it flips to `true` the first time
that figure's initial walk-in reaches its target and transitions to
`idle`. `stepBehavior`'s spawn check gates on `!someoneStillEntering()` in
addition to the existing timer, so the "rate men enter frame" slider now
governs one shared timer across both platforms *and* guarantees at most
one figure is ever mid-entrance — verified live (`maxSimultaneousEntering:
1` across a 20-second stress test).

### Proportions
`torsoH` now includes `* 0.75` (25% shorter); the removed 11×size units of
height were added to the legs, split roughly by their existing ratio
(`thighLen` 16→22, `shinLen` 14→19 × size — legs now ~41×size total vs.
30×size before). `spawnRate` default reverted to 2.0 (slider max stays 3,
per the prior turn's request).

### Known limitations (session 4 update)
- **The float is greatly reduced, not eliminated.** Worst observed case
  across a 15-figure stress test was ~16px (on a now-~22.5px-long leg at
  default size) — noticeably better but still a real, measurable gap in
  some cases. If it's still visually bothersome, the next lever to pull is
  probably the hip/knee joint `stiffness` values (currently 0.55-0.7),
  which weren't touched this session once the `springTo` gain reduction
  had already resolved the bulk of it.
- This was root-caused through live iterative testing, not fully derived
  analytically — the exact mechanism by which an over-strong `springTo`
  gain produces a *sustained one-directional* drift (rather than a
  symmetric oscillation) was observed and fixed, but not fully explained.

---

## Session 5 — 2026-08-26 — kinematic walking (the real fix)

User: "the stickmen aren't really walking, they end up sliding around" —
and suggested the actual right architecture directly: kinematically
translate the whole figure horizontally and just animate the legs, rather
than physically simulating the walk. This is a better fix than anything
tried in session 4, and it isn't a workaround — it eliminates the entire
problem class (force fighting collision/joints) instead of tuning it.

### What changed
`applyLimbControl`/`springTo`/`springLocal`/`chainLeg`/`chainArm` — the
entire force-based walk controller from sessions 1-4 — are gone. In their
place, while a figure is `walking`, `idle`, or `fleeing`:
- `poseKinematic()` runs every tick and directly sets every body's
  position and angle (`Body.setPosition`/`setAngle`, velocity zeroed) —
  nothing is chased with a force, so there's nothing left to fight
  collision or the joint solver.
- The torso moves via a tracked `this.kinX` (updated by
  `dir * 96 * speedMul * dt`, i.e. real px/second, clamped so it can't
  overshoot `walkTargetX`) and sits at a deterministic standing height —
  no physics involved in "staying on the platform" at all for these
  states.
- Legs and arms are placed by real forward kinematics from the hip/
  shoulder outward (`placeLegKinematic`/`placeArmKinematic`): thigh angle
  from a swing formula, knee position derived from the thigh's actual
  placed end point, same chain for the shin — geometrically exact every
  frame, not converging toward something.
- Fixed a real bug found while writing this: arms were resting horizontal
  (built with a horizontal local axis, angle 0 = pointing straight out to
  the side — a "T-pose"), never corrected in any prior session since none
  of them had a working live screenshot to catch it visually. Arms now
  hang from a `sign * PI/2` baseline angle with the swing applied around
  that, so they hang down at rest like a real figure's arms.
- Real ragdoll physics (falling/flying/punch, `checkLanding`) is untouched
  — the kinematic layer only applies to the 3 "controlled and stationary
  on a platform" states. `gettingUp` also stays a real physics phase (just
  an angle-correction nudge + real collision, the same combination proven
  stable in session 4's isolation tests) until it's upright, then hands
  off to kinematic walking.

Also this turn: arms lengthened 10% (`upperArmLen`/`foreArmLen * 1.1`).

### Verification
Live-driven test, 10 figures over ~15 simulated seconds: feet-to-platform
gap **-0.2 to 2.9px** for every walking/idle figure (previously ~16px
worst case after session 4's tuning pass) — essentially exact, not just
improved. Torso angle reads exactly `0` for every controlled figure (no
tilt at all, as expected — nothing physical is acting on them). Confirmed
`torso.position.x === kinX` exactly, confirming the kinematic sync is
correct. Separately re-verified falling/catapult still work normally
(ragdoll physics untouched) and single-entry spawn gating still holds
(`maxSimultaneousEntering: 1`).

### Known limitations (session 5 update)
- All of session 4's "float not eliminated" limitation is superseded —
  it's now effectively eliminated for walking/idle/fleeing. What's left is
  the same as before: nobody has watched this move in a real composited
  browser frame, so whether the *look* of the walk cycle (leg swing
  amplitude/timing, arm counter-swing) actually reads as natural gait is
  still unverified, only that it's geometrically exact and physically
  stable.
- A controlled (walking/idle) figure is effectively immovable by
  collision now — since its position is force-set every tick, a ragdoll
  figure bumping into it won't budge it (the ragdoll bounces off normally;
  the walker doesn't react). Not something the user has asked about; flag
  if it ever looks wrong in practice.

### Repository
Committed and pushed: `STICKO/` was untracked until this session (the
git repo root is shared across all of `J:\CLAUDE\PROJECTS`, one level up —
scoped `git add STICKO/` specifically, since the repo has a large amount
of unrelated, uncommitted work in progress across other project folders).
Pushed to `origin/master` at `https://github.com/LeisHo/GG.git`,
commit `5e24e3d`.
