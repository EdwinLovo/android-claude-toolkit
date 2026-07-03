---
name: navigation-primitives
description: First-time setup for the navigation layer — reference implementations for NavigationRoute, NavigationAction, and ViewModelNav (interface + impl + HandleNavigation + ObserveBooleanResult). Invoke when scaffolding navigation in a new project, when a rule references these types and they don't exist yet, or when adding a sibling result-observer extension.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# Navigation primitives

Reference implementations for the three files that back the navigation pattern documented in `.claude/rules/navigation.md`. Copy them verbatim into `<PKG_ROOT>/presentation/ui/navigation/` when setting up a new project. Every screen route, every ViewModel navigation call, and every `AppNavHost` composable assumes they exist.

For the conventions that use these types (route declarations, `AppNavHost` registration, VM API surface, result-passing patterns, common violations), see `.claude/rules/navigation.md`.

---

## File 1 — `NavigationRoute.kt`

Marker interface for anything a `NavController` can navigate to. Screen routes implement it.

```kotlin
package <PKG_ROOT>.presentation.ui.navigation

interface NavigationRoute
```

---

## File 2 — `NavigationAction.kt`

Sealed hierarchy of navigation intents produced internally by `ViewModelNavImpl` and consumed by `HandleNavigation`. VMs never construct these directly — they call the typed methods on `ViewModelNav` (see File 3).

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

---

## File 3 — `ViewModelNav.kt`

The `ViewModelNav` interface + `ViewModelNavImpl` internal delegate + `HandleNavigation` composable + `NavController.ObserveBooleanResult` extension. This file holds the whole VM-to-NavController bridge.

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

---

## Adding a sibling result-observer extension

For a non-boolean result type, add alongside `ObserveBooleanResult` in this same file (or in an `ext/NavControllerExt.kt` per `rules/extensions.md`). Same shape, generic value type:

```kotlin
@Composable
fun <T : Any> NavController.ObserveResult(key: String, onResult: (T) -> Unit) {
    val backStackEntry by currentBackStackEntryAsState()
    val result = backStackEntry?.savedStateHandle?.getStateFlow<T?>(key, null)?.collectAsStateWithLifecycle()
    LaunchedEffect(result?.value) {
        val value = result?.value ?: return@LaunchedEffect
        backStackEntry?.savedStateHandle?.remove<T>(key)   // consume once
        onResult(value)
    }
}
```

Rolling raw `savedStateHandle.getStateFlow` at the call site is a violation of `rules/navigation.md` — extract the helper on first use.

---

## Verification after copying

Once these three files exist in a project:

- `presentation/ui/navigation/NavigationRoute.kt` — interface only
- `presentation/ui/navigation/NavigationAction.kt` — sealed hierarchy + `PopResultKeyValue` + package-internal `navigate` extension
- `presentation/ui/navigation/ViewModelNav.kt` — interface + impl + two composables

VMs can then `class <Screen>ViewModel @Inject constructor(...) : ViewModel(), ViewModelNav by ViewModelNavImpl()` and call `navigate(<Screen>Route(...))`, `popBackStackWithResult(...)`, etc. See `rules/navigation.md` for the full API surface and result-passing patterns.
