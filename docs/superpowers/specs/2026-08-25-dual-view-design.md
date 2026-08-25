# Dual View — Design

Implements GitHub issue #9.

> Allow user to select two scans to be viewed side by side. Include a lock view
> button which allows the user to zoom/pan/rotate both views simultaneously when
> set. Can zoom/pan/rotate each individually when unset. Also include a reset
> button which brings both views back into sync with one another using the
> original coordinates of each scan.

## Goal

Add an explicit **Dual View** mode that splits the viewport into a left and a
right pane, each showing one user-chosen loaded model with its own independent
camera. A **Lock views** toggle makes orbit / zoom / pan drive both panes at
once; a **Reset both views** button returns each pane to that scan's own
load-time framing, which also puts the two panes back in sync with each other.

Dual view is a *mode*, not a change to the default. With the mode off, the
application behaves exactly as it does today — same single camera, same single
render call, same tools.

## Non-goals

- More than two panes.
- Making any of the existing single-view tools (section cut, flat base, 3-point
  base, margin marking, 3D measure, undercut, deviation, thickness) work per
  pane. They are explicitly disabled while dual view is active — see
  "Tools during dual view".
- Any registration/alignment step between the two scans. The two panes are two
  independent cameras onto two independently placed models; dual view never
  moves geometry.
- Vertical (over/under) split, or a resizable split ratio. The split is a fixed
  50/50 vertical split.

## Rendering: one renderer with two scissored viewports

**Chosen: one `WebGLRenderer`, one canvas, two `setViewport` + `setScissor`
passes per frame.**

The alternative — a second `WebGLRenderer` with its own canvas — was rejected
on blast radius, not on capability. Today's file has a single
`renderer.domElement` (`const canvas`) and *every* pointer, wheel, touch, and
cursor-class interaction in the app is bound to it:
`canvas.addEventListener('mousedown'|'wheel'|'touchstart'|'touchmove'|'touchend'|'contextmenu'|'mousemove')`,
`canvas.classList.add('grabbing'|'move-plane'|'crosshair'|'point-hover')`,
`clientToNDC`/`clientToLocal` off `canvas.getBoundingClientRect()`, plus
`new THREE.TransformControls(camera, canvas)` for the undercut gizmo. A second
canvas means either duplicating all of that or re-binding it through a
dispatcher — a large, risky edit to code that currently works.

Scissoring costs one extra `renderer.render()` call per frame and keeps
everything else — one WebGL context, one shader/program cache, one
`localClippingEnabled` setting, one pixel ratio, one set of listeners, one
`resize()` — exactly as it is. The trade-off accepted is that both panes share
one depth/color buffer and one canvas size, so the split ratio is fixed and
per-pane post-processing is impossible. Neither is wanted.

Frame shape while dual view is active:

```
renderer.setScissorTest(true);
for each pane (left, right):
    show only that pane's model group, hide the other model groups
    updateLightsToCamera(pane)          // light rig follows that pane's camera
    renderer.setViewport(x, 0, halfW, h);
    renderer.setScissor(x, 0, halfW, h);
    renderer.render(scene, pane.camera);
restore every model group's previous .visible
renderer.setScissorTest(false);
```

**Per-pane model isolation is done with `group.visible`, not `THREE.Layers`.**
Visibility is toggled inside the frame and restored at the end of the same
frame, so no per-object state survives a frame boundary and nothing has to be
un-done when leaving the mode. It also gets the analysis overlays right for
free: the undercut, deviation, and thickness overlay meshes are children of
`model.group`, so they follow their model into the correct pane.

The lights and the ground grid are never touched, so they render in both panes.

`CSS2DRenderer` cannot be scissored — it positions real DOM elements over the
whole canvas. Its only consumer is the 3D measurement label, and 3D measure is
disabled in dual view, so the CSS2D layer is simply hidden (`display:none`) for
the duration of the mode and its per-frame `render()` call is skipped.

