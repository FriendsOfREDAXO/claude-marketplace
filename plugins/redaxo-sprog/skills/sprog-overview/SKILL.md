---
name: sprog-overview
description: Using the Sprog addon in REDAXO for wildcard-based multi-language text management. Use when the user asks what Sprog is, wants to output Sprog wildcards in modules or templates, accesses wildcard values from PHP, asks about Sprog cache behavior, or works with rex_sprog_wildcard for the first time.
---

# Sprog – Overview

Sprog stores translatable text snippets ("wildcards") in the database and replaces `{{namespace.key}}` markers in the rendered HTML output. It is the standard way to manage hardcoded UI strings, labels, and ARIA attributes in REDAXO modules and templates — avoid hardcoded strings, use Sprog.

## How it works

1. Wildcards are stored per language in `rex_sprog_wildcard` (columns: `wildcard`, `replace`, `clang_id`)
2. After a page renders, Sprog's `OUTPUT_FILTER` extension point scans the HTML for `{{…}}` markers and replaces them with the stored value for the current `clang_id`
3. Replacements are cached per language — the cache must be cleared after inserts

## Output in modules and templates

```
{{namespace.key}}
```

Examples:

```
{{nav.home}}
{{form.submit_label}}
{{aria.close_button}}
```

The namespace is a free-text prefix — use it to group related wildcards. Convention: `feature.key` (e.g. `hero.headline`, `footer.contact_link`).

## Adding wildcards

Use an idempotent setup script (see the `sprog-add` skill / `/redaxo-sprog:sprog-add`) to insert wildcards across all project languages in one run.

## Reading wildcard values from PHP

Sprog processes the final HTML output, so `{{…}}` in PHP strings that never reach the HTML output won't be replaced. To resolve a wildcard to its current value in PHP:

```php
$value = rex_sprog::getWildcard('namespace.key', rex_clang::getCurrentId());
// Returns the replacement string, or the key itself if not found
```

## Cache

Sprog caches replacements per language. After bulk-inserting wildcards (e.g. via a setup script), clear the cache so changes appear immediately:

```php
rex_delete_cache(); // programmatic
```

Or via the REDAXO backend: System → Cache leeren.

## Backend management

Wildcards are managed in the Sprog backend page (Sprog → Wildcards). Each row shows the wildcard key and one tab per active language. Wildcards without a translation fall back silently to an empty string — always provide a translation for every active language.

## Common pitfalls

- **Empty output**: the wildcard exists in the DB but the cache is stale. Clear the cache after inserts.
- **Wrong language**: `rex_clang::getCurrentId()` returns the current frontend language. In CLI context or backend previews it may return a different value than expected.
- **`{{…}}` not replaced**: the marker was output inside a `<script>` or `<style>` tag, or in a context where `OUTPUT_FILTER` doesn't run (e.g. JSON responses, AJAX endpoints). Use `rex_sprog::getWildcard()` for those cases.
- **Key collision**: two wildcards with the same key in different namespaces are fine — `foo.label` and `bar.label` are distinct. But avoid reusing the exact same `namespace.key` in different addons.
