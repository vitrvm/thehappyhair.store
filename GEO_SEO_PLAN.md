# GEO/SEO Improvement Plan: The Happy Hair Store

## Overview

A structured data (schema.org) strategy to improve Generative Engine Optimization (GEO) and SEO for the store. All custom logic lives in **new snippet files** with a single render tag added to `layout/theme.liquid`, ensuring Dawn theme upgrades require re-adding only one line of code.

---

## Architecture

```
layout/theme.liquid
  └── {% render 'custom-head-seo' %}              ← ONLY Dawn file edit (1 line after {% render 'meta-tags' %})
        │
        ├── custom-jsonld-organization.liquid       ← Every page
        ├── custom-jsonld-breadcrumbs.liquid         ← Every page
        ├── custom-jsonld-product.liquid             ← Product pages only
        ├── custom-jsonld-collection.liquid          ← Collection pages only
        ├── custom-jsonld-faq.liquid                 ← Pages using page.faq template / articles tagged 'faq'
        ├── custom-jsonld-howto.liquid               ← Articles tagged 'howto'
        └── custom-jsonld-article.liquid             ← Article pages only
```

### Design Principles

1. **All custom logic in new files** — never put complex code in Dawn files
2. **One injection point** — a single `{% render %}` tag in `theme.liquid`
3. **Tag/template-based detection** — no hardcoded page handles or fragile content matching
4. **Supplemental approach** — existing Shopify `{{ product | structured_data }}` stays untouched; our custom schemas supplement, not replace
5. **Dawn upgrade safe** — on upgrade, re-add one line to `theme.liquid`; all custom snippets survive

---

## Current State

| Schema Type | Status | Quality | Location |
|---|---|---|---|
| Organization | **Implemented** | Enhanced — filtered `sameAs`, `@id`, `contactPoint`, `areaServed` | `snippets/custom-jsonld-organization.liquid` |
| WebSite + SearchAction | Present (homepage only) | Fair — missing `@id`, `inLanguage` | `sections/header.liquid:452-467` |
| Product | Present | Fair — uses Shopify built-in filter, lacks ratings/GTIN | `sections/main-product.liquid:735-737` |
| Article | Present | Fair — uses Shopify built-in filter, basic | `sections/main-article.liquid:291-293` |
| BreadcrumbList | **Implemented** | All page types covered | `snippets/custom-jsonld-breadcrumbs.liquid` |
| FAQPage | **Missing** | — | Collapsible-content section exists but has no schema |
| HowTo | **Missing** | — | — |
| CollectionPage / ItemList | **Missing** | — | — |
| AggregateRating | **Missing** | — | Data exists in `product.metafields.reviews.rating` but not in structured data |

---

## Detection Logic

| Schema | Detection Method | Trigger |
|---|---|---|
| Organization | Always (global) | Every page |
| BreadcrumbList | Always (global) | Every page |
| Product + AggregateRating | Page type check | `request.page_type == 'product'` |
| CollectionPage / ItemList | Page type check | `request.page_type == 'collection'` |
| FAQPage | Template name + article tags | Pages assigned to `page.faq` template, or articles tagged `faq` |
| HowTo | Article tags + `<ol>` content parsing | Articles tagged `howto` |
| Enhanced Article | Page type check | `request.page_type == 'article'` |

---

## Files to Create

| File | Purpose | Status |
|---|---|---|
| `snippets/custom-head-seo.liquid` | Master router — checks page type and tags, renders sub-snippets | **Done** |
| `snippets/custom-jsonld-organization.liquid` | Enhanced Organization schema | **Done** |
| `snippets/custom-jsonld-breadcrumbs.liquid` | BreadcrumbList for all pages | **Done** |
| `snippets/custom-jsonld-product.liquid` | Full Product + Offer + AggregateRating | Placeholder |
| `snippets/custom-jsonld-collection.liquid` | CollectionPage / ItemList | Placeholder |
| `snippets/custom-jsonld-faq.liquid` | FAQPage schema | Placeholder |
| `snippets/custom-jsonld-howto.liquid` | HowTo schema | Placeholder |
| `snippets/custom-jsonld-article.liquid` | Enhanced Article / BlogPosting | Placeholder |
| `templates/page.faq.json` | FAQ page template (includes collapsible-content section) | Pending |

## Files Modified

| File | Change | Status |
|---|---|---|
| `layout/theme.liquid` | Added `{% render 'custom-head-seo' %}` after `{% render 'meta-tags' %}` (line 32-33) | **Done** |

