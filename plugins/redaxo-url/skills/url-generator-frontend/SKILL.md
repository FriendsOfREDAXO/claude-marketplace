---
name: url-generator-frontend
description: Resolving the current dataset in a REDAXO module or template using the URL addon – Url::resolveCurrent(), reading dataset values, detecting active profile namespace. Use when the user writes a module that should display different content depending on which brand/product URL is currently open, checks Url::resolveCurrent() return value, or asks how to detect the active URL namespace.
---

# URL Generator – Frontend Resolution

When a visitor opens a generated URL (e.g. `/acme/`), the URL addon routes the request to the profile's `article_id`. The module on that article needs to detect *which* dataset is active.

## Url::resolveCurrent()

```php
use Url\Url;

$manager = Url::resolveCurrent();

if ($manager === null) {
    // No URL addon URL matched — regular article, show overview
}
```

`resolveCurrent()` looks up the current request URL in `rex_url_generator_url` and returns a `UrlManager` instance (or `null` if no match).

## Checking the active profile namespace

Each profile has a `namespace` (set in the backend, e.g. `brands`). Use this to distinguish multiple profiles routing to the same article:

```php
if ($manager->getProfile()->getNamespace() === 'brands') {
    // brand detail view
}
```

Or check by profile ID:
```php
if ($manager->getProfile()->getId() === 1) { ... }
```

## Reading dataset values

```php
$datasetId = $manager->getDatasetId(); // = data_id in rex_url_generator_url

// Load the dataset. Any YForm table works via the generic API; a project with
// its own model class can use that instead.
$dataset = rex_yform_manager_dataset::get($datasetId, 'rex_yf_brands');
if ($dataset === null) {
    // Row deleted after URL was generated
    rex_response::setStatus(rex_response::HTTP_NOT_FOUND);
    exit;
}
```

## Typical module output.php pattern

```php
use Url\Url;

if (rex::isBackend()) {
    echo '<div class="alert alert-info">Brand output: detail or overview view.</div>';
    return;
}

$manager = Url::resolveCurrent();

if ($manager !== null && $manager->getProfile()->getNamespace() === 'brands') {
    // Detail view
    $dataset = rex_yform_manager_dataset::get($manager->getDatasetId(), 'rex_yf_brands');
    if (!$dataset) { rex_response::setStatus(rex_response::HTTP_NOT_FOUND); exit; }

    $fragment = new rex_fragment();
    $fragment->setVar('brand', $dataset, false);
    echo $fragment->parse('brandView.php');
} else {
    // Overview
    $fragment = new rex_fragment();
    $fragment->setVar('model', rex_yform_manager_dataset::query('rex_yf_brands')->find(), false);
    echo $fragment->parse('brandListView.php');
}
```

## Common pitfalls

**`resolveCurrent()` returns null on the detail article** — The URL addon only resolves if a generated URL in `rex_url_generator_url` matches the current request. If URLs were not regenerated after the profile was created, there are no entries to match. Regenerate from the backend.

**Wrong article handles the request** — `article_id` in `rex_url_generator_url` must point to the article carrying the detail module. If it points to the overview article, `resolveCurrent()` will return a manager but the wrong module runs. Check with:
```sql
SELECT url, article_id FROM rex_url_generator_url WHERE profile_id = 1 LIMIT 5;
```

**The overview article must not be the same as the detail article** — If the module on article X is supposed to show both overview and detail, the URL addon must route detail URLs to article X. Using the overview article for URL generation base (to get root-level URLs) and then having `article_id` in the url table point to the detail article is the correct setup (see `url-generator-hooks`).
