---
name: mform-widgets
description: Advanced MForm widget fields – Custom-Link (addCustomLinkField, addCustomLinkMultipleField, addMFormLinkField), Media picker (addMediaField, addMFormMediaField), Medialist, Imagelist, Linklist, ColorSwatch (addColorSwatchField). Covers all data-* attributes for controlling which link types are enabled (intern/extern/media/mailto/tel), ylink table links, media type restrictions, preview options, and how each widget stores and returns its value. Use when the user adds a link picker, media picker, image gallery, link list, or color swatch to a REDAXO module or YForm form.
---

# MForm Widgets

> **Written for MForm ≥ 9.0.0** (the version this plugin requires). Items marked **(v9+)** do not exist in MForm 8.x. Runtime check: `rex_version::compare(rex_addon::get('mform')->getVersion(), '9.0.0', '>=')`.

MForm provides advanced picker widgets beyond standard HTML inputs. All work inside modules and inside the Flex Repeater.

> All examples assume `use FriendsOfRedaxo\MForm;` at the top of the file and `$mform = MForm::factory();` already called.

---

## Custom-Link (`addCustomLinkField`)

A unified link picker supporting internal articles, external URLs, media files, mailto and tel links.

```php
$mform->addCustomLinkField(1, [
    'label'               => 'Link',
    'data-intern'         => 'enable',   // show "internal article" tab
    'data-extern'         => 'enable',   // show "external URL" tab
    'data-media'          => 'enable',   // show "media" tab
    'data-mailto'         => 'enable',   // show "e-mail" tab
    'data-tel'            => 'enable',   // show "telephone" tab
    'data-extern-link-prefix' => 'https://',
    'data-link-category'  => 14,         // open link chooser at this category
    'data-media-category' => 1,          // open media pool at this category
    'data-media-type'     => 'jpg,png,gif', // restrict media types
]);
```

### YForm table link

Link to a row in any YForm-managed table:

```php
$ylink = [['name' => 'Länder', 'table' => 'rex_ycountries', 'column' => 'de_de']];
$mform->addCustomLinkField(1, ['label' => 'Land', 'ylink' => $ylink, 'data-intern' => 'disable']);
```

### Stored value format

The widget stores a single string:
- Internal article: `redaxo://12`
- Media file: `filename.jpg`
- External URL: `https://example.com`
- E-mail: `mailto:user@example.com`
- Telephone: `tel:+4912345`
- YForm row: `rex-tablename://42`

Reading in OUTPUT PHP:
```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$link = 'REX_VALUE[1]';
$url  = MFormOutputHelper::getCustomUrl($link);
$data = MFormOutputHelper::prepareCustomLink(['link' => $link], true);
// $data['customlink_url']    → resolved URL (same as $url)
// $data['customlink_text']   → display text (article name / media title / URL)
// $data['customlink_target'] → ' target="_blank" rel="noopener noreferrer"' for external, else ''

if ($url) {
    echo '<a href="' . rex_escape($url) . '"' . $data['customlink_target'] . '>'
        . rex_escape($data['customlink_text']) . '</a>';
}
```

> See the **mform-output** skill for the full `MFormOutputHelper` API including `createLinkData()`, `normalizeLinkData()`, and `normalizeRepeaterItems()`.

---

## Custom-Link Multiple (`addCustomLinkMultipleField`) (v9+)

Multiple links as a JSON array in one value slot.

```php
$mform->addCustomLinkMultipleField(1, [
    'label'       => 'Links',
    'btn_add'     => 'Link hinzufügen',
    'data-intern' => 'enable',
    'data-extern' => 'enable',
    'data-media'  => 'enable',
]);
```

Reading in OUTPUT PHP:

Der Wert aus `REX_VALUE` enthält HTML-Entities – deshalb `html_entity_decode()` vor `json_decode()`:

```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$raw   = html_entity_decode('REX_VALUE[1]', ENT_QUOTES | ENT_HTML5, 'UTF-8');
$links = json_decode($raw, true) ?? [];

foreach ($links as $linkStr) {
    $url  = MFormOutputHelper::getCustomUrl($linkStr);
    $data = MFormOutputHelper::prepareCustomLink(['link' => $linkStr], true);
    echo '<a href="' . rex_escape($url) . '"' . $data['customlink_target'] . '>'
        . rex_escape($data['customlink_text']) . '</a>';
}
```

