# Project Analysis: thehappyhair.store

## Overview

Spanish-language Shopify e-commerce store for curly hair care products, built on Shopify's **Dawn v15.4.0** theme (Online Store 2.0 architecture).

- **Store URL:** `https://t3ebeg-n7.myshopify.com/` (development store)
- **Repository:** `git@github.com:vitrvm/thehappyhair.store.git`
- **Branch:** `main` (5 commits)

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Platform | Shopify (hosted e-commerce) |
| Theme | Dawn 15.4.0 (Shopify's reference theme) |
| Templating | Liquid |
| JavaScript | Vanilla JS + Web Components (Custom Elements) |
| CSS | Vanilla CSS with CSS Custom Properties |
| Build tools | None — no bundler, no preprocessor |
| Deployment | Shopify CLI (`shopify theme push` / `shopify theme dev`) |

---

## Directory Structure

```
├── assets/           # 187 static files (CSS, JS, SVGs, images)
├── config/           # Theme settings (schema + active data + markets)
├── layout/           # 2 layouts (theme.liquid, password.liquid)
├── locales/          # 51 locale files (26 languages, default: English, primary: Spanish)
├── sections/         # 54 section files (.liquid + .json)
├── snippets/         # 37 reusable partials
├── templates/        # 14 page templates + 7 customer account templates
├── zz_db_files/      # Dev-only product export CSV (gitignored)
├── .shopify/         # Shopify CLI metadata + metafield definitions
└── .vscode/          # Editor config (theme-check, Prettier)
```

---

## Key Configuration Files

| File | Purpose | Details |
|------|---------|--------|
| `shopify.theme.toml` | Shopify CLI config | Points to dev store (gitignored) |
| `config/settings_schema.json` | Theme settings schema | Declares Dawn 15.4.0, defines all customizable settings |
| `config/settings_data.json` | Active theme settings | Font: Assistant, page width: 1200px, scroll animations enabled |
| `config/markets.json` | Market-specific settings | Empty — no market overrides configured |
| `.shopify/metafields.json` | Metafield definitions | Rich hair-care product taxonomy |
| `.vscode/setting.json` | Editor config | Auto-save, tab size 2, theme-check linting, Prettier |

---

## Routing / Template Map

| Template | Route | Sections |
|----------|-------|----------|
| `templates/index.json` | `/` | `image-banner` (hero), `featured-collection` ("Vuestros favoritos...") |
| `templates/product.json` | `/products/*` | `main-product` (sticky info, lightbox zoom, stacked gallery), `related-products` |
| `templates/collection.json` | `/collections/*` | `main-collection-banner`, `main-collection-product-grid` (16/page, 4 cols) |
| `templates/cart.json` | `/cart` | `main-cart-items`, `main-cart-footer` |
| `templates/blog.json` | `/blogs/*` | `main-blog` (collage layout) |
| `templates/article.json` | `/blogs/*/articles/*` | `main-article` (featured image, title, share, content) |
| `templates/search.json` | `/search` | `main-search` (4 columns, horizontal filters) |
| `templates/page.json` | `/pages/*` | `main-page` |
| `templates/page.contact.json` | `/pages/contact` | `main-page` + `contact-form` |
| `templates/list-collections.json` | `/collections` | `main-list-collections` |
| `templates/password.json` | Password page | `email-signup-banner` (uses `password` layout) |
| `templates/404.json` | 404 page | `main-404` |
| `templates/gift_card.liquid` | Gift card page | Standalone Liquid template |

### Customer Account Templates

| Template | Route |
|----------|-------|
| `templates/customers/account.json` | `/account` |
| `templates/customers/addresses.json` | `/account/addresses` |
| `templates/customers/login.json` | `/account/login` |
| `templates/customers/register.json` | `/account/register` |
| `templates/customers/order.json` | `/account/orders/*` |
| `templates/customers/activate_account.json` | Account activation |
| `templates/customers/reset_password.json` | Password reset |

### Global Section Groups

- **Header** (`sections/header-group.json`): Announcement bar, dropdown menu, sticky on scroll-up, country/language selectors, customer avatar
- **Footer** (`sections/footer-group.json`): Newsletter ("Recibe ofertas"), social links, Follow on Shop, payment icons, policy links

---

## Architecture Patterns

### JavaScript (32 files)

- **Vanilla Web Components** via `customElements.define()` — base classes defined in `global.js` (1,332 lines)
- **Pub/Sub event system** (`pubsub.js`) for decoupled communication between components
- **Events:** `cart-update`, `quantity-update`, `option-value-selection-change`, `variant-change`, `cart-error`
- **Key modules:** `cart-drawer.js`, `product-info.js`, `product-form.js`, `facets.js`, `predictive-search.js`, `quick-order-list.js`, `media-gallery.js`
- All scripts loaded with `defer`. No bundling, no framework.

### CSS (65 files)

- Pure vanilla CSS with extensive CSS Custom Properties for theming
- Variables dynamically generated in `theme.liquid` from Liquid settings (colors, spacing, typography, shadows, borders, radii)
- Component naming convention: `component-*.css`, `section-*.css`, `template-*.css`
- Responsive breakpoint at 750px
- No preprocessor (Sass/SCSS), no Tailwind, no CSS-in-JS

### State Management

- Lightweight pub/sub pattern (no Redux, Zustand, etc.)
- Cart state via Shopify's AJAX Cart API (`routes.cart_add_url`, `routes.cart_change_url`, `routes.cart_update_url`)
- Cart type: drawer (AJAX slide-out) or full page (configurable)

---

## Product Taxonomy & Metafields

Rich hair-care-specific product metafields (all in Spanish):

| Metafield | Namespace | Type |
|-----------|-----------|------|
| Google: Custom Product | `mm-google-shopping` | boolean |
| Grupo de edad (Age Group) | `shopify` | list.metaobject_reference |
| Sexo objetivo (Target Gender) | `shopify` | list.metaobject_reference |
| Certificaciones y estándares | `shopify` | list.metaobject_reference |
| Forma del producto (Product Form) | `shopify` | list.metaobject_reference |
| Ingredientes constitutivos | `shopify` | list.metaobject_reference |
| Adecuado para tipo de cabello | `shopify` | list.metaobject_reference |
| Características de útiles de peluquería | `shopify` | list.metaobject_reference |
| Color | `shopify` | list.metaobject_reference |
| Acabado para el cuidado capilar | `shopify` | list.metaobject_reference |
| Tipo de champú | `shopify` | list.metaobject_reference |
| Tipo de cabello (Hair Type) | `shopify` | list.metaobject_reference |
| Efecto acondicionador | `shopify` | list.metaobject_reference |
| Nivel de fijación (Hold Level) | `shopify` | list.metaobject_reference |
| Tipo de tratamiento | `shopify` | list.metaobject_reference |
| Tejido (Fabric) | `shopify` | list.metaobject_reference |

**Variant-level metafields (Google Shopping):** Custom Labels 0–4, Size System, Size Type, MPN, Gender, Condition, Age Group — all under `mm-google-shopping` namespace.

---

## Integrations

| Integration | Status |
|-------------|--------|
| Google Shopping / Merchant Center | Active (via `mm-google-shopping` metafields) |
| Google Search Console | Verified (meta tag in `theme.liquid`) |
| Analytics (GA, Pixel, Klaviyo) | Not hard-coded — managed via Shopify Admin / app embeds |
| Review apps (Judge.me, Loox, Yotpo) | Not detected — product ratings supported but disabled |
| Shopify App Blocks | Supported via `sections/apps.liquid` |

---

## SEO & Structured Data

### Currently Implemented

- **JSON-LD:** Organization, WebSite with SearchAction, Product, Article
- **Open Graph + Twitter Card** meta tags via `snippets/meta-tags.liquid`
- **Google Search Console** verification

### Planned (Not Yet Implemented)

Per the GEO strategy document in `README.md`:

- FAQPage structured data
- HowTo structured data
- Enhanced Generative Engine Optimization

---

## Internationalization

- **26 languages** supported via locale files in `locales/`
- **Default locale:** English (`en.default.json`)
- **Primary customer-facing language:** Spanish (all store copy is in Spanish)
- Country and language selectors enabled in header and footer

---

## Deployment & Infrastructure

| Aspect | Detail |
|--------|--------|
| Hosting | Shopify-hosted (no separate infrastructure) |
| Deployment | Automated via GitHub Actions + Shopify CLI |
| Git remote | `git@github.com:vitrvm/thehappyhair.store.git` (SSH) |
| Branches | `main` (production) + feature branches via PRs |
| CI/CD | GitHub Actions (Theme Check + auto-deploy) |
| Docker | None |

### CI/CD Pipeline

```
Feature Branch ──push──> GitHub
                           │
                    ┌──────┴──────┐
                    │ Theme Check  │  (lint on every push)
                    └──────┬──────┘
                           │
                    Open Pull Request
                           │
                    ┌──────┴──────┐
                    │ Theme Check  │  (lint)
                    └──────┬──────┘
                           │
                     Merge to main
                           │
                    ┌──────┴──────┐
                    │ Theme Check  │  (pre-deploy lint)
                    │   Deploy     │  (shopify theme push)
                    └──────┬──────┘
                           │
                     Live on Store
```

**Workflows:**
- `.github/workflows/theme-check.yml` — Runs Shopify Theme Check on all pushes and PRs
- `.github/workflows/deploy.yml` — Deploys to Shopify on push to `main` (with pre-deploy lint gate)

**Required GitHub Secrets:**

| Secret | Used By | Purpose |
|--------|---------|---------|
| `SHOPIFY_CLI_THEME_TOKEN` | Deploy | Theme Access app password |
| `SHOPIFY_FLAG_STORE` | Deploy | Store URL (`t3ebeg-n7.myshopify.com`) |

**Required GitHub Variables:**

| Variable | Used By | Purpose |
|----------|---------|---------|
| `SHOPIFY_THEME_ID` | Deploy | Target theme ID for deployment |

---

## Migration History

Product descriptions contain **WooCommerce/Elementor HTML remnants**, indicating the store was migrated from a WordPress/WooCommerce platform. Legacy markup cleanup is pending.

---

## Gaps & Opportunities

1. **Structured data gaps** — FAQPage and HowTo schema outlined in README but not implemented
2. **No analytics in theme** — likely managed via Shopify Admin, but worth verifying
3. **No review system** — product ratings supported by Dawn but currently disabled (`show_rating: false`)
4. **Legacy HTML cleanup** — WooCommerce/Elementor markup in product descriptions needs sanitization
5. ~~**No CI/CD pipeline**~~ — **Implemented:** Theme Check + auto-deploy via GitHub Actions
6. **Empty TODO.md** — no tracked work items
7. **Markets not configured** — `config/markets.json` is empty despite multi-language support
8. **GEO strategy pending** — comprehensive plan exists in README but zero implementation

---

## Environment & Secrets

| File | Status |
|------|--------|
| `.env` | Gitignored — not present in repo |
| `shopify.theme.toml` | Present but gitignored (contains dev store URL, no secrets) |
| `.shopify/` | Directory gitignored (auth tokens stored here by CLI) |

> **Note:** Google Search Console verification code is hard-coded in `layout/theme.liquid` (publicly visible, not a secret).

---

*Analysis generated on 2026-02-07*
