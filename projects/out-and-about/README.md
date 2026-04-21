# Out & About with Jim

Personal photo & video gallery — 140+ pieces across drone aerials,
landscapes, and travel — indexed on an interactive map. Acts as the
publish target for the [DAM Core editor](../dam-core/README.md).

- **Live site:** <https://outandaboutwithjim.com>
- **Video library:** <https://www.youtube.com/@outandaboutwithjim>
- **Case study:** <https://cessnajim.github.io/projects/gallery/>
- **Source repo:** private (`cessnajim/outandaboutwithjim`)

## What it does

- Storefront / gallery for finished photo + video work
- Interactive map indexed by capture location
- Pulls publish-ready assets from the DAM Core review pipeline
- Public-facing performance and SEO targets — this is a real site
  people land on, not a portfolio scratch surface

## Stack

- **Backend:** Django
- **Database:** PostgreSQL (with PostGIS for the map index)
- **Frontend:** MapLibre for the interactive map, server-rendered
  templates for the gallery
- **Hosting:** self-managed VPS

## Why it's in the lab repo

It's a different shape of project from the others. The LiDAR pipeline
is a one-shot batch processor. DAM Core is a long-lived workflow tool.
NatureNet DataHub is a versioned data platform. The PM KB workspace
is a personal knowledge-work system. Out & About is a public-facing
read-mostly product with SEO, performance, and content-management
constraints. All of them were AI-prompted end to end — which means
the lab repo is partly a study in how the AI-prompted method holds up
across very different shapes of software.

## Status

In production. Continuously fed new assets via the DAM Core pipeline.

## What this folder will eventually contain

Notes on how the prompting practice differs for "shipping a product
that real strangers visit" vs. the back-office tools above. Decisions
around server-rendered Django vs. an SPA, why MapLibre over Mapbox
or Leaflet, and the SEO/performance work that an AI agent doesn't
volunteer unless you specifically prompt for it.
