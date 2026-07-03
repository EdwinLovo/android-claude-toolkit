# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

> Template file from `android-claude-toolkit`. Replace `<PLACEHOLDER>` tokens with your project values (see toolkit README). Delete this quote block when done.

## Project Overview

**<PROJECT_NAME>** — an Android app built with Jetpack Compose and Material 3. Single-module clean architecture.

- **App ID / Namespace:** `<APP_ID>`
- **Package root:** `<PKG_ROOT>`
- **UI:** Jetpack Compose + Material 3
- **DI:** Hilt
- **Networking:** <NET_LIB>

## Conventions

**Never commit automatically.** Only run `git commit` when the user explicitly says "commit" or asks you to commit. Finish all code changes and stop — do not stage, do not commit. Even after fixing a hook failure, do not auto-commit. Wait for the user to ask.

**Commit messages** follow this format: `<TICKET_PREFIX>: short description`. Extract the ticket number from the current branch name (e.g. `feature/<TICKET_PREFIX>-*` or `bugfix/<TICKET_PREFIX>-*`).

**Theme usage** — always source design values from the project theme object, never from `MaterialTheme` (with one exception) and never from hardcoded literals:

| Need | Use |
|---|---|
| Colors | `<THEME>.colors.<token>` |
| Spacing | `<THEME>.spacing.<size>` |
| Padding | `<THEME>.padding.<size>` |
| Corner radius | `<THEME>.cornerRadius.<size>` |
| Icon size | `<THEME>.iconSize.<size>` |
| Border width | `<THEME>.borderWidth.<size>` |
| Typography | `MaterialTheme.typography.<style>` ← only valid `MaterialTheme` use |

Never use `MaterialTheme.colorScheme`, `MaterialTheme.shapes`, or hardcoded `Color(...)` in UI code. If a `.dp` / `Int` / `Float` value is truly one-off and has no token equivalent, declare it as a private file-level constant at the top of the file rather than inlining the literal.

**ViewModel constants** — all `const val` values specific to a ViewModel live in a `companion object` at the **bottom** of the class body, after all methods. Never use file-level `private const val` in ViewModel files, and never inline magic literals in the class body. Shared constants used across multiple classes stay in `presentation/utils/`.

```kotlin
class ExampleViewModel @Inject constructor() : ViewModel() {
    // fields, init, handleEvent, private fns ...

    companion object {
        private const val DEBOUNCE_MS = 300L
        private const val MAX_RETRIES = 3
    }
}
```

**String resources** — all user-visible strings in composables come from `strings.xml` via `stringResource(R.string.xxx)`. Never hardcode string literals in UI code. Enum labels that appear in the UI use a `@param:StringRes val labelRes: Int` property.

**Icons** — all icons come from `<ICONS>` (`presentation/utils/<ICONS>.kt`). Never use Material Icons (`Icons.*`). Use `painterResource(<ICONS>.X)` with `Icon()` for tinted vector icons, or `Image()` with `painterResource(<ICONS>.X)` for full-color raster images. To add a new icon, add a Vector Drawable XML to `res/drawable/` and register it in `<ICONS>`.

**Scaffold** — use `<SCAFFOLD>` (`presentation/ui/components/scaffold/<SCAFFOLD>.kt`) instead of Material3's `Scaffold` directly. Never nest two Scaffolds.

**Dialogs** — every dialog uses `<DIALOG>` (`presentation/ui/components/dialogs/<DIALOG>.kt`) as its outermost wrapper. It provides consistent adaptive sizing, shape, background color, and elevation. Never re-declare `Dialog`/`Surface`/size logic directly in a dialog composable. For confirmation or error dialogs, use the `AlertDialog` variant rather than building a custom body.

**Component files** — each Compose component lives in its own file as `internal` (preferred) or `public`, with its own preview. Do **not** keep `private @Composable` helpers inside another component's file — extract them, mark `internal`, and give them their own preview. The only `private @Composable` allowed in a component file are:
- `*Preview()` functions at the bottom of the file
- The stateless `<Screen>ScreenContent` companion that sits in `<Screen>Screen.kt` next to the entry composable

Placement:
- Feature-specific → `presentation/ux/<feature>/components/<ComponentName>.kt`
- Globally reused → `presentation/ui/components/<category>/<ComponentName>.kt` (categories: `buttons/`, `dialogs/`, `feedback/`, `footer/`, `inputs/`, `rows/`, `scaffold/`, `text/`, etc.)

