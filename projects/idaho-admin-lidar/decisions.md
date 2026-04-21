# decisions

The non-obvious calls made during this project. Each entry: what we
decided, what we considered, why we picked the one we did, and (where
relevant) what we'd revisit.

## STPLS3D checkpoint over scratch training

**Decided:** use Mask3D's published STPLS3D-trained checkpoint and
accept that it will get terrestrial scans wrong in predictable ways.

**Considered:** training a Mask3D head from scratch on terrestrial data
or fine-tuning the STPLS3D checkpoint on a labelled subset of the
Idaho scan.

**Why:** the project is a research probe specifically aimed at the
question "can a client get useful work out of an off-the-shelf 3D
model without a labelling project?" Bringing our own labels would
answer a different question. The honest framing is: this is what the
shelf gives you, and here's the post-processing that makes the shelf
output usable for a real deliverable.

**Revisit when:** a client engagement actually wants a custom-trained
head. Then the decisions tree flips entirely — you'd start by spending
two weeks on labelling tooling.

## 20 cm input voxel, not 10 cm

**Decided:** voxel-downsize to 20 cm before Mask3D inference.

**Considered:** 10 cm (what the original single-chunk pipeline used).

**Why:** Mask3D voxelizes internally to 33 cm regardless of input
density (`data.voxel_size=0.333`). At 10 cm the building-cluster
blocks contained 1M+ points and OOM-killed the inference process. At
20 cm those same blocks fit, and the 33 cm internal voxel means the
model literally never sees the difference. Step 06 (densify) propagates
labels back onto the 3 cm cleaned cloud so the visual fidelity is
preserved end-to-end.

**Lesson:** match input density to internal model resolution, not to
visual aesthetics. The web-facing density gets restored at the end.

## Full-scene multi-block, not 50×50 m crop

**Decided:** segment the entire 240×320 m scene by tiling into 19
STPLS3D-style blocks.

**Considered:** keep cropping to a 50×50 m bbox around the building.
That was what the v1–v3 pipelines did.

**Why:** customer deliverables don't get to crop the parts that look
good. Showing a classified building floating in unclassified space is
a bad demo. The full-scene tile approach produces a defensible product
across the whole site.

**Cost:** ~17 minutes of inference instead of ~2 minutes. Acceptable
given this is a one-shot deliverable, not a per-request service.

**Bug surfaced by this change:** "NO INSTANCES" error when a block
contained zero predicted instances. Fix in step 03: seed dummy
instances when needed. Recorded here because it's the kind of thing
a bench-scale evaluation never sees.

## Cleaned cloud as Mask3D input, not the raw LAS

**Decided:** run the noise-cleaned, voxel-downsized cloud through
Mask3D, not the raw LAS.

**Considered:** running the raw LAS directly (ELM + outlier filtering
at the point of inference).

