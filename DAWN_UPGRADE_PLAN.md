# Dawn Theme Upgrade Plan

## Overview

This document describes how to merge a new version of the Dawn theme into the customized thehappyhair.store repository while preserving all customizations. The same workflow applies to any future Dawn version upgrade.

### Key Principle

All custom files use the `custom-` prefix. This is a Shopify convention that ensures:
- Dawn never ships files with this prefix, so no conflicts on custom files
- Automated conflict resolution can skip `custom-*` files safely
- Custom files survive any upgrade untouched

---

## Current State

- **Base theme:** Dawn v15.4.0
- **Repository:** `git@github.com:vitrvm/thehappyhair.store.git`

---

## Customization Inventory

Keep this inventory updated as customizations are added or removed.

### Custom Files (survive upgrades automatically — no action needed)

| Type | Files |
|---|---|
| Custom snippets | All `snippets/custom-*.liquid` files |
| Custom sections | All `sections/custom-*.liquid` files (none currently) |
| Custom templates | All `templates/page.*.json` custom templates (none currently) |
| CI/CD | `.github/workflows/deploy.yml`, `.github/workflows/theme-check.yml` |
| Config | `.theme-check.yml`, `.shopifyignore` |
| Docs | `GEO_SEO_PLAN.md`, `PHASE_1_VALIDATION.md`, `PROJECT_ANALYSIS.md`, `DAWN_UPGRADE_PLAN.md`, `TODO.md` |

### Modified Dawn Files (need re-patching after merge)

| File | What was changed |
|---|---|
| `layout/theme.liquid` | Google Search Console meta tag + `{% render 'custom-head-seo' %}` injection |
| `.gitignore` | Expanded from Dawn's 2-line default |

### Store Configuration (keep ours during merge)

| File | Notes |
|---|---|
| `config/settings_data.json` | `"current"` section contains store customizations. `"presets"` section is Dawn's. |
| `sections/header-group.json` | Store header configuration |
| `sections/footer-group.json` | Store footer configuration |
| `templates/*.json` | All templates customized via the Shopify Admin theme editor |

---

## One-Time Setup

Add Dawn as an upstream remote. This only needs to be done once:

```bash
git remote add dawn https://github.com/Shopify/dawn.git
git fetch dawn --tags
```

Verify it was added:

```bash
git remote -v
# Should show:
# dawn    https://github.com/Shopify/dawn.git (fetch)
# dawn    https://github.com/Shopify/dawn.git (push)
# origin  git@github.com:vitrvm/thehappyhair.store.git (fetch)
# origin  git@github.com:vitrvm/thehappyhair.store.git (push)
```

---

## Upgrade Workflow

Replace `vCURRENT` with the current Dawn version and `vX.Y.Z` with the target version throughout.

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

**First-time merge** (no common ancestor with Dawn — repositories have unrelated histories):

```bash
git merge vX.Y.Z --no-commit --allow-unrelated-histories
```

**Subsequent merges** (after the first successful merge, a common ancestor exists):

```bash
git merge vX.Y.Z --no-commit
```

The `--no-commit` flag pauses before committing so you can inspect and resolve conflicts.

> **Note:** The first-time merge will produce many more conflicts than subsequent merges because Git has no common ancestor to compare against. Every file that differs between the two histories will show as a conflict. After the first successful merge commit, subsequent upgrades will be much smoother.

### Step 5: Resolve conflicts

Conflict resolution happens in 4 stages, in this order.

#### 5A: Bulk resolve — accept Dawn's version for all stock files (skip custom files)

This single command resolves the majority of conflicts. It accepts Dawn's version for every conflicted file **except** files prefixed with `custom-`:

```bash
for f in $(grep -rl '<<<<<<' snippets/ sections/ assets/ layout/ locales/ config/ templates/ 2>/dev/null); do
  case "$(basename "$f")" in
    custom-*) echo "Skipping (custom file): $f" ;;
    *) git checkout --theirs "$f" && git add "$f" ;;
  esac
done
```

> **What this does:** Loops through every file with conflict markers. If the filename starts with `custom-`, it skips it. Otherwise, it accepts Dawn's version. This is safe because all custom code lives in `custom-*` files, and all other files are either stock Dawn or store configuration that will be restored in the next steps.

#### 5B: Restore store configuration (keep ours)

After 5A accepted Dawn's version for everything, restore the files that contain store-specific data configured via the Shopify Admin:

