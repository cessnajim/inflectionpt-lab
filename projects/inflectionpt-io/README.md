# inflectionpt.io

The advisory site itself — pages, content, contact + subscribe
plumbing, and the embedded scan viewers that surface the
[Idaho admin LiDAR work](../idaho-admin-lidar/README.md). Built end to
end through AI prompting in a Cursor session against Claude.

- **Live site:** <https://inflectionpt.io>
- **Source repo:** private (`cessnajim/inflectionpt.io`)

## What's there

- About / services / engagement pages
- A writing index (long-form articles + a separate playbook section)
- A `/work/` index with `/work/idaho-admin/` as the first deep work page
- Contact form + newsletter subscribe whose Lambda back-ends are
  written and deployed (`lambda/{contact,subscribe}/index.mjs`,
  Function URLs, Amazon SES, optional Buttondown) but **not currently
  wired to the live form components** — the form `<form action>`
  attributes still point at same-origin `/api/*` paths that have no
  CloudFront behavior in front of them, so submissions 404 in
  production. This regressed silently when the deploy moved from the
  earlier Cloudflare Pages target to AWS S3 + CloudFront and was
  caught while doing the docs cleanup that produced this folder. The
  fix is small (point the form `action`s at the Lambda Function URLs
  and delete the `functions/api/*.ts` Cloudflare Pages scaffolding) and
  is described in the source repo's README under "Form wiring."
- Embedded Potree viewers for the Idaho admin scan + classified scan

## Stack

- **Site:** Astro (static-first, content-driven)
- **Styling:** Tailwind, with a custom typographic scale + color
  tokens defined as CSS custom properties
- **Forms:** AWS Lambda functions with Lambda Function URLs
  (`lambda/{contact,subscribe}/index.mjs`). No API Gateway, no Pages
  Functions middleman. (Currently disconnected from the live form
  components — see "What's there" above.)
- **Email:** Amazon SES (verified `inflectionpt.io` identity in
  `us-east-1`); optional Buttondown for the subscribe Lambda.
- **Hosting:** AWS S3 (`inflection-point-advisory-site`) fronted by
  CloudFront (`E3EQIEAXLDNIFA`, aliases `inflectionpt.io` /
  `www.inflectionpt.io`). Deploy is **manual** — `npm run build` then
  `aws s3 sync dist/ s3://inflection-point-advisory-site/ --delete`
  followed by a CloudFront invalidation. There is no auto-deploy from
  `main`; pushing to GitHub does not ship the site.
- **CDN-fronted assets:** the same CloudFront distribution serves
  COPC point clouds for the work-page viewers via a separate `/scans/*`
  behavior pointing at `inflection-point-advisory-scans` over OAC
  (provisioned by the AWS scripts documented in
  [`../idaho-admin-lidar/architecture.md`](../idaho-admin-lidar/architecture.md))

## Why this folder exists in the lab repo

The site is one of the six "100% prompted" projects called out on
the live About page. Including it here keeps the count honest — the
list isn't five production tools and a hand-coded marketing site,
it's six end-to-end-prompted things, including the surface that's
making the claim.

## What this folder will eventually contain

The Astro content architecture (collections, schema, dynamic routes),
the CSS token system, the choice of S3 + CloudFront over a managed
hosting platform (Vercel, Netlify, Cloudflare Pages) — driven by the
fact that the COPC point-cloud delivery was already going to live on
CloudFront via OAC for the LiDAR work, so unifying the marketing
origin onto the same distribution was the cheaper path —
the form-handling architecture (Lambda Function URLs + SES instead of
a third-party form service or Pages Functions), and the prompt
patterns that worked for "build a real marketing site" vs. the more
pipeline-shaped prompts from the LiDAR work.

It will also document the comprehension failure logged in
[`../../method/failure-log.md`](../../method/failure-log.md) where the
public README and curated commit messages claimed Cloudflare Pages
hosting for several weeks while the site was actually on S3 +
CloudFront — a cheap-to-make documentation drift that the comprehension
checklist is supposed to catch and didn't.
