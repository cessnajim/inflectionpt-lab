# pipeline

The numbered scripts in `infrastructure/ml/experiment/` run in order.
Steps 9–12 belong to a path (Poisson meshing + Gaussian splat) that
was tried, evaluated, and abandoned — they're kept in the repo for
honest reproducibility, but the live deliverable runs 01–08 + 13–17.

## 01 — `01_probe.json` (PDAL)

Read the raw multi-gigabyte X9 LAS, voxel-downsize to 1 m, write
`probe.laz`. Cheap sanity check that the input file is valid and
the coordinate system isn't surprise-projected. Throwaway.

## 02 — `02_crop.json` (PDAL)

Read the cleaned full-extent COPC produced by `decimate-and-upload-scan.sh`,
voxel-downsize to 20 cm via `voxelcenternearestneighbor`, write a
non-compressed LAS for Mask3D. The 20 cm choice is itself a decision —
see [`decisions.md`](decisions.md).

We used to crop here to a 50×50 m bbox around the building. Now we
feed the entire 240×320 m scene through and let step 03 tile it. That
gets Mask3D operating on tens of blocks instead of one — same network
call per block, but coverage of the whole site instead of just the
building footprint.

## 03 — `03_to_mask3d.py`

Build STPLS3D-style validation split. Writes:

- `validation/idahofull_admin_v2_<i>.npy` — per-block 12-column array
  `[x, y, z, r, g, b, n1, n2, n3, segments, semantic, instance]`
- `validation/idahofull_admin_v2_<i>_indices.npy` — per-block → scene
  index map (used by step 05 to stitch predictions back)
- `instance_gt/validation/idahofull_admin_v2_<i>.txt` — dummy GT
  (we have no GT and don't care about Mask3D's metric — we only want
  the predictions written to `eval_output/` when running with
  `general.export=true`)
- `validation_database.yaml` — one row per block
- `blocks_manifest.yaml` — block_id → scene + indices map

Two non-obvious things:

