---
name: structure-meta-info
description: Adding custom fields to articles, categories, media, and clang via the REDAXO meta_info plugin. Use when the user adds custom metadata to structure entities (e.g. SEO fields, badges, taxonomies), edits prefix conventions, or programmatically reads meta values.
---

# Meta Info

The `structure/meta_info` plugin (ships with the structure addon) lets you add custom fields to:

- Articles (`art_*` prefix)
- Categories (`cat_*` prefix)
- Media (`med_*` prefix)
- Languages / clang (`clang_*` prefix)

Fields are configured in the backend (Structure → Meta-Information). Each field has a name (matches the SQL column), a label, a type (text, textarea, select, radio, checkbox, media/medialist/link/linklist picker, date, datetime, time, legend), and an optional default. The full list of `type_id` constants lives in `rex_metainfo_default_type` — see the redaxo-metainfo-fields skill for the reference table.

The prefix tells REDAXO which entity it belongs to – `art_my_field` adds a column to `rex_article`, `cat_my_field` adds it to `rex_article` (categories share the table), `med_my_field` extends `rex_media`.

## Reading meta values

```php
// On articles
$article = rex_article::getCurrent();
$value = $article->getValue('art_subtitle');

// On categories
$cat = rex_category::getCurrent();
$value = $cat->getValue('cat_color');

// On media
$media = rex_media::get('hero.jpg');
$copyright = $media->getValue('med_copyright');

// On clang
$clang = rex_clang::getCurrent();
$value = $clang->getValue('clang_currency');
```

`getValue()` returns the raw column value as a string. Cast as needed:

```php
$boolField = (bool) (int) $article->getValue('art_featured');
$intField  = (int) $article->getValue('art_priority_score');
```

## Setting meta values programmatically

Through the article/category service so changes are tracked:

```php
rex_article_service::editArticle($articleId, $clangId, [
    'art_subtitle' => 'A new subtitle',
    'art_featured' => 1,
]);
```

For media, use `rex_media_service::updateMedia()`. For clang, write directly via `rex_sql` (no service wrapper exists for clang updates):

```php
$sql = rex_sql::factory();
$sql->setTable(rex::getTable('clang'));
$sql->setWhere(['id' => $clangId]);
$sql->setValue('clang_currency', 'EUR');
$sql->update();
rex_clang::reset(); // clear in-process clang cache
```

## Adding meta fields programmatically (in install.php)

The meta_info plugin exposes a small API for declaring fields without clicking through the backend. This is the right approach for distributable addons.

The target table is **derived from the field name's prefix** (`art_*` → articles, `cat_*` → categories, `med_*` → media, `clang_*` → languages); there is no explicit table argument. Parameter order is `(title, name, priority, attributes, type, default, params, validate, restrictions, callback)`:

```php
<?php
// install.php
if (rex_plugin::get('structure', 'meta_info')->isAvailable()) {
    rex_metainfo_add_field(
        'Subtitle',                              // $title (backend label)
        'art_subtitle',                          // $name (prefix decides the target table)
        100,                                     // $priority
        '',                                      // $attributes (raw input attrs)
        rex_metainfo_default_type::TEXT,         // $type
        '',                                      // $default
        null,                                    // $params (option list for SELECT/RADIO/CHECKBOX)
        null,                                    // $validate
        '',                                      // $restrictions (role/perm string, '' = visible to all)
    );
}
```

The full constants list is in the redaxo-metainfo-fields skill. For an idempotent re-runnable variant (the function silently no-ops if the field already exists), see the pattern documented there.

To remove a field on uninstall:

```php
<?php
// uninstall.php
if (rex_plugin::get('structure', 'meta_info')->isAvailable()) {
    rex_metainfo_delete_field('art_subtitle');
}
```

## Common SEO meta fields

YRewrite ships these by default; they're already added when YRewrite is installed:

- `art_yrewrite_title` – page title override
- `art_yrewrite_description` – meta description
- `art_yrewrite_index` – `index` / `noindex`
- `art_yrewrite_canonical_url` – canonical override
- `art_yrewrite_image` – social-share image (`REX_MEDIA_WIDGET`)

Don't redefine them in your own addon.

## Filtering by meta in queries

Meta fields are real columns on `rex_article` (or `rex_media`/`rex_clang`), so you can query them with `rex_sql`:

```php
$sql = rex_sql::factory();
$sql->setQuery(
    'SELECT id FROM ' . rex::getTable('article') . '
     WHERE clang_id = :clang
       AND status = 1
       AND art_featured = 1
     ORDER BY priority',
    ['clang' => rex_clang::getCurrentId()]
);
```

For dropdowns (type_id 3) the value stored is the option key, not the label. Decoding labels requires re-reading the field config:

```php
$sql = rex_sql::factory();
$sql->setQuery(
    'SELECT params FROM ' . rex::getTable('metainfo_field') . '
     WHERE name = ?',
    ['art_status']
);
$params = $sql->getValue('params'); // "1,Draft|2,Review|3,Published"
```

## Common pitfalls

- Forgetting the prefix – `subtitle` won't work; it must be `art_subtitle`.
- Adding meta fields with `rex_sql` directly instead of `rex_metainfo_add_field` – the form in the backend doesn't show them.
- Querying by meta value before checking that the field exists across versions – migration scripts should always `ensureColumn()` first.
- Using meta_info for many-to-many data – it's column-per-field. For tags, related entries, etc. use a YForm relation table.
- Treating dropdown values as labels in templates – they're the option keys. Map back to labels via `params` or store labels redundantly.