```bash
# Section groups (header and footer configuration)
git checkout --ours sections/header-group.json && git add sections/header-group.json
git checkout --ours sections/footer-group.json && git add sections/footer-group.json

# All templates customized via the theme editor
git checkout --ours templates/index.json && git add templates/index.json
git checkout --ours templates/product.json && git add templates/product.json
git checkout --ours templates/collection.json && git add templates/collection.json
git checkout --ours templates/cart.json && git add templates/cart.json
git checkout --ours templates/article.json && git add templates/article.json
git checkout --ours templates/search.json && git add templates/search.json
git checkout --ours templates/password.json && git add templates/password.json
git checkout --ours templates/list-collections.json && git add templates/list-collections.json
```

> **Note:** If you add new custom templates in the future (e.g., `templates/page.faq.json`), they use the `custom-` naming convention at the page level and won't conflict. However, if you customize additional stock templates via the theme editor, add them to this list.

#### 5C: Restore custom config files (keep ours)

These are files that don't exist in stock Dawn or that we've expanded beyond Dawn's defaults:

```bash
git checkout --ours .gitignore && git add .gitignore
git checkout --ours .theme-check.yml && git add .theme-check.yml
git checkout --ours .shopifyignore && git add .shopifyignore
```

#### 5D: Manual merge — files requiring line-by-line resolution

**`layout/theme.liquid`**

Step 5A already accepted Dawn's version. Now re-add the custom lines (see Conflict Resolution Reference below for exact code):

1. Google Search Console meta tag (inside `<head>`, before `<title>`)
2. Custom SEO snippet render tag (after `{% render 'meta-tags' %}`)

**`config/settings_data.json`**

Step 5A already accepted Dawn's version, which **overwrote your store customizations**. Restore the `"current"` section:

```bash
# Show what your version of the file looked like before the merge
git show HEAD:config/settings_data.json > /tmp/settings_data_ours.json
```

Open both files and manually merge:
- Copy the entire `"current"` block from `/tmp/settings_data_ours.json` into `config/settings_data.json`
- Keep Dawn's `"presets"` block as-is
- Validate JSON after resolving (see Step 6)

Alternatively, if Dawn didn't change the `"presets"` section (check with `git diff vCURRENT..vX.Y.Z -- config/settings_data.json`), you can simply keep your entire file:

```bash
git checkout --ours config/settings_data.json && git add config/settings_data.json
```

**`config/settings_schema.json`**

Step 5A already accepted Dawn's version (which includes the updated `theme_version`). If a custom settings group exists (Phase 4 of GEO/SEO plan), re-add it at the end of the JSON array before the closing `]`.

### Step 6: Validate before committing

**Never commit without running these checks.** Unresolved conflict markers will break the theme.

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

**If any check fails:**
- Conflict markers found → resolve the remaining conflicts
- Invalid JSON → fix the JSON syntax (missing commas, unclosed brackets, leftover markers)
- Theme Check errors → fix the Liquid errors

**Only proceed to Step 7 when all checks pass.**

### Step 7: Commit the merge

```bash
git add .
git commit -m "Upgrade Dawn from vCURRENT to vX.Y.Z"
```

### Step 8: Test locally

```bash
shopify theme dev --store t3ebeg-n7.myshopify.com --theme UNPUBLISHED_THEME_ID --password YOUR_THEME_ACCESS_PASSWORD
```

### Step 9: Validate customizations

Run through the post-upgrade checklist below.

### Step 10: Push to staging

```bash
git push -u origin upgrade/dawn-vX.Y.Z
```

Then create a PR to `main`, or merge directly:

```bash
git checkout main
git merge upgrade/dawn-vX.Y.Z
git push
```

The CI/CD pipeline will run Theme Check and deploy to the unpublished theme automatically.

---

## Post-Upgrade Checklist

