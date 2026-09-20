# redaxo-url

Claude Code plugin for the [URL addon](https://github.com/tbaddade/redaxo_url) by Thomas Blum.

The URL addon generates dataset-driven pretty URLs from any YForm table — e.g. turning a `rex_yf_products` row into `www.example.com/products/my-product/` without touching REDAXO's article structure.

## What this plugin covers

| Skill | Activates when… |
|---|---|
| `url-generator-profiles` | You configure a URL profile, map a YForm table to an article, or debug generated URLs |
| `url-generator-hooks` | You use `URL_PRE_SAVE` to shorten paths, `URL_PROFILE_RESTRICTION` to filter datasets |
| `url-generator-frontend` | You resolve the current dataset in a module via `Url::resolveCurrent()` |
| `url-generator-rebuild` | You (re)generate URLs from PHP — setup scripts, migrations, or a profile that stays empty |

## Install

```bash
/plugin install redaxo-url@redaxo-marketplace
```

Requires `yrewrite` and `url` addons to be installed in your REDAXO project.
