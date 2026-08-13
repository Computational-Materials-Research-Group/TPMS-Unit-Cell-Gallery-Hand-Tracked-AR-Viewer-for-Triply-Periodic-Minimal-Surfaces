# TPMS Unit Cell Gallery: Hand-Tracked AR Viewer for Triply Periodic Minimal Surfaces

<p align="center">
  <img src="https://img.shields.io/badge/WebXR-Immersive%20AR-blue?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Three.js-r160-black?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Meta%20Quest-Passthrough%20AR-orange?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Hand%20Tracking-Pinch%20to%20Select-purple?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Marching%20Cubes-7%20Isosurfaces-teal?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Web%20Speech%20API-Narrated-yellow?style=for-the-badge"/>
</p>

<p align="center">
  Put on a <b>Meta Quest</b> and the seven canonical <b>triply periodic minimal surface (TPMS)</b>
  unit cells, Primitive, Neovius, Diamond, FRD, IWP, Fischer&ndash;Koch, and Gyroid, appear as a
  row of small lattices floating in your real room. Pinch one with your bare hand to select and
  resize it, walk around the row to see it from any angle, and listen as each lattice is narrated
  aloud with a short, research-grounded note on what it's actually used for. The same file also
  works as an orbiting desktop gallery you tap and drag. In either mode every surface is a live
  <b>marching-cubes isosurface</b> built in the browser from its implicit trigonometric equation
  and shaded with a diverging colormap, all from a single self-contained HTML file with nothing
  else to install.
</p>

<img width="1665" height="867" alt="tpms ar" src="https://github.com/user-attachments/assets/80215338-78e9-4c72-aeaf-0d33bfd9e49f" />

---

## Read This First: What This Viewer Gives You, and What It Does Not

This is a **visualization and interaction layer** built on `three.js`'s `MarchingCubes` object,
not a CAD or simulation tool. Every surface is computed live in the browser from a closed-form
implicit equation — nothing is loaded from an external mesh or dataset.

| This viewer gives you | This viewer does NOT give you |
|---|---|
| Real-time marching-cubes rendering of 7 implicit TPMS surfaces, generated in-browser from their trigonometric formulas | An exportable, manufacturing-ready mesh (STL/STEP) — this is a scalar-field isosurface for looking at, not printing |
| Desktop orbit view with tap-to-select and drag-to-resize per lattice | Any physical simulation — no FEA, no fluid flow, no stress analysis of the shapes shown |
| Passthrough AR on Meta Quest with bare-hand pinch-to-select and pinch-and-move-to-resize | Controller support, or hand tracking on hardware without a WebXR hand-tracking implementation |
| Spoken, research-grounded application notes per surface (Web Speech API) | Inline per-claim citations — the application text is paraphrased from published TPMS lattice research, not footnoted line-by-line |
| One self-contained HTML file, no build step, no bundled data files | A hosted, persistent deployment — you still need to serve the file yourself |

---

## Why Marching Cubes at a Fixed Resolution and Triangle Budget

Each surface is evaluated on a `RESOLUTION = 42` voxel grid and triangulated by `MarchingCubes`
with a fixed polygon budget:

```
RESOLUTION = 42
MAX_POLY   = 32000   // measured worst case at this resolution is ~24,769 tris (Fischer–Koch)
```

`MarchingCubes` pre-allocates its triangle buffer up front, so the budget has to be picked in
advance rather than grown dynamically. 32,000 leaves real headroom above the densest of the seven
surfaces (Fischer&ndash;Koch) without over-allocating for the simpler ones. If `recolor()` ever
finds more triangles than the budget allows, it logs a console warning rather than silently
dropping geometry unnoticed.

## Why a Coolwarm Colormap Driven by Surface Normal

Rather than a flat material color, each lattice is shaded per-vertex using a five-stop,
Moreland-style diverging colormap (`COOLWARM`), sampled by the vertical component of that
vertex's surface normal:

```
t = normal.y * 0.5 + 0.5   // remap [-1, 1] to [0, 1]
color = coolwarm(t)         // blue -> near-white -> red
```