| # | Check | How to Verify | Status |
|---|---|---|---|
| 1 | No conflict markers in any file | `grep -rl '<<<<<<' templates/ sections/ snippets/ layout/ config/ locales/ assets/` returns nothing | [ ] |
| 2 | All JSON files are valid | JSON validation loop (Step 6) returns no errors | [ ] |
| 3 | Theme Check passes | `shopify theme check --fail-level error` shows no errors | [ ] |
| 4 | Custom SEO render tag present in `theme.liquid` | Search `theme.liquid` for `custom-head-seo` | [ ] |
| 5 | Google Search Console meta tag present in `theme.liquid` | Search `theme.liquid` for `google-site-verification` | [ ] |
| 6 | `config/settings_data.json` has store customizations | Check social links, color schemes, shadow settings in `"current"` block | [ ] |
| 7 | `.gitignore` has all custom exclusions | Verify custom entries are listed | [ ] |
| 8 | All custom snippets present | `ls snippets/custom-*` shows all custom files | [ ] |
| 9 | Organization schema renders | View page source on any page, search for `/#organization` | [ ] |
| 10 | BreadcrumbList schema renders | View page source on a product page, search for `BreadcrumbList` | [ ] |
| 11 | CI/CD pipeline runs successfully | Check GitHub Actions | [ ] |
| 12 | Store renders correctly | Browse homepage, product page, collection page, cart | [ ] |

---

## Conflict Resolution Reference

### `layout/theme.liquid` — Re-patching guide

After accepting Dawn's updated `theme.liquid`, re-add these two customizations:

**1. Google Search Console meta tag (inside `<head>`, before `<title>`):**

```html
<meta name="google-site-verification" content="91oI2jsMdK-7vQz13dl2Dyj767rA1YxkO33QQmKLwS4" />
```

**2. Custom SEO snippet (after `{% render 'meta-tags' %}`):**

```liquid
{%- comment -%} Custom SEO code — Do not remove or edit this line. {%- endcomment -%}
{% render 'custom-head-seo' %}
```

### `config/settings_data.json` — Manual merge guide

The file has two main sections:

```json
{
  "current": {
    // YOUR store data — keep ours
  },
  "presets": {
    "Dawn": {
      // Dawn's defaults — accept theirs
    }
  }
}
```

**Before resolving**, compare what each side changed:

```bash
# What Dawn changed in this file
git diff vCURRENT..vX.Y.Z -- config/settings_data.json

# What your version looks like
git show HEAD:config/settings_data.json
```

**Resolution rules:**
- `"current"` block → always keep ours (your store customizations)
- `"presets"` block → accept Dawn's version (their default preset)
- Top-level structural changes → review carefully

**After resolving, validate:**

```bash
python3 -m json.tool config/settings_data.json > /dev/null && echo "Valid" || echo "Invalid"
```

### `config/settings_schema.json` — Version bump

Dawn bumps the `theme_version` field with every release. Step 5A accepts their version automatically. If a custom settings group exists (Phase 4 of GEO/SEO plan), re-add it at the end of the JSON array before the closing `]`.

---

## Common Pitfalls

| Pitfall | Description | Prevention |
|---|---|---|
| Committing unresolved conflict markers | Conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`) left in files break Liquid and JSON parsing | **Always run Step 6 validation before committing** |
| Overwriting `settings_data.json` `"current"` block | Step 5A accepts Dawn's version, which wipes your store data | Always restore `"current"` in Step 5D |
| Overwriting store templates | Step 5A accepts Dawn's version for templates | Always restore customized templates in Step 5B |
| Overwriting section group JSON files | `header-group.json` and `footer-group.json` contain store configuration | Always restore in Step 5B |
| Forgetting to re-patch `theme.liquid` | Step 5A accepts Dawn's version, removing custom lines | Always re-add in Step 5D |
| Locale conflicts | Translation updates are the most frequent Dawn changes | Step 5A handles these automatically |
| Structural migrations | Major versions may move files (e.g., v15.1.0 moved SVGs from snippets to assets) | Read release notes before upgrading across major versions |
| CSS/JS asset conflicts | If custom styles were added to Dawn's core CSS/JS files | Keep custom CSS/JS in separate `custom-*` files |
| Shopify GitHub sync conflicts | If the Shopify GitHub integration pushes while you're merging | Temporarily disconnect the integration during upgrades |
| Pushing upgrade directly to live | Untested merge goes live immediately | Always test on unpublished theme first |
| First-time merge producing excessive conflicts | `--allow-unrelated-histories` causes every differing file to conflict | Expected — only happens once. Subsequent merges are cleaner. |
| Custom files without `custom-` prefix | Files not using the prefix won't be protected by the Step 5A automation | **Always use the `custom-` prefix for new custom files** |

---

## Version History

| Date | From | To | Notes |
|---|---|---|---|
| — | v15.4.0 | — | Initial Dawn version |
| | | | |

---

*Created on 2026-02-08*
*Last updated on 2026-02-08*
