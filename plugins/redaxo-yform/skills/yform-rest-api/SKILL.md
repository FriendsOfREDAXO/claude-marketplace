---
name: yform-rest-api
description: YForm's built-in REST API – exposing YForm tables as JSON:API endpoints. Covers route registration via rex_yform_rest_route, GET filter/include/order/pagination params, JSON:API POST bodies, DELETE, token-based authentication, and CORS headers. Use when the user exposes a YForm table as a REST endpoint, configures token auth, customizes per-method field whitelists, builds a frontend SPA that calls a YForm-backed API, or says "REST-Endpoint", "JSON-API", "Token-Auth", "API-Aufrufe", "Endpunkt für YForm-Tabelle".
---

# YForm REST API

YForm includes a REST layer that exposes a YForm-backed dataset class as JSON:API endpoints. Authentication is token-based; routes are versioned by path; per-method field whitelists control what's readable / writable.

## Route registration

In your project addon's `boot.php`:

```php
$route = new \rex_yform_rest_route([
    'path' => '/v1/articles/',
    'auth' => '\rex_yform_rest_auth_token::checkToken',
    'type' => MyArticle::class,
    'query' => MyArticle::query(),
    'get' => [
        MyArticle::class => [
            'fields' => ['id', 'title', 'text', 'status'],
        ],
    ],
    'post' => [
        MyArticle::class => [
            'fields' => ['title', 'text', 'status'],
        ],
    ],
    'delete' => [
        MyArticle::class => [
            'fields' => ['id'],
        ],
    ],
]);
\rex_yform_rest::addRoute($route);
```

`MyArticle` must be a `rex_yform_manager_dataset` subclass registered with `setModelClass()`. The `query` value gives you a starting point — add `where()` clauses there to scope what the endpoint exposes.

## URL prefix

All routes live under `/rest/`. With the example above, the endpoint URL is:

```
GET    /rest/v1/articles/
POST   /rest/v1/articles/
DELETE /rest/v1/articles/?filter[id]=5
```

## GET parameters

| Parameter | Description | Example |
|---|---|---|
| `filter[field]=value` | Filter records | `?filter[status]=1` |
| `include=relation` | Include related dataset(s) | `?include=category` |
| `per_page=N` | Items per page | `?per_page=10` |
| `page=N` | Page number | `?page=2` |
| `order[field]=dir` | Sort | `?order[created]=desc` |

Multiple filters are AND-combined: `?filter[status]=1&filter[clang_id]=1`.

## POST body (JSON:API format)

```json
{
    "data": {
        "type": "MyArticle",
        "attributes": {
            "title": "Neuer Artikel",
            "text": "Inhalt...",
            "status": 1
        }
    }
}
```

The `type` value must match the class name registered in the route (`MyArticle::class`). Only fields listed in `post.fields` are accepted; everything else is ignored.

## DELETE

```
DELETE /rest/v1/articles/?filter[id]=5
```

Deletes match the same `filter[...]` semantics as GET. Be careful with broad filters — the API will happily delete every matching record.

## Authentication

Default is token-based via `\rex_yform_rest_auth_token::checkToken`. Manage tokens in the backend under **YForm → REST → Tokens**. Token scopes can be limited to specific routes.

Header:

```
Authorization: Bearer <token>
```

For public read-only endpoints, set `'auth' => null` — but think hard before doing this; an open POST is an open data-injection target.

## CORS headers

Global (applies to every YForm REST route):

```php
\rex_yform_rest::setHeader('Access-Control-Allow-Origin', '*');
```

Per-route:

```php
$route->setHeader('Access-Control-Allow-Origin', 'https://example.com');
```

For browser-based SPAs, add:

```php
$route->setHeader('Access-Control-Allow-Methods', 'GET, POST, DELETE, OPTIONS');
$route->setHeader('Access-Control-Allow-Headers', 'Authorization, Content-Type');
```

## Per-method field whitelists

`get.fields`, `post.fields`, `delete.fields` are independent. Practical pattern:

- `get.fields` – everything safe to expose (excluding internal admin notes, soft-delete flags)
- `post.fields` – the user-editable subset (excluding `id`, `created_at`, `status` if those are server-controlled)
- `delete.fields` – usually just `['id']` for filter purposes

## Apache: pass through `Authorization` header

Apache strips `Authorization` by default in some configurations. If valid tokens get rejected with `Authorization failed`, add to `.htaccess`:

```apache
RewriteCond %{HTTP:Authorization} .
RewriteRule ^ - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

## Common pitfalls

- Registering the route in an addon's `boot.php` that fires too late – register in `boot.php` directly (top level), not inside a callback that runs only on backend requests.
- Forgetting to register the model class with `setModelClass()` – the route serializes plain `rex_yform_manager_dataset` instead of your subclass, and `type` mismatches break POST.
- Setting `'auth' => null` for "just a quick test" and forgetting to restore it before deploying.
- Listing fields in `post.fields` that don't exist on the table – the field gets silently dropped from incoming POSTs.
- Forgetting CORS headers when the endpoint is consumed from a different origin – preflight `OPTIONS` requests fail with no clear error.
- Letting `query` start un-scoped – an open `MyArticle::query()` exposes every row in the table, including drafts and soft-deleted records. Always add baseline `where()` filters.
