---
name: mform-flex-repeater
description: Dynamic list / repeater fields in REDAXO modules with MForm – addFlexRepeaterElement(), addRepeaterElement(), nested repeaters, supported widget types inside repeaters (custom-link, media, imagelist, medialist, linklist, radio, checkbox, color-swatch, modal), the __MFRID__ placeholder for unique IDs per row, reading repeater values in OUTPUT PHP with rex_var_custom_medialist / json_decode, and the full list of supported vs. unsupported widget types. Use when the user wants a dynamic list of rows in a module input, uses addRepeaterElement or addFlexRepeaterElement, or asks how to add an image gallery / team list / FAQ with variable rows to a REDAXO module.
---

# MForm Flex Repeater

> **Requires MForm ≥ 9.0.** Items marked **(v9+)** are not available in MForm 8.x (check `rex_addon::get('mform')->getVersion()`).

The Flex Repeater lets editors add, remove and reorder an arbitrary number of rows without Alpine.js (v9+; MForm 8.x only has the Alpine.js-based `addRepeaterElement()`). Data is stored as JSON in a single `REX_VALUE[n]` slot.

## Basic repeater

```php
use FriendsOfRedaxo\MForm;

$mform = MForm::factory();

$mform->addFlexRepeaterElement(1,
    MForm::factory()
        ->addTextField('title',     ['label' => 'Titel'])
        ->addTextAreaField('text',  ['label' => 'Text'])
        ->addMediaField('image',    ['label' => 'Bild', 'preview' => 1])
    ,
    [
        'btn_text'           => 'Eintrag hinzufügen',
        'btn_class'          => 'btn-default',
        'open'               => true,    // rows expanded by default
        'confirm_delete'     => true,
        'min'                => 0,
        'default_count'      => 1,       // pre-fill 1 empty row on first render
    ]
);

echo $mform->show();
```

> **Alias:** `addRepeaterElement()` calls `addFlexRepeaterElement()` internally. Both are equivalent (MForm 9).

## ID conventions inside a repeater

The **outer** repeater itself uses an integer slot ID (`1`, `2`, … = `REX_VALUE[1]`).
The **fields inside the sub-form** use plain string keys – they become array keys in the JSON of each row.

```php
$mform->addFlexRepeaterElement(1,             // outer slot → REX_VALUE[1]
    MForm::factory()
        ->addTextField('headline',  ['label' => 'Überschrift'])  // → $row['headline']
        ->addCustomLinkField('link', ['label' => 'Link'])         // → $row['link']
);
```

No dotted prefix and no row index is needed in the template – the repeater handles row indices automatically at runtime.

## __MFRID__ placeholder (v9+)

For widget-internal IDs that must be unique per row (e.g. TinyMCE, modal IDs), use `__MFRID__` in **HTML attributes** (not in the field ID itself):

```php
->addModalElement(
    'Einstellungen',
    MForm::factory()->addTextField('cssClass', ['label' => 'CSS-Klasse']),
    'btn-default', 'left',
    ['id' => 'modal-__MFRID__']
)
```

The repeater replaces `__MFRID__` with a unique integer for each row.

## Supported widget types inside the repeater

All standard MForm field types work. Special widgets explicitly supported:

| Widget method | Notes |
|---|---|
| `addCustomLinkField()` | Single custom link (intern/extern/media/mailto/tel) |
| `addCustomLinkMultipleField()` | Multiple links as JSON array (v9+) |
| `addMFormLinkField()` | Internal-link-focused custom link (v9+) |
| `addMediaField()` / `addMFormMediaField()` | Single media file from mediapool (`addMFormMediaField()`: v9+) |
| `addMedialistField()` | Multiple media files list |
| `addImagelistField()` | Image gallery with view-toggle (grid/list) |
| `addLinklistField()` | Multiple internal article links |
| `addColorSwatchField()` | Color / CSS-class picker (v9+) |
| `addRadioField()` | Standard radio group |
| `addCheckboxField()` | Checkbox with Bootstrap wrapper |
| `addCheckboxGroupField()` | Multi-checkbox storing comma-separated string (v9+) |
| `addRadioImgField()` | Visual layout picker |
| `addTextReadOnlyField()` | Read-only text display |
| `addTextAreaReadOnlyField()` | Read-only textarea display |
| `addHeadline()` | Static headline inside the row template |
| `addDescription()` | Static hint text inside the row template |
| `addModalElement()` | Sub-form in a Bootstrap modal; uses `__MFRID__` (v9+) |
| `addCollapseElement()` | Collapsible section inside a row |
| `addTextAreaField()` with TinyMCE | Rich text editor; unique instance ID per row via `__MFRID__` |
| MarkdownEditor | Markdown editor; unique instance ID per row via `__MFRID__` |

## Nested repeaters

