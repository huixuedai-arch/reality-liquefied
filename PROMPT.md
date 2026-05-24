# Reality Liquefied — Build Prompt

Hand this entire document to Claude / ChatGPT / Cursor as a single prompt.
It is the distilled spec for the "现实被液化" real-time hand-gesture distortion
effect: webcam background that reads as a fogged pane of water, hand gestures
push and ripple through the surface like fingertips touching real liquid.

---

## What I want

A **single self-contained `index.html`** I can open on iPhone Safari/Chrome
and have it just work — no build step, no `node_modules`, no bundler.

### Visual goal (most important — do not skip this)

1. **At rest** the camera image looks like it's seen through a pane of glass
   that's covered in tiny round water droplets — *bathroom-mirror after a
   shower*. Image is desaturated, soft, milky-fogged, and you can clearly see
   a dense field of small circular beads across the whole frame.
2. **When fingers approach the camera**, the water gets pushed aside locally:
   each fingertip emits **expanding concentric ripples** that propagate
   outward, interfere with each other, and slowly decay. Where the ripple
   crests are, the fog is wiped away and you can see the sharp, lensed,
   magnified image of the room behind. Outside the ripples, fog remains.
3. **A held still finger keeps emitting** small ripples (subtle pulsing —
   like a fingertip trembling on a water surface). It does **not** sit
   on a static "magic circle" that follows it around.
4. **Edge highlights are coloured by whatever is behind the lens** — a
   finger produces a pink rim because skin is being magnified at the ring
   front; the same ring over a white wall produces a white highlight.
   **Never hardcode pink** into the highlight colour.

### Anti-goals (we tried these, they look wrong)

- Plain box-blur fog → produces an X-shaped artifact, reads as "paper", not water.
- Continuous positive force at fingertip → produces a sustained mound that
  follows the finger like a magic circle, never oscillates, no rings.
- Hardcoded pink/blue/coral specular → looks fake because the highlight
  colour stops matching what's actually behind it.
- Chromatic dispersion (RGB split) → makes everything look like a 90s VFX
  filter and obscures the lens effect. Leave it off.

---

## Architecture

- WebGL **1** (not WebGL2 — maximum mobile compatibility).
- Hand tracking via MediaPipe Hands 0.4 from jsdelivr CDN, 10 fingertips
  (5 per hand × 2 hands), landmarks `[4, 8, 12, 16, 20]`.
- Two fragment shaders:
  - **physics**: 2-D damped wave equation with impulse-driven force injection.
  - **render**: composites the fogged base layer and the sharp lensed layer.
- FBO ping-pong with **3 buffers** at **half resolution** for the physics step.
- Final render at full canvas resolution.
- Half-float (`OES_texture_half_float`) preferred for the physics FBO,
  with `OES_texture_float` and `UNSIGNED_BYTE` fallbacks.
- `checkFramebufferStatus` after FBO creation; on failure, visibly report
  the error code and texture type on the page.
- Shader compile failures must show the error message + numbered source on
  the page — never fail silently.

---

## Physics shader (half-resolution)

### Wave update
9-tap isotropic stencil (axis-aligned neighbours 0.2 each, diagonals 0.05 each).
Verlet update: `val = avg * 2.0 - prev`.

### Localization
To suppress excessive propagation while still allowing visible ring spread,
blend the neighbour-average toward the centre value:
```glsl
const float LOCALIZE = 0.05;
float coupled = mix(avg, c, LOCALIZE);
float val = coupled * 2.0 - prev;
```

### Damping
Multiplicative per frame (from `uDamping` uniform, default **0.976** —
this is what makes ripples die within ~1 s and prevents the wave field
from saturating the screen).
Plus a small amplitude-dependent term that gently flattens microscopic
noise: `1.0 - smoothstep(0.0, 0.08, abs(val)) * 0.006`.

### Ambient noise
Tiny hash-based jitter (`* 0.0006`) so the surface is never numerically dead.

### Finger interaction (the critical part)

Each frame, for each of the 10 fingertips, compute four contributions:

1. **First-touch impulse** — single-frame kick the moment the finger appears.
   This is what makes the wave equation actually oscillate (push down →
   release → rebound → propagate) and is what produces real expanding rings.
   ```glsl
   float touchImpulse = step(0.5, uFingers[i].z - uPrevFingers[i].z);
   ```

2. **Velocity force** — proportional to finger speed (pixels/frame).
   Sliding fingers lay down a trail of taps.
   ```glsl
   float velPx = length(fa - fb) * uResolution.y;
   float velForce = clamp(velPx * 0.025, 0.0, 1.0);
   ```

3. **Per-finger sinusoidal pulse** — small oscillating push/pull at ~4 Hz,
   each finger phase-offset. This is what keeps a held still finger
   *emitting* ripples instead of sitting on a frozen mound.
   ```glsl
   float pulse = sin(uTime * 8.0 + float(i) * 1.7) * 0.35;
   ```

