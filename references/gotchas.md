# Common mistakes and fixes

Items marked (unverified) come from reading source or a single report. Confirm
them against your motiongfx version before relying on them.

| Mistake | Fix |
|---|---|
| Copying float-seconds snippets | Use `s`/`cs`/`ms`; check the crate version |
| Sampling before baking (manual API) | Call `bake_actions` once after building |
| `set_target_time` clamps oddly | Call `set_target_track` first; the time clamps to the target track |
| Tweening a discrete field (visibility, bool) | `act_step` |
| Tweening a whole struct field whose start depends on runtime state | Seed it with `act_step` first, or tween per-axis (`::translation::y`) |
| Short tween gates a long reveal inside `ord_all` | Use `ord_any` (see the caution below) |
| Overlapping actions on aliasing paths of one subject (a struct and its sub-field) | Sequence them with `ord_chain`, or split fields. The later clip may replace the earlier one whole (unverified); enable the `diagnostics` feature and check `Track::conflicts()` |
| Many `act_step` clips on one field panic | Keep each clip shorter than the frame interval; half of it works |
| Animated func struct fails to compile | Derive `Default, Clone` on it (unverified) |
| Asset animation does nothing | Keep a strong handle alive on an entity and use `handle.untyped().id()` as the subject |
| Typst func named `label` | Shadows Typst's `label()`; name it `text_label` |
| Trace effect looks wrong or bunched at the end | Give text and SVG an explicit stroke; Typst emits fill and stroke as separate paths |
| Overlay text or logos fade wrongly | Convert lengths in Typst (`x * 1pt`); check for a raw float |
| Hand-measured arrow positions go stale | Re-measure after any Typst layout change |
| A dim overlay stays visible | Draw it only when its opacity is above 0 |
| Arrow shows at `t = 0` or at zero speed | Compute velocity from state and hide below a small speed |
| Scene goes black after adding HDR | Add `Hdr` to the 2D camera as well as the 3D one |
| Nothing renders in the overlay | The 2D camera needs `VelloView` |
| World scene freezes while the camera moves | Suspect culling, the `Aabb` or the transform matrix before Typst |
| Z-fighting on translucent duplicates | Use opaque, differently coloured copies, or offset z slightly and scale to 0.99 |
| Preview and export differ | Something reads wall-clock time; drive it from timeline time |
| Reversing a beat snaps | Use a latched position override, or tween the driver back |
| Camera follow looks different when recording | Compute the camera target as a pure function of position; avoid lerp toward a target |
| Two Velyst versions in the tree | Align Bevy, vello, Typst and bevy_vello; import through velyst's re-exports |

## Caution on `ord_any`

`ord_any` is documented to end when the fastest fragment ends, but the compiled
track duration may still cover the longest clip (unverified; a report from a
docs-site session found the compile step clamps duration to the latest clip
end). Print the compiled track duration to confirm before relying on
`ord_any` to cut a scene short, and if it does not, truncate the shorter or
longer sequences yourself.

## Debug habits

- Print the compiled track duration and compare it with the intended timing.
- Jump to a section with `set_target_time` on a timeline you are working on
  instead of replaying from the start.
- When a regression appears one commit after a good one, diff against the last
  good commit and look only at what changed.
- Keep one scene enabled at a time while previewing.
