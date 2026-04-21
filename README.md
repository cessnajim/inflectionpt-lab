# inflection point lab

Public method, private source. This repo is the explanation layer for
the work shipped under [Inflection Point Advisory](https://inflectionpt.io)
— architecture, decisions, what broke, what got abandoned, and the
reasoning behind the calls.

The actual source for each project lives in private repos. Everything
that proves the work was *comprehended* and not just *generated* —
architecture sketches, decision records, post-mortems, curated commit
messages, the rationale behind individual files — is metadata, not
source. That metadata lives here.

## The deal

Production volume is now free. AI generates code by the megabyte. What
the market increasingly can't tell apart is "shipped a lot of code"
from "shipped a lot of code that the operator actually understands."
The difference shows up the first time something breaks.

This repo exists to make the difference auditable for the work that
happens to be source-private. If you want to know whether the
[`/work/idaho-admin/` LiDAR pipeline](https://inflectionpt.io/work/idaho-admin/)
on the live site really was end-to-end-prompted *and* end-to-end-comprehended,
you should be able to read enough here to form a defensible opinion
without ever seeing the source.

If a future engagement ever wants the source itself opened up, that's a
conversation. The default is private code, public method.

## Layout

```
projects/
  idaho-admin-lidar/      Mask3D + image-fusion classification
                          pipeline behind the live site's
                          /work/idaho-admin/ page.
  dam-core/               AI-assisted editor turning a 50,000-asset
                          Immich library into a working publishing
                          pipeline.
  naturenet-datahub/      Geospatial ML platform on AWS for
                          invasive-species monitoring.
  out-and-about/          Personal photo & video gallery, 140+
                          pieces, indexed on an interactive map.
  inflectionpt-io/        The advisory site itself.
  pm-kb-workspace/        Working knowledge-base workspace used
                          daily for an enterprise PM role.
                          Functional architecture only; the
                          employer, product, and strategy content
                          stay private.
decisions/                ADR-style records for cross-cutting calls
                          (the prompted-not-typed practice, when to
                          abandon an experiment, why source is
                          private but method is public).
method/                   Operational definitions of how the
                          practice works in practice — the
                          comprehension checklist, the failure log.
```

Each `projects/<name>/` folder, where filled in, has roughly:

- `README.md` — what it is, who it's for, current status, links to the live artifact
- `architecture.md` — data flow, container/service layout, dependency map
- `pipeline.md` (or `flow.md`) — the actual sequence of operations the project executes, step by step, with the rationale behind each step
- `decisions.md` — project-specific calls and the reasoning
- `commit-log.md` — curated extracts from the private repo's commit history that read as standalone explanation artifacts

Idaho admin is the worked example. PM KB workspace is the second
filled-in entry, kept deliberately at the functional-architecture
level because its source repo is the most client-sensitive in the
set. The other four are placeholders that get filled in as I write
them up. Listing them with the gap visible is intentional — the
alternative is pretending I've documented six projects when I've
actually documented two.

## What's *not* here

- Source code (lives in private repos)
- Customer data (never leaves the laptop / private S3)
- AWS account IDs, distribution IDs, anything that would let a
  reader poke at the running infrastructure
- Anything that needs to stay confidential for a client engagement

## Status

Just published. Idaho admin is the most worked-through example
(unsurprisingly — it has a corresponding deep work page on the live
site). The rest will fill in as the explanation backlog gets paid down.

## Live work

- Site & writing: <https://inflectionpt.io>
- Most-explained project: <https://inflectionpt.io/work/idaho-admin/>
- Article series: <https://inflectionpt.io/writing/>
- Practice contact: <https://inflectionpt.io/about/>
