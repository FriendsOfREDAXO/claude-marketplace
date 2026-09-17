---
name: redaxo-sql-patterns
description: Database access in REDAXO using rex_sql – queries, prepared statements, transactions, and proper escaping. Use when the user runs SQL in a REDAXO project, mentions rex_sql, queries the rex_* tables, or writes addon code that touches the database.
---

# rex_sql Patterns

`rex_sql` is REDAXO's database wrapper. It enforces prepared statements, namespaced table prefixes (`rex_`), and ships with a fluent builder for inserts/updates.

Always prefer `rex_sql` over raw PDO so you get the configured connection, prefix handling, and the addon's logging.

## Tables and the `rex::getTable()` helper

REDAXO prefixes all tables (default `rex_`). Never hardcode the prefix – use `rex::getTable('article')` which returns `rex_article`. This keeps code portable across installations that use a different prefix (e.g. multi-instance setups).

```php
$table = rex::getTable('article'); // → "rex_article"
$tableMyAddon = rex::getTable('my_addon_items');
```

## Reading – `setQuery` with placeholders

Always use parameter binding. Never concatenate user input into SQL.

```php
$sql = rex_sql::factory();
$sql->setQuery(
    'SELECT id, name FROM ' . rex::getTable('article') . '
     WHERE clang_id = :clang AND status = :status
     ORDER BY priority',
    ['clang' => rex_clang::getCurrentId(), 'status' => 1]
);

foreach ($sql as $row) {
    echo (int) $row->getValue('id') . ': ' . rex_escape($row->getValue('name')) . "\n";
}
```

Iterating the `$sql` object yields one row at a time. Inside the loop, `$row` is the same object positioned on the current row.

To get all rows as an array (small result sets only):

```php
$rows = $sql->getArray();
```

## Counting / aggregates

```php
$sql = rex_sql::factory();
$sql->setQuery('SELECT COUNT(*) AS cnt FROM ' . rex::getTable('article') . ' WHERE status = ?', [1]);
$count = (int) $sql->getValue('cnt');
```

Positional placeholders (`?`) work too – pass values as a positional array.

## Inserting – fluent builder

```php
$sql = rex_sql::factory();
$sql->setTable(rex::getTable('my_addon_log'));
$sql->setValue('user_id', rex::getUser()->getId());
$sql->setValue('action', 'login');
$sql->setDateTimeValue('created_at', time()); // formats to MySQL datetime
$sql->insert();

$newId = (int) $sql->getLastId();
```

## Updating

```php
$sql = rex_sql::factory();
$sql->setTable(rex::getTable('my_addon_log'));
$sql->setWhere(['id' => $logId]);          // becomes "WHERE id = :id"
$sql->setValue('action', 'logout');
$sql->update();
```

For complex `WHERE` clauses, pass the SQL and parameters separately:

```php
$sql->setWhere('user_id = :uid AND created_at > :since', [
    'uid'   => $userId,
    'since' => '2026-01-01',
]);
```

## Deleting

```php
$sql = rex_sql::factory();
$sql->setTable(rex::getTable('my_addon_log'));
$sql->setWhere(['id' => $logId]);
$sql->delete();
```

## Transactions

```php
$sql = rex_sql::factory();
$sql->beginTransaction();
try {
    // multiple statements...
    $sql->commit();
} catch (Throwable $e) {
    $sql->rollBack();
    throw $e;
}
```

`rex_sql::factory()` returns a **new** `rex_sql` object on every call (own query, result and values), but all instances for the same DB share one PDO connection. A transaction therefore spans every `rex_sql` instance – no need to share an instance manually, and any second `beginTransaction()` call while the shared connection is already in a transaction (same instance or another) throws `rex_sql_exception`.

## Schema changes – `rex_sql_table`

For `install.php` / `update.php` in addons, use `rex_sql_table` instead of raw `CREATE TABLE`:

```php
rex_sql_table::get(rex::getTable('my_addon_items'))
    ->ensurePrimaryIdColumn()
    ->ensureColumn(new rex_sql_column('title', 'varchar(255)', false, ''))
    ->ensureColumn(new rex_sql_column('body', 'text', true))
    ->ensureColumn(new rex_sql_column('created_at', 'datetime', false))
    ->ensureIndex(new rex_sql_index('idx_created', ['created_at']))
    ->ensure();
```

`ensure()` is idempotent – safe to run on every install/update. It compares the desired schema with the actual one and only applies the diff.

## Common pitfalls

- Concatenating values into the SQL string – every placeholder must use parameter binding.
- Hardcoding `rex_` as the table prefix – use `rex::getTable()`.
- Using `mysqli_*` or raw `PDO` directly – breaks with non-default DB configs.
- Calling `getValue()` on a row object that hasn't been fetched (after exhausted iteration). Re-run the query or seek with `$sql->next()`.
- Forgetting that `getArray()` consumes the result set – call it once, then operate on the array.
