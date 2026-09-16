---
name: mform-basics
description: Building REDAXO module inputs with MForm – MForm::factory(), all add*Field() methods (text, textarea, select, radio, checkbox, link, media, hidden, headline, description, alert, html), wrapper elements (fieldset, column, tab, collapse, accordion, modal, inline, conditional), setters (setLabel, setAttributes, setOptions, setSqlOptions, setTooltipInfo, setFull, setSize, setMultiple, setDefaultValue), and how the form is rendered via show(). Use when the user writes or edits module INPUT PHP, uses MForm to build a backend form, or asks how to add a specific field type to a REDAXO module.
---

# MForm Basics

> **Requires MForm ≥ 9.0.** Items marked **(v9+)** are not available in MForm 8.x (check `rex_addon::get('mform')->getVersion()`).

MForm renders the backend input form of REDAXO modules. The PHP code is placed in the **INPUT** section of a module.

## Minimal example

```php
use FriendsOfRedaxo\MForm;

$mform = MForm::factory();
$mform->addTextField(1, ['label' => 'Titel']);
$mform->addTextAreaField(2, ['label' => 'Text']);
echo $mform->show();
```

## Field ID system

MForm field IDs come in three notations, depending on where the field lives:

| Context | ID notation | Stored as | Read with |
|---|---|---|---|
| Direct module field (top level) | integer `1`–`20` | `REX_VALUE[n]` | `REX_VALUE[n]` |
| Inside a repeater row | string key (`'title'`, `'link'`) | array key in the row JSON | `$row['title']` after decoding |
| Template (`MForm::fromTemplate()` / `applyTemplate()`) | string key (`'title'`, `'seo.title'`) | resolved when the template is applied — at top level the runtime maps the string to a `REX_VALUE` slot, inside a repeater it becomes an array key | depends on context |

For most plain modules use integer IDs 1–20 — that's the simple, direct mapping to `REX_VALUE[n]`. Use string keys only when you write a repeater inner form or a reusable template.

The repeater inner sub-form is described in the **mform-flex-repeater** skill; the template mechanism is documented further down on this page.

## Input fields

```php
$mform->addTextField(1, ['label' => 'Titel', 'placeholder' => 'Text…']);
$mform->addTextAreaField(2, ['label' => 'Text', 'rows' => 6]);
$mform->addHiddenField(3, 'fixed-value');
$mform->addTextReadOnlyField(4, 'Read-only text');
$mform->addTextAreaReadOnlyField(5, 'Read-only area');

// Typed inputs (renders <input type="…">)
$mform->addInputField('email', 6, ['label' => 'E-Mail']);
$mform->addInputField('url', 7, ['label' => 'URL']);
$mform->addInputField('date', 8, ['label' => 'Datum']);
$mform->addInputField('number', 9, ['label' => 'Zahl', 'min' => 0, 'max' => 100]);
```

## Select / radio / checkbox

```php
$mform->addSelectField(1, ['' => 'Bitte wählen…', 'a' => 'Option A', 'b' => 'Option B'], ['label' => 'Auswahl']);
$mform->addMultiSelectField(2, ['a' => 'A', 'b' => 'B', 'c' => 'C'], ['label' => 'Multi']); // optional 4th param: $size (default 3)
$mform->addRadioField(3, ['yes' => 'Ja', 'no' => 'Nein'], ['label' => 'Option'], 'yes');
$mform->addCheckboxField(4, [1 => 'Aktiviert'], ['label' => 'Status']);
$mform->addToggleCheckboxField(5, [1 => 'Sichtbar'], ['label' => 'Sichtbarkeit']);
$mform->addCheckboxGroupField(6, ['news' => 'News', 'blog' => 'Blog', 'event' => 'Event'], ['label' => 'Kategorien']); // (v9+)

// Optgroups: use nested array (key = group label)
$mform->addSelectField(8, [
    'Typ A' => ['a1' => 'Option A1', 'a2' => 'Option A2'],
    'Typ B' => ['b1' => 'Option B1', 'b2' => 'Option B2'],
], ['label' => 'Gruppenauswahl']);

// SQL-based options (query must return two columns: value + label)
$mform->addSelectField(7, null, ['label' => 'Artikel']);
$mform->setSqlOptions('SELECT id AS id, name AS name FROM rex_article WHERE status=1 ORDER BY name');
```

> For link, media, imagelist, and custom-link picker fields see the **mform-widgets** skill.

## Radio image variants

