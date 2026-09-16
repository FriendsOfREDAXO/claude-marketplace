---
name: yform-tables
description: Defining and managing YForm tables in REDAXO – Table Manager backend UI, rex_yform_manager_table::ensure() in install.php, schema migrations, and tableset import/export. Use when the user works with YForm tables, edits install.php for a yform-using addon, defines field configurations, configures table-level options (pagination, search, history, export/import), or migrates table schemas between versions.
---

# YForm Tables

YForm extends REDAXO with custom database tables that get a backend UI (Table Manager) plus an ORM-style dataset API. A "table" in YForm = a real DB table + a configuration of fields/validators/relations stored in `rex_yform_table` and `rex_yform_field`.

## Two ways to define tables

1. **Via Table Manager UI** – click together fields in the backend. Good for prototyping. Export to JSON for version control.
2. **Programmatically** in `install.php` of an addon – idempotent, safe to commit, runs on every install/update.

For production code, always use option 2.

## Table-level options (Table Manager UI)

| Option | Description |
|---|---|
| `prio` | Sort position among tables in the menu |
| `table_name` | MySQL table name (must include `rex_` prefix) |
| `name` | Backend menu label |
| `description` | Documentation text shown to backend editors |
| `status` | 1 = show in menu, 0 = hidden |
| `list_amount` | Pagination – records per page (default 30) |
| `list_sortfield` / `list_sortorder` | Default sort column and `ASC` / `DESC` |
| `search` | Enable backend search field |
| `hidden` | Hide from menu but keep API access |
| `export` | Show export button |
| `import` | Show import button |
| `history` | Enable record versioning |

## Programmatic definition – `install.php`

```php
<?php
// install.php
$table = rex_yform_manager_table::get(rex::getTable('team_member'));

// 1. Ensure the underlying SQL table exists with the right columns
rex_sql_table::get(rex::getTable('team_member'))
    ->ensurePrimaryIdColumn()
    ->ensureColumn(new rex_sql_column('name', 'varchar(255)', false, ''))
    ->ensureColumn(new rex_sql_column('email', 'varchar(255)', false, ''))
    ->ensureColumn(new rex_sql_column('role', 'varchar(64)', false, ''))
    ->ensureColumn(new rex_sql_column('photo', 'text', true))
    ->ensureColumn(new rex_sql_column('bio', 'text', true))
    ->ensureColumn(new rex_sql_column('status', 'tinyint(1)', false, '1'))
    ->ensureColumn(new rex_sql_column('createdate', 'datetime', false))
    ->ensureColumn(new rex_sql_column('updatedate', 'datetime', false))
    ->ensure();

// 2. Configure YForm metadata for the table
rex_yform_manager_table_api::setTable([
    'table_name'  => rex::getTable('team_member'),
    'name'        => 'Team members',
    'list_amount' => 30,
    'prio'        => 1,
    'hidden'      => 0,
    'status'      => 1,
    'export'      => 1,
    'import'      => 1,
]);

// 3. Define fields (one entry per column the editor sees in the backend)
rex_yform_manager_table_api::setTableField(rex::getTable('team_member'), [
    'type_id'   => 'value',
    'type_name' => 'text',
    'name'      => 'name',
    'label'     => 'Name',
    'prio'      => 1,
]);

rex_yform_manager_table_api::setTableField(rex::getTable('team_member'), [
    'type_id'   => 'value',
    'type_name' => 'text',
    'name'      => 'email',
    'label'     => 'Email',
    'prio'      => 2,
]);

rex_yform_manager_table_api::setTableField(rex::getTable('team_member'), [
    'type_id'   => 'value',
    'type_name' => 'be_manager_relation',
    'name'      => 'role',
    'label'     => 'Role',
    'table'     => rex::getTable('team_role'),
    'field'     => 'name',
    'type'      => '0', // 0=single select, 1=multi select, 2=popup, 3=popup multi, 4=inline 1:n, 5=inline m:n
    'prio'      => 3,
]);

rex_yform_manager_table_api::setTableField(rex::getTable('team_member'), [
    'type_id'   => 'value',
    'type_name' => 'be_media',
    'name'      => 'photo',
    'label'     => 'Photo',
    'prio'      => 4,
]);

rex_yform_manager_table_api::setTableField(rex::getTable('team_member'), [
    'type_id'   => 'value',
    'type_name' => 'textarea',
    'name'      => 'bio',
    'label'     => 'Bio',
    'prio'      => 5,
]);

// 4. Validation
rex_yform_manager_table_api::setTableField(rex::getTable('team_member'), [
    'type_id'   => 'validate',
    'type_name' => 'empty',
    'name'      => 'name',
    'message'   => 'Please enter a name.',
    'prio'      => 100,
]);

rex_yform_manager_table_api::setTableField(rex::getTable('team_member'), [
    'type_id'             => 'validate',
    'type_name'           => 'type',
    'name'                => 'email',
    'type_name_internal'  => 'email',
    'message'             => 'Please enter a valid email address.',
    'prio'                => 101,
]);
```

