# Wassa Bear Holdings — Website

A single‑page brand site for **Wassa Bear Holdings LLC** (a Wyoming holding company).
The page is an interactive **3D scene**: a procedurally‑modeled polar bear standing in a
snowy, moonlit landscape. The bear's head follows the visitor's cursor, snow falls
continuously, and the only on‑screen text is the company name plus a standard legal footer.

There is **no tracking, no cookies, and no analytics** — see `privacy-policy.html`.

---

## Table of Contents

1. [Summary](#summary)
2. [Live preview / running locally](#running-locally)
3. [File overview](#file-overview)
4. [Tech stack & dependencies](#tech-stack--dependencies)
5. [How the 3D scene works (`index.html`)](#how-the-3d-scene-works)
6. [Function reference](#function-reference)
7. [The polar bear model](#the-polar-bear-model)
8. [Privacy policy page](#privacy-policy-page)
9. [Favicon](#favicon)
10. [Browser support & performance](#browser-support--performance)
11. [Customization guide](#customization-guide)

---

## Summary

| Aspect | Detail |
| --- | --- |
| Type | Static website (no backend, no build step) |
| Pages | `index.html` (3D scene), `privacy-policy.html` (legal) |
| 3D engine | [Three.js](https://threejs.org) `r160`, loaded from a CDN via import map |
| Rendering | WebGL, PBR materials, soft shadows, image‑based lighting, ACES tone mapping |
| Interactivity | Bear head + eyes track mouse / touch; animated snowfall; idle breathing |
| Assets | 100% procedural — **no external 3D model, image, or texture files** |
| Fonts | `Cinzel` (Google Fonts) for the banner; Georgia fallback |
| Privacy | No cookies, no trackers, no analytics |
| Dependencies | Three.js + Google Fonts only (both via CDN). No npm install required. |

---

## Running locally

The site is plain static files, but it **must be served over HTTP** (not opened with a
`file://` URL) because `index.html` uses ES‑module `import` and an `<script type="importmap">`,
which browsers block on the `file://` protocol.

From the project folder:

```bash
# Python 3 (no install needed)
python3 -m http.server 8080
```

Then open **http://localhost:8080** in any modern browser.

Any other static server works too, e.g.:

```bash
npx serve .          # Node
php -S localhost:8080  # PHP
```


---

## File overview

```
wassabearholdings-website/
├── index.html          # Home page — the entire Three.js 3D scene (HTML + CSS + JS in one file)
├── privacy-policy.html # Standard no‑cookie privacy policy (self‑contained HTML + CSS)
├── favicon.svg         # Vector "W" badge icon (preferred by modern browsers)
├── favicon.ico         # 32×32 raster fallback "W" icon (older browsers / tabs)
├── grg-lockup-line.svg # Ganges River Group lockup for the footer design credit (CSS mask)
└── README.md           # This document
```

---

## Tech stack & dependencies

### Runtime dependencies (loaded from CDN at page load)

| Dependency | Version | Source | Purpose |
| --- | --- | --- | --- |
| **three** | `0.160.0` | `https://unpkg.com/three@0.160.0/build/three.module.js` | Core WebGL 3D engine |
| **three/addons** | `0.160.0` | `https://unpkg.com/three@0.160.0/examples/jsm/…` | Add‑on modules (see below) |
| **Cinzel** font | — | `https://fonts.googleapis.com/css2?family=Cinzel` | Banner typeface |

Three.js is wired up with a native browser **import map** (no bundler):

```html
<script type="importmap">
{
  "imports": {
    "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
    "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
  }
}
</script>
```

### Three.js modules actually used

| Import | Used for |
| --- | --- |
| `* as THREE` from `three` | Renderer, scene, camera, geometries, materials, lights, math |
| `RoomEnvironment` from `three/addons/environments/RoomEnvironment.js` | A neutral indoor light probe baked into an environment map for realistic PBR reflections/lighting |

Specific Three.js classes in use: `WebGLRenderer`, `Scene`, `FogExp2`, `CanvasTexture`,
`PMREMGenerator`, `PerspectiveCamera`, `HemisphereLight`, `DirectionalLight`,
`MeshPhysicalMaterial`, `MeshStandardMaterial`, `MeshBasicMaterial`, `SphereGeometry`,
`CapsuleGeometry`, `ConeGeometry`, `PlaneGeometry`, `Group`, `Mesh`, `InstancedMesh`,
`BufferGeometry`, `BufferAttribute`, `Points`, `PointsMaterial`, `Object3D`, `Vector2`,
`Vector3`, `Quaternion`, `Color`, `Clock`, `MathUtils`.

### Build / tooling dependencies

**None.** There is no `package.json`, no compiler, and no install step. The browser loads
everything directly.

---

## How the 3D scene works

All scene code lives in the `<script type="module">` block at the bottom of `index.html`.
It executes top‑to‑bottom in these stages:

1. **Renderer** — A `WebGLRenderer` (antialiased) is created and appended into
   `#scene-container`. Pixel ratio is capped at 2 for sharpness without over‑rendering on
   high‑DPI screens. Soft shadow maps (`PCFSoftShadowMap`) and **ACES Filmic** tone mapping
   are enabled for a filmic, realistic look.

2. **Scene + environment** —
   - `FogExp2` gives an exponential blue haze for depth.
   - A **gradient sky** is drawn into an offscreen `<canvas>`, wrapped in a `CanvasTexture`,
     and set as `scene.background`.
   - `PMREMGenerator` bakes a `RoomEnvironment` into an environment map
     (`scene.environment`) so the physically‑based materials get soft, realistic ambient
     reflections and lighting.

3. **Camera** — A `PerspectiveCamera` viewing the bear from a front‑3/4 angle. The
   `placeCamera()` helper positions it relative to a fixed look‑at target and **pulls back on
   narrow / portrait screens** so the whole bear always fits.

4. **Lighting** — Three lights: a `HemisphereLight` (sky/ground fill), a shadow‑casting
   `DirectionalLight` "moon" key light, and a cool `DirectionalLight` rim light for edge
   definition. The environment map provides the remaining ambient illumination.

5. **Materials** — Shared materials are created once and reused (see table in
   [The polar bear model](#the-polar-bear-model)).

6. **The bear** — Built procedurally from primitives grouped under a `bear` group, with the
   head in its own `headPivot` sub‑group so it can rotate independently. (Details below.)

7. **Fur** — `addFur()` scatters thousands of tiny cone "strands" over the body via
   `InstancedMesh` (one draw call per group). See [The polar bear model](#the-polar-bear-model).

8. **Ground** — A `PlaneGeometry` whose vertices are displaced into gentle snow mounds, plus
   a ring of scattered "snow drift" spheres.

9. **Snowfall** — `makeFlakeTexture()` paints a soft radial flake into a canvas texture; 3,500
   points (`Points` + `PointsMaterial`) fall and sway, recycling to the top when they reach
   the ground.

10. **Interaction** — `mousemove` / `touchmove` update a normalized `mouse` vector.

11. **Resize** — `onResize()` keeps the renderer size, camera aspect, and camera distance
    correct on every window resize / device rotation.

12. **Render loop** — `animate()` runs via `requestAnimationFrame`, updating snow, idle
    breathing, and head/body tracking each frame.

---

## Function reference

All functions are defined inside the `index.html` module script.

| Function | Signature | Responsibility |
| --- | --- | --- |
| `placeCamera()` | `() ⇒ void` | Positions the camera from `CAM_TARGET` + `CAM_OFFSET`, scaling the distance up on portrait/narrow aspect ratios so the bear stays fully in frame. |
| `part()` | `(group, mat, x, y, z, sx, sy, sz, shadow=true) ⇒ Mesh` | Core model builder. Clones the shared unit `SPHERE` geometry, positions/scales it into an ellipsoid, toggles shadow casting/receiving, adds it to `group`, and returns the mesh. Used for nearly every body part. |
| `leg()` | `(x, z, frontPitch) ⇒ Group` | Builds one leg: a `CapsuleGeometry` upper limb, a flattened paw sphere, and three dark toe pads, grouped together. |
| `earAt()` | `(sx) ⇒ void` | Builds one ear (outer fur sphere + inner‑ear colored sphere) at horizontal sign `sx` (`-1` left, `+1` right). |
| `eyeAt()` | `(sx) ⇒ void` | Builds one eye: a dark socket recess, a glossy eyeball, and two catch‑light highlights, positioned proud of the skull. |
| `addFur()` | `(parent, ellipsoids) ⇒ InstancedMesh` | Scatters fur strands over a list of ellipsoid regions. For each strand it samples a point on the ellipsoid surface, computes the outward normal (with a downward sweep + jitter), orients a cone along it via a quaternion, randomizes length/thickness/tint, and writes the per‑instance matrix + color. Returns one `InstancedMesh`. |
| `N()` | `(v) ⇒ number` | Density helper — multiplies a strand count by `furDense` (1 on desktop, 0.5 on screens < 700px) and rounds, so mobile gets lighter fur. |
| `makeFlakeTexture()` | `() ⇒ CanvasTexture` | Draws a soft white radial‑gradient snowflake into a 64×64 canvas and returns it as a texture for the snow `PointsMaterial`. |
| `onResize()` | `() ⇒ void` | Updates renderer size + pixel ratio, camera aspect + projection matrix, and re‑runs `placeCamera()`. Bound to the window `resize` event. |
| `animate()` | `() ⇒ void` | The per‑frame loop: advances/​recycles snow, applies idle breathing scale, smooths and clamps the head's yaw/pitch toward the cursor, nudges the whole body's rotation, renders, and re‑schedules itself with `requestAnimationFrame`. |

### Key module‑level values

| Name | Meaning |
| --- | --- |
| `CAM_TARGET`, `CAM_OFFSET` | Camera look‑at point and base offset vector. |
| `SPHERE` | A shared unit `SphereGeometry` reused by `part()` (memory‑efficient). |
| `furMat`, `darkMat`, `eyeMat`, `earInMat`, `snowMat` | Shared materials. |
| `bear` | Root `Group` holding the whole animal. |
| `headPivot` | Sub‑group containing skull, snout, nose, ears, eyes, and head fur; rotates to track the cursor. |
| `mouse`, `headTarget`, `headCur` | Normalized cursor vector, the head's target yaw/pitch, and its smoothed current value. |
| `furDense`, `N` | Mobile‑aware fur density factor and helper. |

---

## The polar bear model

The bear is **fully procedural** — assembled from intersecting primitive meshes, so there is
no model file to download or break.

### Anatomy (built with `part()`, `leg()`, `earAt()`, `eyeAt()`)

- **Torso / belly / shoulder hump / rump / tail** — overlapping ellipsoids forming the long
  body and the characteristic polar‑bear shoulder hump.
- **Four legs** — `CapsuleGeometry` limbs with flattened paws and dark toe pads.
- **Head pivot group** — neck, skull, muzzle bridge, elongated snout (the "Roman nose"
  profile), and a dark nose.
- **Ears** — outer fur sphere + inner‑ear tint.
- **Eyes** — recessed dark socket, glossy eyeball, and two catch‑lights for a "wet" look.

### Materials

| Material | Type | Notes |
| --- | --- | --- |
| `furMat` | `MeshPhysicalMaterial` | White, high roughness, with **sheen** for a soft fur‑like rim highlight. |
| `darkMat` | `MeshStandardMaterial` | Nose & toe pads. |
| `eyeMat` | `MeshStandardMaterial` | Low‑roughness glossy black eyeballs. |
| `earInMat` | `MeshStandardMaterial` | Muted inner‑ear color. |
| `snowMat` | `MeshStandardMaterial` | Ground and drifts. |

### Fur layer

`addFur()` places ~**8,000 cone strands** across the body and head using a single
`InstancedMesh` per group (so the entire coat is just two draw calls). Each strand:

- sits on the surface of one of the body's ellipsoid regions,
- points along the outward surface normal with a **downward sweep + random jitter**,
- has randomized length, thickness, and a subtle cool‑white tint.

Head fur is parented to `headPivot`, so it moves when the bear looks around. Strand count
**auto‑halves on screens under 700px** wide for performance.

### Animation & interaction (in `animate()`)

- **Head tracking** — the cursor maps to a target yaw (`±0.7 rad`) and pitch (`±0.38 rad`),
  smoothed (`lerp`) and clamped so motion is natural and never over‑rotates.
- **Body turn** — the whole `bear` group eases slightly toward the cursor's horizontal
  position.
- **Idle breathing** — a subtle vertical scale + bob driven by a sine wave.
- **Snowfall** — each flake falls at its own speed, sways horizontally, and respawns at the
  top after passing the ground.

---

## Privacy policy page

`privacy-policy.html` is a standalone, self‑contained page (its own embedded styles) linked
from the footer. It is a **standard, no‑cookie privacy policy** stating that the site sets no
cookies, runs no analytics, and embeds no third‑party trackers, with the usual sections
(information collected, children's privacy, security, your rights, governing law = Wyoming,
contact). It shares the dark arctic theme and links back to the home page.

---

## Favicon

Two formats are provided and linked from both pages:

- **`favicon.svg`** — a crisp vector "W" on an icy‑blue rounded badge (preferred by modern
  browsers).
- **`favicon.ico`** — a 32×32 raster fallback (4× supersampled) for browsers/tabs that don't
  use the SVG.

---

## Browser support & performance

- **Requires WebGL** plus support for **ES modules** and **import maps** — i.e. any current
  version of Chrome, Edge, Firefox, or Safari. (Import maps shipped in all evergreen browsers
  by 2023.)
- Pixel ratio is capped at **2×** and fur density **halves below 700px** to keep frame rates
  high on phones and high‑DPI laptops.
- Touch is supported: dragging on a touchscreen moves the bear's gaze.
- First load fetches Three.js (~1 MB) and the Cinzel font from CDNs; everything else is
  generated in‑browser, so there are no image/model downloads.

---

## Customization guide

Common tweaks, all in the `index.html` module script:

| To change… | Edit… |
| --- | --- |
| Company name | The `<h1>` inside `#banner` (and the footer text). |
| Banner font | The `Cinzel` Google Fonts `<link>` and the `font-family` rules. |
| Camera angle / distance | `CAM_TARGET` and `CAM_OFFSET`, or the `k` factor in `placeCamera()`. |
| Bear proportions | The `part()` / `leg()` calls (positions and scales). |
| Fur length / density | The `furStrandGeo` cone size, the `len`/`scale` ranges in `addFur()`, and the strand counts passed via `N(...)`. |
| Snow amount / speed | `SNOW_N` and the `snowVel` range. |
| Sky / fog colors | The gradient‑canvas stops and `FogExp2` color. |
| Lighting mood | The `HemisphereLight` / `DirectionalLight` colors and intensities, and `toneMappingExposure`. |

---

© 2026 Wassa Bear Holdings LLC — All Rights Reserved · Registered in Wyoming, USA
