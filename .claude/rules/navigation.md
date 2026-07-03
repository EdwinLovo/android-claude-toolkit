---
name: navigation
description: Type-safe routes via kotlinx.serialization; ViewModelNav delegate for VM→NavController action flow; project primitives HandleNavigation + ObserveBooleanResult. Includes reference implementations for NavigationRoute, NavigationAction, ViewModelNav.
paths:
  - "**/navigation/**/*.kt"
  - "**/*Route.kt"
---

# Navigation

Single Activity + `NavHost` (Compose Navigation) with **type-safe routes** via `kotlinx.serialization`. ViewModels never touch `NavController` — they call methods on a `ViewModelNav` delegate that emits `NavigationAction` values, and a Compose helper (`HandleNavigation`) executes them against the real `NavController`.

## Routes

Every screen has a `<Screen>Route.kt` in its feature folder:

```kotlin
package <PKG_ROOT>.presentation.ux.<feature>

@Serializable
object <Screen>Route : NavigationRoute
```

With params:

```kotlin
@Serializable
data class <Screen>Route(
    val patientId: String,
    val visitDate: Long,
) : NavigationRoute
```

- **`object`** for parameterless routes
- **`data class`** for parameterized routes — one `val` per param, `Serializable` value types only (`String`, `Long`, `Int`, `Boolean`, or nested `data class` marked `@Serializable`)
- **`NavigationRoute`** is a project-level marker interface (see reference implementation below) — lets `ViewModelNav` accept only valid destinations

## Registration in `AppNavHost.kt`

Alphabetically sorted imports, one `composable<Route>` entry per screen:

```kotlin
import <PKG_ROOT>.presentation.ux.<feature>.<Screen>Route
import <PKG_ROOT>.presentation.ux.<feature>.<Screen>Screen

// inside NavHost { ... }
composable<<Screen>Route> { <Screen>Screen(navController) }
```

## Reference implementation

Copy these three files verbatim into `<PKG_ROOT>/presentation/ui/navigation/`. Every screen route, every VM navigation call, and every `AppNavHost` composable assumes they exist.

**`NavigationRoute.kt`** — marker interface:
```kotlin
package <PKG_ROOT>.presentation.ui.navigation

interface NavigationRoute
```

**`NavigationAction.kt`** — the sealed hierarchy of navigation intents produced by `ViewModelNavImpl` and consumed by `HandleNavigation`:
```kotlin
package <PKG_ROOT>.presentation.ui.navigation

import android.annotation.SuppressLint
import androidx.navigation.NavController

sealed interface NavigationAction {
    data class Navigate(private val route: NavigationRoute) : NavigationAction {
        fun invoke(navController: NavController, resetNavigate: (NavigationAction) -> Unit) {
            navController.navigate(route)
            resetNavigate(this)
        }
    }

    data class NavigateAndPop(
        private val route: NavigationRoute,
        private val popUpToRoute: NavigationRoute,
        private val inclusive: Boolean,
    ) : NavigationAction {
        fun invoke(navController: NavController, resetNavigate: (NavigationAction) -> Unit) {
            navController.navigate(route) {
                popUpTo(this@NavigateAndPop.popUpToRoute) { inclusive = this@NavigateAndPop.inclusive }
            }
            resetNavigate(this)
        }
    }

    data class PopBackStack(
        private val route: NavigationRoute?,
        private val inclusive: Boolean,
    ) : NavigationAction {
        fun invoke(navController: NavController, resetNavigate: (NavigationAction) -> Unit) {
            if (route != null) navController.popBackStack(route = route, inclusive = inclusive)
            else navController.popBackStack()
            resetNavigate(this)
        }
    }

    data class PopWithResult(
        private val resultValues: List<PopResultKeyValue>,
        private val route: NavigationRoute? = null,
        private val inclusive: Boolean = false,
        private val saveState: Boolean = false,
    ) : NavigationAction {
        fun invoke(navController: NavController, resetNavigate: (NavigationAction) -> Unit): Boolean {
            val destination = if (route != null) navController.getBackStackEntry(route)
                              else navController.previousBackStackEntry
            resultValues.forEach { destination?.savedStateHandle?.set(it.key, it.value) }
            val popped = if (route == null) navController.popBackStack()
                         else navController.popBackStack(route, inclusive = inclusive, saveState = saveState)
            resetNavigate(this)
            return popped
        }
    }

    data class PopWithResultUsingRouteName(
        private val resultValues: List<PopResultKeyValue>,
        private val routeClassName: String? = null,
        private val inclusive: Boolean = false,
        private val saveState: Boolean = false,
    ) : NavigationAction {
        @SuppressLint("RestrictedApi")
        fun invoke(navController: NavController, resetNavigate: (NavigationAction) -> Unit): Boolean {
            val destination = if (routeClassName != null) {
                navController.currentBackStack.value.lastOrNull { entry ->
                    entry.destination.route?.contains(routeClassName) == true
                }
            } else navController.previousBackStackEntry
            resultValues.forEach { destination?.savedStateHandle?.set(it.key, it.value) }
            val popped = if (routeClassName != null && destination != null) {
                navController.popBackStack(destination.destination.id, inclusive = inclusive, saveState = saveState)
            } else navController.popBackStack()
            resetNavigate(this)
            return popped
        }
    }

    data class PopBackStackAndNavigate(
        private val route: NavigationRoute,
        private val inclusive: Boolean = true,
    ) : NavigationAction {
        fun invoke(navController: NavController, resetNavigate: (NavigationAction) -> Unit): Boolean {
            navController.navigate(route) { popUpTo(0) { inclusive = this@PopBackStackAndNavigate.inclusive } }
            resetNavigate(this)
            return false
        }
    }
}

data class PopResultKeyValue(val key: String, val value: Any)

internal fun NavigationAction.navigate(
    navController: NavController,
    resetNavigate: (NavigationAction) -> Unit,
) {
    when (this) {
        is NavigationAction.Navigate -> invoke(navController, resetNavigate)
        is NavigationAction.NavigateAndPop -> invoke(navController, resetNavigate)
        is NavigationAction.PopBackStack -> invoke(navController, resetNavigate)
        is NavigationAction.PopBackStackAndNavigate -> invoke(navController, resetNavigate)
        is NavigationAction.PopWithResult -> invoke(navController, resetNavigate)
        is NavigationAction.PopWithResultUsingRouteName -> invoke(navController, resetNavigate)
    }
}
```

