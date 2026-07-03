---
name: error-handling
description: Errors flow through a global Channel-backed ErrorEventBus as UiText values; ViewModels don't own error dialogs
paths:
  - "**/*Repository*.kt"
  - "**/*ViewModel.kt"
  - "**/*Delegate.kt"
---

# Error handling

Errors from the data layer flow through a single global `ErrorEventBus`. `MainScreen` (or the root scaffold) collects the bus once and renders a single error dialog. Individual screens never re-implement error-dialog plumbing.

**Setting up the error-handling files in a new project:** invoke the `misc-primitives` skill (`.claude/skills/misc-primitives/SKILL.md`). It ships `ErrorEventBus.kt`, `UiText.kt`, and `ObserveAsEvents.kt` under `presentation/utils/`.

## Pieces

- **`presentation/utils/ErrorEventBus.kt`** — a singleton `object` backed by a `Channel<UiText>(capacity = Channel.BUFFERED)`, exposing `events: Flow<UiText>` via `receiveAsFlow()` and a `suspend fun send(error: UiText)`. Includes a `fun drain()` for test cleanup.
- **`presentation/utils/UiText.kt`** — a sealed interface with two subtypes: `StringResource(@param:StringRes val id: Int)` and `DynamicString(val value: String)`. Plus `@Composable fun UiText.asString(): String` for rendering and `fun String?.toUiText(): UiText` for API-error normalization.
- **`presentation/utils/ObserveAsEvents.kt`** — a lifecycle-aware `@Composable` that collects a `Flow<T>` using `repeatOnLifecycle(STARTED)` on `Dispatchers.Main.immediate`.

## Reference implementation

Copy these three files verbatim into any new project (`<PKG_ROOT>/presentation/utils/`). The rest of this rule assumes they exist.

**`ErrorEventBus.kt`:**
```kotlin
package <PKG_ROOT>.presentation.utils

import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.receiveAsFlow

object ErrorEventBus {
    private val _events = Channel<UiText>(capacity = Channel.BUFFERED)
    val events: Flow<UiText> = _events.receiveAsFlow()

    suspend fun send(error: UiText) {
        _events.send(element = error)
    }

    // Drains buffered events left over from a previous test so the channel is clean.
    fun drain() {
        generateSequence { _events.tryReceive().getOrNull() }.count()
    }
}
```

**`UiText.kt`:**
```kotlin
package <PKG_ROOT>.presentation.utils

import androidx.annotation.StringRes
import androidx.compose.runtime.Composable
import androidx.compose.ui.res.stringResource
import <PKG_ROOT>.R

sealed interface UiText {
    data class StringResource(@param:StringRes val id: Int) : UiText
    data class DynamicString(val value: String) : UiText
}

@Composable
fun UiText.asString(): String = when (this) {
    is UiText.StringResource -> stringResource(id)
    is UiText.DynamicString -> value
}

fun String?.toUiText(): UiText =
    this?.let { UiText.DynamicString(it) } ?: UiText.StringResource(R.string.error_generic)
```

**`ObserveAsEvents.kt`:**
```kotlin
package <PKG_ROOT>.presentation.utils

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.ui.platform.LocalLifecycleOwner
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.repeatOnLifecycle
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.withContext

@Composable
fun <T> ObserveAsEvents(
    flow: Flow<T>,
    key1: Any? = null,
    onEvent: (T) -> Unit,
) {
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(lifecycleOwner.lifecycle, key1, flow) {
        lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
            withContext(Dispatchers.Main.immediate) {
                flow.collect(onEvent)
            }
        }
    }
}
```

## Flow

1. **Repository** returns `ApiError(code, message)` — never throws.
2. **ViewModel / delegate** chains `.onError { }` on the result and sends into the bus:

    ```kotlin
    viewModelScope.launch {
        repository.getById(id)
            .onSuccess { resource -> uiState.reduce { copy(resource = resource) } }
            .onError { _, message -> ErrorEventBus.send(message.toUiText()) }
    }
    ```