## View state: `viewA` aliases today's globals, `viewB` is new

The camera is not an OrbitControls instance. It is three module-scope bindings
plus a derive function:

```js
const target = new THREE.Vector3(0, 0, 0);   // orbit target
let radius = 12;                             // orbit distance
let orbitQuat = new THREE.Quaternion();      // orientation
function updateCamera(){ /* writes camera.position / camera.up from the above */ }
```

`updateCamera` is called from ~15 places and `target` / `radius` / `orbitQuat`
are read from ~25 more. Rewriting all of those to go through an indirection is
the single largest regression risk in this feature, so it is not done.

Instead a lightweight **view descriptor** is introduced:

```js
const viewA = {
  camera,                                   // the existing camera object
  target,                                   // the existing Vector3 — same reference
  orbitQuat,                                // the existing Quaternion — same reference
  get radius(){ return radius; },
  set radius(v){ radius = v; },
  home: { target: new THREE.Vector3(), radius: 12, quat: new THREE.Quaternion() },
  homeModelRadius: 0,   // model radius the home framing was derived from
  modelId: null
};
```

This is a *true alias*, not a copy:

- `target` and `orbitQuat` are only ever mutated in place in the existing code
  (`.copy`, `.addScaledVector`, `.premultiply`) and never reassigned, verified
  by grep. Sharing the object reference therefore aliases them perfectly.
- `radius` is a primitive, so it is aliased by an accessor pair that reads and
  writes the `let radius` binding itself.

`viewB` is a plain object of the same shape with its own
`THREE.PerspectiveCamera`, `Vector3`, `Quaternion`, and a plain `radius` field.
It is created lazily the first time dual view is entered.

The camera math is factored into four generic functions that take a view:

| function | replaces inline math in |
| --- | --- |
| `applyView(v)` | body of `updateCamera()` |
| `orbitView(v, dx, dy)` | body of `orbitDrag()`, which it replaces outright |
| `panView(v, dx, dy)` | the `if(panning)` branch of the `mousemove` handler |
| `zoomView(v, factor)` | the `wheel` handler and the pinch branch of `touchmove` |

`updateCamera()` keeps its name and signature and simply delegates to
`applyView(viewA)`, so all ~15 of its call sites are unchanged.
`updateLightsToCamera(v)` gains an optional view argument defaulting to `viewA`.
`orbitDrag()` had only two call sites (mouse drag and one-finger touch drag), so
it is replaced by `orbitView` rather than kept as a wrapper.

Zoom clamping goes through `viewModelRadius(v)`, which returns the global
`modelRadius` (the combined bounds of every loaded model, exactly as today)
unless dual view is on, in which case it returns that pane's own scan radius.

**Why this cannot regress single view.** With dual view off, `activeView()`
returns `viewA` and `allViews()` returns `[viewA]`. Every generic function
called with `viewA` reads and writes exactly the same `target`, `radius`,
`orbitQuat`, and `camera` objects the old inline code did, with the same
arithmetic. `viewB` is never constructed and its camera never renders.

### Which pane is "active"

`dualActivePane` (0 or 1) is set on `mousedown` / `wheel` / `touchstart` over
the canvas from the pointer's x position relative to the canvas midpoint. It
persists between interactions, so sidebar controls that act on "the view"
(hex view snapping, FOV, Reset view, auto-rotate) have a well-defined target.
It defaults to 0 (left). `activeView()` returns `viewA` whenever dual view is
off, so nothing outside the mode consults it.

The active pane is outlined in the viewport so it is never ambiguous which pane
a keyboard/sidebar action will hit.

## Selecting the two scans

A new **Dual View** sidebar section, placed directly after **Loaded Models**:

- `Dual view` switch — enables/disables the mode.
- `Left pane` / `Right pane` `<select class="side-select">`, populated from
  `models` exactly like `refreshDevModelSelects()` does.
