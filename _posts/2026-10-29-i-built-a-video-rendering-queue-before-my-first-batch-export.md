---
layout: post
title: "I Built a Video Rendering Queue Before My First Batch Export"
description: "A practical VideoJSON queue architecture for generating, previewing, rendering, and reviewing repeatable video exports."
date: 2026-10-29 02:35:33 +0000
categories: [ecommerce]
tags: [programmatic-video, videojson, typescript, video-rendering, ecommerce]
canonical_url: ""
image: "/assets/img/posts/2026-10-29-i-built-a-video-rendering-queue-before-my-first-batch-export/cover-298220115a16.webp"
---

I did not need a video queue until I tried to make a short product clip for every item in a small catalog. One demo was easy. A batch was a different problem: incomplete product data, retries, someone changing a caption halfway through, and a final MP4 that had no obvious connection to the inputs that produced it.

The thing that made the workflow manageable was deciding that the queue would not carry an opaque render request. It would carry a versioned video document. With [VideoFlow Core](https://videoflow.dev/core), I could build that document from data, compile it to portable VideoJSON, and make the JSON the handoff between generation, review, rendering, and delivery.

![Product event cards moving through validation into a video timeline](/assets/img/posts/2026-10-29-i-built-a-video-rendering-queue-before-my-first-batch-export/image-01-cbf3af42594d.webp)

## Start with an event, but make the job boringly explicit

For a catalog-video job, my input is an event plus a product snapshot: product ID, title, media URLs, price, locale, template version, and a requested output size. I persist that snapshot with the job instead of having the worker ask Shopify for the latest product again. That sounds fussy, but it stops a retry from silently rendering a different price or image than the original request.

Before I enqueue anything, I validate the fields the template actually needs. Missing image? Wrong currency? A description too long for the caption slot? Fail the job with a useful reason. Do not let the renderer become the first place that discovers bad input.

Then I turn the validated input into VideoJSON. A simplified job record looks like this:

```ts
type RenderJob = {
  id: string;
  templateVersion: string;
  video: VideoJSON;
  output: { width: number; height: number; format: "mp4" };
  attempts: number;
  status: "queued" | "preview" | "rendering" | "done" | "failed";
};
```

That `video` field is doing real work. It is inspectable, diffable, and replayable. If a job fails, I can tell whether it was the product payload, template compiler, renderer, or delivery step without reconstructing a mystery request from logs.

## Keep VideoJSON at the center

My first instinct was to put template data in a queue and let every worker recreate the scene. That is fine until the preview team and the rendering worker disagree about how the scene should look. The better contract is: compile once, store the VideoJSON, then run every downstream surface from that same document.

VideoFlow makes this practical because the [Core](https://videoflow.dev/core) can author the video with TypeScript, while the portable JSON can be displayed, edited, or rendered elsewhere. I use the DOM renderer for a live preview in the internal dashboard, so a person can scrub the video before it costs an export. For a customer-facing product, the same document can open in the [React Video Editor](https://videoflow.dev/react-video-editor), where the customer can change permitted copy or media rather than asking for a new render.

The important constraint is to decide what editing is allowed. I let people swap an image, adjust a line of copy, and trim a clip. I do not let them remove the legal card or delete the CTA scene. That turns an editor from a source of production surprises into a safe approval surface.

![A shared VideoJSON document feeding browser and server rendering paths](/assets/img/posts/2026-10-29-i-built-a-video-rendering-queue-before-my-first-batch-export/image-02-89310f6305eb.webp)

## Pick the renderer per job, not per product

There is no universal answer to browser versus server rendering. I use the [VideoFlow renderers](https://videoflow.dev/renderers) as two deliberate lanes.

Browser rendering is useful when a user is exporting a small, private project and I do not want to upload source media or pay for a backend job. It also makes a nice escape hatch for an internal tool where the operator expects to wait at their laptop. I expose progress and cancellation, because an export button without either becomes a trust problem quickly.

Server rendering is the batch lane. It is the right place for hundreds of SKU clips, scheduled campaigns, webhook-triggered videos, and a durable retry policy. A worker takes the stored VideoJSON, marks the job as rendering, writes the MP4 to object storage, and records the result URL plus renderer version. For big queues, I set an attempt limit and send repeated failures to a review state; retrying a corrupt source forever is not resilience.

The shared VideoJSON is the useful part: the browser preview, the editable timeline, and the server export should all have the same source. That is a cleaner architecture than maintaining three slightly different template implementations.

## Make delivery a state, not a side effect

A rendered MP4 is not automatically delivered. In my queue, `done` means the file exists, its metadata matches the requested resolution and duration, and the intended destination accepted it. If I am pushing videos into a product-media workflow, I also retain the job ID and original VideoJSON beside the output. The next person can answer “where did this clip come from?” without digging through a Slack thread.

This is also where versioning pays off. When I improve a lower-third or fix a locale formatter, I increment the template version. Existing clips stay traceable to the old template, while a re-render is an explicit new job rather than a surprise overwrite. It is the same operational habit I use when I [let customers edit AI-generated videos in my SaaS](https://the-lean-ecommerce.github.io/2026/10/28/how-i-let-customers-edit-ai-generated-videos-in-my-saas/): a reviewable source of truth is more valuable than a clever one-shot generation.

![A video delivery review station with approval checklist and versioned source cards](/assets/img/posts/2026-10-29-i-built-a-video-rendering-queue-before-my-first-batch-export/image-03-9069fd7d57c8.webp)

## The small review loop that saved my queue

Before releasing a new template, I run five deliberately awkward records through it: a long title, missing secondary image, discounted price, non-English locale, and a product with a weirdly shaped photo. That one mini suite catches more layout and data-contract mistakes than staring at the happy-path demo. It also echoes the reviewable-pipeline lesson in [this VideoJSON workflow](https://how-to.the-lean-ecommerce.com/2026/10/24/how-to-build-a-reviewable-ai-video-pipeline-with-videojson/): previewing structured output is much cheaper than explaining a bad batch after the fact.

I keep an approval sample for each template release, then let the queue process a limited batch before opening the floodgates. If the workflow later needs an editorial input rather than product data, the same idea works for [catalog-driven reviewable ecommerce videos](https://the-lean-ecommerce.blogspot.com/2026/10/how-to-turn-product-catalog-data-into.html). The inputs change; the contract does not.

## What I would build first

If you are starting from zero, do not begin with a massive rendering service. Build one template, one validated job schema, one stored VideoJSON document, and one visible preview. Add a server worker once you have a real batch or scheduled need. VideoFlow is open source under Apache-2.0, which makes it a practical stack to evaluate when you want the video document, preview, editor, and renderer under your own control.

My next action would be simple: pick ten real products, define the exact fields your template needs, and render a review batch. If you cannot explain one job from event to VideoJSON to MP4, the queue is not ready for a hundred.
