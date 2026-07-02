---
name: screens
description: Two-composable screen split — entry composable + stateless <Screen>ScreenContent
paths:
  - "**/*Screen.kt"
---

# Screen composables

Every screen file has **exactly two composables**: a public entry that collects state and delegates to a private stateless content composable.

## Template — no delegates

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>

@Composable
fun <Screen>Screen(
    navController: NavController,
    viewModel: <Screen>ViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    <Screen>ScreenContent(
        uiState = uiState,
        handleEvent = viewModel::handleEvent,
    )

    HandleNavigation(viewModel, navController)
}

@Composable
private fun <Screen>ScreenContent(
    uiState: <Screen>UiState,
    handleEvent: (<Screen>Event) -> Unit,
) {
    // UI here — never reads viewModel, never calls hiltViewModel()
}
```

## Template — with delegates

Collect each delegate state separately; pass each into `<Screen>ScreenContent`.

```kotlin
@Composable
fun <Screen>Screen(
    navController: NavController,
    viewModel: <Screen>ViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val <child>State by viewModel.<child>State.collectAsStateWithLifecycle()

    <Screen>ScreenContent(
        uiState = uiState,
        <child>State = <child>State,
        handleEvent = viewModel::handleEvent,
    )

    HandleNavigation(viewModel, navController)
}
```

Child event types are **not imported in the Screen** — only the parent `<Screen>Event`.

## Paging

If the ViewModel exposes `Flow<PagingData<T>>`, collect in the entry composable and pass into content:

```kotlin
val items = viewModel.items.collectAsLazyPagingItems()
<Screen>ScreenContent(uiState = uiState, items = items, handleEvent = viewModel::handleEvent)
```

## Hard rules

- **`collectAsStateWithLifecycle()`, never `collectAsState()`** — lifecycle-aware collection stops when the screen is stopped
- **`<Screen>ScreenContent` is `private`** — the only `private @Composable` allowed in a component file besides `*Preview()` functions (see `rules/composables.md`)
- **`<Screen>ScreenContent` takes only value-typed `uiState` and lambdas** — never the ViewModel, never `hiltViewModel()`
- **`HandleNavigation(viewModel, navController)` is called from the entry composable** — never inside `<Screen>ScreenContent`
- **Do not extract more `private @Composable` helpers into `<Screen>Screen.kt`** — put them in `presentation/ux/<feature>/components/` as `internal` composables with their own file and preview
- **Provide a screen-level preview** for `<Screen>ScreenContent` at the bottom of the same file (see `rules/previews.md`)

## Preview at the bottom of the file

```kotlin
private class <Screen>UiStateProvider : PreviewParameterProvider<<Screen>UiState> { ... }

@<PREVIEW>Screen
@Composable
private fun <Screen>ContentPreview(
    @PreviewParameter(<Screen>UiStateProvider::class) uiState: <Screen>UiState,
) {
    <PROJECT_NAME>PreviewContainer {
        <Screen>ScreenContent(uiState = uiState, handleEvent = {})
    }
}
```

## Common violations

- `viewModel.uiState.collectAsState()` (missing `WithLifecycle`) → use `collectAsStateWithLifecycle()`
- `<Screen>ScreenContent` reads `viewModel.something` → hoist the value into `uiState`
- Multiple `private @Composable` helpers in the file → extract each to `components/<Name>.kt` as `internal`
- Missing preview → add `@<PREVIEW>Screen` at bottom
- `HandleNavigation` inside `<Screen>ScreenContent` → move to the entry composable