- `🔒 Lock views` switch.
- `Reset both views` button.
- A hint line naming the tools that are unavailable in the mode.

**Fewer than two models loaded.** The `Dual view` switch is disabled (visually
dimmed, click is a no-op) and the hint reads *"Load at least two models to use
dual view."* The selects are disabled too. This is re-evaluated on every
`renderModelList()` (which already runs on load and on removal).

**Defaults.** On entering the mode, left = `models[0]`, right = `models[1]`.
Re-entering restores the previous pair when both are still loaded.

**Same model in both panes is allowed.** It is a legitimate way to compare two
angles of one scan, and with lock on it degenerates harmlessly to two identical
views.

**A model in a pane is removed.** If fewer than two models remain, dual view
exits automatically. Otherwise the orphaned pane falls back to the first model
not already shown in the other pane.

**A model is loaded while dual view is active.** It appears in both selects but
does not change the current assignment.

## Lock view: what actually gets synced

Not the camera matrix, and not `camera.position`.

The two scans do not share a world origin. Models loaded normally are
auto-centered and dropped at `(0, size.y/2, 0)`; models loaded with *Keep
original alignment* on sit wherever their file puts them, which can be hundreds
of millimetres from the origin. Copying pane A's camera matrix into pane B
would aim pane B's camera at pane A's model's location — pane B would show
empty space. The two scans can also differ in size, so copying an absolute
orbit `radius` would make one of them fill the pane while the other is a speck.

Lock therefore syncs the view **relative to each pane's own home framing**.
After any locked interaction on the active pane, `syncLockedViews(src)` writes
the follower pane as:

```
dst.orbitQuat = src.orbitQuat                                   // verbatim
k             = dst.home.radius / src.home.radius                // size ratio
dst.radius    = src.radius * k                                   // same zoom factor
dst.target    = dst.home.target + (src.target - src.home.target) * k
```

In words: **same orientation, same zoom factor, same pan offset measured in
units of the scan's own size.** Orientation is origin- and scale-independent so
it is copied outright. Zoom and pan are expressed as a delta from the pane's
home framing and scaled by the ratio of home framings, which is what makes two
differently-sized scans at different origins stay visually matched.

Because the shared orientation means both cameras use the same right/up basis,
scaling the world-space pan delta is equivalent to scaling the screen-space pan
— no extra basis change is needed.

With lock **off**, each pane's orbit/pan/zoom writes only its own view
descriptor; the panes drift apart freely, exactly as the issue asks.

Turning lock **on** immediately runs `syncLockedViews(activeView())`, so the
follower snaps to match the pane the user was last driving rather than the two
staying mismatched until the next drag.

## Reset: what "the original coordinates of each scan" means

Precisely: **the framing this application would compute for that scan if it were
the only model loaded.** That is `frameCameraOnModels()`'s existing math
restricted to one model:

```
bounds  = that model's own world-space bounding box
target  = centre of bounds
mr      = max(bounds.x, bounds.y, bounds.z) * 0.6
radius  = mr * 1.3255 / tan(fov/2)
quat    = buildLookQuaternion(bottom)      // the app's default view direction
```

This is computed by a new `computeModelFraming(model)`, which derives the bounds
with `new THREE.Box3().setFromObject(model.group)` so it is correct for models
loaded with *Keep original alignment* (whose geometry is not centred on their
group origin) as well as normally-loaded ones.

It is **computed on demand** — when a pane is assigned a model and on every
reset — rather than snapshotted into a field at load time. The value it returns
at load time *is* the load-time framing, since it reads the scan's placement as
loaded; computing it on demand additionally means that if the user later
re-bases the scan with Pick flat base / Pick 3-point base, reset returns to the
scan as it now sits instead of to a stale snapshot that would frame it
off-centre. Re-basing is an explicit, user-initiated change to the scan's
placement, so treating it as the new "original" is what matches what the user
sees. `Box3.setFromObject` reads cached geometry bounding boxes, so this is
cheap enough not to warrant a cache.

