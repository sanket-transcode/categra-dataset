# Attribute & Attribute Option Label Translations

## Storage

- Translations live in `tbl_attribute_translations` as rows of (`attribute_id`, `key`, `language_code`, `translation`).
- Attribute main label: `key = 'label'`.
- Dropdown / multi-select option label: `key = <option.value>` (the option's `value` from `attribute_configuration.options[]`, never `option{index}`).
- A translation that is empty or null counts as missing, and the next fallback is used.

## Languages involved

| Name | Meaning |
|---|---|
| Interface language | The user's active UI language (`tbl_users.active_language`, mapped to a language code) |
| Selected language | The content language currently selected for the product |
| Channel default language | The default language of the selected Shopify channel (Shopify channels only) |
| Base language | The account's base language |

## Attribute main label

The first non-empty value wins:

1. `label` translation in the **interface language**
2. `label` translation in the **base language**
3. `tbl_attributes.label` (the attribute's fixed label)
4. `tbl_attributes.name`

- The selected language does **not** affect the main label; only the interface language does.

## Attribute option label

Applies to attributes of type dropdown and multi-select. The first non-empty value wins:

1. Option translation in the **selected language**
2. Option translation in the **channel default language** (only when the selected channel is Shopify; skipped otherwise)
3. Option translation in the **base language**
4. The raw `option.value`

- `option.label` from `attribute_configuration` is **not** used as a fallback.
- The stored value is always `option.value`; translations change only what is displayed.
- A stored value that matches no option is shown as-is.

## Exception

- When Shopify variant attributes are mapped, the attribute label is `tbl_attributes.label` first, then the `label` translation in the **base language**. The interface language is not used.
