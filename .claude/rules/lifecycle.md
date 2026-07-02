---
name: lifecycle
description: React to lifecycle events via small reusable Composable helpers (OnLifecycleResumed, OnLifecycleEvent); inline DisposableEffect + LifecycleEventObserver only as a last resort
paths:
  - "**/*Screen.kt"
---

# Lifecycle handling

Screens sometimes need to react to lifecycle events — refresh data when the user returns, pause a camera preview, stop a timer on backgrounding. **Almost always** this is one event → one action, which means a small named `@Composable` helper in `presentation/utils/`. Don't inline `DisposableEffect + LifecycleEventObserver` in every screen.

## Reusable helpers

Two helpers cover ~99% of cases:

| Helper | Use when |
|---|---|
| `OnLifecycleResumed { ... }` | Refresh / re-check / re-fetch when the screen becomes visible again (most common) |
| `OnLifecycleEvent(event) { ... }` | Any single non-`RESUMED` event — `ON_PAUSE`, `ON_STOP`, `ON_START`, etc. |

Both live in `presentation/utils/`. Multiple lifecycle needs on one screen? Call multiple helpers — that's cleaner than switching in one observer.

## Reference implementation

Copy these two files into `<PKG_ROOT>/presentation/utils/`. `OnLifecycleResumed` is the required baseline; add `OnLifecycleEvent` the first time a screen needs a non-resume event.

**`OnLifecycleResumed.kt`:**
```kotlin
package <PKG_ROOT>.presentation.utils

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.compose.LocalLifecycleOwner
import androidx.lifecycle.repeatOnLifecycle
import kotlinx.coroutines.launch

@Composable
fun OnLifecycleResumed(onResumed: () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(lifecycleOwner) {
        launch {
            lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.RESUMED) {
                onResumed()
            }
        }
    }
}
```

Semantics: `onResumed` runs every time the lifecycle enters `RESUMED` — including the first composition and every return from the background. Coroutine cancellation is handled by `repeatOnLifecycle`.

**`OnLifecycleEvent.kt`:**
```kotlin
package <PKG_ROOT>.presentation.utils

import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleEventObserver
import androidx.lifecycle.compose.LocalLifecycleOwner

@Composable
fun OnLifecycleEvent(event: Lifecycle.Event, onEvent: () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, e ->
            if (e == event) onEvent()
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }
}
```

Semantics: `onEvent` runs each time the specified event fires. `onDispose` removes the observer when the composable leaves composition — no leaks.

## Common patterns

**Refresh on return to screen** (by far the most common):
```kotlin
@Composable
fun <Screen>Screen(navController: NavController, viewModel: <Screen>ViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    OnLifecycleResumed { viewModel.handleEvent(<Screen>Event.OnRefresh) }

    <Screen>ScreenContent(uiState = uiState, handleEvent = viewModel::handleEvent)
    HandleNavigation(viewModel, navController)
}
```

**React to `ON_PAUSE` only** (save draft, stop a timer):
```kotlin
OnLifecycleEvent(Lifecycle.Event.ON_PAUSE) {
    viewModel.handleEvent(<Screen>Event.OnPause)
}
```

**Camera / sensor — start on resume, stop on pause:**
```kotlin
OnLifecycleResumed { viewModel.handleEvent(<Screen>Event.OnResume) }
OnLifecycleEvent(Lifecycle.Event.ON_PAUSE) { viewModel.handleEvent(<Screen>Event.OnPause) }
```

Two calls, each doing one thing. No shared state between them, no `when` switch, no observer plumbing.

## `LaunchedEffect` — pick the right key

Some effects aren't lifecycle-observer material — they're one-off side effects keyed on state. Pick the key deliberately:

| Goal | Key | Behavior |
|---|---|---|
| Run once when the composable first enters composition | `LaunchedEffect(Unit)` | Runs on first composition, never again |
| Run whenever a value changes | `LaunchedEffect(value)` | Cancels and restarts when `value` changes |
| Run when the current back-stack entry changes (arrived at this screen) | `LaunchedEffect(navController.currentBackStackEntry)` | Runs once per visit to this destination |
| One-off side effect after a specific state flag | `LaunchedEffect(uiState.someFlag)` | Careful — fires every time the flag value changes |

**`LaunchedEffect(Unit)` is the default.** Only pick a different key when the effect legitimately depends on that value. A lambda passed as key restarts the effect on every recomposition — never do that.

**One-time toast after an event:**
```kotlin
LaunchedEffect(uiState.showSavedToast) {
    if (uiState.showSavedToast) {
        // show toast
        viewModel.handleEvent(<Screen>Event.OnToastShown)  // reset the flag
    }
}
```

## Falling back to raw `DisposableEffect`

Inline `DisposableEffect + LifecycleEventObserver` is a **last resort**. Only reach for it when a single observer must switch on multiple events with **shared state** between the branches — e.g. a `Cursor` opened on `ON_START` that the `ON_STOP` branch needs to close using the same reference.

```kotlin
DisposableEffect(lifecycleOwner) {
    var cursor: Cursor? = null
    val observer = LifecycleEventObserver { _, event ->
        when (event) {
            Lifecycle.Event.ON_START -> cursor = openCursor()
            Lifecycle.Event.ON_STOP -> { cursor?.close(); cursor = null }
            else -> Unit
        }
    }
    lifecycleOwner.lifecycle.addObserver(observer)
    onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
}
```

If you find yourself writing this and the branches DON'T share state, that's the signal to split into two `OnLifecycleEvent` calls (or one `OnLifecycleResumed` + one `OnLifecycleEvent`) instead.

## Hard rules

- **Prefer `OnLifecycleResumed { ... }`** for the "refresh on return" case
- **Prefer `OnLifecycleEvent(event) { ... }`** for a single non-resume event
- **Only inline `DisposableEffect + LifecycleEventObserver`** when a single observer needs shared state across multiple events
- **All lifecycle calls happen in the entry composable**, never in `<Screen>ScreenContent` — content stays previewable and stateless
- **Every `DisposableEffect` has an `onDispose { }`** — no exceptions
- **`LaunchedEffect(Unit)` for run-once effects** — never `LaunchedEffect(true)` (identical semantics but reads worse) or a lambda key (restarts on every recomposition)
- **Handlers never mutate composable state directly** — dispatch to the VM via `handleEvent`

## Common violations

- `DisposableEffect(lifecycleOwner) { LifecycleEventObserver { _, event -> if (event == ON_RESUME) ... } ... }` in an entry composable → replace with `OnLifecycleResumed { ... }`
- Same `DisposableEffect + LifecycleEventObserver` block copy-pasted into two or more screens → extract to `presentation/utils/OnLifecycleXxx.kt` and reuse
- `DisposableEffect { addObserver(observer) }` missing `onDispose { removeObserver(observer) }` → observer leak
- `LaunchedEffect { }` with no key or `LaunchedEffect(true)` → write `LaunchedEffect(Unit)` explicitly
- Lifecycle observer / helper call inside `<Screen>ScreenContent` → hoist to the entry composable; content composables shouldn't know about lifecycle
- `OnLifecycleEvent(Lifecycle.Event.ON_RESUME) { ... }` when `OnLifecycleResumed { }` is available → use the more specific helper (its `repeatOnLifecycle(RESUMED)` semantics handle the coroutine lifecycle cleanly)
