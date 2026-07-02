---
name: contracts
description: UiState + Event definitions — three variants (screen, feature-delegate child, shared-delegate) with distinct file locations
paths:
  - "**/contracts/*Contract.kt"
  - "**/presentation/delegates/**/*Contract.kt"
---

# Contracts

A contract file pairs a `<Name>UiState` (data class) with a `<Name>Event` (sealed interface). Three variants exist, distinguished by where the file lives and whether the event extends a parent.

| Variant | File location | Event pattern |
|---|---|---|
| **Screen** | `presentation/ux/<feature>/contracts/<Screen>Contract.kt` | Standalone `sealed interface <Screen>Event` |
| **Feature delegate child** | `presentation/ux/<feature>/contracts/<Child>Contract.kt` (sibling of the screen contract) | `sealed interface <Child>Event : <Screen>Event` — extends the screen event |
| **Shared delegate** | `presentation/delegates/<name>/<Name>Contract.kt` (co-located with the delegate, NOT in a `contracts/` folder) | Standalone `sealed interface <Name>Event` — no parent; host wraps it |

Never standalone `<Name>UiState.kt` / `<Name>Event.kt` files anywhere. Never an `events/` subfolder.

## Variant 1 — screen contract

`presentation/ux/<feature>/contracts/<Screen>Contract.kt`:

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>.contracts

data class <Screen>UiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    // one field per piece of screen state
)

sealed interface <Screen>Event {
    data object OnPopBackStack : <Screen>Event
    data class OnItemClicked(val id: String) : <Screen>Event
}
```

- `<Screen>UiState` is a **plain `data class`** with **value-typed fields** and **default values** for every field.
- `<Screen>Event` is **always `sealed interface`**, whether or not the screen has delegates. Kotlin's exhaustive `when` catches missing branches in `handleEvent`.
- Child events in delegate contracts (variant 2) extend this sealed parent — Kotlin permits it because sealed hierarchies are closed at the module boundary, not the file boundary.
- Events are `data object` (no params) or `data class` (with params).

## Variant 2 — feature delegate child contract

`presentation/ux/<feature>/contracts/<Child>Contract.kt` — one file per delegate group. Sits alongside `<Screen>Contract.kt` in the same `contracts/` folder.

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>.contracts

data class <Child>UiState(
    val isLoading: Boolean = false,
    // sub-flow fields
)

sealed interface <Child>Event : <Screen>Event {
    data object OnDismissed : <Child>Event
    data class OnConfirm(val value: String) : <Child>Event
}
```

- `sealed interface` **extending `<Screen>Event`** — the parent VM's `handleEvent(<Screen>Event)` receives it, and dispatches with `is <Child>Event -> <child>Delegate.handleEvent(event)`.
- `<Child>UiState` gets its own defaults; the delegate owns its `MutableStateFlow` (see `rules/delegates.md`).

## Variant 3 — shared delegate contract

`presentation/delegates/<name>/<Name>Contract.kt` — **co-located with the delegate**, not in a `contracts/` folder. The delegate is reused across multiple ViewModels (e.g. `ClientSelectorDelegate` used by both Catalog and Checkout), so the contract cannot bind to any one screen's event.

```kotlin
package <PKG_ROOT>.presentation.delegates.<name>

data class <Name>UiState(
    val isLoading: Boolean = false,
    val query: String = "",
    // shared sub-flow fields
)

sealed interface <Name>Event {
    data object OnDismissed : <Name>Event
    data class OnQueryChanged(val query: String) : <Name>Event
    data class OnItemSelected(val id: String) : <Name>Event
}
```

- `sealed interface` — **standalone**, no parent event. Each host ViewModel wraps it inside its own `<Screen>Event`: `data class ClientSelector(val inner: ClientSelectorEvent) : CheckoutEvent`.
- Contract lives **beside the delegate**, not in a sibling `contracts/` folder. This is the one place in the app where a contract does not live under `contracts/`.
- See `rules/delegates.md` for the shared-delegate wiring pattern (host wraps event, host provides `init` callbacks).

## Value-typed only (all variants)

`<Name>UiState` must **never** contain:
- `StateFlow<T>`, `Flow<T>`, `MutableStateFlow<T>` — collect flows in the entry composable instead
- `Flow<PagingData<T>>` — per Google, `PagingData` is mutable and cannot live in an immutable snapshot; expose as a top-level ViewModel/delegate property
- `LiveData<T>`, `MutableState<T>` — Compose-observable state belongs in `remember { }` inside a composable, not in UiState
- A nested `data class` whose fields are themselves states — flatten it, or move to a delegate contract

Value types allowed: `Boolean`, `Int`/`Long`/`Double`, `String`, `enum class` entries, plain `data class` instances, `List<T>`/`Set<T>`/`Map<K,V>` of value types, sealed hierarchies of value types.

## Common violations

- `class <Name>UiState(...)` (missing `data`) → use `data class`
- Field without a default → every field defaults; empty state is the initial state
- `interface <Name>Event` (missing `sealed`) → always `sealed interface`
- Standalone `<Screen>Event.kt` or `<Screen>UiState.kt` file → move into `contracts/<Screen>Contract.kt`
- `events/` subfolder → contracts only; delete the `events/` folder
- Shared delegate contract placed in a `contracts/` subfolder (`presentation/delegates/<name>/contracts/<Name>Contract.kt`) → co-locate with the delegate (`presentation/delegates/<name>/<Name>Contract.kt`)
- Shared delegate event extending some `<Screen>Event` — locks the delegate to one screen → make it standalone; hosts wrap it inside their own event
- Feature delegate child event NOT extending the parent `<Screen>Event` → add `: <Screen>Event`; otherwise the host VM's exhaustive `when` won't route to it
- `data class` shared with both the ViewModel and a Room entity → separate them; presentation state is not persistence