1. The scene name needs exactly 4 underscore-separated parts because
   Mask3D's STPLS3D export path does `scan_id, _, _, crop_id =
   file_names[bid].split("_")`. Naming a block `idaho_admin_chunk_0`
   silently breaks this; `idahofull_admin_v2_0` works.
2. Each block needs at least two unique instance IDs or the inference
   loop hits a `RecursionError` with the message "NO INSTANCES."
   We seed dummy instances when a block has none.

## 04 — `04_run_inference.sh`

`podman run` of the mask3d image. Hydra overrides matching the
authors' `stpls3d_val.sh` plus:

- `general.train_mode=false` — skip training, go straight to
  `runner.test()`
- `general.export=true` — make `trainer.test_epoch_end` return early
  and dump per-block predictions to `eval_output/` instead of trying
  to score them against our fake GT
- `general.use_dbscan=true` with `dbscan_eps=14.0` — what produces
  clean per-instance masks
- `data.voxel_size=0.333` — Mask3D voxelizes internally to 33 cm
  regardless of input density. This is why bumping step 02 from 10 cm
  to 20 cm cost nothing in model fidelity but halved the per-point
  bookkeeping memory and stopped the building blocks from OOM-killing.

Wall time ~17 minutes for 19 blocks on a 4090.

## 05 — `05_predictions_to_las.py`

For each block, read the `*_inst_nostuff.txt` predictions written by
Mask3D, look up the corresponding `_indices.npy` from step 03, and
scatter the per-block per-instance class labels back to a full-scene
LAS.

The original revision read predictions for a single chunk; the
multi-block revision had to add the index-mapping logic. That diff is
in [`commit-log.md`](commit-log.md).

## 06 — `06_densify_classified.py`

The 20 cm classified cloud is too coarse for the web viewer. Build a
k-d tree on the 20 cm cloud, query it with the 3 cm cleaned cloud's
points, and propagate the nearest neighbor's class to each fine-cloud
point. Result: a 50.5 M-point @ 3 cm classified LAS that retains the
full visual fidelity of the input scan with Mask3D's labels painted on.

## 07 — `07_hag_postprocess.py`

CSF (cloth simulation filter) ground extraction → adds class 15
("Ground (CSF)") for any point CSF identifies as ground that Mask3D
left unclassified. Then a height-above-ground sanity layer: anything
> 30 m off the ground that Mask3D labelled as "vegetation" is a false
positive (no trees in this scene are 30 m tall) — flip to unclassified.

This and step 08 together take the v3 (raw Mask3D) classification rate
from ~72% to 91.5%.

## 08 — `08_roof_fix.py`

2D building footprint rule: project all "vegetation"-labelled points
onto the XY plane, dilate the building's known footprint, and any
"vegetation" point inside the dilated footprint with HAG > 2 m flips
to "building (roof)." This catches the most consistent failure mode
of the STPLS3D-trained model: dark slate roofs read as tree canopy
because aerial training data has nothing that looks like a slate roof.

## 09 — `09_mesh_poisson.py` *(retired)*

Per-class Poisson surface reconstruction (building + ground separately)
producing a watertight mesh. Implemented, served on the website for a
while, then removed because the mesh added more visual confusion than
clarity for the actual deliverable. See [`decisions.md`](decisions.md)
for the full reasoning.

## 10–12 — `10_slice_panos.py`, `11_init_pointcloud.py`, `12_splat_to_web.py` *(retired)*

The Gaussian-splat path. Slice the 95 equirectangular panos into
6 perspective images each (cube-map + up), train a splatfacto model
in nerfstudio, post-process the resulting `.ply` for web (alpha clamp,
needle drop, drop-anisotropic) and pack into the antimatter15 gsplat.js
format. Ran the full path; the output never met the visual bar required
to ship next to the Potree viewer. See [`decisions.md`](decisions.md).

## 13 — `13_oneformer_predict.py`

OneFormer Swin-L trained on ADE20K-150, run per-pano. For each of 95
JPEG inputs (1500×1500 each, ~30 MB total), produce
`(label_id_uint8, score_uint8)` PNGs.

Two debug-debt notes:

1. The model was originally loaded with `torch_dtype=torch.float16` to
   save VRAM, which broke convolution because bias terms remained
   `float32`. Fix: load `fp32`, wrap inference in
   `torch.autocast(device_type="cuda", dtype=torch.float16)`.
2. The first confidence calculation used a double-softmax pattern
   that produced ~0.02 mean confidence (uselessly low). Correct form
   is `(sem_max / sem_sum).clamp(0, 1)` where `sem_max` is the per-
   pixel max semantic score and `sem_sum` is the sum across all 150
   ADE classes.

Wall time: ~30 s on a 4090 for all 95 panos.

## 14 — `14_groundedsam_predict.py`

Grounding-DINO-T (text → box) + SAM 2.1-Hiera-Large (box → mask) per
pano. Prompts: car, truck, suv, pickup truck, rooftop hvac unit,
chimney, satellite dish, door, window, sign, person, ladder, trash can,
bicycle. Outputs per-image `.npz` with `masks`, `classes`, `scores`,
`boxes`.

Required updating `transformers` to 4.57.1 and `huggingface_hub` to
0.34.4 to get `Sam2Processor`.

## 15 — `15_depth_buffers.py`

For each camera in `transforms.json`, project the 3 cm classified cloud
into a 1500×1500 depth image using `scatter_reduce_(min)` on the GPU.
Save as `depth/<stem>.npy`. This becomes the visibility test for step 16.

Without z-buffer visibility, projecting 3D points into 2D images and
sampling labels is wrong — points behind walls would inherit labels
from surfaces in front of them. The z-buffer pass is what makes 3D
label lifting honest.

## 16 — `16_lift_to_3d.py`

For each camera:

1. Project the 3 cm cloud into image coordinates
2. Filter to points within frame
3. Filter again by visibility (depth within tolerance of the z-buffer
   value at the projected pixel)
4. Sample the OneFormer label + confidence and the Grounded-SAM-2
   masks at each visible pixel
5. Accumulate weighted votes per point on the GPU using a
   hash-aggregation strategy (this is what keeps memory bounded —
   no scatter into a 50M-row tensor)

Outputs: `lift/ade.npz` (per-point ADE class + confidence) and
`lift/objects.npz` (per-point open-vocab object index + score).

## 17 — `17_fuse_v5.py`

The final decision tree per point:

1. **Open-vocab object detection wins** if its confidence is high
   enough and the class maps cleanly to STPLS3D (cars → vehicles,
   HVAC → new class 16, person → new class 17)
2. **Mask3D unclassified gets ADE fill** — if Mask3D left a point
   unclassified and ADE has a confident label, use the ADE → STPLS3D
   mapping table
3. **Veg-as-building override** — if Mask3D said "vegetation" but
   ADE confidently says "building" or "roof," flip to building. This
   is the same failure mode step 08 catches geometrically; ADE
   catches the photometric version of it.
4. **Default**: keep the v4 Mask3D class

The v4 Mask3D class is preserved in `user_data` for audit. Multi-word
ADE names (e.g. "street lamp," "screen door") survive tokenization
because the lookup splits on `;` and `,` only — not on space.

PDAL writers.copc encodes the result. Final file:
`idaho_admin_full_classified_v5.copc.laz` (~493 MB, 50.5 M points).

## What gets uploaded

- `idaho_admin_full_classified_v5.copc.laz` →
  `s3://.../scans/idaho-admin-classified-v5-3cm.copc.laz`
- CloudFront invalidation on `/scans/idaho-admin-classified-v5-3cm.copc.laz`

The Potree viewer at `/scan-viewer/idaho-admin-classified/index.html`
streams it on demand via HTTP range requests.