**Why:** the customer-facing COPC was already going through PDAL noise
cleanup for the Potree viewer (it's a customer deliverable). Reusing
that same cleaned cloud as the Mask3D input means there's a single
source of geometric truth. If a label looks wrong on the viewer, you
can blame the model, not "wait, did the model see the same points
the viewer is showing?"

**Operational benefit:** the two-pass PDAL pipeline that produces the
cleaned COPC is the same one that produces the Mask3D input. One
artifact, two consumers. (See `decimate-and-upload-scan.sh` in the
private infrastructure repo.)

## CSF over PMF for ground

**Decided:** Cloth Simulation Filter for the ground recovery pass in
step 07.

**Considered:** Progressive Morphological Filter, the SMRF (simple
morphological filter) variant, and writing our own ground rule from
HAG alone.

**Why:** CSF handles the slope discontinuities of a real building
plot (paths, retaining walls, the building footprint itself) without
the parameter-tuning pain of PMF. PDAL's `filters.csf` works out of
the box on this data with default parameters. PMF needed three rounds
of `cell` / `slope` / `max_window` tuning before it stopped wrapping
the building base in "ground."

**Cost:** CSF is slower per point. At our 3 cm density it adds ~30 s
to the pipeline. Acceptable.

## Footprint rule for the slate-roof failure mode

**Decided:** add an explicit 2D building-footprint rule (step 08) that
flips "vegetation" to "building (roof)" when the point is inside the
building's plan-view footprint and HAG > 2 m.

**Considered:** retraining or fine-tuning Mask3D so it stops doing
this. Bigger hammer, longer timeline.

**Why:** the model's failure is photometric, not geometric. A dark
slate roof reads as canopy because nothing in STPLS3D's aerial training
set looks like a slate roof. The 2D footprint rule is cheap, auditable,
and doesn't risk breaking other classes the way fine-tuning would.

**Where this leaks:** any vegetation that's actually growing on top
of the building (none here, common in green-roofed buildings) would
get misclassified as roof. Documented limitation, not a hidden bug.

## Image fusion pass instead of more 3D ML

**Decided:** lift 2D foundation-model predictions (OneFormer + Grounded-SAM-2)
onto the 3D cloud via per-camera z-buffers, fusion in `17_fuse_v5.py`.

**Considered:** OpenScene, SemanticNeRF, OpenMask3D, Open3DIS, SAL3D —
the published "open-vocab 3D" stack. All are research-grade, none
have pre-trained weights for arbitrary scenes, all need either a
labelled scene or a NeRF/Splat fit first.

**Why:** the 95 panos already exist (they're from the X9's onboard
cameras, automatically co-registered with the LiDAR). Z-buffer
visibility + weighted-vote lifting is a 200-line script with no model
training. The win was decisive: 91.5% → 98.4% classification on the
full scene.

**Where this is honest about its limits:** open-vocab object detection
is sparse. Of 14 prompts (`car`, `hvac unit`, `door`, `window`, etc.),
only `window` got dense enough hits to materially change the visible
viewer. The HVAC and person classes (16, 17) have hundreds of points,
not millions — they're the kind of thing you'd surface as click-to-
inspect annotations, not as bulk-painted classes.

## Splat path: tried and abandoned

**Decided:** abandon Gaussian Splatting as a deliverable for this site.

**What we tried:** trained a splatfacto model in nerfstudio from the
95 panos + the LiDAR as a depth prior, post-processed the resulting
`.ply` (alpha clamp, anisotropy filtering, "needle" drop, several
rounds of tightening), packed for web with antimatter15's gsplat.js
viewer.

**What broke:** every iteration produced a "spiky blob" — splats
elongated into needle shapes that no amount of post-filter could turn
into a recognizable building. The fundamental issue: pano-derived
splats don't have the multi-view parallax that splatting was designed
for. The X9 captures from a single point per scan station; splats
need the *between-station* views to triangulate Gaussian shapes
correctly.

**Alternative paths:** the X9's video walk would provide the parallax,
but we don't have that capture. The "right" tool for converting an
existing terrestrial scan into a render-branch artifact is probably
NeRFs over splats; we didn't try that because the deliverable is
model-branch (classified COPC), not render-branch.

**Lesson:** abandoning was the right call within ~3 sessions of
"that doesn't work." Continuing to chase quality on the wrong-tool-
for-the-input was the failure mode being avoided. The retired scripts
(10–12) stay in the repo because honest abandonment is part of the
explanation artifact.

## Mesh path: built, deployed, then removed

**Decided:** remove the per-class Poisson mesh deliverable from the
website.

**What we built:** step 09 ran per-class Poisson reconstruction on
the v3 building points and the v3 ground points separately, producing
a watertight 330 k-tri GLB (decimated from 1.8 M tris of full-res
PLY for engineering use). It served from `/scans/idaho-admin-mesh-v1.glb`
through `<model-viewer>` for several days.

**Why we removed it:** the mesh added more visual noise to the work
page than clarity. Three viewers stacked on a portfolio page is one
viewer too many. The classification COPC is the headline deliverable;
the mesh was an interesting follow-on, not a primary artifact.

**What we kept:** the script (`09_mesh_poisson.py`) is still in the
repo. The architecture-and-decisions write-up that *was* a published
article (`we-turned-the-scan-into-a-mesh.md`) was deleted from the
content tree. That deletion is recorded in [`commit-log.md`](commit-log.md).

**Where this leaves the mesh idea:** valid model-branch artifact for
a real client engagement (it's the substrate for BIM exchange,
change-detection, energy modelling). Just not the right thing for the
public portfolio piece, where the COPC viewer is the demo and the
mesh is a less-good second viewer of the same scene.

## Private repo for the source, public lab repo for the method

**Decided:** keep the source for `inflectionpt.io` and the ML pipeline
private; publish architecture, decisions, pipeline reasoning, and
curated commit messages here.

**Considered:** going fully public (one command), or keeping everything
private and relying on the live site for explanation.

**Why:** see [`../../decisions/003-private-source-public-method.md`](../../decisions/003-private-source-public-method.md).
Short version: the source isn't where the comprehension lives; the
explanation artifacts are. Publishing the artifacts gets the
accountability without the "now I have to support every drive-by
question about every line" cost of full open source.

**Revisit when:** a client engagement specifically needs the source
opened. Or when there's a reason for the source to be the artifact
(a library, a CLI, a reusable component).