```php
// Image-based radio (provides visual layout picker)
$options = [];
for ($i = 1; $i <= 3; $i++) {
    $options[$i] = ['img' => rex_url::addonAssets('my_addon', "img/layout$i.svg"), 'label' => "Layout $i"];
}
$mform->addRadioImgField(1, $options, ['label' => 'Layout']);

// Icon-based radio
$mform->addRadioIconField(2, [
    'left'   => ['icon' => 'fa fa-align-left',   'label' => 'Links'],
    'center' => ['icon' => 'fa fa-align-center', 'label' => 'Mitte'],
    'right'  => ['icon' => 'fa fa-align-right',  'label' => 'Rechts'],
], ['label' => 'Ausrichtung']);

// Color radio
$mform->addRadioColorField(3, [
    'white' => ['color' => '#ffffff', 'label' => 'Weiß'],
    'black' => ['color' => '#000000', 'label' => 'Schwarz'],
    'trans' => ['color' => 'transparent', 'label' => 'Transparent'],
], ['label' => 'Farbe']);
```

## Static content elements

```php
$mform->addHeadline('Abschnitt A');
$mform->addDescription('Hinweistext für den Redakteur.');
$mform->addHtml('<hr>');
$mform->addAlertInfo('Hinweis: Diese Einstellung gilt global.');
$mform->addAlertWarning('Achtung: Nicht leer lassen!');
$mform->addAlertDanger('Fehler – Pflichtfeld.'); // alias: addAlertError()
$mform->addAlertSuccess('Konfiguration gespeichert.');
```

## Wrapper elements

### Columns (Bootstrap grid)

```php
$mform->addColumnElement(6, MForm::factory()
    ->addTextField(1, ['label' => 'Spalte 1'])
);
$mform->addColumnElement(6, MForm::factory()
    ->addTextField(2, ['label' => 'Spalte 2'])
);
```

### Fieldset

```php
$mform->addFieldsetArea('Einstellungen', MForm::factory()
    ->addTextField(3, ['label' => 'Name'])
    ->addSelectField(4, ['de' => 'Deutsch', 'en' => 'Englisch'], ['label' => 'Sprache'])
);
```

### Tabs

```php
$mform->addTabElement('Tab 1', MForm::factory()
    ->addTextField(1, ['label' => 'Inhalt Tab 1'])
, true); // true = open by default

$mform->addTabElement('Tab 2', MForm::factory()
    ->addTextAreaField(2, ['label' => 'Inhalt Tab 2'])
);
```

### Collapse / Accordion

```php
$mform->addCollapseElement('Erweiterte Einstellungen', MForm::factory()
    ->addTextField(5, ['label' => 'CSS-Klasse'])
, false, false); // openCollapse=false, hideToggleLinks=false

// Accordion (only one open at a time)
$mform->addAccordionElement('Abschnitt A', MForm::factory()->addTextField(6));
$mform->addAccordionElement('Abschnitt B', MForm::factory()->addTextField(7));
```

### Inline layout

Places multiple fields side-by-side in a single row without the Bootstrap column wrapper. The first parameter is an optional shared label; pass an empty string if you don't need one.

```php
$mform->addInlineElement('Name', MForm::factory()
    ->addTextField(1, ['label' => 'Vorname'])
    ->addTextField(2, ['label' => 'Nachname'])
);
```

### Modal sub-form (v9+)

```php
$mform->addModalElement(
    'Erweiterte Einstellungen',       // button label + modal title
    MForm::factory()
        ->addTextField(8, ['label' => 'CSS-Klasse'])
        ->addSelectField(9, ['sm' => 'Klein', 'lg' => 'Groß'], ['label' => 'Größe']),
    'btn-default',   // button class
    'left'           // alignment: left|center|right
);
```

### Conditional fieldset (show/hide based on another field) (v9+)

```php
$mform->addSelectField(1, ['img' => 'Bild', 'video' => 'Video'], ['label' => 'Medientyp']);

$mform->addConditionalFieldsetArea(
    1,          // source field ID
    '=',        // operator: = / == | != | > | < | contains | in | empty | !empty
    'img',      // compare value (for 'in': comma-separated list)
    'Bild-Einstellungen',
    MForm::factory()->addImagelistField(2, ['label' => 'Bilder'])
);

// Optional last param $action = 'hide' inverts the behaviour (hide when condition is true)
$mform->addConditionalFieldsetArea(1, '=', 'video', 'Video-URL',
    MForm::factory()->addTextField(3, ['label' => 'YouTube-URL'])
);
```

## Setters (chainable after any addField call)

```php
$mform->addTextField(1)
    ->setLabel('Titel')
    ->setPlaceholder('Bitte eingeben…')
    ->setDefaultValue('Standard')
    ->setFull()                             // full-width (no label column)
    ->setFormItemColClass('col-sm-8')       // override Bootstrap col for input
    ->setLabelColClass('col-sm-4')          // override Bootstrap col for label
    ->setTooltipInfo('Hilfetext', 'fa-info-circle')
    ->setAttribute('data-custom', 'value')
    ->setAttributes(['class' => 'my-class', 'maxlength' => 255]);

$mform->addSelectField(2, ['a' => 'A', 'b' => 'B'])
    ->setMultiple()
    ->setSize(5)
    ->setDisableOptions(['b']);
```

