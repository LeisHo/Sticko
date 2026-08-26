@Working_With_Me.md
@Technical_Background.md

---

# General Project Instructions

- **Single-file architecture, on purpose.** `deliverable/index.html` is the entire app — HTML, CSS, and JS in one file, physics engine pulled from a CDN (`<script src="...matter.js">`). This is the deliberate exception §11 of `J:\CLAUDE\PROJECTS\CLAUDE.md` allows for a project this small (same convention as Quiz Game / Chess Prompt Writer / Living Globe). No `src/`, no build step, no bundler.
- No `data/` or `scripts/` folders — this project has no data acquisition and no build/maintenance scripts to house. Don't create them just to match the generic skeleton; add them later only if a real need shows up.
- `deliverable/` — the actual running app (`index.html`). This is both the source and the shippable artifact, per DOGGO's convention of `deliverable/` holding "final deliverable files / index html".
- `docs/` — project summary and code summary pointers (see `PROJECT_PROGRESS.md` for the real narrative log, same split as DOGGO).
- Procedural only, never a hand-authored keyframe animation — but not all procedural motion is physics-*simulated*. Walking/idling/fleeing is kinematic: position/angle are computed directly each frame by formula (real forward kinematics for the limbs) and set outright, not chased with a force — this is deliberate (see PROJECT_PROGRESS.md session 5), not a shortcut to fix later. Falling, flying, punches, and getting back up are genuine Matter.js physics the whole way through. If a behavior needs scripting, script the *rule* or *formula*, not a canned motion clip.
- All dev-panel sliders must be live — dragging one updates the running simulation immediately, no page reload, no "apply" button.
- Keep the visual rendering (limb drawing, colors, proportions) and the physics/behavior logic (Matter.js bodies, constraints, state machine) as separate concerns within the one file — easy to find, not necessarily separate files, per this project's single-file exception.