**`ViewModelNav.kt`** — the interface, impl, and Compose glue including the `ObserveBooleanResult` extension:
```kotlin
package <PKG_ROOT>.presentation.ui.navigation

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.navigation.NavController
import androidx.navigation.compose.currentBackStackEntryAsState
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

interface ViewModelNav {
    val navigationActionFlow: StateFlow<NavigationAction?>

    fun navigate(route: NavigationRoute)
    fun navigateAndPop(route: NavigationRoute, popUpToRoute: NavigationRoute, inclusive: Boolean)
    fun resetNavigate(action: NavigationAction)
    fun popBackStack(route: NavigationRoute? = null, inclusive: Boolean = false)
    fun popBackStackWithResult(
        resultValues: List<PopResultKeyValue>,
        route: NavigationRoute? = null,
        inclusive: Boolean = false,
    )
    fun popBackStackWithResultUsingRouteName(
        resultValues: List<PopResultKeyValue>,
        routeClassName: String?,
        inclusive: Boolean,
    )
    fun popBackStackAndNavigate(route: NavigationRoute, inclusive: Boolean = true)
}

internal class ViewModelNavImpl : ViewModelNav {
    private val _navigationActionFlow = MutableStateFlow<NavigationAction?>(null)
    override val navigationActionFlow = _navigationActionFlow.asStateFlow()

    override fun navigate(route: NavigationRoute) {
        _navigationActionFlow.compareAndSet(null, NavigationAction.Navigate(route))
    }
    override fun navigateAndPop(route: NavigationRoute, popUpToRoute: NavigationRoute, inclusive: Boolean) {
        _navigationActionFlow.compareAndSet(null, NavigationAction.NavigateAndPop(route, popUpToRoute, inclusive))
    }
    override fun resetNavigate(action: NavigationAction) {
        _navigationActionFlow.compareAndSet(action, null)
    }
    override fun popBackStack(route: NavigationRoute?, inclusive: Boolean) {
        _navigationActionFlow.compareAndSet(null, NavigationAction.PopBackStack(route, inclusive))
    }
    override fun popBackStackWithResult(
        resultValues: List<PopResultKeyValue>,
        route: NavigationRoute?,
        inclusive: Boolean,
    ) {
        _navigationActionFlow.compareAndSet(null, NavigationAction.PopWithResult(resultValues, route, inclusive))
    }
    override fun popBackStackWithResultUsingRouteName(
        resultValues: List<PopResultKeyValue>,
        routeClassName: String?,
        inclusive: Boolean,
    ) {
        _navigationActionFlow.compareAndSet(
            null,
            NavigationAction.PopWithResultUsingRouteName(resultValues, routeClassName, inclusive),
        )
    }
    override fun popBackStackAndNavigate(route: NavigationRoute, inclusive: Boolean) {
        _navigationActionFlow.compareAndSet(null, NavigationAction.PopBackStackAndNavigate(route, inclusive))
    }
}

@Composable
fun HandleNavigation(viewModelNav: ViewModelNav, navController: NavController?) {
    navController ?: return
    val action by viewModelNav.navigationActionFlow.collectAsStateWithLifecycle()
    LaunchedEffect(action) {
        action?.navigate(navController, viewModelNav::resetNavigate)
    }
}

@Composable
fun NavController.ObserveBooleanResult(key: String, onResult: () -> Unit) {
    val backStackEntry by currentBackStackEntryAsState()
    val result = backStackEntry?.savedStateHandle?.getStateFlow(key, false)?.collectAsStateWithLifecycle()
    LaunchedEffect(result?.value) {
        if (result?.value == true) {
            backStackEntry?.savedStateHandle?.set(key, false)   // consume once
            onResult()
        }
    }
}
```

## Navigating from a ViewModel

