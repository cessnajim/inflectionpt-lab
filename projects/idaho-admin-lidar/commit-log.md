# commit log (curated)

The five commits that took the private `inflectionpt.io` repo from
the initial scaffold to the current production state. Each commit
message is itself an explanation artifact — extracted here so the
reasoning is readable without repo access.

The commit hashes are real and can be verified in the private repo if
that ever becomes available to a reader (e.g. inside a client
engagement).

---

## `35f1946` — Tighten gitignore for ML pipeline transients and document regen path

> The `infrastructure/ml/` tree carries ~29 GB of upstream Mask3D clone,
> checkpoints, raw scan data, processed blocks, image-fusion buffers, and
> classification outputs that are either huge, customer data, or
> regenerated from the committed scripts (and the final COPCs live on
> S3 anyway). Ignore all of it, with explicit re-includes for the two
> hand-crafted Mask3D config yamls (label_database, color_mean_std).
>
> Also add `infrastructure/ml/README.md` walking through the regen path
> from a fresh clone — vendor Mask3D, drop in the STPLS3D checkpoint,
> build the three containers, then run `experiment/01..17_*`. Lands the
> working tree at ~41 MB, of which 42 MB is vendored Potree.

**Why this commit existed:** the ML pipeline produces large
intermediate artifacts that aren't source. The previous gitignore was
too permissive and would have committed gigabytes of regenerable
data. This commit also acknowledges that hiding the regenerable data
creates a reproducibility gap, and writes the README that closes it.

---

## `f8178b6` — Build out site: pages, content, components, contact + subscribe wiring

48 files. The site itself: pages (about, decision-review,
operations-ai-engagement, production-ai-review, strategy-sprint,
subscribed, thanks, plus work/ writing/ playbook/ index + dynamic
slug routes), 5 long-form articles, 6 playbook chapters wired through
content.config.ts, 6 components, the layout + nav + homepage,
Cloudflare Pages functions for the contact + subscribe forms, AWS
Lambda backers behind those functions.

The about page calls out that DAM Core, NatureNet DataHub, Out & About
with Jim, this advisory site, and the LiDAR classification pipeline
behind /work/idaho-admin/ were all built end to end through AI
prompting — the advisory practice's "what can an agent actually
carry" answer comes from shipping these.

**Why this commit existed:** the previous state was a stub. This is
the working site.

---

## `f0cf7cb` — Vendor Potree and ship scan + splat viewers

411 files. Embeds the open-source Potree WebGL point-cloud viewer
(~42 MB) under `public/potree/` so Cloudflare Pages serves it directly —
no per-seat license, no managed SaaS, just CDN-cached static assets.

`public/scan-viewer/idaho-admin/` and `idaho-admin-classified/` host
the two iframe-embedded viewers used by /work/idaho-admin/. The
classified viewer paints points by STPLS3D class, with HVAC and person
classes seeded by the Grounded-SAM-2 image-fusion pass, and toggles
the v5-fused class counts derived from `17_fuse_v5.py`.

`public/splat-viewer/idaho-admin/` is the retired antimatter15
gsplat.js viewer page from the Gaussian-splat experiment — left in
place but no longer linked from /work/idaho-admin/.

**Why this commit existed:** the viewer assets are big. Vendoring
them in git is the simplest thing that works for a static-site
deploy. Versioning them with the rest of the site means upgrades are
explicit, not surprise.

---

## `2b637b3` — Provision AWS scans bucket and CloudFront pipeline

Idempotent shell scripts that stand up the inflection-point-advisory
isolation slice in the shared AWS account and wire it to the existing
CloudFront distribution that fronts inflectionpt.io.

```
00-config.sh        common env (account, region, distribution ID,
                    naming + tagging conventions)
01-tag-existing     backfill Project/Entity/Environment/ManagedBy
                    tags on the pre-existing CloudFront distribution
                    so Cost Explorer rolls up cleanly
02-provision        S3 bucket inflection-point-advisory-scans with
                    bucket-key SSE, versioning, public-access-block,
                    and OAC for CloudFront-only access
03-add-behavior     attach a /scans/* behavior to the distribution
                    that proxies to the new bucket via OAC
04-apply-policy     bucket policy granting the OAC sole read access
decimate-and-upload two-pass PDAL pipeline (ELM + statistical SOR
                    noise filtering, voxel decimation) that turns a
                    raw Faro X9 LAS into a customer-grade COPC and
                    uploads it under s3://.../scans/
```

