# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
go run .              # serve on http://localhost:8080 (frontend from the embedded copy)
go run . -dev         # serve the frontend from ./web on disk — use while editing web/
go build              # single self-contained binary
go vet ./... && gofmt -l .
```

There are no Go tests. The automated checks are all browser harnesses that
render a pass/fail list on load — start the server and open:

| Page | Covers |
|---|---|
| `/liftoff-test.html` | Liftoff exporter conversion maths |
| `/glb-test.html` | `.glb` export: scene pruning, billboard conversion, GLB header |
| `/vr-test.html` | VR rig maths: spawn placement, walking, snap-turn pivot |

All three share `web/liftoff-test-fixtures.json`, so none needs a designer
session or a saved track. They can be run headless:

```bash
chromium --headless --disable-gpu --enable-unsafe-swiftshader \
  --virtual-time-budget=30000 --dump-dom http://localhost:8080/glb-test.html
```

The page title becomes `PASS n/n` or `FAIL k/n`, so that line alone is the
result. `--enable-unsafe-swiftshader` is required: the harnesses build a real
`SceneManager`, which needs a WebGL context.

`vr-test.html` covers the rig geometry only. Session lifecycle, reference
spaces, controller axis mapping and restoring the desktop camera on exit all
need a real headset.

Useful flags: `-addr`, `-gates`, `-tracks`, `-banners`, `-admin-password`
(or `$TRACK_ADMIN_PASSWORD`).

## Constraints that shape the code

- **Go backend is stdlib-only** (`go.mod` has no requires) and the **frontend has
  no build step and no dependencies** — plain ES modules plus a vendored,
  minified Three.js resolved through an importmap in `index.html`. Both are
  deliberate; don't introduce a bundler, npm, or a Go module.
- **`web/vendor/` is Three.js r165 plus four official addons**, copied verbatim
  with one exception: `GLTFExporter.js` imports `TextureUtils.js` from `./`
  instead of `./../utils/`, because `vendor/` is flat. Re-apply that on upgrade,
  and keep addon versions pinned to the core build's revision.
- **`//go:embed web`** bakes the frontend into the binary. Frontend edits are
  invisible without a rebuild unless the server is running with `-dev`.
- **`gates/`, `banners/` and the tracks dir are read from disk at runtime**, not
  embedded — they must ship alongside the binary.
- **Cache headers are load-bearing.** API responses are `no-store` (server) plus
  `cache: 'no-store'` (client) because a heuristically-cached GET after a save
  looks like the save was lost; static files are `no-cache` so a rebuilt binary's
  frontend is picked up immediately.

## Architecture

`main.go` wires everything: flags, `LoadGates`, `NewTrackStore`, `NewBanners`,
route table, and the gzip wrapper. `server/` is four small files — gate registry,
track CRUD + passwords, banner listing/serving, gzip middleware.

**The server does not understand tracks.** A track's `data` is stored as
`json.RawMessage` and round-tripped verbatim. Every part of the document schema
(arena, gates, measurements) lives in the frontend, and all schema migration
happens in `Editor.loadFrom` / `Editor.toJSON` in `web/js/editor.js` — e.g. old
tracks carry `reversed: true` instead of `dir`, and pre-prop tracks have no
`prop` field. Add compatibility fallbacks there, never server-side.

**Gate types are data, not code.** `gates/*.json` describes a gate
parametrically (shape, innerSize, tubeWidth, depth, defaultHeight, color); the
frontend builds the geometry. `POST /api/gates` writes a new JSON file and
registers it without a restart, so the set of `typeId`s is open and minted at
runtime — never key logic on `typeId`. `shape` + `innerSize` is the stable pair.

**Auth model** (`server/tracks.go`): the `X-Track-Password` header carries either
a track's own password or the admin master password. `authorize` gates
edit/delete; `canView` gates listing and reading of `private` (unreleased)
tracks. Passwords are salted and iterated SHA-256 (`iters$salt$hash`), compared
with `subtle.ConstantTimeCompare`, and the hash never reaches a client — only the
`protected` / `private` booleans do. On `PUT`, `name`/`data` are optional and
`private` is a pointer so a release/unlist needn't resend the whole track.

### Frontend module roles (`web/js/`)

