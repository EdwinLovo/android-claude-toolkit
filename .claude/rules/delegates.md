---
name: delegates
description: "@ViewModelScoped delegates for isolated sub-flows — own StateFlow, init(scope), handleEvent. Covers both feature-scoped delegates and reusable delegates shared across screens."
paths:
  - "**/delegates/**/*Delegate.kt"
---

# Delegates

Use a delegate when a sub-flow (detail dialog, multi-step form, side panel, checkout step, client selector, "create X" mini-flow) manages its **own** loading / error / data state independently of the screen's main state.

Do not use a delegate for pure code organization — if a chunk of logic doesn't have its own state, extract it into private VM functions or a use case instead.

## Two placements — same shape

Delegates live in one of two locations depending on whether the sub-flow is used in one screen or in many:

| Placement | When | Example |
|---|---|---|
| `presentation/ux/<feature>/delegates/<Child>Delegate.kt` — **feature-scoped** | Used inside a single feature. The child event extends that feature's `<Screen>Event`. | `ProductDetailDelegate` inside `ux/catalog/delegates/` |
| `presentation/delegates/<name>/<Name>Delegate.kt` — **shared / reusable** | Injected into multiple host ViewModels (across features). The event is standalone, not tied to any one screen. | `ClientSelectorDelegate` (used by Catalog + Checkout), `CreateClientDelegate` (used by Catalog + Checkout + Settings) |

**The delegate's shape is identical in both cases.** Only three things differ:
1. Location on disk
2. Contract placement — feature-scoped puts the contract in a sibling `contracts/` folder; shared co-locates the contract in the same folder as the delegate
3. Event parenthood — feature-scoped extends the host `<Screen>Event`; shared is a standalone `sealed interface <Name>Event` with no parent

## Anatomy — feature-scoped

```kotlin
// presentation/ux/<feature>/delegates/<Child>Delegate.kt
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

Contract lives in `presentation/ux/<feature>/contracts/<Child>Contract.kt` — `sealed interface <Child>Event : <Screen>Event`.

## Anatomy — shared / reusable

```kotlin
// presentation/delegates/<name>/<Name>Delegate.kt
package <PKG_ROOT>.presentation.delegates.<name>

@ViewModelScoped
class <Name>Delegate @Inject constructor(
    private val repository: <Resource>Repository,
) {
    internal val uiState: StateFlow<<Name>UiState>
        field = MutableStateFlow(<Name>UiState())

    private lateinit var scope: CoroutineScope
    private var on<Something>: ((<Payload>) -> Unit)? = null

    // init MAY take extra callback params — that's how shared delegates
    // notify the host about cross-cutting events (e.g. "a client was reassigned")
    internal fun init(scope: CoroutineScope, on<Something>: (<Payload>) -> Unit = {}) {
        this.scope = scope
        this.on<Something> = on<Something>
    }

    internal fun handleEvent(event: <Name>Event) { ... }
}
```

Contract lives **in the same folder** as the delegate: `presentation/delegates/<name>/<Name>Contract.kt`. Its event is standalone:

```kotlin
// presentation/delegates/<name>/<Name>Contract.kt
sealed interface <Name>Event {
    data object OnDismissed : <Name>Event
    data class OnItemSelected(val id: String) : <Name>Event
}
```

Shared delegates commonly also expose non-`UiState` flows that the host collects — e.g. a `Flow<PagingData<T>>` search result:

```kotlin
internal val searchResults: Flow<PagingData<ClientSummary>> =
    uiState.map { it.query }.distinctUntilChanged().flatMapLatest { ... }.cachedIn(scope)
```

## Structure rules (both placements)

- **`@ViewModelScoped`** — one instance per host ViewModel; garbage-collected with the VM
- **`internal` visibility** — `uiState`, `init(scope, ...)`, `handleEvent(event)` all `internal`. Only ViewModels in the same module should touch these.
- **Own `StateFlow`** — do NOT nest `<Name>UiState` inside the host's `<Screen>UiState`. A nested state dataclass would mark the parent Compose-unstable.
- **`scope` passed via `init`** — the delegate borrows `viewModelScope` from the parent so cancellation propagates cleanly. Never create a `CoroutineScope()` inside a delegate.
- **`handleEvent` accepts only its own event type** — the host VM narrows before dispatching.
- **Shared delegates: extra `init` params are for host callbacks**, not for injecting dependencies. Dependencies come through the constructor. `init` param callbacks let the host react to cross-cutting events (e.g. `onClientCreated: (ClientSummary) -> Unit`).

## Wiring in a host ViewModel

**Feature-scoped delegate — child event extends screen event:**

```kotlin
@HiltViewModel
class <Screen>ViewModel @Inject constructor(
    private val <child>Delegate: <Child>Delegate,
) : ViewModel(), ViewModelNav by ViewModelNavImpl() {

    val <child>State: StateFlow<<Child>UiState> get() = <child>Delegate.uiState

    init { <child>Delegate.init(viewModelScope) }

    fun handleEvent(event: <Screen>Event) {
        when (event) {
            is <Child>Event -> <child>Delegate.handleEvent(event)
            // other <Screen>Event branches
        }
    }
}
```

**Shared delegate — event does NOT extend `<Screen>Event`; host must forward explicitly:**

```kotlin
@HiltViewModel
class CheckoutViewModel @Inject constructor(
    private val clientSelectorDelegate: ClientSelectorDelegate,
) : ViewModel(), ViewModelNav by ViewModelNavImpl() {

    val clientSelectorState: StateFlow<ClientSelectorUiState> get() = clientSelectorDelegate.uiState
    val searchResults: Flow<PagingData<ClientSummary>> get() = clientSelectorDelegate.searchResults

    init {
        clientSelectorDelegate.init(
            scope = viewModelScope,
            onClientReassigned = { refreshCart() },   // cross-cutting callback
        )
    }

    // Shared delegate events do NOT extend <Screen>Event, so the host exposes
    // its own event branch that forwards to the shared delegate:
    fun handleEvent(event: CheckoutEvent) {
        when (event) {
            is CheckoutEvent.ClientSelector -> clientSelectorDelegate.handleEvent(event.inner)
            // ... other CheckoutEvent branches
        }
    }
}
```

The host's `<Screen>Event` typically wraps the shared delegate's event: `data class ClientSelector(val inner: ClientSelectorEvent) : <Screen>Event`.

## Common misuses

- Delegate with no `StateFlow` of its own → this is not a delegate, it's a helper class or a use case
- Delegate injected into the screen composable → delegates are VM-scoped, only the VM sees them
- Delegate calling `viewModelScope` directly → use the `scope` passed via `init`
- Delegate exposed publicly → keep `uiState` / `handleEvent` `internal`
- Multiple delegates sharing state through the parent VM → merge them, or wire via a repository/flow
- Shared delegate placed under `presentation/ux/<feature>/delegates/` but used by more than one feature → move to `presentation/delegates/<name>/`
- Feature-only delegate placed under `presentation/delegates/` → move under the feature that owns it (`presentation/ux/<feature>/delegates/`)
- Shared delegate's contract stored in a sibling `contracts/` folder → co-locate `<Name>Contract.kt` with the delegate in `presentation/delegates/<name>/`
- Shared delegate whose event extends some `<Screen>Event` → decouple; a shared delegate's event is standalone so multiple hosts can wrap it
- Shared delegate constructor taking a callback lambda → put callbacks in `init(...)`, not the constructor; constructor is for Hilt-injected dependencies only