3. **`MainScreen`** observes the bus once at the top of the compose tree:

    ```kotlin
    ObserveAsEvents(ErrorEventBus.events) { uiText ->
        // set local state that renders <PROJECT_NAME>ErrorAlertDialog
    }
    ```

    `ObserveAsEvents` collects on `Dispatchers.Main.immediate` inside `repeatOnLifecycle(STARTED)` — collection pauses when `MainScreen` is not at least `STARTED` (e.g. app backgrounded, activity destroyed) and resumes cleanly when it returns. The `Channel` buffers events emitted during the pause; nothing is lost.

## Why a `Channel`, not a `SharedFlow`

- **Exactly-once delivery to a single collector.** Errors are UI notifications; one screen (`MainScreen`) renders them; no fan-out is needed. `Channel` gives one-shot semantics; `SharedFlow` is designed for multi-consumer broadcast.
- **No drops when the collector is briefly absent.** `MainScreen` pauses collection when off-screen (rotation, backgrounding, dialog cover). `Channel(capacity = Channel.BUFFERED)` queues events emitted during that window; a `SharedFlow` with `replay = 0` would silently discard them, and a `SharedFlow` with `replay > 0` would re-fire old errors every time a new subscriber attaches — both wrong for one-shot notifications.
- **`suspend fun send` back-pressures the producer** if the buffer somehow fills (huge burst of errors). `SharedFlow.tryEmit` returns a boolean and silently succeeds-or-drops depending on overflow strategy — an easy footgun.

## Why a global bus (and not per-screen state)

- **Consistency** — every error renders through the same dialog wrapper and styling.
- **Composition** — background work triggered by a delegate on screen A can surface an error while the user has already navigated to screen B; the bus doesn't care where the send came from.
- **Testability** — the bus is pure Kotlin (a `Channel` plus a `Flow`); VMs can be tested against the "did this method send X?" contract without pulling in Compose. Reset between tests with `ErrorEventBus.drain()`.

## Hard rules

- **Repositories never throw** — always wrap in `ApiResult<T>` via `safeCallSuspend` / `safeApolloCallSuspend`.
- **ViewModels never own an error dialog** — they push `UiText` into the bus and move on.
- **`ErrorEventBus.send(...)` is `suspend`** — call it from inside a coroutine (already true in `viewModelScope.launch { }`, delegate `scope.launch { }`, and inside `.onError { }` chained on an `ApiResult` where the surrounding block is already suspending).
- **Errors that need contextual UI** (form field validation, "retry" affordances) live in `UiState`, not the bus. The bus is for one-shot notifications only.
- **Use `UiText`, not raw `String`** — the bus payload is always a `UiText` so localization works at render time.
- **Fallback with `.toUiText()`** — a `String?` from an API becomes `UiText.DynamicString(value)` on non-null, or `UiText.StringResource(R.string.error_generic)` when null.
- **Reset between tests** — call `ErrorEventBus.drain()` in `@Before` (or your test framework's setup) so buffered events from a previous test don't leak.

## Common violations

- `try { ... } catch (e: Exception) { ... }` inside a ViewModel calling a repo → the repo returns `ApiResult`, never throws.
- `showErrorDialog: Boolean` state inside a screen's `UiState` for a one-shot notification → use the bus. Reserve `UiState.error` for **contextual** errors (form field, retry affordance).
- `ErrorEventBus.send(rawString)` → always wrap in `UiText` (`rawString.toUiText()`).
- ViewModel dispatching to the bus AND setting a `UiState.error` field → pick one; if the user needs to retry a specific action, use `UiState.error`; if it's a one-off notification, use the bus.
- Rendering `UiText` via `.toString()` → use `uiText.asString()` (it's `@Composable`, no `Context` parameter needed).
- `UiText.StringResource(id, listOf(arg1, arg2))` — the type has no `args` list. If you need format args, keep them at the call site: `stringResource(id, arg1, arg2)` inside the composable that renders the message, or store a fully-formatted `UiText.DynamicString`.
- `ErrorEventBus` backed by `SharedFlow` / `MutableSharedFlow` — must be `Channel<UiText>(capacity = Channel.BUFFERED)` exposed via `receiveAsFlow()`. See "Why a `Channel`" above.
- Calling `ErrorEventBus.send(...)` from a non-suspend context — wrap in `viewModelScope.launch { }` (VM) or `scope.launch { }` (delegate).
- Skipping `ErrorEventBus.drain()` in tests that assert on collected events → prior-test buffer leaks; assertions become flaky.
