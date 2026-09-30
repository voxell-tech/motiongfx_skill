# Typst visuals with velyst

Setup that has worked for projects that combine Bevy scenes with Typst graphics.
Keep Bevy, vello, Typst and bevy_vello versions aligned; mixed versions fail in
confusing ways. Import through `velyst::bevy_vello::prelude::*` to avoid skew.

## Cameras

- Two cameras layered by `order`, both clearing to `Color::NONE`: a `Camera3d`
  for physical objects and a `Camera2d` with `VelloView` for Typst overlays.
- bevy_vello only draws to a 2D camera that carries `VelloView`, so overlays
  must be 2D.
- If you add `Hdr` (and `Bloom` for a glow) to the 3D camera, add `Hdr` to the
  2D camera too, or the 3D layer can turn black after the first frame.
- Clearing to `Color::NONE` keeps an alpha channel for compositing later
  (ProRes 4444). Never add an opaque full-frame element in that case.
- Separate overlapping 2D elements with small `z` offsets in `Transform`.
- Scenes that need a floor spawn their own ground plane, so 2D-only scenes are
  not affected.

## Typst functions

- Register each function with `register_typst_func::<T>()`.
- The `typst_func!` name string must equal the Typst `#let` name. Positional
  order matters, and named args become `Option<T>` (omitted when `None`).
- Give each func a `pub type VFooFunc = VelystFunc<FooFunc>;` alias so
  motiongfx can address its fields with `path!(<VFooFunc>::data::field)`.
- Put logic in Typst and expose scalar knobs to animate: `reveal`, `progress`,
  `collapse`, `tint`. Let Typst derive layout from those, such as cell width
  from a total width.
- Lengths: convert in Typst (`x * 1pt`). A raw float where a length is expected
  is a Typst error.
- Frame counts and similar must be integers (`i64`).
- Do not name a func `label`; it shadows Typst's built-in `label()`.
- Use one palette module for colors on both sides (a Typst file and a matching
  Rust module). Avoid raw hex in scenes.

## Two ways to animate

1. Mutate the func's data fields. Each change recompiles the Typst, which is
   fine for small scenes.
2. Add `VelystKanva`. The frame is converted once and paths can then be
   modified without a recompile. Use this for trace, fade and stagger effects.

## KanvaAnim and KanvaGroup

`VelystKanva` + `KanvaGroup` (which paths) + `KanvaAnim` (how they reveal),
then tween `KanvaAnim::t` from 0 to 1.

- `KanvaGroup::all()`: every path.
- `KanvaGroup::inner("name")`: paths inside one labeled group.
- `KanvaGroup::wrap("start", "end")`: paths between two labeled markers.
- `.with_target(entity)`: drive a different entity's kanva.
- `KanvaAnim` constructors take `path_window`, the per-path fraction of `t`
  (smaller means more staggered): `trace`, `fade`, `trace_fade(pw, ratio)`,
  `scale_fade`, `fade_up`.

To reveal regions of one visual on their own cues:

1. Mark regions in the Typst function with `<label>` markers.
2. Spawn one driver entity per region, each with its own `KanvaAnim` and a
   `KanvaGroup::inner(..)` or `::wrap(..)`, all `.with_target(visual)`.
3. Tween each driver's `t` separately.

Spawn order decides which driver wins when two touch the same path. Group
names passed to `inner`/`wrap` must be `&'static str`.

Typst splits fill and stroke into separate paths, so trace effects look wrong
on text or SVG with no stroke. Give text an explicit stroke (for example
`set text(stroke: col + 10pt)`), and give SVG logos strokes matching their
fills.

## Overlays that follow 3D objects

- Project a world position into overlay pixels with the camera's world-to-NDC
  conversion times half the frame size.
- Draw arrows in Typst and control length with a length argument, not by
  scaling the transform.
- Compute velocity for an arrow from the state, and hide it below a small
  speed. Otherwise it shows at `t = 0`.

## Anchors and layout

- `WorldScene::with_anchor(Vec2::splat(0.5))` centers; `(0.5, 0.0)` anchors at
  top-center (for example something swinging from a shackle).
- The anchor only shifts the origin within the frame's laid-out size. For a
  stable frame, use a `100%` box with `place(center + horizon)` and set
  `with_width` and `with_height`.
- Re-measure any hand-placed positions after changing a Typst layout.
- A code block that overflows the screen needs a width cap.
- Multi-line math that must align belongs in one math function, not separate
  entities.

## Code blocks as a Typst function

A useful pattern for showing code while a scene runs: rows carry numeric
tokens (`{0}`, `{1}`) that sync systems fill from state, collapsible rows
reveal one by one with per-row `KanvaGroup` drivers, and a row can be swapped
in place (`v` becomes `v += dv`) instead of rebuilt.

## Video and audio in scenes

- Decode footage asynchronously into frames and play it by tweening a time
  field, with a separate alpha field for fades. Realtime decode is laggy, and
  whole clips sit in memory, so transcode to a smaller size first.
- Remove stride padding when copying frames. Frames may arrive as BGRA.
- Footage rotation metadata may be ignored by the decoder; size and rotate the
  quad yourself.
- Pause audio on scrub and restart from the timeline position on play.

## Export

- Record with `FixedRatePlayer`, pipe frames to `ffmpeg`, and exit only once
  the file is confirmed saved.
- ProRes 4444 keeps alpha for compositing; H.264 with a low CRF is a good
  no-alpha option.
- Wait for the render pipeline to be ready before recording.
