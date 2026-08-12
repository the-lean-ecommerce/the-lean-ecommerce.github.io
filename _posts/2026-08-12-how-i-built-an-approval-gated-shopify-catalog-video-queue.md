---
layout: post
title: "How I Built an Approval-Gated Shopify Catalog Video Queue"
description: "A practical VideoFlow workflow for turning Shopify product data into reviewable video drafts before browser or server rendering."
date: 2026-08-12 01:27:17 +0000
categories: [ecommerce]
tags: [shopify, video-automation, typescript, ecommerce, videoflow]
canonical_url: ""
image: "/assets/img/posts/2026-08-12-how-i-built-an-approval-gated-shopify-catalog-video-queue/cover-81c778e652c9.webp"
---

I wanted a short product video on every Shopify page, but I did not want a pile of automatic MP4s that nobody had looked at. The real bottleneck was not rendering. It was getting from a product record to something the marketing team could approve without turning every SKU into a tiny editing project.

My fix was to treat video as a reviewable build artifact: product data in, structured video data in the middle, a preview for approval, then an MP4 only after someone gives it the nod. [VideoFlow](https://videoflow.dev/) is a good fit for this because its TypeScript builder compiles a project into portable VideoJSON, which can then drive a live preview, a browser export, or a server job.

![Catalog data to VideoJSON approval workflow](/assets/img/posts/2026-08-12-how-i-built-an-approval-gated-shopify-catalog-video-queue/image-01-54111cbc3b78.webp)

## The Smallest Useful Contract

I started with a deliberately boring input shape: title, price, three image URLs, a benefit list, destination URL, and the variant label. That is enough to make a 10–15 second product clip without asking an LLM—or a teammate—to invent facts. Keep the product source authoritative and put the design decisions in a template.

The template has named slots for the hero image, headline, price, proof point, and CTA. From there, the VideoFlow core builder gives me text, image, shape, audio, and caption layers; the output is VideoJSON instead of an opaque export. That matters because I can store the JSON next to the product revision, diff it in a pull request, and regenerate the same draft later. The [core docs](https://videoflow.dev/core) and [examples](https://videoflow.dev/examples) are the places I’d start before adding more scene logic.

```ts
const draft = await buildProductVideo({
  product: approvedProductData,
  template: "product-brief-v1",
  locale: "en-CA",
});

await saveDraft({ productId, videoJSON: draft, status: "needs-review" });
```

The important bit is the status, not the syntax. My render queue only accepts `approved` drafts. Everything else stays cheap and reversible.

## Preview Before You Pay for the Render

For review, I mount the same JSON in a live DOM preview. The reviewer checks the unglamorous things that ruin a product-page video: the title wraps cleanly, the correct variant image appears, the price is current, and the CTA points to the right product. If a promo changes, I change the data or the template and make a new draft; I do not patch an MP4.

![Video draft review desk with structured JSON](/assets/img/posts/2026-08-12-how-i-built-an-approval-gated-shopify-catalog-video-queue/image-02-e8b507077c03.webp)

This is where the workflow is nicer than a fully automatic generator. A person can flag a weak crop or a bad product-photo sequence while the source is still editable. If you are already running human review on campaign video, my earlier notes on [adding human review to Shopify product-video automation](https://how-to.the-lean-ecommerce.com/2026/08/10/how-to-add-human-review-to-shopify-product-video-automation/) make a useful companion.

For teams that need more than approve/reject, VideoFlow’s [React video editor](https://videoflow.dev/react-video-editor) gives the reviewer a way to trim, reorder, adjust text, or move a layer while keeping the same VideoJSON as the source of truth. I would still lock the template’s brand-critical elements. Controlled flexibility is much easier to operate than a blank timeline.

## Pick the Renderer After Approval

I use the renderer as a deployment decision, not a fork in the authoring process. A small one-off export can happen in the browser with WebCodecs, which is handy when I want to avoid pushing source media to a render server. A scheduled product-feed batch goes to the server renderer, where a queue can manage retries and heavier exports. Both use the same JSON; the [renderer guide](https://videoflow.dev/renderers) covers the browser, server, and DOM options.

![Browser and server video render queue](/assets/img/posts/2026-08-12-how-i-built-an-approval-gated-shopify-catalog-video-queue/image-03-c1aea4462bb7.webp)

A simple queue record has been enough for me:

```json
{
  "productId": "sku_123",
  "templateVersion": "product-brief-v1",
  "videoJSONVersion": 4,
  "approval": "approved",
  "renderTarget": "server"
}
```

That gives operations a clean retry boundary. If an asset download fails, retry the render. If the product changes, create a new draft and send it back through review. Do not quietly overwrite a video that has already been approved.

## What I Would Measure First

Before expanding this to the whole catalog, I would pilot 20 products and measure: draft-to-approval time, review rejection reasons, render failures, and how often a product data change invalidates an approved draft. Those numbers tell you whether to improve the feed, the template, or the review UI. They also keep you from mistaking a fast renderer for a reliable workflow.

If your input images need a cleanup gate first, the product-photo routing setup I wrote about in [my Shopify photo QA pipeline](https://the-lean-ecommerce.github.io/2026/08/03/how-i-build-a-shopify-photo-routing-system-for-product-pages-and-ads/) fits naturally in front of this queue. If your goal is creative testing rather than PDP coverage, use the same contract to branch a product brief into [a week of UGC video tests](https://the-lean-ecommerce.github.io/2026/08/11/how-i-turn-one-shopify-product-into-a-week-of-ugc-video-tests/). And if you are building the whole intake side from a feed, my [reviewable video-queue walkthrough](https://the-lean-ecommerce.github.io/2026/08/09/how-i-turn-a-shopify-product-feed-into-a-reviewable-video-queue/) shows why the queue should be visible long before rendering starts.

## Start With One Template and One Gate

The win here is not that every product gets an MP4. It is that every generated video has a traceable input, an editable VideoJSON draft, and an explicit approval decision. That is the part that makes programmatic video feel like infrastructure instead of a content gamble.

Pick one product template, send ten real SKUs through it, and keep the approval gate manual for the first pass. Once the rejection reasons get boring, you have earned the right to automate the next piece. Try the [VideoFlow playground](https://videoflow.dev/playground) with a single product brief, then wire the resulting contract into your own queue.
