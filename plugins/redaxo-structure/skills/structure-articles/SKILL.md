---
name: structure-articles
description: Working with REDAXO articles via rex_article – fetching, listing, status, dates, URLs, and the start/notfound/site articles. Use when the user reads or modifies article data, builds article lists for templates, queries the structure tree, or handles per-language article variants.
---

# rex_article

`rex_article` represents a single article (page) in the structure tree. Articles live inside categories and inherit from a category template unless they override it.

Each article exists once per language: `rex_article::get($id, $clang)` returns a different instance for each clang, but they share the structural metadata (parent, priority, name).

## Fetching

```php
// Current article (the one being rendered)
$article = rex_article::getCurrent();

// By ID (uses current clang)
$article = rex_article::get($id);

// By ID + specific language
$article = rex_article::get($id, $clangId);

// Special articles
$start    = rex_article::getSiteStartArticleId();    // home
$notFound = rex_article::getNotfoundArticleId();     // 404
```

Always check for `null`:

```php
$article = rex_article::get($id);
if (!$article) {
    rex_logger::factory()->log('warning', "Article $id not found");
    return;
}
```

## Common getters

```php
$article->getId();             // int
$article->getName();           // article name (editor-facing)
$article->getCategoryId();     // parent category ID
$article->getStatus();         // 1 = online, 0 = offline
$article->getTemplateId();     // assigned template
$article->getCreateDate();     // unix timestamp
$article->getUpdateDate();     // unix timestamp
$article->getCreateUser();     // username string
$article->getUpdateUser();     // username string
$article->getValue('art_yrewrite_title'); // any meta field
$article->getUrl();            // resolves through yrewrite if installed
$article->getPath();           // category path string (e.g. "|2|7|")
$article->isOnline();          // true if status == 1
$article->isStartArticle();    // true if this is the category's start article
$article->isSiteStartArticle();// true if this is the homepage
$article->isNotFoundArticle(); // true for the 404 page
```

## Listing articles in a category

```php
$categoryId = 5;
$onlyOnline = true;
$articles = rex_article::getArticlesOfCategory($categoryId, $onlyOnline);

foreach ($articles as $article) {
    echo '<li><a href="' . $article->getUrl() . '">'
       . rex_escape($article->getName()) . '</a></li>';
}
```

To include offline articles (e.g. in admin tools), pass `false`. The result is already sorted by `priority`.

## Listing across categories – use rex_sql

For more complex filters (date range, custom meta field, cross-category), drop down to SQL:

```php
$sql = rex_sql::factory();
$sql->setQuery(
    'SELECT id FROM ' . rex::getTable('article') . '
     WHERE clang_id = :clang
       AND status = 1
       AND createdate >= :since
     ORDER BY createdate DESC
     LIMIT 10',
    ['clang' => rex_clang::getCurrentId(), 'since' => '2026-01-01']
);

$articles = [];
foreach ($sql as $row) {
    if ($a = rex_article::get((int) $row->getValue('id'))) {
        $articles[] = $a;
    }
}
```

## Article in a different language

```php
$article = rex_article::getCurrent();
$en = rex_article::get($article->getId(), 1);  // 1 = clang_id for English
if ($en && $en->isOnline()) {
    echo 'EN version: ' . $en->getUrl();
}
```

This is how language switchers know whether the current page even has a translation.

## Programmatically creating / updating articles

Use `rex_article_service`, not direct SQL inserts. It handles `path`, `priority`, slug generation, and triggers `ART_ADDED` / `ART_UPDATED` extension points so other addons (yrewrite, search) get notified.

```php
// addArticle() returns a status message, not the ID – capture the ID from ART_ADDED
$articleId = null;
rex_extension::register('ART_ADDED', static function (rex_extension_point $ep) use (&$articleId): void {
    $articleId = (int) $ep->getParam('id');
});

$data = [
    'name'         => 'New article', // required
    'category_id'  => 5,             // required, 0 for root
    'priority'     => 1,             // required
    'template_id'  => 1,
];

rex_article_service::addArticle($data);
```

The article is always created in all languages – a `clang` key in `$data` has no effect. It starts offline in every language; set the status afterwards with `rex_article_service::articleStatus($articleId, $clangId, 1)`.

Editing:

```php
rex_article_service::editArticle($articleId, rex_clang::getCurrentId(), [
    'name'        => 'Renamed',
    'template_id' => 2,
]);
```

Deleting (removes all language variants):

```php
rex_article_service::deleteArticle($articleId);
```

## Common pitfalls

- Caching a `rex_article` instance and using it after a different language's request – mix-up between clangs. Always re-fetch with the right clang.
- Iterating thousands of articles via `rex_article::get()` in a loop – each call does a lookup. For bulk reads, use SQL and `rex_article::get()` only for items you actually display.
- Using the return value of `addArticle()` as the article ID – it is a status message (`string`), `rex_article::get()` then returns `null`. Get the ID from the `ART_ADDED` extension point (param `id`). `rex_extension` has no unregister, so register the listener once, not per call.
- Modifying `rex_article` table directly via SQL – breaks the cache. Always go through `rex_article_service`.
- Forgetting that `getValue('field')` only returns metadata (from `meta_info`-managed columns), not slice content. Slice content is fetched via `rex_article_content` (see structure-content skill).
