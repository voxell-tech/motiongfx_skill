# Builder API and timeline idioms

Targets motiongfx 0.3.x (`Duration` API). See the version table in `SKILL.md`.

## Pick the right action

- `act` tweens with the type's built-in interpolation (floats, ints rounded,
  glam and peniko types).
- `act_step` does not tween. The value flips when progress reaches 1. Use it
  for visibility, bools, enums, rebuilding data while hidden, and instant
  resets. With a zero duration it is an instant set.
- `act_builder(..).with_interp(f)` supplies a custom `fn(&T, &T, f32) -> T` for
  types with no built-in interpolation (for example an optional Typst color).
  A custom interpolation is often cleaner than a custom ease, such as a sine
  idle bob with a whole number of periods so it ends where it started.

Ints are rounded when interpolated. Tween a float and cast if smoothness
matters.

## Ordering

Free functions in `motiongfx::track`, or `.ord_*()` on any iterator of
fragments.

| Combinator | Duration | Use |
|---|---|---|
| `ord_chain()` | sum | sequential beats |
| `ord_all()` | max | parallel, next step waits for the slowest |
| `ord_any()` | min | parallel, next step starts when the fastest ends |
| `ord_flow(cs(15))` | last start + its duration | stagger: each starts a fixed delay after the previous *start* |
| `delay(d, frag)` | + d | offset one fragment inside an `ord_all` |

```rust
circles.iter().map(|c| c.play(cs(60))).ord_flow(cs(15))       // wave
[zoom, delay(cs(60), [cubes_in, swaps].ord_chain())].ord_all() // nested
chain([[a.play(s(1)), b.play(cs(50))].ord_all(), c.play(cs(50))])
```

Combinators consume their input and are `#[must_use]`. Nest them freely.

`ord_flow` staggers by start offset, so the second item starts after the first
*starts*, not after it finishes. That is what makes a wave.

## Holds

Put `TrackFragment::silent(s(1))` at the start and end of a track so the first
and last states are readable. About one second on each side is a good default.

## Easing

Families: `cubic`, `quad`, `quart`, `sine`, `circ`, `back`, each with
`ease_in`, `ease_out`, `ease_in_out`. Defaults that read well:

- `cubic::ease_out` for entrances
- `sine::ease_in_out` for reveals and `KanvaAnim::t`
- `back::ease_out` for pops with overshoot
- `circ::ease_in_out` for swings
- `cubic::ease_in_out` for general moves

No ease means linear, which is right for constant-velocity motion and for a
time driver.

## Drive one value, derive the rest

Tween a single scalar and compute everything else from it, instead of tweening
many fields that must stay in step.

- For physics-style scenes, keep a small state struct (for example
  `Motion { accel, vel, init_pos, t }` with `position()` computed from those)
  and tween only `t`, with linear ease so `t` tracks real time. Sync systems
  then copy the derived values to the transform, on-screen numbers and arrows.
  Never hand-compute end positions; changing a parameter then updates the
  whole scene.
- To reverse, tween `t` back. The path retraces with no extra code.
- Use `act_step` to set the parameters that change between beats
  (`accel`, `vel`, `init_pos`, and resetting `t` to zero).
- Velocity and similar quantities come from the state (`vel + accel * t`), not
  from position deltas.
- Keep world units separate from physical units. Fold the conversion factor
  into the one method that maps state to position, so displayed values stay
  in real units.
- For Typst reveals, drive `KanvaAnim::t` from 0 to 1, and back to 0 to
  un-reveal. See `typst-velyst.md`.
- A `path!` to a Typst func field (`path!(<VFooFunc>::data::seek)`) animates a
  parameter of the drawn graphic. A nested timeline can be scrubbed from a
  parent by tweening its time field.

If a sync system copies a source component into a target, write it as a plain
system (`sync::<Source, Target>`) rather than a plugin, so calls can be
`.chain()`ed for explicit ordering.

When a later move must start from wherever the object landed (a slide to a
corner, a hand-off), use a position override that latches the current position
while inactive and wins over the driver while active. Then set the new origin
and reset `t`. Keep the chain minimal; drop syncs that turn out redundant.

## Reusable closures for repeated beats

- Wrap a beat in a function that returns a `TrackFragment` and takes
  `(b, ids, params)`. Put durations, delays and eases in arguments or named
  constants so timings live in one place.
- Generate with iterators, then `.ord_flow()` or `.ord_all()`.
- Each `act` borrows the builder's registry, so collect fragments into a `Vec`
  or array first, then combine. Build one tree and compile once; throwaway
  builders per action do not work because action keys only resolve in the
  builder that created them.
- Use `Duration` arithmetic, never float seconds. Chained sums then match the
  intended timing exactly.
- Frame stepping is a loop: `ord_flow(interval)` over `act_step` snaps. Keep
  each clip shorter than the interval (half of it works) or overlapping clips
  on one field panic.
- Track running state (a `start` value) in Rust while building long chains,
  then emit `act_step` snaps per segment.

## Players and manual control

- `RealtimePlayer`: `time_scale` magnitude sets the speed and the sign sets the
  direction, so a negative value rewinds.
- `FixedRatePlayer`: driven by a frame counter and a fixed fps, so exports are
  deterministic.
- `PassivePlayer`: you set the track index and time. It switches track first so
  the time clamp uses the new track.
- Driving a timeline by hand: bake once after building, then
  `set_target_track` before `set_target_time` (the time is clamped to the
  target track), `queue_actions`, then `sample_queued_actions`.
- Multiple tracks act as chapters. Debug by jumping to a section with
  `set_target_time` on a timeline you are working on.

## Subjects

- Entities are subjects for components.
- For assets use the type-erased id (`handle.untyped().id()`), and keep a
  strong handle alive on an entity so the asset is not dropped.
- `act` requires `S: Clone`, so animated func structs need
  `#[derive(Default, Clone)]`. Confirm against your version.
