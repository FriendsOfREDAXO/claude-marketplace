---
name: ydeploy-diff-migrations
description: YDeploy database migrations in REDAXO – how `ydeploy:diff` compares the local database with schema.yml/fixtures.yml and generates a migration file, how `ydeploy:migrate` runs pending migrations (order, rex_ydeploy_migration status table, --fake), writing a migration by hand (`ydeploy:diff --empty --unmarked`), file naming and timestamps, and what the generated code looks like (rex_sql_table ensure/alter, FOREIGN_KEY_CHECKS). Use when the user runs ydeploy:diff or ydeploy:migrate, reviews or writes a file in data/addons/ydeploy/migrations/, sees unexpected removeColumn/dropTable/DELETE in a generated migration, or asks whether a migration already ran.
---

# YDeploy Diff & Migrations

All files live in the ydeploy data folder – `redaxo/data/addons/ydeploy/` (classic) or `var/data/addons/ydeploy/` (Yak), i.e. `rex_path::addonData('ydeploy')`:

```
schema.yml        # structure of all tables with the REDAXO table prefix (+ views)
fixtures.yml      # rows of the configured fixture tables (see ydeploy-fixtures)
migrations/       # one PHP file per change set
```

## `ydeploy:diff`

```bash
bin/console ydeploy:diff [--empty] [--unmarked]
```

1. Reads all tables with the REDAXO table prefix (`rex_` by default) from the **current database**. Tables without the prefix are ignored.
2. Compares them with `schema.yml` and the fixture rows with `fixtures.yml`.
3. If there are differences, writes a migration file and **rewrites `schema.yml` and `fixtures.yml` to the current database state**.
4. Inserts the new migration's timestamp into `rex_ydeploy_migration` – the local DB already has the changes, so the migration counts as executed locally. `--unmarked` skips this.

`--empty` creates a migration file even if nothing changed.

The source of truth is the **local database**, not the code. Whatever the local DB lacks compared to `schema.yml` becomes a deletion.

### What the diff detects

- New / dropped tables, charset or collation changes.
- Added, removed and changed columns, **column order** (generated as `ensureColumn($column, $after)`).
- **Renames**: a column missing from the DB plus a new column with an identical definition (type, nullable, default, extra) is written as `renameColumn()`.
- Primary key, indexes, foreign keys, views.
- Fixture rows added, changed (compared per primary key) or removed.

### Generated file

```php
<?php

$sql = rex_sql::factory();
$sql->setQuery('SET FOREIGN_KEY_CHECKS = 0');

try {
    rex_sql_table::get('rex_my_table')
        ->ensureColumn(new rex_sql_column('title', 'varchar(191)'), 'id')
        ->removeColumn('old_field')
        ->alter();

    $sql->setQuery(<<<'SQL'
        INSERT INTO `rex_yform_field` (`id`, …)
        VALUES
            (12, …)
        ON DUPLICATE KEY UPDATE `table_name` = VALUES(`table_name`), …
        SQL);
} finally {
    $sql = rex_sql::factory();
    $sql->setQuery('SET FOREIGN_KEY_CHECKS = 1');
}
```

Order inside the file: drop views, create tables, alter tables, drop tables, create/replace views, fixtures (upserts, then `DELETE … WHERE <pk> = …`).

## `ydeploy:migrate`

```bash
bin/console ydeploy:migrate [--fake]
```

- Collects files in `migrations/` matching `YYYY-MM-DD HH-MM-SS.micro.php` and runs every file whose timestamp is not in `rex_ydeploy_migration`, **sorted by file name** (= timestamp).
- A migration is a plain PHP file executed via `require` in the REDAXO console context.
- After each successful file its timestamp is inserted. On an exception it stops: earlier files stay marked as done, the failing one and all later ones are not. Output: `Executed 2 of 5, aborted with "<file>"`.
- There is no transaction and no rollback. MySQL DDL commits implicitly, so a half-run migration leaves a half-migrated DB.
- The REDAXO cache is cleared at the end (also after a failure).
- `--fake` marks all pending migrations as executed without running them (e.g. after importing a dump that already contains the changes).
- Also used locally to pull in migrations from other developers after `git pull`.

Check what ran:

```sql
SELECT `timestamp` FROM rex_ydeploy_migration ORDER BY `timestamp` DESC LIMIT 10;
```

## Writing a migration by hand

For data changes the diff cannot express (backfilling values, transforming data, renames the diff misreads):

```bash
bin/console ydeploy:diff --empty --unmarked
```

This updates `schema.yml`/`fixtures.yml`, then creates a file with the skeleton above (`// Add migration stuff here`) and the correct UTC timestamp, **not** marked as executed. Fill it in, then:

```bash
bin/console ydeploy:migrate      # runs it locally and marks it
bin/console ydeploy:diff         # brings schema.yml/fixtures.yml up to date
```

If the migration changed the structure, the second `ydeploy:diff` generates another migration with the same changes (already marked locally). As long as the hand-written file uses the idempotent `rex_sql_table` API (`ensure()`, `ensureColumn()`, `alter()`), both files can run in sequence on the server. If you delete the redundant file, also delete its row from `rex_ydeploy_migration`.

Rules for hand-written migrations:

- **Keep the timestamp format and use the real current UTC time.** The diff command names files in UTC with microseconds. A hand-made name like `2026-01-01 12-00-00.000000.php` can sort after a later generated diff and run in the wrong order on fresh environments.
- **Make them idempotent** (`rex_sql_table::…->ensure()`, `INSERT … ON DUPLICATE KEY UPDATE`, existence checks). On servers they run against a DB you don't see.
- Use `rex_sql` and `rex::getTable()`; see `redaxo-core:redaxo-sql-patterns`.

## Review before committing

Always read the generated migration. Things to look for:

- **`removeColumn`, `dropTable`, `DELETE FROM` you didn't intend** – almost always a stale local DB (see pitfalls).
- **A `renameColumn` that is really "drop A, add B"** – if you drop a column and add a different one with the same definition in the same diff, ydeploy treats it as a rename and keeps the old data under the new name. Run `ydeploy:diff` between the two steps if that is not what you want.
- **Many `ensureColumn(…, 'after')` lines** without a real change – the physical column order differs from `schema.yml`.
- **Fixture changes** – they overwrite rows on the server by primary key (see `ydeploy-fixtures`).

Commit the migration together with `schema.yml` and `fixtures.yml`.

## Common pitfalls

- **Diff on a stale local DB.** After `git pull` / rebase / branch switch, `schema.yml` contains changes the local DB doesn't have yet. `ydeploy:diff` now generates a migration that **removes** them and resets `schema.yml` to the old state. Always run `ydeploy:migrate` first, then `ydeploy:diff`.
- **Discarding a generated migration file** without deleting its row in `rex_ydeploy_migration` (the diff marked it) and without resetting `schema.yml`/`fixtures.yml` – the next diff won't regenerate those changes.
- **Merge conflicts in `schema.yml`/`fixtures.yml`** – don't hand-merge them. Take one side, run `ydeploy:migrate` so the local DB has all migrations from both branches, then `ydeploy:diff` and check that nothing unexpected is generated.
- **Changing the DB on the server by hand** – the next deploy's migrations are written against the local state; manual server changes cause failing or conflicting migrations. Make the change locally and deploy it.
- **Assuming `install.php`/`update.php` run on deploy** – they don't. See `ydeploy-fixtures`.
