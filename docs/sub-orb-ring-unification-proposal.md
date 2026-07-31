# Proposal: unify each delegate's sub-orbs onto a single shared orbit

Status: **implemented**. Step 1 (designerPhases) and Step 2 (researchModes)
are both in `index.html` -- see the `STEP 1 (sub-orb-ring-unification)` and
`STEP 2 (sub-orb-ring-unification)` comment blocks there. This document is kept
as the design record; see "Implementation notes" at the bottom for how the open
questions below were resolved.

## Current state

`delegate_designer`'s and `delegate_research`'s sub-orbs are each derived
independently from their own hand-placed static position, not from one shared
ring:

- **designerPhases** (`generate`, `validate`, `fix`) -- each phase's orbit comes
  from `orbitFromStaticPos(p.pos, parentDelegate.pos, ...)`, computed from that
  phase's own static `(x, y, z)` relative to `delegate_designer`. Their derived
  `yOffset`s differ slightly (`-2`, `-3`, `-3` from the parent), and radii/phases
  aren't evenly spaced by construction -- they just fall out of the original
  hand-placed layout, then get uniformly scaled by `DESIGNER_PHASE_ORBIT_SCALE`.
- **researchModes** (`precision`, `wide`) -- same pattern, derived independently
  from each mode's own static pos relative to `delegate_research`, with
  `yOffset`s of `-3` and `-4`.

Because each sub-orb's `{radius, tilt, yOffset}` differs from its siblings',
`maybeAddOrbitRing`'s dedupe key (`parentId + radius + tilt + yOffset`) does not
collapse them -- each sub-orb currently draws its own separate faint orbit ring.
This is different from the top-tier delegate/service rings, which were
deliberately put on one shared radius/speed/yOffset in the ring-restructure plan
so they collapse into a single shared ring mesh.

## Proposed change

Give each delegate's sub-orb group one shared fixed orbit, mirroring the pattern
already used for `DELEGATE_RING_RADIUS`/`SERVICE_RING_RADIUS`:

1. Replace each sub-orb's `orbitFromStaticPos(...)` derivation with one shared
   fixed `radius` and `yOffset` per group (one value for all of designerPhases,
   another for all of researchModes).
2. Evenly space `phase` by `360deg / n` -- `120deg` apart for the 3 designerPhases,
   `180deg` apart for the 2 researchModes -- instead of phases derived from the
   old static layout.
3. Keep `speed` shared within each group so relative spacing stays constant over
   time. This is what gives the no-drift/no-collision guarantee algebraically
   (verified by simulation for the top-tier rings in the ring-restructure and
   delegate-reposition plans' Step 5s) -- since every member of a ring shares one
   radius and one angular speed, their relative phase offsets are time-invariant.

Expected result: `maybeAddOrbitRing`'s existing dedupe key would then naturally
collapse each group onto one shared ring mesh per delegate -- 3 designerPhases on
one ring around `delegate_designer`, 2 researchModes on one ring around
`delegate_research` -- visually matching how the top-tier service/delegate rings
already read, instead of each sub-orb tracing its own separate faint path.

## Open questions for implementation (resolved)

- **Exact radius/yOffset per group** -- chosen close to each group's old
  individually-derived range so the footprint didn't jump visually:
  - designerPhases: `DESIGNER_PHASE_RING_RADIUS = 2.2`,
    `DESIGNER_PHASE_RING_Y_OFFSET = -2.7` (old range was ~2.0-2.8 radius,
    -2/-3 yOffset).
  - researchModes: `RESEARCH_MODE_RING_RADIUS = 2.0`,
    `RESEARCH_MODE_RING_Y_OFFSET = -3.5` (old range was ~1.8/2.24 radius,
    -3/-4 yOffset).
  Checked against both the delegate node's own glow sprite radius and the
  inner `DELEGATE_RING_RADIUS` neighbor at delegate_designer/delegate_research's
  repositioned y=7 spot -- the closest point on either sub-orb ring
  (`sqrt(radius^2 + yOffset^2)` from the parent, since tilt=0 keeps y constant
  around the ring) clears the parent's glow sprite footprint in both cases, so
  no further tuning was needed.
- **`DESIGNER_PHASE_ORBIT_SCALE` / `RESEARCH_MODE_ORBIT_SCALE`** -- moot: both
  are gone. Once radius became a directly-chosen constant per group there was
  nothing left to scale, so each `orbit` object is now built as a literal
  `{radius, phase, yOffset, speed, tilt}` rather than derived-then-scaled.
- **`updateOrbits()` resolution order** -- confirmed unchanged. Both groups are
  still resolved in the same forEach blocks positioned after `graph.delegates`
  in `updateOrbits()`, so each sub-orb reads its parent delegate's
  already-current-frame position, same as before this change.