The result is stored on the view descriptor as `view.home` (`target`, `radius`,
`quat`) plus `view.homeModelRadius`. Nothing in single view reads either.

**Reset both views** copies each pane's `home` into its live `target` / `radius`
/ `orbitQuat` and re-derives both cameras. Because both panes are then at zero
delta from their own home, they are by construction back in sync with one
another — satisfying the second half of the issue's requirement — regardless of
whether lock is currently on.

Entering dual view, and any change of a pane's model, performs the same
operation for the affected pane.

The existing **Reset view** button (Display section) is left alone in single
view. In dual view it resets the *active pane only* (and propagates through lock
if lock is on), which keeps it meaningful without duplicating the dual-view
reset.

## Tools during dual view

**Every existing single-view tool is disabled while dual view is active.** This
is a deliberate scope decision, not an oversight.

All of them raycast with `raycaster.setFromCamera(clientToNDC(x, y), camera)`
against `models.map(m => m.mesh)` — full-canvas NDC, the single global camera,
and *all* models rather than the pane's model. Several of them (section cut,
undercut, deviation, thickness) additionally operate over the whole `models`
array and put their result into shared scene-level state. Making them pane-aware
means threading a view through the picking path of eight separate tools and
re-scoping their model sets, which is a far larger change than the feature
itself and a much larger regression surface.

Disabled: **Draw section line, Pick flat base, Pick 3-point base, Mark margin,
Measure distance, Show undercuts, Compute deviation, Compute thickness**, plus
the section-cut sub-controls (Clip solid, Flip clip side, Clear section) and the
2D cross-section panel's measure controls.

**The UI shows it.** Each containing sidebar section (`secSectionCut`,
`secReorient`, `secPrepMargin`, `secMeasure`, `secUndercut`, `secMapping`) gets a
`.dual-locked` class whose rule is `opacity: 0.4; pointer-events: none` on the
section *body*. One class both dims the whole group and makes every control
inside it — buttons, switches, sliders, selects — genuinely inert, and removing
the class on exit restores all of them exactly as they were, with no per-control
`disabled` state to snapshot and replay. The section headers stay clickable so
sections can still be collapsed.

The Dual View section carries the explanatory line *"Section cut, reorient,
margin, measure, undercut and mapping tools are unavailable while dual view is
on."*

The tools' keyboard shortcuts are suppressed too: the global `keydown` handler
returns immediately while dual view is active, so Escape and the margin
Ctrl+Z / Ctrl+Y cannot mutate state the user can't see.

On **entering** the mode, non-destructively:

1. Disarm any armed tool (`setSectionArmed(false)`, `setBaseArmed(false)`,
   `setThreePointArmed(false)`, `setMarginArmed(false)`,
   `setMeasure3DArmed(false)`), and force `setUndercutActive(false)` — which
   restores `mesh.visible = true` on every model, undoing undercut mode's own
   visibility override so it cannot fight the per-pane visibility swap.
2. Add `.dual-locked` to each affected sidebar section.
3. Hide every scene-level tool visual — the cut plane mesh, the section outline,
   the 3-point markers, margin markers/lines/preview, the 3D measurement
   markers/lines, the undercut gizmo. Rather than enumerating those variables
   (fragile), this walks `scene.children` and hides every direct child that is
   not a light, not the grid, and not a model group, recording each object's
   prior `.visible` for restoration.
4. Suspend section clipping: `updateClipping()` gains a single guard so that
   while dual view is active it applies an empty plane array. `clipEnabled` and
   `cutPlane` are left untouched in memory.
5. Hide the CSS2D DOM layer and the 2D cross-section panel.

No user data is destroyed. A section cut, a marked margin, or a computed
deviation map survives the round trip through dual view intact.

## Leaving dual view

