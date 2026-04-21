# PM knowledge-base workspace

A working knowledge-base workspace used daily for an enterprise product
management role. Not a shipped product, not a client deliverable — the
system the operator uses to keep a complex, multi-surface PM job
coherent across sessions, environments, and weeks.

- **Source repo:** private (and more strictly so than the others —
  see [boundaries](#boundaries) below)

## Boundaries

The employer, the product being PM'd, the pod, the competitive
context, the current quarter's commitments, the customer list, the
meeting participants, and any specific strategy content are
intentionally omitted from this write-up. Including the `graph.yaml`
itself would leak strategy structure, so it also stays out.

What follows is the **functional architecture** — the patterns, the
agent workflows, the data model shape, the scripts and services and
rules that make the workspace work as a system. That architecture is
the reusable artifact; the content that lives inside it is not.

## Why it's in the lab

Of the six projects documented here, this one is the most directly
an answer to "what does comprehension at scale actually look like when
an agent is writing most of the code." It isn't a product that gets
shipped; it's the operator's pattern-recognition apparatus, built out
incrementally, and used every day to catch things that would otherwise
slip.

It's also the most generalizable pattern. The other five projects are
domain-specific (LiDAR, DAM, geospatial ML, personal gallery,
marketing site). The workspace architecture below is a transferable
shape for "how one person running a complex knowledge-work job
coordinates with agents across multiple tools and environments."

## The shape, top-level

~400 files across eight areas:

- **`.cursor/rules/`** — seven always-loaded `.mdc` rule files that
  every session inherits. Each rule scopes a different concern:
  current-state pointers, architecture facts, file-layout
  conventions, agent session lifecycle, a failure taxonomy (see
  [PM Intelligence Stack](#pm-intelligence-stack)), prose
  conventions, and skill routing.
- **`reference/`** — institutional memory. Long-lived knowledge that
  changes slowly (primers, research, roadmaps, product-context
  docs, meeting notes).
- **`planning/`** — the working layer, organized by
  `[year]-Q[n]/pods/[pod]/`. Current quarter's rocks, scorecards,
  annual goals, competitive analysis, positioning brief, session
  notes, and spec directories.
- **`skills/`** — eight skill packages. Each is an agent capability
  with a defined trigger, reads a specific set of inputs, writes
  a specific set of outputs.
- **`services/kb-api/`** — a local FastAPI + SQLite service that
  mirrors the knowledge graph and filesystem into a queryable DB.
- **`scripts/`** — derivation utilities (manifest generation,
  reference-index generation, cowork-instructions generation,
  graph-navigation benchmarks, health checks).
- **`.claude/`** — gitignored. Per-machine permission allowlists and
  scheduled task references.
- **Root files** — `CLAUDE.md` (workspace config), `graph.yaml`
  (the knowledge graph), `MANIFEST.md` (auto-generated file
  inventory).

## Knowledge graph

The distinctive architectural choice: a single `graph.yaml` at the
workspace root is the primary navigation and dependency model.
Everything else derives from or defers to it.

**Five layers** — the graph has a typed five-layer structure rather
than a flat edge list:

1. **Content** — topics (conceptual entities) and their artifacts
   (files with roles: spec, ticket, evidence, brief, summary).
   Topics group files by *what's being worked on*.
2. **Governance** — authority files (a small explicit list, on the
   order of half a dozen) and propagation rules. Which files are
   authoritative, what confidence level they carry, what needs
   updating when they change. This layer replaced per-file
   frontmatter (`source_of_truth`, `downstream`, `derived_from`
   fields scattered across ~40 files) with one declarative source.
3. **Activation** — workstreams (quarterly commitments) that group
   topics by *purpose* rather than by ownership. A topic can be in
   multiple workstreams; the purpose axis is the one that answers
   "which topics serve which goal."
4. **Outcome** — evidence entries tagged with which topics they
   touch, plus claims with evidence classes (verified, supported,
   asserted) and upgrade conditions.
5. **Learning** — insights derived from evidence that cross
   multiple topics and change how the operator thinks about them.

**Two traversal axes.** A question about a specific feature enters
the graph through `topics`. A question about purpose or capability
enters through `claims` or `workstreams`. Both axes reach the same
artifacts via different paths; the routing rules specify which axis
is correct for which question shape. This matters because it makes
"what does the platform enable" a different traversal than "what are
we working on" — they can return different sets.

**Artifact inference.** Most artifacts don't need to be declared in
the graph. A file in `specs/{topic_id}/` automatically belongs to
that topic, with its role inferred from filename (`spec-v1.md`
becomes role `spec`, `feature-ticket.md` becomes role `ticket`).
Only cross-cutting artifacts that can't be inferred from path get
explicit declarations. This keeps the graph small enough to hold in
a human's head while staying complete.

**Benchmark tests.** A separate script runs 31 deterministic
navigation queries across 11 categories (topic lookup, hierarchy
traversal, spec folder convention, authority propagation, evidence
resolution, claim tracing, workstream membership, relationship
traversal, foundation indexing, full multi-step paths, insight
integrity). Unlike usage logging, the tests are instant,
repeatable, and don't depend on agent compliance. They run after
every graph edit and as part of the weekly health check.

## Skills

Eight skill packages in `skills/`, each with a trigger set and a
clear input/output contract.

- **Spec workflow.** Multi-agent pipeline. Response agent researches
  context, interviewer runs a six-phase structured interview that
  pushes for precision, spec writer transforms the interview into a
  structured document, product critic scores the spec against
  quality gates, delivery critic scores for buildability across six
  dimensions. Interview state persists so sessions can resume.
- **Strategic planning.** Four coaching modes (goal setting, rock
  drafting, rock critique, scorecard design) that push from
  activity lists to outcome-driven strategy. Techniques like the
  Empty Pod Test and the Competitor Mirror help surface the real
  goal behind the stated one.
- **Blast radius.** Downstream impact analysis. Given a changed
  file, traverse graph authority and propagation chains, topic
  co-membership, and evidence-touches to rank every file that
  likely needs review. Three modes: single-file impact, graph
  validation, full health check.
- **Answer propagation.** Conversational KB updates. The operator
  reports findings in natural language ("here's what happened in
  the meeting"); the skill matches findings to tracked items
  (research agenda unknowns, action items, claims) and proposes
  the exact edits. The operator approves. The operator never
  hand-edits these files.
- **Spec-table reporting.** Scans spec folders, computes maturity
  stages, emits a summary spreadsheet.
- **Spreadsheet formatter.** Two output shapes (lean internal,
  full shared) sourced from the same canonical rocks document.
- **Ticket-reader.** Browser-automation integration that reads
  status metadata from the operator's authenticated session in an
  internal tracking tool and summarizes across a batch.
- **Product-analysis vetting.** Cross-checks competitive/product
  analyses against the evidence index to flag claims that haven't
  been grounded.

Skills live in a single repo alongside the workspace (not submodules).
Routing from a user request to a skill is itself a rule file —
trigger phrases map to a skill, the skill's own `SKILL.md` runs the
rest.

## KB API service

`services/kb-api/` is a local FastAPI + SQLite service that mirrors
`graph.yaml` and the workspace filesystem into a queryable database.

**Why it exists.** Agent-readable graph navigation is fine for single
questions. For higher-frequency operations — coverage analysis,
proposal generation, health checks — reading and parsing a 1,000-line
YAML per query is slow and wastes context. The service loads the
graph once into SQLite and answers via REST in milliseconds.

**What it exposes.**

- `/sync` — reload graph + filesystem into the DB. A post-commit git
  hook fires this automatically when the server is up.
- `/context` — compact session briefing for agent startup. Coverage
  stats, pending proposal count, graph summary, last sync time in a
  single response. Order-of-magnitude smaller than reading the raw
  planning files.
- `/coverage` — files tracked (explicit in graph), inferred (spec
  folder convention), or orphaned (unreachable from graph). Orphans
  are viewable grouped by directory.
- `/proposals` — structural suggestions for graph additions.
  Detects orphaned spec folders without topic mappings, reference
  files not in the foundation index, meeting notes without evidence
  entries, files whose paths suggest topic associations. Each
  proposal has approve/reject/defer state.
- `/health` — validation, staleness, and integrity checks.
- `/topics/{id}` — full topic detail with artifacts, children,
  relationships, workstream memberships, and claims.
- `/blast-radius/{path}` — downstream-impact lookup.

**Stack:** Python, FastAPI, SQLite (stdlib), PyYAML, uvicorn. No
other dependencies. The same code runs locally under a dev server
or deploys to any container host — the architecture doesn't care.

## PM Intelligence Stack

A five-level failure taxonomy that structures what the daily and
weekly review routines look for. Each level catches a different class
of thing that goes wrong in a complex knowledge-work job.

| Level | Class | What it catches |
|-------|-------|-----------------|
| L1 | Execution Risk | Deadline, blocker, or dependency gap |
| L2 | Plan Drift | Inconsistency between commitments and metrics, stale work, coverage gap |
| L3 | Coherence Gap | A strategic asset orphaned, contradictory, or stale-but-referenced |
| L4 | Decision Blind Spot | An unknown that would change a current decision if answered |
| L5 | Narrative Liability | A claim asserted externally without verified proof |

Each level has a canonical artifact. A research agenda tracks L4
unknowns; a positioning brief tracks L5 claims with evidence classes;
the weekly coherence block in the session-continuity file tracks L3.

**Top 3 rule.** The daily planner doesn't just list everything
open — it forces a ranked top three with one item per band
(Execution L1/L2, Coherence L3, Intelligence-Narrative L4/L5). A
plan that's all-L1 means the week is about to lose coherence. A
plan that's all-L5 means nothing will ship. The rule is an
opinionated constraint that surfaces the balance problem.

This taxonomy is the most transferable part of the whole workspace —
it works for any knowledge-work role where execution and strategic
coherence both have to be maintained, not just PM work.

## Three-environment workflow

The workspace is edited from three places by the same operator:

- **Cursor IDE** — interactive sessions. Rules auto-inject.
- **Claude Code (terminal)** — scripting, git operations, background
  tasks. Reads rules via `CLAUDE.md` pointers.
- **Claude Cowork (desktop)** — scheduled tasks, autonomous
  multi-step work, file organization. Reads both `CLAUDE.md` and
  per-project instructions derived from the same rule files.

**Divergence prevention.** All three environments commit to `main`
only. Side branches are forbidden at the practice level (not
technically enforced — a discipline). On session start, each
environment runs `git branch --no-merged main` and `git status --short`
to detect uncommitted work from one of the others and flag it to the
operator before starting new work.

**Single-source derivation.** The rule files and the graph are
canonical. The Cowork Project Instructions are generated from them
by a script, not maintained separately. The reference-folder README
is generated from the graph's foundation index, not hand-edited. This
is the same discipline as the rest of the lab: there is only one copy
of every fact, and the rest are derived.

## Spec pipeline

Specs move through a directory per feature with a defined progression:

```
pre-spec.md              early exploration
   ↓
spec-v1.md               full spec from the interviewer agent
gaps.md                  issues surfaced during spec writing
   ↓
review-v1.md             product critic's scored review
dm-review-v1.md          delivery critic's buildability review
   ↓
approved-requirements.md stakeholder-facing version
feature-ticket.md        engineering-ready
```

Not every spec has all stages. Maturity stage is computed (not
declared) by scanning filenames in the folder. The `spec-table`
skill produces a rollup across all specs in the current quarter.

## Scripts

Derivation and validation utilities, each enforcing a single-source
invariant:

- **Manifest generator** — scans the tree, writes a typed inventory
  (`MANIFEST.md`) with status columns and spec maturity stages.
- **Reference-index generator** — reads the graph's foundation
  section, writes `reference/README.md` as a typed catalog. Run
  when a file is added to `reference/`.
- **Cowork-instructions generator** — reads the rule files and the
  graph, emits a self-contained instructions block for the Cowork
  project. Run when rules change or the quarter rolls over.
- **Graph navigation tests** — 31 queries across 11 categories;
  each failure reports the specific structural gap with a "why"
  explanation.
- **Health check** — composes the above into a single pass/fail
  report.

## Session lifecycle

Every session follows the same protocol across all three
environments:

**Start:**
1. Rules auto-load (Cursor) or load via pointers (Claude Code,
   Cowork).
2. Read the session-continuity file for what's in flight.
3. Check git status + unmerged branches.

**During:**
- After editing any authority file, run blast-radius.
- After editing the graph, run validation + nav tests.
- After meetings or findings, use answer-propagation to update
  the KB rather than hand-editing.

**End:**
1. Write the session-continuity file (what changed, what's in
   progress, what's next; under ~30 lines).
2. Suggest a commit if meaningful changes happened.

## What this is evidence of

The workspace is the operator's comprehension layer made into a
system. Every pattern above exists because the operator noticed,
at some point, that without it a specific class of thing would be
missed — an L4 unknown would age into an L5 liability, a source-of-
truth edit would leave seven dependent files stale, a rock would
drift from its scorecard without triggering a review. Each piece of
the architecture is a generalized answer to "this almost got past
me once."

The [`method/comprehension-checklist.md`](../../method/comprehension-checklist.md)
in this lab is an abstract version of the same claim. This folder is
the concrete, lived version of it — the one that shows what the
abstract claim looks like when it's implemented as a working system
over several months.

## What this isn't documenting

- The specific product, its features, or its roadmap
- The employer, competitive context, or strategic positioning
- Customer names, meeting participants, or internal people
- Current-quarter rocks, goals, or commitments
- Any content inside `graph.yaml`

Those remain in the private source, where they belong.
