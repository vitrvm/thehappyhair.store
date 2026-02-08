# Dawn Theme Upgrade Plan

## Overview

This document describes how to merge a new version of the Dawn theme into the customized thehappyhair.store repository while preserving all customizations. The repository is a **fork of Shopify/dawn**, which means merges have shared Git history and produce minimal conflicts.

### Key Principles

1. **Repository is a fork of Dawn** — shared Git history enables clean, standard merges
2. **All custom code uses the `custom-` prefix** — Dawn never ships `custom-*` files, so zero conflicts on custom code
3. **Store configuration is in `.shopifyignore`** — templates, `settings_data.json`, and section groups are never pushed from Git. Shopify Admin is the source of truth. Even if these files conflict during a merge, it only affects the local Git reference copy — the live store is unaffected.
4. **Deploy uses `--nodelete`** — code deploys never delete files from the store

---

## Current State

- **Base theme:** Dawn v15.4.1
- **Repository:** `git@github.com:vitrvm/thehappyhair.store.git` (fork of `Shopify/dawn`)
- **Upstream remote:** `dawn` → `https://github.com/Shopify/dawn.git`

---

## Customization Inventory

Keep this inventory updated as customizations are added or removed.

### Custom Files (survive upgrades automatically — no action needed)

| Type | Files |
|---|---|
| Custom snippets | All `snippets/custom-*.liquid` files |
| Custom sections | All `sections/custom-*.liquid` files (none currently) |
| Custom assets | All `assets/custom-*` files (none currently) |
| CI/CD | `.github/workflows/deploy.yml`, `.github/workflows/theme-check.yml` |
| Config | `.theme-check.yml`, `.shopifyignore` |
| Docs | `GEO_SEO_PLAN.md`, `PHASE_1_VALIDATION.md`, `PROJECT_ANALYSIS.md`, `DAWN_UPGRADE_PLAN.md` |

### Modified Dawn Files (only file requiring re-patching after merge)

| File | What was changed |
|---|---|
| `layout/theme.liquid` | Line 4: Google Search Console meta tag. After `{% render 'meta-tags' %}`: `{% render 'custom-head-seo' %}` injection |

### Store Configuration (protected by `.shopifyignore` — safe during upgrades)

These files are tracked in Git as a **read-only reference** but are never pushed to Shopify. The store's live versions are managed entirely via the Shopify Admin theme editor.

| File | Notes |
|---|---|
| `config/settings_data.json` | Colors, fonts, social links, store customizations |
| `config/markets.json` | Market-specific settings |
| `sections/header-group.json` | Header configuration |
| `sections/footer-group.json` | Footer configuration |
| `templates/*.json` | All page templates (homepage, product, collection, etc.) |
| `templates/customers/*.json` | Customer account templates |

Even if these files conflict during a Dawn merge, the conflict only affects your local Git copy — the live store configuration is unaffected because `.shopifyignore` prevents these files from ever being pushed.

---

## Upgrade Workflow

Replace `vCURRENT` with the current Dawn version and `vX.Y.Z` with the target version.

### Step 1: Fetch latest Dawn tags

```bash
git fetch dawn --tags
```

### Step 2: Review what changed

```bash
# See all commits in the new release
git log vCURRENT..vX.Y.Z --oneline

# See which files changed and how much
git diff vCURRENT..vX.Y.Z --stat

# Read the full diff (optional, for detailed review)
git diff vCURRENT..vX.Y.Z
```

Also read the release notes at: `https://github.com/Shopify/dawn/releases/tag/vX.Y.Z`

### Step 3: Create an upgrade branch

```bash
git checkout main
git pull
git checkout -b upgrade/dawn-vX.Y.Z
```

### Step 4: Merge the Dawn release tag

```bash
git merge vX.Y.Z
```

Since the repository is a fork of Dawn, this is a standard merge with shared history. Conflicts will only occur on files you've actually modified.

**Expected conflicts:**
- `layout/theme.liquid` — likely, since this is the only Dawn file we modify
- `config/settings_schema.json` — possible, if you've added custom settings (Phase 4 of GEO/SEO plan)
- `.gitignore` — possible, if Dawn updates their default

**Files that will NOT conflict:**
- `snippets/custom-*.liquid` — Dawn doesn't have these files
- `templates/*.json`, `config/settings_data.json`, section groups — may auto-merge or conflict, but it doesn't matter since `.shopifyignore` protects the live store

### Step 5: Resolve conflicts

**`layout/theme.liquid`** (most likely conflict)

If conflicted, accept Dawn's version and re-add the 2 custom lines:

```bash
git checkout --theirs layout/theme.liquid
git add layout/theme.liquid
```

Then manually re-add (see Conflict Resolution Reference below for exact code):
1. Google Search Console meta tag (inside `<head>`, before `<meta charset>`)
2. Custom SEO snippet render tag (after `{% render 'meta-tags' %}`)

**`config/settings_schema.json`** (if conflicted)

Accept Dawn's version (contains the updated `theme_version`):

```bash
git checkout --theirs config/settings_schema.json
git add config/settings_schema.json
```

If a custom settings group exists (Phase 4 of GEO/SEO plan), re-add it at the end of the JSON array before the closing `]`.

**`.gitignore`** (if conflicted)

Keep ours:

```bash
git checkout --ours .gitignore
git add .gitignore
```

**Store config files** (if conflicted)

These files are in `.shopifyignore` and never pushed to the store. You can resolve them however you prefer — the live store is unaffected either way. To keep your local reference copy intact:

