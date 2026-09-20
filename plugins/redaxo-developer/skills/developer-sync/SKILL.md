---
name: developer-sync
description: File-to-database synchronization of the developer addon, and why an edited file has no effect. Use when a change to a module, template, action or YForm email file does not show up in the frontend or backend, when `developer:sync` reports success but the database still holds the old code, when a backend edit disappeared after a `git pull`, checkout or branch switch, when deciding whether to use `--force-db` / `--force-files`, or whenever files under `redaxo/data/addons/developer/` (or a relocated developer base path) are edited.
---

# developer addon: file ↔ database synchronization

## The core fact

**REDAXO renders the database, not the file.** Template, module and action code lives in `rex::getTable('template')`, `rex::getTable('module')` and `rex::getTable('action')`. The files the `developer` addon exposes are a mirror. The referee between the two sides is the **modification timestamp**.

Consequence, and it is the opposite of every other codebase: **saving the file does not apply the change.** A change counts as made only once it is in the database.

## The working loop

```bash
# 1. edit the file under the developer base path
# 2. push it into the database
redaxo/bin/console developer:sync
# 3. drop the generated caches
redaxo/bin/console cache:clear
# 4. verify in the browser, not by reading the source
```

Prove it rather than assume it:

```sql
SELECT updatedate, INSTR(output, '<the new fragment>') FROM rex_module WHERE id = <id>;
```

Step 3 is not optional. The addon clears only the caches of articles that **already** use the module (`setEditedCallback` in `manager.php`); template edits clear that one template's cache. For a **newly created** item nothing is cleared at all.

## Where the files are

Default: `redaxo/data/addons/developer/`, with `templates/`, `modules/`, `actions/` and `yform_email/` beneath it.

**Other addons may relocate that path** — the `theme` addon, for example, redirects it to `theme/private/redaxo/` when its own `synchronize` setting is on. The previous directory then stays behind as a dead twin that still looks plausible and is never read again. Check before the first edit:

```bash
grep -rn "setBasePath" --include=*.php redaxo/src/addons/ | grep -v "addons/developer/lib"
```

The redirect runs on the `DEVELOPER_MANAGER_START` extension point, which also fires on the console. If the relocating addon's own config switch is off, the default path applies again — and the well-filled directory is suddenly the dead one.

## Why the sync does not run by itself

Four gates. Three of them are invisible in the file tree:

| Gate | Condition |
|---|---|
| Console | `boot.php` returns immediately when `rex::getConsole()` is set. Only `developer:sync` gets through, because the command calls `rex_developer_manager::start()` directly. Hand-written PHP scripts accomplish nothing. |
| Privilege | The `PACKAGES_INCLUDED` callback requires `rex::isDebugMode()` **or** a valid admin backend session. |
| Configuration | `sync_frontend` / `sync_backend` in `rex_config`, namespace `developer`. |
| Session domain | Logged into the backend on `www.`, calling the frontend without it (or http vs https) means no session, so no sync. Same for multi-domain setups. |

An anonymous frontend request with debug mode off **never** synchronizes. That looks exactly like a broken patch and is not one.

## Never force it

`--force-db` and `--force-files` are not fallbacks for "the sync will not take". They act **globally across every item** of every synchronizer, not on the one being worked on:

- `--force-files` overrides all database state with the files **and deletes database items that have no directory**. Any backend work not yet materialized as a file is gone.
- `--force-db` overwrites all files with the database state — including the edit just made.

**When the sync does not take, the timestamp is the cause, not a lack of force.** `touch` achieves the same thing for one item, without the collateral damage:

```bash
# FIRST: is the change even still in the file?
git diff -- "<base path>/modules/<dir>/output.php"

stat -c '%y %n' "<base path>/modules/<dir>/"*.php   # file mtime vs. rex_module.updatedate
touch "<base path>/modules/<dir>/output.php"
redaxo/bin/console developer:sync
```

That `git diff` is not optional. `developer:sync` prints "Synchronized developer files." and exits 0 **regardless of which direction it synchronized**. If `updatedate` was newer than the file, the run did not ignore the edit — it overwrote it. Syncing again without checking overwrites it a second time. Restore the work first, then `touch`.

Use the force options only after telling the user explicitly what will be lost.

## The timestamp arbitration

Per file, in `synchronizer.php`:

```php
if ($dbUpdated > $fileUpdated && $dbUpdated > $lastUpdated || !$fileExists) {
    rex_file::put($filePath, $item->getFile($file));  // DB wins -> file overwritten
    touch($filePath, $updated);                       // file takes the DB timestamp
} elseif ($fileUpdated > $dbUpdated) {
    $updateFiles[$file] = rex_file::get($filePath);   // file wins -> DB overwritten
}
```

A tie does nothing. `$lastUpdated` comes from the addon's own memory of the previous run (`developer.items` in `rex_config`), not from the file.

Two traps follow:

- **`git pull`, `checkout` and branch switches set the mtime to now.** The checked-out state then beats any backend edit not yet pulled into files — silently. Run `developer:sync` **before** pulling, so pending backend work exists as a file first.
- **A database backup import forces `FORCE_DB` internally** (`BACKUP_AFTER_DB_IMPORT`). After a restore, the imported database state wins and overwrites the files.

## Common pitfalls

| Assumption | Reality |
|---|---|
| "The file is saved, so it is done." | The database is rendered. Without a sync, nothing happened. |
| "`developer:sync` exited 0, so it worked." | The message only says it ran, not in which direction. It may have overwritten the edit. |
| "My patch has no effect, the code must be wrong." | Rule out the sync before suspecting the code. |
| "I will write a small PHP script that synchronizes." | `boot.php` returns on the console. Only `developer:sync` works. |
| "Ask the user to log into the backend." | The console is enough. A backend login is the fallback, not the first move. |
| "`--force-files` just enforces my file." | It acts globally and deletes database items without a directory. `touch` on the one file is enough. |
| "`output.php` is being ignored." | With `developer.prefix` enabled the addon reads `<id>.<name>.output.php`. Nobody reads the unprefixed file. |
| "The directory is ignored entirely." | A `.rex_ignore` inside it — left behind by an item deleted in the backend. See `developer-items`. |

Creating, naming, renaming and deleting items through the file system is a separate topic: see the `developer-items` skill.
