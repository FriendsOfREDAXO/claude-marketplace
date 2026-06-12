---
name: redaxo-addon-development
description: Building and maintaining custom REDAXO addons – package.yml, boot.php, install/update/uninstall, lang files, fragments, backend pages, the assets pipeline (and the reinstall-to-sync rule), cache-busting on asset URLs, vendor dependencies, and the common-pitfalls checklist. Use when the user creates a new addon, edits package.yml, sets up backend pages, ships frontend assets from an addon, integrates Composer dependencies, or asks how to package functionality as a reusable addon.
---

# Building a Custom Addon

Addons live under `redaxo/src/addons/<addon_name>/`. Every addon must have a `package.yml`; everything else is optional but follows conventions.

## Minimum file layout

```
redaxo/src/addons/my_addon/
├── package.yml          # Manifest – required
├── boot.php             # Runs on every request after the addon is enabled
├── install.php          # Runs once on install (DB schema, defaults)
├── uninstall.php        # Runs on uninstall (clean up)
├── update.php           # Runs on version upgrade
├── lang/
│   ├── de_de.lang
│   └── en_gb.lang
├── pages/               # Backend pages (one PHP file per subpage)
├── fragments/           # Reusable view fragments
├── lib/                 # Your PHP classes (autoloaded)
├── functions/           # Procedural helpers (manually included)
└── assets/              # Public files (mirrored to assets/addons/my_addon/)
```

## package.yml

```yaml
package: my_addon
version: '1.0.0'
author: 'Your Name'
supportpage: https://example.com

requires:
    redaxo: '^5.18'
    php: '>=8.1'

# Optional: addons your addon needs to be installed
requires_addons:
    yform: '^4.0'

# Backend page registration
page:
    title: 'My Addon'
    icon: rex-icon fa-cube
    perm: my_addon[]
    subpages:
        list:
            title: 'List'
        settings:
            title: 'Settings'
            perm: my_addon[settings]

# Default config values (read via $addon->getConfig('key'))
default_config:
    items_per_page: 25
    feature_enabled: 0
```

The `perm: my_addon[]` line creates a permission you can grant to user roles in the backend. `my_addon[]` = the addon as a whole; `my_addon[settings]` = a fine-grained permission.

## boot.php – runtime setup

Runs on every request once the addon is enabled. Register extension points and load helpers here.

```php
<?php
$addon = rex_addon::get('my_addon');

// Vendor dependencies (composer.json + vendor/ in your addon) are auto-loaded
// by REDAXO. No manual `require __DIR__ . '/vendor/autoload.php'` needed.
// Note: this only works once the addon is INSTALLED — pre-install code that
// touches vendor/ classes will silently no-op.

// Register fragment directory
rex_fragment::addDirectory($addon->getPath('fragments/'));

if (rex::isBackend() && rex::getUser()) {
    rex_view::addCssFile($addon->getAssetsUrl('backend.css'));
}

// Hook into the system
rex_extension::register('OUTPUT_FILTER', [my_addon_filter::class, 'apply']);

// Optional: include procedural helpers
require_once __DIR__ . '/functions/helpers.php';
```

`$this` inside `boot.php` also refers to the `rex_addon` instance, so `$this->getAssetsUrl(...)` works the same as `$addon->getAssetsUrl(...)`.

Classes under `lib/` are autoloaded automatically: `lib/my_addon_filter.php` → class `my_addon_filter`.

## install.php / update.php / uninstall.php

Throw a `rex_functional_exception` to abort with a user-visible error. Use `rex_sql_table::ensure()` for idempotent schema changes.

```php
<?php
// install.php
rex_sql_table::get(rex::getTable('my_addon_items'))
    ->ensurePrimaryIdColumn()
    ->ensureColumn(new rex_sql_column('title', 'varchar(255)', false, ''))
    ->ensureColumn(new rex_sql_column('data', 'json', false))
    ->ensureColumn(new rex_sql_column('created_at', 'datetime', false))
    ->ensureIndex(new rex_sql_index('title', ['title'], rex_sql_index::UNIQUE))
    ->ensure();

// Set defaults
rex_config::set('my_addon', 'items_per_page', 25);
```

```php
<?php
// uninstall.php
rex_sql_table::get(rex::getTable('my_addon_items'))->drop();
rex_config::removeNamespace('my_addon');
```

```php
<?php
// update.php – runs after the new files are in place
$installed = rex_addon::get('my_addon')->getVersion();
if (rex_version::compare($installed, '1.1.0', '<')) {
    rex_sql_table::get(rex::getTable('my_addon_items'))
        ->ensureColumn(new rex_sql_column('priority', 'int', false, '0'))
        ->ensure();
}
```