VMs implement `ViewModelNav by ViewModelNavImpl()` and call methods directly — they never construct `NavigationAction` values themselves and never import `NavController`.

| Method | Effect |
|---|---|
| `navigate(route)` | Push a new destination |
| `navigateAndPop(route, popUpToRoute, inclusive)` | Push, popping back to `popUpToRoute` first |
| `popBackStack(route = null, inclusive = false)` | Pop; optionally up to a specific destination |
| `popBackStackAndNavigate(route, inclusive = true)` | Pop everything and push a new destination (typical for logout / re-auth) |
| `popBackStackWithResult(resultValues, route?, inclusive)` | Pop while writing `resultValues` onto the destination's `savedStateHandle` |
| `popBackStackWithResultUsingRouteName(resultValues, routeClassName?, inclusive)` | Same, but target is identified by class-name substring (used when the exact route object isn't reachable, e.g. cross-module) |

```kotlin
class <Screen>ViewModel @Inject constructor(...) : ViewModel(), ViewModelNav by ViewModelNavImpl() {

    private fun onPatientClicked(id: String) = navigate(PatientDetailRoute(id))

    private fun onSaved() = popBackStackWithResult(
        resultValues = listOf(PopResultKeyValue(EDIT_RESULT_KEY, true)),
    )
}
```

The screen entry composable executes emitted actions once via:
```kotlin
HandleNavigation(viewModel, navController)
```

## Navigating back with a result

**Producer (Screen B — closing with a result):** call the typed method on `ViewModelNav`. Do NOT reach for `navController.previousBackStackEntry?.savedStateHandle?.set(...)` — that plumbing is inside `NavigationAction.PopWithResult.invoke` where it belongs.

```kotlin
private fun onSaved() {
    popBackStackWithResult(
        resultValues = listOf(PopResultKeyValue(EDIT_RESULT_KEY, true)),
    )
}

companion object {
    const val EDIT_RESULT_KEY = "edit_result"
}
```

**Consumer (Screen A — waiting for a boolean result):** use the project extension `NavController.ObserveBooleanResult` (defined in `ViewModelNav.kt` above). It reads the flag, consumes it, and re-arms itself automatically. Do NOT hand-roll `LaunchedEffect(navController.currentBackStackEntry) { savedStateHandle.getStateFlow(...).collect { ... } }`.

```kotlin
@Composable
fun <Screen>Screen(navController: NavController, viewModel: <Screen>ViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    navController.ObserveBooleanResult(EDIT_RESULT_KEY) {
        viewModel.handleEvent(<Screen>Event.OnEditResultReceived)
    }

    <Screen>ScreenContent(uiState = uiState, handleEvent = viewModel::handleEvent)
    HandleNavigation(viewModel, navController)
}
```

**Non-boolean results:** `ObserveBooleanResult` covers the "did the operation succeed?" case (the vast majority of result-passing needs). For other result types, add a sibling extension `NavController.ObserveResult<T>(key: String, onResult: (T) -> Unit)` alongside it in `ViewModelNav.kt` (or in an `ext/NavControllerExt.kt` per `rules/extensions.md`) — same shape, generic value type. Rolling raw `savedStateHandle.getStateFlow` at the call site is a violation; extract the helper on first use.

## Hard rules

- **All routes `@Serializable`** with `kotlinx.serialization`
- **`object` when no params, `data class` when params** — never both together
- **Route param types are serializable value types** — no `Parcelable`-only types
- **ViewModels call `ViewModelNav` methods; screens run `HandleNavigation`** — the VM never imports `NavController`, and never constructs `NavigationAction` values
- **Result-passing goes through `popBackStackWithResult` + `ObserveBooleanResult`** — direct `savedStateHandle` access is banned outside these primitives
- **Result keys are `const val` on the route/screen file** — e.g. `const val EDIT_RESULT_KEY = "edit_result"` in `<Screen>Route.kt` or the screen's `companion object`; never inlined magic strings

## Common violations

- Route defined as a raw `String` constant → migrate to a `@Serializable` type
- ViewModel injects `NavController` → replace with `ViewModelNav` delegate
- ViewModel constructing `NavigationAction.Navigate(route)` directly → call the typed method (`navigate(route)`) instead; actions are an internal detail of `ViewModelNavImpl`
- ViewModel calling `navController.previousBackStackEntry?.savedStateHandle?.set(...)` → use `popBackStackWithResult(listOf(PopResultKeyValue(KEY, value)))`
- Consumer hand-rolling `LaunchedEffect(navController.currentBackStackEntry) { navController.currentBackStackEntry?.savedStateHandle?.getStateFlow(KEY, ...).collect { ... } }` for a boolean → use `navController.ObserveBooleanResult(KEY) { ... }`
- Non-boolean result consumed via hand-rolled `savedStateHandle.getStateFlow` at the call site → extract a sibling helper (`ObserveResult<T>`) once, reuse everywhere
- Result key inlined as a magic string in two places → extract to a `const val` in the route file
- Result passed via a singleton or a shared VM → use `popBackStackWithResult` + `ObserveBooleanResult`; results are scoped to the navigation transition, and the back-stack entry cleans up automatically
