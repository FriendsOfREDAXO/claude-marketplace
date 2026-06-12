---
name: redaxo-metainfo-fields
description: Defining and managing REDAXO metainfo fields (cat_*, art_*, med_*) – the SELECT/RADIO/CHECKBOX `params` separator rules (| between options, : inside an option), the SQL-query escape hatch, the rex_metainfo_add_field idempotency caveat, and filtering categories by metainfo via rex_navigation::addFilter. Use when the user adds a metainfo field, sees broken/extra options in a metainfo dropdown, writes idempotent setup scripts that touch metainfo, or filters navigation by category metainfo.
---

# REDAXO Metainfo Fields

Metainfo fields extend the standard structure tables (`rex_article`, `rex_clang_article`, `rex_category`) with custom columns. They show up in the article/category/media edit forms in the backend and are read with `rex_article::getValue('art_<name>')`, `rex_category::getValue('cat_<name>')`, `rex_media::getValue('med_<name>')`.

## The `params` separator gotcha (SELECT / RADIO / CHECKBOX)

When defining metainfo SELECT/RADIO/CHECKBOX fields via `rex_metainfo_add_field(...)` (or via the backend "Meta Infos" UI), the `params` string uses two separators that are easy to confuse:

- **`|`** separates options
- **`:`** separates a single option's `key:label`

A literal comma is part of the option string — it is **not** a separator.

```php
// ✅ Correct — Yes/No dropdown with empty placeholder
$params = ':–|1:Ja';
//          └┬┘ └┬─┘
//           │   └── option 2: key=1, label="Ja"
//           └────── option 1: key="", label="–"

// ✅ Correct — three options
$params = '1:Tag|2:Woche|3:Monat';

// ❌ Wrong — comma is treated as part of the label
$params = '|–,1|Ja';  // produces 3 broken options: "", "–,1", "Ja"
```

Source of truth: `addons/metainfo/lib/handler/handler.php` — the parser does `explode('|', $params)` then `explode(':', $valueGroup, 2)`.

## SQL-query escape hatch

If `params` parses as a `SELECT ...` query, REDAXO loads options from the DB instead of treating it as a literal list. Useful for dynamic dropdowns:

```php
$params = 'SELECT id, name FROM ' . rex::getTable('team_role') . ' ORDER BY name';
```

The first column becomes the option key, the second becomes the label.

## Function signature

`rex_metainfo_add_field()` has a non-obvious parameter order — the target table is **derived from the field name's prefix** (`art_*` → articles, `cat_*` → categories, `med_*` → media, `clang_*` → languages), there's no explicit table argument.

```php
rex_metainfo_add_field(
    string  $title,         // backend label
    string  $name,          // column name WITH prefix, e.g. 'cat_in_navi_top'
    int     $priority,      // display order in the backend form
    string  $attributes,    // raw input attributes, e.g. 'class="form-control"'
    int     $type,          // type_id — see table below
    string  $default,       // default value (still a string)
    ?string $params   = null, // SELECT/RADIO/CHECKBOX options OR a SELECT-query
    ?string $validate = null,
    string  $restrictions = '',
    ?string $callback = null,
);
```

## Idempotent setup pattern

`rex_metainfo_add_field()` only **adds** — it silently no-ops if a field with that name already exists, and it doesn't update `params`/`default`/etc. on subsequent runs. For setup scripts that should be safe to re-run:

```php
$tableName = rex::getTable('metainfo_field');
$existing  = rex_sql::factory()->getArray(
    'SELECT id, params FROM ' . $tableName . ' WHERE name = :name',
    ['name' => 'cat_in_navi_top']
);

if (empty($existing)) {
    rex_metainfo_add_field(
        'In top navi',         // $title (backend label)
        'cat_in_navi_top',     // $name (must start with cat_/art_/med_/clang_)
        100,                   // $priority
        '',                                  // $attributes
        rex_metainfo_default_type::SELECT,   // $type
        '',                                  // $default
        ':–|1:Ja',             // $params (option list)
    );
} elseif ($existing[0]['params'] !== ':–|1:Ja') {
    // UPDATE the row directly to fix params on an existing field
    rex_sql::factory()
        ->setTable($tableName)
        ->setWhere(['id' => $existing[0]['id']])
        ->setValue('params', ':–|1:Ja')
        ->update();
}
```

## Filtering navigation by category metainfo

`rex_navigation` has `addFilter()` for narrowing the rendered tree to categories matching a metainfo value:

```php
$nav = rex_navigation::factory();
$nav->addFilter('cat_in_navi_top', 1, '=');  // matches stored value '1'
$nav->get(0, 1, true, true);
```

`addFilter` does a strict `==` against `$category->getValue('cat_in_navi_top')` — so the empty option (key `""`) naturally filters out, only categories with value `'1'` pass through.

## Common pitfalls

- Using a comma to separate options – ends up as part of the label, dropdown looks broken.
- Forgetting the leading colon-pipe (`:–|`) for the empty placeholder – the dropdown has no "blank" first option, so categories without a chosen value can't show "(none)".
- Calling `rex_metainfo_add_field()` twice with the same name and expecting the second call to update – it silently no-ops if the field exists.
- Defining a SELECT with a SQL `params` value that returns more than 2 columns – everything past the second column is ignored.
- Filtering navigation with `addFilter('cat_x', 1)` (no operator) when stored values are strings – `1 != '1'` in some configurations. Pass the value as the same type as stored, or use `=` operator.
- Using metainfo for fields that should be a real relation – `be_manager_relation` (via YForm) is more powerful for FK relationships. Reserve metainfo for simple, structure-page-level annotations.

## Type IDs reference

Source of truth: `rex_metainfo_default_type` (`addons/metainfo/lib/default_type.php`). Prefer the constants over the raw integers — they survive any future re-ordering.

| `type_id` | Constant | Field type |
|---|---|---|
| 1 | `rex_metainfo_default_type::TEXT` | Text |
| 2 | `rex_metainfo_default_type::TEXTAREA` | Textarea |
| 3 | `rex_metainfo_default_type::SELECT` | Select |
| 4 | `rex_metainfo_default_type::RADIO` | Radio |
| 5 | `rex_metainfo_default_type::CHECKBOX` | Checkbox |
| 6 | `rex_metainfo_default_type::REX_MEDIA_WIDGET` | Media picker (single) |
| 7 | `rex_metainfo_default_type::REX_MEDIALIST_WIDGET` | Media picker (multi) |
| 8 | `rex_metainfo_default_type::REX_LINK_WIDGET` | Link picker (single) |
| 9 | `rex_metainfo_default_type::REX_LINKLIST_WIDGET` | Link picker (multi) |
| 10 | `rex_metainfo_default_type::DATE` | Date |
| 11 | `rex_metainfo_default_type::DATETIME` | Datetime |
| 12 | `rex_metainfo_default_type::LEGEND` | Legend (display-only separator) |
| 13 | `rex_metainfo_default_type::TIME` | Time |
