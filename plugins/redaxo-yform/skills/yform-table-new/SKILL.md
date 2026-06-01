---
name: yform-table-new
description: Creating a new YForm table programmatically in REDAXO via an idempotent setup script. Use when the user wants to create a new rex_yform_* table, scaffold YForm fields in code, or add a table that should be reproducible via install/setup script. Invoke as /redaxo-yform:yform-table-new when the user explicitly asks to scaffold a new YForm table.
---

# YForm – Creating a New Table via Setup Script

For production code, always define YForm tables programmatically (not only via the Table Manager UI). An idempotent script can be committed, re-run safely, and serves as the source of truth for the table structure.

## Steps

1. Create the script (e.g. `setup/mytable_setup.php` in your addon or theme)
2. Add the REDAXO bootstrap
3. Check if the table already exists before inserting
4. Insert the table record into `rex_yform_table`
5. Add fields via `rex_yform_manager_table_api::setTableField()` — **never raw SQL**
6. Generate the schema
7. Optionally seed initial records
8. Run the script

## Bootstrap

```php
<?php
if (php_sapi_name() !== 'cli') { die('CLI only.'); }
$REX = [];
$REX['REDAXO']         = false;
$REX['HTDOCS_PATH']    = dirname(__DIR__, N) . '/'; // adjust N to reach the webroot
$REX['BACKEND_FOLDER'] = 'redaxo';
require dirname(__DIR__, N) . '/redaxo/src/core/boot.php';
```

## Table record

```php
$tableName = 'rex_my_table';

$check = rex_sql::factory();
$check->setQuery('SELECT COUNT(*) AS cnt FROM rex_yform_table WHERE table_name = :t', ['t' => $tableName]);
if ((int)$check->getValue('cnt') === 0) {
    $sql = rex_sql::factory();
    $sql->setTable('rex_yform_table');
    $sql->setValue('table_name', $tableName);
    $sql->setValue('name', 'My Table');        // backend menu label
    $sql->setValue('status', 1);               // 1 = visible in menu
    $sql->setValue('list_amount', 30);
    $sql->setValue('list_sortfield', 'id');
    $sql->setValue('list_sortorder', 'ASC');
    $sql->setValue('search', 1);
    $sql->setValue('history', 0);
    $sql->setValue('export', 0);
    $sql->setValue('import', 0);
    $sql->insert();
    echo "Table record created.\n";
}
```

## Adding fields via `setTableField()`

Always use `rex_yform_manager_table_api::setTableField()` — never insert into `rex_yform_field` directly.

```php
$t = $tableName;

// text / integer / email / url / checkbox / date
rex_yform_manager_table_api::setTableField($t, [
    'type_id'      => 'value',
    'type_name'    => 'text',
    'name'         => 'title',
    'label'        => 'Title',
    'db_type'      => 'varchar(191)',
    'not_required' => '',
]);

// choice (select)
rex_yform_manager_table_api::setTableField($t, [
    'type_id'   => 'value',
    'type_name' => 'choice',
    'name'      => 'status',
    'label'     => 'Status',
    'choices'   => '{"Active":"active","Inactive":"inactive"}',
    'multiple'  => '0',
    'expanded'  => '0',
]);

// be_manager_relation (relation to another table)
rex_yform_manager_table_api::setTableField($t, [
    'type_id'   => 'value',
    'type_name' => 'be_manager_relation',
    'name'      => 'category_id',
    'label'     => 'Category',
    'table'     => 'rex_other_table',
    'field'     => 'name',
    'type'      => '0',
]);

// validate – required
rex_yform_manager_table_api::setTableField($t, [
    'type_id'   => 'validate',
    'type_name' => 'empty',
    'name'      => 'title',
    'label'     => 'Title is required',
    'no_db'     => '1',
]);

// validate – unique
rex_yform_manager_table_api::setTableField($t, [
    'type_id'   => 'validate',
    'type_name' => 'unique',
    'name'      => 'title',
    'label'     => 'Title already exists',
    'no_db'     => '1',
    'table'     => $tableName,
    'field'     => 'title',
]);
```

## Generate schema

```php
rex_yform_manager_table::deleteCache();
$tableObj = rex_yform_manager_table::get($tableName);
if ($tableObj) {
    rex_yform_manager_table_api::generateTableAndFields($tableObj);
    echo "Schema generated.\n";
}
```

## Seed initial records (optional)

```php
$exists = rex_sql::factory();
$exists->setQuery("SELECT COUNT(*) AS cnt FROM $tableName WHERE name = :n", ['n' => 'Default']);
if ((int)$exists->getValue('cnt') === 0) {
    $row = rex_sql::factory();
    $row->setTable($tableName);
    $row->setValue('name', 'Default');
    $row->setValue('status', 'active');
    $row->insert();
}
```

## Run

```bash
php path/to/mytable_setup.php 2>&1 | grep -v "^PHP Deprecated\|^Deprecated"
```

## Common pitfalls

- **Never insert into `rex_yform_field` directly** — always use `setTableField()`. Direct inserts bypass the field registry and schema sync, leading to inconsistent state.
- **`php` field type**: does not create a DB column (`no_db: '1'`). It accesses the row via `$params['main_id']`, **not** via `$params['value']`. Don't confuse it with value fields.
- **Schema must be regenerated** after every field change. `generateTableAndFields()` creates/alters the MySQL table to match the field config. Forgetting this leaves the DB out of sync.
- **Cache**: call `rex_yform_manager_table::deleteCache()` before `generateTableAndFields()`, otherwise stale config may be used.
- **Idempotency**: wrap every insert in an existence check so the script can be re-run safely during deployments or after rollbacks.
