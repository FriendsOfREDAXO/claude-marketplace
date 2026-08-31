---
name: url-generator-rebuild
description: Rebuilding URL addon urls from PHP – Profile::buildUrls()/deleteUrls(), the Cache::deleteProfiles() step, why a profile inserted via raw SQL stays empty, why UrlManagerSql::deleteAll() destroys existing urls, and the cache order that leaves a freshly created clang with zero urls. Use when the user creates URL profiles programmatically, writes a setup or migration script that must not lose existing urls, wonders why a new profile or language has no generated urls, or looks for the sitemap flags of a profile.
---

# URL Generator – Rebuilding URLs from PHP

The backend button *"regenerate all urls"* is `url/pages/generator.profiles.php`.
Its logic is the reference for every script:

```php
use Url\Profile;
use Url\UrlManagerSql;

// single profile ("refresh")
$profile = Profile::get($id);
$profile->deleteUrls();      // only this profile's urls
$profile->buildUrls();

// everything ("refresh_all") – see the warning below
UrlManagerSql::deleteAll();
foreach (Profile::getAll() as $profile) {
    $profile->buildUrls();
}
```

All classes live in the `Url` namespace (`Url\Profile`, `Url\Cache`,
`Url\UrlManagerSql`, `Url\Seo`). Verified against addon version 2.2.1.

## A profile inserted with raw SQL has no urls

Writing a row into `rex_url_generator_profile` directly — cloning a profile for a
new language, for instance — does **not** trigger the generator. The profile shows
up in the backend, but `rex_url_generator_url` stays empty for it, and nothing
warns you.

Two steps are required afterwards:

```php
Url\Cache::deleteProfiles();     // profile cache is stale, getAll() would miss the new row
foreach (Profile::getAll() as $profile) {
    if (!in_array((int) $profile->getId(), $myNewProfileIds, true)) {
        continue;                // leave foreign profiles alone
    }
    $profile->deleteUrls();
    $profile->buildUrls();
}
```

## Never call `UrlManagerSql::deleteAll()` in a migration

It is what the backend does for *"regenerate all"*, and it is destructive: urls
whose dataset no longer qualifies are gone for good. Observed on a live-like
database: the count for one namespace dropped from 104 to 86 in a single run —
18 urls that were reachable before returned 404 afterwards.

If existing urls must survive, work **per profile**. `Profile::deleteUrls()` calls
`UrlManagerSql::deleteByProfileId()` and touches nothing else.

## Refresh caches *before* building, not after

When a script creates clangs or changes the yrewrite domain assignment and then
builds urls in the same request, the addon still sees the state from the start of
the request. Symptom: the language created **last** ends up with zero urls while
the earlier one is complete.

```php
rex_delete_cache();
rex_clang::reset();
rex_yrewrite::init();            // domain assignment must be current
// only now:
Url\Cache::deleteProfiles();
// ... buildUrls()
```

Add a check afterwards, because the failure is silent — a profile with no urls
whose template profile has some is a failed build, so build it a second time:

```php
$empty = rex_sql::factory()->getArray(
    'SELECT p.id FROM ' . rex::getTable('url_generator_profile') . ' p
       LEFT JOIN ' . rex::getTable('url_generator_url') . ' u ON u.profile_id = p.id
      WHERE p.id IN (' . $ids . ') GROUP BY p.id HAVING COUNT(u.id) = 0');
```

## Verify by diffing, not by counting

Counting urls proves that *something* was generated, not that it is correct. In a
multi-language setup a generated url should equal the source language's url plus
the language prefix — compare them:

```
de:  /awning-acme-model-x/
at:  /at/awning-acme-model-x/   correct
at:  /awning-acme-at/model-x/   prefix ended up inside the slug
```

Record the url count per profile before and after any migration and diff both the
numbers and a sample of actual urls.

## Sitemap flags live in `table_parameters`

`rex_url_generator_profile` has no sitemap columns. `Profile::inSitemap()`,
`getSitemapFrequency()` and `getSitemapPriority()` read from the JSON in
`table_parameters`:

```json
{"sitemap_add":"1","sitemap_frequency":"always","sitemap_priority":"1.0","column_sitemap_lastmod":""}
```

Cloning a profile copies these along. Note that `sitemap_add = 1` does not by
itself guarantee the urls appear in `sitemap.xml` — the addon registers its
sitemap hook in `boot.php` only when `Url::getRewriter()->getSitemapExtensionPoint()`
resolves, so verify the actual output instead of trusting the flag.
