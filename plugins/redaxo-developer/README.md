# redaxo-developer

Claude Code support for [developer](https://github.com/FriendsOfREDAXO/developer), the REDAXO addon that exposes templates, modules, actions and YForm email templates as editable files.

The addon makes REDAXO code version-controllable, but it also introduces the one assumption that does not hold in a REDAXO project: saving a file does not apply the change. REDAXO renders the database; the files are a mirror, and a timestamp decides which side wins. This plugin exists so that an agent does not rediscover that from the addon source every time — or reach for `--force-files` when a `touch` was needed.

## Skills

- **developer-sync** - why an edited file has no effect, the four gates that keep the sync from running, timestamp arbitration between file and database, the `git pull` trap, and why `--force-db` / `--force-files` are not fallbacks
- **developer-items** - creating templates, modules and actions from the file system, `.rex_id` as the item identity, `.rex_ignore`, name transliteration, and the rename/delete semantics

## Install

```bash
/plugin install redaxo-developer@redaxo-marketplace
```

Use this plugin if your project has the `developer` addon installed and template, module or action code is edited as files — in an editor, through an agent, or via version control.
