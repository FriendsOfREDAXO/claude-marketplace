---
name: yform-fields
description: YForm field types, validators, and actions – the building blocks for forms and tables. Covers all value/validate/action types with parameters, the choice / be_manager_relation / upload field configurations, custom field types, and the pipe syntax used in the Formbuilder. Use when picking field types, configuring validators, building custom field/validator classes, troubleshooting field rendering or save behavior, writing forms in pipe syntax, or when the user says "Feldtypen", "YForm-Felder", "Validierung", "Formular-Felder", "be_manager_relation", or "Formbuilder Pipe-Syntax".
---

# YForm Fields, Validators & Actions

YForm fields have three roles:

- `type_id: value` – an input that renders + (optionally) stores data (text, number, media, relation)
- `type_id: validate` – a rule applied during save (empty, type, unique, customfunction)
- `type_id: action` – post-save side effects (db, tpl2email, redirect, callback)

Pipe-syntax description (Formbuilder):

```
fieldtype|name|label|[param3]|[param4]|...
validate|type|name|[param3]|...
action|type|[param2]|[param3]|...
objparams|key|value
```

## Value field types (overview)

**Text/Input:** `text`, `textarea`, `email`, `integer`, `float`, `number`, `password`
**Date/Time:** `date`, `datetime`, `time`, `datestamp`
**Selection:** `checkbox`, `choice`, `prio`
**Relations:** `be_link`, `be_manager_relation`, `be_media`, `be_table`
**System:** `hidden`, `hashvalue`, `uuid`, `generate_key`, `index`, `ip`, `showvalue`
**Layout:** `fieldset`, `html`, `php`, `emptyname`, `submit`, `resetbutton`
**Special:** `upload`, `signature`, `google_geocode`, `objparams`, `article`

### Pipe syntax for every value field

```
text|name|label|default|[no_db]|[attributes]|[notice]|[prepend]|[append]
textarea|name|label|default|[no_db]|[attributes]|[notice]
email|name|label|default|[no_db]|[attributes]|[notice]
password|name|label|default|[no_db]|[attributes]|[notice]|[prepend]|[append]
integer|name|label|default|[no_db]|[unit]|[notice]
number|name|label|precision|scale|default|[no_db]|[widget]|[unit]|[notice]|[attributes]
float|name|label|scale
checkbox|name|label|default|[no_db]|[attributes]|[notice]|[output_values]
choice|name|label|choices|[expanded]|[multiple]|[default]|[group_by]|[preferred_choices]|[placeholder]|[group_attributes]|[attributes]|[choice_attributes]|[notice]|[no_db]
date|name|label|year_start|year_end|format|current_date|[no_db]|[widget]|[attributes]|[notice]|[modify_default]
datetime|name|label|year_start|year_end|format|current_date|[no_db]|[widget]|[attributes]|[notice]|[modify_default]
time|name|label|format|current_time|[no_db]|[widget]|[attributes]|[notice]
datestamp|name|label|format|[no_db]|only_empty|[modify_default]
hidden|name|value||[no_db]
hidden|name|key|REQUEST/GET/POST/SESSION|[no_db]
upload|name|label|sizes|types|required|messages|[notice]|[config_json]
be_link|name|label|[multiple]|[notice]
be_manager_relation|name|label|table|field|type|empty_option|empty_value|size|filter|relation_table|notice
be_media|name|label|preview|multiple|category|types|[notice]
be_table|name|label|columns|[notice]|[no_db]
fieldset|name|label|[attributes]|[options]
html|name|label|html_content
php|name|label|php_code
showvalue|name|label|default|notice
prio|name|label|fields|scope|default|[attributes]|[notice]
submit|name|labels|values|[no_db]|default|css_classes
resetbutton|name|label|value|cssclassname
hashvalue|name|label|field|function|salt|[no_db]
uuid|name|label|[no_db]|[show_value]
generate_key|name|label|only_empty|[show_value]|[no_db]
index|name|label|names|[no_db]|function|add_this_param|salt
ip|name|label|[no_db]|[server_var]
emptyname|name|label|[show_value]
signature|name|label|default|[no_db]|[attributes]|[notice]
article|article_id
objparams|key|value|[init/runtime]
```

### Parameter cheatsheet