4. **No static base force** — a continuous positive push creates the
   magic-circle artifact. Omit it entirely.

Combine via a capsule SDF between current and previous finger positions
(so fast slides produce continuous strokes, not gappy dots), in
aspect-corrected UV space (multiply `x` by `aspect`). Use
`baseRadius = 0.038 * uRadius`.

```glsl
float strength = velForce + touchImpulse * 0.55 + pulse;
float dist = capsuleSDF(aspectUv, fa, fb, baseRadius);
float falloff = smoothstep(baseRadius, -baseRadius * 0.5, dist);
force += strength * falloff;
```

After summing all fingers, **allow negative net force** (the pull half of
the pulse is essential — clamping to `[0, +]` would kill the oscillation):
```glsl
force = clamp(force, -3.0, 3.0);
val += force * 0.05;
val = clamp(val, -1.2, 1.2);
```

---

## Render shader (full resolution)

### Two layers, mixed by clarity

The whole point of the look is `mix(hazyBase, sharpLensed, clarity)`.

### Layer 1: bathroom-mirror condensation (`hazyBase`)

Use a **Voronoi cell field** for the droplet pattern, not box blur.

```glsl
// hash + Voronoi droplet helpers
float hashR(vec2 p) { return fract(sin(dot(p, vec2(127.1, 311.7))) * 43758.5453); }
vec2  hash2R(vec2 p) { return fract(sin(vec2(
  dot(p, vec2(127.1, 311.7)),
  dot(p, vec2(269.5, 183.3))
)) * 43758.5453); }
vec3 droplet(vec2 p) {
  vec2 i = floor(p); vec2 f = fract(p);
  vec2 best = vec2(0.0); float bestD = 9.0;
  for (int y = -1; y <= 1; y++) for (int x = -1; x <= 1; x++) {
    vec2 g = vec2(float(x), float(y));
    vec2 r = g + hash2R(i + g) - f;
    float d = dot(r, r);
    if (d < bestD) { bestD = d; best = r; }
  }
  return vec3(-best, sqrt(bestD)); // .xy outward from centre, .z distance
}
```

For each pixel:
- Sample the droplet field at `vUv * vec2(aspect, 1.0) * 180.0`
  (dense — about 24k beads on a 1080p frame).
- **Refract toward the cell centre** (convex lens magnification — `-cellDir`,
  not `+cellDir`!), strongest near the centre, tapering at the rim.
- Sample the video at the refracted UV with a small two-ring box blur
  (inner radius ~0.004, outer ~0.0075, weighted) for the inter-bead haze.
- **Heavy desaturation (0.65) + milky frost tint (0.55)** so the resting
  image is unambiguously fogged, not slightly hazy.
- Add a **thin circular rim** at fixed radius from each bead centre
  (`smoothstep(0.30, 0.40, cellDist) * (1 - smoothstep(0.40, 0.48, cellDist))`)
  — **must be radius-based, not cell-boundary-based**, otherwise the rim
  traces polygonal Voronoi edges and you get star/diamond artifacts.
- Tiny dark dot at the very centre for "bead nucleus" feel.

### Layer 2: sharp lensed image (`sharpLensed`)

Standard physics-driven refraction with a Sobel gradient of the wave field:

```glsl
vec2 grad     = sobel(uPhysics) * 0.125;
float gradLen = length(grad);
float lap     = (tc + bc + ml + mr) * 0.25 - mc;

vec2 refractOffset = grad * 0.55 * uIntensity * vec2(1.0/aspect, 1.0);
vec2 lensOffset    = normalize(grad) * lap * 0.18 * uIntensity * vec2(1.0/aspect, 1.0);
vec2 totalOffset   = refractOffset + lensOffset;

vec3 sharpLensed = texture2D(uVideo, toVideoUv(vUv + totalOffset)).rgb;
sharpLensed *= 1.0 + clamp(mc * 2.2, -0.38, 0.45);
```

**All highlights are colour-neutral gains, not coloured additions:**
- **Fresnel reflection**: mix with sample at `-1.6 * totalOffset`, no tint.
- **Ridge highlight**: `pow(gradLen * 7.5, 2.0)` applied as a multiplier
  on a video sample further along the offset direction — this is what
  produces the "pink rim" over skin, without any hardcoded colour.
- **Specular (twin-lobe, 180 + 28 exponents)**: brighten the underlying
  video sample, don't add a coloured constant.
- **Caustics** at high-curvature crests: amplify the existing colour
  (`sharpLensed *= mask * 0.55`), don't tint.

**No chromatic dispersion. No fixed blue/coral tints anywhere.**

### Clarity mix

```glsl
float clarity = clamp(gradLen * 40.0 + abs(mc) * 12.0, 0.0, 1.0);
vec3  liquid  = mix(hazyBase, sharpLensed, clarity);
```

