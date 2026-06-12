---
name: search-it-modules
description: REDAXO search modules for Search It – building search form input modules, search result output modules, pagination modules, media search, file/PDF results, URL addon results, complex search with category filters, autocomplete integration. Use when the user creates or edits a REDAXO module for search, asks for a search form, search result template, or search output code.
---

# Search It – REDAXO Modules

Search It does not ship ready-made modules. You create them as REDAXO modules (Input + Output) and adapt them to the project.

## Search form module

### Input

```html
<fieldset>
    <legend>Search form</legend>
    <label>Result page article ID</label>
    REX_LINK[id=1 widget=1]
</fieldset>
```

### Output

```php
<form class="search_it-form" action="<?= rex_getUrl(REX_LINK_ID[1]) ?>" method="get">
    <input type="text" name="search" value="<?= rex_escape(rex_request('search', 'string', '')) ?>" placeholder="Search...">
    <button type="submit">Search</button>
</form>
```

The `class="search_it-form"` is required if you use the autocomplete/suggest feature. `REX_LINK_ID[1]` points to the article where the result module sits.

For multilingual projects with Sprog, replace the placeholder:

```php
placeholder="<?= rex_escape(sprogdown('search_placeholder', 'Suchen...')) ?>"
```

## Search result module

### Input

No input needed (or optionally a heading via `REX_VALUE[1]`).

### Output

The result loop is the same regardless of which sources you've configured — what varies is which `$hit['type']` values can show up. Handle every type you've enabled; default-skip the rest. (`hit.type` ∈ `article`, `db_column`, `file`, `url` — see the search-it-search skill for the full hit structure.)

```php
<?php
use FriendsOfRedaxo\SearchIt\SearchIt;

$searchTerm = rex_request('search', 'string', '');
if ($searchTerm === '') {
    return;
}

$search = new SearchIt();
$result = $search->search($searchTerm);

if ($result['count'] === 0) {
    echo '<p>No results for "' . rex_escape($searchTerm) . '".</p>';
    return;
}

echo '<p>' . $result['count'] . ' results for "' . rex_escape($searchTerm) . '"</p>';

foreach ($result['hits'] as $hit) {
    $url = '';
    $title = '';
    $teaser = $hit['highlightedtext']; // already highlight-wrapped, safe to echo

    switch ($hit['type']) {
        case 'article':
            $article = rex_article::get($hit['fid'], $hit['clang']);
            if (!$article) continue 2;
            $url = rex_getUrl($hit['fid'], $hit['clang']);
            $title = $article->getName();
            break;

        case 'db_column':
            // Only worthwhile if you've added DB-column sources in backend settings.
            if ($hit['table'] === rex::getTable('article')) {
                $article = rex_article::get($hit['fid'], $hit['clang']);
                if (!$article) continue 2;
                $url = rex_getUrl($hit['fid'], $hit['clang']);
                $title = $article->getName();
            } else {
                continue 2; // skip non-article DB hits or wire your own handler
            }
            break;

        case 'file':
            // Only if file/PDF indexing is enabled.
            $url = rex_url::media($hit['filename']);
            $title = $hit['filename'];
            break;

        default:
            continue 2; // 'url' (URL addon) etc. — add a branch when needed
    }

    echo '<div class="search-results__item">';
    echo '<h3><a href="' . $url . '">' . rex_escape($title) . '</a></h3>';
    echo '<p>' . $teaser . '</p>';
    echo '</div>';
}
```

If you only ever search articles, you can drop the `db_column` / `file` branches — the structure stays the same.

## Pagination

```php
<?php
use FriendsOfRedaxo\SearchIt\SearchIt;

$searchTerm = rex_request('search', 'string', '');
if ($searchTerm === '') {
    return;
}

$page = rex_request('page', 'int', 1);
$perPage = 10;

$search = new SearchIt();
$search->setLimit(($page - 1) * $perPage, $perPage);
$result = $search->search($searchTerm);

$totalPages = (int) ceil($result['count'] / $perPage);

// ... render results ...

// Pagination links
if ($totalPages > 1) {
    $currentUrl = rex_getUrl(rex_article::getCurrentId(), rex_clang::getCurrentId());
    echo '<nav class="pagination">';
    for ($i = 1; $i <= $totalPages; $i++) {
        $activeClass = ($i === $page) ? ' class="active"' : '';
        echo '<a href="' . $currentUrl . '?search=' . urlencode($searchTerm) . '&page=' . $i . '"' . $activeClass . '>' . $i . '</a> ';
    }
    echo '</nav>';
}
```

## Media search module

First configure additional sources in the backend: add `rex_media.title`, `rex_media.filename`, `rex_media.med_description` as indexed columns.

```php
<?php
use FriendsOfRedaxo\SearchIt\SearchIt;

$searchTerm = rex_request('search', 'string', '');
if ($searchTerm === '') {
    return;
}

$search = new SearchIt();
$search->searchInDbColumn(rex::getTable('media'), 'title');
$result = $search->search($searchTerm);

if ($result['count'] === 0) {
    echo '<p>No media found.</p>';
    return;
}

echo '<div class="media-results">';
foreach ($result['hits'] as $hit) {
    if ($hit['type'] !== 'db_column' && $hit['type'] !== 'file') {
        continue;
    }

    $filename = $hit['filename'] ?? $hit['values']['filename'] ?? '';
    if ($filename === '') {
        continue;
    }

    $media = rex_media::get($filename);
    if (!$media) {
        continue;
    }

    echo '<div class="media-results__item">';
    if ($media->isImage()) {
        echo '<img src="' . rex_url::media($filename) . '" alt="' . rex_escape($media->getTitle()) . '">';
    }
    echo '<p>' . rex_escape($media->getTitle()) . '</p>';
    echo '</div>';
}
echo '</div>';
```

