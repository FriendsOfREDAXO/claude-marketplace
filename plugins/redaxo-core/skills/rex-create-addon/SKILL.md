---
name: rex-create-addon
description: Scaffold a complete REDAXO addon skeleton — package.yml, boot.php, install/uninstall.php, lang files, optional backend page — under redaxo/src/addons/. Distinct from rex-create-module (which scaffolds a single content module). Invoke as /redaxo-core:rex-create-addon when the user wants a new addon scaffolded, says "neues Addon", "Addon erstellen", "Addon-Gerüst", "Addon erzeugen", or asks "how do I start a new REDAXO addon".
---

You are scaffolding a complete REDAXO addon under `redaxo/src/addons/`.

## Steps

1. **Ask the user** for:
   - The addon name (lowercase, snake_case, e.g. `team_directory`)
   - A short description
   - Whether it needs a database table (and if yes, which columns)
   - Whether it needs a backend page (and which subpages, e.g. `list`, `settings`)
   - The minimum REDAXO version (default `^5.18`) and PHP version (default `>=8.1`)

2. **Verify the path.** Find the REDAXO root by looking for a `redaxo/src/addons/` folder upwards from the current working directory. If not found, tell the user this command must run inside a REDAXO project.

3. **Create the directory layout** at `redaxo/src/addons/<name>/`:

```
package.yml
boot.php
install.php
uninstall.php
lang/de_de.lang
lang/en_gb.lang
lib/.gitkeep
fragments/.gitkeep
assets/.gitkeep
```

If backend pages were requested, also create `pages/<subpage>.php` for each.

4. **Generate `package.yml`** from the user's answers. Include a `page:` block with `perm: <name>[]` if backend pages are present.

5. **Generate `install.php`** with `rex_sql_table::ensure(...)` calls for each requested column. Use `rex::getTable('<name>_<entity>')` for table names and never hardcode the `rex_` prefix.

6. **Generate `uninstall.php`** that drops the addon's tables and removes its config namespace via `rex_config::removeNamespace('<name>')`.

7. **Generate `boot.php`** with:
   - A backend asset hook scaffold (commented out unless the user wants assets)
   - An empty space for `rex_extension::register(...)` calls
   - A backend-only guard pattern showing how to require admin/specific permissions

8. **Generate `lang/de_de.lang` and `lang/en_gb.lang`** with at minimum a `<name>_title` key matching the addon's display title.

9. **For each backend page**, generate a starter `pages/<subpage>.php` that uses `rex_fragment` with `core/page/section.php` to render a titled section, plus a placeholder content area.

10. **Do not** add the addon to `redaxo/data/core/config.yml` – installation happens through the REDAXO backend (System → Addons → install). Tell the user to install via the backend after the files are in place.

11. **After generating**, summarize what was created and the next steps:
    - Open the REDAXO backend
    - Go to System → Addons
    - Click Install on the new addon, then Activate
    - Optionally grant the new permission(s) to user roles

Keep the generated code clean and commented in English. Use `rex_i18n::msg('<name>_<key>')` for any user-facing strings, even in the scaffolded backend pages.
