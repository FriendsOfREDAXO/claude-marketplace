---
name: url-generator-hooks
description: URL addon extension points in REDAXO – URL_PRE_SAVE to shorten or modify generated URLs before they are stored, URL_PROFILE_RESTRICTION to filter datasets, and the CLI isBackend gotcha. Use when the user wants to remove an article path prefix from URLs (e.g. domain.de/acme instead of domain.de/brand-output/acme), filter which datasets get a URL, or debugs unexpected URL regeneration from the CLI.
---

# URL Generator – Extension Point Hooks

## URL_PRE_SAVE — modify a URL before it is stored

Fires once per dataset row, before the URL is written to `rex_url_generator_url`. The subject is a `Url\Url` object; return a modified `Url\Url` to replace it, or `false` to skip this entry entirely.

Params available via `$ep->getParams()`:

| Key | Type | Value |
|---|---|---|
| `profile_id` | int | ID from `rex_url_generator_profile` |
| `article_id` | int | Routing target article |
| `clang_id` | int | Language ID |
| `data_id` | int | Dataset primary key |
| `profile` | `Url\Profile` | Full profile object |

### Recipe: strip article path so URLs are at the domain root

By default the profile article's yrewrite path is prepended to every slug. The profile article ("Brand output") has URL `/brand-output/` → brand URLs become `/brand-output/acme/`.

To generate `/acme/` instead, register this hook in your project's boot file (e.g. the `theme` addon's `functions.php`) **outside** of `isBackend()` / `isFrontend()` blocks so it runs in all contexts:

```php
rex_extension::register('URL_PRE_SAVE', function (rex_extension_point $ep) {
    $params = $ep->getParams();

    // Apply only to the relevant profile (check ID in rex_url_generator_profile)
    if (($params['profile_id'] ?? 0) !== 1) {
        return $ep->getSubject();
    }

    /** @var \Url\Url $url */
    $url       = $ep->getSubject();
    $articleId = $params['article_id'];
    $clangId   = $params['clang_id'];

    $domain     = rex_yrewrite::getDomainByArticleId($articleId, $clangId);
    $articleUrl = rex_getUrl($articleId, $clangId);
    $startUrl   = rex_getUrl($domain->getStartId(), $clangId);

    // Nothing to strip if the profile article IS the domain start article
    if ($articleId === $domain->getStartId()) {
        return $url;
    }

    // Remove the article path segment (handles both with and without language slug)
    $relative = strlen($startUrl) === 1
        ? str_replace('/' . strtolower(rex_clang::get($clangId)->getCode()) . '/', '/', $articleUrl)
        : str_replace($startUrl, '/', $articleUrl);

    $parts    = array_map('urlencode', explode('/', $relative));
    $shortened = new \Url\Url(str_replace(implode('/', $parts), '/', $url->toString()));

    // Reject duplicate URLs (two datasets normalizing to the same slug)
    $exists = rex_sql::factory()->getArray(
        'SELECT id FROM ' . rex::getTable('url_generator_url') . ' WHERE url = ?',
        [$shortened->toString()]
    );
    if (!empty($exists)) {
        return false;
    }

    return $shortened;
});
```

**Important:** use `$url->toString()` — not `__toString()` (the method does not exist on `Url\Url`).

This hook is the documented approach from the URL addon README.

## URL_PROFILE_RESTRICTION — limit which datasets get a URL

```php
rex_extension::register('URL_PROFILE_RESTRICTION', function (rex_extension_point $ep) {
    $params = $ep->getParams();
    // $params['profile']  — \Url\Profile object
    // $params['dataset']  — \rex_yform_manager_dataset
    // Return false to skip this dataset (no URL generated)
    $dataset = $params['dataset'];
    if ($dataset->getValue('status') !== '1') {
        return false;
    }
    return $ep->getSubject();
});
```

## CLI gotcha: isBackend() is true in the console

`redaxo/bin/console` sets `$REX['REDAXO'] = true`, so `rex::isBackend()` returns `true` even in CLI context. The URL addon registers its regeneration listeners when `isBackend() && getUser()`.

`developer:sync` calls `rex_backend_login::createUser()` → a user exists → `CACHE_DELETED` fires → full URL regeneration runs.

**Consequence:** running `php redaxo/bin/console cache:clear && php redaxo/bin/console developer:sync` will trigger URL regeneration — potentially before your `URL_PRE_SAVE` hook is saved to the database (developer:sync writes the hook file to DB during the same run). The result: URLs are regenerated with the original path prefix and your hook has no effect.

**Fix:** always regenerate URLs from the REDAXO backend (URL addon → Generator → URLs → "Alle URLs neu generieren"), not from the CLI. Own setup scripts should set `$REX['REDAXO'] = false` to avoid inheriting backend behaviour.
