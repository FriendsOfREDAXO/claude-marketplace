---
name: mform-yform
description: Using MForm-provided YForm value types in YForm table definitions and YForm forms – custom_link, custom_link_multi, color_swatch, imagelist, medialist, linklist value types. Covers how to add these fields programmatically (rex_yform_manager_table_api::setTableField) and via tableset JSON, their stored value formats, how to read values via YOrm datasets, and ytemplates used for rendering in the backend. Use when the user adds a custom_link, color_swatch, imagelist, medialist, or linklist field to a YForm table, uses these field types in a YForm form, asks how to render MForm widget values stored in a YForm database column, or says "MForm-Feldtyp in YForm-Tabelle", "Custom-Link-Spalte", "Color-Swatch im Backend", "Galerie-Feld", "Datei-Liste".
---

# MForm YForm Value Types

MForm registers custom value types that extend the YForm field palette. They appear in the YForm Manager field type dropdown once MForm is installed.

---

## Available YForm value types

| Type name | Description | Stored as |
|---|---|---|
| `custom_link` | Single custom link picker (intern/extern/media/mailto/tel) | String (`redaxo://12`, `https://…`, filename, `mailto:…`) |
| `custom_link_multi` | Multiple links JSON array | JSON string (`["redaxo://12","https://…"]`) |
| `color_swatch` | Color/CSS-class picker with swatches popup | String (hex `#2f77bc` or CSS class `.bg-primary`) |
| `imagelist` | Image gallery picker (comma-separated filenames) | `img1.jpg,img2.png` |
| `medialist` | Multi-file media picker (comma-separated filenames) | `file1.pdf,file2.pdf` |
| `linklist` | Multiple internal article links (comma-separated IDs) | `12,14,22` |

## Tableset JSON ↔ PHP definition

The three ways to define a field — `rex_yform_manager_table_api::setTableField()`, `$yform->setValueField()`, and a Tableset JSON export — share the same key set. To go from the PHP form (used throughout this skill) to a Tableset JSON entry:

1. **Add `db_type`.** Use `text` for everything except `color_swatch`, which fits in `varchar(191)`.
2. **Add `table_name`** with the full table including the `rex_` prefix (`"rex_my_table"`).
3. **Stringify every value.** Booleans become `"0"`/`"1"`, integers become `"3"`. JSON-in-JSON values (`color_swatch.swatches`) need their inner quotes escaped (`"swatches": "{\"#fff\":\"Weiß\"}"`).

The Tableset JSON blocks below are shown for `custom_link` and `color_swatch` as concrete references (the quoting edge cases differ). For the other four types, take the PHP definition and apply the three rules above.

---

## custom_link

### Programmatic definition (`install.php`)

```php
rex_yform_manager_table_api::setTableField(rex::getTable('my_table'), [
    'type_id'        => 'value',
    'type_name'      => 'custom_link',
    'name'           => 'cta_link',
    'label'          => 'CTA Link',
    'intern'         => 1,
    'external'       => 1,
    'media'          => 1,
    'mailto'         => 1,
    'phone'          => 0,
    'anchor'         => 0,
    'prio'           => 3,
    'not_required'   => 1,
]);
```

### Tableset JSON

```json
{
    "type_id": "value",
    "type_name": "custom_link",
    "name": "cta_link",
    "label": "CTA Link",
    "db_type": "text",
    "not_required": "1",
    "intern": "1",
    "external": "1",
    "media": "1",
    "mailto": "1",
    "phone": "0",
    "anchor": "0",
    "table_name": "rex_my_table"
}
```

### Via PHP form builder (`$yform->setValueField`)

```php
$yform->setValueField('custom_link', [
    'name'     => 'cta_link',
    'label'    => 'CTA Link',
    'intern'   => 1,
    'external' => 1,
    'media'    => 1,
    'mailto'   => 1,
    'phone'    => 0,
    'anchor'   => 0,
]);
```

### Reading values in PHP

```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$dataset = MyTable::get($id);
$value = $dataset->getValue('cta_link');
$url  = MFormOutputHelper::getCustomUrl($value);
$data = MFormOutputHelper::prepareCustomLink(['link' => $value], true);

if ($url) {
    echo '<a href="' . rex_escape($url) . '"' . $data['customlink_target'] . '>'
        . rex_escape($data['customlink_text']) . '</a>';
}
```

---

## custom_link_multi

### Programmatic definition (`install.php`)

```php
rex_yform_manager_table_api::setTableField(rex::getTable('my_table'), [
    'type_id'      => 'value',
    'type_name'    => 'custom_link_multi',
    'name'         => 'links',
    'label'        => 'Links',
    'intern'       => 1,
    'external'     => 1,
    'media'        => 1,
    'mailto'       => 1,
    'phone'        => 0,
    'anchor'       => 0,
    'btn_add'      => 'Link hinzufügen',
    'prio'         => 4,
    'not_required' => 1,
]);
```

### Tableset JSON

Same shape as `custom_link` plus the `btn_add` key — apply the translation rules above. `db_type: "text"`.

### Reading values in PHP

The value is stored as a JSON array directly in the database (no HTML-entity encoding), so `json_decode()` alone is sufficient:

```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$raw   = $dataset->getValue('links');
$links = json_decode($raw, true) ?? [];

foreach ($links as $linkStr) {
    $url  = MFormOutputHelper::getCustomUrl($linkStr);
    $data = MFormOutputHelper::prepareCustomLink(['link' => $linkStr], true);
    echo '<a href="' . rex_escape($url) . '"' . $data['customlink_target'] . '>'
        . rex_escape($data['customlink_text']) . '</a>';
}
```

---

## color_swatch

### Programmatic definition (`install.php`)

