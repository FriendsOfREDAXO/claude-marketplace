---
name: mform-output
description: Reading MForm field values in REDAXO module OUTPUT PHP – MFormOutputHelper::createLinkData(), normalizeLinkData(), normalizeRepeaterItems(), getCustomLinkUrl(), getCustomUrl(), prepareCustomLink(), decoding repeater JSON, working with customlink_url / customlink_text / customlink_target / customlink_class / type / article_id / metadata. Use when the user writes the OUTPUT section of a REDAXO module that uses MForm, processes link values, displays repeater rows, or needs to resolve redaxo:// URLs to actual hrefs.
---

# MForm Output Helpers

> **Requires MForm ≥ 9.0.** Items marked **(v9+)** are not available in MForm 8.x (check `rex_addon::get('mform')->getVersion()`).

The `MFormOutputHelper` class (namespace `FriendsOfRedaxo\MForm\Utils`) normalises all link and repeater values for use in frontend templates.

---

## createLinkData() / normalizeLinkData() (v9+)

Unified entry point that accepts **any** link format and returns a consistent array.

```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$data = MFormOutputHelper::createLinkData('REX_VALUE[1]');
```

### Input formats accepted

| Input | Example |
|---|---|
| Internal article (redaxo:// prefix) | `redaxo://12` |
| Internal article (plain numeric ID) | `14` |
| Media filename | `image.jpg` |
| External URL | `https://example.com` |
| E-mail | `mailto:user@example.com` |
| Telephone | `tel:+49123456789` |
| Already-prepared array with `customlink_url` | passed through as-is |
| Repeater array with `id` / `name` keys | `['id' => 'redaxo://12', 'name' => 'Artikel [12]']` |

### Return value keys

```php
[
    'customlink_url'    => 'https://example.com',     // resolved URL ready for href=
    'customlink_text'   => 'Example',                 // display text (article name / media title / URL)
    'customlink_target' => ' target="_blank" rel="noopener noreferrer"', // for external links
    'customlink_class'  => 'external',                // 'internal' | 'external' | 'media' | 'mail' | 'tel'
    'type'              => 'external',                // 'internal' | 'external' | 'media' | 'email' | 'telephone'
    'article_id'        => null,                      // int if internal link
    'clang_id'          => null,                      // int if internal link
    'filename'          => null,                      // string if media
    'extension'         => null,                      // file extension if media
    'protocol'          => 'https',                   // scheme for external
    'domain'            => 'example.com',
    'metadata'          => [],                        // article / media metadata
]
```

### Options

```php
$data = MFormOutputHelper::createLinkData('REX_VALUE[1]', [
    'extern_blank' => false,  // default true – don't add target="_blank" for external
    'mode'         => 'raw',  // 'frontend' (default) | 'raw' | 'strict'
]);
```

---

## Rendering a link in a module OUTPUT

```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$data = MFormOutputHelper::createLinkData('REX_VALUE[1]');

if ('' !== $data['customlink_url']) {
    echo '<a href="' . rex_escape($data['customlink_url']) . '"'
        . $data['customlink_target'] . '>'
        . rex_escape($data['customlink_text'])
        . '</a>';
}
```

---

## getCustomLinkUrl() – URL only

When you just need the resolved URL without metadata:

```php
$url = MFormOutputHelper::getCustomLinkUrl('REX_VALUE[1]');
// or from an already-prepared array:
$url = MFormOutputHelper::getCustomLinkUrl($linkData);
```

---

## Repeater rows

Use `MFormRepeaterHelper::decode()` (v9+) – it handles JSON decoding, HTML entity decoding, and filters disabled rows (online/offline toggle) in one call.

> See the **mform-flex-repeater** skill for the full Repeater API including `filterByField()`, `sortByField()`, `groupByField()`, and `limitItems()`.

```php
use FriendsOfRedaxo\MForm\Repeater\MFormRepeaterHelper;

$rows = MFormRepeaterHelper::decode('REX_VALUE[1]');

// Alternative (v8-kompatibel / already-decoded array; prepareItemsForOutput() itself is v9+):
// $raw  = json_decode(html_entity_decode('REX_VALUE[1]', ENT_QUOTES | ENT_HTML5, 'UTF-8'), true) ?? [];
// $rows = MFormRepeaterHelper::prepareItemsForOutput($raw);
```

### Normalising link fields in repeater rows (v9+)

```php
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

// Adds <field>_normalized key alongside the original
$items = MFormOutputHelper::normalizeRepeaterItems($items, ['link', 'cta']);

foreach ($items as $item) {
    $link = $item['link_normalized'];
    echo '<a href="' . rex_escape($link['customlink_url']) . '">'
        . rex_escape($link['customlink_text']) . '</a>';
}

// Replace original field (no _normalized suffix)
$items = MFormOutputHelper::normalizeRepeaterItems($items, ['link'], ['replace' => true]);
```

### Complete repeater module OUTPUT example

```php
use FriendsOfRedaxo\MForm\Repeater\MFormRepeaterHelper;
use FriendsOfRedaxo\MForm\Utils\MFormOutputHelper;

$rows = MFormRepeaterHelper::decode('REX_VALUE[1]');
$rows = MFormOutputHelper::normalizeRepeaterItems($rows, ['cta_link']);

foreach ($rows as $row) {
    $title    = rex_escape($row['title'] ?? '');
    $text     = $row['text'] ?? '';            // may contain HTML from TinyMCE
    $filename = $row['image'] ?? '';
    $link     = $row['cta_link_normalized'];

    echo '<div class="card">';
    if ($filename) {
        echo '<img src="' . rex_url::media($filename) . '" alt="">';
    }
    echo '<h3>' . $title . '</h3>';
    echo '<div>' . $text . '</div>';
    if ($link['customlink_url']) {
        echo '<a href="' . rex_escape($link['customlink_url']) . '"'
            . $link['customlink_target'] . '>'
            . rex_escape($link['customlink_text']) . '</a>';
    }
    echo '</div>';
}
```

---

## Media metadata via createLinkData

When a media filename is passed, the returned array includes:

```php
$data = MFormOutputHelper::createLinkData('hero.jpg');
// $data['type']              → 'media'
// $data['customlink_url']    → '/media/hero.jpg'
// $data['filename']          → 'hero.jpg'
// $data['extension']         → 'jpg'
// $data['metadata']['title'] → media title from mediapool
// $data['metadata']['width'] → image width
// $data['metadata']['height']→ image height
```

---

## isFirstSlice() helper

Check if the current slice is the first slice of an article:

```php
if (MFormOutputHelper::isFirstSlice(REX_SLICE_ID)) {
    // render differently for first module instance
}
```

---

## Common pitfalls

- **Always use `createLinkData()` or `normalizeLinkData()` instead of manually parsing `redaxo://`** – the helper resolves article names, media URLs and external link targets in one call.
- **`customlink_target` is already a ready-to-embed attribute string** (` target="_blank" rel="noopener noreferrer"`) or empty string – just echo it directly, no need to check it.
- **`customlink_class` is a plain string** (`'internal'`, `'external'`, etc.), not an HTML class attribute string.
- **`normalizeRepeaterItems()` does not mutate** the original array – assign the return value.
- **`mode = 'raw'`** keeps backend helper labels (article name with `[ID]` suffix) in the `name` key. Use for debugging only.
- **Always decode repeater values with `MFormRepeaterHelper::decode('REX_VALUE[n]')`** – it handles JSON decoding, HTML entity decoding, empty-string values on fresh modules, and filtering of disabled rows in one call. Only fall back to `json_decode(html_entity_decode($raw, ENT_QUOTES | ENT_HTML5, 'UTF-8'), true) ?? []` when you have a v8-era already-decoded array (then post-process with `MFormRepeaterHelper::prepareItemsForOutput()`; both helpers are v9+).
