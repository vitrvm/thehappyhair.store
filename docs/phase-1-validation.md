# Phase 1 Validation Checklist

## Prerequisites

- Local dev server running (`shopify theme dev`) or unpublished theme deployed
- Preview URL available (the `https://*.myshopify.com` URL, not `localhost`)

---

## Testing Tools

| Tool | URL | Purpose |
|---|---|---|
| Google Rich Results Test | https://search.google.com/test/rich-results | Validates structured data and checks rich result eligibility |
| Schema Markup Validator | https://validator.schema.org/ | Strict JSON-LD syntax validation |

---

## Pages to Test

| # | Page Type | URL Path | Expected Schemas |
|---|---|---|---|
| 1 | Homepage | `/` | Organization only (no BreadcrumbList) |
| 2 | Product page | `/products/any-product` | Organization + BreadcrumbList (Home → Collection → Product) |
| 3 | Collection page | `/collections/any-collection` | Organization + BreadcrumbList (Home → Collection) |
| 4 | Generic page | `/pages/any-page` | Organization + BreadcrumbList (Home → Page Title) |
| 5 | Search results | `/search?q=test` | Organization + BreadcrumbList (Home → Search) |
| 6 | Cart | `/cart` | Organization + BreadcrumbList (Home → Cart) |

---

## Organization Schema Checklist

Test on any page. The enhanced Organization schema should render alongside the existing Dawn default.

| # | Check | Expected | Status |
|---|---|---|---|
| 1.1 | `@context` is `https://schema.org` | HTTPS, not HTTP | [x] |
| 1.2 | `@id` ends with `/#organization` | Present and correct | [x] |
| 1.3 | `name` matches shop name | `"The Happy Hair"` or equivalent | [x] |
| 1.4 | `url` is just the origin | No page path appended (e.g., `https://store.com`, not `https://store.com/products/...`) | [x] |
| 1.5 | `description` is present | Shop description, non-empty | [x] |
| 1.6 | `logo` is an `ImageObject` | Has `url` and `width` properties | [x] |
| 1.7 | `sameAs` contains only non-empty links | Instagram + TikTok only, no empty strings `""` | [x] |
| 1.8 | `contactPoint` is present | Type `ContactPoint`, has `email` and `contactType` | [x] |
| 1.9 | `areaServed` is `ES` | Present | [x] |
| 1.10 | ~~`inLanguage` is `es`~~ | ~~Present~~ | [x] Removed — not valid on Organization |
| 1.11 | No JSON syntax errors | Valid JSON (no trailing commas, unclosed brackets) | [x] |
| 1.12 | No Google Rich Results warnings | Clean validation | [x] |

---

## BreadcrumbList Schema Checklist

### Homepage (should NOT have BreadcrumbList)

| # | Check | Expected | Status |
|---|---|---|---|
| 2.1 | BreadcrumbList absent on homepage | No `BreadcrumbList` JSON-LD block | [x] |

### Product Page

| # | Check | Expected | Status |
|---|---|---|---|
| 2.2 | BreadcrumbList is present | `@type: BreadcrumbList` exists in page source | [x] |
| 2.3 | First item is Home | Position 1, name is "Inicio" or "Home", URL is site root | [x] |
| 2.4 | Second item is the collection | Position 2, name is the collection title, URL is the collection URL | [x] |
| 2.5 | Third item is the product | Position 3, name is the product title, URL is the product URL | [x] |
| 2.6 | Positions are sequential | 1, 2, 3 (no gaps, no duplicates) | [x] |
| 2.7 | All URLs are fully qualified | Start with `https://` | [x] |
| 2.8 | No JSON syntax errors | Valid JSON | [x] |

### Product Page (no collection)

| # | Check | Expected | Status |
|---|---|---|---|
| 2.9 | Falls back to 2-level breadcrumb | Home → Product (no collection in between) | [x] |

### Collection Page

| # | Check | Expected | Status |
|---|---|---|---|
| 2.10 | BreadcrumbList is present | 2 items: Home → Collection Title | [x] |
| 2.11 | Collection name and URL are correct | Matches the actual collection | [x] |

### Generic Page

| # | Check | Expected | Status |
|---|---|---|---|
| 2.12 | BreadcrumbList is present | 2 items: Home → Page Title | [x] |
| 2.13 | Page name and URL are correct | Matches the actual page | [x] |

### Search Results

| # | Check | Expected | Status |
|---|---|---|---|
| 2.14 | BreadcrumbList is present | 2 items: Home → Search | [x] |
| 2.15 | Search label is translated | "Buscar" or localized equivalent (not hardcoded English) | [x] |

### Cart

| # | Check | Expected | Status |
|---|---|---|---|
| 2.16 | BreadcrumbList is present | 2 items: Home → Cart | [x] |
| 2.17 | Cart label is translated | "Carrito" or localized equivalent | [x] |

---

## Duplicate Schema Check

View page source (`Cmd+U` / `Ctrl+U`) on the homepage and search for `"@type": "Organization"`.

| # | Check | Expected | Status |
|---|---|---|---|
| 3.1 | Two Organization schemas exist | One from `header.liquid` (Dawn default) + one from `custom-jsonld-organization.liquid` | [x] |
| 3.2 | New schema has `@id` property | The custom one has `/#organization` | [x] |
| 3.3 | Old schema still has empty `sameAs` | Expected — we did not modify `header.liquid` | [x] |
| 3.4 | Both are valid JSON | No broken syntax in either block | [x] |

---

## How to Test

### Using Google Rich Results Test

1. Go to https://search.google.com/test/rich-results
2. Enter the preview URL for each page listed above
3. Click "Test URL"
4. Review the detected structured data types
5. Check for errors and warnings
6. Mark the corresponding checkboxes above

### Using Schema Markup Validator

1. Go to https://validator.schema.org/
2. Select "Fetch URL" and enter the preview URL
3. Review each detected schema type
4. Check for syntax errors or unknown properties
5. Mark the corresponding checkboxes above

### Using View Source

1. Navigate to the page in your browser
2. Right-click → "View Page Source" (or `Cmd+U`)
3. Search for `application/ld+json`
4. Verify each JSON-LD block is well-formed
5. Check for empty strings, trailing commas, or null values

---

## Common Issues and Fixes

| Issue | Likely Cause | Fix |
|---|---|---|
| Trailing comma in JSON | Liquid conditional block outputs comma before a skipped property | Adjust comma placement in the snippet |
| Empty `sameAs` array `[]` | All social links are blank | Snippet should skip `sameAs` entirely when no links are configured |
| `null` or `""` values in properties | Liquid variable is blank | Add `blank` checks before outputting the property |
| BreadcrumbList on homepage | `request.page_type` check missing | Verify `unless request.page_type == 'index'` wraps the output |
| Missing collection in product breadcrumb | Product not assigned to any collection | Expected fallback: Home → Product (2 levels) |
| Google cannot access preview URL | Store is password-protected | Use the store password or test with "Code" tab (paste HTML source directly) |

---

## Result Summary

| Area | Total Checks | Passed | Failed | Notes |
|---|---|---|---|---|
| Organization Schema | 12 | 12 | 0 | `inLanguage` removed — not valid on Organization type |
| BreadcrumbList Schema | 17 | 17 | 0 | |
| Duplicate Schema Check | 4 | 4 | 0 | |
| **Total** | **33** | **33** | **0** | |

---

**Date tested:** 2026-02-07
**Tested by:** vitrvm
**Preview URL used:** Unpublished theme preview
**Status:** [x] All passed / [ ] Issues found (see notes)
