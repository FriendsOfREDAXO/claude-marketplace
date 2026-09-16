---
name: redaxo-api-functions
description: Building HTTP/AJAX endpoints in REDAXO using rex_api_function – the lib/ class location, naming convention, $published flag for frontend access, JSON responses, CORS for cross-domain widgets, and the rex-api-call URL. Use when the user adds an AJAX/HTTP endpoint to a custom addon, builds a JSON API for a frontend widget, or asks how to expose data via /index.php?rex-api-call=. Distinct from the FriendsOfRedaxo/api addon (covered by the redaxo-api-addon plugin).
---

# `rex_api_function` Endpoints

`rex_api_function` is the built-in REDAXO mechanism for HTTP/AJAX endpoints from your own addon. Each endpoint is a class under `lib/` that extends `rex_api_function`. URL: `/index.php?rex-api-call=<class_suffix>`.

This is **not** the same as the FriendsOfRedaxo/api addon (which is a full REST framework with route table, OpenAPI, etc.). `rex_api_function` is the lower-level core feature — perfect for one-off AJAX endpoints in a single addon.

## Skeleton

```php
<?php
// lib/api/save_item.php
class rex_api_my_addon_save_item extends rex_api_function
{
    protected $published = true; // true = callable from frontend; false = backend only

    public function execute()
    {
        // For JSON request bodies:
        $input = json_decode(file_get_contents('php://input'), true);

        if (!is_array($input)) {
            rex_response::cleanOutputBuffers();
            rex_response::setStatus(400);
            rex_response::sendJson(['error' => 'Invalid JSON']);
            exit;
        }

        // ...do work...
        $result = ['id' => 42, 'saved' => true];

        rex_response::cleanOutputBuffers();
        rex_response::setStatus(200);
        rex_response::sendJson(['data' => $result]);
        exit;
    }
}
```

## Required conventions

- **Class location:** `lib/` directory of your addon (autoloaded by REDAXO)
- **Class name prefix:** `rex_api_` — without this, REDAXO doesn't dispatch to it
- **`$published` flag:**
  - `true` – endpoint reachable from frontend (no login required)
  - `false` (default) – only callable when a backend user is logged in
- **`exit` after `sendJson()`** – REDAXO appends HTML otherwise

## Calling the endpoint

```
GET/POST /index.php?rex-api-call=my_addon_save_item
```

The URL suffix matches the part of the class name after `rex_api_`. So `rex_api_my_addon_save_item` → `?rex-api-call=my_addon_save_item`.

## CORS for cross-domain widgets

When the endpoint is called from a different origin (e.g. an embedded widget on a third-party site), set CORS headers **before** any output. Only allowlisted origins get CORS headers:

```php
public function execute()
{
    rex_response::cleanOutputBuffers();

    // CORS preflight + actual response headers – allowlisted origins only
    $allowed = ['https://partner1.example', 'https://partner2.example'];
    $origin  = rex_request::server('HTTP_ORIGIN', 'string', '');
    header('Vary: Origin');
    if (in_array($origin, $allowed, true)) {
        header('Access-Control-Allow-Origin: ' . $origin);
        header('Access-Control-Allow-Credentials: true');
        header('Access-Control-Allow-Methods: GET, POST, OPTIONS');
        header('Access-Control-Allow-Headers: Content-Type, Authorization');
    }

    if (rex_request::requestMethod() === 'options') {
        rex_response::setStatus(204);
        exit;
    }

    // ...handle the actual request...
}
```

Never echo back `HTTP_ORIGIN` unchecked together with `Access-Control-Allow-Credentials: true` – any website could then read the response with the visitor's cookies, including a logged-in session. For public, cookie-free data use `Access-Control-Allow-Origin: *` **without** the credentials header.

## Reading input safely

| Source | Code | Notes |
|---|---|---|
| Query string | `rex_request::get('id', 'int', 0)` | Type-coerced |
| Form body | `rex_request::post('name', 'string', '')` | Type-coerced |
| JSON body | `json_decode(file_get_contents('php://input'), true)` | Validate before use |
| Header | `$_SERVER['HTTP_AUTHORIZATION'] ?? ''` | Apache may strip — see below |

`rex_request::get()` / `rex_request::post()` give you type safety for free — prefer them over reading `$_GET`/`$_POST` directly.

## Apache: passing through `Authorization`

If your endpoint reads a `Bearer` token from the `Authorization` header and Apache strips it, add to `.htaccess`:

```apache
RewriteCond %{HTTP:Authorization} .
RewriteRule ^ - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
```

## Common pitfalls

- Setting `$published = false` (or omitting it) and wondering why frontend AJAX gets 403 – set it to `true` for frontend endpoints.
- Forgetting `exit` after `sendJson()` – the response gets the JSON, then a blob of REDAXO's normal HTML appended. Browsers parse this as malformed JSON.
- Setting CORS headers AFTER calling `sendJson()` – headers can't be sent after body. Set them first.
- Missing `rex_response::cleanOutputBuffers()` – stray output from `boot.php` (warnings, debug `dump()` calls) leaks into the response.
- Hardcoding API keys in the class – use `rex_config::get('my_addon', 'api_key')`.
- Endpoint URL doesn't match the class name suffix – `rex_api_my_addon_save_item` must be called as `?rex-api-call=my_addon_save_item`, not `save_item` or `my_addon/save_item`.
- Assuming CSRF protection is on – `requiresCsrfProtection()` returns `false` by default, so any site can trigger a state-changing endpoint in a logged-in visitor's browser. Override it and send the token with the request:
  ```php
  protected function requiresCsrfProtection() { return true; }
  // URL/form: rex_api_my_addon_save_item::getUrlParams() or ::getHiddenFields()
  ```
  The token (`rex_csrf_token::PARAM`, `_csrf_token`) is read via `rex_request()` from `$_REQUEST` (query string or form field), not from a JSON body or a header. On a missing or invalid token `execute()` is skipped silently – no HTTP error status.
- Trying to read JSON via `$_POST` – `$_POST` is only populated for `application/x-www-form-urlencoded` or `multipart/form-data`. For `application/json`, read `php://input`.
