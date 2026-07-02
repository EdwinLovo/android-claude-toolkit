---
name: delegates
description: "@ViewModelScoped delegates for isolated sub-flows — own StateFlow, init(scope), handleEvent"
paths:
  - "**/delegates/*Delegate.kt"
---

# Delegates

Use a delegate when a sub-flow (detail dialog, multi-step form, side panel, checkout step) manages its **own** loading / error / data state independently of the screen's main state.

Do not use a delegate for pure code organization — if a chunk of logic doesn't have its own state, extract it into private VM functions or a use case instead.

## Anatomy

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>.delegates

@ViewModelScoped
class <Child>Delegate @Inject constructor(
    private val repository: <Resource>Repository,
) {
    internal val uiState: StateFlow<<Child>UiState>
        field = MutableStateFlow(<Child>UiState())

    private lateinit var scope: CoroutineScope

    internal fun init(scope: CoroutineScope) {
        this.scope = scope
    }

    internal fun handleEvent(event: <Child>Event) {
        when (event) {
            <Child>Event.OnDismissed -> uiState.reduce { <Child>UiState() }
            is <Child>Event.OnConfirm -> confirm(event.value)
        }
    }

    private fun confirm(value: String) {
        scope.launch {
            uiState.reduce { copy(isLoading = true) }
            repository.submit(value)
                .onSuccess { uiState.reduce { copy(isLoading = false) } }
                .onError { _, message -> ErrorEventBus.send(message.toUiText()) }
        }
    }
}
```

## Structure rules

- **`@ViewModelScoped`** — one instance per host ViewModel; garbage-collected with the VM
- **`internal` visibility** — `uiState`, `init(scope)`, `handleEvent(event)` all `internal`. Only the ViewModel in the same module should touch these.
- **Own `StateFlow`** — do NOT nest `<Child>UiState` inside `<Screen>UiState`. A nested state dataclass would mark the parent Compose-unstable.
- **`scope` is passed in via `init(scope)`** — the delegate borrows `viewModelScope` from the parent so cancellation propagates cleanly. Never create a `CoroutineScope()` inside a delegate.
- **`handleEvent` accepts only its own `<Child>Event` type** — the parent VM narrows the event before dispatching:
  ```kotlin
  fun handleEvent(event: <Screen>Event) {
      when (event) {
          is <Child>Event -> <child>Delegate.handleEvent(event)
      }
  }
  ```

## Wiring in the ViewModel

```kotlin
@HiltViewModel
class <Screen>ViewModel @Inject constructor(
    private val <child>Delegate: <Child>Delegate,
) : ViewModel(), ViewModelNav by ViewModelNavImpl() {

    val uiState: StateFlow<<Screen>UiState>
        field = MutableStateFlow(<Screen>UiState())

    val <child>State: StateFlow<<Child>UiState>
        get() = <child>Delegate.uiState

    init {
        <child>Delegate.init(viewModelScope)
    }
}
```

The screen composable collects each delegate's state separately (`collectAsStateWithLifecycle`) and passes them into `<Screen>ScreenContent` as explicit params.

## Common misuses

- Delegate with no `StateFlow` of its own → this is not a delegate, it's a helper class or a use case
- Delegate injected into the screen composable → delegates are VM-scoped, only the VM sees them
- Delegate calling `viewModelScope` directly → use the `scope` passed via `init`
- Delegate exposed publicly → keep `uiState` / `handleEvent` `internal`
- Multiple delegates sharing state through the parent VM → merge them, or wire via a repository/flow