Moderate sensitivity — strong enough that ripple crests visibly wipe the
fog along their leading edge, low enough that the rest of the screen stays
fogged even while ripples are alive.

### CRYSTAL mode (alternate look — quantize the gradient)

`facetGrad = floor(grad * 18.0) / 18.0`, use that as the refraction
offset, draw an icy edge highlight where adjacent facets differ. Switch
between LIQUID and CRYSTAL with `step(0.5, uMode)` — **never use an `int`
uniform for branching on mobile WebGL1**.

---

## MediaPipe Hands integration

- Load from `https://cdn.jsdelivr.net/npm/@mediapipe/hands@0.4/` via
  `<script src=>` plus `locateFile` for the wasm/data files.
- `maxNumHands: 2`, `modelComplexity: 0`, confidence 0.5.
- **iOS critical**: `hands.initialize()` can hang silently for tens of
  seconds (or forever) on iOS Safari / Chrome while the ~10 MB wasm
  downloads. **Race it against a 15 s timeout** and let the boot continue
  without hand tracking on failure — the pointer/touch fallback below
  must still work.
- Send frames every other RAF tick (`(frameCount & 1) === 0`); skip if a
  previous send hasn't returned.
- Convert each fingertip:
  ```js
  let sx = (lm.x - 0.5) / videoScale[0] + 0.5;
  let sy = (lm.y - 0.5) / videoScale[1] + 0.5;
  if (currentFacingMode === 'user') sx = 1.0 - sx; // mirror front cam
  fingers[i*3 + 0] = sx;
  fingers[i*3 + 1] = 1.0 - sy;  // flip Y to GL UV origin
  fingers[i*3 + 2] = 1.0;       // active flag
  ```
- `videoScale` derives from a `cover`-style fit of the video aspect into
  the canvas aspect, so the finger pixels line up with what the user sees.

---

## Pointer / touch fallback

Even when MediaPipe loads, also wire pointerdown/move/up on the canvas
into `fingers[0]` so phones without MediaPipe (or with it still loading)
remain interactive. Use `touch-action: none` on the canvas and
`canvas.setPointerCapture(e.pointerId)` for stable drag.

---

## UI

- Bottom-centre glass panel (CSS `backdrop-filter: blur(14px) saturate(160%)`,
  dark semi-transparent background).
- Two mode buttons: `LIQUID` / `CRYSTAL`, exclusive active state.
- Three sliders with live numeric readout:
  - `INTENSITY`  0.1 – 3.0    default **1.80**
  - `RADIUS`     0.2 – 4.0    default **0.70**
  - `DECAY`      0.85 – 0.995 default **0.976**
- Top-right glass buttons: `SWITCH CAM` (toggles user/environment) and
  `HIDE UI` (hides everything; a `SHOW UI` button reappears in the corner).
- Loading screen showing the current boot stage
  (`Requesting camera…` → `Loading hand tracking…`).
- Error overlay (`<pre id="error" style="display:none">`) that's only
  shown on real failure — never default-visible (this *will* cover the
  scene at 0.85 black and break clicks if you leave `display: none` off).

---

## Compatibility / boot order

- `<meta viewport>` with `maximum-scale=1.0, user-scalable=no` to stop
  iOS double-tap zoom.
- `precision highp float;` at the top of every fragment shader.
- All uniform arrays are `Float32Array` of the exact declared length
  (10 fingers = 30 floats).
- Resize listener rebuilds FBOs only when canvas size actually changes
  (don't allocate every frame).
- **Boot order matters on iOS**:
  1. Call `getUserMedia` → start camera.
  2. Start the RAF render loop immediately once camera is up.
  3. Hide the loading screen (don't wait for MediaPipe).
  4. Kick off `initHands()` in the background, awaited by no-one.

  If you instead `await initHands()` before hiding the loading screen,
  iOS users on slow networks will see "Initializing…" forever.

---

## Performance notes

- Physics FBO at `floor(canvas.width / 2), floor(canvas.height / 2)`.
- Update video texture each frame with `texImage2D(..., video)` and
  `UNPACK_FLIP_Y_WEBGL = true`.
- `prevFingers.set(fingers)` AFTER each physics step (not before — the
  shader needs both arrays from the same frame to compute velocity AND
  the touch-impulse step).

---

## Done means

Open `index.html` over `https://` (or `http://localhost`) on:
- Desktop Chrome / Safari: see a fogged scene, wave your hand, get pink
  expanding ripples that follow your fingertips.
- iPhone Safari / Chrome: same, no infinite loading screen even on
  slow networks. Touch works even before MediaPipe finishes loading.

If any of the **anti-goals** above are visible (magic circle, X-shaped
fog, hardcoded pink), the implementation is wrong — re-read the
"physics shader" and "render shader" sections.
