---
name: ydeploy-setup
description: Getting started with YDeploy in a REDAXO project – installing the ydeploy addon and Deployer 7, writing the project deploy.php (classic vs. Yak structure via deploy_yak.php), the recommended .gitignore, creating the initial schema.yml/fixtures.yml, the first `dep deploy` to a new host and what the `setup` task does there (config.yml, database copy, media copy, developer addon config, yrewrite domains), and `dep setup local` for onboarding a developer. Use when the user wants to start using ydeploy, set up deployment for a REDAXO project, add a new server/host, asks "how do I install ydeploy / deployer", or onboards a new local instance from a server.
---

# YDeploy Setup

[YDeploy](https://github.com/yakamara/ydeploy) (by Yakamara) has two parts:

1. **Database migrations** – console commands `ydeploy:diff` and `ydeploy:migrate` (see `ydeploy-diff-migrations`).
2. **A REDAXO-tailored [Deployer](https://deployer.org) configuration** – builds the release locally, uploads it, runs migrations on the server (see `ydeploy-deployer`).

YDeploy 2.x requires **Deployer 7** (the recipe throws if another major version is used), PHP >= 8.1 and REDAXO >= 5.13.

## 0. Optional: start from Yak

[Yak](https://github.com/yakamara/yak) is Yakamara's project skeleton for this setup (Yak directory structure, Webpack Encore, a `deploy.php` template). Starting a new project with it:

```bash
git remote add yak https://github.com/yakamara/yak.git
git fetch yak && git merge --allow-unrelated-histories yak/main   # later updates: git merge yak/main
php setup/presetup.php     # downloads REDAXO into public/ and creates the Yak structure – overwrites an existing instance
cp .env .env.local         # set APP_HOST=<project>.localhost and APP_ENV=dev
yarn
```

Then run the REDAXO setup in the browser (`https://<project>.localhost/redaxo`) with the web server's document root on `public/`, and install the `developer` and `ydeploy` addons.

## 1. Install the addon

Install ydeploy like any other addon (installer or `bin/console install:download ydeploy <version>` + `package:install ydeploy`). The install creates only one table: `rex_ydeploy_migration` (column `timestamp`, primary key).

## 2. Install Deployer 7

The addon does **not** ship Deployer. Pick one:

```bash
composer global require deployer/deployer:^7.0     # global
composer require --dev deployer/deployer:^7.0      # per project (optionally in a separate .tools/ folder)
# or the phar from https://deployer.org/download
```

## 3. Project `deploy.php`

In the project root:

```php
<?php

namespace Deployer;

if ('cli' !== PHP_SAPI) {
    throw new \Exception('The deployer configuration must be used in cli.');
}

// Classic structure (redaxo/ in the project root):
require __DIR__ . '/redaxo/src/addons/ydeploy/deploy.php';
// Yak structure (public/, src/, var/, bin/console) – use this instead:
// require __DIR__ . '/src/addons/ydeploy/deploy_yak.php';

set('repository', 'git@github.com:user/repo.git');

host('staging')
    ->setHostname('example.com')
    ->setRemoteUser('ssh-user')
    ->setDeployPath('/var/www/staging.example.com')
;

host('production')
    ->setHostname('example.com')
    ->setRemoteUser('ssh-user')
    ->setDeployPath('/var/www/www.example.com')
;
```

- Server-specific settings go on the host: `->set('bin/php', '/usr/bin/php8.3')` when the CLI default `php` is a different version (common on shared hosting), `->set('writable_mode', 'chmod')` when the default ACL mode is not available.
- `deploy_yak.php` just sets the Yak paths (`base_dir` = `public/`, `var/cache`, `var/data`, `src`, `bin/console`) and adds `var/log` to the shared and writable dirs.
- Anything set **after** the `require` overrides the ydeploy defaults (last `set()` wins; host-level settings beat global ones).
- Stage name: taken from `->setLabels(['stage' => 'production'])`, or – without a label – from the host alias if it is one of `staging`, `test`, `testing`, `live`, `prod`, `production`. It shows up as the backend badge and in `rex_ydeploy::factory()->getStage()`.
- `url` defaults to `https://<hostname>`. Set it explicitly (`->set('url', 'https://www.example.com')`) when the SSH hostname is not the website domain – the `setup` task uses it as default for the REDAXO `server` URL and YRewrite domains.
- The build runs `yarn install`/`npm install` + the build script automatically if a `package.json` exists (webpack or gulp are detected). Override with `set('assets_install', …)` / `set('assets_build', …)`.
- `build:vendors` is empty by default. If the project needs a Composer install during build, replace the task in `deploy.php`.

## 4. `.gitignore`

Recommended base from the ydeploy README (classic structure; adjust paths for Yak, e.g. `/var/data/addons/*/*`):

```
/.build
/media/*
!/media/.redaxo
/redaxo/cache/*
!/redaxo/cache/.*
/redaxo/data/addons/*/*
!/redaxo/data/addons/developer/*
!/redaxo/data/addons/mblock/*
!/redaxo/data/addons/mform/*
!/redaxo/data/addons/ydeploy/*
/redaxo/data/core/*
/redaxo/data/log/*
```

`redaxo/data/addons/ydeploy/` **must** be committed – it holds `schema.yml`, `fixtures.yml` and `migrations/`. `.build/` is the local build directory.

## 5. Initial schema

```bash
bin/console ydeploy:diff        # redaxo/bin/console in the classic structure
```

The first run only creates `schema.yml` and `fixtures.yml` (no migration file). Commit them. From now on every structural DB change goes through `ydeploy:diff` → commit → deploy.

## 6. First deploy to a new host

```bash
dep deploy staging
```

The release pipeline contains the `setup` task. On the target it checks whether `<data_dir>/core/config.yml` exists (the `core` data dir is a shared dir, so this is a per-host check). If it is missing, `setup` runs **interactively**:

1. **Choose the source**: another host from `deploy.php` (asked if there are several), or `local`.
2. **config.yml**: taken from the source, then asks for server URL, server name, error email and the target DB credentials (host, name, user, password). `setup` is set to `false`, `debug` to `false` on servers.
3. **Database copy**: dumps the source DB (`bin/console db:connection-options | xargs mysqldump`) and imports it into the target.
4. **Developer addon**: on servers, all developer settings except `templates`, `modules`, `actions`, `items` are set to `false` (no request-triggered sync on production).
5. **YRewrite domains**: asks for the new domain of every `rex_yrewrite_domain` row.
6. **Media**: copies the `media` folder from the source host.

After that the normal release continues (`database:migration`, publish). On later deploys `setup` is skipped.

Point the domain's document root to `<deploy_path>/current/public` (Yak) or `<deploy_path>/current` (classic) – `current` is a symlink to the active release.

The developer addon settings that `setup` writes match the recommendation in the Yak README: locally templates/modules/actions sync with frontend/backend sync on; on staging/production only templates/modules/actions stay on – there the deploy itself runs the sync (see `ydeploy-deployer`).

## 7. New local instance for a developer

```bash
dep setup local
```

Same interactive flow, targeting the local checkout: import from one of the hosts or from a dump file (default `./dump.sql`), creates the local database (`utf8mb4_unicode_ci`) if needed, writes `config.yml` with `debug: true`, and enables the developer addon's request-triggered sync (`sync_frontend`, `sync_backend`, `rename`, `dir_suffix`, `delete`). There is **no** way to build a database from `schema.yml`/`fixtures.yml` alone – a dump or a running host is always the starting point.

## Common pitfalls

- **Yak's `deploy.php` template uses the Deployer 6 host API** (`->hostname()`, `->user()`, `->set('deploy_path', …)`). With ydeploy 2.x/Deployer 7 use `->setHostname()`, `->setRemoteUser()`, `->setDeployPath()`.
- **Deployer 6 installed** → the recipe aborts with "YDeploy 2.x requires Deployer 7.x". Upgrade Deployer (see the addon's `UPGRADE.md` for the 1.x → 2.x changes in `deploy.php`: `setHostname()`, `setDeployPath()`, `setRemoteUser()`, `setLabels()`, `dep build local`, `dep setup local`).
- **`ydeploy/` data folder ignored by git** → migrations never reach the server; the deploy "works" but the DB stays old.
- **Local DB host not resolvable from where `dep` runs** – the local dump uses the host from the local `config.yml` (`db:connection-options`). In a Docker setup where `dep` runs outside the container, a service name like `db` does not resolve; `setup` fails at the database copy.
- **Dump import fails on zero dates** (`0000-00-00`, `0000-00-00 00:00:00`) under a strict `sql_mode` on the target server. Temporarily clear `sql_mode` for the import or fix the values in the source first.
- **Running `setup` against production with the wrong source** – it overwrites the target database. Double-check the source host before confirming.
- **Addon data folders outside the shared dirs** – see `ydeploy-deployer`: everything an addon writes to `data/addons/<addon>` at runtime is lost with the next release unless the folder is shared.
