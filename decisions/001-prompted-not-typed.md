# 001 — Prompted, not typed

**Status:** active practice
**Date:** 2026-04 (formalized after the Idaho admin pipeline ship)

## What

Every line of code in the five projects listed in this lab repo —
the LiDAR pipeline, DAM Core, NatureNet DataHub, Out & About with
Jim, and inflectionpt.io itself — was written by an AI agent in
response to prompts in a Cursor session. No code was typed by hand.
The human role across every project was: architecture, prompts, diff
review, judgement on what to ship and what to throw away.

## Why this decision is worth recording

There are two confident positions on AI-generated code in mid-2026
and they're both wrong:

1. *"Prompted code doesn't count, it's just stochastic completion of
   what someone else already wrote."* This collapses immediately when
   the operator can explain why each piece exists, what it cost to
   get right, and which paths were tried and discarded. The LiDAR
   pipeline's [decisions](../projects/idaho-admin-lidar/decisions.md)
   document exists specifically to make that explanation auditable.

2. *"AI lets you ship anything, you just iterate."* This collapses
   the moment the model is confidently wrong about something the
   operator can't independently evaluate. The fp16 conv-bias bug in
   `13_oneformer_predict.py` (silent: training-time defaults flow
   through the model and break inference in a way that doesn't throw)
   was caught because the operator knew the failure shape, not
   because the agent flagged it.

The honest version is somewhere in between: the agent handles the
keystrokes; the operator handles the comprehension, the architecture,
and the "is this actually working or is it just running" judgement.
That's the practice this repo documents.

## How it shows up in the work

- The agent produces 60-line commit messages because the operator
  reviews them and edits them into explanation artifacts (see
  [`commit-log.md`](../projects/idaho-admin-lidar/commit-log.md))
- The agent doesn't volunteer that "your splat will look like a needle
  blob because pano-only capture lacks parallax" — the operator
  recognizes the failure mode after a session or two and calls the
  abandonment (see splat decision in `decisions.md`)
- The agent will happily run `pip install --upgrade` on packages
  that will break Mask3D — the Dockerfile pins exist because the
  operator knows the dependency stack is fragile and the agent
  doesn't have priors on which version bumps are safe
- Architecture choices (full-scene tiling vs. crop, image fusion via
  z-buffer vs. via NeRF, three containers vs. one) are operator
  decisions; the agent implements them but doesn't pick between them

## How it relates to the public discourse

A common piece of advice in 2026 reads roughly: *"Workers who keep
chasing output volume are missing that taste comes from understanding
enough things deeply enough to recognize patterns — and that's the
rare commodity now. One project you fully comprehend teaches more
than ten you vibe coded."*

That's right. The risk of the prompted-not-typed practice is exactly
the failure mode it warns about: shipping ten things you don't
understand instead of one thing you do. Mitigations:

- **Five projects, four placeholder READMEs.** Listing all five with
  the gap visible (only the LiDAR work is deeply documented here yet)
  is more honest than pretending we've documented five.
- **Explanation artifacts as the deliverable.** The lab repo exists
  because the comprehension claim is testable: read the architecture,
  decisions, and commit log; if you can't reverse-engineer "this
  operator knows what's going on" from those, the claim is broken.
- **Failure log.** [`method/failure-log.md`](../method/failure-log.md)
  enumerates things that broke, were caught late, or were caught by
  the operator after the agent missed them. A practice with no
  failure log is hiding something.

## How it relates to the live site

The [About page](https://inflectionpt.io/about) calls this out
explicitly under the "Method" heading: *"100% prompted. DAM Core,
NatureNet DataHub, Out & About with Jim, this advisory site, and the
LiDAR classification pipeline behind the Idaho admin work page were
all built end to end through AI prompting. No code was typed by
hand."* The live site says it; this lab repo backs it up.

## What this is not

It is not "the agent did it, I just clicked accept." It's also not
"I would have done the same thing typing it myself, just slower."
The work the operator does is genuinely different from what they'd do
typing — more architecture, less syntax; more "is this actually
working" verification; more diff review; more "abandon this and try
the other thing." That different kind of work is what the practice
is, and what the lab repo is the evidence of.

## Revisit when

- The operator catches themselves accepting a diff they don't
  understand "because it works." That's the failure mode this
  decision exists to prevent.
- A project ships that the operator can't write a `decisions.md`
  for. Same thing in a different shape.
- A regression goes uncaught for more than one cycle. Caught
  regressions are fine; *uncaught* ones are the signal that the
  comprehension layer is thinning.