A repeater can contain another repeater one level deep. The inner sub-key is just a plain string – in the outer row's JSON it becomes an array under that key.

```php
// Inner sub-form
$stepForm = MForm::factory()
    ->addTextField('title', ['label' => 'Schritt-Titel'])
    ->addTextAreaField('body', ['label' => 'Inhalt']);

// Outer sub-form embeds the inner one under sub-key 'steps'
$sectionForm = MForm::factory()
    ->addTextField('section_title', ['label' => 'Abschnittstitel'])
    ->addRepeaterElement('steps', $stepForm, true, true, [
        'btn_text' => 'Schritt hinzufügen',
    ]);

$mform->addRepeaterElement(2, $sectionForm, true, true, [
    'btn_text' => 'Abschnitt hinzufügen',
]);
```

In OUTPUT each section row contains `$section['steps']` as a normal array of step rows.

## Reading repeater values in OUTPUT PHP (v9+)

Use `MFormRepeaterHelper::decode()` – it handles JSON decoding, HTML entity decoding and filters disabled rows in one step.

```php
use FriendsOfRedaxo\MForm\Repeater\MFormRepeaterHelper;

$items = MFormRepeaterHelper::decode('REX_VALUE[2]');

foreach ($items as $item) {
    echo rex_escape($item['title'] ?? '');
}
```

### Filter, sort, group, paginate

```php
use FriendsOfRedaxo\MForm\Repeater\MFormRepeaterHelper;

$items = MFormRepeaterHelper::decode('REX_VALUE[1]');

// Filter by field value
$news = MFormRepeaterHelper::filterByField($items, 'category', 'news');
// Strict comparison (===)
$active = MFormRepeaterHelper::filterByField($items, 'status', '1', strict: true);

// Sort (asc / desc, auto-detects numeric vs. alphabetic)
$sorted = MFormRepeaterHelper::sortByField($items, 'date', 'desc');

// Group → [groupname => [items]]
$grouped = MFormRepeaterHelper::groupByField($items, 'category');

// Pagination
$paged = MFormRepeaterHelper::limitItems($items, perPage: 10, offset: $page * 10);
```

### Custom link values inside repeater rows

```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$items = MFormRepeaterHelper::decode('REX_VALUE[2]');

foreach ($items as $item) {
    $url  = MFormOutputHelper::getCustomUrl($item['link'] ?? '');
    $data = MFormOutputHelper::prepareCustomLink(['link' => $item['link'] ?? ''], true);

    if ($url) {
        echo '<a href="' . rex_escape($url) . '"' . $data['customlink_target'] . '>'
            . rex_escape($data['customlink_text'])
            . '</a>';
    }
}
```

### `decode()` vs `prepareItemsForOutput()`

| Method | When to use |
|---|---|
| `decode(string $rexValue)` | **Recommended** – use directly in module OUTPUT |
| `prepareItemsForOutput(array $items)` | When the array is already decoded (e.g. from a DB query) |

## Medialist / Imagelist values inside a repeater

Medialist and Imagelist store a comma-separated filename string:

```php
$mediaNames = array_filter(explode(',', $item['images'] ?? ''));
foreach ($mediaNames as $filename) {
    $media = rex_media::get($filename);
    if ($media) {
        echo '<img src="' . rex_url::media($filename) . '" alt="' . rex_escape($media->getTitle()) . '">';
    }
}
```

Or use `rex_var_custom_medialist::getMediaOutput()` for the full widget-compatible output.

## Options reference

| Option key | Type | Default | Description |
|---|---|---|---|
| `btn_text` | string | `'Hinzufügen'` | Label on the "add row" button |
| `btn_class` | string | `'btn-default'` | Bootstrap class for the add button |
| `open` | bool | `true` | Rows expanded by default |
| `confirm_delete` | bool | `true` | Show confirmation dialog before deleting a row |
| `min` | int | `0` | Minimum number of rows (rows below this cannot be deleted) |
| `default_count` | int | `0` | Number of empty rows to pre-fill on first render (v9+) |
| `sortable` | bool | `true` | Allow drag-and-drop reordering |

## Common pitfalls

- **Use `0` as the row-index placeholder in IDs**, not the actual index. The repeater renders the template once and patches all IDs at JS runtime.
- **Do not share the slot ID** between a repeater and a regular field in the same module – they would overwrite each other.
- **Nested repeaters only go one level deep** in the current implementation.
- **`__MFRID__` is a string placeholder** – only use it inside attributes (e.g. `id`, `data-*`), not as the field value ID.
- **Always decode repeater values with `MFormRepeaterHelper::decode()`**, not `json_decode()` directly – the helper handles entity decoding and filters disabled rows automatically.
- **TinyMCE and MarkdownEditor inside a repeater are supported** – use `__MFRID__` in the editor's ID attribute to ensure uniqueness per row (e.g. `['id' => 'tinymce-__MFRID__']`).