**Errors & UiText** — repository errors flow through a global `ErrorEventBus` (`presentation/utils/ErrorEventBus.kt`). ViewModels and delegates call `ErrorEventBus.send(message.toUiText())` inside `.onError { _, message -> ... }` blocks. `MainScreen` collects the bus and renders an `AlertDialog` — screens never need their own error-dialog plumbing. `UiText` (`presentation/utils/UiText.kt`) is the sealed type carried by the bus (`StringResource` / `DynamicString`); convert API error strings with `String?.toUiText()`.

**UiState pattern** — `<Screen>UiState` is a plain `data class` with value-typed fields and default values, defined in `contracts/<Screen>Contract.kt`. The ViewModel declares `uiState` using an explicit backing field (enabled via `-Xexplicit-backing-fields`): the public type is `StateFlow<T>` but inside the class it behaves as `MutableStateFlow<T>` — no `_uiState` prefix needed. Updates use the `reduce` extension (`presentation/utils/ext/StateFlowExt.kt`) for receiver-style copies: `uiState.reduce { copy(field = value) }`. `handleEvent` dispatches to private setter functions that call `reduce`. The Screen entry composable collects the state once and passes the plain value to the stateless content composable.

```kotlin
// UiState — plain data class, value-typed fields with defaults
data class ExampleUiState(
    val isLoading: Boolean = false,
    val items: List<Item> = emptyList(),
)

// ViewModel — explicit backing field
@HiltViewModel
class ExampleViewModel @Inject constructor() : ViewModel(), ViewModelNav by ViewModelNavImpl() {
    val uiState: StateFlow<ExampleUiState>
        field = MutableStateFlow(ExampleUiState())

    fun handleEvent(event: ExampleEvent) {
        when (event) {
            is ExampleEvent.OnLoad -> loadItems()
        }
    }

    private fun loadItems() {
        uiState.reduce { copy(isLoading = true) }
    }
}

// Screen — collect once in entry composable, pass plain value to content
@Composable
fun ExampleScreen(navController: NavController, viewModel: ExampleViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    ExampleScreenContent(uiState = uiState, handleEvent = viewModel::handleEvent)

    HandleNavigation(viewModel, navController)
}
```