## The assets pipeline (read this!)

REDAXO does **not** serve files from `src/addons/<addon>/assets/` directly. On install, those files are copied to the public path `/assets/addons/<addon>/`. Editing source files does **not** update the public version.

After editing any file in `assets/`:

```bash
echo "y" | redaxo/bin/console package:install my_addon
```

This re-syncs assets and re-runs `install.php` (which is why `install.php` must be idempotent). Always use this command instead of manually copying files.

### Cache-busting

Always append `?v=<version>` to asset URLs in fragments / templates / backend pages so browsers don't serve stale JS/CSS:

```php
<script src="<?= $addon->getAssetsUrl('script.js') ?>?v=<?= $addon->getVersion() ?>"></script>
```

In `boot.php` for backend assets:

```php
rex_view::addCssFile($addon->getAssetsUrl('backend.css') . '?v=' . $addon->getVersion());
rex_view::addJsFile($addon->getAssetsUrl('backend.js')   . '?v=' . $addon->getVersion());
```

## Configuration

```php
// Save
rex_config::set('my_addon', 'key', $value);

// Read
$value = rex_addon::get('my_addon')->getConfig('key');

// Defaults via package.yml `default_config:` block (see above)
```

### Checkbox fields in config forms (gotcha)

Unchecked checkboxes don't send a value, so the post-handler must coerce to int:

```php
// pages/config.php
$addon = rex_addon::get('my_addon');

if (rex_post('save', 'bool')) {
    $addon->setConfig('feature_enabled', rex_post('feature_enabled', 'int', 0));
    echo rex_view::success('Settings saved.');
}

// Form
'<input type="checkbox" name="feature_enabled" value="1"
        ' . ($addon->getConfig('feature_enabled') ? 'checked' : '') . ' />';

// In package.yml default_config use 0/1, not true/false
```

## Backend pages (`pages/<subpage>.php`)

Each subpage from `package.yml` resolves to `pages/<subpage>.php`. The file outputs the page body; REDAXO wraps the chrome.

```php
<?php
$content = '<fieldset><legend>Settings</legend>';
// ... form HTML ...
$content .= '</fieldset>';

$fragment = new rex_fragment();
$fragment->setVar('title', rex_i18n::msg('my_addon_title'), false);
$fragment->setVar('body', $content, false);
$fragment->setVar('buttons', $buttons ?? '', false);
echo $fragment->parse('core/page/section.php');
```

Use `rex_be_controller::getCurrentPagePart(N)` to inspect the current subpage path if you need conditional logic.

### `pages/index.php` dispatcher (required for top-level `page:` with subpages)

When `package.yml` uses the singular `page:` key with `subpages`, REDAXO requires a `pages/index.php` file for the main entry. Without it the page errors out with:

> Package "my_addon": the page path "pages/index.php" neither exists as standalone path nor as package subpath ".../pages/index.php"

The convention (used by core addons like `cronjob` and `backup`) is a thin dispatcher that renders the page title **once** and includes the active subpage file:

```php
<?php
// pages/index.php
echo rex_view::title($this->i18n('title'));
rex_be_controller::includeCurrentPageSubPath();
```

The subpage files (`pages/list.php`, `pages/settings.php`) then render **only their body** — calling `rex_view::title()` in a subpage when the dispatcher already does prints the title twice.

### Mounted pages — `pages:` (plural) with dot-notation files

