---
name: redaxo-modules
description: Authoring REDAXO modules (input/output PHP) and using REX_VALUE / REX_MEDIA / REX_LINK / REX_LINKLIST / REX_MEDIALIST placeholders. Use when the user creates or edits a module, mentions "Modul" or "Module", works with REX_VALUE[1..20], or builds backend forms for content blocks.
---

# REDAXO Modules

Modules are the building blocks of content in REDAXO. Each module has two parts:

- **Input** – the backend form an editor sees when adding/editing the block
- **Output** – the PHP/HTML that renders on the frontend

Modules are placed inside templates via "slices". Editors can add multiple slices of different module types per article.

## Available placeholders

| Placeholder | Range | Notes |
|---|---|---|
| `REX_VALUE[1]`…`REX_VALUE[20]` | 20 slots | text, longtext, anything serializable |
| `REX_MEDIA[1]`…`REX_MEDIA[10]` | 10 slots | single media file (filename string) |
| `REX_MEDIALIST[1]`…`REX_MEDIALIST[10]` | 10 slots | comma-separated filenames |
| `REX_LINK[1]`…`REX_LINK[10]` | 10 slots | single article ID |
| `REX_LINKLIST[1]`…`REX_LINKLIST[10]` | 10 slots | comma-separated article IDs |

These are hard limits set by the core. If you need more, store JSON in one `REX_VALUE` or use the `yform` addon.

## Input form pattern

Always use `name="REX_INPUT_VALUE[N]"` (input) but read with `REX_VALUE[N]` (output). The framework handles the round-trip.

```php
<?php
// Input – module editing form
?>
<fieldset>
    <legend>Headline</legend>
    <div class="form-group">
        <label for="rex-headline">Headline</label>
        <input type="text" id="rex-headline" name="REX_INPUT_VALUE[1]"
               value="REX_VALUE[1]" class="form-control">
    </div>

    <div class="form-group">
        <label for="rex-text">Text</label>
        <textarea id="rex-text" name="REX_INPUT_VALUE[2]"
                  class="form-control" rows="6">REX_VALUE[2]</textarea>
    </div>

    REX_MEDIA[widget=1 id=1 preview=1]
    REX_LINK[id=1 widget=1]
</fieldset>
```

For media and link widgets, use the special widget syntax shown above – not plain inputs. They render the matching picker UI in the backend.

## Output pattern

Always escape user-controlled data when rendering:

```php
<?php
$headline = 'REX_VALUE[1]';
$text     = 'REX_VALUE[2]';
$mediaSrc = 'REX_MEDIA[1]';
$linkId   = (int) 'REX_LINK[1]';
?>
<section class="content-block">
    <h2><?= rex_escape($headline) ?></h2>
    <div class="text"><?= rex_escape($text, 'html_simplified') ?></div>

    <?php if ($mediaSrc && $media = rex_media::get($mediaSrc)): ?>
        <img src="<?= rex_url::media($media->getFileName()) ?>"
             alt="<?= rex_escape($media->getTitle()) ?>"
             width="<?= $media->getWidth() ?>"
             height="<?= $media->getHeight() ?>">
    <?php endif; ?>

    <?php if ($linkId && $article = rex_article::get($linkId)): ?>
        <a href="<?= $article->getUrl() ?>"><?= rex_escape($article->getName()) ?></a>
    <?php endif; ?>
</section>
```

## Escape strategies

`rex_escape()` is REDAXO's safe-by-default helper. Pass a strategy as the second argument:

- `'html'` (default) – full HTML escaping
- `'html_attr'` – inside HTML attributes
- `'html_simplified'` – allow basic formatting tags (b, i, p, br, ul, ol, li, a, etc.)
- `'js'` – inside `<script>` blocks
- `'css'` – inside `<style>`
- `'url'` – for URL components

For Markdown-style WYSIWYG content stored in `REX_VALUE`, use `'html_simplified'` to keep formatting while stripping dangerous tags.

## Storing structured data

When 20 `REX_VALUE` slots aren't enough or you need structured data:

```php
// Input: hidden field with JSON
<input type="hidden" name="REX_INPUT_VALUE[1]"
       value='REX_VALUE[1]' id="rex-data">
// + JS/UI that writes JSON into #rex-data on form submit

// Output: decode safely (REX_VALUE is HTML-escaped, see pitfalls)
$data = rex_var::toArray('REX_VALUE[1]') ?? [];
```

## Common pitfalls

- Mixing `REX_VALUE[1]` and `REX_INPUT_VALUE[1]` – input form ALWAYS uses `REX_INPUT_VALUE`, output ALWAYS uses `REX_VALUE`.
- Forgetting `rex_escape()` – any field touched by an editor must be escaped on output.
- Hardcoding image dimensions – use `rex_media`'s `getWidth()` / `getHeight()` or the `media_manager` addon for responsive variants.
- Querying inside output PHP for many slices on one page → N+1 problem. Cache lookups or batch them.
- Decoding JSON with `json_decode('REX_VALUE[1]', true)` – `REX_VALUE[n]` is substituted HTML-escaped (`"` → `&quot;`, in output additionally `nl2br()`), so `json_decode()` returns `null` without any error and the fallback kicks in silently. Use `rex_var::toArray('REX_VALUE[1]')`, which decodes the entities first (compact JSON only – line breaks become `<br />`), or `REX_VALUE[id=1 output=html]` for the unescaped value.
