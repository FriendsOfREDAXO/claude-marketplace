---
name: ycom-tokens
description: YCom's token system for direct login links, password reset links, and registration confirmation links. Covers ycom_user_token create/validate, the token types (login, password_reset, register), email template integration, and end-to-end flow patterns. Use when the user wants magic-link login, password-reset emails, registration-confirmation links, or any single-use token-based flow.
---

# YCom Tokens

YCom's token system replaces the old `activation_key` patterns with a proper one-time-token model. Tokens are stored, single-use, and tied to a YCom user.

## Token types

| Type | Use |
|---|---|
| `login` | Magic link – click to log in |
| `password_reset` | Click to land logged-in on the password-change form |
| `register` | Click to confirm a new registration |

You can also use custom strings as types when you need a flow that isn't one of these.

## Field syntax

```
ycom_user_token|token|create|type|email_field
ycom_user_token|token|validate|type|error_message
```

- `create` – generates a token, stores it on the matched user, and exposes it as `REX_YFORM_DATA[field=token]` for the email template
- `validate` – consumes a token from the URL; on success, the matching user becomes the current dataset (and is logged in for `login` and `password_reset` types)

> **Fourth slot in `create` is documentation, not configuration.** Despite the `email_field` label, the `create` branch reads `value_pool['sql']['email']` directly — the slot is ignored at runtime. Always name the email form field literally `email`, regardless of what you put in the fourth slot.

## Direct login via token

### Article A — token generation form

```
ycom_user_token|token|create|login|email
text|email|E-Mail
validate|empty|email|Bitte E-Mail eingeben.
validate|in_table|email|rex_ycom_user|email|E-Mail nicht gefunden.
action|tpl2email|direct_login_de|email|
action|showtext|Link wurde per E-Mail gesendet.|||1
```

### Email template

```php
<?php
$url = rex_getUrl($validation_article_id, null, ['token' => 'REX_YFORM_DATA[field=token]']);
$full_url = trim(rex::getServer(), '/') . trim($url, '.');
?>
<p><a href="<?= $full_url ?>">Direkt einloggen</a></p>
```

### Article B — token validation (the link target)

```
ycom_user_token|token|validate|login|Token ist ungueltig oder abgelaufen.
```

That single line, on the destination article, validates the token and logs the user in. Add a redirect or success message after.

## Password reset via token

```
ycom_user_token|token|create|password_reset|email
```

Validation form (combined with forced password change):

```
ycom_user_token|token|validate|password_reset|Token ungueltig.
hidden|new_password_required|1
action|ycom_auth_db|update
```

After validation, render the password-change form (see the `ycom-forms` skill) — the user is already logged in and the `new_password_required` flag will be cleared on submit.

## Registration via token

```
text|email|E-Mail
validate|type|email|email|Bitte gueltige E-Mail eingeben.
validate|unique|email|E-Mail wird bereits verwendet.|rex_ycom_user
ycom_auth_password|password|Passwort|...
hidden|status|0
action|copy_value|email|login
action|db|rex_ycom_user
ycom_user_token|token|create|register|email
action|tpl2email|register_confirm|email|
```

Important: the form must contain a field literally named `email` (see "Field syntax" above for why the fourth pipe slot doesn't help here).

Confirmation:

```
ycom_user_token|token|validate|register|Token ungueltig.
hidden|status|1
action|ycom_auth_db|update
```

This pattern is cleaner than the older `activation_key` approach (see `ycom-forms`) because the token is single-use and validated centrally.

## Common pitfalls

- The form lacks a field literally named `email` – the `create` branch reads `value_pool['sql']['email']` and throws `email not found`. The fourth pipe slot is a documentation hint, not a configurable field selector.
- Building the link URL with `rex_getUrl()` but forgetting the article ID matches the article that has the `validate` field – wrong target = `Token ungueltig`.
- Setting `csrf_protection|1` (default) on the validation article – the email link can't carry a CSRF token, so always add `objparams|csrf_protection|0` on validation forms.
- Using a `login`-type token to also reset the password – use `password_reset` type. `login` only logs in.
- Tokens don't have an explicit TTL field in pipe syntax — they expire on first use. If you need time-bounded tokens, gate the validation step with a `validate|customfunction` that checks `created_at` against now.
