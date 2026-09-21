# Scraping product lists from AndyMark, goBILDA, REV

  

## user - 21 Sep 2026, 17:51:22

  

I am looking to get a comprehensive list of all the parts that are sold by: "https://andymark.com", "https://www.gobilda.com", "https://www.revrobotics.com". Have a look to see whether they have publicly available APIs, if they do tell me. Otherwise maybe a site scraper could work? Create a solid plan before moving on. Ask me questions if needed.

  

## AI - 21 Sep 2026, 17:51:22

  

I'll research the three sites' structure and whether they expose APIs before drafting a plan.

  

*Time to respond: 5s*

  

## AI - 21 Sep 2026, 17:51:28

  

Interesting — AndyMark runs on Shopify (with an agent/MCP endpoint), while goBILDA and REV appear to be BigCommerce. Let me verify the data surfaces.

  

*Time to respond: 3s*

  

## AI - 21 Sep 2026, 17:51:31

  

AndyMark exposes a clean public JSON API. goBILDA and REV are BigCommerce; let me locate their sitemaps and product-page structure.

  

*Time to respond: 3s*

  

## AI - 21 Sep 2026, 17:51:35

  

Both use BigCommerce product sitemaps. Let me inspect product counts and page structure.

  

*Time to respond: 3s*

  

## AI - 21 Sep 2026, 17:51:38

  

Now let me inspect the BigCommerce product pages for structured data and any embedded Storefront API tokens (which would give a cleaner catalog route than scraping).

  

*Time to respond: 8s*

  

## AI - 21 Sep 2026, 17:51:47

  

Both BigCommerce stores embed rich product JSON in each page. Let me size the catalogs (sitemap paging) and confirm structured data.

  

*Time to respond: 13s*

  

## AI - 21 Sep 2026, 17:52:00

  

Let me confirm the exact fields available in each BigCommerce page payload and check the goBILDA sitemap composition.

  

*Time to respond: 7s*

  

## AI - 21 Sep 2026, 17:52:08

  

Let me check the existing repo to align the plan with your stack.

  

*Time to respond: 5s*

  

## AI - 21 Sep 2026, 17:52:23

  

## Findings: APIs vs. scraping

  

**AndyMark (andymark.com) — public API, no scraping needed.** It's a Shopify store with a documented agent/read surface:

- `GET /products.json?limit=250&page=N` → full catalog JSON. I confirmed **~1,972 products** across 8 pages (variants nested).

- Also `/products/{handle}.json`, `/collections/{handle}/products.json`, `/sitemap.xml`.

- A full **UCP/MCP commerce endpoint** exists at `POST /api/ucp/mcp` (discovery at `/.well-known/ucp`, docs at `/agents.md`), but for a read-only catalog `products.json` is simpler and sufficient.

  

**goBILDA (gobilda.com) — BigCommerce, no public product API.** No embedded Storefront token, and `robots.txt` blocks `search.php`. But it publishes a product sitemap: `/xmlsitemap.php?type=products&page=1` → **2,387 URLs, of which 325 are `/rd-` landing pages** (not products) → **~2,062 real product pages**. Each page server-renders structured data: JSON-LD **and** a `window.jsContext` JSON blob (sku, price, category, image).

  

**REV (revrobotics.com) — same BigCommerce platform.** `/xmlsitemap.php?type=products&page=1` → **390 product URLs**. Each page embeds `var BCData` (sku, upc, price, stock) plus JSON-LD.

  

So: AndyMark is a clean API pull; goBILDA and REV need a polite sitemap-seeded page scraper. Note `robots.txt` sets `Crawl-delay: 10` for named AI bots (not the default `*`), so I'd use conservative throttling regardless.

  

Both Python 3.12 and Node 24 are available locally. The repo is the Kotlin/Compose **FTC Equipment Tracker** app whose data model wants `parts/{sku, name, vendor}` — so this is likely a catalog seed.

  

## Proposed plan (draft)

  

1. **Fetch layer**

   - AndyMark: paginate `products.json`, flatten variants, map to schema.

   - goBILDA + REV: download product sitemap, filter `/rd-`/non-product URLs, fetch each page with ~1 req/s delay + retry/backoff, parse JSON-LD first (fallback to the embedded JSON blob).

2. **Normalize** into one schema: `sku, name, vendor, url, price, currency, availability, category, imageUrl, description` (+ source-specific extras kept in a raw field).

3. **De-dupe & QA**: check SKU collisions, truncated names, missing prices; produce a QA report with counts per vendor.

4. **Output**: normalized JSON (+CSV) written where you want it; a re-runnable script under e.g. `tools/catalog/`.

  