> When reading from a **YForm dataset** (`$dataset->getValue('links')`), skip `html_entity_decode()` – the value is stored without entities. See the **mform-yform** skill.

---

## MForm Link Field (`addMFormLinkField`) (v9+)

Internal-article-only wrapper around Custom-Link. `data-*` link-type flags go in `$parameter` (2nd); `label` goes in `$attributes` (4th).

```php
$mform->addMFormLinkField(1, ['data-extern' => 'disable', 'data-media' => 'disable'], 5, ['label' => 'Artikel']);
```

---

## Media (`addMediaField` / `addMFormMediaField`)

Single file from the media pool.

```php
// All widget options (label, types, preview, category) go in the 2nd param ($parameter)
$mform->addMediaField(1, ['label' => 'Bild', 'preview' => 1]);
// With type restriction and category
$mform->addMediaField(2, ['label' => 'Grafik', 'preview' => 1, 'types' => 'jpg,png,gif,svg,webp', 'category' => 3]);

// MForm variant (v9+; mform-media, stores filename in REX_VALUE; label in $attributes, data-* in $parameter)
$mform->addMFormMediaField(1, ['types' => 'jpg,png,webp', 'preview' => '1'], null, ['label' => 'Datei']);
```

Reading in OUTPUT PHP:
```php
$filename = 'REX_VALUE[1]';
$media = rex_media::get($filename);
if ($media) {
    echo '<img src="' . rex_url::media($filename) . '" alt="' . rex_escape($media->getTitle()) . '">';
}
```

---

## Medialist (`addMedialistField`)

Multiple media files; stores a comma-separated filename string.

```php
$mform->addMedialistField(1, ['label' => 'Dateien', 'types' => 'pdf,doc,docx']);
// With view options:
$mform->addMedialistField(2, [
    'label'       => 'Galerie',
    'types'       => 'jpg,png,webp',
    'view'        => 'gallery',   // start view: list | grid | gallery (v9+)
    'views'       => 'gallery,grid,list', // (v9+)
    'view_switch' => 1,           // show view-toggle button (default: 1) (v9+)
]);
```

Reading in OUTPUT PHP:
```php
$filenames = array_filter(explode(',', 'REX_VALUE[1]'));
foreach ($filenames as $filename) {
    $media = rex_media::get($filename);
    if ($media) {
        echo '<a href="' . rex_url::media($filename) . '">' . rex_escape($media->getTitle()) . '</a>';
    }
}
```

---

## Imagelist (`addImagelistField`)

Image gallery with view-toggle (grid / list / gallery). Stores a comma-separated filename string.

```php
$mform->addImagelistField(1, ['label' => 'Bildergalerie', 'types' => 'jpg,jpeg,png,webp,gif']);
```

Reading in OUTPUT PHP:
```php
$filenames = array_filter(explode(',', 'REX_VALUE[1]'));
foreach ($filenames as $filename) {
    echo '<img src="' . rex_url::media($filename) . '" alt="">';
}
```

---

## Linklist (`addLinklistField`)

Multiple internal article links; stores a comma-separated list of article IDs.

```php
$mform->addLinklistField(1, ['label' => 'Verwandte Artikel'], 5);
```

Reading in OUTPUT PHP:
```php
$ids = array_filter(explode(',', 'REX_VALUE[1]'));
foreach ($ids as $id) {
    $art = rex_article::get((int) $id);
    if ($art) {
        echo '<a href="' . rex_getUrl($art->getId()) . '">' . rex_escape($art->getName()) . '</a>';
    }
}
```

---

## Native REDAXO link (`addLinkField`)

Stores a single article ID as `REX_LINK`. Output uses the classic `REX_LINK[id=n]` placeholder.

```php
// label and widget config go in 2nd param ($parameter); catId as 3rd param
$mform->addLinkField(1, ['label' => 'Artikel'], 5); // 3rd param = start category ID
```

Reading in OUTPUT PHP:
```php
$id  = (int) 'REX_LINK[id=1]';
$art = rex_article::get($id);
if ($art) {
    echo '<a href="' . rex_getUrl($id) . '">' . rex_escape($art->getName()) . '</a>';
}
```

> Use `addCustomLinkField` when you need external, media or mailto support. Use `addLinkField` when you only need internal articles and want the native `REX_LINK` placeholder.

