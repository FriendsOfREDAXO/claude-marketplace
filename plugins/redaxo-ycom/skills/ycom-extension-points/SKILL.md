---
name: ycom-extension-points
description: YCom extension points (YCOM_CONFIG, YCOM_AUTH_INIT, YCOM_AUTH_LOGIN, YCOM_USER_CREATE, etc.), the injection system for intercepting the auth flow (OTP, password change, terms-of-use, custom injections), session management (rex_ycom_user_session), and user activity logging (rex_ycom_log). Use when the user customizes the auth flow, intercepts login/logout, adds custom auth steps (e.g. mandatory profile completion), manages sessions, or sets up activity logging.
---

# YCom Extension Points, Injections, Sessions, Logging

## Extension points

Register in your addon's `boot.php` (no special guard needed — YCom EPs only fire when the auth plugin is active).

| EP | Fired in | Purpose |
|---|---|---|
| `YCOM_CONFIG` | `rex_ycom_config::init()` | Override config values at runtime |
| `YCOM_AUTH_INIT` | `rex_ycom_auth::init()` | After login check, before injections run |
| `YCOM_AUTH_LOGIN` | `rex_ycom_auth::login()` | After successful login |
| `YCOM_AUTH_LOGIN_SUCCESS` | `rex_ycom_auth::login()` | On credential login success |
| `YCOM_AUTH_USER_CHECK` | `articleIsPermitted()` | Add custom article permission rules |
| `YCOM_AUTH_MATCHING` | `auth/boot.php` | Map external auth data to user (generic) |
| `YCOM_AUTH_SAML_MATCHING` | `auth/boot.php` | SAML-specific field mapping |
| `YCOM_AUTH_OAUTH2_MATCHING` | `auth/boot.php` | OAuth2-specific field mapping |
| `YCOM_AUTH_CAS_MATCHING` | `auth/boot.php` | CAS-specific field mapping |
| `YCOM_ARTICLE_PERM_SELECT` | `group/boot.php` | Extend the article permission sidebar |
| `YCOM_USER_CREATE` | `createUserByEmail()` | Before creating a user |
| `YCOM_USER_UPDATE` | `updateUser()` | Before updating a user |
| `YCOM_USER_STATUS_OPTIONS` | `getStatusOptions()` | Customize status dropdown options |

### Example: enforce mandatory profile fields after login

```php
rex_extension::register('YCOM_AUTH_LOGIN_SUCCESS', function (rex_extension_point $ep) {
    $user = $ep->getSubject(); // the logged-in user
    if ($user instanceof rex_ycom_user) {
        if (empty($user->getValue('firstname')) || empty($user->getValue('name'))) {
            rex_ycom_auth::setSessionVar('profile_incomplete', true);
        }
    }
});
```

Then add an injection that redirects users with `profile_incomplete=true` to a profile-completion form.

## Injection system

Injections intercept the auth flow to enforce additional requirements (OTP not yet completed, password change required, terms not accepted yet). They run after `init()` but before the article renders.

### Built-in injections

| Injection | Priority | Trigger |
|---|---|---|
| `rex_ycom_injection_otp` | 1 | OTP not verified in session |
| `rex_ycom_injection_passwordchange` | 4 | `new_password_required=1` on user |
| `rex_ycom_injection_termsofuse` | 8 | `termsofuse_accepted!=1` on user |

Lower priority numbers run first. OTP runs before password change so a stale password can't be set without 2FA.

### Custom injection

The abstract base class is — verbatim, including the missing `s` — `rex_ycom_injection_abtract`. The typo is in the YCom source (`plugins/auth/lib/injections/abstract.php`), not in this skill. Use it as-is; `rex_ycom_injection_abstract` does not exist and would throw a fatal error.

```php
class my_injection extends rex_ycom_injection_abtract {
    public function getRewrite(): bool|string {
        $user = rex_ycom_auth::getUser();
        if (!$user) return false;

        if ($user->getValue('profile_complete') != 1) {
            return rex_getUrl($completeProfileArticleId);
        }

        return false;
        // Return convention:
        //   false   = skip this injection (no redirect)
        //   true    = stay on current page (rare; the auth flow continues)
        //   string  = redirect to this URL
    }

    public function getSettingsContent(): string { return ''; }
    public function triggerSaveSettings(): void {}
}

// Register in boot.php
rex_ycom_auth::addInjection(new my_injection(), 5);
```

The priority `5` slots this injection between `passwordchange` (4) and `termsofuse` (8).

## Sessions

YCom stores sessions in the `rex_ycom_user_session` table:

| Column | Use |
|---|---|
| `session_id` | PHP session ID |
| `cookie_key` | Stay-logged-in cookie value |
| `user_id` | The owner |
| `ip` / `useragent` | Capture for audit |
| `starttime` / `last_activity` | TTL enforcement |
| `otp_verified` | OTP state for this session |

### Session API

```php
// Singleton instance
$session = rex_ycom_user_session::getInstance();

// Clear all expired sessions (good for cronjob)
rex_ycom_user_session::clearExpiredSessions();

// Force single-session: kick all other sessions for a user
$session->removeSessionsExceptCurrent($userId);

// Nuclear option: delete every session in the system (logs everyone out)
rex_ycom_user_session::deleteAllSessions();
```

Single-session enforcement at login:

```php
rex_extension::register('YCOM_AUTH_LOGIN_SUCCESS', function (rex_extension_point $ep) {
    $user = $ep->getSubject();
    if ($user instanceof rex_ycom_user) {
        rex_ycom_user_session::getInstance()
            ->removeSessionsExceptCurrent($user->getId());
    }
});
```

## Activity logging

YCom can log auth events to a dedicated log file:

```php
rex_ycom_log::activate();
rex_ycom_log::log($user, rex_ycom_log::TYPE_LOGIN_SUCCESS, ['extra' => 'info']);
```

### Log types

`access`, `logout`, `click`, `login_failed`, `login_not_found`, `login_success`, `login_updated`, `login_deleted`, `registerd`, `session_failed`, `cookie_failed`, `session_impersonate`

Log file path: `rex_path::log('ycom_user.log')`.

Activate logging once in `boot.php` (not on every request):

```php
if (!rex_ycom_log::isActive()) {
    rex_ycom_log::activate();
}
```

For high-volume sites, consider feeding the log into a centralized log aggregator instead of relying on file rotation.

## Common pitfalls

- Registering an EP callback that always returns `false` instead of returning the modified subject – downstream listeners then see the broken value.
- Custom injection priority colliding with a built-in one – the order between same-priority injections is undefined. Always pick a unique number.
- Modifying the user object inside `YCOM_AUTH_LOGIN_SUCCESS` without saving it – changes don't persist. Call `$user->save()` (and watch for validation failures).
- `rex_ycom_user_session::clearExpiredSessions()` only removes session DB rows; the underlying PHP session files aren't touched. Configure PHP's `session.gc_maxlifetime` separately.
- Activity logging captures IP and UA – check GDPR / data-retention policies before enabling on production for EU users.
- `rex_ycom_log::TYPE_REGISTERD` is misspelled in the constant (missing `e`) — that's the actual identifier; don't "fix" it.