**Paging flows** — `Flow<PagingData<T>>` must **not** be placed inside UiState (per Google's guidance: `PagingData` is mutable and cannot live in an immutable state snapshot). Expose them as top-level ViewModel properties derived from the public `uiState` field:

```kotlin
val items: Flow<PagingData<Item>> = uiState
    .map { it.searchQuery }
    .distinctUntilChanged()
    .debounce { if (it.isBlank()) 0L else DEBOUNCE_TIME }
    .flatMapLatest { query -> repository.getItems(query) }
    .cachedIn(viewModelScope)
```

Collect paging flows in the entry composable via `collectAsLazyPagingItems()` and pass as parameters to the content composable alongside `uiState`.

**Previews** — every composable and screen must have a Compose preview:
- Use `<PREVIEW>` for components (wraps content, default size) and its screen variant for full screens (e.g. 1280×800dp tablet) — both render light and dark
- Wrap content in a preview container that applies the theme
- Place the preview at the **bottom of the same file** as the composable, marked `private`
- Naming: `fun <ComponentName>Preview()` / `fun <ScreenName>ContentPreview()`
- Use `@PreviewParameter` + `PreviewParameterProvider` for components with meaningful state variants; trivial providers (enum entries) stay in the same file
- Complex providers (screen-level UiState construction) go in a separate `<Screen>PreviewParams.kt` sibling file

## Architecture

Single-module project (`app`) using Single Activity + Jetpack Compose with clean architecture.

**Entry point:** `presentation/ux/main/MainActivity.kt`

**Package structure** (`<PKG_ROOT>`):
```
├── domain/
│   ├── model/
│   │   ├── response/
│   │   │   └── <category>/     ← response models grouped by feature
│   │   └── uimodel/            ← display-oriented models used by use cases
│   ├── repository/             ← interfaces only
│   └── usecase/
├── data/
│   ├── di/                     ← ApiModule, DataSourceModule, RepositoryModule
│   ├── mappers/
│   │   └── <category>/         ← network → domain mapper extension functions
│   ├── paging/                 ← PagingSource implementations
│   ├── remote/
│   │   ├── datasource/         ← RemoteDataSource interface + impl
│   │   └── <NET_LIB folder>/   ← queries / API definitions grouped by feature
│   └── repository/             ← implementations + BaseRepository
└── presentation/
    ├── ui/
    │   ├── navigation/         ← NavigationRoute, NavigationAction, ViewModelNav, AppNavHost
    │   ├── components/         ← shared Compose components
    │   └── theme/              ← Material3 theme, colors, typography
    └── ux/
        ├── <feature>/          ← one folder per screen (see Screen Anatomy)
        └── main/               ← MainActivity, MainScreen, MainViewModel, MainUiState
```

**Dependency versions** are centralized in `gradle/libs.versions.toml`. Always add new dependencies there rather than hardcoding versions in `build.gradle.kts`.

**Data layer patterns:**

- `RemoteDataSource` is a call factory: methods are non-suspend and return call objects for query building only. The single impl class imports the network client and constructs input types. Repositories execute single calls via `BaseRepository.safeCallSuspend(call, mapper)`. Paging sources execute via `BasePagingSource.toLoadResult { }`. Mapper extension functions in `data/mappers/<feature>/` handle all domain mapping.
- Response models live in `domain/model/response/<category>/`. Plain data classes, no framework dependencies.
- Mapper extension functions live in `data/mappers/<category>/` and convert network types to domain models. When a mapper transforms only between domain models (no network types on either side), it lives in `domain/model/<area>/<Name>Mappers.kt` next to the target model. `data/mappers/` is reserved for network↔domain bridging.
- DI is split across three modules in `data/di/`: `ApiModule` (network client), `DataSourceModule` (RemoteDataSource binding), `RepositoryModule` (repository bindings).

**Presentation extensions** — reusable extension functions on standard types (e.g. `Double.toDisplayPrice()`) live in `presentation/utils/ext/` as `<Type>Ext.kt` files (e.g. `StringExt.kt`).

**Testing:**
- Unit tests → `app/src/test/`
- Instrumented tests → `app/src/androidTest/`

---

## Screen Anatomy

> Full templates are in `.claude/skills/screen-anatomy/SKILL.md`.
> Use the `/new-screen` command to scaffold a new screen interactively.

Each feature screen lives in `presentation/ux/<feature>/` and consists of these files:

```
ux/<feature>/
├── <Screen>Route.kt        ← @Serializable object or data class : NavigationRoute
├── <Screen>ViewModel.kt    ← @HiltViewModel, ViewModel() + ViewModelNav
└── <Screen>Screen.kt       ← two-composable split: entry + stateless content

ux/<feature>/contracts/     ← always used; UiState + Event co-located per group
├── <Screen>Contract.kt     ← <Screen>UiState + interface <Screen>Event (screen-level)
└── <Child>Contract.kt      ← <Child>UiState + sealed interface <Child>Event : <Screen>Event

ux/<feature>/delegates/     ← only when screen has @ViewModelScoped sub-flows
└── <Child>Delegate.kt      ← owns StateFlow<<Child>UiState>, init(scope), handleEvent

ux/<feature>/utils/         ← feature-scoped enums and extension functions
├── <Type>.kt               ← enum class
└── <Type>Ext.kt            ← extension functions on that type
```

**No BaseViewModel.** ViewModels extend `ViewModel()` directly and implement `ViewModelNav by ViewModelNavImpl()`.
**No BaseUiState.** The entry composable collects state and delegates to a stateless content composable.
**Contracts pattern:** always use `contracts/`; never standalone `<Screen>UiState.kt` / `<Screen>Event.kt` files, never an `events/` subfolder.
**Delegates:** add a `delegates/` subfolder only when sub-flows (e.g. detail dialogs, multi-step forms) each manage their own loading/error/data state.

---

## Skills & Commands

| | What it does |
|---|---|
| `.claude/skills/screen-anatomy/` | Templates for creating a new MVVM screen (Route, Contract, ViewModel, Screen, Preview, delegates) |
| `.claude/skills/data-feature/` | Templates for a networking-backed data feature (query file, model, mapper, data source, repository, DI) |
| `.claude/skills/navigation-primitives/` | First-time setup for the navigation layer — reference files for `NavigationRoute`, `NavigationAction`, `ViewModelNav` (interface + impl + `HandleNavigation` + `ObserveBooleanResult`) |
| `.claude/skills/audit-branch/` | Audit workflow — invoked by `/audit-branch`; runs in a **forked context** (`context: fork`) so findings aren't biased by the current conversation |
| `.claude/rules/` | Path-scoped conventions auto-loaded when editing matching files (ViewModels, contracts, delegates, screens, previews, mappers, DI, ...) |
| `/new-screen` | Scaffold a new screen (Route, Contract, ViewModel, Screen + AppNavHost registration) |
| `/new-data-feature` | Scaffold a new data-layer feature (network op, model, mapper, repository, DI) |
| `/audit-branch` | Audit changed source files against project standards; runs in an isolated sub-agent (via the `audit-branch` skill) so the audit reads the branch cold |
| `/pr-template` | Generate a PR against `dev` branch |
| `/add-tests` | Add/update unit tests for ViewModels, repositories, mappers, use cases |
| `/update-docs` | Update CLAUDE.md, rules, and skills after new patterns are introduced |