- `no_db`: if set, value is not saved to database (form-only field)
- `attributes`: JSON for HTML attributes, e.g. `{"placeholder":"Text...","class":"my-class"}`
- `notice`: help text displayed below the field
- `widget` (date/time): `select` (default), `input:text`, `input:date`
- `format` (date): `YYYY-MM-DD`, `DD.MM.YYYY`, etc.
- `format` (datestamp): `mysql` (=`YYYY-MM-DD HH:ii:ss`), `U` (Unix timestamp), or PHP date format
- `only_empty` (datestamp): `0`=always update, `1`=only if empty (=created_at), `2`=never
- `modify_default` (date/datestamp): PHP strtotime string, e.g. `+1 week`
- `expanded` (choice): `0`=select/multiselect, `1`=radio/checkboxes
- `multiple` (choice): `0`=single, `1`=multiple
- `choices` (choice): JSON, SQL, or comma-separated
- `type` (be_manager_relation): `0`=select, `1`=select multi, `2`=popup, `3`=popup multi, `4`=inline 1:n, `5`=inline multi
- `sizes` (upload): max KB or range `100,500`
- `types` (upload): `.jpg,.gif,.png,.pdf`
- `messages` (upload): `min_err,max_err,type_err,empty_err,delete_msg`
- `options` (fieldset): `onlyopen` (default), `onlyclose`, `openclose`
- `output_values` (checkbox): custom values, e.g. `nein,ja`

## Field type: `choice`

The most versatile selection field. Renders dropdowns, radio buttons, or checkboxes depending on `expanded`/`multiple`.

| `expanded` | `multiple` | Result |
|---|---|---|
| 0 | 0 | Single-select dropdown |
| 0 | 1 | Multi-select dropdown |
| 1 | 0 | Radio buttons |
| 1 | 1 | Checkbox group |

**JSON choices:**

```json
{"Berlin":"berlin","Hamburg":"hamburg","Muenchen":"muenchen"}
```

**Grouped JSON:**

```json
{"Nord":{"Hamburg":"hamburg","Bremen":"bremen"},"Sued":{"Muenchen":"muenchen"}}
```

**SQL choices:**

```sql
SELECT id, name FROM rex_categories ORDER BY name
```

## Field type: `be_manager_relation`

Manages relations between YForm tables.

| Option | Description |
|---|---|
| `table` | Target table name |
| `field` | Display field in target table |
| `type` | 0=single select, 1=multi select, 2=popup, 3=popup multi, 4=inline 1:n, 5=inline m:n |
| `empty_option` | Show empty first option |
| `filter` | SQL WHERE filter for target records |

**1:n frontend output:**

```php
$items = rex_sql::factory()->getArray(
    'SELECT t1.*, t2.name AS category_name
     FROM rex_products t1
     LEFT JOIN rex_categories t2 ON t1.category_id = t2.id
     WHERE t1.status = 1'
);
```

**m:n frontend output:**

```php
$items = rex_sql::factory()->getArray(
    'SELECT t1.*, t2.name AS tag_name
     FROM rex_products t1
     LEFT JOIN rex_products_tags rt ON t1.id = rt.product_id
     LEFT JOIN rex_tags t2 ON rt.tag_id = t2.id
     WHERE t1.id = :id',
    ['id' => $id]
);
```

For ORM-style relation access, see the `yform-datasets` skill (`getRelatedDataset()`, `getRelatedCollection()`).

## Field type: `upload`

Configure via JSON in the `config_json` slot:

```json
{
    "accept": "image/*,.pdf",
    "max_size": 10485760,
    "upload_folder": "media/uploads/",
    "allowed_extensions": ["jpg","jpeg","png","gif","pdf"],
    "required": true
}
```

## Validate types

```
validate|empty|name|message
validate|compare|name1|name2|operator|message
validate|compare_value|name|compare_value|operator|message
validate|type|name|type|message|[not_required]
validate|intfromto|name|from|to|message
validate|size|name|size|message
validate|size_range|name|min|max|message
validate|preg_match|name|pattern|message
validate|unique|name|message|[table]|[empty_option]
validate|in_table|name|tablename|fieldname|message
validate|in_names|names|min|max|message
validate|customfunction|name|function|params|message|[validate_type]
validate|password_policy|name|message|rules_json
```

| Validator | Use for |
|---|---|
| `empty` | Required field |
| `compare` | Compare two fields (`==`, `!=`, `<`, `>`, `<=`, `>=`) |
| `compare_value` | Compare to literal value |
| `type` | `int`, `float`, `numeric`, `string`, `email`, `url`, `date`, `datetime`, `time`, `hex`, `iban`, `json` |
| `intfromto` | Integer in range |
| `size` / `size_range` | Exact / ranged string length |
| `preg_match` | Regex match |
| `unique` | Unique in table (comma-separate names for composite uniqueness, e.g. `email,company_id`) |
| `in_table` | Value must exist in another table |
| `in_names` | Value must be in field-name list |
| `customfunction` | Custom PHP callback (prefix function with `!` to negate) |
| `password_policy` | Password strength rules JSON |

**`customfunction` example:**

```php
// Pipe: validate|customfunction|my_field|myclass::validate||Error message

class myclass {
    public static function validate($label, $value, $params, $return) {
        // Return true = error, false = valid
        if ($value < 10) return true;
        return false;
    }
}
```

**`validate_type`** for customfunction: `normal` (default), `pre` (before processing), `post` (after).

**Composite unique example:**

