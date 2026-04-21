# 003 — Private source, public method

**Status:** active for the current state of the practice
**Date:** 2026-04 (the decision this repo's existence implements)

## What

The source code for the five projects listed in this lab repo
(`inflectionpt.io`, the LiDAR pipeline under
`infrastructure/ml/`, DAM, NatureNet DataHub, Out & About) lives in
private GitHub repos. The architecture, decisions, pipeline reasoning,
curated commit messages, and method documentation live here, in this
public lab repo.

This lab repo *is* the implementation of the decision.

## Why not fully public

Three reasons, in decreasing order of weight:

1. **Some of it is client-adjacent.** DAM and NatureNet DataHub were
   built for / with specific partner organizations. Going public
   without a conversation with them would be a unilateral decision
   on something that should be a joint one. Default-private until
   that conversation has happened is the lower-regret default.

2. **The infra scripts contain identifiers that aren't credentials but
   aren't *nothing* either.** AWS account IDs, distribution IDs,
   bucket names, region pinnings — none of these are secrets in the
   "would compromise the system" sense, but publishing them does
   mean anyone scanning the public source can target the live
   infrastructure with curiosity attacks. Cheap to avoid by keeping
   that one repo private.

3. **Open source has a maintenance bill that this isn't ready to pay.**
   A public repo with the source produces drive-by issues, drive-by
   PRs, drive-by "why don't you do X" — most of which deserve a real
   answer the project doesn't have time to give. The lab repo (this
   one) gets to be a publish-and-walk surface; the source repos, if
   public, would not.

## Why not fully private

The mirror image: the [About page on the live site](https://inflectionpt.io/about)
makes a strong public claim — *"DAM Core, NatureNet DataHub, Out &
About with Jim, this advisory site, and the LiDAR classification
pipeline behind the Idaho admin work page were all built end to end
through AI prompting"*. A claim that strong with zero public surface
to verify against starts to fail the smell test. The advice in
circulation right now is largely correct: *production without
comprehension is a liability,* and *working in the open creates
accountability that closed-door development cannot.* A fully-private
posture makes the comprehension claim un-testable from the outside.

## What the lab repo solves for

It threads the gap. Specifically:

- **Architecture is public.** A reader can evaluate "does this person
  know what they're building" without needing the source.
- **Decisions are public.** Every non-obvious call has a recorded
  reason. A reader can spot-check whether the reasoning is real or
  retroactive.
- **Commit messages are public.** Curated extracts in
  [`projects/*/commit-log.md`](../projects/idaho-admin-lidar/commit-log.md)
  expose the operator's working notes without exposing the source they
  describe.
- **Failures are public.** The [failure log](../method/failure-log.md)
  is where the practice would be most embarrassed to be caught lying;
  publishing it is the cheapest way to commit to not lying.
- **Source stays private** until there's a specific reason to open a
  specific repo (a client engagement, a decision to build a library,
  etc.).

## What it doesn't solve for

- A reader who wants to compile and run the pipeline still can't.
  The regen path documented in the private repo's
  `infrastructure/ml/README.md` is enough for someone with the same
  scan and the same hardware to reproduce — but they need the source
  to follow it. That's a real limit. Mitigation: the
  [`pipeline.md`](../projects/idaho-admin-lidar/pipeline.md) is
  detailed enough that someone reimplementing it from scratch could
  follow the same architecture; it just won't be a copy-paste run.

- A reader who wants to verify the commit messages against the actual
  diffs can't. The curation step is a trust point. Mitigation: if and
  when a specific repo opens up, the curated `commit-log.md` becomes
  spot-checkable against `git log`; in the meantime, the curation is
  what it is.

## Revisit when

- Any one of the source repos has a client-side green light to open.
  The lab repo doesn't disappear in that case — it becomes the
  reading-order guide *into* the source.
- The lab repo grows substantial enough that "show me your code"
  questions stop arriving and "show me your reasoning" becomes the
  default ask. That's the signal the public-method posture is
  working.
- A specific repo turns into something library-shaped (reusable,
  not project-shaped). The library shape is the natural inflection
  point for "open source it on its own merits."

## How to read this repo if you're evaluating the practice

1. Start with the top-level [`README.md`](../README.md) for the deal.
2. Read [`projects/idaho-admin-lidar/`](../projects/idaho-admin-lidar/)
   in order: `README` → `architecture` → `pipeline` → `decisions` →
   `commit-log`. That's the worked example.
3. Read the three decisions in [`decisions/`](.) for the cross-cutting
   posture.
4. Read [`method/failure-log.md`](../method/failure-log.md) for the
   honest version of how this all really plays out.
5. Compare to the live work at <https://inflectionpt.io/work/idaho-admin/>
   and <https://inflectionpt.io/writing/>. The lab repo is the
   evidence layer behind those.
