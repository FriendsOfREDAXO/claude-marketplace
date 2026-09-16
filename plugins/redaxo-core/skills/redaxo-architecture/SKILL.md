---
name: redaxo-architecture
description: REDAXO project structure, request lifecycle, and core classes. Use when the user works on a REDAXO codebase, mentions REDAXO directory layout, asks about rex_addon/rex_config/rex_path/rex_url/rex_clang/rex_request, or when files under redaxo/src/, redaxo/data/, or redaxo/cache/ appear in the conversation.
---

# REDAXO Architecture

REDAXO is a PHP CMS organized around a small core plus addons. Almost every feature lives in an addon – including `structure` (articles/categories), `mediapool` (media management), `users`, and `cronjob`.

## Directory layout

```
redaxo/
├── src/
│   ├── core/              # Core classes (rex_*, never edit)
│   └── addons/
│       ├── structure/     # Articles & categories
│       ├── mediapool/     # Media library
│       ├── users/         # Backend user management
│       └── <your-addon>/  # Custom addons live here
├── data/
│   ├── addons/            # Addon-managed runtime data
│   ├── core/              # Core runtime data (config.yml lives here)
│   └── log/               # System logs
└── cache/                 # Generated files; safe to delete
```

The user's project root usually contains `redaxo/` plus `assets/` (public mirror of addon assets) and a frontend directory (often `public/` or root).

## Request lifecycle (high level)

1. `index.php` boots `rex` (frontend) or `redaxo/index.php` boots the backend.
2. Core loads `config.yml` and the `package.yml` of every installed addon.
3. Each addon's `boot.php` runs – this is where extension points get registered.
4. For frontend requests: structure resolves the article via URL → loads template → executes modules.
5. `OUTPUT_FILTER` runs on the final HTML before it's sent.

## Key core classes

- `rex_addon::get('addon_name')` – access addon instance, config, paths, version
- `rex_config::get('namespace', 'key', $default)` / `set()` – persistent key-value config
- `rex_path::addon('addon_name', 'sub/path')` – absolute filesystem paths
- `rex_path::frontend('relative/path')` – paths in the public frontend
- `rex_url::backendController(['page' => 'addon/section'])` – backend URLs
- `rex_url::frontendController(['article_id' => 5])` – frontend URLs (use yrewrite if installed)
- `rex_clang::getCurrentId()` / `rex_clang::getAll()` – language handling
- `rex_request::get('key', 'string', '')` / `post()` – type-safe input
- `rex_response::sendJson($data)` – JSON responses for AJAX
- `rex_logger::factory()->log('error', $msg)` – write to system log
- `rex::isBackend()` / `rex::isFrontend()` / `rex::getUser()` – context

## Conventions

- **Never edit core files.** Hook into the system via extension points or override via your own addon.
- Addon code goes in `redaxo/src/addons/<name>/` during development; published addons get distributed via the REDAXO Installer or [redaxo.org](https://redaxo.org).
- Public assets are placed in your addon's `assets/` folder. They're symlinked/copied to `assets/addons/<name>/` on install.
- For paths, always use `rex_path::*` instead of hardcoding `__DIR__ . '/../...'`.
- For URLs, always use `rex_url::*` so the system stays portable across installations and subdirectory deployments.

## Common pitfalls

- Forgetting `rex::isBackend()` checks in `boot.php` – many extension points fire in both contexts.
- Using `$_GET` / `$_POST` directly instead of `rex_request` – loses type safety and the built-in CSRF token handling.
- Hardcoding language IDs (assuming `1 = German`). Use `rex_clang::getCurrent()` and `rex_clang::getCurrentId()`.
- Changing `rex_config` with a direct SQL `UPDATE` on `rex::getTable('config')` – `rex_config` reads from the cache file `rex_path::coreCache('config.cache')` and only falls back to the table when that file is missing, so the old value stays active without any error. Values are also stored JSON-encoded. Use `rex_config::set()` (written to the table at shutdown, cache file invalidated); after an unavoidable SQL change call `rex_config::refresh()` or clear the cache.
