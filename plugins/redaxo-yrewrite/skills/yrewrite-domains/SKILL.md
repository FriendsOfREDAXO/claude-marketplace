---
name: yrewrite-domains
description: YRewrite domain configuration in REDAXO – mapping hostnames to category trees, multi-language sites, and fetching URLs via rex_yrewrite. Use when the user configures domains, builds multi-domain sites, generates internal links via rex_yrewrite::getFullUrlByArticleId, or troubleshoots URL routing.
---

# YRewrite Domains

YRewrite ties hostnames (like `www.example.com`, `example.de`, `en.example.com`) to a "mount category" in the article structure. Each domain can be limited to a subset of languages (clangs) and have its own start article, 404 article, and sitemap.

## Configuring a domain

Domains are configured in the backend (YRewrite → Domains) and stored in the table `rex::getTable('yrewrite_domain')` (alias domains in `rex::getTable('yrewrite_alias')`). `rex_yrewrite::generateConfig()` compiles them into the cache file `rex_path::addonCache('yrewrite', 'config.php')`, which is what `rex_yrewrite` reads at runtime; the backend calls `rex_yrewrite::deleteCache()` after every change. Code can read them but typically doesn't write them.

Programmatic access:

```php
$current = rex_yrewrite::getCurrentDomain();
echo $current->getName();              // 'www.example.com'
echo $current->getMountId();           // mount category ID
echo $current->getStartId();           // start article ID for this domain
echo $current->getNotfoundId();        // 404 article ID
echo $current->getStartClang();        // default language ID

$all = rex_yrewrite::getDomains();      // all domains by name
$de  = rex_yrewrite::getDomainByName('www.example.de');
```

## Generating URLs

Always use YRewrite's helpers – never construct URLs by hand. The helpers know about domains, language slugs, and pretty paths.

```php
// Full URL including scheme and domain
$url = rex_yrewrite::getFullUrlByArticleId($articleId, $clangId);

// Core function; YRewrite generates the URL via the URL_REWRITE extension point.
// Path if the article belongs to the domain of the current host (or to no configured domain),
// otherwise the full URL of the article's domain.
$url = rex_getUrl($articleId, $clangId);
```

Use `getFullUrlByArticleId()` for anything that leaves the page (canonical tags, og:url, hreflang alternates, sitemap entries, email confirmations). Use `rex_getUrl()` for in-page links – there is no separate path-only helper.

## Multi-language URLs

Each domain maps to one or more languages. The default language usually has no path prefix; alternate languages get a prefix matching their `code` (e.g. `/en/about`, `/fr/about`).

```php
foreach (rex_clang::getAll(true) as $clang) {
    if (!in_array($clang->getId(), rex_yrewrite::getCurrentDomain()->getClangs(), true)) {
        continue;
    }
    $url = rex_yrewrite::getFullUrlByArticleId(
        rex_article::getCurrentId(),
        $clang->getId()
    );
    echo '<link rel="alternate" hreflang="' . $clang->getCode() . '" href="' . rex_escape($url, 'url') . '">';
}
```

Add a self-referencing canonical with the current language's code as well, plus an `x-default`:

```php
$defaultClang = rex_yrewrite::getCurrentDomain()->getStartClang();
$defaultUrl = rex_yrewrite::getFullUrlByArticleId(rex_article::getCurrentId(), $defaultClang);
echo '<link rel="alternate" hreflang="x-default" href="' . rex_escape($defaultUrl, 'url') . '">';
```

## Detecting the current request context

```php
if (rex_yrewrite::getCurrentDomain()->getName() === 'shop.example.com') {
    // shop-specific frontend logic
}
```

For language-prefix detection, prefer `rex_clang::getCurrentId()` over parsing the URL yourself.

## Switching domains in code (rare)

Generally you stay on whichever domain the user requested. If you need to redirect across domains:

```php
$other = rex_yrewrite::getDomainByName('en.example.com');
if ($other) {
    $url = rex_yrewrite::getFullUrlByArticleId(
        $articleId,
        $other->getStartClang()
    );
    rex_response::sendRedirect($url, 302);
}
```

## Common pitfalls

- Constructing URLs with `rex::getServer() . '/' . $path` – ignores domain mappings, hreflang prefixes, and per-domain start articles.
- Using `rex_getUrl()` and copy-pasting the result somewhere external – it's a relative path whenever the article belongs to the domain of the current host, and a full URL only for articles of another domain. Use `rex_yrewrite::getFullUrlByArticleId()` for anything that leaves the page.
- Hardcoding language codes in URL paths – fetch them via `rex_clang::get($id)->getCode()`.
- Running multiple domains in dev without `/etc/hosts` entries – YRewrite picks the wrong domain and routing silently goes to the default.
- Forgetting to set up the 404 article per domain. Without one, REDAXO falls back to the global 404, which may be in the wrong language.
