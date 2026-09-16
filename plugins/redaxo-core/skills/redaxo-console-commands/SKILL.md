---
name: redaxo-console-commands
description: Building CLI / diagnostic / maintenance commands for REDAXO addons using Symfony Console (rex_console_command). Covers the class location, naming convention, package.yml registration, and SymfonyStyle helpers. Use when the user adds a CLI tool to an addon, wants a cronjob entry point, builds a diagnostic command, or asks "how do I run X from the command line in REDAXO". Replaces the anti-pattern of writing standalone bootstrap scripts in bin/.
---

# REDAXO Console Commands

For any CLI / diagnostic / maintenance task in an addon, build a **console command** — not a standalone PHP bootstrap script. REDAXO uses Symfony Console under the hood; the bootstrap (autoloading, addon init, package order) only works correctly through `bin/console`.

## Anti-pattern (don't do this)

```php
// bin/_my_check.php — WRONG
require __DIR__ . '/../src/core/boot.php';   // missing path provider
$client = rex_elasticsearch_client::factory();  // wrong API
```

Standalone scripts like this either fail to boot, or end up using APIs that don't exist. Even if you get them working, they're invisible in `bin/console list`, can't run as cronjobs, and aren't reviewable as part of the addon.

## Structure — required vs. conventional

1. **File path:** any PHP file under `src/addons/<addon>/lib/` (REDAXO's autoloader scans the whole folder); `lib/command/<name>.php` is a common convention
2. **Class name:** free choice, namespaced classes work too — the loader instantiates exactly the class given in `package.yml`. A common convention is `rex_<addon>_command_<name>` with `:` becoming `_` (e.g. command `ndcg:detail` → class `rex_elasticsearchtools_command_ndcg_detail`); the cronjob addon itself uses `rex_command_cronjob_run`
3. **Extends:** `rex_console_command` (REDAXO's wrapper around `Symfony\Component\Console\Command\Command`)
4. **Registration in `package.yml`** under `console_commands:` — without this REDAXO won't expose the command. Autoloading alone is not enough.

## Skeleton

```php
<?php

use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Input\InputOption;
use Symfony\Component\Console\Output\OutputInterface;

/**
 * Usage:
 *   bin/console addon:topic:action --option=value
 */
final class rex_addon_command_topic_action extends rex_console_command
{
    protected function configure(): void
    {
        $this
            ->setDescription('One-line description shown in `bin/console list`')
            ->addOption('alias', 'a', InputOption::VALUE_REQUIRED, 'Index alias', 'default_alias')
            ->addOption('query', null, InputOption::VALUE_REQUIRED, 'Search query', 'test');
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $io = $this->getStyle($input, $output);  // SymfonyStyle helper
        $io->title('My Command');

        $alias = (string) $input->getOption('alias');

        // SQL access works as usual
        $sql = rex_sql::factory();
        $rows = $sql->getArray(
            'SELECT COUNT(*) AS c FROM ' . rex::getTable('table') . ' WHERE state = 7'
        );

        $io->table(['Header'], [['row1'], ['row2']]);
        $io->success('Done');

        return 0;  // 0 = success, non-zero = failure
    }
}
```

## Registration in `package.yml`

```yaml
console_commands:
  addon:topic:action: rex_addon_command_topic_action
  addon:topic:other:  rex_addon_command_topic_other
```

The command name uses `:` as namespace separator. By convention, the class name uses `_` instead.

## SymfonyStyle helper (`$io`)

`$this->getStyle($input, $output)` returns a `SymfonyStyle` instance:

| Method | Use for |
|---|---|
| `$io->title($t)` | Big heading |
| `$io->section($t)` | Subheading |
| `$io->writeln($msg)` | Plain output |
| `$io->note($msg)` | Informational box |
| `$io->success($msg)` | Green success box |
| `$io->error($msg)` | Red error box |
| `$io->warning($msg)` | Yellow warning box |
| `$io->table($headers, $rows)` | Formatted table |
| `$io->progressStart($total)` / `progressAdvance()` / `progressFinish()` | Progress bar |
| `$io->ask($q)` / `confirm($q)` / `choice($q, $opts)` | Interactive prompts |

## Verifying registration

```bash
bin/console list | grep <addon>
```

If the command isn't there, the `package.yml` entry is missing or the addon isn't installed and active.

## Common patterns

### Reading from a YForm table

```php
$items = MyTable::query()->where('status', 1)->find();
foreach ($items as $item) {
    $io->writeln($item->getValue('name'));
}
```

### Running as a cronjob via the cronjob addon

In `package.yml` of the cronjob addon's host configuration (or via the cronjob backend UI), schedule a system command:

```
bin/console addon:topic:action --alias=production
```

## Common pitfalls

- Forgetting the `console_commands:` entry in `package.yml` – the file is autoloaded but never registered, so `bin/console list` doesn't show it.
- Class name in `package.yml` doesn't match the class declared in `lib/` (typo, missing namespace) – `rex_console_command_loader::get()` fails on `new $class()` as soon as the command is loaded; `bin/console list` loads all commands, so it breaks too.
- Returning a non-int from `execute()` – Symfony Console treats anything other than `0` (or the `Command::SUCCESS`/`FAILURE` constants on newer versions) as a failure.
- Writing directly to STDOUT with `echo` instead of `$output->writeln()` / `$io->writeln()` – Symfony's `--quiet` flag has no effect.
- Building the command but skipping the `setDescription()` – `bin/console list` shows no help text, future-you won't remember what it does.
- Calling `rex::getUser()` – there's no logged-in user in CLI context. It returns `null`. For permission checks, gate on `--user` flags or environment.