Before I finalize, I need a few decisions:

  

*Time to respond: 4m 40s*

  

## AI - 21 Sep 2026, 17:57:03

  

## Final Plan — FTC parts catalog (AndyMark, goBILDA, REV)

  

### Approach

| Vendor | Method | Est. items |

|---|---|---|

| AndyMark | Shopify `products.json` API (paged, no auth) | ~1,972 products (more variant rows) |

| goBILDA | BigCommerce product sitemap → scrape each page's JSON-LD (fallback: `window.jsContext`) | ~2,062 (2,387 sitemap URLs − 325 `/rd-` pages) |

| REV | Same, using `var BCData` / JSON-LD | 390 |

  

### Deliverables (in repo)

- `tools/catalog/scrape_catalog.py` — Python 3, stdlib-only, idempotent/resumable, dated snapshots.

- `tools/catalog/out/parts_andymark.json`, `parts_gobilda.json`, `parts_rev.json`

- `tools/catalog/out/parts_all.json`, `parts_all.csv`

- `tools/catalog/out/meta.json` — snapshot timestamp, per-vendor counts, QA stats.

  

### Row schema (per your answers)

```json

{ "sku": "am-5866_blue", "name": "BIOBUZZ Hive Cell Ribs", "vendor": "andymark",

  "url": "https://andymark.com/products/biobuzz-hive-cell-ribs", "nonRobot": false }

```

One row per variant/SKU. All sellable items included, merch/apparel/non-robot flagged via a `nonRobot` boolean.

  

### Steps

1. **Scaffold** `tools/catalog/` with the script (stdlib `urllib`/`json`/`csv`, so no pip installs).

2. **AndyMark fetcher** — page `/products.json?limit=250&page=N` until empty, flatten variants, map to schema; `nonRobot` from product type/tags.

3. **BigCommerce fetcher (goBILDA + REV)** — pull `xmlsitemap.php?type=products&page=1`, drop `/rd-` and non-product URLs, fetch each page at **~1 req/s with retry/backoff and a resumable checkpoint**, parse JSON-LD first, fall back to embedded JSON; derive `nonRobot` from category/breadcrumb.

4. **Normalize + de-dupe + QA** — dedupe on `(vendor, sku)` falling back to `(vendor, url)`; report counts, missing SKUs/names, duplicates, and non-robot totals.

5. **Emit** JSON + CSV + `meta.json`.

6. **Verify** — run end-to-end, confirm counts, spot-check several SKUs against the live pages.

  

### Notes / risks

- BigCommerce has no public catalog API (no storefront token embedded), so page scraping is the only route; expect occasional 403/429 — handled by throttling + retry + resume.

- Non-robot tagging is heuristic; QA output will show counts so you can adjust patterns.

