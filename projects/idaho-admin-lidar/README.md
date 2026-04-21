# idaho admin — terrestrial LiDAR + image-fusion classification

End-to-end pipeline that takes a raw Faro X9 terrestrial LiDAR capture
of an Idaho administrative building and produces a customer-grade,
classified, browser-streamable point cloud. **98.4% of points
classified** across the full 240×320 m site after the v5 image-fusion
pass.

- **Live viewer:** [/work/idaho-admin/](https://inflectionpt.io/work/idaho-admin/)
- **Article series:**
  - [We ran Mask3D on a terrestrial scan it was never trained on](https://inflectionpt.io/writing/we-ran-mask3d-on-a-terrestrial-scan/)
  - [Your scan is actually two scans](https://inflectionpt.io/writing/your-scan-is-actually-two-scans/)
- **Source repo:** private (`cessnajim/inflectionpt.io`, `infrastructure/ml/`)

## What it does, in one screen

1. Take a raw, multi-gigabyte colorized terrestrial LiDAR scan
2. Denoise + voxel-downsize it to a Mask3D-friendly density (PDAL,
   two-pass: ELM + statistical SOR + voxel center)
3. Tile it into the 50×50 m blocks STPLS3D-trained Mask3D expects
4. Run Mask3D inference on each block (containerized, RTX 4090,
   ~17 minutes for the full 240×320 m site)
5. Stitch per-block predictions back to a full-scene classified LAS
6. Project labels onto a 3 cm cleaned cloud
7. Apply CSF/HAG/footprint post-processing rules to fix the model's
   most consistent failure modes (slate-roof-as-tree, missing ground,
   trees-as-fences)
8. **Image fusion (v5):** OneFormer-ADE20K + Grounded-SAM-2 on the
   95 co-registered scanner panos, lifted to 3D via per-camera z-buffers
   and weighted-vote hash-aggregation on GPU
9. Fuse Mask3D + ADE20K + open-vocab into the v5 classified COPC
10. Push to S3, serve via CloudFront, render in browser with Potree

The classification rate journey was 72% (raw Mask3D) → 91.5% (v4 with
post-processing) → 98.4% (v5 with image fusion).

## Stack

- **Containerization:** Podman (3 images: `inflection-mask3d`,
  `inflection-imgseg`, `inflection-splat`)
- **3D ML:** Mask3D (transformer-based 3D instance segmentation, RWTH
  Aachen) with the published STPLS3D checkpoint
- **2D ML:** OneFormer-ADE20K (Swin-L), Grounded-SAM-2 (Grounding-DINO-T
  + SAM 2.1-Hiera-Large)
- **Point-cloud tooling:** PDAL, laspy, NumPy, k-d tree NN
- **Web:** Cloud-Optimized Point Cloud (COPC) over CloudFront with HTTP
  range requests, rendered in Potree
- **Cloud:** S3 + CloudFront with OAC, fronted by an existing
  `inflectionpt.io` distribution
- **Hardware:** Single RTX 4090 (24 GB VRAM)

## Status

Customer-grade, deployed, live. The classified COPC at
`/scans/idaho-admin-classified-v5-3cm.copc.laz` is what the work page
viewer streams. Splat path was attempted and abandoned (see
[`decisions.md`](decisions.md)). Mesh path was implemented and removed
from the website (see same).

The deeper "what worked, what broke, what it means for clients
evaluating off-the-shelf 3D AI" write-up is the [Mask3D article on the
live site](https://inflectionpt.io/writing/we-ran-mask3d-on-a-terrestrial-scan/);
this folder covers the architecture and decisions that the article
doesn't fit.

## See also

- [`architecture.md`](architecture.md) — container layout, data flow, where each piece runs
- [`pipeline.md`](pipeline.md) — the 17-step sequence, what each step does, why it exists
- [`decisions.md`](decisions.md) — splat abandonment, mesh removal, voxel size, full-scene vs cropped, image-fusion choice
- [`commit-log.md`](commit-log.md) — curated extracts from the private repo's commit history
