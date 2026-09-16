---
name: yform-frontend
description: Building public-facing YForm forms in REDAXO – contact forms, registrations, surveys, edit forms loading existing records, file uploads with email attachments, spam protection (honeypot/timestamp), CSRF tokens, and the full objparams cheat sheet. Use when the user creates or edits a frontend form, configures email actions, builds an edit-existing-record form, troubleshoots form rendering on the frontend, or asks how to set form_action / form_anchor / form_showformafterupdate.
---

# YForm Frontend Forms

YForm builds frontend forms from one of two equivalent definitions:

- **Pipe syntax** – one rule per line, used in modules / template fragments / Formbuilder UI
- **PHP API** – `rex_yform::factory()->setValueField/setValidateField/setActionField`

Pick whichever the surrounding code already uses. Both compile to the same form.

## Minimum contact form (PHP)

```php
<?php
$yform = rex_yform::factory();
$yform->setObjectparams('form_name', 'contact');
$yform->setObjectparams('form_action', rex_getUrl());        // submit to same page
$yform->setObjectparams('form_anchor', 'contact-form');      // jump to anchor on submit
$yform->setObjectparams('real_field_names', true);           // input names = field names

// Fields
$yform->setValueField('text',     ['name', 'Name']);
$yform->setValueField('text',     ['email', 'Email']);
$yform->setValueField('textarea', ['message', 'Message']);
$yform->setValueField('checkbox', ['agree', 'I agree to the privacy policy.']);

// Validators
$yform->setValidateField('empty', ['name',    'Please enter your name.']);
$yform->setValidateField('empty', ['email',   'Please enter your email.']);
$yform->setValidateField('type',  ['email',   'email', 'Please enter a valid email.']);
$yform->setValidateField('empty', ['message', 'Please enter a message.']);
$yform->setValidateField('empty', ['agree',   'You must agree to the privacy policy.']);

// Action: send mail using a configured email template
$yform->setActionField('tpl2email', [
    'contact_form',  // template name (managed in backend → YForm → Email Templates)
    'email',         // sender field (used as From)
    'name',          // sender display-name field
]);

// Hide the form once it was submitted successfully
$yform->setObjectparams('form_showformafterupdate', false);

echo $yform->getForm();

// After successful submit:
if (1 == $yform->objparams['actions_executed']) {
    echo '<div class="alert alert-success">Thank you – we\'ll get back to you.</div>';
}
```

## Equivalent in pipe syntax

```
objparams|form_action|/?article_id=5
objparams|form_name|contact
text|name|Name
text|email|Email
textarea|message|Message
checkbox|agree|I agree to the privacy policy.
validate|empty|name|Please enter your name.
validate|empty|email|Please enter your email.
validate|type|email|email|Please enter a valid email.
validate|empty|message|Please enter a message.
validate|empty|agree|You must agree to the privacy policy.
action|tpl2email|contact_form|email|name
action|showtext|<p>Thank you – we'll get back to you.</p>||1
```

## Important objparams

| Param | Default | Description |
|---|---|---|
| `form_action` | `index.php` | Form submit target; does not preserve the current request's query string |
| `form_action_query_params` | `[]` | Extra query params for the action URL |
| `form_method` | `post` | HTTP method |
| `form_anchor` | `` | Anchor (`#id`) used after submit |
| `form_name` | `formular` | Form name attribute (must be unique per page!) |
| `form_class` | `rex-yform` | CSS class |
| `form_wrap_id` / `form_wrap_class` | `` | Wrapper div ID/class |
| `form_label_type` | `` | `html5` for HTML5 input attributes |
| `form_ytemplate` | `bootstrap` | Frontend template (`bootstrap`, `classic`) |
| `form_show` | `true` | Render the form |
| `form_showformafterupdate` | `false` | Show the form again after submit |
| `submit_btn_label` | `Abschicken` | Submit button text |
| `submit_btn_show` | `true` | Render the default submit button |
| `error_class` | `form-warning` | CSS class for error fields |
| `real_field_names` | `false` | Use real field names (not yform indices) |
| `csrf_protection` | `true` | Enable CSRF protection |
| `debug` | `false` | Print debug info |
| `hide_top_warning_messages` | `false` | Hide the top error summary |
| `hide_field_warning_messages` | `false` | Hide per-field error messages |
| `getdata` | `false` | Load existing data (for edit forms) |
| `main_table` | `` | Table for `getdata` |
| `main_where` | `` | WHERE clause for `getdata` |
| `form_exit` | `true` | Stop processing after form |

## Edit existing record

```php
$yform = rex_yform::factory();
$yform->setObjectparams('form_action', rex_getUrl(REX_ARTICLE_ID));
$yform->setObjectparams('getdata', true);
$yform->setObjectparams('main_table', 'rex_my_table');
$yform->setObjectparams('main_where', 'id=' . (int) rex_request('id', 'int'));

// ...fields and validators (same as a create form)...

// manage_db: insert or update depending on existence
$yform->setActionField('manage_db', ['rex_my_table']);
echo $yform->getForm();
```

For dataset-driven edit forms, use the dataset's own form helpers (see the `yform-datasets` skill).

## File upload + email attachment

```php
$yform->setValueField('upload', ['attachment', 'Attachment']);
$yform->setValueField('php', ['php_attach', 'Attach',
    '<?php if (isset($this->params["value_pool"]["files"])) {
        $this->params["value_pool"]["email_attachments"] =
            $this->params["value_pool"]["files"];
    } ?>'
]);
$yform->setActionField('tpl2email', ['contact_form', 'email', 'name']);
```