---

## Phase 1: Foundation — COMPLETED

**Goal:** Establish the snippet architecture and implement global schemas that apply to every page.

**Status:** All tasks completed and validated. See `PHASE_1_VALIDATION.md` for the full checklist.

**Validation note:** `inLanguage` was removed from Organization schema during validation — it is not a recognized property on `Organization` per schema.org. It will be used on `Article` and `HowTo` schemas in Phase 3 where it is valid.

### Task 1.1 — Create the master SEO snippet — DONE

**File:** `snippets/custom-head-seo.liquid`

This is the central router. It checks the current page type and conditionally renders the appropriate sub-snippets.

**Logic:**

```
1. Always render:
   - custom-jsonld-organization
   - custom-jsonld-breadcrumbs

2. If request.page_type == 'product':
   - render custom-jsonld-product (pass product object)

3. If request.page_type == 'collection':
   - render custom-jsonld-collection (pass collection object)

4. If request.page_type == 'article':
   - render custom-jsonld-article (pass article object)
   - If article.tags contains 'faq':
     - render custom-jsonld-faq (pass article object)
   - If article.tags contains 'howto':
     - render custom-jsonld-howto (pass article object)

5. If request.page_type == 'page':
   - If template name matches 'page.faq':
     - render custom-jsonld-faq (pass page object)
```

### Task 1.2 — Add the render tag to theme.liquid — DONE

**File:** `layout/theme.liquid`

Add one line after the existing `{% render 'meta-tags' %}` on line 30:

```liquid
{% render 'custom-head-seo' %}
```

This is the only modification to an existing Dawn file in the entire plan.

### Task 1.3 — Create the enhanced Organization schema — DONE

**File:** `snippets/custom-jsonld-organization.liquid`

**What it fixes:**
- Filters out empty social links from the `sameAs` array (current version outputs blank strings)
- Uses `request.origin` for the `url` property (current version incorrectly uses `page.url`)

**Properties to include:**

| Property | Source |
|---|---|
| `@type` | `Organization` |
| `@id` | `{{ request.origin }}/#organization` |
| `name` | `{{ shop.name }}` |
| `url` | `{{ request.origin }}` |
| `logo` | `{{ settings.logo | image_url: width: 500 }}` |
| `description` | `{{ shop.description }}` |
| `sameAs` | Filtered array — only non-empty social links from `settings.social_*_link` |
| `contactPoint` | Type: `ContactPoint`, contactType: `customer service`, email from shop settings |
| `areaServed` | `ES` (Spain) — adjust as needed for target markets |

### Task 1.4 — Create the BreadcrumbList schema — DONE

**File:** `snippets/custom-jsonld-breadcrumbs.liquid`

Generates a `BreadcrumbList` based on the current page's position in the site hierarchy.

**Breadcrumb paths by page type:**