## Migrations between versions – `update.php`

Always gate each schema change with a version check so reinstalls and partial upgrades stay correct.

```php
<?php
// update.php
$installed = rex_addon::get('my_addon')->getVersion();

if (rex_version::compare($installed, '1.1.0', '<')) {
    rex_sql_table::get(rex::getTable('team_member'))
        ->ensureColumn(new rex_sql_column('twitter', 'varchar(64)', true))
        ->ensure();

    rex_yform_manager_table_api::setTableField(rex::getTable('team_member'), [
        'type_id'   => 'value',
        'type_name' => 'text',
        'name'      => 'twitter',
        'label'     => 'Twitter handle',
        'prio'      => 6,
    ]);
}
```

## Tableset import/export (JSON)

Tablesets let you ship a table definition as JSON (Table Manager → Export). Useful for:

- Sharing prototypes between projects
- Generating fixtures for tests
- Snapshotting "known-good" configurations

Structure:

```json
{
    "rex_team_member": {
        "table": {
            "status": 1,
            "table_name": "rex_team_member",
            "name": "Team members",
            "list_amount": 30,
            "search": 1,
            "hidden": 0,
            "export": 1,
            "import": 0
        },
        "fields": [
            {
                "type_id": "value",
                "type_name": "text",
                "db_type": "varchar(255)",
                "name": "name",
                "label": "Name",
                "list_hidden": 0,
                "search": 1
            }
        ]
    }
}
```

For production projects, prefer programmatic definition in `install.php` over imported tablesets — tablesets aren't versioned with the rest of your code.

## Common pitfalls

- Defining a YForm field without first ensuring the underlying SQL column – the field shows up in the backend but throws on save.
- Forgetting `prio` – fields appear in random order. Use ascending integers per topic block (1-99 = values, 100-199 = validators).
- Hardcoding `rex_team_member` instead of `rex::getTable('team_member')` – breaks multi-instance / non-default-prefix setups.
- Calling `rex_yform_manager_table_api::setTable()` without a `name` key – on insert **and** on update it sets `name` to the table name, so a later call that only changes e.g. `list_amount` silently replaces the backend label. Pass `name` every time; other missing keys stay untouched.
- Expecting a changed `db_type` to alter an existing column – `generateTableAndFields()` only re-ensures existing columns when the table has `schema_overwrite = 1` (the column default for tables inserted via `setTable()`; `migrateTable()` defaults to `0`). With `0` the field definition changes, the column keeps its old type.
- Expecting a deleted field to drop its column – deleting a field in the Table Manager or via `removeTablefield()` keeps column and data. Only `generateTableAndFields($table, true)` (Table Manager button "Update table/s with field deletion") drops them, and it drops **every** column without a YForm field except `id`, including columns added on purpose via `rex_sql_table`.
- Not including `createdate` / `updatedate` columns – YForm's standard `datestamp` fields silently break without them.
- Defining validators before the corresponding `value` field – sort by `prio` so values come first, then validators.
- Using `rex_yform_manager_table_api::setTable()` without first creating the SQL table via `rex_sql_table::ensure()` – YForm metadata then references a nonexistent table.
