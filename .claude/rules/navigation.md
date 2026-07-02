---
name: navigation
description: Type-safe routes via kotlinx.serialization; ViewModelNav delegate; navigate-back-with-result via SavedStateHandle
paths:
  - "**/navigation/**/*.kt"
  - "**/*Route.kt"
---

# Navigation

Single Activity + `NavHost` (Compose Navigation) with **type-safe routes** via `kotlinx.serialization`.

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
- **`data class`** for parameterized routes — one `val` per param, `Serializable` value types only (`String`, `Long`, `Int`, `Boolean`, or `data class` marked `@Serializable`)
- **`NavigationRoute`** is a project-level marker interface (`presentation/ui/navigation/NavigationRoute.kt`) — helps generic navigation helpers accept only valid destinations

## Registration in `AppNavHost.kt`

Alphabetically sorted imports, one `composable<Route>` entry per screen:

```kotlin
import <PKG_ROOT>.presentation.ux.<feature>.<Screen>Route
import <PKG_ROOT>.presentation.ux.<feature>.<Screen>Screen

// inside NavHost { ... }
composable<<Screen>Route> { <Screen>Screen(navController) }
```

## Navigating from a ViewModel — `ViewModelNav`

ViewModels never touch `NavController`. They emit navigation actions through the `ViewModelNav` delegate:

```kotlin
class <Screen>ViewModel @Inject constructor(...)
    : ViewModel(), ViewModelNav by ViewModelNavImpl() {

    private fun onPatientClicked(id: String) {
        navigate(NavigationAction.To(PatientDetailRoute(id)))
    }

    private fun onSaved() {
        navigate(NavigationAction.Back)
    }
}
```

The screen entry composable pipes these to the `NavController`:

```kotlin
HandleNavigation(viewModel, navController)
```

## Navigating back with a result

Compose Navigation exposes the previous entry's `SavedStateHandle`. The pattern:

**Screen B (returning a result):**
```kotlin
navController.previousBackStackEntry
    ?.savedStateHandle
    ?.set(RESULT_KEY, resultValue)
navController.popBackStack()
```

**Screen A (waiting for the result):** collect once with `LaunchedEffect` keyed on the current back-stack entry:

```kotlin
LaunchedEffect(navController.currentBackStackEntry) {
    navController.currentBackStackEntry
        ?.savedStateHandle
        ?.getStateFlow<Result?>(RESULT_KEY, null)
        ?.filterNotNull()
        ?.collectLatest { result ->
            viewModel.handleEvent(<Screen>Event.OnResultReceived(result))
            navController.currentBackStackEntry
                ?.savedStateHandle
                ?.remove<Result>(RESULT_KEY)   // consume once
        }
}
```

Prefer this over a shared ViewModel or a singleton — the result is scoped to the navigation transition, and the back-stack entry cleans up automatically.

## Hard rules

- **All routes `@Serializable`** with `kotlinx.serialization`
- **`object` when no params, `data class` when params** — never both together
- **Route param types are serializable value types** — no `Parcelable`-only types
- **ViewModels emit `NavigationAction`; screens execute** — the VM never imports `NavController`
- **`popBackStack()` result set BEFORE the pop** — the `previousBackStackEntry` is only accessible while B is still on top
- **Result keys are file-level `private const val` in the route file** — e.g. `private const val RESULT_KEY = "edit_result"` in `<Screen>Route.kt`, exposed via a helper if consumed elsewhere

## Common violations

- Route defined as a raw `String` constant → migrate to a `@Serializable` type
- ViewModel injects `NavController` → replace with `ViewModelNav` delegate
- Result passed via a singleton or a shared VM → use `SavedStateHandle`
- Screen A recomputes the effect on every recomposition → key `LaunchedEffect` on `navController.currentBackStackEntry`
- Result key inlined as a magic string in two places → extract to a `const val` in the route file
