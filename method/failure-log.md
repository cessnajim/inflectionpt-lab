# failure log

Things that broke, almost broke, or got caught by the operator after
the agent missed them. This file exists because a practice with no
failure log is hiding something — and because the entries here are
the most concrete evidence of what the comprehension layer actually
does.

Entries are ordered roughly chronologically within the Idaho admin
project. Format: what failed, how it surfaced, who caught it (operator
or agent), what the fix was, what the lesson is.

## fp16 conv-bias mismatch in OneFormer

**What:** `13_oneformer_predict.py` was first written with the model
loaded as `torch_dtype=torch.float16` to save VRAM. Inference produced
nonsense outputs — confidence near zero, predictions looking nothing
like the input.

**How surfaced:** operator looked at the per-pano confidence map and
recognized "this isn't even in the right ballpark." Not because the
agent flagged it; because the operator had a prior on what an ADE20K
output should look like.

**Caught by:** operator.

**Fix:** load the model in `fp32`, wrap the inference call in
`torch.autocast(device_type="cuda", dtype=torch.float16)`. The bias
parameters then stay `float32` while the conv math runs in `float16`.

**Lesson:** dtype defaults in HF model loading are the kind of silent
bug AI agents will let through indefinitely because the code "works"
(no exception thrown). The operator catches it because they know what
the output is supposed to look like.

---

## Confidence calculation: double-softmax

**What:** the same script computed per-pixel confidence using a
double-softmax pattern that produced ~0.02 mean confidence (uselessly
low — useful confidences for ADE20K's 150 classes are in the
0.3–0.9 range).

**How surfaced:** operator looked at the confidence histogram and
saw the distribution was wrong-shaped — too compressed near zero, no
bimodal "sure / unsure" split.

**Caught by:** operator.

**Fix:** the correct form is `(sem_max / sem_sum).clamp(0, 1)` where
`sem_max` is the per-pixel max semantic score and `sem_sum` is the sum
across all 150 ADE classes. No softmax involved.

**Lesson:** "confidence" in segmentation models has multiple
correct-looking formulations and they don't produce comparable
numbers. The one that lets you set a useful threshold is the one
where the histogram has a defensible shape, not the one that compiles
without errors.

---

## Mask3D: NO INSTANCES → RecursionError

**What:** running Mask3D inference on a block that contained zero
predicted instances threw a `RecursionError` with the cryptic message
"NO INSTANCES" and a stack trace deep in Mask3D's STPLS3D export
path.

**How surfaced:** the inference run crashed on block 7 of 19 the
first time it was attempted on the full scene. Worked fine on the
single-building crop because that crop always had instances.

**Caught by:** the agent (the crash was hard to miss). But the
diagnosis — "this is the export path doing something defensive that
breaks on empty blocks" — was operator-driven; the agent's first
instinct was to upgrade Mask3D, which would have broken everything
else.

