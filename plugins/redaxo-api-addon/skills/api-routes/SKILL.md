---
name: api-routes
description: Calling the FriendsOfRedaxo/api REST endpoints (v1.2+) — Bearer + Backend-Session auth, the route table for articles/categories/slices/modules/templates/languages/media/users/metainfo, the slice POST schema, OpenAPI spec, the unified `{data, meta}` list format, and a 401/404/405/500 diagnostic flow. Covers the Backend-mirror under `/api/backend/...`, multipart media upload, metainfo values for articles/categories/media/clangs, and yrewrite cache invalidation. Use when the user calls the api addon over HTTP, hits a confusing 401/404/405, syncs articles between systems via this API, builds tooling around it, or says "API-Aufruf", "REST-API ansprechen", "Bearer-Token", "Slice anlegen per API", "Artikel über API erzeugen".
---

# REDAXO `api` Addon – Calling the API (v1.2+)

The `api` addon (Repo: `FriendsOfRedaxo/api`, Vendor-NS: `FriendsOfRedaxo\Api`) exposes a RESTful API for a REDAXO backend. Auth via Bearer token *or* backend session cookie, Symfony `UrlMatcher` routing, mounted at `/api/...` via `YREWRITE_PREPARE`.

Canonical reference (always trust over this skill if they disagree): the addon `README.md` and `lib/RoutePackage/*.php`.

## Activation & auth

- Tokens are created in the backend under **AddOns → API → Token**.
- Each token has a list of **scopes** (= route names). The token can only call routes whose scope name is in the list.
- Auth header: `Authorization: Bearer <token>`
- Two auth classes:
  - `BearerAuth` — token-based frontend API (default for most routes)
  - `BackendUser` — uses the REDAXO backend session cookie. Every Bearer route is automatically mirrored under `/api/backend/...` with `BackendUser` auth and a `backend/<scope>` scope name. These routes follow the backend user's REDAXO permissions (admin / structure / media / clang / etc.) — no token needed.

### Apache: pass through `Authorization`

Apache strips `Authorization` in some configurations. If the token is correct but the server returns `Authorization failed`, add to `.htaccess`:

```apache
RewriteCond %{HTTP:Authorization} .
RewriteRule ^ - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

## Route table (high-level)

Authoritative list lives in `lib/RoutePackage/*.php` (scopes) and the addon's `README.md` (paths + bodies). High-level overview of what's available in v1.2:

### Structure (articles / categories / slices)

| Method | Path                                                  | Scope                                  |
|--------|-------------------------------------------------------|----------------------------------------|
| GET    | `/api/structure/articles`                             | `structure/articles/list`              |
| GET    | `/api/structure/articles/{id}`                        | `structure/articles/get`               |
| POST   | `/api/structure/articles`                             | `structure/articles/add`               |
| PUT    | `/api/structure/articles/{id}`                        | `structure/articles/update`            |
| DELETE | `/api/structure/articles/{id}`                        | `structure/articles/delete`            |
| POST   | `/api/structure/categories`                           | `structure/categories/add`             |
| PUT    | `/api/structure/categories/{id}`                      | `structure/categories/update`          |
| DELETE | `/api/structure/categories/{id}`                      | `structure/categories/delete`          |
| GET    | `/api/structure/articles/{id}/slices`                 | `structure/articles/slices/list`       |
| GET    | `/api/structure/articles/{id}/slices/{slice_id}`      | `structure/articles/slices/get`        |
| POST   | `/api/structure/articles/{id}/slices`                 | `structure/articles/slices/add`        |
| PUT    | `/api/structure/articles/{id}/slices/{slice_id}`      | `structure/articles/slices/update`     |
| DELETE | `/api/structure/articles/{id}/slices/{slice_id}`      | `structure/articles/slices/delete`     |

### Templates / modules / clangs

GET/POST list+add, GET/PUT/DELETE on `{id}` for all three. Scopes: `templates/{list,get,add,update,delete}`, `modules/{list,get,add,update,delete}`, `system/clangs/{list,get,add,update,delete}`.

### Media

| Method | Path                                  | Scope                  |
|--------|---------------------------------------|------------------------|
| GET    | `/api/media`                          | `media/list`           |
| POST   | `/api/media` (multipart/form-data)    | `media/add`            |
| GET    | `/api/media/{filename}/info`          | `media/get`            |
| GET    | `/api/media/{filename}/file`          | `media/get/file`       |
| PUT    | `/api/media/{filename}/update`        | `media/update`         |
| DELETE | `/api/media/{filename}/delete`        | `media/delete`         |
| GET    | `/api/media/category`                 | `media/category/list`  |
| POST   | `/api/media/category`                 | `media/category/add`   |
| PUT    | `/api/media/category/{id}`            | `media/category/update`|
| DELETE | `/api/media/category/{id}`            | `media/category/delete`|

Multipart upload: send the file under field name `file_new` (multipart/form-data); other metadata fields (`title`, `category_id`) sit alongside as form fields.

### Users / roles

User CRUD on `/api/users[/{id}]` (`users/{list,get,add,update,delete}`), role CRUD on `/api/users/roles[/{id}]` (`users/roles/{list,get,add,update,delete,duplicate}`), and user↔role assignment:

- `GET /api/users/{id}/role` → `users/role/list`
- `POST /api/users/{id}/role/{role_id}` → `users/role/assign`
- `DELETE /api/users/{id}/role/{role_id}` → `users/role/remove`

### Metainfo (v1.2+)

Field definitions and value-roundtrips for the four metainfo prefixes (`art_*`, `cat_*`, `med_*`, `clang_*`):

- `GET /api/metainfo/types` → `metainfo/types/list`
- CRUD on `/api/metainfo/fields[/{id}]` → `metainfo/fields/{list,get,add,update,delete}`
- Per-resource values:
  - Articles: `GET/PUT /api/structure/articles/{id}/metainfo` → `metainfo/articles/values/{get,update}`
  - Categories: `GET/PUT /api/structure/categories/{id}/metainfo` → `metainfo/categories/values/{get,update}`
  - Media: `GET/PUT /api/media/{filename}/metainfo` → `metainfo/media/values/{get,update}`
  - Clangs: `GET/PUT /api/system/clangs/{id}/metainfo` → `metainfo/clangs/values/{get,update}` (admin-only via Backend-Session)

### Backend-mirror routes

Every Bearer route above is also exposed under `/api/backend/<original-path>` with scope `backend/<original-scope>`, authenticated via the REDAXO backend session cookie (no token needed). Permissions follow the user's REDAXO `complexPerm` settings — admins see everything, restricted users only their assigned structures/media-categories/clangs.

Use these for first-party backend tooling (a custom backend page calling its own API, for example) so you don't have to issue a token to your own backend session.

## List response format

All list endpoints return:

```json
{
  "data":  [ ... ],
  "meta":  { "page": 1, "per_page": 20, "total": 137, "total_pages": 7 }
}
```

Common query parameters on list endpoints:

- `page`, `per_page` — pagination (per_page is capped at 1000)
- `sort` — comma-separated `field:direction`, e.g. `name:asc,createdate:desc`. Each list endpoint has its own whitelist of allowed sort fields; passing an invalid one returns 400 with the allowed list.
- `filter[<field>]` — endpoint-specific filters

## Diagnostic flow (status codes)

| Status | Body                                                                     | Meaning                                                                |
|--------|--------------------------------------------------------------------------|------------------------------------------------------------------------|
| 401    | `{"error":"Authorization failed"}`                                       | Bearer auth failed: token invalid, or scope not in token's scope list  |
| 404    | `{"error":"Route not found"}`                                            | No route matched the path                                              |
| 405    | `{"error":"Method not allowed","allowed":[...]}`                         | Path matched but the HTTP method is wrong; `allowed` lists valid ones  |
| 500    | `{"error":"Internal server error","message":"..."}`                      | Controller threw — check `var/log/system.log` for the exception        |

Quick path:

1. **401 `Authorization failed`** → `BearerAuth`: extend the token's scope list to include the route's scope. `BackendUser` route: log into the backend first.
2. **404 `Route not found`** → check the path against `lib/RoutePackage/*.php`.
3. **405 `Method not allowed`** → use one of the methods in `allowed[]`.
4. **500 `Internal server error`** → `tail var/log/system.log` for the underlying exception. The `message` field also includes the immediate error string.
5. **HTTP 201 but frontend 404?** → yrewrite path cache stale (see below).
6. **Slice add returns 4xx / 500 with "Template has no module in such ctype"** → in the backend, check the template's module/ctype assignment.

## Slice schema

`POST /api/structure/articles/{id}/slices` body:

```json
{
  "module_id": <int, required>,
  "clang_id":  <int, required>,
  "ctype_id":  <int, default 1>,
  "value1":   "...",   "value2":   "...",   ...   "value20":   "...",
  "media1":   "...",   ...   "media10":  "...",
  "medialist1":"...", ...   "medialist10":"...",
  "link1":    "...",   ...   "link10":   "...",
  "linklist1":"...",  ...   "linklist10":"..."
}
```

One slice = one module. Which `value*` / `media*` slot maps to which editor field is defined in the module's input PHP as `REX_INPUT_VALUE[n]` / `REX_MEDIA[id=n]`.

The addon validates before insert:

- Article exists (`rex_article::get(...)`)
- Language exists (`rex_clang::getAllIds()`)
- Module exists (`rex_module`)
- **Module is allowed in the template's ctype** (`rex_template::hasModule(...)`) — otherwise the call fails. If this check fails, look at the template's ctype/module assignment in the backend.

`PUT` on a slice replaces the same fields; `DELETE` removes the slice and triggers the same EP chain as the backend page (`SLICE_DELETE` PRE → `rex_sql` delete → `SLICE_DELETED` POST → `art_content_updated`).

## Frontend visibility — yrewrite path cache

A newly-created article/category may return `404` on the frontend even with `status=1`:

- yrewrite caches URL paths in `var/cache/addon/yrewrite/...`
- `rex_article_service::addArticle()` triggers the right EPs, but yrewrite doesn't always regenerate its path cache (especially for new root categories)
- Fix: backend → **AddOns → yrewrite → Allgemein** → "Alle Caches generieren". Or in code: `rex_yrewrite::generatePathFile([])`
- There is currently **no** API endpoint for cache refresh.

## Article metainfo

Set custom metainfos (`art_*`, `cat_*`, `med_*`, `clang_*`) via the metainfo value endpoints:

```
PUT /api/structure/articles/{id}/metainfo
{ "art_color": "blue", "art_layout": "wide" }
```

Field definitions are managed via `/api/metainfo/fields` (admin-only when called via `BackendUser`; via Bearer the token needs the `metainfo/fields/*` scopes).

## OpenAPI spec

Available in the backend at `?page=api/openapi`. Generated from the route definitions via `OpenAPIConfig`. External consumers (Postman, OpenAPI generator): the JSON spec is only viewable when authenticated via backend login — Swagger UI renders client-side.

## Common pitfalls

- Adding a scope to the token that doesn't quite match the route name (e.g. `structure/article/add` vs. `structure/articles/add`) → 401 `Authorization failed`. Copy from the route source, don't re-type.
- Calling a Bearer route via the cookie session (or vice versa) → use the `/api/backend/...` mirror for backend-session calls, the plain path for Bearer.
- Building tooling that posts a slice and expects the article to be reachable on the frontend immediately → regenerate yrewrite path cache after creating articles/categories (no API endpoint exists; do it in the backend or call `rex_yrewrite::generatePathFile([])`).
- Slice POST body using `value21`, `media11`, `link11`, … — REDAXO modules only have slots 1–20 (values) / 1–10 (media/medialist/link/linklist). Extra slots are silently ignored, not validated.
- Slice POST without `module_id` + `clang_id`, or with a `module_id` that isn't allowed in the article's template/ctype → 4xx / 500 with "Template has no module in such ctype". Check the template config before retrying.
- Requesting `per_page` above 1000 — the value is silently capped at 1000, not honoured as-is. Paginate explicitly with `page=N` when you actually need more.
- Passing a `sort` field that isn't in the endpoint's whitelist → 400 with the allowed list in the body. Read that list; don't guess at `?sort=name:asc` if the endpoint only accepts e.g. `priority`/`createdate`/`updatedate`.
- 500 with no log line: check that `rex_logger` is configured and writeable; the addon logs every controller exception.
- `Authorization` header missing on Apache → add the `RewriteRule` above.
