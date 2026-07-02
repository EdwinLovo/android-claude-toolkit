---
name: contracts
description: UiState + Event definitions live in contracts/<Screen>Contract.kt; one file per screen/delegate group
paths:
  - "**/contracts/*Contract.kt"
---

# Contracts

Every screen has a `contracts/` folder. Never standalone `<Screen>UiState.kt` or `<Screen>Event.kt`, never an `events/` subfolder.

## Screen-level contract — `contracts/<Screen>Contract.kt`

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>.contracts

data class <Screen>UiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
    // one field per piece of screen state
)

interface <Screen>Event
```

- `<Screen>UiState` is a **plain `data class`** with **value-typed fields** and **default values** for every field.
- `<Screen>Event` is a **plain `interface`** (not sealed) so child events in delegate contracts can extend it.
  - **If no delegates exist and every event belongs to the screen**, use `sealed interface` instead — Kotlin's exhaustive `when` will catch missing branches.
- Events are `data object` (no params) or `data class` (with params):
  ```kotlin
  sealed interface <Screen>Event {
      data object OnPopBackStack : <Screen>Event
      data class OnItemClicked(val id: String) : <Screen>Event
  }
  ```

## Delegate contract — `contracts/<Child>Contract.kt`

One file per delegate group.

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

- `sealed interface` extending `<Screen>Event` — the parent VM's `handleEvent(<Screen>Event)` receives it, but each delegate handles its own subset.
- `<Child>UiState` gets its own defaults; the delegate owns its `MutableStateFlow`.

## Value-typed only

`<Screen>UiState` must **never** contain:
- `StateFlow<T>`, `Flow<T>`, `MutableStateFlow<T>` — collect flows in the entry composable instead
- `Flow<PagingData<T>>` — per Google, `PagingData` is mutable and cannot live in an immutable snapshot; expose as a top-level ViewModel property
- `LiveData<T>`, `MutableState<T>` — Compose-observable state belongs in `remember { }` inside a composable, not in UiState
- A nested `data class` whose fields are themselves states — flatten it, or move to a delegate contract

Value types allowed: `Boolean`, `Int`/`Long`/`Double`, `String`, `enum class` entries, plain `data class` instances, `List<T>`/`Set<T>`/`Map<K,V>` of value types, sealed hierarchies of value types.

## Common violations

- `class <Screen>UiState(...)` (missing `data`) → use `data class`
- Field without a default → every field defaults; empty state is the initial state
- `<Screen>Event.kt` sitting outside `contracts/` → move into `contracts/<Screen>Contract.kt`
- `data class` shared with both the ViewModel and a Room entity → separate them; presentation state is not persistence