```php
[
    'type_id'   => 'validate',
    'type_name' => 'unique',
    'name'      => 'email,company_id',
    'message'   => 'Email already used in this company.',
]
```

For update forms, also pass `not_id` to skip the current row (otherwise every save fails on its own match).

## Action types

```
action|db|tablename|[where]
action|db_query|query_with_?_placeholders|fieldnames
action|email|from|to|subject|body|[error_message]
action|tpl2email|template_name|email_field|[fixed_email]
action|callback|function_or_class::method
action|redirect|target|[request_or_label]|[field]
action|showtext|text|open_tag|close_tag|format
action|html|html_content
action|readtable|tablename|fieldname|label
action|copy_value|from_field|to_field
action|create_table|tablename
action|manage_db|[tablename]
action|encrypt_value|fieldname|algorithm|[save_in_field]
action|fulltext_value|target_field|source_fields
action|wrapper_value|fieldname|template
```

| Action | Description |
|---|---|
| `db` | Save to a YForm-managed table |
| `db_query` | Execute custom SQL with `?` placeholders |
| `email` | Send a plain email |
| `tpl2email` | Send email rendered from a YForm email template |
| `callback` | Execute PHP callback `function_or_class::method` |
| `redirect` | Redirect after submit |
| `showtext` | Display text after submit (`format`: 0=plaintext, 1=HTML, 2=Textile) |
| `html` | Output HTML |
| `readtable` | Pre-fill values from a table |
| `copy_value` | Copy field value to another field |
| `create_table` | Auto-create the underlying table from form fields |
| `manage_db` | Insert or update depending on existence (for edit forms) |
| `encrypt_value` | Encrypt the field's value |
| `fulltext_value` | Generate a fulltext-search value from source fields |
| `wrapper_value` | Wrap field value with a template |

In `email` and `showtext` bodies, use `###fieldname###` as placeholders for form values.

## Form-only fields (no DB column)

Set `no_db: 1` to render a field in the form without persisting it. Useful for honeypots, "I agree" checkboxes that get checked by a validator but not stored, or computed previews.

```php
[
    'type_id'   => 'value',
    'type_name' => 'checkbox',
    'name'      => 'agree_terms',
    'label'     => 'I agree to the privacy policy',
    'no_db'     => 1,
]
```

Validate as required without storing:

```php
[
    'type_id'   => 'validate',
    'type_name' => 'empty',
    'name'      => 'agree_terms',
    'message'   => 'You must agree to the privacy policy.',
]
```

## Custom field types

Define your own under `lib/yform/value/` or `lib/yform/validate/` in your addon. YForm autodiscovers them on boot if the class name matches `rex_yform_value_<name>` or `rex_yform_validate_<name>`.

Minimal value field:

```php
<?php
// lib/yform/value/colorpicker.php
class rex_yform_value_colorpicker extends rex_yform_value_abstract
{
    public function enterObject()
    {
        $this->setValue((string) $this->getValue());

        if ($this->needsOutput()) {
            $this->params['form_output'][$this->getId()] = $this->parse(
                ['value.colorpicker.tpl.php', 'value.text.tpl.php'],
                ['type' => 'color']
            );
        }
        $this->params['value_pool']['email'][$this->getName()] = $this->getValue();
        if ($this->saveInDb()) {
            $this->params['value_pool']['sql'][$this->getName()] = $this->getValue();
        }
    }

    public function getDescription(): string
    {
        return 'colorpicker|name|label';
    }

    public function getDefinitions(): array
    {
        return [
            'type'      => 'value',
            'name'      => 'colorpicker',
            'values'    => [
                'name'  => ['type' => 'name',  'label' => 'Field name'],
                'label' => ['type' => 'text',  'label' => 'Label'],
            ],
            'description' => 'A color picker',
            'db_type'     => ['varchar(7)'],
        ];
    }
}
```

## YForm Tools addon enhancements

If `yform_tools` is installed, attribute JSON gives you these for free:

- **Searchable select:** `{"class":"form-control selectpicker","data-live-search":"true"}`
- **Input masking:** `{"data-inputmask-alias":"datetime","data-inputmask-inputformat":"yyyy-mm-dd"}`
- **Date range picker:** `{"data-yform-tools-datepicker":"YYYY-MM-DD"}`

## Common pitfalls

- Misspelling `type_name_internal` for the `type` validator – falls back to silent pass.
- Setting `no_db: 1` on a field whose SQL column also exists – data gets ignored on save, then read back as empty.
- Missing `prio` on validators – they may fire before the value field is populated.
- Using `unique` without `not_id` in update forms – every save fails because the value matches itself.
- `customfunction` validator returning `true` to mean "valid" – it's the opposite: `true` = error, `false` = valid.
- Using `validate|customfunction|...|||1` – the trailing `1` means `validate_type=post`, runs after DB write. Default (no value) is `normal`.
