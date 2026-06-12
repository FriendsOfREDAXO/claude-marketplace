---
name: search-it-indexing
description: Search It indexing – generating and managing the search index, reindex articles, index database columns, index files and PDFs, console commands search_it:reindex and search_it:clearCache, cronjobs, plaintext conversion via FriendsOfRedaxo\SearchIt\Plaintext\PlaintextConverter (configured through backend settings + SEARCH_IT_PLAINTEXT extension point) and PDF text extraction via FriendsOfRedaxo\SearchIt\Pdf\PdfConverter (requires pdftotext / poppler-utils on the server). Use when the user asks about rebuilding the search index, indexing custom content, or the search returns stale results.
---

# Search It – Indexing

Search It stores all searchable content in database tables (`rex_tmp_search_it_index`, `rex_tmp_search_it_keywords`). Content must be indexed before it appears in search results.

## What gets indexed

By default:
- **Articles** – all online articles in all languages, rendered to plaintext
- **Media files** – if configured in backend settings (Additional sources)
- **DB columns** – any table/column pair configured in backend settings
- **PDF files** – text extracted via `pdftotext` (must be installed on server)
- **URL addon URLs** – if URL addon is installed and enabled in settings

## Full reindex

### Via backend

Backend > Search It > Generate index > "Generate index" button.

### Via console

```bash
php redaxo/bin/console search_it:reindex          # full reindex
php redaxo/bin/console search_it:clearCache        # clear search cache only
```

### Via PHP

```php
use FriendsOfRedaxo\SearchIt\SearchIt;

$search = new SearchIt();
$search->generateIndex();   // full reindex
```

## Index individual items

```php
$search = new SearchIt();

// Single article (optionally for specific language)
$search->indexArticle(42);                    // all languages
$search->indexArticle(42, rex_clang::getCurrentId());  // current language only

// Database column
$search->indexColumn(rex::getTable('my_table'), 'description');

// File
$search->indexFile('document.pdf');
```

## Clear index and cache

```php
$search = new SearchIt();
$search->deleteIndex();     // drop entire index (requires regeneration)
$search->deleteCache();     // clear cached search results
$search->deleteKeywords();  // clear similarity keyword index
```

## Automatic re-indexing

Search It hooks into REDAXO extension points via `EventHandler` and automatically re-indexes articles when they are saved, published or deleted. This happens for:

- `ART_ADDED`, `ART_UPDATED`, `ART_DELETED`
- `ART_STATUS` (online/offline toggle)
- `SLICE_ADDED`, `SLICE_UPDATED`, `SLICE_DELETED`
- `MEDIA_ADDED`, `MEDIA_UPDATED`

No manual action needed for article content changes. But if you change backend settings (e.g. add a new DB column source), you must trigger a full reindex.

## Cronjobs

In the backend under Cronjob addon, two cronjob types are available:

- **Search it: Reindex** – scheduled reindex (full, articles only, columns only, or URLs only)
- **Search it: Clear Cache** – scheduled cache clearing

Useful for sites with frequently changing external data sources (e.g. DB columns filled by imports).

## Plaintext conversion

Articles are fetched via HTTP (or socket), rendered, then converted to plaintext by `FriendsOfRedaxo\SearchIt\Plaintext\PlaintextConverter`. It strips HTML, applies CSS selector exclusions, runs regex replacements and optionally parses Textile. Settings + the `SEARCH_IT_PLAINTEXT` extension point are the supported ways to tune it — instantiating the converter directly is rarely necessary.

Configure in backend: Settings > Plaintext settings:
- **CSS selectors to exclude** – e.g. `nav, .no-search, footer` (content in these elements is not indexed)
- **Regex replacements** – custom find/replace patterns applied before indexing
- **Strip HTML tags** – remove all remaining tags after processing

### Customise plaintext via extension point

```php
rex_extension::register('SEARCH_IT_PLAINTEXT', function(rex_extension_point $ep) {
    $text = $ep->getSubject();
    // Remove specific content before indexing
    $text = preg_replace('/<div class="no-index">.*?<\/div>/s', '', $text);
    return $text;
});
```

Return an array to control further processing:

```php
return ['text' => $cleanedText, 'process' => true];
// process = true: standard plaintext conversion still runs after your hook
// process = false: use your text as-is, skip built-in conversion
```

## PDF indexing

Requires `pdftotext` (from `poppler-utils`) on the server:

```bash
apt-get install poppler-utils    # Debian/Ubuntu
```

Search It uses `FriendsOfRedaxo\SearchIt\Pdf\PdfConverter` (shells out to `pdftotext`) to extract text from PDF files in the media pool. Enable file indexing and add `pdf` to the allowed extensions in backend settings.

## Database tables

| Table | Purpose |
|-------|---------|
| `rex_tmp_search_it_index` | Main index (plaintext, metadata per article/column/file) |
| `rex_tmp_search_it_keywords` | Keywords for similarity search |
| `rex_tmp_search_it_cache` | Cached search results |
| `rex_tmp_search_it_cacheindex_ids` | Links cache entries to index entries |
| `rex_tmp_search_it_stats_searchterms` | Search term statistics |

Tables use the `tmp_` prefix because they are regenerable – the index can always be rebuilt from source content.

## Common pitfalls

- Adding a new DB column in the backend settings but forgetting to reindex – the column data does not appear until the next full index run.
- Running `generateIndex()` on every page load – this is an expensive operation. Only call it from console, cronjob, or backend.
- Expecting immediate results after `indexArticle()` – if the search cache contains stale results, call `deleteCache()` as well.
- Forgetting `poppler-utils` on the server – PDF files are silently skipped if `pdftotext` is not available.
- Indexing articles on a domain that requires HTTPS but having SSL verification enabled in settings when using a self-signed cert – indexing fails silently. Disable SSL verification for local development.
