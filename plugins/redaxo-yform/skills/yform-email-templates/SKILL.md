---
name: yform-email-templates
description: YForm email templates – placeholder syntax, sending mails from forms via tpl2email, sending mails programmatically from PHP, attachments, and using REX_YFORM_DATA placeholders. Use when the user creates or edits a YForm email template, configures tpl2email actions, sends YForm-template-based emails outside a form (e.g. cronjob, callback), or troubleshoots placeholder substitution.
---

# YForm Email Templates

YForm ships its own email-template system, separate from REDAXO's content. Templates live under **YForm → Email Templates** in the backend with a key, subject, plain-text body, optional HTML body, and From/Reply-To metadata. The recipient is not part of the template – it is set by the calling code (`tpl2email` or `mail_to`).

## Placeholder syntax

Inside a template body or subject, reference form fields with:

```
REX_YFORM_DATA[field="fieldname"]
```

For `choice` fields, two extra suffixes give you the human-readable labels instead of the stored values:

```
REX_YFORM_DATA[field="choice_field_LABELS"]    -- choice labels as comma-separated text
REX_YFORM_DATA[field="choice_field_LIST"]      -- choice labels with line breaks
```

Inline PHP also works inside templates:

```php
<?php
if ('REX_YFORM_DATA[field="anrede"]' == 'w') {
    echo "Frau";
} else {
    echo "Herr";
}
?>
```

Substitution happens after placeholder replacement, so the literal `'REX_YFORM_DATA[...]'` becomes the field value before PHP runs.

## Sending from a form (`tpl2email`)

```php
$yform->setActionField('tpl2email', [
    'template_key',  // backend template key
    'email',         // recipient: fixed address, or the name of a form field holding one
    '',              // recipient display name (literal text, not a field name)
]);
```

In pipe syntax:

```
action|tpl2email|template_key|recipient_address_or_field|[recipient_name]|[error_message_on_failure]
```

There is no sender parameter – From and Reply-To always come from the template. If the recipient slot is neither a valid address nor the name of a form field, the mail goes to `rex::getErrorEmail()` (legacy exception: an empty recipient slot followed by an address in the name slot sends to that address).

The action runs after validation passes. Multiple `tpl2email` actions per form are allowed (e.g. notify admin + send confirmation to user).

## Sending programmatically (no form context)

Use this when you need to send a YForm email template from a cronjob, a callback action, or anywhere outside the form lifecycle:

```php
$tpl = rex_yform_email_template::getTemplate('template_key');
if (!$tpl) {
    return; // template doesn't exist
}

$values = [
    'vorname'  => 'Max',
    'nachname' => 'Mustermann',
    'email'    => 'max@example.com',
];

foreach ($values as $key => $value) {
    foreach (['body', 'body_html', 'subject'] as $part) {
        $tpl[$part] = str_replace(
            'REX_YFORM_DATA[field="' . $key . '"]',
            $value,
            $tpl[$part]
        );
    }
}

$tpl['mail_to'] = 'recipient@example.com';
rex_yform_email_template::sendMail($tpl);
```

The `$tpl` array stays mutable up to `sendMail()` – set `mail_from`, `mail_to`, `mail_cc`, `mail_bcc`, `mail_reply_to`, `attachments` directly before sending.

## File attachments from a form

When a form has an `upload` field, route the uploaded files into the template attachments:

```php
$yform->setValueField('upload', ['attachment', 'Attachment']);
$yform->setValueField('php', ['php_attach', 'Attach',
    '<?php if (isset($this->params["value_pool"]["files"])) {
        $this->params["value_pool"]["email_attachments"] =
            $this->params["value_pool"]["files"];
    } ?>'
]);
$yform->setActionField('tpl2email', ['template_key', 'email']);
```

The `php` value field re-keys uploads into the `email_attachments` pool that `tpl2email` reads.

## HTML email safety

If the template uses an HTML body, every `REX_YFORM_DATA[...]` placeholder that contains user input is interpolated raw. Always wrap user-provided strings in HTML-escaping inside the template:

```html
<p>From: <?= rex_escape('REX_YFORM_DATA[field="email"]') ?></p>
```

For plain-text bodies, escaping is unnecessary but newlines from textareas come through as-is — make sure the template doesn't accidentally HTML-render those.

## Multilingual templates

Create one template per language with a `_<clang_code>` suffix (e.g. `contact_form_de`, `contact_form_en`). Pick the right one in PHP before sending:

```php
$key = 'contact_form_' . rex_clang::getCurrent()->getCode();
$tpl = rex_yform_email_template::getTemplate($key)
    ?? rex_yform_email_template::getTemplate('contact_form_de'); // fallback
```

## Common pitfalls

- Forgetting to escape placeholders in HTML bodies – HTML-injection vector.
- Sending to a `tpl2email` that doesn't exist – the action silently fails. Verify the template key in the backend.
- Setting `mail_to` from a user-controlled field without validating it as an email – open relay risk.
- Using `body_html` only without a plain-text `body` – Gmail and several corporate mail filters quarantine messages.
- Re-using the same template key across addons – the most-recently-saved template wins. Prefix template keys with the addon name.