```php
rex_yform_manager_table_api::setTableField(rex::getTable('my_table'), [
    'type_id'   => 'value',
    'type_name' => 'color_swatch',
    'name'      => 'bg_color',
    'label'     => 'Hintergrundfarbe',
    'swatches'  => '{"#ffffff":"Weiß","#000000":"Schwarz",".bg-primary":{"label":"Primär","preview":"#2f77bc"}}',
    'default'   => '#ffffff',
    'prio'      => 5,
    'not_required' => 1,
]);
```

### Tableset JSON

```json
{
    "type_id": "value",
    "type_name": "color_swatch",
    "name": "bg_color",
    "label": "Hintergrundfarbe",
    "db_type": "varchar(191)",
    "not_required": "1",
    "swatches": "{\"#ffffff\":\"Weiß\",\"#000000\":\"Schwarz\",\".bg-primary\":{\"label\":\"Primär\",\"preview\":\"#2f77bc\"}}",
    "default": "#ffffff",
    "table_name": "rex_my_table"
}
```

> `swatches` is a **JSON object** where the key is the stored value (hex or `.css-class`) and the value is either a label string or `{"label":"…","preview":"#hex"}`.

### Reading values in PHP

```php
$color = $dataset->getValue('bg_color');
if (str_starts_with($color, '.')) {
    echo '<div class="' . rex_escape(ltrim($color, '.')) . '">';
} else {
    echo '<div style="background-color:' . rex_escape($color) . '">';
}
```

---

## imagelist

### Programmatic definition (`install.php`)

```php
rex_yform_manager_table_api::setTableField(rex::getTable('my_table'), [
    'type_id'   => 'value',
    'type_name' => 'imagelist',
    'name'      => 'gallery',
    'label'     => 'Galerie',
    'types'     => 'jpg,jpeg,png,webp,avif',
    'category'  => '',
    'prio'      => 6,
    'not_required' => 1,
]);
```

### Tableset JSON

Apply the translation rules above to the PHP definition. `db_type: "text"`.

### Reading values in PHP

```php
$filenames = array_filter(explode(',', $dataset->getValue('gallery') ?? ''));
foreach ($filenames as $filename) {
    echo '<img src="' . rex_url::media($filename) . '" alt="">';
}
```

---

## medialist

### Programmatic definition (`install.php`)

```php
rex_yform_manager_table_api::setTableField(rex::getTable('my_table'), [
    'type_id'      => 'value',
    'type_name'    => 'medialist',
    'name'         => 'downloads',
    'label'        => 'Downloads',
    'types'        => 'pdf,doc,docx',
    'category'     => '',
    'view'         => 'list',
    'views'        => 'list,grid',
    'toolbar'      => 'horizontal',
    'prio'         => 7,
    'not_required' => 1,
]);
```

### Tableset JSON

Apply the translation rules above to the PHP definition. `db_type: "text"`.

### Reading values in PHP

```php
$filenames = array_filter(explode(',', $dataset->getValue('downloads') ?? ''));
foreach ($filenames as $filename) {
    $media = rex_media::get($filename);
    if ($media) {
        echo '<a href="' . rex_url::media($filename) . '">' . rex_escape($media->getTitle()) . '</a>';
    }
}
```

---

## linklist

### Programmatic definition (`install.php`)

```php
rex_yform_manager_table_api::setTableField(rex::getTable('my_table'), [
    'type_id'      => 'value',
    'type_name'    => 'linklist',
    'name'         => 'related',
    'label'        => 'Verwandte Artikel',
    'category'     => 0,
    'toolbar'      => 'horizontal',
    'prio'         => 8,
    'not_required' => 1,
]);
```

### Tableset JSON

Apply the translation rules above to the PHP definition. `db_type: "text"`.

### Reading values in PHP

```php
$ids = array_filter(explode(',', $dataset->getValue('related') ?? ''));
foreach ($ids as $id) {
    $art = rex_article::get((int) $id);
    if ($art) {
        echo '<a href="' . rex_getUrl($art->getId()) . '">' . rex_escape($art->getName()) . '</a>';
    }
}
```

---

## YForm list view (ytemplates)

MForm ships ytemplates for the YForm Manager list view that render these fields correctly:

- `ytemplates/bootstrap/value.custom_link.tpl.php` – shows link URL + type icon
- `ytemplates/bootstrap/value.custom_link_multi.tpl.php` – shows count + preview of links
- `ytemplates/bootstrap/value.color_swatch.tpl.php` – shows color preview square
- `ytemplates/bootstrap/value.imagelist.tpl.php` – shows thumbnail count
- `ytemplates/bootstrap/value.medialist.tpl.php` – shows file count
- `ytemplates/bootstrap/value.linklist.tpl.php` – shows article count

No configuration needed – MForm registers these templates automatically.

---

## Common pitfalls

- **`custom_link` and `custom_link_multi` use `intern`/`external`/`media`/`mailto`/`phone`/`anchor`** (not `data_intern`/`data_extern`). Values are `0`/`1` (int), not `'enable'`/`'disable'`.
- **`color_swatch` swatches is a JSON object**, not an array. Keys are the stored values (hex or `.css-class`), values are labels or `{"label":"…","preview":"#hex"}`.
- **`imagelist` and `medialist` store comma-separated filenames, not JSON** – use `explode(',', …)`.
- **`linklist` stores plain integer IDs** separated by commas – not `redaxo://` format.
- **`custom_link_multi` stores a JSON array** directly in the DB – use `json_decode($val, true) ?? []` (no `html_entity_decode` needed, unlike in module `REX_VALUE`).
- **Always wrap values with `rex_escape()`** when rendering to HTML to prevent XSS.
- **YForm tableset `db_type` for link/list fields should be `text`** to accommodate long values.
