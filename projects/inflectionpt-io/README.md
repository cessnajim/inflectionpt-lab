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
- Contact form + newsletter subscribe (Cloudflare Pages functions
  forwarding to AWS Lambda backers — SES for email send, SNS for
  notification)
- Embedded Potree viewers for the Idaho admin scan + classified scan

## Stack

- **Site:** Astro (static-first, content-driven)
- **Styling:** Tailwind, with a custom typographic scale + color
  tokens defined as CSS custom properties
- **Forms:** Cloudflare Pages Functions (`functions/api/*.ts`)
  forwarding to AWS Lambda (`lambda/{contact,subscribe}/index.mjs`)
- **Email/notify:** AWS SES + SNS
- **Hosting:** Cloudflare Pages (auto-deploy from `main`)
- **CDN-fronted assets:** S3 + CloudFront for the COPC point clouds
  served by the work-page viewers (provisioned by the AWS scripts
  documented in [`../idaho-admin-lidar/architecture.md`](../idaho-admin-lidar/architecture.md))

## Why this folder exists in the lab repo

The site is one of the five "100% prompted" projects called out on
the live About page. Including it here keeps the count honest — the
list isn't four production tools and a hand-coded marketing site,
it's five end-to-end-prompted things, including the surface that's
making the claim. (The lab repo documents a sixth —
[`../pm-kb-workspace/`](../pm-kb-workspace/README.md) — which is
intentionally not part of the live-site count for employer-privacy
reasons explained in [`decisions/003`](../../decisions/003-private-source-public-method.md).)

## What this folder will eventually contain

The Astro content architecture (collections, schema, dynamic routes),
the CSS token system, the choice of Cloudflare Pages over Vercel /
Netlify, the form-handling architecture (why CF function → Lambda
rather than direct to Lambda or to a third-party form service), and
the prompt patterns that worked for "build a real marketing site"
vs. the more pipeline-shaped prompts from the LiDAR work.
