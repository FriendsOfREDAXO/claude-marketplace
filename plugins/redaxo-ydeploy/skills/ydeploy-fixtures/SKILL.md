---
name: ydeploy-fixtures
description: What YDeploy carries from the local database to the server and what it does not – fixture tables (fixtures.yml, config.fixtures.tables in the ydeploy package.yml), the rex_config namespaces that are synced (core package-config/package-order, media_manager, sprog …) and those that are not, adding own fixture tables with conditions, why templates/modules/actions are left to the developer addon, and why install.php/update.php never run on deploy. Use when the user asks whether a backend setting, media manager type, metainfo field, YForm table definition, addon config or installed addon "goes live" with the deploy, edits fixtures.yml, wants to add a table to the fixtures, or an addon works locally but is unconfigured/empty on the server.
---

# YDeploy Fixtures – what travels with a deploy

YDeploy moves **structure** (all prefixed tables → `schema.yml`) and **selected data** (fixture tables → `fixtures.yml`) via migrations. Everything else in the database stays where it is. Content (articles, slices, media, YForm datasets, users) is not deployed – unless its table is explicitly added to the fixtures (see below).

## Default fixture tables

From `config.fixtures.tables` in the ydeploy `package.yml` (table names without prefix):

| Table | What it is |
|---|---|
| `config` – only namespaces `core`, `mblock`, `media_manager`, `mform`, `rexstan`, `sprog` | selected `rex_config` rows |
| `media_manager_type`, `media_manager_type_effect` | Media Manager types and effects |
| `metainfo_field`, `metainfo_type` | Metainfo definitions |
| `yform_table`, `yform_field` | YForm table manager definitions |
| `module_action` | Module ↔ action assignment |
| `markitup_profiles`, `redactor_profile`, `redactor2_profiles` | Editor profiles |

Rows are compared **per primary key**. On the server a fixture change is applied as `INSERT … ON DUPLICATE KEY UPDATE` (the local row wins) or `DELETE … WHERE <pk> = …`.

## Consequences

- **Installing/uninstalling an addon locally deploys it.** `core/package-config` and `core/package-order` are fixture rows. After `ydeploy:diff` the migration carries the new package state; after deploy the addon is installed and active on the server.
- **But `install.php` never runs on the server** (nor `update.php`, `uninstall.php`). YDeploy has no install step. What an addon's install routine does besides creating tables is missing on the server:
  - `default_config` from its `package.yml` and anything it writes via `rex_config::set()` under its own namespace (only the namespaces listed above travel),
  - seed rows it inserts into its own tables (the tables arrive via `schema.yml`, but empty),
  - files it copies into its data folder.
  After deploying an addon with a real install routine, check its config and tables on the server. Options: re-install it on the server once while its tables are still empty, add its config namespace to the fixtures, or write a migration.
- **Addon updates**: the new files arrive with the release, schema changes arrive via the diff, but the addon's `update.php` does not run.
- **Templates, modules and actions are not fixtures.** They are synced by the [developer addon](https://github.com/FriendsOfREDAXO/developer) from files; the deploy runs `developer:sync --force-files` after the migrations (see `ydeploy-deployer`). Don't add `template`, `module` or `action` to the fixtures.
- **YForm table definitions travel, YForm data does not.** `rex_yform_field` rows include `prio`, `label`, `list_hidden` etc. – changing field order or labels in the local table manager produces fixture changes that overwrite the server's definitions.

## Adding own fixture tables

The list is an addon property, so it can be extended from the project addon's `boot.php`. Pattern from the [Yak README](https://github.com/yakamara/yak#zus%C3%A4tzliche-datenbank-tabellen-synchronisieren):

```php
// src/addons/project/boot.php (local instance)
if (rex::isBackend() && rex_addon::get('ydeploy')->isAvailable()) {
    rex_extension::register('PACKAGES_INCLUDED', static function () {
        $config = rex_addon::get('ydeploy')->getProperty('config');

        // never add action, module, module_action, template – the developer addon syncs those
        $config['fixtures']['tables'] = array_merge([
            'sprog_wildcard' => null,                       // whole table
        ], $config['fixtures']['tables']);
        $config['fixtures']['tables']['config'][] = ['namespace' => 'my_addon']; // one more rex_config namespace

        rex_addon::get('ydeploy')->setProperty('config', $config);
    });
}
```

- Key = table name **without** prefix. `null` = all rows. An array = list of conditions; each condition is a column ⇒ value map (AND), the conditions are OR-ed (`config` uses this for namespaces).
- The table **must have a primary key**, otherwise `ydeploy:diff` throws.
- With conditions, rows outside the conditions are neither exported nor deleted on the server.
- Run `ydeploy:diff` afterwards; the first run exports all matching rows as upserts.

Only add tables whose content is **owned by developers**. A fixture table is overwritten on every deploy that touches it – if editors maintain the same rows on the live system, their changes are lost. The Yak README uses this deliberately before launch (even `article`, `article_slice`, `clang`, `media`, `media_category` to ship initial content) and says to remove those entries as soon as editors work on the live instance.

Be careful with config namespaces that hold secrets or per-environment values (API keys, hosts): as fixtures they would be copied from local to every server.

## Protected backend pages

On a deployed instance (`rex_ydeploy::factory()->isDeployed()`), ydeploy hides the backend pages whose data is managed by deployment, so nobody changes them on the server: installer, packages (AddOns), templates, modules, media manager, metainfo, editor profiles, rexstan, and in YForm the setup/docs pages plus table edit/migrate/import/field management (and YForm email templates if the developer addon syncs them). Opening such a page shows a warning with an "Unlock and open it anyway" link (per session). The list is `config.protected_pages` in the ydeploy `package.yml`.

## Common pitfalls

- **"It works locally, the server has no settings"** → the addon's `rex_config` namespace is not a fixture, or its `install.php` seeded data. See Consequences.
- **Editing fixture data on the server** (unlocking a protected page and changing a media manager type there) → overwritten by the next deploy that includes that row.
- **Reviewing only `schema.yml`** – fixture changes (e.g. YForm field order, labels) are only visible in `fixtures.yml` and the migration's `INSERT … ON DUPLICATE KEY UPDATE` block.
- **Adding content tables to the fixtures** "to get the data live once" and forgetting to remove them – later deploys overwrite editor changes.
