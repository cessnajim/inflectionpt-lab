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

## Public README claimed Cloudflare Pages; the contact form has actually never had a working back-end at all

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

This was logged in three escalating rounds during a single
docs-cleanup pass, and each round corrected a wronger version of the
last:

**Round 1 (documentation drift).** The wording in the docs and commit
message was wrong; the *implementation* was assumed to be on AWS as
`infrastructure/README.md` claimed. Logged as "fix the words."

**Round 2 (silent production breakage).** A direct `curl` against
`https://inflectionpt.io/api/contact` returned CloudFront 403 for
POSTs and S3 404 for GETs. The contact form and newsletter signup
have been broken in production the entire time the AWS deploy has
been live. Initially logged as: "the form components POST to
same-origin `/api/*` paths that have no CloudFront behavior in front
of them, but the Lambdas in `lambda/` are deployed and just
disconnected — fix is to point the `action`s at the existing Lambda
Function URLs."

**Round 3 (the Lambdas don't exist either).** `aws lambda
list-functions` against `us-east-1`, `us-east-2`, `us-west-1`, and
`us-west-2` returned no `inflection*` / `contact` / `subscribe`
matches. The `lambda/` directory is source code that was never
deployed. There is no Lambda runtime to point form `action`s at; there
are no Function URLs to capture. The `inflectionpt.io` SES identity
*is* verified in `us-east-1` (so the email-send side is half-set-up),
but no Lambda is configured to use it. The contact form has not had a
working back-end *of any kind* since the site went live: not a
Cloudflare Pages Function (the site isn't on Cloudflare Pages), not a
Lambda (the Lambdas aren't deployed), not anything else. Pure 404.

The wrong-deploy-target docs perfectly hid all of this, because the
docs claimed an architecture (Cloudflare Pages Functions on the same
origin as the site) under which the existing form `action="/api/*"`
attributes would have actually worked. Anyone reading the README would
have correctly concluded "this looks fine."

The contradiction was sitting in the open the entire time: the site's
own `infrastructure/README.md` documents the CloudFront → S3 pipeline
in detail (including the distribution ID and the IaC-lite shell scripts
that provisioned it). `lambda/README.md` claims the Lambdas are
deployed and explains the SES + Function URL design. AWS itself, when
asked, said the Lambdas don't exist. No one — agent or operator — had
cross-read the high-level README against the infrastructure
subdirectories, no one had probed the live form endpoints, and no one
had asked AWS what was actually running.

**How surfaced:** operator, while reviewing the lab repo and the live
About page, asked in sequence:

1. "You need to review where this gets deployed and change any docs
   that are wrong." → surfaced the documentation drift (round 1).
2. "Let's clean up the docs to reflect reality." → forced a `curl`
   against production, which surfaced the silent 404 (round 2).
3. "Search for the AWS — you've done the AWS deploy." → forced an
   `aws lambda list-functions` query, which surfaced that the Lambdas
   don't exist at all (round 3).

Each prompt was one or two sentences. Each one peeled back another
layer of "what does this actually do." The agent could have asked any
of these questions itself at any point in the last several weeks and
didn't.

**Caught by:** operator, three times in three rounds in a single
session.

**Fix:**

- *Documentation* (done in this cleanup pass): rewrite the Stack /
  Build / Deploy / Form-wiring sections of the site README, the
  Stack section of `projects/inflectionpt-io/README.md` in the lab,
  the curated `f8178b6` entry in the commit log, and the About page's
  tech stack list. The site README's "Form wiring" section now
  describes the actual three-layer absence (no Pages runtime, no
  Lambda runtime, no `/api/*` CloudFront behavior) and lists the four
  ordered steps required to make the form work.
- *Live forms* (not in this pass): deploy the two Lambdas as Function
  URLs in `us-east-1` against the verified SES identity, add an
  idempotent `infrastructure/scripts/05-deploy-form-lambdas.sh`,
  rewire the form `action`s to the captured Function URLs, delete the
  `functions/` directory, and probe with `curl` against production
  to confirm 303s back to `/thanks/` and `/subscribed/`.

**Lesson:** the same gap, hit three times in one hour:

1. Docs do not reconcile against the infrastructure code. (Caught by
   round 1.)
2. Infrastructure code does not reconcile against the live cloud
   resources. (Caught by round 3 — `lambda/README.md` claimed
   deployment that hadn't happened.)
3. Neither of the above is the same as "the live behavior was
   tested." (Caught by round 2.)

All three need to be checked, in that order, before claiming a feature
works. The [comprehension-checklist](comprehension-checklist.md) now
has items for the first and third; the second is implied by the third
and probably should be split out as its own item ("the IaC and the
cloud account agree on what's actually deployed").

The meta-lesson is the one this failure log is most useful for: an AI
agent will happily generate a README that says a thing is deployed,
will happily generate a `lambda/README.md` that says a thing is
deployed, will happily generate form components that POST to a path
the README implies will route, and will keep agreeing with itself
across all three artifacts forever. None of those three artifacts is
the actual cloud account. The cloud account is the only ground truth.
Asking "what's actually in the account" is not something an AI agent
spontaneously does — it has to be prompted, and on a single project
it had to be prompted three times in a row to chase the drift down to
the real bottom.

---

## What this log says about the practice

Seven entries across two projects. None of them were caught by the AI
agent unprompted; all of them were caught by the operator's recognition
of "this output doesn't match what I expect" — including the most
recent one, which started as "the documentation doesn't match the
infrastructure," became "the production behavior doesn't match the
documentation," and finally became "the cloud account doesn't match
the infrastructure code." That progression — three rounds of the same
question peeling back a deeper layer of wrong each time — is the
clearest example in this log of why the comprehension layer is the
work, not the code generation. Each round closed in one or two
sentences from the operator. The agent did not initiate any of them.

The pattern across entries: the agent produces something that runs
without throwing an exception (or, in the most recent case, produces
a README that reads plausibly across three artifacts that all agree
with each other and disagree with reality); the operator notices the
output is wrong-shaped or wrong-claimed; the operator diagnoses why;
the operator directs the fix. None of those four steps are skippable,
and none of them are the agent's job.
