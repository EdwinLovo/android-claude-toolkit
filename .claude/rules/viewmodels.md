---
name: viewmodels
description: MVVM ViewModel structure — explicit backing field for StateFlow, handleEvent dispatch, member ordering
paths:
  - "**/*ViewModel.kt"
---

# ViewModels

Extend `ViewModel()` directly. Never a `BaseViewModel`. Implement `ViewModelNav by ViewModelNavImpl()` to reuse the navigation delegate.

## Structure — top to bottom

Members appear in this order. Deviating breaks the mental model reviewers use.

1. **Injected dependencies** (`@Inject constructor(...)`)
2. **`uiState` backing field** — `StateFlow<T>` public type, `MutableStateFlow<T>` field
3. **Delegate state getters** (only when using delegates) — `val <child>State: StateFlow<T> get() = <child>Delegate.uiState`
4. **Other flows / paging** — `val items: Flow<PagingData<T>> = uiState.map { ... }.cachedIn(viewModelScope)`
5. **`init { }` block** — one-time subscriptions, delegate init calls
6. **`handleEvent(event)`** — sealed-when dispatch to private setter functions
7. **Private setter functions** — one per event or event group; each calls `uiState.reduce { copy(...) }`
8. **`companion object`** — all `const val` values used by this ViewModel

Nothing else. No public helpers, no free-floating properties, no file-level `private const val`.

## Explicit backing field

Enable in `build.gradle.kts`:
```kotlin
kotlin { compilerOptions { freeCompilerArgs.add("-Xexplicit-backing-fields") } }
```

Then:

```kotlin
@HiltViewModel
class ExampleViewModel @Inject constructor(
    private val repository: ExampleRepository,
) : ViewModel(), ViewModelNav by ViewModelNavImpl() {

    val uiState: StateFlow<ExampleUiState>
        field = MutableStateFlow(ExampleUiState())

    init {
        loadItems()
    }

    fun handleEvent(event: ExampleEvent) {
        when (event) {
            is ExampleEvent.OnRefresh -> loadItems()
            is ExampleEvent.OnItemClicked -> selectItem(event.id)
        }
    }

    private fun loadItems() {
        viewModelScope.launch {
            uiState.reduce { copy(isLoading = true) }
            repository.getItems()
                .onSuccess { items -> uiState.reduce { copy(isLoading = false, items = items) } }
                .onError { _, message -> ErrorEventBus.send(message.toUiText()) }
        }
    }

    private fun selectItem(id: String) {
        uiState.reduce { copy(selectedId = id) }
    }

    companion object {
        private const val DEBOUNCE_MS = 300L
    }
}
```

## Hard rules

- **No `_uiState` prefix.** Explicit backing field means the property IS the state; no shadow variable.
- **`MutableStateFlow` is never public.** Only `StateFlow<T>` is exposed.
- **No `uiState.value = ...` assignment.** Use `uiState.reduce { copy(...) }`.
- **No direct `MutableStateFlow.update { }`** — use the `reduce` extension in `presentation/utils/ext/StateFlowExt.kt` for consistency.
- **`handleEvent(event: <Screen>Event)` is a single function** — one `when`, one branch per event or event group. Never overload it.
- **Shared delegates (in `presentation/delegates/<name>/`) get their own public `handle<Delegate>Event(event: <Delegate>Event)` function on the ViewModel** — the delegate's event type is standalone, not a subtype of `<Screen>Event`, so its dispatch sits alongside `handleEvent`, not inside its `when`. Feature-scoped delegates (in `ux/<feature>/delegates/`) keep the subtype pattern and route through `handleEvent`. See `rules/delegates.md`.
- **`else` branch never appears** in `handleEvent` when the parent event is sealed.
- **`companion object` at the bottom.** Never file-level `private const val` in a VM file.
- **No `@Composable` in a ViewModel file.** Composables live with the screen.

## When to use a delegate

If a sub-flow (detail dialog, multi-step form, side panel) manages its own loading/error/data state, extract it into a `@ViewModelScoped` delegate. See `rules/delegates.md`.
