---
name: navigation
description: Type-safe routes via kotlinx.serialization; ViewModelNav delegate for VM→NavController action flow; project primitives HandleNavigation + ObserveBooleanResult (reference implementations live in the navigation-primitives skill)
paths:
  - "**/navigation/**/*.kt"
  - "**/*Route.kt"
---

# Navigation

Single Activity + `NavHost` (Compose Navigation) with **type-safe routes** via `kotlinx.serialization`. ViewModels never touch `NavController` — they call methods on a `ViewModelNav` delegate that emits `NavigationAction` values, and a Compose helper (`HandleNavigation`) executes them against the real `NavController`.

**Setting up navigation in a new project:** invoke the `navigation-primitives` skill (`.claude/skills/navigation-primitives/SKILL.md`). It ships the three reference files this rule assumes exist: `NavigationRoute.kt`, `NavigationAction.kt`, `ViewModelNav.kt` (which also contains `HandleNavigation` and `NavController.ObserveBooleanResult`).

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
- **`NavigationRoute`** is a project-level marker interface (defined by the `navigation-primitives` skill) — lets `ViewModelNav` accept only valid destinations

## Registration in `AppNavHost.kt`

Alphabetically sorted imports, one `composable<Route>` entry per screen:

```kotlin
import <PKG_ROOT>.presentation.ux.<feature>.<Screen>Route
import <PKG_ROOT>.presentation.ux.<feature>.<Screen>Screen

// inside NavHost { ... }
composable<<Screen>Route> { <Screen>Screen(navController) }
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
| `popBackStackWithResultUsingRouteName(resultValues, routeClassName?, inclusive)` | Same, but target is identified by class-name substring (for cases where the exact route object isn't reachable, e.g. cross-module) |

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

**Producer (Screen B — closing with a result):** call the typed method on `ViewModelNav`. Do NOT reach for `navController.previousBackStackEntry?.savedStateHandle?.set(...)` — that plumbing lives inside `NavigationAction.PopWithResult.invoke` where it belongs.

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

**Consumer (Screen A — waiting for a boolean result):** use the project extension `NavController.ObserveBooleanResult` (shipped by the `navigation-primitives` skill). It reads the flag, consumes it, and re-arms itself automatically. Do NOT hand-roll `LaunchedEffect(navController.currentBackStackEntry) { savedStateHandle.getStateFlow(...).collect { ... } }`.

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

**Non-boolean results:** `ObserveBooleanResult` covers the "did the operation succeed?" case (the vast majority). For other result types, add a sibling `NavController.ObserveResult<T>(key, onResult)` — the `navigation-primitives` skill includes a template you can drop in. Rolling raw `savedStateHandle.getStateFlow` at the call site is a violation; extract the helper on first use.

## Hard rules

- **All routes `@Serializable`** with `kotlinx.serialization`
- **`object` when no params, `data class` when params** — never both together
- **Route param types are serializable value types** — no `Parcelable`-only types
- **ViewModels call `ViewModelNav` methods; screens run `HandleNavigation`** — the VM never imports `NavController` and never constructs `NavigationAction` values
- **Result-passing goes through `popBackStackWithResult` + `ObserveBooleanResult`** — direct `savedStateHandle` access is banned outside these primitives
- **Result keys are `const val` on the route/screen file** — e.g. `const val EDIT_RESULT_KEY = "edit_result"` in `<Screen>Route.kt` or the screen's `companion object`; never inlined magic strings

## Common violations

- Route defined as a raw `String` constant → migrate to a `@Serializable` type
- ViewModel injects `NavController` → replace with `ViewModelNav` delegate
- ViewModel constructing `NavigationAction.Navigate(route)` directly → call the typed method (`navigate(route)`); actions are an internal detail of `ViewModelNavImpl`
- ViewModel calling `navController.previousBackStackEntry?.savedStateHandle?.set(...)` → use `popBackStackWithResult(listOf(PopResultKeyValue(KEY, value)))`
- Consumer hand-rolling `LaunchedEffect(navController.currentBackStackEntry) { savedStateHandle.getStateFlow(KEY, ...).collect { ... } }` for a boolean → use `navController.ObserveBooleanResult(KEY) { ... }`
- Non-boolean result consumed via hand-rolled `savedStateHandle.getStateFlow` at the call site → extract a sibling helper (`ObserveResult<T>`) once, reuse everywhere (template in the `navigation-primitives` skill)
- Result key inlined as a magic string in two places → extract to a `const val` in the route file
- Result passed via a singleton or a shared VM → use `popBackStackWithResult` + `ObserveBooleanResult`; results are scoped to the navigation transition, and the back-stack entry cleans up automatically
- `NavigationRoute`, `ViewModelNav`, `NavigationAction`, or `HandleNavigation` referenced but the files don't exist in `<PKG_ROOT>/presentation/ui/navigation/` → invoke the `navigation-primitives` skill
