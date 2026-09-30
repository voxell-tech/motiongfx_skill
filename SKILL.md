---
name: motiongfx
description: >
  Author animations with motiongfx in Bevy, optionally with Typst visuals through velyst. Use when building, editing or debugging motiongfx timelines: tweening fields with act,
  act_step or act_builder, ordering fragments (chain, all, any, flow, delay, ord_flow), Duration helpers
  (s, cs, ms), easing, the TimelineBuilder, players (RealtimePlayer, FixedRatePlayer, PassivePlayer),
  KanvaAnim and KanvaGroup reveals, or preview vs record output.
---

# motiongfx

How to use motiongfx efficiently. Targets the `Duration`-based API (motiongfx 0.3.x).

Detail lives in `references/`. Read the file that matches the task instead of
loading everything:

| Task | Read |
|---|---|
| Writing or editing a timeline (act, ordering, easing, drivers, closures) | `references/builder-api.md` |
| Typst visuals, cameras, KanvaAnim, velyst setup | `references/typst-velyst.md` |
| Something behaves oddly, or an error appears | `references/gotchas.md` |

## Version check (do this first)

Read the project's `Cargo.toml` and lockfile, then run `cargo check` before
writing any scene. Older code and snippets pass float seconds; that API is
gone in the current version.

| Old (float seconds) | Current (`Duration`) |
|---|---|
| `.play(1.6)` | `.play(cs(160))` or `.play(ms(1600))` |
| `TrackFragment::new().delay(1.0)` | `delay(s(1), frag)` or `TrackFragment::silent(s(1))` |
| `.ord_flow(0.3)` | `.ord_flow(cs(30))` |
| `b.add_tracks(..); b.compile()` | `b.compile(track)` |
| separate `velyst_motiongfx` crate | `bevy_motiongfx` with the `velyst` feature |

`s`, `cs` (centiseconds) and `ms` live in `motiongfx::time`. `s()` takes whole
seconds, so use `cs` or `ms` for fractions. Bevy 0.19 pairs with motiongfx 0.3,
0.18 with 0.2, 0.17 with 0.1.

## Mental model

1. `b.act(subject, path!(<Type>::field), |old| new)` says what changes. The
   closure maps the current value to the target, so it can be relative
   (`|x| x + 6.0`) or absolute (`|_| 1.0`).
2. `.with_ease(f)` and `.with_interp(f)` say how. `.play(duration)` says how
   long and returns a `TrackFragment`.
3. Combine fragments with ordering, then `.compile()` to a `Track` (a chapter),
   then `b.compile(track)` to a `Timeline`.
4. Baking runs every closure once and stores start and end values. Sampling
   only interpolates two baked values, so scrubbing backward is free and any
   time can be sampled in any order.
5. Actions on the same field chain in start-time order, so a later relative
   closure builds on the earlier one's end value.

```rust
app.add_plugins((DefaultPlugins, BevyMotionGfxPlugin));

let mut b = motiongfx.create_builder();
let track = b
    .act(entity, path!(<Transform>::translation::x), |x| x + 6.0)
    .with_ease(ease::cubic::ease_in_out)
    .play(s(1))
    .compile();
commands.spawn((
    motiongfx.add_timeline(b.compile(track)),
    RealtimePlayer::new().with_playing(true),
));
```

Three habits pay off most: tween one driver value and derive the rest, build
repeated beats as functions that return fragments, and use `Duration`
arithmetic everywhere. `references/builder-api.md` covers all three.

## Preview and record must match

- Preview uses `RealtimePlayer`. Record uses `FixedRatePlayer` (a frame counter
  at a fixed fps), so exports are deterministic.
- Anything not driven by timeline time (noise drift, wall clock, `Time` deltas,
  lerp-toward-target camera follow) differs between runs. Write those as pure
  functions of the timeline's current time or of state the timeline drives.
- Derive playback length from the compiled timeline's duration. Never
  hard-code it.

## Working rules

- Verify with `cargo check`. Do not launch windowed apps to judge a visual
  result.
- Expose named constants for positions, spacing and durations so they are easy
  to tune.
- Comments: as few as possible, one line, explaining why. No banner comments.
