# Gotchas & lessons — Tidepool build log

Working notes from building this against three.js r185 WebGPURenderer + TSL,
deploying to GitHub Pages, and debugging it live. Read this before making
changes — several of these will silently reintroduce a real bug.

## Code / TSL

**`floor()` name collision — the actual outage this repo shipped with.**
TSL exports a function called `floor()`. It's easy to also want a JS
variable called `floor` (a `THREE.Mesh` for a floor, a loop variable, etc).
Don't. A `const floor = new THREE.Mesh(...)` anywhere in the same module
scope as a TSL `floor(...)` call will shadow the import for the *entire*
scope, including inside `Fn(() => {...})` closures defined earlier in the
file — because those closures don't execute until the GPU builds the node
graph, which happens after all your `const` declarations have already run.
Result: `Uncaught TypeError: floor is not a function`, thrown on every
frame the compute shader rebuilds. Same risk applies to any TSL export
that's also a tempting plain-English variable name — `mix`, `cross`,
`normalize`, `abs`, `float`, `color`, `pass`. Before naming a local
variable, grep it against your `three/tsl` import list.

**Before touching TSL code, download the exact pinned-version examples
and grep them — don't rely on training-data memory of the API.** The
compute/TSL surface has moved fast (RenderPipeline replaced the old
PostProcessing class; bloom moved to `three/addons/tsl/display/BloomNode.js`
with a new signature). Pull `https://raw.githubusercontent.com/mrdoob/three.js/rXXX/examples/webgpu_*.html`
for the version you're pinning and copy patterns from there, not memory.

**`BloomNode`'s `strength`/`radius`/`threshold` accept live TSL uniforms.**
Constructor: `strength.isNode ? strength : uniform(strength)` — pass your
own `uniform(...)` object instead of a plain number and you can adjust
bloom intensity every frame (e.g. from a UI slider) without rebuilding the
`RenderPipeline`.

**`material.positionNode` replaces `positionLocal`, not the final world
position.** It's still subject to the normal model/view/projection chain
afterward (confirmed in `NodeMaterial.js`: `positionLocal.assign(...)`).
For objects with an identity transform (the common case — object added
directly to the scene, never repositioned) this is numerically
indistinguishable from writing absolute world-space coordinates, which is
why compute-driven `Sprite` particle systems "accidentally" work either
way. Don't rely on that coincidence if you ever give the mesh its own
transform.

**Negate with `negate()`, never unary `-`, inside TSL node graphs.**
Straight from the official birds/boids example: `-node` silently resolves
to NaN. Every sign flip in a matrix/vector expression needs `negate(x)`.

**`renderer.compute()` genuinely works on the automatic WebGL2 fallback**
via real transform-feedback double-buffering (see `WebGLBackend.js`,
`compute()` — not a stub). Don't assume "no WebGPU" means "no particles."
The one documented WebGL-backend limitation (three.js issue #27642) is
cross-index storage reads (`buffer.element(someOtherIndex)`, not your own
`instanceIndex`) — same-index read/write is fine.

**`setupWebGLXRFallback` (`three/addons/webxr/WebGLXRFallback.js`) only
patches `renderer.xr.setSession`.** It does nothing until someone actually
enters an XR session — it cannot explain a broken desktop page load, so
don't waste time suspecting it for non-XR bugs.

**`requiredLimits: { maxStorageBuffersInVertexStage: N }`** — reading N
storage buffers in the *vertex* stage (a `Sprite`/`InstancedMesh` material
that does `buffer.element(instanceIndex)` for N different `instancedArray`
buffers inside `positionNode`/`colorNode`/`scaleNode`) can exceed some
devices' default WebGPU limits. Pass it in the `WebGPURenderer`
constructor options, and have a fallback path that retries without it if
`renderer.init()` throws — some adapters reject the request entirely
rather than clamping.

## Deployment

**First-ever GitHub Pages deploy on a brand-new org/repo can fail once or
twice with a content-free "Deployment failed, try again later" error**,
even with correct config (branch policy, org Pages permissions, Actions
permissions all fine). Checked via `gh api repos/{org}/{repo}/pages`,
`.../pages/builds/latest`, `.../environments/github-pages/deployment-branch-policies`
— all clean both times it failed. A plain retry (empty commit + push, or
`gh run rerun --failed`) resolved it within a couple of attempts. Don't
spiral into reconfiguring things that already check out; just retry with
a bit of spacing.

**Diff the live URL against your local tested file before debugging
"why doesn't it work"** — `curl -s <live-url> | diff - local-file` rules
out a stale-deploy or wrong-branch problem in one step, before you start
suspecting the code.

## Debugging workflow

**Static verification (syntax check + grep every import against the
actual downloaded bundle) catches typos and missing exports, but not
runtime shadowing bugs, node-graph misuse, or logic errors.** The `floor`
bug above passed `node --check` and a full import-existence audit cleanly
— it's a valid JS/TSL API call, just resolving to the wrong binding at
runtime. There is no substitute for the actual browser console. Ask for
it early rather than continuing to theorize from source code.

**Real console error > any number of plausible-sounding static theories.**
Multiple hypotheses were pursued here (WebGL backend compute support,
`positionNode` space semantics, XR fallback wiring) that were all
internally consistent, all individually verified against source, and all
completely wrong. The actual bug was one shadowed identifier. Get the
stack trace before spending more than a few minutes on static reasoning.

## Accessibility

**Bloom + additive blending + per-particle brightness tied to velocity
can produce whole-screen luminance changes fast enough to be a WCAG 2.3.1
concern (flashing >3×/second, ≥25% relative luminance change over a
meaningful screen area)** — worth checking against per WCAG's own
definition: https://www.w3.org/WAI/WCAG22/Understanding/three-flashes-or-below-threshold.html
Mitigations used here: a user-facing "calm ↔ vivid" slider (default
biased calm, more so under `prefers-reduced-motion`) that scales bloom
strength, per-particle speed→brightness gain, and bioluminescence
decay/gain simultaneously — not just an opt-in toggle defaulting to the
original intensity.