## Complex module with category filter and form

### Input

```html
<fieldset>
    <legend>Search settings</legend>
    <label>Results per page</label>
    <input type="text" name="REX_INPUT_VALUE[1]" value="REX_VALUE[1]" placeholder="10">
</fieldset>
```

### Output

```php
<?php
use FriendsOfRedaxo\SearchIt\SearchIt;

$searchTerm = rex_request('search', 'string', '');
$categoryId = rex_request('category', 'int', 0);
$page = rex_request('page', 'int', 1);
$perPage = (int) ('REX_VALUE[1]' ?: 10);

// Search form with category filter
$categories = rex_category::getRootCategories(true);
?>
<form class="search_it-form" action="<?= rex_getUrl(rex_article::getCurrentId()) ?>" method="get">
    <input type="text" name="search" value="<?= rex_escape($searchTerm) ?>" placeholder="Search...">
    <select name="category">
        <option value="0">All categories</option>
        <?php foreach ($categories as $cat): ?>
            <option value="<?= $cat->getId() ?>"<?= $cat->getId() === $categoryId ? ' selected' : '' ?>>
                <?= rex_escape($cat->getName()) ?>
            </option>
        <?php endforeach ?>
    </select>
    <button type="submit">Search</button>
</form>
<?php

if ($searchTerm === '') {
    return;
}

$search = new SearchIt();
$search->setLimit(($page - 1) * $perPage, $perPage);

if ($categoryId > 0) {
    $search->searchInCategoryTree($categoryId);
}

$result = $search->search($searchTerm);

if ($result['count'] === 0) {
    echo '<p>No results.</p>';

    // Similarity suggestion
    if (!empty($result['simwordsnewsearch'])) {
        echo '<p>Did you mean: <a href="?search=' . urlencode($result['simwordsnewsearch']) . '">'
            . rex_escape($result['simwordsnewsearch']) . '</a>?</p>';
    }
    return;
}

echo '<p>' . $result['count'] . ' results</p>';

foreach ($result['hits'] as $hit) {
    switch ($hit['type']) {
        case 'article':
        case 'db_column':
            $fid = (int) $hit['fid'];
            $article = rex_article::get($fid, $hit['clang']);
            if (!$article) continue 2;
            $url = rex_getUrl($fid, $hit['clang']);
            $title = $article->getName();
            break;

        case 'file':
            $url = rex_url::media($hit['filename']);
            $title = $hit['filename'];
            if (in_array($hit['fileext'], ['pdf', 'doc', 'docx'])) {
                $title .= ' (Download)';
            }
            break;

        default:
            continue 2;
    }

    echo '<div class="search-results__item">';
    echo '<h3><a href="' . $url . '">' . rex_escape($title) . '</a></h3>';
    echo '<p>' . $hit['highlightedtext'] . '</p>';
    echo '</div>';
}

// Pagination
$totalPages = (int) ceil($result['count'] / $perPage);
if ($totalPages > 1) {
    $baseUrl = rex_getUrl(rex_article::getCurrentId(), rex_clang::getCurrentId());
    $params = '?search=' . urlencode($searchTerm);
    if ($categoryId > 0) {
        $params .= '&category=' . $categoryId;
    }

    echo '<nav class="pagination">';
    for ($i = 1; $i <= $totalPages; $i++) {
        $activeClass = ($i === $page) ? ' class="active"' : '';
        echo '<a href="' . $baseUrl . $params . '&page=' . $i . '"' . $activeClass . '>' . $i . '</a> ';
    }
    echo '</nav>';
}
```

## URL addon results

When using the URL addon (v2.0+), enable URL indexing in the backend settings. URL addon hits have `type === 'url'`:

```php
case 'url':
    // Reconstruct URL from URL addon profile
    $urlData = rex_sql::factory()
        ->setTable(rex::getTable('url_generator_url'))
        ->setWhere('md5' , $hit['fid'])
        ->select();

    if ($urlData->getRows() > 0) {
        $url = $urlData->getValue('url');
        $title = $urlData->getValue('seo_title') ?: $urlData->getValue('url');
    }
    break;
```

## Autocomplete / Suggest

Enable in backend settings (Settings > Suggest). The search form needs `class="search_it-form"` and input `name="search"`. Then include the generated JS before `</body>` in your template:

```php
<?php
if (rex_addon::get('search_it')->isAvailable()) {
    echo rex_addon::get('search_it')->getProperty('suggest_js', '');
}
?>
```

## Common pitfalls

- Using `REX_VALUE[id=1]` as article ID directly instead of `REX_LINK_ID[1]` – use the link widget so editors pick the target article visually.
- Forgetting `return;` or wrapping in `if` when search term is empty – the module output runs on every page load, not just when searching.
- Not handling all hit types – if your index contains DB columns or files, unhandled types silently disappear from results.
- Rendering `$hit['highlightedtext']` through `rex_escape()` – the highlighted text contains intentional HTML (`<mark>` tags). Escaping it strips the highlighting. Only escape user-provided values like article names.
- Hard-coding result page URLs instead of using `rex_getUrl()` – breaks when URL structure changes.
- Building pagination with `$result['count']` but forgetting to call `setLimit()` – you get all results on every page.