- Default polite throttle is ~1 req/s (goBILDA's `robots.txt` sets `Crawl-delay: 10` for named AI bots). Full BigCommerce run ≈ 40–50 min. Say the word if you want it slower/faster.

- Studica is out of scope per this request.

  

Ready to implement on your go-ahead.

  

*Time to respond: 11s*

  

## user - 21 Sep 2026, 17:59:48

  

Also create tags for general categories of items, e.g. "motor", "extrusion", etc. if possible.

  

## AI - 21 Sep 2026, 17:59:48

  

Good addition. Let me verify what category data each site exposes so the tagging is grounded rather than guessed.

  

*Time to respond: 9s*

  

## AI - 21 Sep 2026, 17:59:58

  

Category data is available at all three sources, so tagging is feasible:

  

- **AndyMark** — `product_type` (clean-ish: `Gearmotors`, `Bearings`, `Servos`, `Extrusion & Structure`, `Screws`, `Washers`, `Merchandise`…) plus `tags` and 214 collections. Some `product_type` values are season-specific and need mapping.

- **goBILDA** — JSON-LD `BreadcrumbList` (Home › … › product) **and** `window.jsContext.product.category` (e.g. `["KITS/Chassis Kits"]`) plus a full category tree (MOTION, ELECTRONICS, HARDWARE, KITS, MERCH…).

- **REV** — JSON-LD `BreadcrumbList` (e.g. Home › REV ION › Brushless › NEO Vortex & SPARK Flex › product).

  

## Revised plan additions

  

**Schema gains two fields:**

```json

{

  "sku": "REV-21-2807",

  "name": "Vortex Shaft - 8mm",

  "vendor": "rev",

  "url": "https://www.revrobotics.com/rev-21-2807/",

  "nonRobot": false,

  "categories": ["REV ION", "Brushless", "NEO Vortex & SPARK Flex"],

  "tags": ["shaft", "motor-accessory"]

}

```

  

- **`categories`** — the vendor's own breadcrumb/product-type path, preserved verbatim (array). Keeps fidelity and lets you re-map later without re-scraping.

- **`tags`** — a normalized, controlled vocabulary derived by keyword rules over name + vendor category. Starter set (tunable): `motor`, `gearbox`, `servo`, `bearing`, `gear`, `sprocket`, `chain`, `belt`, `pulley`, `wheel`, `hub`, `shaft`, `extrusion`, `bracket`, `plate`, `screw`, `nut`, `washer`, `spacer`, `standoff`, `collar`, `cable`, `connector`, `controller`, `sensor`, `battery`, `power`, `camera`, `kit`, `merch`, `tool`, `misc`.

- `nonRobot` becomes derivable from `tags`/`categories` (merch/apparel → true).

  

**Scrape steps updated:**

2. AndyMark — also read `product_type`/`tags` for `categories`.

3. BigCommerce — parse JSON-LD `BreadcrumbList` for `categories` (goBILDA additionally uses `jsContext.product.category`).

4. Normalize — apply keyword rule table → `tags`; report tag distribution + untagged count in `meta.json` for tuning.

  

### One decision

How should `tags` be produced?

  

*Time to respond: 21s*

  

## AI - 21 Sep 2026, 18:00:19

  

## Final Plan — FTC parts catalog scraper (AndyMark, goBILDA, REV)

  

### Deliverables (repo)

- `tools/catalog/scrape_catalog.py` — Python 3, stdlib-only (`urllib`, `json`, `csv`), resumable, dated snapshots.

- `tools/catalog/out/parts_andymark.json`, `parts_gobilda.json`, `parts_rev.json`

- `tools/catalog/out/parts_all.json`, `parts_all.csv`

- `tools/catalog/out/meta.json` — snapshot date, per-vendor counts, tag distribution, untagged/duplicate/missing-field QA stats.

  

### Row schema

```json

{

  "sku": "am-5866_blue",

  "name": "BIOBUZZ Hive Cell Ribs",

  "vendor": "andymark",

  "url": "https://andymark.com/products/biobuzz-hive-cell-ribs",

  "nonRobot": false,

  "categories": ["FIRST Tech Challenge 2026-2027 - BIOBUZZ"],

  "tags": ["plate"]

}

```

One row per variant/SKU; all sellable items; `nonRobot` derived from tags/categories; `categories` = vendor breadcrumb/type verbatim; `tags` = normalized vocabulary via keyword rules over name + vendor category.

  

### Steps

1. **Scaffold** `tools/catalog/` with the script and an editable keyword→tag rule table.

2. **AndyMark** — page `/products.json?limit=250&page=N` to exhaustion (~1,972 products), flatten variants, map `product_type`/`tags` → `categories`.

3. **goBILDA + REV** — fetch `xmlsitemap.php?type=products&page=1`, drop `/rd-`/non-product URLs (~2,062 goBILDA, 390 REV), GET each page at **~1 req/s, retry/backoff, checkpointed so it resumes after interruption**; parse JSON-LD `Product` + `BreadcrumbList` (fallback: goBILDA `window.jsContext`, REV `var BCData`).

4. **Normalize** — apply keyword rule table → `tags`; derive `nonRobot`; dedupe on `(vendor, sku)` else `(vendor, url)`.

5. **Emit** JSON (per-vendor + combined), CSV, and `meta.json` with QA/tag stats.

6. **Verify** — run end-to-end, report counts, spot-check SKUs and tags against live pages, tune rules for the largest untagged bucket.

  

### Notes

- BigCommerce has no public catalog API, so scraping is required; throttling/retry/resume handles anti-bot responses.

- Tag rules will need one tuning pass after the first run; `meta.json` highlights where.

- Full run ≈ 40–50 min (dominated by ~2,450 BigCommerce page fetches).

- AndyMark's MCP/UCP endpoint exists but `products.json` is simpler and sufficient for a read-only catalog.

- Studica out of scope for this request.

  

Ready to implement when you approve.

  

*Time to respond: 5s*

  

## user - 21 Sep 2026, 18:01:18

  

Approved. Make no mistakes.

  

## AI - 21 Sep 2026, 18:01:18

  

Approved — implementing now. First, a quick inspection to get variant handling exactly right before writing the scraper.

  

*Time to respond: 10s*

  

## AI - 21 Sep 2026, 18:01:39

  

JSON-LD gives uniform `name`/`sku`/`url`/`offers` plus breadcrumbs for categories. Let me confirm variant handling on products that actually have options.

  

*Time to respond: 7s*

  

## AI - 21 Sep 2026, 18:01:53

  

Sitemaps include category/landing pages mixed with real products (e.g. REV `5mm-Hex-Shafts` has no SKU, REV JSON-LD can emit a numeric pseudo-SKU). I'll detect and skip those. Let me check whether any goBILDA products expose option-level SKUs.

  

*Time to respond: 11s*

  

## AI - 21 Sep 2026, 18:02:29

  

Two pages returned no structured data at all — need to see whether that's a real template difference or a soft block.

  

*Time to respond: 4s*

  

## AI - 21 Sep 2026, 18:02:38

  

Critical finding: goBILDA sitemap contains **meta-refresh redirect stubs**, some of which encode variant SKUs via `?sku=` (apparel sizes) while others point to category anchors. I'll handle stubs generically. Let me profile the sitemaps.

  

*Time to respond: 11s*

  

## AI - 21 Sep 2026, 18:03:06

  

I have what I need. Now writing the scraper.

  

*Time to respond: 21s*

  

## AI - 21 Sep 2026, 18:03:50

  

Let me review the written file and fix a few rule/parse issues I want to be certain about.

  

*Time to respond: 13s*

  

## AI - 21 Sep 2026, 18:04:47

  

AndyMark looks clean. Let me unit-test the tricky BigCommerce branches (stub variant, category page, real product) before the long run.

  

*Time to respond: 5s*

  

## AI - 21 Sep 2026, 18:05:02

  

Two fixes needed: tags should come from the product name first (category text caused a `shaft` to be tagged `motor`/`controller` via "Brushless"/"SPARK"), and I should check the skipped REV plush page.

  

*Time to respond: 6s*

  

## AI - 21 Sep 2026, 18:05:19

  

`Smart-servo-v2` is a real product being skipped. Let me inspect why.

  

*Time to respond: 4s*

  

## AI - 21 Sep 2026, 18:05:28

  

REV has option-parent pages where JSON-LD emits an integer internal id (e.g. 894, 955) and BCData SKU is null. I need the real SKU/options. Let me dig.

  

*Time to respond: 12s*

  

## AI - 21 Sep 2026, 18:05:44

  

REV option-parent pages expose variant SKUs only indirectly. Let me locate the exact data structure.

  

*Time to respond: 6s*

  

## AI - 21 Sep 2026, 18:05:54

  

The variant SKUs live in the option label markup / SKU table. Let me pin the exact pattern.

  

*Time to respond: 5s*

  

## AI - 21 Sep 2026, 18:06:05

  

REV variants are recoverable from the option labels' `(SKU)` suffix. Let me refactor the BigCommerce path to return multiple variant rows cleanly.

  

*Time to respond: 12s*

  

## AI - 21 Sep 2026, 18:06:32

  

Now update the scrape loop to handle descriptors and multi-variant entries.

  

*Time to respond: 4s*

  

## AI - 21 Sep 2026, 18:06:54

  

All branches verified. Now running the full scrape.

  

*Time to respond: 8s*

  

## AI - 21 Sep 2026, 19:04:50

  

Full run complete with zero errors. Let me inspect the outputs and audit the skipped pages to ensure no real products were dropped.

  

*Time to respond: 5s*

  

## AI - 21 Sep 2026, 19:04:59

  

The goBILDA skipped list includes pages that look like real products (option-parent pages). Let me inspect them.

  

*Time to respond: 9s*

  

## AI - 21 Sep 2026, 19:05:17

  

Real goBILDA products (screws, Shark gearboxes, Overlander, hoodie) return **no** structured data at all — only an `<h1>`. Let me find their SKU markup.

  

*Time to respond: 10s*

  

## AI - 21 Sep 2026, 19:05:32

  

goBILDA family pages expose variant SKUs via `data-product-sku`. Let me confirm the surrounding structure and whether names are available.

  

*Time to respond: 6s*

  

## AI - 21 Sep 2026, 19:05:42

  

goBILDA family pages pair `<h4 class="card-title">Variant</h4>` with `data-product-sku="SKU"`. Adding that extraction path.

  

*Time to respond: 13s*

  

## AI - 21 Sep 2026, 19:06:23

  

Variants expand correctly, but variant text leaks into tags (e.g. Overlander → `power` due to "Battery & Charger"). I'll base tags on the parent product name.

  

*Time to respond: 7s*

  

## AI - 21 Sep 2026, 19:06:31

  

Now propagate `tag_basis` through the descriptors, state, and assembly.

  

*Time to respond: 2s*

  

## AI - 21 Sep 2026, 19:08:36

  

goBILDA family variants now captured (2082 rows, 91 SKU overlaps collapsed by dedupe). Recombining all vendors from cache.

  

*Time to respond: 18s*

  

## AI - 21 Sep 2026, 19:08:55

  

Final combine done — 8,081 unique SKUs, zero errors. Now a verification pass.

  

*Time to respond: 5s*

  

## AI - 21 Sep 2026, 19:09:01

  

Verification shows no data-integrity problems. The remaining `misc` items (bushings, pins, springs, clips, keys, tubing) are legitimate but worth new tags. Let me improve coverage.

  

*Time to respond: 9s*

  

## AI - 21 Sep 2026, 19:09:39

  

Two real bugs found: the regexes treat `cap` and `magnet` as merch, causing cap-screws and magnets to be flagged `nonRobot`/`merch`. Fixing and broadening a few categories.

  

*Time to respond: 19s*

  

## AI - 21 Sep 2026, 19:10:36

  

More false positives appearing (`Heat Shrink`, `Banner ... Sensor`). Let me debug the matcher rather than guess.

  

*Time to respond: 7s*

  

## AI - 21 Sep 2026, 19:10:48

  

`Banner` (the sensor brand) collides with the merch rule. Removing it from both patterns.

  

*Time to respond: 5s*

  

## AI - 21 Sep 2026, 19:10:57

  

Now let me verify the REV family-page expansions are legitimate (not category listings).

  

*Time to respond: 3s*

  

## AI - 21 Sep 2026, 19:11:09

  

REV "family" pages are legitimate (ION-Bearings lists 10 real variant SKUs; the parent slug just looks category-like). Re-running final assembly and verification.

  

*Time to respond: 17s*

  

## AI - 21 Sep 2026, 19:11:29

  

Let me audit the last few suspicious goBILDA skips to confirm they're not lost products.

  

*Time to respond: 6s*

  

## AI - 21 Sep 2026, 19:11:57

  

All goBILDA skips are legitimate (redirects/anchors or SKUs already captured). Let me handle repo hygiene for the cache and compiled files.

  

*Time to respond: 4s*

  

## AI - 21 Sep 2026, 19:12:02

  

`.gitignore` has a global `out/` rule, so outputs there would be untracked. I'll move generated data to `tools/catalog/data/` (tracked) and ignore only the resume cache.

  

*Time to respond: 6s*

  

## AI - 21 Sep 2026, 19:12:49

  

Done. Catalog built and verified.

  

## What was produced

- `tools/catalog/scrape_catalog.py` — Python 3 (stdlib only), resumable, re-runnable.

- `tools/catalog/data/parts_all.json` + `parts_all.csv`, per-vendor `parts_{andymark,gobilda,rev}.json`, and `meta.json`.

- Resume cache in `data/.state/` (git-ignored; output data is tracked).

  

**Result: 8,081 unique SKUs — 0 missing SKUs/names, 0 duplicate SKUs, 0 errors.**

| Vendor | Rows | non-robot |

|---|---|---|

| AndyMark | 5,374 | 96 |

| goBILDA | 1,991 | 29 |

| REV | 716 | 3 |

  

Row schema: `sku, name, vendor, url, nonRobot, categories, tags`. `categories` holds the vendor's own breadcrumb/type verbatim; `tags` is a normalized vocabulary (motor, gearbox, servo, bearing, extrusion, wheel, screw, cable, sensor, merch, …).

  

## On your API question

- **AndyMark — yes, a real public API.** It's Shopify: `GET /products.json?limit=250&page=N` (paged, no auth, all ~1,972 products + variants). It also exposes an agent-oriented **UCP/MCP** commerce endpoint at `/api/ucp/mcp` (docs at `/agents.md`), though `products.json` is all a read-only catalog needs.

- **goBILDA & REV — no public API** (BigCommerce; no storefront token). They publish product sitemaps (`/xmlsitemap.php?type=products&page=1`), so the scraper reads each product page's JSON-LD/embedded data. Both hide many SKUs behind **family/option pages** and meta-refresh stub links; the scraper resolves those (e.g. REV Smart Servo → `REV-41-3334`/`REV-41-3336`; goBILDA hoodie → size SKUs).

- **Studica** was out of scope per your request.

  

## Re-running

```

python tools/catalog/scrape_catalog.py                # all vendors, reuses cache

python tools/catalog/scrape_catalog.py --vendor rev --force --delay 1.0

```

  

Notes: REV/goBILDA family variants share the parent page URL (the SKU is the unique key); `nonRobot` follows vendor categories/tags, so a few AndyMark items filed under its "Merchandise" category (e.g. heat shrink) are flagged non-robot. I did not commit anything.

  

*Time to respond: 5s*