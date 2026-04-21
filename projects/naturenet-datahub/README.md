# NatureNet DataHub

Geospatial ML platform on AWS for invasive-species monitoring.
Ingests aerial imagery (drone + fixed-wing), produces orthomosaics,
runs object detection on tiled rasters, and publishes versioned,
reproducible exports for downstream conservation work.

- **Case study:** <https://cessnajim.github.io/projects/datahub/>
- **Source repo:** private (`cessnajim/naturenetdatahub`)

## What it does

- Ingests raw aerial imagery
- Stitches into orthomosaics with GDAL
- Tiles to a slippy-map tileset for interactive viewing
- Runs RF-DETR (real-time DETR variant) on the tiles to detect
  invasive species at field-relevant resolutions
- Versions every artifact (raw imagery, orthomosaic, tile cache,
  detection results) so a year-over-year comparison is defensible
  rather than vibes
- Publishes detections + orthomosaic + raw imagery as a coherent
  reproducible export bundle

## Stack

- **ML:** RF-DETR for detection, custom training loop for the
  invasive-species classes
- **Geospatial:** GDAL for orthomosaic + reprojection, MapLibre for
  the interactive viewer
- **Cloud:** AWS (S3 for imagery, Lambda for tile generation,
  versioned bucket policies for reproducibility)
- **API:** Python service layer fronting the ML + GIS plumbing

## What's interesting

Versioning is the thing most "ML for conservation" platforms get
wrong: detections this year are compared to last year's detections,
but there's no reproducible way to re-run last year's model on this
year's imagery (or vice versa) to separate model drift from actual
ecological change. The DataHub treats every artifact (imagery,
orthomosaic, model weights, detections) as a versioned object and
makes the comparison defensible.

## Status

Used by the partner organization for active monitoring work.

## What this folder will eventually contain

The architecture (raster ingest → ortho → tile → infer → export
pipeline), the versioning model, decisions around RF-DETR vs.
alternatives (YOLO, traditional Faster R-CNN), and the AI-prompted
build notes specific to a long-lived data platform vs. a one-shot
classification pipeline.
