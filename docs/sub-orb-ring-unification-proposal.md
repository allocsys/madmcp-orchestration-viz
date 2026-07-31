# Proposal: unify each delegate's sub-orbs onto a single shared orbit

Status: **proposed, not yet implemented**. No code changes have been made for this
yet -- this document captures the design so a future session can pick it up as a
plan/PR.

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

## Open questions for implementation

- Exact radius/yOffset per group -- needs visual tuning so the sub-orb ring reads
  as clearly "orbiting the delegate" without overlapping the delegate's own node
  glow or (for designerPhases, at delegate_designer's now-repositioned y=7 spot)
  crowding the inner delegate ring's neighbor.
- Whether `DESIGNER_PHASE_ORBIT_SCALE` / `RESEARCH_MODE_ORBIT_SCALE` are still the
  right post-hoc scale factors once radius is a directly-chosen constant instead
  of derived-then-scaled.
- Confirm `updateOrbits()`'s existing resolution order (parent delegate resolved
  before its sub-orbs each frame) still holds -- it should, since this change
  only affects how each sub-orb's `orbit` object is constructed, not the
  per-frame resolution order in `updateOrbits()`.
