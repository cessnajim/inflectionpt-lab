# 002 — When to abandon an experiment

**Status:** active heuristic
**Date:** 2026-04

## What

Two paths inside the Idaho admin pipeline were implemented, run to
the point of producing real output, evaluated, and then abandoned:

1. **Gaussian splat path** (steps 10–12). Trained a splatfacto model
   from the X9 panos using the LiDAR as a depth prior. Produced
   visible output, never met the bar, abandoned after ~3 sessions of
   "needles everywhere" iteration.
2. **Mesh path** (step 09). Per-class Poisson reconstruction produced
   a watertight 330k-tri GLB; deployed to the live site for several
   days; removed because it added more visual confusion than clarity.

These weren't half-finished experiments cut for time. Each ran to
"this works at the level it can work" and was abandoned because that
level wasn't useful enough. That's a different — and harder —
abandonment to call.

## The heuristic

Three signals that an experiment should be abandoned:

1. **Wrong tool for the input.** The splat path was abandoned because
   pano-only capture doesn't give the multi-view parallax splatting
   was designed for. No amount of post-filter recovers from that.
   Continuing to chase quality on the wrong-tool path is the
   classic "sunk cost" failure mode.

2. **Right output, wrong product.** The mesh was a real, valid
   model-branch artifact. It just wasn't the right *third* viewer on
   a portfolio page that already had two. "This is a good thing in
   the abstract" is not the same as "this is the right thing to ship
   here."

3. **Cost of keeping it exceeds cost of rebuilding it later.** The
   mesh script (`09_mesh_poisson.py`) stays in the repo because
   keeping a Python file is free. The website mesh viewer was removed
   because keeping it required ongoing maintenance (model-viewer
   version pins, GLB hosting, page layout space). Abandonment isn't
   binary — what gets thrown away depends on what carrying it costs.

## Symmetric heuristic — when *not* to abandon

The 91.5% → 98.4% jump from image fusion looked at first like more
post-processing band-aids on top of Mask3D's failures. It was
tempting to call "we're done at 91.5%, the rest is incremental." Two
reasons we kept going:

1. The 91.5% number, looked at honestly, contained a known structural
   gap (no model in the system had ever seen the panos, even though
   the panos contained per-pixel ground-truth-quality information about
   what the operator could see in the scene). Closing a structural
   gap is different from chasing diminishing returns on the same
   signal source.

2. The proposed fix was small (200-line lift script + a fusion rule
   table) relative to the size of the gap. Cheap intervention into
   a structural gap is the *opposite* of the abandonment heuristic —
   it's exactly when to keep going.

## How this decision shows up in the artifacts

- The retired splat scripts (10–12) stay in the repo with a comment
  noting they're retired, not deleted. Honest reproducibility means
  keeping the dead-end visible.
- The retired mesh script (09) stays in the repo with the same
  treatment. The mesh *article* on the live site was deleted because
  an article that points to a viewer that no longer exists is worse
  than no article.
- This lab repo's [`projects/idaho-admin-lidar/decisions.md`](../projects/idaho-admin-lidar/decisions.md)
  documents both abandonments under their own sections. They're not
  hidden; they're part of the explanation artifact.

## Anti-pattern this avoids

The most common AI-prompted-development failure mode in 2026 is
"the agent will keep iterating on anything you point it at." Without
an explicit abandonment heuristic, the agent will produce a 17th
revision of a splat with marginally fewer needles. The operator's
job is to call the wrong-tool diagnosis and stop the spiral.

## Revisit when

- An experiment runs longer than ~3 sessions without converging on
  either "ship it" or "abandon it." That's the signal that the
  abandonment heuristic isn't being applied.
