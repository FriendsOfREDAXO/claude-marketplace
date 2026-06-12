---
name: structure-categories
description: Working with REDAXO categories via rex_category – tree traversal, navigation building, parents/children, current path. Use when the user builds navigation menus, breadcrumbs, sitemaps, or queries the category structure of a REDAXO site.
---

# rex_category

`rex_category` represents a node in the structure tree. Each category has a "start article" (the page shown when the category itself is requested) plus zero or more child articles and child categories.

Like articles, a category exists per language. The tree topology is shared across languages, but `name` and `online` status differ.

## Fetching

```php
// By ID (current clang)
$cat = rex_category::get($id);

// By ID + specific clang
$cat = rex_category::get($id, $clangId);

// Current category (parent of the current article)
$cat = rex_category::getCurrent();

// All top-level (root) categories
$rootCats = rex_category::getRootCategories(true); // true = only online
```

## Tree traversal

```php
$cat->getId();
$cat->getName();
$cat->getParentId();          // 0 for root categories
$cat->getParent();            // rex_category or null
$cat->getPath();              // pipe-separated ID path: "|2|7|"
$cat->getPathAsArray();       // [2, 7]
$cat->getLevel();             // depth (0 = root)
$cat->getStartArticleId();    // article shown when category is requested
$cat->getStartArticle();      // rex_article instance
$cat->getUrl();               // URL of the start article
$cat->getValue('cat_meta_X'); // category meta_info field
$cat->isOnline();
$cat->isPermitted();          // current user can see it (backend perms)
```

```php
$cat->getChildren(true);   // direct subcategories (true = only online)
$cat->getArticles(true);   // direct articles in this category
```

For children of a category by ID without instantiating it first:

```php
rex_category::getChildrenById($parentId, true); // root if $parentId === 0
```

## Building navigation

Recursive simple example (top-2-level menu):

```php
function renderNavigation(array $categories, int $maxLevel = 2): string
{
    $html = '<ul class="nav">';
    foreach ($categories as $cat) {
        if (!$cat->isOnline()) {
            continue;
        }
        $isCurrent = in_array($cat->getId(), rex_category::getCurrent()?->getPathAsArray() ?? [], true)
                  || $cat->getId() === rex_category::getCurrent()?->getId();
        $cls = $isCurrent ? ' class="active"' : '';
        $html .= '<li' . $cls . '>';
        $html .= '<a href="' . $cat->getUrl() . '">' . rex_escape($cat->getName()) . '</a>';
        if ($cat->getLevel() < $maxLevel - 1 && ($children = $cat->getChildren(true))) {
            $html .= renderNavigation($children, $maxLevel);
        }
        $html .= '</li>';
    }
    return $html . '</ul>';
}

echo renderNavigation(rex_category::getRootCategories(true));
```

## Breadcrumbs

```php
$current = rex_category::getCurrent();
if ($current) {
    $crumbs = [];
    foreach ($current->getPathAsArray() as $id) {
        $crumbs[] = rex_category::get($id);
    }
    $crumbs[] = $current; // include the current category itself

    echo '<nav class="breadcrumb"><ol>';
    foreach ($crumbs as $crumb) {
        echo '<li><a href="' . $crumb->getUrl() . '">' . rex_escape($crumb->getName()) . '</a></li>';
    }
    // current article (if not the start article of the category)
    $article = rex_article::getCurrent();
    if ($article && !$article->isStartArticle()) {
        echo '<li class="active">' . rex_escape($article->getName()) . '</li>';
    }
    echo '</ol></nav>';
}
```

## Programmatically managing categories

Use `rex_category_service`, not direct SQL:

```php
$id = rex_category_service::addCategory(
    $parentId,                       // 0 for root
    rex_clang::getCurrentId(),
    [
        'name'     => 'New section',
        'priority' => 1,
        'status'   => 1,
    ]
);

rex_category_service::editCategory($id, rex_clang::getCurrentId(), [
    'name' => 'Renamed section',
]);

rex_category_service::deleteCategory($id);
```

Setting/changing the start article of a category:

```php
rex_category_service::categoryIsStartArticle($categoryId, $articleId);
```

## Permissions

In backend tools, always check before showing/manipulating:

```php
$cat = rex_category::get($id);
if (!$cat || !rex_article_perm::has(rex::getUser(), $cat->getId())) {
    return;
}
```

For frontend navigation, `isOnline()` is enough – the structure addon already filters by online status.

## Common pitfalls

- Building the active-state check by string-comparing names instead of IDs – breaks when categories share a name.
- Generating a navigation tree with thousands of categories without caching – every page render walks the full tree.
- Forgetting `isOnline()` on each level – offline parents whose children are online become reachable via direct URL but not from the menu.
- Hardcoding category IDs in templates instead of using `rex_yrewrite::getCurrentDomain()->getMountId()` for "below this domain's root". On multi-domain sites see the yrewrite-domains skill for the full domain-rooted navigation pattern.
- Calling `getChildren()` without the `$ignore_offlines = true` flag in a public template, then leaking offline categories into navigation.