`setDualView(false)` reverses each step above in order and leaves **no residual
state**:

- restores `.visible` on every scene child and every model group from the
  recorded snapshots;
- removes the `.dual-locked` classes, which restores every tool control;
- calls `updateClipping()` again, which now sees dual view off and re-applies
  the real clip plane;
- un-hides the CSS2D layer and re-shows the cross-section panel if it was open;
- `renderer.setScissorTest(false)` and a forced full-viewport `resize()`
  (`lastResizeW/H` are invalidated so the cached-size early-out cannot skip it),
  which restores `camera.aspect`, the renderer size, and the draw-overlay size;
- hides the pane divider / labels;
- `viewB` is retained in memory but nothing reads it, and its camera is not in
  the scene graph, so it has no effect.

`viewA`'s `target` / `radius` / `orbitQuat` are the same live objects the whole
time. Whatever the user did to the left pane simply *is* the single view's
camera when they come back out — no snapshot/restore needed, and no risk of a
stale copy.

## Sidebar controls that touch "the camera"

Three existing controls read or write the camera state and need a defined
meaning in the mode. Each is generalised with `allViews()` / `activeView()`,
both of which collapse to `viewA` when dual view is off:

| control | dual-view behaviour |
| --- | --- |
| FOV slider | applied to **both** cameras, each with its own distance compensation. Each view's `home.radius` gets the same compensation, since it is FOV-derived too — otherwise a locked pair would jump on the next drag, lock being defined on `radius / home.radius` ratios |
| Reset view (Display) | resets the **active** pane; propagates via lock |
| Hex view selector | snaps the **active** pane; propagates via lock |
| Auto-rotate | spins **both** panes (each around its own target), so a locked pair stays locked and an unlocked pair both animate |
| Axis indicator (bottom-left) | drawn from the **active** pane's orientation |

## UI additions in the viewport

- `#dualDivider` — a 1px vertical rule down the canvas midpoint.
- `#dualPaneL` / `#dualPaneR` — absolutely-positioned overlays covering each
  half, carrying the model name in the top-left corner and a 1px accent outline
  when that pane is active. `pointer-events: none` so they never intercept a
  drag.

All three are `display:none` outside the mode.

## Error handling

- Entering dual view with fewer than two models is not reachable through the UI
  (the switch is disabled); `setDualView(true)` also guards on
  `models.length >= 2` and no-ops otherwise.
- A pane whose model has gone missing renders nothing rather than throwing —
  the render pass skips a pane with a null model.
- `computeModelFraming` clamps the derived radius the same way
  `frameCameraOnModels` does, so a degenerate zero-size model cannot produce
  `Infinity`/`NaN`.

## Testing plan

Manual, in-browser (static HTML, no test harness). Fixtures from
`test-fixtures/`.

1. Load one model — Dual view switch is disabled, hint reads "Load at least two
   models".
2. Load a second — switch enables, both selects populate.
3. Enter dual view — split appears, left/right show the two chosen models,
   labels correct, tool sections dim and their buttons disable.
4. Change the right pane's select to the other model — right pane re-frames.
5. Lock **off**: orbit / pan / shift-drag / wheel-zoom in the left pane; the
   right pane must not move. Repeat in the right pane.
6. Lock **on**: the same gestures in either pane move both, staying visually
   matched.
7. Drag the two panes far out of alignment with lock off, then **Reset both
   views** — both return to their own scan's framing and are matched again.
8. Hex view snap and FOV slider while in dual view.
9. Leave dual view — single view returns with the tool buttons re-enabled and
   the camera where the left pane left it.
10. **Regression:** with dual view off, exercise Draw section line + Clip solid,
    Pick flat base, Measure distance, and Mark margin, and confirm each behaves
    as before.
11. **Regression:** create a section cut, enter dual view (cut hidden, clipping
    suspended), leave dual view — the cut and its clipping come back intact.
12. Console must be free of errors throughout.
