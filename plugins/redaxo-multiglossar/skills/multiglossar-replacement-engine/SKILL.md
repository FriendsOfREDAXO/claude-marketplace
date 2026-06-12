---
name: multiglossar-replacement-engine
description: MultiGlossar replacement engine internals - boot.php OUTPUT_FILTER hook, the (internal) MultiGlossar\\Parser in lib/glossar/multiglossar.php, DOMDocument parsing, text_replace regex matching, replace_definition template placeholders, and URL namespace resolution via Url\\Profile/url_generator_profile. Stable integration is via OUTPUT_FILTER and the addon config — Parser is not a public API. Use when the user debugs missing replacements, broken tooltip HTML, URL addon integration, or replacement performance.
---

# MultiGlossar Replacement Engine

Frontend replacement is executed in `boot.php` via `rex_extension::register('OUTPUT_FILTER', ...)` and delegated to `MultiGlossar\\Parser` (`lib/glossar/multiglossar.php`).

## Runtime flow

1. `OUTPUT_FILTER` receives the rendered page HTML.
2. MultiGlossar checks exclusion rules (error article, excluded IDs/templates/meta condition).
3. Optional cache read happens before parsing.
4. Parser initializes DOM and glossary data for current `clang_id`.
5. Text nodes are traversed and terms are replaced with configured output HTML.
6. Header/footer and exclude markers are restored.

This architecture avoids raw global string replacement and limits changes to parsed text nodes.

## Parser and DOM behavior

> `MultiGlossar\Parser` is **internal**. Don't construct or call it from your own code — the stable integration points are the `OUTPUT_FILTER` hook (which the addon already registers) and the addon's config. The methods below are documented so you can read along when debugging, not as an API to invoke.

Key parser methods:

- `init_dom($source)` - detects configured start/end scope, loads glossary rows, initializes ignored tags/classes
- `parse_dom()` - traverses the DOM and rebuilds final output
- `parse_childs($node)` - recursive node walk with lock checks
- `text_replace($content)` - regex-based term matching and replacement token strategy

`<!--exclude-->...<!--endexclude-->` comment pairs are transformed into temporary elements and excluded from matching.

## Matching logic

Replacement behavior includes:

- aliases from `term_alt` are additional searchable markers
- optional case-sensitive matching (`casesensitive`)
- per-term replacement control: usually first match only, except for configured `article_complete` IDs where all matches are replaced
- word-boundary regex matching (`\\b...\\b`) to reduce accidental substring hits

## Output template placeholders

Replacement markup is driven by config `replace_definition`. Supported placeholders include:

- `---DEFINITION---`, `---URL---`, `---TERM---` (escaped variants)
- `---DEFINITION_RAW---`, `---URL_RAW---`, `---TERM_RAW---` (raw variants)

This enables custom tooltip/link markup without touching PHP code.

## URL addon integration

`resolveUrlKey()` resolves the query key/namespace for glossary detail URLs:

- default fallback: `gloss_id_<clangId>`
- if URL addon is available, namespace is resolved from `Url\\Profile` or `rex_url_generator_profile`

This is important when URL profiles rewrite parameter names.

## UIkit tooltip normalization

The parser normalizes certain UIkit tooltip patterns so configured output remains valid with dynamic definitions (for example converting direct `data-uk-tooltip` usage to a `uk-tooltip="title: ..."` format when needed).

## Common pitfalls

- Running replacements against raw HTML with custom code instead of using parser hooks, causing broken markup in attributes/tags.
- Using raw placeholders in output without escaping when values come from editor content.
- Forgetting URL namespace/profile handling, resulting in broken glossary detail links.
- Expecting replacements inside ignored elements (`a`, headings, `script`, `style`, configured classes/tags).
- Debugging only with one language while issues are actually `clang_id` specific.
