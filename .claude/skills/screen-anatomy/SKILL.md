---
name: screen-anatomy
description: Full templates for scaffolding a new MVVM feature screen in an Android/Compose project — Route, Contract, ViewModel, Screen, delegates, previews, and AppNavHost registration. Use when creating a new screen, planning feature implementation, or reviewing screen structure.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

# Screen Anatomy

Single source of truth for all new feature screen boilerplate. `/new-screen` references this skill for every template.

---

## Conventions

| Token | Meaning | Example |
|---|---|---|
| `<Screen>` | PascalCase screen name | `PatientProfile` |
| `<feature>` | Lowercase, no separators | `patientprofile` |
| `<PKG_ROOT>` | Kotlin package root | `com.example.myapp` |

**Package root:** `<PKG_ROOT>`
**Base path:** `app/src/main/java/<PKG_ROOT_PATH>/presentation/ux/<feature>/` (where `<PKG_ROOT_PATH>` is the package root with `.` → `/`)
**AppNavHost path:** `app/src/main/java/<PKG_ROOT_PATH>/presentation/ui/navigation/AppNavHost.kt`

**No BaseViewModel.** ViewModels extend `ViewModel()` directly and implement `ViewModelNav by ViewModelNavImpl()`.
**No BaseUiState.** Screen composables receive only `uiState` and `handleEvent` — no base state parameter.
**Contracts always used.** UiState + Event live in `contracts/<Screen>Contract.kt`, never as standalone files.

---

## File 1 — `<feature>/<Screen>Route.kt`

**Object route (no params):**
```kotlin
package <PKG_ROOT>.presentation.ux.<feature>

import <PKG_ROOT>.presentation.ui.navigation.NavigationRoute
import kotlinx.serialization.Serializable

@Serializable
object <Screen>Route : NavigationRoute
```

**Data class route (with params):**
```kotlin
package <PKG_ROOT>.presentation.ux.<feature>

import <PKG_ROOT>.presentation.ui.navigation.NavigationRoute
import kotlinx.serialization.Serializable

@Serializable
data class <Screen>Route(
    val <paramName>: <ParamType>,
) : NavigationRoute
```

One `val` per param, on separate lines.

---

## Files 2 & 3 — `contracts/` folder (always used)

Every screen has a `contracts/` subfolder. Never standalone `<Screen>Event.kt` / `<Screen>UiState.kt`, never an `events/` subfolder.

### File 2a — `contracts/<Screen>Contract.kt`

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>.contracts

data class <Screen>UiState(
    val <fieldName>: <Type> = <default>,
)

sealed interface <Screen>Event {
    data object <EventName> : <Screen>Event
    data class <EventName>(val <paramName>: <ParamType>) : <Screen>Event
}
```

- **`<Screen>Event` is always `sealed interface`**, whether or not the screen has delegates. Kotlin's exhaustive `when` catches missing branches.
- Delegate child events extend the sealed parent — permitted because sealed hierarchies are closed at the module boundary (see File 2b).
- `<Screen>UiState` has value-typed fields only — no `StateFlow`, no `Flow<PagingData<T>>`
- Every field has a default — empty state is initial state

### File 2b — `contracts/<Child>Contract.kt` (one per delegate group)

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>.contracts

data class <Child>UiState(
    val isLoading: Boolean = false,
    // add state fields
)

sealed interface <Child>Event : <Screen>Event {
    data object OnDismissed : <Child>Event
    data class <EventName>(val <paramName>: <ParamType>) : <Child>Event
}
```

`sealed interface` extending `<Screen>Event`. Omit if the screen has no delegates.

---

## File 4 — `<feature>/<Screen>ViewModel.kt`

`ViewModel()` + `ViewModelNav by ViewModelNavImpl()`. Explicit backing field. Always import `<Screen>UiState` and `<Screen>Event` from `contracts/`.