The `php` value field re-keys the uploaded files into the email-attachments pool that `tpl2email` reads.

## CSRF protection

YForm enables CSRF tokens by default — keep them enabled. The token is rendered automatically when the form is built; if you wrap the form in custom JS-driven submission, make sure the token field travels with the request.

## Spam protection

### Timestamp / dwell-time check

Real users take more than ~3 seconds to fill a form. Bots usually don't.

```
hidden|timestamp|{"type":"hidden"}|REQUEST|time
validate|customfunction|timestamp|myclass::checkTimestamp||Bot detected
```

```php
class myclass {
    public static function checkTimestamp($label, $value, $params, $return) {
        return (time() - $value) < 3; // true = error if submitted in < 3 seconds
    }
}
```

### Honeypot field

Hidden field bots fill in but humans never see.

```
text|website|{"type":"hidden"}
validate|customfunction|website|myclass::checkHoneypot||Bot detected
```

```php
class myclass {
    public static function checkHoneypot($label, $value, $params, $return) {
        return !empty($value); // true = error if filled
    }
}
```

For higher-stakes forms, integrate `friendly_captcha` or `friendsofredaxo/recaptcha` — they ship YForm value/validate types you can drop in.

## Conditional fields with `showif`

Show field B only when field A has a specific value:

```php
$yform->setValueField('choice', [
    'inquiry_type', 'Inquiry',
    'General=general,Sales=sales,Support=support',
    'general', '', '', '0', 'select'
]);
$yform->setValueField('text', ['order_id', 'Order ID', '', '', '', 'showif=inquiry_type==support']);
```

`showif` supports `==`, `!=`, `>=`, `<=`, and comma-separated value lists.

## Predefined action patterns

**Database only:**

```php
$yform->setActionField('db', ['rex_tablename']);
```

**Email only:**

```php
$yform->setActionField('tpl2email', ['template_key', '', 'email_field']);
```

**Database + Email:**

```php
$yform->setActionField('db', ['rex_tablename']);
$yform->setActionField('tpl2email', ['template_key', '', 'email_field']);
```

**Show success message:**

```php
$yform->setActionField('showtext', ['<p>Vielen Dank!</p>', '', '1']); // '1' = HTML format
```

## Frontend form inside a REDAXO module

Wrap the form in a module so editors can place it on any page:

```php
<?php
// Module input
?>
<fieldset>
    <legend>Contact form</legend>
    <div class="form-group">
        <label>Email template</label>
        <input type="text" name="REX_INPUT_VALUE[1]"
               value="REX_VALUE[1]" class="form-control"
               placeholder="contact_form">
    </div>
</fieldset>
```

```php
<?php
// Module output
$template = trim('REX_VALUE[1]') ?: 'contact_form';

$yform = rex_yform::factory();
$yform->setObjectparams('form_name', 'contact_' . REX_SLICE_ID);
$yform->setObjectparams('form_action', rex_getUrl());
$yform->setObjectparams('form_anchor', 'contact-form');

$yform->setValueField('text',     ['name', 'Name']);
$yform->setValueField('text',     ['email', 'Email']);
$yform->setValueField('textarea', ['message', 'Message']);
$yform->setValidateField('empty', ['name',    'Please enter your name.']);
$yform->setValidateField('empty', ['email',   'Please enter your email.']);
$yform->setValidateField('type',  ['email',   'email', 'Please enter a valid email.']);
$yform->setValidateField('empty', ['message', 'Please enter a message.']);
$yform->setActionField('tpl2email', [$template, 'email', 'name']);

echo '<a id="contact-form"></a>';
echo $yform->getForm();

if (1 == $yform->objparams['actions_executed']) {
    echo '<div class="alert alert-success">' . rex_escape(rex_i18n::msg('contact_form_success')) . '</div>';
}
```

The slice-id suffix keeps `form_name` unique when several slices on the same article render this form – an article-id suffix would be identical for all of them.

## Common pitfalls

- Reusing the same `form_name` for two forms on one page – field names (`FORM[<form_name>][…]`) and the CSRF token id derive from it, so submitting one form makes the other one process the request as well. With `real_field_names` enabled, `form_name` no longer separates the inputs at all: every form reads the same top-level request keys (`send`, field names).
- Losing GET parameters on submit – `form_action` defaults to `index.php`, and `rex_getUrl()` only appends the `$params` passed to it, so a query parameter of the current request (e.g. a detail page's `?id=`) is gone after POST. Add them with `$id = rex_request::get('id', 'int', 0); if ($id) { $yform->setObjectparams('form_action_query_params', ['id' => $id]); }` – an explicit default avoids appending `id=0` when the current request has none.
- Sending unescaped form values to email templates – HTML email is exploitable. Use plain-text bodies, or `rex_escape()` placeholders in the HTML body.
- Forgetting `setObjectparams('real_field_names', true)` – field input names get yform-mangled, breaking analytics and JS form-builder integrations.
- Not handling the `actions_executed` state – the form re-renders empty after submit but no confirmation appears.
- Skipping a privacy/consent checkbox – required under GDPR for personal-data forms in the EU.
- Mixing `manage_db` with explicit `db` action – `manage_db` already does the insert/update, adding `db` causes double-writes.
