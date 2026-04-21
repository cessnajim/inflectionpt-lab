# DAM Core

AI-assisted editor that turns a 50,000-asset Immich library into a
working publishing pipeline. The "publishing" target is
[Out & About with Jim](../out-and-about/README.md) — a personal
photo and video gallery — but the editor itself is built as a
generic enough product to backstop any DAM that needs an AI-aware
review and curation surface in front of a high-volume asset library.

- **Live target:** <https://outandaboutwithjim.com> (the publish site)
- **Case study:** <https://cessnajim.github.io/projects/dam/>
- **Source repo:** private (`cessnajim/DAM`)

## What it does

- Sits in front of an Immich library as a review/curation/edit surface
- Runs OpenCV + local LLM passes (via Ollama) per asset for tagging,
  categorization, scene description, and quality scoring
- Hands curated, edited, captioned exports to the publish pipeline
- Lets a human reviewer override or accept any AI suggestion before
  the asset is marked publish-ready

## Stack

- **Frontend:** React
- **Backend:** FastAPI
- **AI:** Ollama (local LLM), OpenCV (vision pipelines)
- **Storage:** Immich's underlying media + Postgres + the project's
  own metadata layer
- **Deploy:** self-hosted

## Status

In production for the personal Out & About library. The "Core" in the
name acknowledges that the per-customer differentiation lives in the
publish-target adapters, not in the review/edit surface itself.

## What this folder will eventually contain

A worked-through architecture, the AI-prompted method as it played
out for this codebase (the patterns are different from the LiDAR
work — fewer one-shot pipelines, more long-lived UI state), and the
decisions around local-LLM vs. cloud, Immich vs. roll-your-own
storage, and where the AI handoff surface to a human reviewer should
sit.

For now: the [Idaho admin folder](../idaho-admin-lidar/README.md) is
the reference example of what "filled in" looks like.
