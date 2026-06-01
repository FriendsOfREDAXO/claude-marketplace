---
name: sprog-add
description: Adding Sprog wildcards to the database in REDAXO. Use when the user wants to register new text wildcards, i18n labels, or translations via the Sprog addon, mentions rex_sprog_wildcard, or asks how to add/update Sprog entries programmatically.
---

# Sprog – Adding Wildcards

Sprog manages translatable text wildcards stored in `rex_sprog_wildcard`. Each wildcard has one row per language (`clang_id`). Use `{{namespace.key}}` in modules and templates to output them.

## When a setup script already exists

Append the new wildcards to the existing script and re-run it. The idempotent check (see below) prevents duplicates.

## Creating a new setup script

Place the script at a path your project can reach via CLI (e.g. `setup/sprog_setup.php` inside your addon or theme).

### Bootstrap

```php
<?php
if (php_sapi_name() !== 'cli') { die('CLI only.'); }
$REX = [];
$REX['REDAXO']         = false;
$REX['HTDOCS_PATH']    = dirname(__DIR__, N) . '/'; // adjust N to reach the webroot
$REX['BACKEND_FOLDER'] = 'redaxo';
require dirname(__DIR__, N) . '/redaxo/src/core/boot.php';
```

### Wildcard insert

```php
$wildcards = [
    'namespace.key' => ['de' => 'Deutsch', 'en' => 'English'],
    'namespace.other' => ['de' => 'Weiteres', 'en' => 'Another'],
];

// Load all languages
$clangSql = rex_sql::factory();
$clangSql->setQuery('SELECT id, code FROM rex_clang ORDER BY id ASC');
$clangs = [];
foreach ($clangSql as $row) {
    $clangs[(int)$clangSql->getValue('id')] = $clangSql->getValue('code');
}

// Next free id
$maxIdSql = rex_sql::factory();
$maxIdSql->setQuery('SELECT COALESCE(MAX(id), 0) + 1 AS next_id FROM rex_sprog_wildcard');
$nextId = (int)$maxIdSql->getValue('next_id');

foreach ($wildcards as $key => $translations) {
    $check = rex_sql::factory();
    $check->setQuery(
        'SELECT COUNT(*) AS cnt FROM rex_sprog_wildcard WHERE wildcard = :key',
        ['key' => $key]
    );
    if ((int)$check->getValue('cnt') > 0) {
        echo "SKIP $key\n";
        continue;
    }
    $id = $nextId++;
    foreach ($clangs as $clangId => $code) {
        $sql = rex_sql::factory();
        $sql->setTable('rex_sprog_wildcard');
        $sql->setValue('id', $id);
        $sql->setValue('clang_id', $clangId);
        $sql->setValue('wildcard', $key);
        $sql->setValue('replace', $translations[$code] ?? $translations['de'] ?? '');
        $sql->setValue('createdate', date('Y-m-d H:i:s'));
        $sql->setValue('createuser', 'setup');
        $sql->setValue('updatedate', date('Y-m-d H:i:s'));
        $sql->setValue('updateuser', 'setup');
        $sql->setValue('revision', 0);
        $sql->insert();
    }
    echo "OK $key\n";
}
```

### Run

```bash
php path/to/sprog_setup.php
```

## Output in modules/templates

```
{{namespace.key}}
```

## Common pitfalls

- **Language fallback**: the script falls back to `de` if the current language code has no translation. Add all languages you actually use to avoid empty wildcards.
- **id collision**: `COALESCE(MAX(id), 0) + 1` works for sequential runs but breaks under concurrent inserts. Fine for CLI setup scripts; don't use in request handlers.
- **Cache**: Sprog caches wildcard replacements. After bulk inserts, clear the REDAXO cache (`rex_delete_cache()` or via the backend) so changes appear immediately.
- **Wildcard naming**: use `namespace.key` notation to group related wildcards and avoid key conflicts between addons.