**Fix:** in `03_to_mask3d.py`, seed at least two dummy instance IDs
into every block's GT before writing the npy. Mask3D's export path
needs the IDs to construct a comparison; we don't care about the
comparison (we're using `general.export=true`), so dummy instances
are fine.

**Lesson:** defensive code in upstream libraries that assumes "of
course there will be instances" is a common shape of bug when running
the library outside the regime it was tested in. The operator's job
is to recognize "this is a tested-regime mismatch, fix it at the
data prep step" rather than "this is a library bug, upgrade it."

---

## Multi-word ADE class names lost in tokenization

**What:** `17_fuse_v5.py`'s ADE → STPLS3D mapping table looked up class
names by tokenizing the ADE label string. The first version split on
whitespace, which silently dropped multi-word classes like
"street lamp", "screen door", "swimming pool" — none of those would
match the lookup table.

**How surfaced:** operator noticed that "lamp" was getting mapped but
"streetlight" wasn't, even though both were in the ADE150 vocabulary.
The mapping table had `"street lamp": <class>` and the tokenizer was
looking for `street` and `lamp` as separate tokens.

**Caught by:** operator, while spot-checking the v5 fusion output
against an aerial view of the site.

**Fix:** split on `;` and `,` only — not on whitespace. ADE class
strings are formatted as `"primary; alt1, alt2"` and within each
slot, multi-word names are intentional.

**Lesson:** when interfacing with vocabulary-style data (class names,
prompts, labels), the agent will default to the most common Python
tokenization pattern (whitespace split), which is wrong for any
controlled vocabulary that has multi-word entries. Always check the
upstream specification.

---

## Splat path: spent too long before abandoning

**What:** the Gaussian splat path (steps 10–12) was iterated on across
several sessions trying to fix needle artifacts before being
abandoned.

**How surfaced:** operator finally asked "is this fixable in
principle, or am I trying to fix the wrong thing." The answer was
the second one — pano-only capture lacks the parallax splatting was
designed for. No filter could have fixed it.

**Caught by:** operator, slowly. This is the failure-mode entry,
not a clean save.

**Fix:** abandoned the path. Kept the scripts for honesty, removed
the splat viewer from the live site.

**Lesson:** the abandonment heuristic in
[`decisions/002`](../decisions/002-when-to-abandon-an-experiment.md)
exists because of this entry. Three sessions of "this iteration is
slightly less needle-y" should have triggered the wrong-tool
diagnosis a session earlier.

---

## Mesh viewer: shipped, then removed

**What:** the per-class Poisson mesh was reconstructed, packaged as
a GLB, served via `<model-viewer>` from `/scans/idaho-admin-mesh-v1.glb`,
and embedded in the `/work/idaho-admin/` page for several days.

**How surfaced:** operator looked at the work page after deployment
and noticed three viewers (raw scan, classified scan, mesh) competed
for attention rather than reinforcing each other.

**Caught by:** operator, post-deploy. Not a bug; a product judgement
that should have been made at architecture time.

**Fix:** removed the mesh section from the page; deleted the mesh
article (which pointed to a viewer that no longer existed); kept the
script and updated the cross-reference in `your-scan-is-actually-two-scans.md`.

**Lesson:** "is this the right *third* viewer on this page" is a
product question the agent doesn't ask. Operator has to.

---

## .gitignore negation didn't work the way I expected

**What:** the first attempt to keep `data/processed/idaho/` files
under git while ignoring everything else under `infrastructure/ml/data/`
used `!infrastructure/ml/data/processed/idaho/`. Git ignored the
included files anyway because the parent directory was already
ignored.

**How surfaced:** `git status` after the gitignore edit showed the
files I expected to be tracked were still untracked.

**Caught by:** operator, before the commit.

**Fix:** rewrite the patterns to ignore `raw/`, `splat/`, and
`processed/*` (not `processed/`), then re-include
`processed/idaho/` and the two specific yaml files.

**Lesson:** git's gitignore semantics around negation patterns and
already-ignored parents are subtle, well-documented, and easy to get
wrong. Every iteration of "this should work, why doesn't it" should
trigger a pause to re-read the actual gitignore docs rather than
guessing at the pattern.

---

## Public README claimed Cloudflare Pages; site is on AWS S3 + CloudFront

**What:** for several weeks the site README, this lab repo's
`projects/inflectionpt-io/README.md`, and the curated entry for commit
`f8178b6` in `projects/idaho-admin-lidar/commit-log.md` all described
the inflectionpt.io deploy as "Cloudflare Pages + Cloudflare Pages
Functions + MailChannels." The original git commit message itself
described the form-handling architecture the same way.

The actual deploy is AWS S3 (`inflection-point-advisory-site`) fronted
by CloudFront (distribution `E3EQIEAXLDNIFA`), with form handlers as
AWS Lambda Function URLs sending via Amazon SES. The Cloudflare Pages
Function files under `functions/api/*.ts` are inert leftover scaffolding
from an earlier attempted hosting choice that was abandoned without
removing the code or updating the docs.

The contradiction was sitting in the open the entire time: the site's
own `infrastructure/README.md` documents the CloudFront → S3 pipeline
in detail, including the distribution ID and the IaC-lite shell scripts
that provisioned it, and `lambda/README.md` documents the SES + Lambda
Function URL form handlers. No one — agent or operator — had cross-read
the high-level README against the infrastructure subdirectories.

**How surfaced:** operator, while reviewing the lab repo and the live
About page after a recent push, asked: "you need to review where this
gets deployed and change any docs that are wrong." That single sentence
forced the cross-check that should have happened on day one.

**Caught by:** operator.

**Fix:** rewrite the Stack / Build / Deploy / Forms sections of the
site README, the Stack section of `projects/inflectionpt-io/README.md`
in the lab, and the curated `f8178b6` entry in the commit log to
describe what actually shipped. Add a curator's note on the commit-log
entry acknowledging that the original git commit message was itself
wrong. Fix the About page's tech stack list (which had named "Cloudflare
Workers" as part of what the agent shipped — also untrue). Decide
separately whether to remove the legacy `functions/` directory from the
source repo; for now the site README flags it explicitly as inert.

**Lesson:** the comprehension checklist had per-diff and per-feature
items but no "where does this actually deploy?" item. That gap let a
high-level README narrate one architecture while the implementation
shipped another, for weeks, in public. The checklist should grow a
"deploy-truth" item: for any project visible in the lab, the README
must reconcile against the infrastructure code (Terraform, IaC scripts,
or whatever exists), and the reconciliation must be redone any time the
deploy target moves. The cost of the reconciliation is minutes; the
cost of the drift is reader trust.

There's a meta-version of this that's worth saying directly: an AI
agent asked to write a README for a project will happily restate
whatever the previous README claimed. If the previous claim was an
aspirational deploy target that never happened, the new README will
inherit the wrong claim with extra confidence. The fix is human
cross-checking against the infrastructure, not better prompts.

---

## What this log says about the practice

Seven entries across two projects. None of them were caught by the AI
agent unprompted; all of them were caught by the operator's recognition
of "this output doesn't match what I expect" — including the most
recent one, which was about the *documentation* not matching what the
infrastructure actually does. That's the comprehension layer doing its
job — and it's also the answer to the question "what's the operator
actually doing if the agent is writing the code." Catching these is
the work.

The pattern across entries: the agent produces something that runs
without throwing an exception (or, in the most recent case, produces
a README that reads plausibly); the operator notices the output is
wrong-shaped or wrong-claimed; the operator diagnoses why; the
operator directs the fix. None of those four steps are skippable, and
none of them are the agent's job.