```bash
git checkout --ours config/settings_data.json && git add config/settings_data.json
git checkout --ours templates/ && git add templates/
git checkout --ours sections/header-group.json && git add sections/header-group.json
git checkout --ours sections/footer-group.json && git add sections/footer-group.json
```

### Step 6: Validate before committing

```bash
# 1. Check for any remaining conflict markers
grep -rl '<<<<<<' templates/ sections/ snippets/ layout/ config/ locales/ assets/

# 2. Validate all JSON files
for f in templates/*.json templates/customers/*.json sections/*.json config/*.json; do
  python3 -m json.tool "$f" > /dev/null 2>&1 || echo "Invalid JSON: $f"
done

# 3. Run Theme Check
shopify theme check --fail-level error
```

**Only proceed when all checks pass.**

### Step 7: Commit the merge

```bash
git add .
git commit -m "Upgrade Dawn from vCURRENT to vX.Y.Z"
```

### Step 8: Test locally

```bash
shopify theme dev --store t3ebeg-n7.myshopify.com --theme THEME_ID --password YOUR_THEME_ACCESS_PASSWORD
```

### Step 9: Push and deploy

```bash
git push -u origin upgrade/dawn-vX.Y.Z
```

Then merge to main (via PR or directly):

```bash
git checkout main
git merge upgrade/dawn-vX.Y.Z
git push
```

The CI/CD pipeline will run Theme Check and deploy to Shopify automatically. The deploy uses `--nodelete` and respects `.shopifyignore`, so store configuration is never touched.

---

## Post-Upgrade: Sync Store Config (Optional)

After upgrading, you may want to update your local Git reference copy of the store configuration to stay in sync:

```bash
shopify theme pull \
  --only config/settings_data.json \
  --only templates/ \
  --only sections/header-group.json \
  --only sections/footer-group.json \
  --store t3ebeg-n7.myshopify.com \
  --theme THEME_ID \
  --password YOUR_THEME_ACCESS_PASSWORD

git add .
git commit -m "Sync store configuration from Shopify Admin"
```

This is purely for reference and backup. These files are never pushed back to the store.

---

## Post-Upgrade Checklist

| # | Check | How to Verify | Status |
|---|---|---|---|
| 1 | No conflict markers in any file | `grep -rl '<<<<<<' templates/ sections/ snippets/ layout/ config/ locales/ assets/` returns nothing | [ ] |
| 2 | All JSON files are valid | JSON validation loop (Step 6) returns no errors | [ ] |
| 3 | Theme Check passes | `shopify theme check --fail-level error` shows no errors | [ ] |
| 4 | Custom SEO render tag present in `theme.liquid` | Search `theme.liquid` for `custom-head-seo` | [ ] |
| 5 | Google Search Console meta tag present in `theme.liquid` | Search `theme.liquid` for `google-site-verification` | [ ] |
| 6 | All custom snippets present | `ls snippets/custom-*` shows all custom files | [ ] |
| 7 | Organization schema renders | View page source on any page, search for `/#organization` | [ ] |
| 8 | BreadcrumbList schema renders | View page source on a product page, search for `BreadcrumbList` | [ ] |
| 9 | CI/CD pipeline runs successfully | Check GitHub Actions | [ ] |
| 10 | Store renders correctly | Browse homepage, product page, collection page, cart | [ ] |

---

## Conflict Resolution Reference

### `layout/theme.liquid` — Re-patching guide

After accepting Dawn's updated `theme.liquid`, re-add these two customizations:

**1. Google Search Console meta tag (inside `<head>`, before `<meta charset>`):**

```html
<meta name="google-site-verification" content="91oI2jsMdK-7vQz13dl2Dyj767rA1YxkO33QQmKLwS4" />
```

**2. Custom SEO snippet (after `{% render 'meta-tags' %}`):**

```liquid
{%- comment -%} Custom SEO code — Do not remove or edit this line. {%- endcomment -%}
{% render 'custom-head-seo' %}
```

### `config/settings_schema.json` — Version bump

Dawn bumps the `theme_version` field with every release. Accept their version. If a custom settings group exists (Phase 4 of GEO/SEO plan), re-add it at the end of the JSON array before the closing `]`.

---

## Common Pitfalls

| Pitfall | Description | Prevention |
|---|---|---|
| Committing unresolved conflict markers | Conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) left in files break Liquid and JSON parsing | **Always run Step 6 validation before committing** |
| Forgetting to re-patch `theme.liquid` | After accepting Dawn's version, the custom lines are gone | Always re-add in Step 5 — the only manual merge step |
| Structural migrations in major versions | Major versions may move files (e.g., v15.1.0 moved SVGs from snippets to assets) | Read release notes before upgrading across major versions |
| CSS/JS asset conflicts | If custom styles were added to Dawn's core CSS/JS files | Keep custom CSS/JS in separate `custom-*` files |
| Custom files without `custom-` prefix | Files not using the prefix may conflict with Dawn's files | **Always use the `custom-` prefix for new custom files** |
| Pushing upgrade directly to live | Untested merge goes live immediately | Test locally with `shopify theme dev` before merging to main |

---

## Version History

| Date | From | To | Notes |
|---|---|---|---|
| 2026-02-08 | — | v15.4.1 | Initial Dawn version (forked from Shopify/dawn) |
| | | | |

---

*Created on 2026-02-08*
*Last updated on 2026-02-08 — Rewritten for fork-based setup with .shopifyignore protection*
