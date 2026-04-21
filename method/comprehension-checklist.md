# comprehension checklist

The operational definition of "I reviewed every diff" — what that
phrase has to mean if it's going to mean anything.

When the [About page](https://inflectionpt.io/about) says *"I drove
the architecture, made the judgement calls, and reviewed every diff;
the agent wrote every line"*, this is what's being claimed. If a
project shipped without satisfying these, the claim doesn't apply.

## Per-diff checks (every accepted change)

- [ ] **Read it.** Not "scrolled through it." Read every changed line.
- [ ] **Know why each block is there.** If the agent added a function
      I can't justify, the diff goes back. "It works" is not a
      justification.
- [ ] **Know what it would break if I removed it.** Defensive code,
      try/except blocks, edge case handling — all need a justifiable
      "this catches X" answer.
- [ ] **Know what it depends on.** New import? I should be able to say
      what that library does and why it was picked over alternatives.
- [ ] **Spot the silent defaults.** Especially for ML and infrastructure
      code: dtype defaults (the `torch_dtype=float16` conv-bias bug),
      region defaults, retry defaults, timeout defaults. The bugs that
      bite in production almost always live in defaults nobody read.

## Per-feature checks (every shipped capability)

- [ ] **Can I explain it without the source open.** If I can't tell
      someone what step 16 does end-to-end without re-reading the
      file, I don't actually understand step 16.
- [ ] **Can I predict its output.** Given a sample input, can I write
      down what I expect on the output side before running it? If
      consistently yes, comprehension is real. If consistently no,
      I'm running an experiment, not shipping a feature.
- [ ] **Can I list at least one way it can break.** A feature with
      no failure modes is a feature I haven't thought hard enough
      about. Every entry in the [`failure-log`](failure-log.md) was
      something I could have predicted in advance if I'd asked this
      question; I want to ask it on purpose, not retroactively.

## Per-project checks (every project that ships)

- [ ] **The architecture diagram is in my head, not just in a file.**
      I should be able to draw it on a whiteboard from memory. If I
      can't, I have a documented architecture but not a comprehended
      one.
- [ ] **I know the dependency stack.** Not the names — the *reasons*.
      Why podman? Why Astro? Why CSF for ground? Each has an answer
      that isn't "the agent suggested it" or "Stack Overflow said so."
- [ ] **The decisions document is real.** Every entry in
      `decisions.md` is a call I made, not a call the agent made and
      I retroactively documented. The difference shows: real
      decisions list the alternatives I considered; retroactive ones
      don't.
- [ ] **I can answer "what would you do differently next time."**
      A project I can't critique is a project I haven't comprehended.
- [ ] **The README reconciles against the infrastructure.** For any
      project that ships, the high-level README's claims about hosting,
      deploy mechanism, runtime, and form/back-end handlers must be
      cross-checked against the actual infrastructure code (Terraform,
      CDK, IaC scripts, `lambda/`, `infrastructure/`, whatever exists).
      Re-run this check every time the deploy target moves.
- [ ] **The IaC and the cloud account agree on what's deployed.** It
      is not enough that `infrastructure/` and `lambda/` describe a
      complete system. The cloud account itself has to actually
      contain the resources those directories describe. `aws lambda
      list-functions`, `aws s3 ls`, `aws cloudfront list-distributions`,
      `aws ses list-identities` (or the equivalent for whatever cloud
      and IaC tool the project uses) is a one-minute check that the
      IaC isn't aspirational. Skipping it is how a `lambda/README.md`
      ends up describing functions that have never existed.
- [ ] **Every user-visible interaction has been tested end-to-end
      against production.** Not "the back-end is deployed," not "the
      form renders," not "it works in `npm run dev`." The submit
      button on the live site, with a real POST, returning the real
      success page. For any non-GET interaction (form submits, API
      calls, webhook receivers), `curl` against the live URL is part
      of accepting the work. The three items above were added after the
      [Cloudflare-Pages-vs-AWS-with-no-back-end-at-all](failure-log.md#public-readme-claimed-cloudflare-pages-the-contact-form-has-actually-never-had-a-working-back-end-at-all)
      entry, where the docs claimed one architecture, the
      infrastructure code claimed a second, the cloud account
      contained a third (much smaller) reality, and the live site
      silently 404'd. The three items chain: docs reconcile against
      IaC; IaC reconciles against the cloud account; cloud account
      reconciles against live behavior. Skip any link and the chain
      breaks silently.

## Per-week checks (the ongoing practice)

- [ ] **At least one thing got abandoned this week.** A practice that
      ships everything it tries is a practice that hasn't tested
      enough. The splat path and the mesh path were both correctly
      abandoned; that's a sign the practice is working.
- [ ] **At least one bug got caught by me before it shipped.** A
      practice where the agent catches all bugs and the operator
      catches none is one wrong dependency upgrade away from a silent
      regression in production. The fp16 conv-bias bug, the multi-word
      ADE name tokenization bug, the missing-instances STPLS3D
      crash — all caught by the operator, not by the agent. That ratio
      matters.
- [ ] **The failure log got updated.** New entries when something
      broke; new entries when something *almost* broke. A flat
      failure log isn't a low-failure week, it's a low-honesty week.

## What this checklist isn't

It isn't a CI gate. There's no script that runs it. The point of
making it explicit is that *I* can run it on a per-diff or per-week
basis and notice when I'm slipping. The slipping is the failure mode
this exists to prevent — not the diffs themselves.

## What slipping looks like

- "I'll read it more carefully later" (later doesn't come)
- "It works, ship it" (works ≠ understood)
- "The agent already explained it in the comments" (the comments
  echo what the code does, not why; if the why is only in comments,
  it's not actually in my head)
- "Tests pass" (tests pass *in the cases that were tested*; the bug
  lives in the case nobody wrote a test for)

## What "passing the checklist" doesn't claim

It does not claim the work is bug-free. It does not claim the
operator could have written the same code by hand. It does not claim
the architecture is optimal. It claims one specific, narrower thing:
the operator understands, in operationally useful detail, what the
shipped system is doing and why. That understanding is the thing the
practice is selling.
