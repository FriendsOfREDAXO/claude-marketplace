---
name: ydeploy-deployer
description: Deploying REDAXO with YDeploy's Deployer 7 recipe – `dep deploy`, `dep build local`, `dep release <host>`, which branch/commit gets deployed, the task order (build, upload, shared dirs, setup, database:migration, publish, server:clear_cache), shared_dirs/writable_dirs/copy_dirs/clear_paths, addon data folders that get lost between releases, `deploy:unlock`, rollback, server cache options (clear_web_php_cache, restart_apache, kill_process), and the rex_ydeploy API (isDeployed, getStage, badge). Use when the user runs or configures `dep` / deploy.php, a deploy fails or hangs, asks what exactly gets deployed, adds a shared directory, or needs to distinguish live/staging/local in code.
---

# YDeploy Deployer

YDeploy's `deploy.php` recipe (Deployer 7) splits a deploy into a **local build** and a **remote release**.

```bash
dep deploy <host>         # = build on host "local" + release to <host>
dep build local           # only build into .build/release
dep release <host>        # upload the existing build to <host>
dep deploy:unlock <host>  # after an aborted deploy left the lock
dep rollback <host>       # Deployer: switch "current" back to the previous release
```

Build once, release twice: `dep deploy staging`, test, then `dep release production` ships **exactly** the same build.

## What gets deployed

- `branch` is set by ydeploy to the **currently checked-out local branch** (`git rev-parse --abbrev-ref HEAD`), not the remote's default branch.
- The build does **not** upload your working tree. `build:setup` runs Deployer's `deploy:update_code` into `.build/release`, i.e. it fetches that branch **from the remote repository**. Uncommitted or unpushed changes are not deployed.
- The first line of the build (`build:info`) prints `building <target>` – check it.
- Deploy a different branch without switching: `dep deploy staging --branch=main`. To pin production to one branch, set it on the host in `deploy.php`: `->set('branch', 'main')`.
- `CI` environment variable set → no local `.build` dir; the CI checkout is used as-is.

## Task order

**`build`** (on `local`): `build:info` → `build:setup` (fresh checkout) → `build:vendors` (empty, override for Composer) → `build:assets` (yarn/npm install + build if `package.json` exists; `node_modules` is cached in `.build/.node_modules`) → `deploy:clear_paths`.

**`release`** (on the target):

```
deploy:info → deploy:setup → deploy:lock → deploy:release → deploy:copy_dirs
→ deploy:upload      rsync of .build/release (--delete; excludes .git, node_modules, deploy.php, .tools, .cache)
→ deploy:shared      symlinks shared_dirs into the release
→ deploy:dump_info   writes data/addons/ydeploy/info.json (host, stage, branch, commit, timestamp)
→ deploy:writable
→ setup              only if the host has no config.yml yet (see ydeploy-setup)
→ database:migration ydeploy:migrate, then developer:sync --force-files (if the developer addon exists)
→ deploy:publish     switches the "current" symlink, then server:clear_cache, cleanup (keep_releases = 5)
```

On failure Deployer runs `deploy:failed`, which ydeploy hooks to `deploy:unlock`.

**Migrations run before the symlink switch.** If a migration fails, the new release never goes live, but the migrations that ran before it are applied to the live database while the old code is still serving. Fix forward: correct the migration locally, commit, deploy again (the already-executed migrations are skipped).

## Directory settings

Defaults (`deploy.php`; paths relative to `base_dir`, Yak adds `var/log`):

| Setting | Default |
|---|---|
| `shared_dirs` | `media`, `data/addons/cronjob`, `data/addons/phpmailer`, `data/addons/yform`, `data/core` |
| `writable_dirs` | `assets`, `media`, cache dir, data dir |
| `copy_dirs` | `assets`, `src` (copied from the previous release before upload) |
| `clear_paths` | `.github`, `.idea`, `gulpfile.js`, `.gitignore`, `.gitlab-ci.yml`, `.php-cs-fixer.dist.php`, `package.json`, `README.md`, `webpack.config.js`, `yarn.lock`, `REVISION` |

Extend with `add()` in the project `deploy.php` – `set()` replaces the whole list:

```php
add('shared_dirs', ['{{data_dir}}/addons/my_addon']);
```

Every deploy creates a new release directory. **Anything an addon writes at runtime into `data/addons/<addon>` that is not in `shared_dirs` is gone after the next deploy** (the old release still has it until it is rotated out after `keep_releases`). Typical candidates: downloaded/generated files, caches that cannot be rebuilt, uploads stored outside the media pool. Add such folders to `shared_dirs`. If files are already lost, look in the older releases under `<deploy_path>/releases/` before they are cleaned up.

Conversely, data an addon needs that comes from the repository (e.g. theme files copied at install) must be committed – see the `.gitignore` in `ydeploy-setup`.

## Server cache

`server:clear_cache` runs after publish (`before('deploy:cleanup')`). It always clears the CLI stat and opcache. Optional per host:

```php
set('clear_web_php_cache', true);  // writes a temp PHP file into the web root and calls it via curl {{url}}
set('restart_apache', true);       // apachectl configtest && graceful
set('kill_process', 'php-fpm');    // pkill -u <user> <name>
```

Use `clear_web_php_cache` when the web opcache keeps paths of the previous release after the symlink switch. It needs `url` to point to the public website of that host.

## Using deployment info in code

```php
$ydeploy = rex_ydeploy::factory();

$ydeploy->isDeployed();   // false on local instances (no info.json)
$ydeploy->getStage();     // e.g. 'production', 'staging' or null
$ydeploy->getHost();      // host alias from deploy.php
$ydeploy->getBranch();
$ydeploy->getCommit();
$ydeploy->getTimestamp(); // DateTimeImmutable of the last deploy

if ('production' === rex_ydeploy::factory()->getStage()) {
    // e.g. load tracking only on production
}
```

The backend shows a badge (`Development` locally, host/stage on servers) and adds body classes `ydeploy-is-deployed` / `ydeploy-stage-<stage>` / `ydeploy-is-not-deployed`. The badge text can be changed or removed (return `''`) via the extension point `YDEPLOY_BADGE`. The page *System → YDeploy* shows the deployment info.

## Common pitfalls

- **Wrong branch deployed** – you were on a feature branch. Watch `building <target>`, use `--branch=`, or pin the branch per host.
- **"My fix isn't live"** – it wasn't pushed. The build fetches from the remote.
- **Deploy aborted, next deploy says "Deploy locked"** → `dep deploy:unlock <host>`.
- **Addon data disappears after each deploy** → the folder is not in `shared_dirs`.
- **Editing files on the server** – the next release replaces the whole release directory; only shared dirs survive.
- **`dep release` after new commits** – it ships the old build. Run `dep deploy` (or `dep build local`) again.
