---
name: redaxo-templates
description: REDAXO template authoring – page layouts that wrap article content. Use when the user creates or edits a template, asks about getArticle(), works with rex_fragment, sets up multi-language output, or wants to embed slices into a layout.
---

# REDAXO Templates

Templates are the page layouts in REDAXO. They include the document shell (`<html>`, `<head>`, `<body>`) plus navigation, header/footer – and call `getArticle()` to render the article content built from modules.

Each article in the structure addon is assigned a template (with optional inheritance and category-based defaults).

## Minimum template structure

```php
<!DOCTYPE html>
<html lang="<?= rex_clang::getCurrent()->getCode() ?>">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <title><?= rex_escape(rex_article::getCurrent()->getName()) ?> – <?= rex_escape(rex::getServerName()) ?></title>
    <link rel="stylesheet" href="<?= rex_url::frontend('assets/style.css') ?>">
</head>
<body>
    <header>
        <a href="<?= rex_getUrl(rex_article::getSiteStartArticleId()) ?>">
            <?= rex_escape(rex::getServerName()) ?>
        </a>
        <?php // Navigation – see "Navigation" below ?>
    </header>

    <main>
        <?php // Render all slices of the current article ?>
        <?= REX_ARTICLE[] ?>
    </main>

    <footer>
        <p>&copy; <?= date('Y') ?> <?= rex_escape(rex::getServerName()) ?></p>
    </footer>
</body>
</html>
```

`REX_ARTICLE[]` (the bracket placeholder) is the standard way to render the current article's slices. There's also `REX_ARTICLE[id=5]` to render a specific article inline.

## Multi-language

REDAXO supports multiple languages via `rex_clang`. Each language has an ID, a code (e.g. `de`, `en`), and a name.

```php
<html lang="<?= rex_clang::getCurrent()->getCode() ?>">

<?php // Language switcher ?>
<ul class="lang-switch">
<?php foreach (rex_clang::getAll(true) as $clang): ?>
    <?php $url = rex_getUrl(null, $clang->getId()); ?>
    <li class="<?= $clang->getId() === rex_clang::getCurrentId() ? 'active' : '' ?>">
        <a href="<?= $url ?>" hreflang="<?= $clang->getCode() ?>">
            <?= rex_escape($clang->getName()) ?>
        </a>
    </li>
<?php endforeach; ?>
</ul>
```

`rex_getUrl(null, $clangId)` keeps the current article but switches the language. `rex_getUrl($articleId, $clangId)` switches both.

## Navigation patterns

The structure addon doesn't ship navigation builders; use one of these:

**Manual recursion (small sites):**

```php
function renderNav(int $parentId = 0, int $maxDepth = 2, int $depth = 0): string
{
    if ($depth >= $maxDepth) {
        return '';
    }
    $items = rex_category::getChildrenById($parentId);
    if (!$items) {
        return '';
    }
    $html = '<ul class="nav-level-' . $depth . '">';
    foreach ($items as $cat) {
        if (!$cat->isOnline()) {
            continue;
        }
        $current = $cat->getId() === rex_category::getCurrentId() ? ' class="active"' : '';
        $html .= '<li' . $current . '>';
        $html .= '<a href="' . $cat->getUrl() . '">' . rex_escape($cat->getName()) . '</a>';
        $html .= renderNav($cat->getId(), $maxDepth, $depth + 1);
        $html .= '</li>';
    }
    return $html . '</ul>';
}

echo renderNav();
```

**Bootstrap-style menus:** consider the `bootstrap_components` or `jp_navtagger` addons – they generate navigation HTML from the structure.

## Fragments – reusable template parts

`rex_fragment` lets you split templates into reusable pieces. Looks for fragments under `<addon>/fragments/` or in your project's `fragments/` directory.

```php
$f = new rex_fragment();
$f->setVar('title', $article->getName(), false);
$f->setVar('image', $heroImage);
echo $f->parse('hero.php');
```

In `fragments/hero.php`:

```php
<section class="hero">
    <h1><?= rex_escape($this->getVar('title')) ?></h1>
    <?php if ($img = $this->getVar('image')): ?>
        <img src="<?= rex_url::media($img) ?>" alt="">
    <?php endif; ?>
</section>
```

## Cache-aware patterns

Templates run on every request unless cached. For expensive lookups (DB queries, external APIs), use `rex_extension::register('OUTPUT_FILTER', ...)` for full-page caching, or store fragments via `rex_file::put(rex_path::cache(...))`.

## Common pitfalls

- Hardcoding the start article's ID instead of `rex_article::getSiteStartArticleId()`.
- Forgetting the `lang` attribute on `<html>` – breaks accessibility and SEO.
- Outputting unescaped article names or category titles. Editors can put HTML there.
- Using `$_SERVER['HTTP_HOST']` instead of `rex::getServer()` / `rex_yrewrite::getCurrentDomain()`.
- Forgetting to handle the "article not found" case for `rex_article::get()` – it returns `null`.
- Editing a template file of the `developer` addon and expecting an immediate effect – template code (like module input/output) is served from the database (`rex::getTable('template')`, cached by `rex_template_cache`); the files (by default under `redaxo/data/addons/developer/`) are only a mirror. The addon syncs automatically on a web request only if `sync_frontend`/`sync_backend` is enabled (default) **and** debug mode is on or the request has an admin backend session – without debug mode, anonymous visitors never trigger it, and other console commands don't either. Otherwise sync explicitly:
  ```bash
  bin/console developer:sync   # newer side wins; --force-files / --force-db pick a side
  ```
  `--force-files` also deletes templates/modules from the database that have no directory.
