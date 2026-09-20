---
name: url-generator-profiles
description: URL addon (tbaddade/redaxo_url) profile setup in REDAXO – mapping a YForm table to pretty URLs via a profile article. Use when the user configures a URL profile, maps rex_yf_* tables to pretty URLs, debugs generated URLs in rex_url_generator_url, asks why brand/product URLs show the wrong path prefix, or works with the URL addon backend page.
---

# URL Generator – Profiles

A **profile** maps one YForm table to pretty URLs. Each row in the table gets a URL generated from one or more column values (segments).

## How it works

```
rex_url_generator_profile
  ├── article_id      → the REDAXO article that handles detail requests
  ├── table_name      → e.g. rex_yf_brands
  └── table_parameters (JSON)
        ├── column_id              → primary key column (usually id)
        ├── column_segment_part_1  → column whose value becomes the URL slug
        └── column_seo_title/description/image → for sitemap/meta

rex_url_generator_url  (generated, do not edit by hand)
  ├── url        → e.g. //www.example.com/brands/acme/
  ├── article_id → routing target (same as profile.article_id)
  ├── data_id    → primary key of the dataset row
  └── url_hash   → SHA1(url)
```

## The article_id dual role — the most common source of confusion

`profile.article_id` controls **two things at once**:

1. **URL base path** — the article's yrewrite URL is prepended to the slug  
   The profile article ("Brand output") has yrewrite URL `/brands/` → generated URL: `/brands/acme/`

2. **Routing target** — incoming requests matching a generated URL are routed to this article

There is no separate "routing article" setting. If you want the routing article to differ from the URL base, use the `URL_PRE_SAVE` hook (see `url-generator-hooks`).

## Setting up a profile (backend)

URL addon → Generator → Profiles → Add profile:

- **Article**: the article that renders the detail view
- **Table**: the YForm table (e.g. `rex_yf_brands`)
- **Segment part 1**: column that becomes the URL slug (e.g. `name`)
- **SEO columns**: `seo_title`, `seo_description`, `logo`
- **Sitemap**: enable + set frequency/priority

## Regenerating URLs

After changing a profile, regenerate from the backend: **URL addon → Generator → URLs → "Alle URLs neu generieren"**.

Do **not** trigger regeneration via CLI (`php redaxo/bin/console cache:clear`). `bin/console` sets `$REX['REDAXO'] = true`, so `rex::isBackend()` is `true` in CLI context. Combined with `developer:sync` (which calls `rex_backend_login::createUser()`), this triggers `CACHE_DELETED` and causes a full URL regeneration — potentially before your `URL_PRE_SAVE` hooks are active (see `url-generator-hooks`).

## Checking what was generated

```sql
SELECT url, article_id, data_id FROM rex_url_generator_url WHERE profile_id = 1;
```

The `url` column uses `//domain/path/` format (scheme-relative). `article_id` must match the article that carries the detail module.

## Getting a dataset URL in PHP

```php
// Via a dataset that exposes getUrl() (YForm models can add such a helper):
$url = $dataset->getUrl();

// Or look the generated urls up by profile namespace:
$urls = \Url\UrlManager::getUrlsByProfile('brands');
```

## Common pitfalls

**Wrong prefix in generated URLs** — The article's yrewrite URL is used as the base path. If the profile article has URL `/brand-output/`, all brand URLs get `/brand-output/acme/`. To generate root-level URLs (`/acme/`), use the `URL_PRE_SAVE` hook (see `url-generator-hooks`).

**Regeneration overwrites manual DB edits** — The `rex_url_generator_url` table is fully replaced on every regeneration. Never edit it permanently by hand; configure profiles or hooks instead.

**Slug encoding** — Segments are normalized with yrewrite's scheme: umlauts transliterated (`ä→ae`), spaces to hyphens, lowercased. `Weishäupl` → `weishaeupl`. `Wo & Wo` → `wo-wo`.
