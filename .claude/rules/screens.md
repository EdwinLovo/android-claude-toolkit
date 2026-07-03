---
name: screens
description: Two-composable screen split — entry composable + stateless <Screen>ScreenContent
paths:
  - "**/*Screen.kt"
---

# Screen composables

Every screen renders through the app's single `<SCAFFOLD>` composable — one component, shared by every feature. That's how background color, insets, and any global chrome stay consistent across screens. Screens pass their own top/bottom bars as slot arguments; they never fork a per-feature scaffold variant.

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
    <SCAFFOLD> { paddingValues ->
        // UI here — apply paddingValues to the top-level container.
        // Never reads viewModel, never calls hiltViewModel().
    }
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

@Composable
private fun <Screen>ScreenContent(
    uiState: <Screen>UiState,
    <child>State: <Child>UiState,
    handleEvent: (<Screen>Event) -> Unit,
) {
    <SCAFFOLD> { paddingValues ->
        // UI here — reads uiState + <child>State, dispatches via handleEvent.
    }
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

- **One `<SCAFFOLD>` for the whole app.** A single `<SCAFFOLD>` composable at `<PKG_ROOT>/presentation/ui/components/scaffold/<SCAFFOLD>.kt` is the *only* scaffold in the codebase. Every feature screen calls it directly. Screen-specific chrome (top bar, bottom bar) is passed in as slot arguments — never wrapped in a new `<Feature>Scaffold` composable.
- **`<Screen>ScreenContent` wraps its UI in `<SCAFFOLD>`** — that's how the app's background color, top/bottom bars, and inset paddings are wired. Apply the `paddingValues` from the `content` lambda to the top-level container.
- **App-shell exception**: the composable that hosts `AppNavHost` (typically `MainScreen`) does **not** use `<SCAFFOLD>` — it operates below the scaffold level, and each hosted screen brings its own.
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
- `<Screen>ScreenContent` body starts with `Column {}` / `Box {}` at the root → wrap in `<SCAFFOLD> { paddingValues -> ... }`; background color and window insets go missing otherwise
- `Scaffold { }` from `androidx.compose.material3` imported and used inline in a screen → use `<SCAFFOLD>`; the file must not import Material3's `Scaffold`
- Feature-specific scaffold wrapper (`<Feature>Scaffold`) declared in `presentation/ux/<feature>/components/` → delete it; the screen calls `<SCAFFOLD>` directly and passes its top/bottom bar composables as slot arguments
