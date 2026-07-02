---
name: previews
description: Every composable has a preview at the bottom of its file; renders light and dark
paths:
  - "**/*.kt"
---

# Previews

Every `@Composable` function must have a Compose preview at the bottom of its file. No exceptions — reviewers should see the component in the IDE without running the app.

## Annotations

Two annotations, both defined in `presentation/utils/<PROJECT_NAME>Preview.kt`:

- `@<PREVIEW>` — components (default size, wraps content)
- `@<PREVIEW>Screen` — full screens (target device size, e.g. 1280×800dp for tablet, 411×892dp for phone)

**Both must render light and dark in a single annotation** using `@Preview(uiMode = UI_MODE_NIGHT_YES)` variants. Never dark-only — light comes first.

## Preview container

Every preview wraps the composable in a container that applies the project theme:

```kotlin
@<PREVIEW>
@Composable
private fun ProductRowPreview() {
    <PROJECT_NAME>PreviewContainer {
        ProductRow(product = Product(...), onClick = {})
    }
}
```

The `<PROJECT_NAME>PreviewContainer` composable applies `<THEME>` and a default background.

## Naming

- Component preview → `fun <ComponentName>Preview()`
- Screen preview → `fun <ScreenName>ContentPreview()` (previews `<Screen>ScreenContent`, not the entry composable — the entry needs a NavController)

Always `private`.

## Preview parameters

For components with meaningful state variants (selected/unselected, enabled/disabled, loading/loaded, empty/populated), use `@PreviewParameter`:

```kotlin
private class ProductRowStateProvider : PreviewParameterProvider<ProductRowState> {
    override val values = sequenceOf(
        ProductRowState.Idle,
        ProductRowState.Selected,
        ProductRowState.Disabled,
    )
}

@<PREVIEW>
@Composable
private fun ProductRowPreview(
    @PreviewParameter(ProductRowStateProvider::class) state: ProductRowState,
) {
    <PROJECT_NAME>PreviewContainer {
        ProductRow(state = state, onClick = {})
    }
}
```

**Trivial providers stay in the same file.** `PreviewParameterProvider` implementations that produce simple enum entries or tiny datasets belong at the bottom of the component file.

**Complex screen-level providers go in a sibling file.** When you need to build a full `<Screen>UiState` with realistic nested data, extract the provider to `<Screen>PreviewParams.kt` next to the screen file to keep the screen file readable.

## Hard rules

- Every composable in `.kt` files under `presentation/` has a preview at the bottom — no missing previews
- Preview functions are always `private`
- Preview renders light + dark (never dark-only, never light-only)
- Wrap in `<PROJECT_NAME>PreviewContainer { }` — applies theme + background
- Screen previews target `<Screen>ScreenContent`, not the entry composable
- Prefer the smallest realistic parameter set — 2–3 variants beats a dozen

## Common violations

- Composable without a preview → add one
- `@Preview` used directly (not `@<PREVIEW>`) → migrate to the project annotation for consistent size and light/dark
- Preview without `<PROJECT_NAME>PreviewContainer` → theme won't apply, colors will be wrong
- Preview parameter provider in `PreviewParameterProviders.kt` (separate shared file) → keep trivial ones in the same file
- Multiple preview functions for the same variants → collapse with `@PreviewParameter`
