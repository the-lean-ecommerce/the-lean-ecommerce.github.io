---
layout: post
title: "How I Clean Shopify Product CSVs Before They Break an Import"
description: "A practical, no-upload browser workflow for inspecting, standardizing, and QAing Shopify product CSVs before import."
date: 2026-08-22 05:32:22 +0000
categories: [ecommerce]
tags: [shopify, csv, product-data, ecommerce-operations, automation]
canonical_url: ""
image: "/assets/img/posts/2026-08-22-how-i-clean-shopify-product-csvs-before-they-break-an-import/cover-5d1d862ff92f.webp"
---

I do not open a product CSV to admire its columns. I open it because I am about to change something that can make a Shopify import unexpectedly expensive: a collection refresh, a supplier handoff, a bulk price update, or an asset migration. The first problem is usually not the data itself. It is that the export has become too wide, too duplicated, or too inconsistent to trust at a glance.

For small cleanup jobs, I now start in [Tiny Online Tools](https://tiny-online.tools/). It is a collection of browser-based utilities that emphasizes no uploads, no accounts, and no tracking. That is a good fit for a catalog export with product titles, SKUs, inventory notes, or supplier fields that I do not want to hand to a random converter. The useful part is not replacing a proper data pipeline; it is making the five-minute preflight repeatable.

![Ecommerce product catalog CSV inspection grid](/assets/img/posts/2026-08-22-how-i-clean-shopify-product-csvs-before-they-break-an-import/image-01-a04351db3b83.webp)

## 1. Inspect the shape before touching a row

My first stop is the [CSV Viewer](https://tiny-online.tools/data-tools/csv-viewer). I upload or paste a copy of the export, then scan the header row and a few records as a table. The viewer is deliberately boring, which is exactly what I want. It turns raw commas and quoted values into visible columns without committing me to a spreadsheet cleanup session.

I check three things before I edit anything:

- Is this actually the export I meant to change? A stale feed is a surprisingly easy way to undo yesterday’s work.
- Do the fields I care about exist, and are their headers consistent? Product data often arrives with variants of the same idea: `image_url`, `image`, `main_image`, or a supplier-specific label.
- Are there obvious rows that should not be included in the next operation: archived items, internal samples, draft products, or a second copy of the same handle?

That last check matters most when I am planning bulk edits. I use the same pause before a [bulk Shopify product edit](https://how-to.the-lean-ecommerce.com/2026/08/20/how-to-bulk-edit-shopify-products-without-risking-variant-data/): scope the cohort first, then change it. A CSV is not safer just because it is a file.

## 2. Make the import schema explicit

Next I reduce the file to the columns the next step actually needs. [CSV Column Extractor](https://tiny-online.tools/data-tools/csv-column-extractor) can keep fields by header name or position, so I use it to make a smaller working export. For an image refresh, that might be handle, title, image source, alt text, and position. For a collection change, it might be handle, tags, type, and status.

The win is reviewability. A 70-column supplier file can be valid and still be impossible to reason about quickly. A ten-column task-specific file makes gaps visible. It also gives a teammate a much clearer artifact to approve before I import.

![CSV schema normalization for product catalog fields](/assets/img/posts/2026-08-22-how-i-clean-shopify-product-csvs-before-they-break-an-import/image-02-55e605e38a4b.webp)

If the headers differ between exports, I follow with [CSV Column Renamer](https://tiny-online.tools/data-tools/csv-column-renamer). Its mapping is intentionally narrow: rename the headers you specify and leave data rows alone. I use it to normalize a file to the names expected by the next tool or script, not to pretend mismatched source data is fixed.

For example, I may map a vendor’s `product_code` to my working `sku` field, but only after sampling values to confirm that assumption. That is the line between a useful cleanup and a silent data translation bug. This is also where product-media work benefits from a clear schema: the same small discipline behind a [photo decision system](https://the-lean-ecommerce.github.io/2026/08/14/how-i-build-a-publish-safe-shopify-photo-decision-system/) applies to a CSV that controls those photos.

## 3. Dedupe intentionally, then sort for review

Duplicates need a rule before they need a button. The [CSV Deduplicator](https://tiny-online.tools/data-tools/csv-deduplicator) lets me choose which columns define a duplicate and whether to keep the first or last matching row. That makes it much safer than blindly deleting repeated lines.

For a product feed, I normally begin with the identifier that is supposed to be unique for this task—often handle, SKU, or a vendor ID—not every cell in the row. Two rows can represent the same product while legitimately carrying different timestamps or notes. If duplicates are expected because variants or extra images are modeled as separate rows, I stop and write down the rule before removing anything.

Then I use [CSV Sorter](https://tiny-online.tools/data-tools/csv-sorter) to group the output by the field I am reviewing. Sorting by handle reveals repeated products; sorting by image source exposes missing asset patterns; sorting by product type makes accidental category drift obvious. It is the low-tech counterpart to the release gate I use for [size guides](https://the-lean-ecommerce.github.io/2026/08/20/my-shopify-size-chart-release-gate-four-checks-before-a-new-drop/): arrange the data so a human can see the bad assumption.

## 4. Run a tiny import gate

Before I move the cleaned file into Shopify or another catalog tool, I do one final pass:

1. Re-open the saved output in CSV Viewer and compare a few representative rows with the original export.
2. Confirm the column count and header names match the intended importer.
3. Check every filter or dedupe rule against edge cases: one variant product, one product with multiple images, one archived product, and one row with a blank optional field.
4. Import the smallest safe sample first when the operation can have broad consequences.

![Product feed quality gate before import](/assets/img/posts/2026-08-22-how-i-clean-shopify-product-csvs-before-they-break-an-import/image-03-1cb199e3926f.webp)

This is where I also verify related media, because an apparently clean feed can still point at an unready visual asset. If I have changed image URLs or alt text, I run the same three-pass thinking from [my AI product-photo publishing check](https://the-lean-ecommerce.gitlab.io/2026/08/21/my-three-pass-check-before-publishing-ai-product-photos/): inspect the source, inspect the output, and inspect the customer-facing result.

## Why I keep this browser workflow around

I still use scripts and proper transformations for recurring jobs. But a one-off catalog repair should not require installing another app, sharing a customer export with an unknown service, or building a throwaway script just to inspect and rename four fields. Tiny Online Tools gives me a quick, private-feeling workspace for those narrow steps, and its [data tools](https://tiny-online.tools/data-tools) keep the related jobs together.

My next action is simple: take the next product CSV you are tempted to import as-is, open it in [CSV Viewer](https://tiny-online.tools/data-tools/csv-viewer), and write down the exact column and duplicate rules before changing a row. That five-minute gate is much cheaper than cleaning up a broken catalog after it ships.