This gives every lattice the same visual language — undersides read cool, upward-facing surfaces
read warm — which is what the legend bar in the bottom-right of the stage is showing.

## Keeping Every Surface the Same Apparent Size, on Desktop and in AR

Desktop and AR use two entirely different physical scales for the same seven lattices:

```
DESKTOP_RADIUS = 0.5   // world units, tuned for an orbiting camera
AR_RADIUS      = 0.075 // meters — a lattice you can hold in one hand
```

`applyDesktopLayout()` and `applyArLayout()` re-scale the whole row (and its spacing) when
entering or leaving an XR session. A lattice's on-screen size is actually the product of two
independent scale factors that are only combined at render time: a brief "selected" pulse
(1.0&times; to 1.16&times;) and the user's own manual resize (`userScale`, clamped 0.4&times; to
2.0&times;) — so selecting a lattice never overwrites a resize you already made to it, and vice
versa.

## Reference Space and Row Placement in AR

WebXR offers more than one coordinate-origin convention. Requesting `local-floor` as a session
*feature* is not the same as telling three.js to actually use it — this file calls
`renderer.xr.setReferenceSpaceType('local-floor')` explicitly before building the AR button, so a
y-coordinate reliably means "height above the floor" rather than "height above wherever your eyes
happened to be when the session started."

The row itself isn't placed at a fixed absolute coordinate. `placeRowInFrontOfUser()` reads your
live head pose two frames into the session and anchors the row relative to it:

```
AR_DISTANCE        = 0.6   // meters in front of you
AR_VERTICAL_OFFSET = -0.12 // meters below eye line — easier to look at than dead ahead
```

A fixed world coordinate only looks right if the headset grants a true floor-calibrated space,
which isn't guaranteed across devices — anchoring to the camera's own pose at session start is
the more robust choice.

## Orienting the Row to Match Your Point of View

Placing the row in front of you isn't enough on its own — it also has to face you. After
`placeRowInFrontOfUser()` sets the row's position, it calls:

```js
rowGroup.lookAt(_camPos.x, rowGroup.position.y, _camPos.z)
```