| Page Type | Breadcrumb Path |
|---|---|
| Homepage | (no breadcrumb — it's the root) |
| Collection | Home → Collection Title |
| Product | Home → Product's First Collection → Product Title |
| Article | Home → Blog Title → Article Title |
| Page | Home → Page Title |
| Search | Home → Search Results |
| Cart | Home → Cart |

**Properties per `ListItem`:**

| Property | Value |
|---|---|
| `@type` | `ListItem` |
| `position` | Integer (1, 2, 3...) |
| `name` | Page/collection/product title |
| `item` | Full URL of the breadcrumb item |

### Task 1.5 — Validate Phase 1 — DONE

- ~~Test every page type with [Google Rich Results Test](https://search.google.com/test/rich-results)~~
- ~~Verify Organization schema renders on all pages with no empty `sameAs` values~~
- ~~Verify BreadcrumbList renders correctly on product, collection, article, and page types~~
- ~~Verify no duplicate Organization schemas (the existing one in `header.liquid` will coexist — Google merges them, but monitor for warnings)~~

Full validation checklist: `PHASE_1_VALIDATION.md`

---

## Phase 2: Product & Collection — PENDING

**Goal:** Add enhanced Product schema with AggregateRating and CollectionPage schema. These have the most direct revenue impact through rich results (star ratings in search, product carousels).

### Task 2.1 — Create enhanced Product schema

**File:** `snippets/custom-jsonld-product.liquid`

**Receives:** `product` object via `{% render 'custom-jsonld-product', product: product %}`

**Relationship to existing schema:** This supplements the existing `{{ product | structured_data }}` output in `main-product.liquid`. Both JSON-LD blocks will exist on the page. Google merges them using the same `url` as the identifier.

**Properties to include:**

| Property | Source | Notes |
|---|---|---|
| `@type` | `Product` | |
| `@id` | `{{ request.origin }}{{ product.url }}#product` | Unique identifier |
| `name` | `{{ product.title }}` | |
| `description` | `{{ product.description | strip_html | truncate: 5000 }}` | Plain text, truncated |
| `image` | Array of `product.images` URLs | All product images |
| `url` | `{{ request.origin }}{{ product.url }}` | Canonical product URL |
| `brand` | `{ "@type": "Brand", "name": "{{ product.vendor }}" }` | |
| `sku` | `{{ product.selected_or_first_available_variant.sku }}` | |
| `gtin` | `{{ product.selected_or_first_available_variant.barcode }}` | If barcode is present |
| `material` | From product metafield if available | Optional |
| `color` | From product metafield if available | Optional |
| `offers` | See Offer details below | |
| `aggregateRating` | See AggregateRating details below | |

**Offer object:**

| Property | Source |
|---|---|
| `@type` | `Offer` |
| `price` | `{{ variant.price | money_without_currency }}` |
| `priceCurrency` | `{{ cart.currency.iso_code }}` |
| `availability` | `https://schema.org/InStock` or `https://schema.org/OutOfStock` based on `variant.available` |
| `url` | `{{ request.origin }}{{ product.url }}` |
| `seller` | Reference to Organization `@id` |

**AggregateRating object (conditional — only if review data exists):**

| Property | Source |
|---|---|
| `@type` | `AggregateRating` |
| `ratingValue` | `{{ product.metafields.reviews.rating.value.rating }}` |
| `bestRating` | `{{ product.metafields.reviews.rating.value.scale_max }}` |
| `worstRating` | `{{ product.metafields.reviews.rating.value.scale_min }}` |
| `ratingCount` | `{{ product.metafields.reviews.rating_count }}` |

**Condition:** Only output `aggregateRating` if `product.metafields.reviews.rating_count` is present and greater than 0.

### Task 2.2 — Create CollectionPage / ItemList schema

**File:** `snippets/custom-jsonld-collection.liquid`

**Receives:** `collection` object via `{% render 'custom-jsonld-collection', collection: collection %}`

**Properties to include:**

| Property | Source |
|---|---|
| `@type` | `CollectionPage` |
| `@id` | `{{ request.origin }}{{ collection.url }}#collection` |
| `name` | `{{ collection.title }}` |
| `description` | `{{ collection.description | strip_html }}` |
| `url` | `{{ request.origin }}{{ collection.url }}` |
| `mainEntity` | `ItemList` object (see below) |

**ItemList (nested inside CollectionPage):**

| Property | Source |
|---|---|
| `@type` | `ItemList` |
| `numberOfItems` | `{{ collection.products.size }}` |
| `itemListElement` | Array of `ListItem` objects |

**Each ListItem:**

| Property | Source |
|---|---|
| `@type` | `ListItem` |
| `position` | Loop index (1-based) |
| `url` | `{{ request.origin }}{{ product.url }}` |
| `name` | `{{ product.title }}` |

**Limit:** First 20 products to keep the JSON-LD payload reasonable.

### Task 2.3 — Validate Phase 2

- Test product pages with [Google Rich Results Test](https://search.google.com/test/rich-results)
  - Verify star ratings appear in the Product schema when review data exists
  - Verify no conflicts between the Shopify default Product JSON-LD and the custom one
  - Verify `aggregateRating` does NOT appear when a product has zero reviews
- Test collection pages
  - Verify `CollectionPage` with `ItemList` renders correctly
  - Verify product URLs in the list are correct
- Run [Schema Markup Validator](https://validator.schema.org/) for syntax validation

---

## Phase 3: Content Authority (GEO Advantage) — PENDING

**Goal:** Implement FAQPage, HowTo, and enhanced Article schemas. These target the conversational search patterns described in the README's GEO strategy.

### Task 3.1 — Create the FAQ page template

**File:** `templates/page.faq.json`

A new page template that includes the `collapsible-content` section. When a page is assigned to this template in the Shopify Admin, the FAQPage schema activates automatically.

**Template structure:**

```json
{
  "sections": {
    "main": {
      "type": "main-page",
      "settings": {}
    },
    "collapsible-content": {
      "type": "collapsible-content",
      "settings": {}
    }
  },
  "order": ["main", "collapsible-content"]
}
```

**Store admin workflow:** To create an FAQ page:
1. Create a new page in Shopify Admin
2. Assign it the `FAQ` template (this `page.faq.json`)
3. Open the Theme Customizer, navigate to the page, and populate the collapsible-content section with question/answer pairs
4. The FAQPage schema is automatically generated

### Task 3.2 — Create FAQPage schema

**File:** `snippets/custom-jsonld-faq.liquid`

**Triggered by:**
- Pages assigned to the `page.faq` template
- Articles tagged `faq`

**Data extraction approach:**

For **pages** using the `page.faq` template, the snippet needs to access the collapsible-content section's blocks. Shopify sections in JSON templates expose their block data through `content_for_layout`. However, accessing another section's blocks from a snippet rendered in `theme.liquid` is not directly possible via Liquid.

**Alternative approach for pages:** Parse the rendered page content for the FAQ structure. The collapsible-content section renders `<summary>` and `<div class="accordion__content">` elements. However, content parsing in Liquid is limited.

**Recommended approach for pages:** Use a **section-level render** instead. Add the FAQ schema as a dedicated section (`sections/custom-faq-schema.liquid`) that is included in the `page.faq.json` template's section order. This section outputs only the JSON-LD script tag (no visible HTML). It receives the FAQ data from the same collapsible-content section's blocks.

**Revised approach:**
- For pages: Create `sections/custom-faq-schema.liquid` as a hidden section in the `page.faq.json` template. This section has its own `{% schema %}` with FAQ block types (question + answer), so the data is directly accessible.
- For articles: Parse `article.content` for `<h2>`/`<h3>` + following `<p>` pairs tagged as Q&A.

**FAQPage properties:**

| Property | Source |
|---|---|
| `@type` | `FAQPage` |
| `@id` | `{{ request.origin }}{{ page.url }}#faq` |
| `mainEntity` | Array of `Question` objects |

**Each Question object:**

| Property | Source |
|---|---|
| `@type` | `Question` |
| `name` | The question text (from section block or article heading) |
| `acceptedAnswer.@type` | `Answer` |
| `acceptedAnswer.text` | The answer text (from section block or article content) |

### Task 3.3 — Create HowTo schema

**File:** `snippets/custom-jsonld-howto.liquid`

**Receives:** `article` object

**Triggered by:** Articles tagged `howto`

**Data extraction:**
1. `name` — `{{ article.title }}`
2. `description` — `{{ article.excerpt | strip_html }}` or first `<p>` of `article.content`
3. `image` — `{{ article.image }}` (featured image)
4. `step` — Parse `article.content` for the first `<ol>` block, extract each `<li>` as a step

**HowTo properties:**

| Property | Source |
|---|---|
| `@type` | `HowTo` |
| `@id` | `{{ request.origin }}{{ article.url }}#howto` |
| `name` | `{{ article.title }}` |
| `description` | `{{ article.excerpt | strip_html }}` |
| `image` | `{{ article.image | image_url: width: 1200 }}` |
| `inLanguage` | `es` |
| `step` | Array of `HowToStep` objects |

**Each HowToStep:**

| Property | Source |
|---|---|
| `@type` | `HowToStep` |
| `position` | Step number (1-based) |
| `text` | Content of the `<li>` element |

**Content authoring convention:** When writing how-to articles, the author uses an ordered list (`<ol>`) for the steps. This is a natural pattern for instructional content and requires no special markup or metafields.

### Task 3.4 — Create enhanced Article schema

**File:** `snippets/custom-jsonld-article.liquid`

**Receives:** `article` object

**Triggered by:** `request.page_type == 'article'`

**Supplements** the existing `{{ article | structured_data }}` in `main-article.liquid`.

**Additional properties beyond Shopify's default:**

| Property | Source |
|---|---|
| `@type` | `Article` (or `BlogPosting`) |
| `@id` | `{{ request.origin }}{{ article.url }}#article` |
| `inLanguage` | `es` |
| `mainEntityOfPage` | `{{ request.origin }}{{ article.url }}` |
| `wordCount` | Approximate from `article.content | strip_html | split: ' ' | size` |
| `keywords` | `{{ article.tags | join: ', ' }}` |
| `isPartOf` | Reference to the parent blog: `{ "@type": "Blog", "@id": "{{ request.origin }}{{ blog.url }}", "name": "{{ blog.title }}" }` |

### Task 3.5 — Validate Phase 3

- Create a test FAQ page assigned to the `page.faq` template, populate with Q&A pairs, validate with Google Rich Results Test
- Create a test blog article tagged `howto` with an ordered list, validate HowTo schema renders correctly
- Create a test blog article tagged `faq`, validate FAQPage schema renders correctly
- Test an article with both `howto` and `faq` tags to ensure both schemas render without conflict
- Test a regular article (no special tags) to ensure only the enhanced Article schema renders
- Validate all with [Schema Markup Validator](https://validator.schema.org/)

---

## Phase 4: Settings & Polish (Optional) — PENDING

**Goal:** Add theme customizer toggles and perform final validation.

### Task 4.1 — Add custom settings group

**File:** `config/settings_schema.json`

Append a new settings group before the closing `]`:

```json
{
  "name": "Custom GEO/SEO",
  "settings": [
    {
      "type": "checkbox",
      "id": "enable_custom_seo",
      "label": "Enable custom structured data",
      "default": true,
      "info": "Master toggle for all custom JSON-LD schemas"
    },
    {
      "type": "checkbox",
      "id": "enable_faq_schema",
      "label": "Enable FAQPage schema",
      "default": true
    },
    {
      "type": "checkbox",
      "id": "enable_howto_schema",
      "label": "Enable HowTo schema",
      "default": true
    },
    {
      "type": "checkbox",
      "id": "enable_enhanced_product_schema",
      "label": "Enable enhanced Product schema",
      "default": true
    },
    {
      "type": "checkbox",
      "id": "enable_collection_schema",
      "label": "Enable CollectionPage schema",
      "default": true
    }
  ]
}
```

**Note:** This modifies a Dawn file (`settings_schema.json`). On Dawn upgrade, re-add this group at the end of the array.

### Task 4.2 — Wire up settings in the master snippet

Update `snippets/custom-head-seo.liquid` to check `settings.enable_custom_seo` as a master toggle and individual `settings.enable_*` for each schema type.

### Task 4.3 — Full site validation

- Run Google Rich Results Test on every page type:
  - Homepage
  - Product page (with reviews)
  - Product page (without reviews)
  - Collection page
  - FAQ page (page.faq template)
  - Blog article (no tags)
  - Blog article (tagged `faq`)
  - Blog article (tagged `howto`)
  - Generic page
  - Search results
  - Cart
- Run Schema Markup Validator for syntax errors
- Check Google Search Console for structured data reports after indexing
- Verify no console errors in the browser

---

## Dawn Upgrade Checklist

When upgrading to a new version of Dawn, follow these steps:

### Files that survive automatically (no action needed)

| File | Why |
|---|---|
| All `snippets/custom-*.liquid` | New files — Dawn never touches files with `custom-` prefix |
| `templates/page.faq.json` | New file — Dawn never overwrites custom templates |
| `sections/custom-faq-schema.liquid` | New file — Dawn never touches custom sections |

### Files that need re-patching

| File | Action | Effort |
|---|---|---|
| `layout/theme.liquid` | Re-add `{% render 'custom-head-seo' %}` after `{% render 'meta-tags' %}` | 1 line |
| `config/settings_schema.json` | Re-add the "Custom GEO/SEO" settings group at the end of the array (Phase 4 only) | 1 JSON block |

### Verification after upgrade

1. Check that `{% render 'custom-head-seo' %}` is present in `layout/theme.liquid`
2. Visit any page and inspect source for `application/ld+json` blocks
3. Run Google Rich Results Test on a product page and an FAQ page
4. Check Google Search Console for any new structured data errors

---

## Content Authoring Guide

For store content creators to take advantage of the schemas:

### Creating an FAQ page

1. Go to Shopify Admin → Online Store → Pages → Add page
2. Write the page title and any introductory content
3. In the **Theme template** dropdown (right sidebar), select `faq`
4. Save the page
5. Open the Theme Customizer, navigate to the page
6. In the **Collapsible content** section, add question/answer blocks
7. The FAQPage schema is generated automatically

### Creating a how-to article

1. Go to Shopify Admin → Online Store → Blog posts → Add blog post
2. Write the article title and introductory paragraph
3. Write the steps as an **ordered list** (use the numbered list button in the rich text editor)
4. Add the tag `howto` to the article
5. Publish — the HowTo schema is generated automatically

### Creating an FAQ article

1. Write a blog post with Q&A content
2. Add the tag `faq` to the article
3. Publish — the FAQPage schema is generated automatically

---

*Plan created on 2026-02-07*
*Last updated on 2026-02-07 — Phase 1 completed*
