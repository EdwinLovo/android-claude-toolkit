---
name: composables
description: One component per file, internal preferred, no private @Composable helpers except Preview and <Screen>ScreenContent
paths:
  - "**/presentation/ui/components/**/*.kt"
  - "**/presentation/ux/**/components/**/*.kt"
---

# Composables

Each Compose component lives in **its own file** with:
- A single `internal` (preferred) or `public` `@Composable` function
- A `private @Composable ...Preview()` at the bottom of the same file
- Nothing else — no other visible composables, no free-floating helpers

## Placement

| Component | Location |
|---|---|
| Feature-specific — only used inside one screen | `presentation/ux/<feature>/components/<Name>.kt` |
| Globally reused across features | `presentation/ui/components/<category>/<Name>.kt` |

Categories under `presentation/ui/components/` you may see: `buttons/`, `dialogs/`, `feedback/`, `footer/`, `inputs/`, `qr/`, `rows/`, `scaffold/`, `text/`, `toolbar/`. Create a new category folder only when at least two components would live there.

## Visibility

- **Default to `internal`.** Only mark `public` when the component is genuinely reused across modules — irrelevant in a single-module app, so `internal` almost always wins.
- **`private @Composable` is banned**, with exactly two exceptions:
  1. `*Preview()` functions at the bottom of the file
  2. `<Screen>ScreenContent` — the stateless companion inside `<Screen>Screen.kt`
- If you feel the pull to extract a private helper, that's the signal to extract it to its own file as `internal`.

## Parameter conventions

Follows the [Android Compose API guidelines](https://developer.android.com/develop/ui/compose/api-guidelines#modifier-parameter). Parameter order:

1. **Required params** (no defaults) first
2. **`modifier: Modifier = Modifier`** — the **first parameter with a default value**, positioned between required params and other defaulted params
3. **Other optional (defaulted) params** after modifier
4. **Trailing content lambda** (`content: @Composable () -> Unit`) last, if any

```kotlin
@Composable
internal fun ProductRow(
    // 1. Required params first (no defaults)
    product: Product,
    onClick: () -> Unit,
    // 2. Modifier — FIRST parameter with a default value
    modifier: Modifier = Modifier,
    // 3. Other optional params after modifier
    isSelected: Boolean = false,
    onLongClick: (() -> Unit)? = null,
    // 4. Trailing content lambda last (if any)
    trailingIcon: @Composable (() -> Unit)? = null,
) { ... }
```

Hard rules:
- **`modifier` defaults to `Modifier`**, never to `null` and never to a pre-wrapped modifier chain
- **Callers pass `modifier` positionally as the first optional arg** — the naming and position are the API contract
- **State is hoisted** — components consume props, do not read `hiltViewModel()` inside components

## File layout

```kotlin
// 1. package
package <PKG_ROOT>.presentation.ui.components.rows

// 2. imports

// 3. the component — one @Composable function only
@Composable
internal fun ProductRow(...) { ... }

// 4. preview at the bottom
@<PREVIEW>
@Composable
private fun ProductRowPreview() {
    <PROJECT_NAME>PreviewContainer {
        ProductRow(product = Product(...), onClick = {})
    }
}
```

**Non-composable helpers do not live in component files.** Extract them to:

- `presentation/utils/ext/<Type>Ext.kt` — extension with a natural receiver, shared across features
- `presentation/ux/<feature>/utils/<Type>Ext.kt` — extension with a natural receiver, used only inside one feature
- `presentation/ux/<feature>/utils/<Purpose>.kt` — receiver-less top-level function (rare; prefer an extension whenever a receiver exists)

See `.claude/rules/extensions.md` and `.claude/rules/utils.md` for placement details.

## Common violations

- Two `internal @Composable` functions in the same file → split into two files
- `private @Composable` helper alongside the main component → extract to `components/<Name>.kt`, mark `internal`, add its own preview
- `private fun` non-composable helper inside a component file → extract as an extension in `utils/ext/` or a receiver-less function in `<feature>/utils/` (see `rules/extensions.md`, `rules/utils.md`)
- `modifier` placed as the last parameter or after other defaulted params → move to first-with-default position
- `modifier: Modifier? = null` → change default to `Modifier`
- Reusable component in `ux/<feature>/components/` → move to `presentation/ui/components/<category>/`
- Feature-specific component in `presentation/ui/components/` when only one screen uses it → move to `ux/<feature>/components/`
- Missing preview → add one at the bottom
- Component reads `hiltViewModel()` → hoist state; components consume props, not view models
