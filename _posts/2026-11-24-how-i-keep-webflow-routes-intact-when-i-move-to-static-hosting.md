---
layout: post
title: "How I Keep Webflow Routes Intact When I Move to Static Hosting"
description: "Use a route contract to move a Webflow site to static hosting without losing CMS URLs, redirects, or the paths customers already use."
date: 2026-11-24 12:33:09 +0000
categories: [ecommerce]
tags: [webflow, static-hosting, site-migration, ecommerce]
canonical_url: ""
image: "/assets/img/posts/2026-11-24-how-i-keep-webflow-routes-intact-when-i-move-to-static-hosting/cover-6823fe30857b.webp"
---

I do not treat a Webflow-to-static move as a file transfer anymore. I treat it as a route contract. If every public URL has one explicit destination, one canonical shape, and one owner for its redirect, a static handoff can preserve the paths customers, ads, and search engines already use. That contract is the first thing I make before I change DNS.

For the export itself, I use [ExFlow's Webflow exporter](https://exflow.site/webflow): it can crawl a published site, collect public pages and assets, and ship static output to a ZIP, Git, S3, FTP, or its own hosting. That gets me a tangible artifact. It does not remove the need to decide what `/sale`, `/sale/`, an old campaign URL, or a CMS product URL should do after launch.

![Route contract cards mapping Webflow URLs to static destinations](/assets/img/posts/2026-11-24-how-i-keep-webflow-routes-intact-when-i-move-to-static-hosting/image-01-ac04fa9b01ae.webp)

## The route problem hiding inside a clean export

A checkout-linking campaign page, a collection item, and an old blog post all look like ordinary pages in Webflow. Once I put the static copy on a different host, they can fail for different reasons: the host may prefer `index.html`, strip or require a trailing slash, lack a redirect rule, or publish a folder under a new base path. The homepage can look perfect while a paid link lands on a 404.

That distinction matters for ecommerce teams. Search, bookmarks, email flows, QR codes, and partner links often point well beyond `/`. I have used a separate [12-URL export receipt](https://the-lean-ecommerce.github.io/2026/11/09/i-test-a-webflow-collection-export-with-a-12-url-receipt/) for spot checks; the route contract is the upstream document that says what those checks should expect.

## Make a small route contract, not a heroic spreadsheet

Start with the pages that carry intent: homepage, collections, product or CMS detail pages, campaigns, high-traffic posts, policy pages, and any known legacy URLs. I export analytics and Search Console lists if they are available, but I do not wait for perfect data. The first useful version is usually 15–30 URLs.

| Incoming path | Expected result | Canonical path | Rule owner | Test case |
| --- | --- | --- | --- | --- |
| `/summer-sale` | 301 to current campaign | `/collections/sale/` | host redirect config | direct request |
| `/journal/old-guide` | 301 to replacement | `/guides/new-guide/` | host redirect config | direct request |
| `/products/widget-a/` | 200 static page | `/products/widget-a/` | exported files | desktop + mobile |
| `/contact` | 200 plus working submission endpoint | `/contact/` | static host + form service | submit test |

The key column is **rule owner**. HTML can own an existing page; the host, CDN, or a redirect config must own a legacy redirect. When nobody owns it, I assume it will be lost.

## Export the site, then inspect its route shape

In ExFlow I enable the page, CSS, JavaScript, and media options I need; for a Webflow site with collection content, I also use its all-pages crawl so public collection item pages are included. ExFlow documents that its output includes static HTML, CSS, JavaScript, assets, and crawled CMS pages, while the output can be downloaded or synced to Git, S3, or FTP.

Before deploying, I inspect the ZIP or repository rather than trusting filenames. I answer four boring questions:

1. Does a route become a folder with `index.html`, a `.html` file, or both?
2. Can the target host serve that form at the desired public URL?
3. Are internal links relative and still correct at the new host base path?
4. Which routes require a redirect outside the export?

This is also where I name the non-static dependencies. ExFlow notes that Webflow-native form submissions, Ecommerce checkout, site search, password-protected pages, and the Editor depend on Webflow at request time. A page that contains a form is not a working form just because it rendered in the export. I use the same discipline as in my [Webflow form replacement checklist](https://the-lean-ecommerce.github.io/2026/11/22/how-to-replace-webflow-forms-before-you-host-a-static-export/): point the form to a replacement endpoint and run a real submission before cutover.

![Retro mechanical sorter showing successful and broken redirect paths](/assets/img/posts/2026-11-24-how-i-keep-webflow-routes-intact-when-i-move-to-static-hosting/image-02-8b85b6ac7af0.webp)

## Normalize first; redirect second

I pick one public form for each route—usually the trailing-slash version for folder-style static sites—and make every alternative redirect to it. This avoids accidentally supporting two canonical URLs for the same page. The exact syntax depends on the host, so I keep redirect rules in the host or repository configuration rather than inventing them in the exported HTML.

For each redirect, check the first response rather than just the final page in the browser. A correct destination after a chain of redirects is still a maintenance problem. My minimum test set is: old path, canonical path, no-slash variant, slash variant, and an encoded query-string campaign URL when one exists.

Do not redirect everything to the homepage. If an old page has a close replacement, redirect there. If it is truly retired, use a deliberate status strategy with the site owner instead of quietly turning a relevant link into an irrelevant landing page.

## Run the contract on a staging URL before DNS

I deploy the exported copy to a staging host first, then make a tiny route test list from the contract. It is intentionally boring:

```text
GET /summer-sale              -> 301 -> /collections/sale/
GET /collections/sale/        -> 200, canonical matches
GET /products/widget-a/       -> 200, assets load
GET /contact/                 -> 200, replacement form submits
GET /journal/old-guide        -> 301 -> intended guide
```

I check these in a browser and with a redirect-following HTTP client, then test a collection/detail page on a narrow viewport. That last bit catches relative assets and navigation states that a homepage-only check misses. My [Framer static smoke-test workflow](https://the-lean-ecommerce.github.io/2026/11/23/how-i-smoke-test-a-framer-static-export-before-i-publish-it/) is a useful reminder that a visually complete static build still needs route, asset, interaction, and mobile verification.

![Static site verification station with desktop and mobile storefront screens](/assets/img/posts/2026-11-24-how-i-keep-webflow-routes-intact-when-i-move-to-static-hosting/image-03-6ac554a36009.webp)

## Keep the contract for the next export

The practical win is not the first migration; it is repeatability. Store the route contract beside the deployment configuration and rerun it after a Webflow redesign, campaign change, or new export. If you sync ExFlow output to Git, that contract can live beside the static files and make a rollback much less mysterious.

If you are exporting a different builder later, ExFlow also has dedicated [Framer](https://exflow.site/framer) and [Squarespace](https://exflow.site/squarespace) paths, but I would still make the route contract platform-specific. The hosting rules and dynamic features are where surprises hide.

My next action before a Webflow static cutover is simple: list 20 URLs that matter, give each one an expected status and canonical destination, then prove the list on staging. Exporting is the easy part; preserving the routes is what makes the move safe.
