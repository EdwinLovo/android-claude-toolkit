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

```kotlin
@Composable
internal fun ProductRow(
    product: Product,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,   // always last non-lambda arg, always defaults to Modifier
    onLongClick: (() -> Unit)? = null,
) { ... }
```

- **`modifier: Modifier = Modifier`** is always the last non-lambda parameter, and defaults to `Modifier`
- **Trailing lambdas** (`content: @Composable () -> Unit`) go after `modifier`
- **State is hoisted** — pass in values, pass out events. Do not read `viewModel` inside `internal` components.

## File layout

```kotlin
// 1. package
package <PKG_ROOT>.presentation.ui.components.rows

// 2. imports

// 3. the component (one function only)
@Composable
internal fun ProductRow(...) { ... }

// 4. small private helpers only if they cannot be reused elsewhere (non-composable)
private fun formatPrice(cents: Int): String = ...

// 5. preview at the bottom
@<PREVIEW>
@Composable
private fun ProductRowPreview() {
    <PROJECT_NAME>PreviewContainer {
        ProductRow(product = Product(...), onClick = {})
    }
}
```

Non-composable `private fun` helpers are allowed but should be small. If they get more than ~10 lines or you feel the urge to test them, move them to `presentation/ux/<feature>/utils/` (feature-scoped) or `presentation/utils/ext/` (shared).

## Common violations

- Two `internal @Composable` functions in the same file → split into two files
- `private @Composable` helper alongside the main component → extract to `components/<Name>.kt`, mark `internal`, add its own preview
- Reusable component in `ux/<feature>/components/` → move to `presentation/ui/components/<category>/`
- Feature-specific component in `presentation/ui/components/` when only one screen uses it → move to `ux/<feature>/components/`
- Missing preview → add one at the bottom
- Component reads `hiltViewModel()` → hoist state; components consume props, not view models
