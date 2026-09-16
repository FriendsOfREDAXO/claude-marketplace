---
name: yrewrite-seo
description: SEO helpers in YRewrite – meta tags via rex_yrewrite_seo, sitemap.xml generation, robots.txt, canonical and hreflang tags. Use when the user adds SEO meta to templates, configures sitemap inclusion, customizes robots.txt, or sets up structured social-share metadata (Open Graph, Twitter cards).
---

# YRewrite SEO

YRewrite ships with `rex_yrewrite_seo` – a helper that pulls SEO data from articles (title, description, image) and renders meta tags. It also generates `sitemap.xml` and a `robots.txt` block.

## Article SEO data

Each article has SEO fields managed via the "SEO Data" panel in the backend (article edit sidebar). They are columns of `rex::getTable('article')` added by YRewrite's `install.php` – not meta-info fields, so there is no `art_` prefix. Read them from an article instance, e.g. `rex_article::getCurrent()->getValue('yrewrite_title')`:

- `yrewrite_title` – `<title>` override (falls back to the domain's title scheme, default `%T / %SN` = article name / server name)
- `yrewrite_description` – meta description
- `yrewrite_index` – `0` = follow the article status (default), `1` = index, `-1` = noindex, `2` = noindex but follow links
- `yrewrite_canonical_url` – override canonical
- `yrewrite_image` – Open Graph image filename
- `yrewrite_changefreq`, `yrewrite_priority` – sitemap values (empty priority = derived from the category depth)

For domain-level defaults (site title, description), set them per domain in YRewrite → Domains → SEO.

## Rendering tags in the template

Place this in the `<head>` of your template:

```php
<?php $seo = new rex_yrewrite_seo(); ?>
<?= $seo->getTags() ?>
```

`getTags()` returns `<title>`, meta description, robots, canonical, hreflang, Open Graph and Twitter tags as one markup string – don't wrap it in `rex_escape`, and don't add separate `<title>` / description tags next to it. It replaces `getTitleTag()`, `getDescriptionTag()`, `getRobotsTag()`, `getCanonicalUrlTag()` and `getHreflangTags()`, which are `@deprecated` ("use getTags instead"). The canonical link is only included for indexable articles (`yrewrite_index` `1`, or `0` and online). `getTitle()` and `getDescription()` stay available for your own markup; escape them yourself.

## Open Graph & Twitter cards

`rex_yrewrite_seo::getTags()` already renders title, description, robots, canonical, hreflang and basic Open Graph / Twitter tags in one call (the image via the media manager type `yrewrite_seo_image`). To control them yourself, e.g. with a fallback image, build them from the SEO data:

```php
<?php
$seo = new rex_yrewrite_seo();
$ogImage = (string) rex_article::getCurrent()->getValue('yrewrite_image');
$ogImageUrl = $ogImage
    ? rex::getServer() . rex_url::media($ogImage)
    : rex::getServer() . rex_url::frontend('assets/og-default.jpg');
?>
<meta property="og:title" content="<?= rex_escape($seo->getTitle(), 'html_attr') ?>">
<meta property="og:description" content="<?= rex_escape($seo->getDescription(), 'html_attr') ?>">
<meta property="og:url" content="<?= rex_escape(rex_yrewrite::getFullUrlByArticleId(rex_article::getCurrentId(), rex_clang::getCurrentId()), 'html_attr') ?>">
<meta property="og:type" content="website">
<meta property="og:image" content="<?= rex_escape($ogImageUrl, 'html_attr') ?>">

<meta name="twitter:card" content="summary_large_image">
<meta name="twitter:title" content="<?= rex_escape($seo->getTitle(), 'html_attr') ?>">
<meta name="twitter:description" content="<?= rex_escape($seo->getDescription(), 'html_attr') ?>">
<meta name="twitter:image" content="<?= rex_escape($ogImageUrl, 'html_attr') ?>">
```

## sitemap.xml

YRewrite auto-generates `https://<domain>/sitemap.xml`. An article is listed only if `yrewrite_index` is `1`, or `0` and the article is online. To exclude an article, set `yrewrite_index` to `-1` or `2` (there is no separate "exclude from sitemap" option).

To extend the sitemap with dynamic entries (e.g. YForm dataset URLs), hook into the sitemap generation:

```php
// In your addon's boot.php
rex_extension::register('YREWRITE_SITEMAP', function (rex_extension_point $ep) {
    $items = $ep->getSubject(); // list of ready-made '<url>…</url>' XML strings

    foreach (team_member::query()->where('status', 1)->find() as $member) {
        $loc = rex_yrewrite::getFullUrlByArticleId(42, rex_clang::getCurrentId())
            . '?member=' . $member->getId();
        $items[] = '<url>'
            . '<loc>' . rex_escape($loc) . '</loc>'
            . '<lastmod>' . date('c', strtotime((string) $member->getValue('updatedate'))) . '</lastmod>'
            . '<changefreq>monthly</changefreq>'
            . '<priority>0.6</priority>'
            . '</url>';
    }

    return $items; // YRewrite joins the strings with implode() inside <urlset>
});
```

Items must be XML strings – `implode()` turns arrays into the text `Array` (with an "Array to string conversion" warning). `YREWRITE_SITEMAP` has no params. For entries that belong to one domain, use `YREWRITE_DOMAIN_SITEMAP` instead: same subject, the `rex_yrewrite_domain` is in `$ep->getParam('domain')`, and it only fires when the requested host is a configured domain (or no domain is configured).

## robots.txt

YRewrite serves `robots.txt` per domain. Configure each domain's robots block in YRewrite → Domains → robots.txt. Default content:

```
User-agent: *
Allow: /
Sitemap: https://www.example.com/sitemap.xml
```

For staging environments, swap to:

```
User-agent: *
Disallow: /
```

Detect environment in your template or `boot.php` and switch with an extension point if you don't want to maintain separate domain configs.

## Multi-language SEO checklist

- ✅ `<html lang="...">` matches the active clang code
- ✅ Self-canonical with full URL (already handled by `getTags()` for indexable articles)
- ✅ `hreflang` for every supported language including `x-default` (`getTags()` adds `x-default` only on the domain start article when the domain detects the start language automatically)
- ✅ Translated `<title>` and meta description per article per language
- ✅ Domain-specific sitemap (don't merge sitemaps across domains)

## Structured data (JSON-LD)

YRewrite doesn't render JSON-LD; add it explicitly. For an article page:

```php
<script type="application/ld+json">
<?= json_encode([
    '@context'      => 'https://schema.org',
    '@type'         => 'Article',
    'headline'      => $seo->getTitle(),
    'description'   => $seo->getDescription(),
    'datePublished' => date('c', rex_article::getCurrent()->getCreateDate()),
    'dateModified'  => date('c', rex_article::getCurrent()->getUpdateDate()),
    'author'        => [
        '@type' => 'Organization',
        'name'  => rex::getServerName(),
    ],
    'mainEntityOfPage' => rex_yrewrite::getFullUrlByArticleId(
        rex_article::getCurrentId(),
        rex_clang::getCurrentId()
    ),
], JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE) ?>
</script>
```

## Common pitfalls

- Wrapping `getTags()` in `rex_escape()` – it returns markup, escaping it breaks the tags.
- Forgetting OG images on social posts – Twitter and LinkedIn fall back to no image, hurting click-throughs.
- Setting `yrewrite_canonical_url` to a duplicate URL on multiple articles – Google then ignores all but one.
- Adding articles to the sitemap that return 404 / 403 – costs crawl budget. Run a periodic sitemap audit.
- Leaving `Disallow: /` in production after copying from staging.
