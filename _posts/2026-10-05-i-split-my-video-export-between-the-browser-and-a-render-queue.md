---
layout: post
title: "I Split My Video Export Between the Browser and a Render Queue"
description: "A practical VideoFlow architecture for choosing local browser exports versus queued server renders in an ecommerce app."
date: 2026-10-05 20:34:21 +0000
categories: [ecommerce]
tags: [video-automation, typescript, ecommerce, videoflow, architecture]
canonical_url: ""
image: "/assets/img/posts/2026-10-05-i-split-my-video-export-between-the-browser-and-a-render-queue/cover-d5d6388dc6d2.webp"
---

I had an export button that was doing too much. A customer could make a tidy 10-second product clip, but the same button also accepted a 90-second, media-heavy job. One of those belongs in the browser; the other quietly turns into infrastructure.

The useful distinction is not *browser versus server* as a technology preference. It is whether a customer is waiting for a small, private, interactive result or whether my system is responsible for reliably producing a lot of video. I now split those paths and keep one VideoJSON document in the middle.

![A browser exports a compact ecommerce product video locally](/assets/img/posts/2026-10-05-i-split-my-video-export-between-the-browser-and-a-render-queue/image-01-8df2bc8f30c7.webp)

## Start with the export promise

For an inline product-video tool, I use browser rendering when the customer initiated the action, the clip is short, and the browser already has the assets. A progress indicator and cancellation matter more than a job ID. [VideoFlow’s renderers](https://videoflow.dev/renderers) support a browser renderer alongside server and DOM renderers, which means I do not have to maintain separate scene definitions just to offer a direct export.

That route fits a merchant polishing a single hero clip, a support agent making one reply video, or a user downloading a small personalized recap. It keeps source media close to the user and avoids sending every experimental export through my backend.

I do **not** make the browser route the default for every request. Long timelines, large media, scheduled campaigns, retries, and hundreds of catalog variants all need a durable job boundary. The browser should ask for an export; it should not impersonate a queue.

## Keep the handoff boring: one VideoJSON document

The part that made this split manageable was treating VideoJSON as the artifact, not the MP4. My app takes product data and template choices, produces a versioned JSON document, and saves it before any expensive render begins. That document can drive a live preview, an editing surface, or either export path.

![One portable video document connects editing preview and render outputs](/assets/img/posts/2026-10-05-i-split-my-video-export-between-the-browser-and-a-render-queue/image-02-3961c4bcd69b.webp)

The shape is deliberately plain:

```ts
type VideoJob = {
  video: VideoJSON;
  delivery: "browser" | "queue";
  templateVersion: string;
  assetRefs: string[];
};

const delivery = isSmallInteractiveExport(request) ? "browser" : "queue";
```

That gives me a reviewable source of truth before I commit compute. It also leaves room for a human to adjust a generated draft in the [React Video Editor](https://videoflow.dev/react-video-editor) instead of forcing them to start over. I used the same general escape-hatch idea when I [stopped treating video variations as separate projects](https://the-lean-ecommerce.github.io/2026/09/14/i-stopped-treating-product-video-variations-as-separate-projects/): template constraints are useful until a real campaign needs a controlled exception.

## Put server work behind an explicit queue

My queued path begins after validation, not after rendering. I verify required asset references, template version, duration limit, locale, and ownership. Then a worker renders the saved document through the server renderer and writes a delivery record. A retry produces the same job input instead of trying to reconstruct a timeline from a half-finished request.

![Ecommerce product video jobs moving through a server render queue](/assets/img/posts/2026-10-05-i-split-my-video-export-between-the-browser-and-a-render-queue/image-03-fb6cac590f33.webp)

For a catalog campaign, that means a worker can process a predictable envelope: one SKU, one template version, one locale, one VideoJSON payload, one output destination. It is the right place for concurrency limits, retry policy, webhooks, and scheduled work. It is also where I want audit history when a price or product image changes after a job has started.

The authoring layer is still code-first: [VideoFlow Core](https://videoflow.dev/core) can compile the fluent TypeScript builder into portable VideoJSON. That is especially handy when a feed, an internal tool, or an agent is assembling the video. I have already found that reviewing structured output is much calmer than trying to infer what an autonomous render job did from a final MP4; that was the central lesson in my earlier piece on [using AI agents to draft reviewable product videos](https://the-lean-ecommerce.github.io/2026/09/16/how-to-use-ai-agents-to-draft-reviewable-product-videos/).

## Make the rule visible to users

The architecture only feels friendly if the UI tells the truth. For browser work, I say “Export on this device,” show progress, and offer cancellation. For the queue, I say “Prepare render,” save the draft, and show a job state with a later download or delivery link.

A simple routing checklist has been enough:

- Choose browser export for small, user-initiated, private jobs where waiting is acceptable.
- Choose the queue for batch work, scheduled work, longer videos, shared delivery, or outputs that must survive a tab closing.
- Keep preview, edit, and render attached to the same VideoJSON record.
- Store the template version and asset references with every queued job.

This split also protects my product roadmap. I can improve the in-app editing experience without redesigning the batch renderer, and I can add queue policies without making a merchant’s quick export feel like submitting a support ticket. The same discipline is why I prefer a [single content system across catalogs](https://the-lean-ecommerce.github.io/2026/10/04/i-built-one-shopify-product-content-system-for-three-catalogs/) instead of a collection of one-off interfaces: shared structure makes variation cheaper.

## The next thing I would build

Start with one template and two clearly named actions: **Export now** and **Queue render**. Log which route users choose, how long each takes, and where failures occur. That data will tell you whether the split is right far sooner than an architecture diagram will.

If you are building programmable video into a product, begin with [VideoFlow](https://videoflow.dev/): author one portable video document, preview it, let people edit it where appropriate, and send each export to the place that can actually keep its promise.
