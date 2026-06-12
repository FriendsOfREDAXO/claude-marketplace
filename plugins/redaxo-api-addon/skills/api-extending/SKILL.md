---
name: api-extending
description: Adding custom REST routes to the FriendsOfRedaxo/api addon (v1.2+) from your own addon — building a RoutePackage subclass, registering routes, scope management, the `Body`/`query` schema keys, request-context caveats (auth runs in frontend context, isBackend() returns false, PRE-EPs that call rex::requireUser() fail under Bearer), and the addon's exact-mirror-of-core convention. Use when the user adds endpoints to the api addon, exposes their own data over the same auth/scope mechanism, runs into "EP fires only in backend but I need it from the API" / "rex::requireUser threw" problems, or says "eigene Route registrieren", "API um eigenen Endpunkt erweitern", "RoutePackage bauen".
---

# Extending the `api` Addon (v1.2+)

You can add your own routes to the `api` addon from any custom addon. They reuse the same Bearer token, the same scope mechanism, the same OpenAPI generator, and the same `JsonResponse`-based response convention as the built-in routes.

## RoutePackage skeleton

```php
<?php
namespace MyVendor\MyAddon\RoutePackage;

use FriendsOfRedaxo\Api\Auth\BearerAuth;
use FriendsOfRedaxo\Api\RouteCollection;
use FriendsOfRedaxo\Api\RoutePackage;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Route;

class MyResource extends RoutePackage
{
    public function loadRoutes(): void
    {
        RouteCollection::registerRoute(
            'myaddon/things/list',                          // becomes the token scope name
            new Route(
                'myaddon/things',                            // → /api/myaddon/things
                [
                    '_controller' => self::class . '::handleList',
                    'query' => [                             // GET parameter schema
                        'page'     => ['type' => 'integer', 'default' => 1],
                        'per_page' => ['type' => 'integer', 'default' => 20],
                        'sort'     => ['type' => 'string'],
                        'filter'   => ['type' => 'object', 'properties' => [
                            'name' => ['type' => 'string'],
                        ]],
                    ],
                ],
                [], [], '', [], ['GET'],
            ),
            'List things',                                   // OpenAPI summary
            null,                                            // custom response definitions, or null
            new BearerAuth(),
        );

        RouteCollection::registerRoute(
            'myaddon/things/add',
            new Route(
                'myaddon/things',
                [
                    '_controller' => self::class . '::handleAdd',
                    'Body' => [                              // POST/PUT body schema (note the capital B)
                        'name'  => ['type' => 'string', 'required' => true],
                        'color' => ['type' => 'string', 'default' => 'blue'],
                    ],
                ],
                [], [], '', [], ['POST'],
            ),
            'Create a thing',
            null,
            new BearerAuth(),
        );
    }

    public static function handleList($Parameter): Response
    {
        $Query = RouteCollection::getQuerySet($_REQUEST, $Parameter['query']);
        // ... build response data ...
        return new JsonResponse(['data' => [/* ... */], 'meta' => [/* ... */]]);
    }

    public static function handleAdd($Parameter): Response
    {
        $Body = json_decode(\rex::getRequest()->getContent(), true);
        try {
            $Data = RouteCollection::getQuerySet($Body, $Parameter['Body']);
        } catch (\Throwable $e) {
            return new JsonResponse(['error' => 'Body field: `' . $e->getMessage() . '` is required'], 400);
        }
        // ... persist ...
        return new JsonResponse(['data' => [/* created */]], 201);
    }
}
```

Both `query` and `Body` are validated/normalized by `RouteCollection::getQuerySet()` against the schema you declared on the route — don't re-validate manually.

## Register in the addon's `boot.php`

```php
\FriendsOfRedaxo\Api\RouteCollection::registerRoutePackage(
    new \MyVendor\MyAddon\RoutePackage\MyResource()
);
```

After registration, the new scope (`myaddon/things/list`) appears in the backend token editor. Add it to the relevant token, otherwise calls return `Authorization failed`.

## Trailing-slash convention

Stay consistent: pick either trailing-slash or no-trailing-slash per resource and keep all methods on that resource using the same convention. The built-in routes converged on **no trailing slash** in v1.2 — recommend the same in your own RoutePackages:

```php
'myaddon/things'         // GET   → list
'myaddon/things'         // POST  → create
'myaddon/things/{id}'    // GET   → show
'myaddon/things/{id}'    // PATCH → update
'myaddon/things/{id}'    // DELETE → remove
```

## Authentication options

| Class | Use for |
|---|---|
| `BearerAuth` | Standard token-based API |
| `BackendUser` | Routes that should only be callable when logged into the backend (cookie session) |
| `null` | Unauthenticated public endpoint — only do this for actually public data |

