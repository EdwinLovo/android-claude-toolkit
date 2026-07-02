---
name: error-handling
description: Errors flow through ErrorEventBus as UiText; ViewModels don't own error dialogs
paths:
  - "**/*Repository*.kt"
  - "**/*ViewModel.kt"
  - "**/*Delegate.kt"
---

# Error handling

Errors from the data layer flow through a single global `ErrorEventBus`. `MainScreen` (or the root scaffold) collects the bus once and renders a single error dialog. Individual screens never re-implement error-dialog plumbing.

## Pieces

- `presentation/utils/ErrorEventBus.kt` — a singleton `object` exposing `events: SharedFlow<UiText>` and `send(message: UiText)`
- `presentation/utils/UiText.kt` — a sealed type: `UiText.StringResource(@StringRes val id: Int, val args: List<Any>)` and `UiText.DynamicString(val value: String)`
- `presentation/utils/ext/UiTextExt.kt` — `String?.toUiText()` fallback + `UiText.asString(context)` extension for rendering

## Flow

1. Repository returns `ApiResult.Error(code, message)` — never throws
2. ViewModel / delegate collects the result and dispatches to the bus:

```kotlin
repository.getById(id)
    .onSuccess { resource -> uiState.reduce { copy(resource = resource) } }
    .onError { _, message -> ErrorEventBus.send(message.toUiText()) }
```

3. `MainScreen` observes the bus once at the top of the compose tree:

```kotlin
ObserveAsEvents(ErrorEventBus.events) { uiText ->
    // update local error state, or set a variable that renders <PROJECT_NAME>ErrorAlertDialog
}
```

`ObserveAsEvents` (`presentation/utils/ObserveAsEvents.kt`) is a helper that collects a `Flow` with lifecycle awareness inside a composable.

## Why a global bus (and not per-screen)

- **Consistency** — every error looks the same (same dialog wrapper, same styling)
- **Composition** — even background work triggered by a delegate on screen A can surface an error while the user has navigated to screen B
- **Testability** — the bus is a pure Kotlin `SharedFlow`; VMs test the "send this message" contract without pulling in Compose

## Hard rules

- **Repositories never throw** — always wrap in `ApiResult<T>` via `safeCallSuspend` / `safeCall`
- **ViewModels never own an error dialog** — they push `UiText` into the bus and move on
- **Errors that need contextual UI** (form field validation, "retry" affordances) live in `UiState`, not the bus. The bus is for one-shot user-facing errors.
- **Use `UiText`, not raw `String`** — the bus payload is always a `UiText` so localization works at render time
- **Fallback with `.toUiText()`** — a `String?` from an API becomes `UiText.DynamicString(value)` or a default `UiText.StringResource(R.string.error_generic)` when null

## Common violations

- `try { ... } catch (e: Exception) { ... }` inside a ViewModel calling a repo → the repo returns `ApiResult`, never throws
- `showErrorDialog` state inside a screen's `UiState` → for one-shot errors, use the bus; for form errors, keep them field-scoped
- `ErrorEventBus.send(rawString)` — always wrap in `UiText`
- ViewModel dispatching to the bus AND setting a `UiState.error` field → pick one; if the user needs to retry a specific action, use `UiState.error`; if it's a one-off notification, use the bus
- Rendering `UiText` by calling `.toString()` → use `UiText.asString(LocalContext.current)` so string resources resolve