## Embedding another MForm instance

```php
$inner = MForm::factory()->addTextField(3, ['label' => 'Sub-Feld']);
$mform->addForm($inner);
```

## Rendering

```php
echo $mform->show();          // renders and outputs the form
$html = $mform->show();       // or capture as string
```

## Templates (project-level form reuse) (v9+)

Templates encapsulate recurring field groups in a class and register them by key. They are defined once in `project/boot.php` and applied in any module.

### 1 – Implement the interface

Place template classes in `project/lib/MFormTemplate/` (namespace `FriendsOfRedaxo\Project\MFormTemplate`):

```php
<?php

namespace FriendsOfRedaxo\Project\MFormTemplate;

use FriendsOfRedaxo\MForm;
use FriendsOfRedaxo\MFormTemplate\TemplateInterface as MFormTemplateInterface;

// Optional: project-side interface extends core interface
interface TemplateInterface extends MFormTemplateInterface {}
```

```php
<?php

namespace FriendsOfRedaxo\Project\MFormTemplate;

use FriendsOfRedaxo\MForm;

final class SeoDefaultsTemplate implements TemplateInterface
{
    public function apply(MForm $form, array $context = []): MForm
    {
        return $form
            ->addTextField('seo.title',       ['label' => 'SEO-Titel'])
            ->addTextAreaField('seo.description', ['label' => 'Meta-Beschreibung'])
            ->addMediaField('seo.image',      ['label' => 'OG-Bild']);
    }
}
```

`$context` is passed through from the call site and lets you parametrise the template (e.g. `['variant' => 'dark']`).

### 2 – Register in project/boot.php

```php
use FriendsOfRedaxo\MForm;
use FriendsOfRedaxo\Project\MFormTemplate\SeoDefaultsTemplate;
use FriendsOfRedaxo\Project\MFormTemplate\CardDefaultsTemplate;

MForm::registerTemplate('seo_defaults',  SeoDefaultsTemplate::class);
MForm::registerTemplate('card_defaults', CardDefaultsTemplate::class);
```

### 3 – Use in modules

**`MForm::fromTemplate()`** – start a new form from a template:

```php
use FriendsOfRedaxo\MForm;

echo MForm::fromTemplate('seo_defaults')
    ->addTextField('title', ['label' => 'Seitentitel'])
    ->show();
```

**`->applyTemplate()`** – merge a template into an existing form (chainable):

```php
use FriendsOfRedaxo\MForm;

$mform = MForm::factory()
    ->addTextField('headline', ['label' => 'Überschrift'])
    ->applyTemplate('card_defaults', ['variant' => 'dark'])
    ->addTextAreaField('body', ['label' => 'Text']);

echo $mform->show();
```

**Inside a repeater:**

```php
use FriendsOfRedaxo\MForm;

$itemForm = MForm::fromTemplate('card_defaults')
    ->addTextField('title', ['label' => 'Titel'])
    ->addMediaField('image', ['label' => 'Bild']);

echo MForm::factory()
    ->addRepeaterElement(1, $itemForm, true, true, ['btn_text' => 'Karte hinzufügen'])
    ->show();
```

> Multiple templates can be chained: `->applyTemplate('base')->applyTemplate('seo')`.
> Without a registered key the form is returned unchanged.

---

## Common pitfalls

- **Never reuse the same integer ID for different top-level fields** – each value slot (1–20) can only hold one value.
- **Inside a repeater, inner fields use plain string keys** (`'title'`, `'link'`, …), not dotted IDs. See the **mform-flex-repeater** skill for the full Repeater API.
- **`addRepeaterElement()` is just an alias** for `addFlexRepeaterElement()` – both exist for backwards compatibility (MForm 9; in 8.x `addRepeaterElement()` is the Alpine.js repeater).
- **`setSqlOptions()` must follow immediately** after the `addSelectField()` call it refers to.
- **`setDefaultValue()` only applies** if the value slot is empty (first render). Existing saved values take precedence.
- **`addToggleCheckboxField()`** stores `1` when checked, empty string when not. Check with `'1' === REX_VALUE[n]` in OUTPUT PHP.
- **`addCheckboxGroupField()`** stores a comma-separated string. Split with `array_filter(explode(',', $val))`.
- **Link, media, imagelist, and custom-link fields** are described in the **mform-widgets** skill.
- **Reading module output values** (resolving `redaxo://` links, decoding repeater JSON) is described in the **mform-output** skill.