---

## Switching classic widgets to custom_link rendering (v9+)

By default `addMediaField()` and `addLinkField()` use the classic REDAXO core widget (backward-compatible). You can globally switch them to the modern `custom_link` UI:

```php
// In project/boot.php (affects the whole project):
MForm::useCustomLinkForClassicWidgets(true);
```

When enabled, both fields store their value in `REX_VALUE` (same format as `addCustomLinkField`/`addMFormMediaField`) instead of `REX_MEDIA`/`REX_LINK`. Read the output with `MFormOutputHelper::getCustomUrl()`.

Check the current state:
```php
MForm::isUsingCustomLinkForClassicWidgets(); // bool
```

---

## ColorSwatch (`addColorSwatchField`) (v9+)

Text input + visual color/class picker. Stores either a hex color (`#2f77bc`) or a CSS class name (`.bg-primary`).

```php
$mform->addColorSwatchField(1, [
    '#ffffff' => 'Weiß',
    '#000000' => 'Schwarz',
    '#2f77bc' => 'Primär',
    '.bg-warning' => ['label' => 'Warnung', 'preview' => '#f0ad4e'],  // CSS class with preview color
], ['label' => 'Hintergrundfarbe'], '#ffffff');
```

Reading in OUTPUT PHP:
```php
$color = 'REX_VALUE[1]';
if (str_starts_with($color, '.')) {
    // CSS class
    echo '<div class="' . rex_escape(ltrim($color, '.')) . '">';
} else {
    // Hex value
    echo '<div style="background-color:' . rex_escape($color) . '">';
}
```

---

## Using widgets outside modules

### In own addon backend pages (direct `::getWidget()`) (v9+)

```php
// Medialist widget – standalone, no MForm::factory() needed
echo rex_var_custom_medialist::getWidget(
    'settings_media',   // unique widget ID
    'settings[media]',  // form field name (for POST)
    $savedValue,        // current value (comma-separated filenames)
    ['view' => 'gallery', 'views' => 'gallery,grid,list']
);

// Custom-Link-Multi widget
echo rex_var_custom_link_multi::getWidget(
    'settings_links',
    'settings[links]',
    $savedValue,
    ['intern' => 1, 'external' => 1, 'media' => 1]
);
```

### In `rex_form`

Dedicated `rex_form_widget_*_element` classes are available (MForm 8.x only has `rex_form_widget_customlink_element` and `rex_form_widget_imglist_element`):

```php
// Medialist
$field = $form->addField('', 'media_list', null,
    ['internal::fieldClass' => 'rex_form_widget_mform_medialist_element'], true);
$field->setTypes('jpg,png,pdf');
$field->setView('gallery');

// Linklist
$links = $form->addField('', 'links', null,
    ['internal::fieldClass' => 'rex_form_widget_mform_linklist_element'], true);
$links->setCategoryId(0);

// Custom-Link-Multi
$custom = $form->addField('', 'custom_links', null,
    ['internal::fieldClass' => 'rex_form_widget_mform_custom_link_multi_element'], true);
$custom->setIntern(1);
$custom->setExternal(1);
$custom->setMedia(1);
```

Available `rex_form` field classes: `rex_form_widget_mform_imglist_element`, `rex_form_widget_mform_medialist_element`, `rex_form_widget_mform_linklist_element`, `rex_form_widget_mform_customlink_element`, `rex_form_widget_mform_custom_link_multi_element`.

> Widget assets are auto-loaded in the backend via `boot.php` – no manual asset include needed.

---

## Common pitfalls

- **`data-intern` / `data-extern` / `data-media` default to `enable`** unless explicitly set to `disable`. All tabs are shown unless you restrict them.
- **Medialist and Imagelist store comma-separated filenames**, not JSON. Use `explode(',', $val)`, not `json_decode()`.
- **Linklist stores comma-separated article IDs** as plain integers, not `redaxo://` format.
- **`addMediaField()` (REDAXO native) vs `addMFormMediaField()` (MForm custom)**: both store a filename string. `addMFormMediaField` uses the Custom-Link widget internally and has slightly different UI.
- **ColorSwatch CSS-class values start with `.`** – remember to strip the leading dot when using as an HTML class attribute: `ltrim($val, '.')`.
- **`ylink` on Custom-Link requires YForm** – ensure YForm is available before adding such a field.