which rotates the whole row of seven lattices to point back at your head position (holding the
y-coordinate fixed, so the row doesn't tilt). That guarantees your point of view the moment AR
starts is always a straight-on read of the row — you'll never spawn looking at it edge-on or from
behind, regardless of which way you happened to be facing when you tapped **Enter AR**.

This placement and orientation only run once per session (gated by the `arPlaced` flag, two
frames after `sessionstart`). After that the row behaves like a real object anchored in your
room — it does **not** keep re-facing you as you walk around it, so you can circle it and view it
from the side or back, the same way you would a physical model on a table.

## Audio and Speech

Narration runs on the browser's built-in **Web Speech API** (`speechSynthesis`), not a bundled
audio file, so no extra assets are downloaded:

```js
function speak(text){
  if (!audioEnabled || !('speechSynthesis' in window)) return;
  const clean = text.replace(/\bTPMS\b/g, 'T P M S'); // spelled out so it isn't mispronounced as one word
  window.speechSynthesis.cancel();                    // stop whatever was playing before queuing the next line
  const u = new SpeechSynthesisUtterance(clean);
  u.rate = 0.98;
  window.speechSynthesis.speak(u);
}
```

Two things trigger it: a one-time welcome line, fired on the very first `pointerdown` on desktop
or on `sessionstart` in AR (browsers require a real user gesture before they'll allow speech
audio to play at all); and a per-lattice line — name plus application — spoken every time
`selectIndex()` is called with a *different* index than the current selection, so re-tapping the
same lattice doesn't repeat itself. The audio button simply flips `audioEnabled` and calls
`speechSynthesis.cancel()` immediately, cutting off whatever is mid-sentence. In AR, since the 2D
detail panel is hidden (`body.ar-mode #panel{ opacity:0 }`), the floating `detailSprite` text is
the visual counterpart to whatever is being spoken — the two are meant to be read and heard
together.

**If narration stays silent on a Quest**, the most likely causes, roughly in order of how often
they trip people up:
- The audio toggle got tapped off before the AR session started (check it reads "Audio on").
- The very first interaction happened *before* a valid user gesture — reloading the page and
  tapping the audio-off/on button once, or tapping a lattice, before entering AR usually clears
  this.
- The Quest Browser has no TTS voice loaded yet. `speechSynthesis.getVoices()` returns an empty
  array until the OS's voice pack finishes loading — this can lag on a first boot after a system
  update.
- Rapid selection changes right after `sessionstart` can cut the welcome line off almost
  instantly, since `speak()` cancels the in-flight utterance every time it's called — this can
  look like "no audio" when it's actually being interrupted a fraction of a second in.

---

## Interaction Model

| Context | Gesture / Action | Effect |
|---|---|---|
| Desktop | Tap a lattice | Selects it, updates the detail panel below, and speaks its name + application |
| Desktop | Drag away from / toward a selected lattice | Resizes that lattice only (clamped 0.4&times;&ndash;2.0&times;); camera orbit is disabled for the duration of the drag |
| Desktop | Drag on empty space | Orbits the camera (`OrbitControls`; auto-rotate resumes once idle) |
| Desktop | "Reset sizes" button | Returns every lattice's manual scale to 1&times; |
| Desktop | Audio on/off button | Toggles Web Speech API narration |
| AR (Quest) | Pinch (thumb tip within 2.8 cm of index tip) near a lattice | Selects it and begins a resize grab |
| AR (Quest) | Keep pinching, move your hand away from or closer to the lattice | Resizes it (clamped 0.4&times;&ndash;2.0&times;) |
| AR (Quest) | Two hands pinching independently | Each hand can grab and resize a *different* lattice at the same time |

---

## Repository Structure

```
tpms-unit-cell-gallery/
|
|-- index.html   # The entire viewer — markup, styles, and the three.js module script, in one file
```

Everything (fonts, `three.js`, `OrbitControls`, `MarchingCubes`, and `ARButton`) is pulled from
CDNs via an import map at the top of the `<script type="module">` block, so there's nothing to
build and nothing else to bundle or host alongside it.

---

## How to Run

### Requirements

- A modern desktop browser with WebGL2, or
- A Meta Quest headset with Hand Tracking enabled (Settings &gt; Movement Tracking)
- Any static file host reachable over HTTPS from the headset (or `localhost` via `adb reverse` for local iteration)

### Step 1 — Open it on desktop

No setup required — open the HTML file directly, or host it and load the URL. The gallery loads,
auto-rotates gently, and the first tap on any lattice also unlocks narration (most browsers
require a user gesture before `speechSynthesis` will play audio).

### Step 2 — Host it for the headset

WebXR's `immersive-ar` mode requires a secure context, so the file needs to be served over HTTPS
(or reached via `localhost`/`adb reverse` for testing). A plain static-site host works fine —
there are no server-side dependencies.

### Step 3 — Open it on the Meta Quest

Navigate to the hosted URL in a WebXR- and hand-tracking-capable browser on the headset, and tap
**Enter AR** (rendered by `ARButton` at the bottom of the page). Accept the passthrough camera
and hand-tracking permission prompts.

### Step 4 — Find the row

The seven lattices appear about 0.6 m in front of wherever you were facing when the session
started, roughly at chest height. A short banner at the bottom of the page reminds you to enable
Hand Tracking if it isn't already on.

### Step 5 — Interact

Pinch to select and resize a lattice; use both hands to grab two different ones at once. The
name and application text for whichever lattice is selected appear as floating 3D text above the
row (the flat 2D detail panel is disabled while in AR, since DOM overlays render unreliably in
headsets).

---

## Common Errors and Fixes

| Symptom | Cause | Fix |
|---|---|---|
| Blank/black stage on desktop | WebGL2 unavailable, or `three.js` failed to load from the CDN | Check the browser console; confirm network access to the `unpkg.com` import-map URLs |
| "Enter AR" button never appears | Browser/device doesn't support `immersive-ar`, or the page isn't served over HTTPS/`localhost` | Serve over HTTPS; use a WebXR-capable browser on a Quest headset |
| Pinch doesn't select anything in AR | Your pinch point is farther than `AR_SELECT_RADIUS` (0.11 m) from any lattice's center, or Hand Tracking is off | Move your pinch closer to the lattice; enable Hand Tracking in Settings &gt; Movement Tracking |
| Lattice row appears far above or below where it should | Reference space defaulted to `local` (eye-level origin) instead of `local-floor` | Confirm `renderer.xr.setReferenceSpaceType('local-floor')` runs before `ARButton.createButton` — already set in this file |
| Row appears behind you or at an odd angle on entering AR | The row is anchored to your head pose at session start, not a fixed world coordinate, with a short placement delay | Re-enter the AR session while facing the direction you want the row to appear |
| No narration plays | Audio toggle is off, or the browser is blocking autoplay before any tap | Check the audio button is on; the first `pointerdown` on the page is what unlocks speech |
| A lattice looks jagged or has missing chunks | The fixed triangle budget (`MAX_POLY = 32000`) was exceeded for that field | Raise `MAX_POLY` (a console warning fires when the budget is hit) or lower `RESOLUTION` |
| Labels drift off-screen or vanish while orbiting | Expected — labels are re-projected from 3D to 2D every frame and hidden once their anchor leaves the camera's clip range | No action needed, this is intentional off-screen culling |

---

## Extending the Gallery

| Extension | What to change |
|---|---|
| Add another TPMS surface | Add an entry to the `STRUCTURES` array with `name`, `alt`, `speak`, `application`, and its implicit `f(...)` formula |
| Change mesh detail | `RESOLUTION` (raise `MAX_POLY` to match — check the console warning after any increase) |
| Change spacing or how many lattices fit on screen | `DESKTOP_SPACING` / `AR_SPACING` |
| Change the colormap | The five control points in the `COOLWARM` array |
| Change resize limits | The `THREE.MathUtils.clamp(..., 0.4, 2.0)` calls in the desktop drag handler and the AR pinch-grab handler |
| Change how close a pinch must be to count, or to select a lattice | `PINCH_THRESHOLD` / `AR_SELECT_RADIUS` |
| Change where the row appears relative to the user in AR | `AR_DISTANCE` / `AR_VERTICAL_OFFSET` |
| Change the narrated text | The `application` string on each entry in `STRUCTURES` |

---

## Citation

If you use or adapt this viewer, a citation template is provided below:

```bibtex
@software{mishra_2026_tpmsgallery,
  author    = {Mishra, Akshansh},
  title     = {TPMS Unit Cell Gallery: Hand-Tracked AR Viewer for Triply Periodic Minimal Surfaces},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.21911200},
  url       = {https://doi.org/10.5281/zenodo.21911200}
}
```

> Mishra, A. (2026). *TPMS Unit Cell Gallery: Hand-Tracked AR Viewer for Triply Periodic Minimal Surfaces* [Computer software]. Zenodo. https://doi.org/10.5281/zenodo.21911200

---

## Author

**Akshansh Mishra**
GitHub: [akshansh11](https://github.com/akshansh11)

---

## License

<p align="center">
  <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
    <img alt="Creative Commons Licence" style="border-width:0; margin: 12px 0;"
      src="https://licensebuttons.net/l/by-nc/4.0/88x31.png"/>
  </a>
  <br/>
  <a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">
    Creative Commons Attribution-NonCommercial 4.0 International License
  </a>
</p>

This work is licensed under a **Creative Commons Attribution-NonCommercial 4.0 International License**.

You are free to:
- **Share** — copy and redistribute the material in any medium or format
- **Adapt** — remix, transform, and build upon the material

Under the following terms:
- **Attribution** — You must give appropriate credit to Akshansh Mishra ([akshansh11](https://github.com/akshansh11)) and link back to this repository
- **NonCommercial** — You may not use the material for commercial purposes

Copyright (c) 2026 Akshansh Mishra. All rights reserved for commercial use.
