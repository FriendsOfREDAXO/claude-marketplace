# redaxo-ydeploy

Claude Code support for REDAXO's [YDeploy](https://github.com/yakamara/ydeploy) addon. YDeploy provides database migrations (`ydeploy:diff`, `ydeploy:migrate`) and a REDAXO-tailored [Deployer](https://deployer.org) 7 configuration.

## Skills

- **ydeploy-setup** – installing ydeploy and Deployer, `deploy.php` (classic and [Yak](https://github.com/yakamara/yak) structure), `.gitignore`, first deploy and the interactive `setup` task, `dep setup local`
- **ydeploy-diff-migrations** – how `ydeploy:diff` and `ydeploy:migrate` work, writing and reviewing migrations, `rex_ydeploy_migration`
- **ydeploy-fixtures** – what travels with a deploy (fixture tables, `rex_config` namespaces, installed addons) and what does not (`install.php`, addon config, content), adding own fixture tables, protected backend pages
- **ydeploy-deployer** – `dep deploy` / `build` / `release`, deployed branch, task order, shared dirs, server cache, `rex_ydeploy` API

## Install

```bash
/plugin install redaxo-ydeploy@redaxo-marketplace
```

Written against ydeploy 2.1 (Deployer 7).