Designed for clean lift-and-shift into a dedicated AWS account once
the practice grows past the shared-account stage.

**Why this commit existed:** the infrastructure is itself a
deliverable. Tagging, naming, and isolation discipline now means a
multi-day migration becomes a half-day migration when the practice
spins out into its own account.

---

## `433fdcf` — Add Mask3D + image-fusion classification pipeline

The full ML pipeline. Three containers (`mask3d`, `imgseg`, `splat`),
17 numbered experiment scripts plus the two hand-crafted Mask3D
config yamls. The commit message is a 60-line method document:

```
Pipeline (experiment/, run in numeric order):
  01_probe / 02_crop  PDAL: noise-filter the raw X9 LAS, voxel-downsize
                      to a Mask3D-friendly 20 cm density.
  03_to_mask3d        Tile the cleaned cloud into STPLS3D-style
                      50x50 m blocks with the 12-column .npy layout
                      Mask3D's loader expects; auto-handles the
                      "two unique instance IDs per block" requirement
                      that triggered NO INSTANCES errors on the first
                      pass.
  04_run_inference    Hydra-driven podman invocation of Mask3D's test
                      harness with general.export=true; ~17m on a 4090
                      across 19 blocks.
  05_predictions      Stitch per-block predictions back to the full
                      scene LAS via the indices arrays from 03.
  06_densify          Project block-level labels onto the 3 cm cleaned
                      cloud via k-d-tree NN.
  07_hag_postprocess  Add CSF-confirmed ground (class 15) using a
                      cloth simulation filter + height-above-ground
                      sanity layer.
  08_roof_fix         2D building-footprint rule that flips slate-
                      roof-as-tree misclassifications back to building.
  09_mesh_poisson     Per-class Poisson reconstruction (retired path,
                      mesh section dropped from the website).
  10/11/12            Pano slicing + initial pointcloud + splat-to-web
                      packaging for the retired Gaussian-splat path.
  13_oneformer        Per-pano ADE20K semantic segmentation (OneFormer
                      Swin-L). fp32 weights with autocast(fp16) to dodge
                      the conv-bias dtype mismatch; confidence is
                      sem_max / sem_sum, not double-softmax.
  14_groundedsam      Per-pano open-vocab object masks (Grounding-DINO
                      + SAM 2.1-Hiera-Large) for cars, HVAC units,
                      doors, windows, etc.
  15_depth_buffers    Per-camera z-buffer rasterization of the 3 cm
                      cloud for visibility testing during 2D->3D lift.
  16_lift_to_3d       Project + visibility-test + weighted-vote per
                      point; hash-aggregation on GPU keeps it inside
                      24 GB VRAM on a 50 M-point cloud.
  17_fuse_v5          ADE -> STPLS3D mapping + open-vocab override
                      rules + veg-as-building flip. Preserves the v4
                      Mask3D class in user_data for audit. Multi-word
                      ADE names (e.g. "street lamp") survive
                      tokenization via comma/semicolon-only splitting.
                      Writes the v5 classified COPC.
```

**Why this commit existed:** the pipeline is the project. The
commit message is intentionally long because it is the index that
makes the per-script source readable. Anyone reading the source
should be able to read this commit message first and know exactly
what each numbered file is supposed to do.

---

## What this log is *not*

It's not the full history. The work to get to the current state was
roughly two weeks of iterative sessions, a lot of which involved
trying paths that didn't work (the splat path is documented in
[`decisions.md`](decisions.md); the in-flight commit history of
those experiments isn't preserved in the public repo). The five
commits above are the *consolidated* state, written as explanation
artifacts after the experiment phase was complete.

That's its own decision: the commit history of the *exploration* is
private and messy; the commit history of the *deliverable* is public
(here) and organized. Both are honest representations of different
phases of the same work.
