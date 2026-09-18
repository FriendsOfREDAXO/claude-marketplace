# REDAXO Claude Code Marketplace

A collection of [Claude Code](https://claude.com/claude-code) plugins that turn Claude into a knowledgeable assistant for [REDAXO](https://www.redaxo.org) – a flexible PHP-based content management system.

The marketplace is modular: install only the plugins for the addons you actually use in your project. Claude then loads the matching skills automatically when relevant.

## Quick Start

In Claude Code, add this marketplace once:

```bash
/plugin marketplace add FriendsOfREDAXO/claude-marketplace
```

Then install the plugins you need. The **core plugin is always recommended**:

```bash
/plugin install redaxo-core@redaxo-marketplace
```

Add addon-specific plugins based on what your project uses:

```bash
/plugin install redaxo-yform@redaxo-marketplace
/plugin install redaxo-yrewrite@redaxo-marketplace
/plugin install redaxo-url@redaxo-marketplace
/plugin install redaxo-structure@redaxo-marketplace
/plugin install redaxo-mform@redaxo-marketplace
/plugin install redaxo-multiglossar@redaxo-marketplace
/plugin install redaxo-search-it@redaxo-marketplace
```

## Available Plugins

| Plugin                | What it covers                                                                                                                                          | When to install                           |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `redaxo-core`         | Architecture, modules, templates, `rex_sql`, extension points, addon development, console commands, `rex_api_function`, metainfo fields                | Every REDAXO project                      |
| `redaxo-structure`    | Articles, categories, content editing, meta info                                                                                                        | Almost always (Structure is part of core) |
| `redaxo-yform`        | YForm tables, datasets (YOrm), field/validate/action reference, frontend forms, email templates, REST API                                               | If `yform` is installed                   |
| `redaxo-mform`        | Module input forms with `MForm::factory()`, flex repeaters, custom-link/imagelist/colorswatch widgets, output helpers, MForm-provided YForm value types | If `mform` is installed                   |
| `redaxo-yrewrite`     | Domains, pretty URLs, redirects, multi-language SEO                                                                                                     | If `yrewrite` is installed                |
| `redaxo-url`          | URL addon: dataset-driven pretty URLs, profiles, `URL_PRE_SAVE` hooks, frontend resolution, programmatic rebuilds                                        | If `url` is installed                     |
| `redaxo-ycom`         | Frontend user auth, login/registration/password forms, groups, media protection, OTP/2FA, tokens, SAML/OAuth2/CAS                                       | If `ycom` is installed                    |
| `redaxo-api-addon`    | FriendsOfRedaxo/api – Bearer-token REST API for articles/categories/slices/modules/templates/media                                                      | If you call (or extend) the `api` addon   |
| `redaxo-search-it`    | Full-text search, indexing, search modules, highlighting, autocomplete, similarity search, extension points                                             | If `search_it` is installed               |
| `redaxo-multiglossar` | MultiGlossar term management, multilingual glossary content, DOM-based frontend replacement, tooltip/link output, exclusion rules                       | If `multiglossar` is installed            |

## How it works

Each plugin contains [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) – focused instruction packages that Claude loads on demand based on what you're working on. Edit a module file, and `redaxo-modules` activates. Ask about URL routing, and `yrewrite-domains` kicks in. Skills are namespaced per plugin, so `redaxo-core` skills won't collide with anything else.

## Updating

When the marketplace is updated, refresh your local copy:

```bash
/plugin marketplace update redaxo-marketplace
```

## Contributing

Found a missing pattern, an outdated example, or a bug? PRs welcome. See [`CLAUDE.md`](./CLAUDE.md) for plugin authoring conventions, the skill activation rule, and the local testing workflow.

## License

MIT
