# MotionGfx Skill

An agent skill for authoring animations with
[motiongfx](https://github.com/voxell-tech/motiongfx) in Bevy, optionally with
Typst visuals through velyst.

It teaches Claude to:

- use the motiongfx builder efficiently (`act`, `act_step`, ordering,
  easing, one driver value with everything else derived from it, reusable
  fragment functions)
- keep preview and record output identical
- avoid known pitfalls, and check which motiongfx API version a project uses

It targets the `Duration`-based API (motiongfx 0.3.x). Projects on the older
float-seconds API are covered by a migration table in `SKILL.md`.

## Layout

```
SKILL.md                     mental model, preview/record rules, working rules
references/
  builder-api.md             actions, ordering, easing, drivers, players
  typst-velyst.md            cameras, Typst funcs, KanvaAnim, export
  gotchas.md                 mistakes and fixes, with unverified items marked
```

The agent loads `SKILL.md` when the skill triggers and opens a reference file
only when the task needs it.

## Status

This skill was distilled from a series of motiongfx projects. The `SKILL.md` and
reference files mark claims that come from reading source rather than from
running code as "unverified"; confirm those against your version.

## License

MIT. See `LICENSE`.