Use the plural `pages:` key to mount additional entries **under existing paths** (e.g. into REDAXO's system menu), separately from your main `page:` entry:

```yaml
pages:
    system/log/my_addon:
        title: 'My Addon Log'
        perm: admin[]
```

Mounted page files use **dot-notation matching the path**: slashes become dots, so `system/log/my_addon` → `pages/system.log.my_addon.php`. This naming only applies to the plural `pages:` form — plain `pages/<subpage>.php` is still correct for subpages of your own top-level `page:`.

## Language files

```
my_addon_title = My Addon
my_addon_item_saved = Item “{0}” has been saved.
```

```php
echo rex_i18n::msg('my_addon_title');
echo rex_i18n::msg('my_addon_item_saved', $title); // {0} replaced
```

Always prefix every key with the addon name to avoid collisions.

## Vendor dependencies (Composer)

Addons can ship their own `composer.json` + `vendor/` directory. REDAXO autoloads vendor classes on install — no manual `require` needed.

- **Never modify files in `vendor/`** — changes get lost on `composer update`.
- If vendor code throws unhelpful exceptions, catch and rethrow with a better message in your own wrapper class.
- Vendor autoloading only kicks in once the addon is **installed**. Code that runs before install (e.g. a custom installer script) won't have access to vendor classes.

### Single-file libraries without a `rex_*` class name

REDAXO's autoloader resolves classes in `lib/` by filename: `lib/rex_foo.php` → class `rex_foo`. A drop-in single-file library that ships as e.g. class `QRencode` (no `rex_*` prefix, no `composer.json`) won't be picked up — neither by the REDAXO autoloader nor by the Composer fallback.

Pattern: wrap it in a `rex_*` class and `require_once` the file from inside the wrapper. The wrapper is autoloaded on first reference; the wrapped file is included only when actually needed.

```
lib/rex_qr_wrapper.php   // class rex_qr_wrapper — autoloaded
lib/phpqrcode.php        // class QRencode — NOT autoloaded
```

```php
class rex_qr_wrapper
{
    public static function encode(string $data): array
    {
        require_once __DIR__ . '/phpqrcode.php';
        return QRencode::factory(QR_ECLEVEL_H, 1, 0)->encode($data);
    }
}
```

Calling code only ever touches `rex_qr_wrapper`, never the raw library. Same trick works for any non-namespaced single-file lib you'd otherwise be tempted to `require` from random places.

## Web components / embeddable widgets

When building a widget that gets embedded on third-party sites:

- Use Shadow DOM for style isolation.
- Store session tokens in `localStorage` (no cookies needed for cross-domain).
- Load conversation/state from the server on every page load (history endpoint) — don't rely on JS state surviving navigation.
- Handle token expiration gracefully (auto-create new session on 401).
- Always cache-bust the script URL: `<script src=".../widget.js?v=2.3.1">`.

## Common pitfalls (checklist)

1. **Edited source assets** but didn't reinstall → old version served from `/assets/addons/`.
2. **No cache-busting** on asset URLs → browser serves stale JS/CSS.
3. **Raw SQL queries** instead of `rex_sql` API → SQL injection risk, no table prefix handling.
4. **Missing `$published = true`** on `rex_api_function` class → 403 from frontend.
5. **Hardcoded API keys** → use `rex_config` instead.
6. **Not using `rex::getTable()`** → missing table prefix breaks multi-instance setups.
7. **Forgot fragment directory** registration in `boot.php`.
8. **Missing `exit`** after `rex_response::sendJson()` → REDAXO appends HTML to JSON.
9. **Inline `<style>` or `<script>` tags** → blocked by CSP. Always external files via `rex_view::addCssFile()` / `rex_view::addJsFile()`.
10. **Top-level `page:` with subpages but no `pages/index.php`** → REDAXO errors on the main entry ("page path ... neither exists ..."). Add a dispatcher (`echo rex_view::title(...); rex_be_controller::includeCurrentPageSubPath();`).
11. **Putting non-idempotent code in `install.php`** without checks – the user may re-install.
12. **Hardcoding asset paths** instead of `rex_url::addonAssets()` – breaks subdirectory deployments.
13. **Skipping `perm:` declarations** – any backend user can then access the page.
14. **Backend download links broken by PJAX** – the backend uses PJAX, which intercepts `<a href>` clicks and loads the response into the current page instead of triggering a download. `rex_api_function` returning binary data doesn't help (REDAXO expects a `rex_api_result` object, not raw output), and `target="_blank"` is unreliable across PJAX configurations. For small payloads (SVG, CSV, generated text) the cleanest fix is to inline the data as a `data:` URL on the link and set `download="filename.ext"` — no server round-trip, PJAX leaves it alone:

    ```php
    $svg = '<svg ...>...</svg>';
    $href = 'data:image/svg+xml;base64,' . base64_encode($svg);
    echo '<a href="' . rex_escape($href, 'html_attr') . '" download="export.svg" class="btn btn-xs">SVG</a>';
    ```

    For larger payloads, render a standalone PHP entry point outside the backend page tree (or use a `rex_api_function` that hands back a `rex_api_result` with a redirect to a streaming endpoint).

## Testing & debugging

```php
// Log exceptions (preferred way)
rex_logger::logException(new rex_exception('Something went wrong'));
rex_logger::logException($exception);

// logError requires 4 params; prefer logException
// rex_logger::logError($errno, $errstr, $errfile, $errline);

// Verbose error output for developers
if (rex::isDebugMode()) {
    // ...
}

// Dump (uses Symfony VarDumper if debug addon is active)
dump($variable);
```

For CLI/diagnostic tasks, see the `redaxo-console-commands` skill — never write standalone bootstrap scripts under `bin/`.
