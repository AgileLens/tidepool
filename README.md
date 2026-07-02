# Tidepool v2 — a WebGPU compute study

262,144 particles + 1,024 fish + bioluminescence, all simulated on the GPU
with three.js r185 WebGPURenderer + TSL compute. Zero CPU physics.

## Systems
- **Tide particles** — curl-noise currents, 4 modes (Drift/Kelp/School/Pulse)
- **Bioluminescence** — per-particle excitement buffer; stirring, bursts, and
  turbulence make the water glow, then it fades back to dark
- **Fish school** — 2nd compute system; instanced darts oriented along velocity
  with speed-paced tail wiggle; they school, flee your cursor, scatter on Pulse
- **Caustic sand floor** — procedural ridged-noise caustics
- **Post** — RenderPipeline bloom + vignette (flat screens only)
- **WebXR** — stand inside the pool; WebGL fallback for browsers without
  WebXR+WebGPU (`setupWebGLXRFallback`)

## Gotchas applied (from the Unbuilt Theater / easteregg session)
- Billboard quads, never gl_Points (no ~64px Apple GPU size cap)
- All animation dt-based (90 Hz headset safe)
- Angular size clamp `min(size, camDist*0.14)` — fill-rate/white-out guard
  when standing inside the cloud
- No bloom in XR + `uBoost` compensation; left-pinch brightness ladder
  (×0.5/0.75/1/1.5/2.25); right pinch cycles modes
- Meters-scale via `uWorldScale` uniform (0.09 → ~4.7 m pool), not group scale
- `requiredLimits: { maxStorageBuffersInVertexStage: 3 }` — 3 storage buffers
  read in the vertex stage exceeds some default device limits

## URL params
`?n=65536|262144|524288` · `?xrscale=0.09` · `?xrboost=1.5`

## Deploy
Static, no build step. Hosted on **GitHub Pages** from this repo
(`main` branch, root). Live at https://agilelens.github.io/tidepool/
To point a subdomain at it later: CNAME `tidepool.agilelens.com` ->
`agilelens.github.io` and set the custom domain in repo Settings > Pages.
