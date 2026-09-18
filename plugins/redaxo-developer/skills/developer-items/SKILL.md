---
name: developer-items
description: Creating, naming, renaming and deleting templates, modules, actions and YForm email templates through the developer addon's file system. Use when a new module or template is to be created as files instead of in the backend, when a directory under the developer base path is renamed, copied or deleted, when `.rex_id` or `.rex_ignore` come up, when an item directory is silently ignored, or on the error "There are two item directories with the same ID".
---

# developer addon: items in the file system

## Directory layout

One directory per type under the developer base path (default `redaxo/data/addons/developer/`, but addons may relocate it — see `developer-sync`), and one directory per item beneath that.

| Type | Directory | Expected files | Table |
|---|---|---|---|
| Templates | `templates/` | `template.php` | `rex::getTable('template')` |
| Modules | `modules/` | `input.php`, `output.php` | `rex::getTable('module')` |
| Actions | `actions/` | `preview.php`, `presave.php`, `postsave.php` | `rex::getTable('action')` |
| YForm emails | `yform_email/` | `body.php`, `body_html.php` | `rex::getTable('yform_email_template')` |

Each item directory additionally holds `<id>.rex_id` and a `metadata.yml`. The metadata carries the item name plus type-specific columns (`key` for templates and modules, the `*mode` flags for actions, sender and subject for YForm emails).

## Identity is the `.rex_id` file, not the directory name

A directory containing `<id>.rex_id` **is** that item. A directory without one is treated as **new** and inserted into the database on the next sync. A directory containing `.rex_ignore` is skipped entirely.

That single rule explains most surprises:

- **Renaming an item directory is safe.** The id travels in the file; the addon simply picks the new name up.
- **Copying an item directory is not.** The copied `.rex_id` yields two directories claiming the same id, and the sync aborts with `E_USER_ERROR`: `There are two item directories with the same ID`. Remove the copy, or delete its `.rex_id` to turn it into a new item.
- **A directory that is ignored no matter what edits it receives** almost always contains a `.rex_ignore` — the remains of an item deleted in the backend while the `delete` setting was off.

## Creating an item from files

Create a directory with at least one of the expected files. Nothing else.

- **Do not create `.rex_id`.** The sync assigns the id and writes the file itself. A hand-picked id collides with, or overwrites, somebody else's item.
- **Do not put an id in the directory name.** With `dir_suffix` enabled (the default) the addon appends ` [<id>]` on its own. Square brackets in the name are rewritten to round ones beforehand, so `Teaser [43]` ends up as `Teaser (43) [43]`.
- **Do not write `metadata.yml` by hand.** The sync generates it. If one exists, its `name` overrides the directory name, so the two must match or the item is called something else in the backend than its directory suggests.
- **Run the sync twice.** The first pass inserts the record and writes `<id>.rex_id`; the directory is only renamed to its final `… [<id>]` form on the second pass.
- Afterwards `cache:clear` — for a new item the addon clears nothing by itself.

## What the addon does to a name

The directory name becomes the item name, and the item name becomes the directory name. Both directions transform:

| In the name | Becomes |
|---|---|
| `_` underscore | space (directory → item name) |
| `[` `]` `/` | `(` `)` `-` |
| `\ \| : < > ? * " ' +` | removed |
| Umlauts, `ß` | `ae` / `oe` / `ue` / `ss`, unless the `umlauts` setting is on |
| Trailing dot or space | trimmed |

So a name chosen in the backend and the directory carrying it will not always match character for character. That is expected, not corruption.

## Renaming and deleting

- **Renaming in the backend does not rename the directory.** Only `metadata.yml` is updated. The reverse works too: editing `name` in `metadata.yml` renames the item in the backend. (With the `rename` setting on, the addon does realign directory and file names on the next run.)
- **Deleting a directory or a file achieves nothing.** The sync recreates it from the database. Items are deleted in the backend.
- **After deleting an item in the backend**, the directory is removed if the `delete` setting is on; otherwise `.rex_id` is replaced by `.rex_ignore` and the directory stays behind, inert.

## Settings that change the file layout

In `rex_config`, namespace `developer`:

| Setting | Default | Effect |
|---|---|---|
| `templates`, `modules`, `actions`, `yform_email` | on | Which types are synchronized at all |
| `dir_suffix` | on | Appends ` [<id>]` to item directories |
| `prefix` | off | Expects `<id>.<name>.output.php` instead of `output.php` |
| `rename` | on | Realigns directory and file names with the item name |
| `umlauts` | off | Keeps umlauts in names instead of transliterating |
| `delete` | on | Removes the directory when the item is deleted in the backend |

Custom file names are allowed as long as they **end** with the expected name — `navigation.template.php` is found and kept. With `rename` on, the addon renames it back to the canonical form.

## Common pitfalls

| Assumption | Reality |
|---|---|
| "I will copy an existing module directory as a starting point." | The copied `.rex_id` breaks the sync with a duplicate-id fatal error. Copy the files, not the directory. |
| "I will create the `.rex_id` so the ids stay tidy." | The sync assigns ids. An invented one collides with an existing item. |
| "I will delete the directory to remove the module." | It comes back on the next sync. Delete the item in the backend. |
| "The directory name is wrong after syncing." | The suffix arrives on the second pass, and characters are transliterated. Both are intended. |
| "My edits to this directory never do anything." | Look for `.rex_ignore` in it. |
| "The name in the backend differs from my directory." | A hand-written `metadata.yml` overrides the directory name. |

Why an edited file has no effect at all, and how the sync decides between file and database, is a separate topic: see the `developer-sync` skill.
