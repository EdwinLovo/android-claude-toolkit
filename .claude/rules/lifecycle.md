---
name: lifecycle
description: DisposableEffect + LifecycleEventObserver for onResume/onPause; LaunchedEffect vs LaunchedEffect(Unit)
paths:
  - "**/*Screen.kt"
---

# Lifecycle handling

Screens sometimes need to react to lifecycle events (onResume, onPause, onStart, onStop) — refresh data when the user returns, release resources when leaving, pause a camera preview. Do this from the entry composable via `DisposableEffect` + `LifecycleEventObserver`.

## Reacting to lifecycle events

```kotlin
@Composable
fun <Screen>Screen(
    navController: NavController,
    viewModel: <Screen>ViewModel = hiltViewModel(),
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    val lifecycleOwner = LocalLifecycleOwner.current

    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, event ->
            when (event) {
                Lifecycle.Event.ON_RESUME -> viewModel.handleEvent(<Screen>Event.OnResume)
                Lifecycle.Event.ON_PAUSE -> viewModel.handleEvent(<Screen>Event.OnPause)
                else -> Unit
            }
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }

    <Screen>ScreenContent(uiState = uiState, handleEvent = viewModel::handleEvent)
    HandleNavigation(viewModel, navController)
}
```

- **`DisposableEffect(lifecycleOwner)`** — re-subscribes if the lifecycle owner changes (rare, but correct)
- **`onDispose { removeObserver(...) }`** — mandatory; leaking the observer keeps the composable alive after the screen is gone
- **Dispatch events, don't mutate state** — the VM decides what to do with `OnResume` / `OnPause`
- **`when (event)` uses `else -> Unit`** to satisfy exhaustiveness without listing every event you don't care about

## `LaunchedEffect` — pick the right key

| Goal | Key | Behavior |
|---|---|---|
| Run once when the composable first enters composition | `LaunchedEffect(Unit)` | Runs on first composition, never again |
| Run whenever a value changes | `LaunchedEffect(value)` | Cancels and restarts when `value` changes |
| Run when the current back-stack entry changes (e.g. arrived at this screen) | `LaunchedEffect(navController.currentBackStackEntry)` | Runs once per visit to this destination |
| One-off side effect after a specific state | `LaunchedEffect(uiState.someFlag)` | Careful — this fires *every* time the flag changes value |

**`LaunchedEffect(Unit)` is the default.** Only pick a different key when the effect legitimately depends on that value. Watch out for accidental restarts — a lambda passed as key restarts the effect on every recomposition.

## Common patterns

**Refresh on return to screen:**
```kotlin
LaunchedEffect(navController.currentBackStackEntry) {
    viewModel.handleEvent(<Screen>Event.OnRefresh)
}
```

**One-time toast after an event:**
```kotlin
LaunchedEffect(uiState.showSavedToast) {
    if (uiState.showSavedToast) {
        // show toast
        viewModel.handleEvent(<Screen>Event.OnToastShown)  // reset the flag
    }
}
```

**Camera/sensor resource that must pause with the screen:** use the `DisposableEffect` + `LifecycleEventObserver` pattern above and pause on `ON_PAUSE`, resume on `ON_RESUME`.

## Hard rules

- **`onDispose { }` present in every `DisposableEffect`** — no exceptions
- **`LifecycleEventObserver` handlers never mutate composable state directly** — dispatch to the VM
- **`LaunchedEffect(Unit)` for run-once effects** — do not use `LaunchedEffect(true)` (identical but reads worse) or a lambda key (restarts on every recomposition)
- **Screen lifecycle events belong in the entry composable**, not in `<Screen>ScreenContent` — `ScreenContent` should be stateless and previewable

## Common violations

- `DisposableEffect { ... }` missing `onDispose` → observer leaks
- `LaunchedEffect { }` with no key → compiles, but semantics are unclear; write `LaunchedEffect(Unit)` explicitly
- Observing lifecycle inside a component composable (not the entry) → hoist to the screen; components shouldn't know about lifecycle
- Reading `LifecycleEventObserver` events to update Compose state directly → dispatch to the VM instead