**No delegates:**
```kotlin
package <PKG_ROOT>.presentation.ux.<feature>

import androidx.lifecycle.ViewModel
import <PKG_ROOT>.presentation.ui.navigation.ViewModelNav
import <PKG_ROOT>.presentation.ui.navigation.ViewModelNavImpl
import <PKG_ROOT>.presentation.utils.ext.reduce
import <PKG_ROOT>.presentation.ux.<feature>.contracts.<Screen>Event
import <PKG_ROOT>.presentation.ux.<feature>.contracts.<Screen>UiState
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow

@HiltViewModel
class <Screen>ViewModel @Inject constructor() : ViewModel(), ViewModelNav by ViewModelNavImpl() {

    val uiState: StateFlow<<Screen>UiState>
        field = MutableStateFlow(<Screen>UiState())

    fun handleEvent(event: <Screen>Event) {
        when (event) {
            is <Screen>Event.<EventName> -> { /* TODO */ }
        }
    }
}
```

**With delegates** (also import `<Child>Event`):
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

    fun handleEvent(event: <Screen>Event) {
        when (event) {
            is <Child>Event -> <child>Delegate.handleEvent(event)
        }
    }
}
```

**`when` branch rules:**
- `data object` event → no `is` prefix: `<Screen>Event.OnSave -> { }`
- `data class` event → use `is`: `is <Screen>Event.OnOpen -> { }`
- Child delegate group → one branch, always `is`: `is <ChildName>Event -> <child>Delegate.handleEvent(event)`
- `when` is a statement (Unit) — no `else` needed even with a non-sealed parent interface

**Paging note** — if the screen loads paginated data, expose `Flow<PagingData<T>>` as a top-level property, not inside `<Screen>UiState`:
```kotlin
val items: Flow<PagingData<Item>> = uiState
    .map { it.searchQuery }
    .distinctUntilChanged()
    .debounce { if (it.isBlank()) 0L else DEBOUNCE_TIME }
    .flatMapLatest { query -> repository.getItems(query) }
    .cachedIn(viewModelScope)
```

---

## File 5 — `<feature>/<Screen>Screen.kt`

Two-composable split: public entry + `private` stateless content. Import only `<Screen>Event` / `<Screen>UiState` from `contracts/`.

**`<Screen>ScreenContent` is the only `private @Composable` allowed** in a non-preview file. Any other reusable piece belongs in its own file under `components/`.

**No delegates:**
```kotlin
package <PKG_ROOT>.presentation.ux.<feature>

import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.navigation.NavController
import <PKG_ROOT>.presentation.ui.navigation.HandleNavigation
import <PKG_ROOT>.presentation.ux.<feature>.contracts.<Screen>Event
import <PKG_ROOT>.presentation.ux.<feature>.contracts.<Screen>UiState

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
    Text(text = "<Screen>")
}
```

**With delegates** — collect each delegate state separately:
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
```

Child event types are not imported in the Screen — only the parent `<Screen>Event`.

If the screen has paging flows, collect them and pass as parameters:
```kotlin
val items = viewModel.items.collectAsLazyPagingItems()
<Screen>ScreenContent(uiState = uiState, items = items, handleEvent = viewModel::handleEvent)
```

**Preview at the bottom of the same file:**
```kotlin
private class <Screen>UiStateProvider : PreviewParameterProvider<<Screen>UiState> {
    override val values = sequenceOf(<Screen>UiState())
}

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

---

## Delegate template — `delegates/<Child>Delegate.kt`

For each `@ViewModelScoped` sub-flow:

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>.delegates

import <PKG_ROOT>.presentation.ux.<feature>.contracts.<Child>Event
import <PKG_ROOT>.presentation.ux.<feature>.contracts.<Child>UiState
import <PKG_ROOT>.presentation.utils.ext.reduce
import dagger.hilt.android.scopes.ViewModelScoped
import javax.inject.Inject
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow

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
        }
    }
}
```

---

## AppNavHost update rules

File: `app/src/main/java/<PKG_ROOT_PATH>/presentation/ui/navigation/AppNavHost.kt`

**Add 2 imports** — alphabetically among existing `<PKG_ROOT>.presentation.ux.*` imports:
```kotlin
import <PKG_ROOT>.presentation.ux.<feature>.<Screen>Route
import <PKG_ROOT>.presentation.ux.<feature>.<Screen>Screen
```

**Add 1 composable entry** — append inside the `NavHost` block before the closing `}`:
```kotlin
// <Screen>
composable<<Screen>Route> { <Screen>Screen(navController) }
```

Match the existing indentation style (usually a single tab).
