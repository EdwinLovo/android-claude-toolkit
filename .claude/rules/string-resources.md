---
name: string-resources
description: All user-visible strings come from strings.xml via stringResource; enums use @StringRes labelRes
paths:
  - "**/presentation/**/*.kt"
---

# String resources

Every user-visible string in a composable comes from `strings.xml` via `stringResource(R.string.xxx)`. Never hardcode literals in UI code.

## Basics

```kotlin
// ❌ Hardcoded
Text(text = "Save")

// ✅ Resource
Text(text = stringResource(R.string.save))
```

## Formatted strings

Use `stringResource` overloads with format args:

```kotlin
// strings.xml
// <string name="hello_user">Hello %1$s, you have %2$d items</string>

Text(text = stringResource(R.string.hello_user, userName, itemCount))
```

## Plurals

Plural strings live in `plurals.xml`, accessed with `pluralStringResource`:

```kotlin
// plurals.xml
// <plurals name="items_count">
//   <item quantity="one">%d item</item>
//   <item quantity="other">%d items</item>
// </plurals>

Text(text = pluralStringResource(R.plurals.items_count, count, count))
```

## Enum labels for the UI

An enum whose entries appear in the UI carries the resource id as a property:

```kotlin
enum class CatalogTab(@param:StringRes val labelRes: Int) {
    Products(R.string.catalog_tab_products),
    Services(R.string.catalog_tab_services),
    Packages(R.string.catalog_tab_packages),
}

@Composable
fun TabRow(tab: CatalogTab) {
    Text(text = stringResource(tab.labelRes))
}
```

**Why `@param:StringRes`** — targets the constructor parameter (which Kotlin lint uses to enforce `@StringRes`). The plain `@StringRes` annotation targets the property, which some tools don't check.

## Naming conventions in `strings.xml`

- **Feature-scoped keys** — `<screen>_<element>_<state>`:
  - `checkout_button_pay_now`
  - `catalog_empty_state_title`
  - `patientprofile_field_email_hint`
- **Reusable global strings** — plain descriptive keys: `save`, `cancel`, `delete`, `error_generic`
- **Alphabetize within groups** — makes conflicts obvious in code review
- **Never duplicate** — if two features use the same phrase, one shared key is cheaper to translate than two identical keys

## Hard rules

- **No literal strings** in composables — everything through `stringResource` / `pluralStringResource`
- **User-visible strings outside a composable use `UiText`, never `Context.getString(...)`.** ViewModels, delegates, repositories, use cases, and domain-model factories carry `UiText` values (`UiText.StringResource(R.string.xxx)` for keyed strings, `UiText.DynamicString(value)` for pre-resolved server text). The composable renders via `uiText.asString()` — no `Context` needed. See `rules/error-handling.md` for the `UiText` definition.
- **Enum UI labels use `@param:StringRes val labelRes: Int`** — never a `String` field
- **Content descriptions use `stringResource` too** (see `rules/icons.md`) — never hardcoded
- **No string concatenation for translated output** — use format placeholders (`%1$s`, `%2$d`) so translators can reorder

## Exceptions

Non-user-facing strings can be literals:
- Log messages (`Timber.d("Refresh started")`)
- Analytics event names (`analytics.track("screen_view")`)
- Route/param keys, `SavedStateHandle` keys — declare as `private const val`

## Common violations

- `Text("Save")` in a composable → `Text(stringResource(R.string.save))`
- `Text(stringResource(R.string.hello) + " " + userName)` → use a format placeholder: `Text(stringResource(R.string.hello_user, userName))`
- Enum with a `label: String` field → change to `@param:StringRes val labelRes: Int`
- `contentDescription = "Delete"` → `contentDescription = stringResource(R.string.delete)`
- Same string key defined twice for two features → collapse into one shared key
- `context.getString(R.string.xxx)` called from a ViewModel / delegate / repository / use case / domain model → produce a `UiText.StringResource(R.string.xxx)` instead; the composable renders it with `uiText.asString()`
- Non-UI layer injects `@ApplicationContext Context` just to call `getString` → drop the injection; return `UiText` and let the composable resolve