Prefer `BearerAuth` for external-facing endpoints. For internal backend tools, mirror your Bearer routes under `backend/<scope>` with `BackendUser` auth — that's the convention used by the built-in `Backend/*` packages, see `lib/RoutePackage/Backend/Clangs.php` for a tiny reference implementation that clones every Bearer route from a parent package and re-registers it with `BackendUser`.

## Request context caveats

`RouteCollection::handle()` runs in the **frontend user context** (specifically: it dispatches inside `YREWRITE_PREPARE`, before any backend bootstrapping). Three consequences:

1. **`rex::isBackend()` returns `false`** in API calls — even on `BackendUser` routes. Code paths gated on `rex::isBackend()` silently skip.
2. **Extension points registered only in `if (rex::isBackend()) { ... }` branches don't fire.** Either register them unconditionally and guard inside the callback, or trigger them explicitly inside the API route.
3. **`rex::getUser()` is `null` on Bearer routes.** Some REDAXO PRE-extension-points (notably `SLICE_UPDATE`, `SLICE_DELETE`, parts of structure-management) call `rex::requireUser()` internally. Calling those service methods from an API route that authenticated via Bearer will throw. Two options:
   - Guard the EP yourself: only call the service method or fire the EP when `rex::getUser() !== null` (i.e. only on `BackendUser` routes).
   - Bypass the service and replicate the relevant SQL + the POST-EP yourself, the way the built-in `Structure::handleUpdateArticleSlice` / `handleDeleteArticleSlice` do.

When porting backend logic to an API endpoint, audit every `rex::isBackend()`, `rex::getUser()`, and `rex::requireUser()` check in the call path.

## Mirror REDAXO-core behaviour exactly

The built-in routes follow a hard rule (see the addon's `CLAUDE.md`): the API is not a parallel system, it's an HTTP frontend for the existing backend pages. If a backend page (`pages/*.php`) calls `rex_article_service::editArticle()` and then fires `STRUCTURE_CONTENT_ARTICLE_UPDATED` and clears `rex_article_cache::delete($id)`, the API route does the same — same calls, same EP names, same parameter keys, same order, same cache invalidation. Don't:

- Invent EPs that core doesn't fire (e.g. don't fire a custom `MY_THING_DELETED` if the equivalent backend page is silent)
- Add fields to the EP payload that core doesn't set
- Skip cache invalidation that the backend page does

This rule applies equally to your custom RoutePackages if you're wrapping core resources.

## Returning the right response shape

Use `Symfony\Component\HttpFoundation\JsonResponse` (the built-in routes converged on this in #18). It sets `Content-Type: application/json` automatically and serialises arrays for you:

```php
return new JsonResponse(['data' => $items, 'meta' => $meta], 200);
return new JsonResponse(['data' => $created], 201);
return new JsonResponse(null, 204);                       // empty body
return new JsonResponse(['error' => 'Bad request'], 400);
```

For consistency with the built-in routes:

- **200** for successful reads
- **201** with the created resource for successful creates
- **204** (empty) for successful deletes
- **4xx** with `{"error": "..."}` for client errors (extra context fields are fine)
- **5xx** with `{"error": "..."}` only after logging the underlying exception (`RouteCollection` already wraps unhandled throwables with `rex_logger::logException`)

For paginated lists, use `ListHelper::paginate()` / `ListHelper::wrapResponse()` to get the standard `{data, meta}` envelope used by every built-in list endpoint.

## OpenAPI integration

The addon generates an OpenAPI spec at `?page=api/openapi` from registered routes. The third and fourth arguments to `registerRoute()` (description + custom responses) feed that generator. Spend a minute writing a meaningful description — Swagger UI uses it.

The `query` and `Body` schemas you declare in `_controller` are mirrored into the spec automatically — that's why they live in the route definition and not just inside the handler.

## Common pitfalls

- Forgetting to register the RoutePackage in `boot.php` — the route never appears, calls return 404 `Route not found`.
- Forgetting to add the new scope to the token in the backend — calls return 401 `Authorization failed`.
- Using `new Response(json_encode(...))` and forgetting `Content-Type: application/json` — use `JsonResponse` instead, that's the addon convention since #18.
- Reading bodies via `$_POST` — JSON bodies aren't in `$_POST`. Use `json_decode(rex::getRequest()->getContent(), true)`.
- Using closures in `_controller` instead of `Class::method` references — Symfony's `UrlMatcher` doesn't serialize closures cleanly across the routing layer.
- Calling a service method that internally fires a PRE-EP requiring `rex::getUser()` from a Bearer route — wrap the EP-firing in a `null !== rex::getUser()` guard or replicate the service logic in your handler.
- Throwing exceptions and assuming the framework will format them — yes, `RouteCollection` catches and logs, but you lose the structured error message. For business-logic errors, return `JsonResponse(['error' => '…'], 4xx)` directly.