`app.js` is the composition root and owns the track document, the toolbar, the
keyboard map, and the shared `state.mode` (`select` | `place` | `measure`) that
`Editor`, `MeasureTool` and `UI` all read. Ownership is strict: `scene.js` owns
the renderer/camera/arena, `editor.js` owns placed gates and the transform
gizmo, `measure.js` owns measurement chains, `ui.js` owns DOM/dialogs,
`gates.js` owns all geometry, `vr.js` owns the XR session, `export3d.js` owns
the `.glb` export. `editor.prePick` is the hook that lets measurement markers
win a click over gate selection.

**Scene conventions.** The arena's corner sits at the world origin and extends
into +X/+Z so gate coordinates read as "meters from the corner"; everything is
meters. A gate is a `THREE.Group` whose origin is the bottom-center of the frame,
with named children `frame`, `pickFill` (invisible click target across the
opening), `arrow`, `legs`, `numberLabel`, `startLine`, and `userData` flags read
across modules (`isGate`, `def`, `height`, `tableHeight`, and on the frame:
`frameHeight`, `isTable`, `directional`, `multiDirectional`, `noStand`,
`pickSize`). Numbering, overlap-colouring and the start line are recomputed
wholesale by `Editor._renumber` / `_updateStartMarker`; props (`prop: true`) are
skipped by the flight sequence entirely.

### VR (`web/js/vr.js`)

Scene units are already real metres with the floor at y = 0, which is exactly
what a `local-floor` reference space expects, so the arena maps 1:1 onto the
room with no conversion. `scene.js` also already drove rendering through
`renderer.setAnimationLoop`, the callback form WebXR requires.

The camera is **parented to a rig group**, never moved directly: three.js
overwrites `camera.position`/`quaternion` with the headset pose every frame and
composes it with the parent's world matrix, so the rig is the only place a
player position can live. That is also why `enter()` snapshots the desktop
camera pose — by exit it has been overwritten with head poses. `_snapTurn`
pivots about the head rather than the rig origin; turning about the origin
throws the player sideways whenever they have walked away from it.

### 3D model export (`web/js/export3d.js`)

Runs on clones, never the live scene, and is GLB-only by design (see the README
for why not OBJ). Three constraints drive the whole module:

- **glTF has no billboard**, so `THREE.Sprite` is silently dropped by the
  exporter — that would lose every gate number, metre marker and distance
  label. They are re-issued as flat textured quads aimed at the arena centre,
  built from the *live* objects' world matrices (a detached clone has never
  been through `updateMatrixWorld`).
- **Measurements are rebuilt from their points, not cloned.** `Object3D.copy`
  deep-copies `userData` through `JSON.stringify`, and a measurement marker's
  `userData` references the chain that owns it — a cycle, so cloning throws.
  Anything else that grows a self-referential `userData` will hit this too.
- **Invisible pick helpers must be dropped** — the fill across every gate's
  opening and the oversized hit spheres on measurement points would otherwise
  export as phantom geometry.

The viewport's ➤ and 👁 toggles are deliberately *not* the authority over what
lands in the file; the export dialog's checkboxes are.

### Adding a gate shape

Touch three registries in `web/js/gates.js` — `shapeBuilders` (geometry),
`thumbnailDrawers` (palette icon), `shapeFieldMeta` (New-gate form labels,
`propByDefault`) — and add the shape to `SHAPES` in
`web/js/liftoff/convert.js` so the exporter classifies it. Unmapped shapes fall
back to a 0.75 m square gate and are reported in the export dialog's `unknown`
list.

### Liftoff exporter (`web/js/liftoff/`)

Self-contained, browser-only, nothing server-side; runs on the in-memory
document so unsaved edits export. `convert.js` (designer → geometry) →
`xml.js` (.track/.race writers) → `zip.js`, with `catalog.js` holding the real
Liftoff prop table and the whole-track gate-matching pass, and `uuid5.js`
producing deterministic GUIDs (re-export replaces rather than duplicates).

Two things are easy to break: this is a **port of a Python implementation kept
in a separate project that asserts byte-identical output**, so any change to the
emitted XML is a cross-project contract change; and the geometry constants
(`MIRROR_X`/`YAW_SIGN` handedness, the `<scale>`-is-ignored-on-flags rule that
forces floor tiling, prop dimensions mined from Workshop tracks) were settled
empirically by flying the results. The reasoning is in comments beside each —
read it before adjusting a number.
