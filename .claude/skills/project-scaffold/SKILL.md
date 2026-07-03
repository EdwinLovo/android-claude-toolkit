---
name: project-scaffold
description: One-time project scaffold — asks about networking library and paging, creates the presentation/data/domain folder tree, invokes theme-primitives + navigation-primitives + repository-primitives + misc-primitives + component-primitives, and writes app-shell stubs (Application, MainActivity, AppNavHost) plus DI module stubs. Invoke via /init-project or when starting a fresh Android project against this toolkit.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Skill
---

# Project scaffold

Interactive first-run scaffolder. Composes five primitives skills (`theme-primitives`, `navigation-primitives`, `repository-primitives`, `misc-primitives`, `component-primitives`) and layers on the project-specific app-shell + DI stubs. Produces everything a fresh project needs under `<PKG_ROOT>/` — folder tree, primitives, utilities, app-shell stubs, DI module stubs — with placeholders substituted.

**Out of scope:** `build.gradle.kts`, `libs.versions.toml`, `AndroidManifest.xml`, signing config, feature code, real palette values. See the "Next steps" checklist emitted at the end.

---

## Step 1 — Gather answers

Run three rounds of `AskUserQuestion` (the tool caps at 4 questions per call). Do not write any files until all three rounds are answered and the placeholder table (Step 2) is confirmed.

### Round 1 — architecture choices

Ask these four questions in a single `AskUserQuestion` call:

1. **`Networking library`** (`header: Networking`) — options: `Retrofit` (Recommended) / `Apollo (GraphQL)`. Drives which `BaseRepository` variant `repository-primitives` writes, whether `HttpConstants.kt` and `data/remote/datasource/` are created, and whether `DataSourceModule.kt` gets a stub.
2. **`Paging 3`** (`header: Paging`) — options: `Yes` / `No` (Recommended). Drives `data/paging/BasePagingSource.kt` presence.
3. **`Include preview infrastructure`** (`header: Previews`) — options: `Yes` (Recommended) / `No`. Adds the `<PREVIEW>` annotations + `<PROJECT_NAME>PreviewContainer` under `presentation/utils/`.
4. **`Include ErrorEventBus + UiText`** (`header: Error bus`) — options: `Yes` (Recommended) / `No`. Adds `ErrorEventBus.kt`, `UiText.kt`, `ObserveAsEvents.kt` under `presentation/utils/`.

### Round 2 — naming (identity)

Ask these four questions in a single `AskUserQuestion` call:

1. **`Project display name`** (`header: Project`) — free text. Becomes `<PROJECT_NAME>` (e.g. `Acme`). Suggest based on the current directory name.
2. **`Package root`** (`header: Package`) — free text. Becomes `<PKG_ROOT>` and `<APP_ID>` (e.g. `com.acme.pos`). Must be lowercase, dot-separated, no dashes.
3. **`Theme accessor + composable name`** (`header: Theme`) — options: `<PROJECT_NAME>Theme` (Recommended) / other. Becomes `<THEME>`. Shared by the theme composable AND accessor object (Material3 pattern).
4. **`Ticket prefix pattern`** (`header: Ticket`) — free text. Becomes `<TICKET_PREFIX>` (e.g. `ACME-XXX`).

### Round 3 — naming (components)

Ask these four questions in a single `AskUserQuestion` call:

1. **`Scaffold composable name`** (`header: Scaffold`) — options: `<PROJECT_NAME>Scaffold` (Recommended) / other. Becomes `<SCAFFOLD>`.
2. **`Dialog container name`** (`header: Dialog`) — options: `<PROJECT_NAME>DialogContainer` (Recommended) / other. Becomes `<DIALOG>`.
3. **`Icons registry name`** (`header: Icons`) — options: `<PROJECT_NAME>Icons` (Recommended) / other. Becomes `<ICONS>`.
4. **`Preview annotation name`** (`header: Preview`) — options: `<PROJECT_NAME>Preview` (Recommended) / other. Becomes `<PREVIEW>` (and `<PREVIEW>Screen` for tablet variants).

---

## Step 2 — Confirm placeholder substitutions

Echo the resolved substitutions back for confirmation before writing anything:

```
Placeholder substitutions:
  <PROJECT_NAME>    → <resolved>
  <PKG_ROOT>        → <resolved>
  <APP_ID>          → <resolved>            (same as PKG_ROOT)
  <THEME>           → <resolved>
  <SCAFFOLD>        → <resolved>
  <DIALOG>          → <resolved>
  <ICONS>           → <resolved>
  <PREVIEW>         → <resolved>
  <TICKET_PREFIX>   → <resolved>
  <NET_LIB>         → Retrofit | Apollo
  Paging 3          → yes | no
  Preview infra     → yes | no
  ErrorEventBus     → yes | no
```

If any looks wrong, ask which to change and re-run the relevant round. Otherwise proceed.

---

## Step 3 — Create folder tree

Under `<PKG_ROOT>/` (which typically maps to `app/src/main/java/<pkg-path>/`):

```
domain/
  model/
  repository/
  usecase/
  utils/
data/
  di/
  mappers/
  paging/            [only if Paging 3 = yes]
  remote/
    datasource/      [only if Apollo]
  repository/
presentation/
  ui/
    components/
      buttons/
      dialogs/
      feedback/
      inputs/
      rows/
      scaffold/
      text/
      toolbar/
    navigation/
    theme/
      colors/
  utils/
    ext/
  ux/
    main/
```

Empty directories don't survive git without a marker — drop a `.gitkeep` in each leaf that would otherwise stay empty.

---

## Step 4 — Invoke sub-skills in order

Each of these existing skills covers one slice of the scaffold. Invoke them via the `Skill` tool with the resolved placeholder values so their file writes land pre-substituted.

1. **`theme-primitives`** — writes 7 files under `presentation/ui/theme/`:
   - `Theme.kt`, `Type.kt`, `Dimens.kt`, `colors/PrimitiveColors.kt`, `colors/ExtendedColors.kt`, `colors/LightMaterialColorScheme.kt`, `colors/DarkMaterialColorScheme.kt`
2. **`navigation-primitives`** — writes 3 files under `presentation/ui/navigation/`:
   - `NavigationRoute.kt`, `NavigationAction.kt`, `ViewModelNav.kt`
3. **`repository-primitives`** — writes:
   - `domain/utils/ApiResult.kt` (always)
   - `data/repository/BaseRepository.kt` (Apollo or Retrofit variant, per Round 1 answer)
   - `data/remote/HttpConstants.kt` (Apollo only)
   - `data/paging/BasePagingSource.kt` (only if Paging 3 = yes)
4. **`misc-primitives`** — writes 9 utility files under `presentation/utils/` and `presentation/utils/ext/`: `ErrorEventBus.kt`, `UiText.kt`, `ObserveAsEvents.kt`, `OnLifecycleResumed.kt`, `OnLifecycleEvent.kt`, `Constants.kt`, `ext/StateFlowExt.kt`, `<PREVIEW>.kt`, `<ICONS>.kt`. Pass the resolved `<PREVIEW>` / `<ICONS>` / `<THEME>` / `<PROJECT_NAME>` values so its file writes land pre-substituted.
5. **`component-primitives`** — writes 4 shared UI component files under `presentation/ui/components/scaffold/` and `presentation/ui/components/dialogs/`: `<SCAFFOLD>.kt`, `<DIALOG>.kt`, `<PROJECT_NAME>AlertDialog.kt`, `<PROJECT_NAME>ErrorAlertDialog.kt`. Always runs — the MainScreen stub renders `<PROJECT_NAME>ErrorAlertDialog`. Pass the resolved `<SCAFFOLD>` / `<DIALOG>` / `<THEME>` / `<PROJECT_NAME>` values.

If Round 1 answered "No" to Include ErrorEventBus, tell `misc-primitives` to skip `ErrorEventBus.kt`, `UiText.kt`, and `ObserveAsEvents.kt`. If Round 1 answered "No" to preview infrastructure, tell it to skip `<PREVIEW>.kt`.

If any sub-skill errors, stop and report — don't half-scaffold the project.

---

## Step 5 — Write app-shell stubs

These files exist to make the project *run*. Fill in real content as you build.

### `<PKG_ROOT>/<PROJECT_NAME>Application.kt`

Register this class in `AndroidManifest.xml` under `<application android:name=".<PROJECT_NAME>Application">`.

```kotlin
package <PKG_ROOT>

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class <PROJECT_NAME>Application : Application()
```

### `<PKG_ROOT>/presentation/ux/main/MainActivity.kt`

The single Activity. Hosts the Compose graph.

```kotlin
package <PKG_ROOT>.presentation.ux.main

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.viewModels
import androidx.navigation.compose.rememberNavController
import dagger.hilt.android.AndroidEntryPoint
import <PKG_ROOT>.presentation.ui.theme.<THEME>

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            <THEME> {
                val navController = rememberNavController()
                MainScreen(
                    navController = navController,
                    viewModel = viewModel,
                    startDestination = TODO("pick a start route, e.g. HomeRoute"),
                )
            }
        }
    }
}
```

### `<PKG_ROOT>/presentation/ux/main/MainScreen.kt`

The app shell — **does not** wrap in `<SCAFFOLD>` (see `rules/screens.md`, app-shell exception). Hosts `AppNavHost` + observes `ErrorEventBus`.

```kotlin
package <PKG_ROOT>.presentation.ux.main

import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.navigation.NavHostController
import <PKG_ROOT>.presentation.ui.navigation.AppNavHost
import <PKG_ROOT>.presentation.ui.navigation.HandleNavigation
import <PKG_ROOT>.presentation.ui.navigation.NavigationRoute
import <PKG_ROOT>.presentation.utils.ErrorEventBus
import <PKG_ROOT>.presentation.utils.ObserveAsEvents
import <PKG_ROOT>.presentation.utils.UiText

@Composable
fun MainScreen(
    navController: NavHostController,
    viewModel: MainViewModel,
    startDestination: NavigationRoute,
) {
    var errorToShow by remember { mutableStateOf<UiText?>(null) }
    ObserveAsEvents(ErrorEventBus.events) { errorToShow = it }

    AppNavHost(navController, startDestination)
    HandleNavigation(viewModelNav = viewModel, navController = navController)

    // TODO — render errorToShow via a <DIALOG>-based error dialog. See rules/error-handling.md.
}
```

### `<PKG_ROOT>/presentation/ux/main/MainViewModel.kt`

```kotlin
package <PKG_ROOT>.presentation.ux.main

import androidx.lifecycle.ViewModel
import dagger.hilt.android.lifecycle.HiltViewModel
import javax.inject.Inject
import <PKG_ROOT>.presentation.ui.navigation.ViewModelNav
import <PKG_ROOT>.presentation.ui.navigation.ViewModelNavImpl

@HiltViewModel
class MainViewModel @Inject constructor() : ViewModel(), ViewModelNav by ViewModelNavImpl()
```

### `<PKG_ROOT>/presentation/ui/navigation/AppNavHost.kt`

```kotlin
package <PKG_ROOT>.presentation.ui.navigation

import androidx.compose.foundation.layout.systemBarsPadding
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.navigation.NavHostController
import androidx.navigation.compose.NavHost

@Composable
fun AppNavHost(
    navController: NavHostController,
    startDestination: NavigationRoute,
) {
    NavHost(
        navController = navController,
        startDestination = startDestination,
        modifier = Modifier.systemBarsPadding(),
    ) {
        // Register routes here as you build features:
        // composable<HomeRoute> { HomeScreen(navController) }
        // composable<LoginRoute> { LoginScreen(navController) }
    }
}
```

---

## Step 6 — Write DI module stubs

Under `<PKG_ROOT>/data/di/`.

### `data/di/ApiModule.kt`

**Apollo variant:**

```kotlin
package <PKG_ROOT>.data.di

import com.apollographql.apollo.ApolloClient
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object ApiModule {
    @Provides
    @Singleton
    fun provideApolloClient(): ApolloClient =
        ApolloClient.Builder()
            .serverUrl(TODO("BuildConfig.SERVER_URL or similar"))
            // TODO — auth interceptors, cache, etc.
            .build()
}
```

**Retrofit variant:**

```kotlin
package <PKG_ROOT>.data.di

import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import kotlinx.serialization.json.Json
import okhttp3.MediaType.Companion.toMediaType
import okhttp3.OkHttpClient
import retrofit2.Retrofit
import retrofit2.converter.kotlinx.serialization.asConverterFactory
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object ApiModule {
    @Provides
    @Singleton
    fun provideJson(): Json = Json { ignoreUnknownKeys = true }

    @Provides
    @Singleton
    fun provideOkHttp(): OkHttpClient = OkHttpClient.Builder()
        // TODO — auth interceptor, logging interceptor, timeouts
        .build()

    @Provides
    @Singleton
    fun provideRetrofit(client: OkHttpClient, json: Json): Retrofit =
        Retrofit.Builder()
            .baseUrl(TODO("BuildConfig.SERVER_URL or similar"))
            .client(client)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .build()

    // TODO — provide each API service:
    // @Provides @Singleton
    // fun provideAuthApi(retrofit: Retrofit): AuthApi = retrofit.create(AuthApi::class.java)
}
```

### `data/di/DataSourceModule.kt`

**Apollo only.** Binds `<Resource>RemoteDataSource` implementations.

```kotlin
package <PKG_ROOT>.data.di

import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class DataSourceModule {
    // TODO — bind each RemoteDataSource:
    // @Binds @Singleton
    // abstract fun bindAuthRemoteDataSource(impl: AuthRemoteDataSourceImpl): AuthRemoteDataSource
}
```

### `data/di/RepositoryModule.kt`

Binds `<Resource>Repository` implementations to their interfaces.

```kotlin
package <PKG_ROOT>.data.di

import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    // TODO — bind each repository:
    // @Binds @Singleton
    // abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
}
```

---

## Step 7 — Emit next-steps checklist

Print this to the user verbatim (adjusting `[NET_LIB]` to the resolved value). Every item is manual work the toolkit deliberately doesn't touch:

```
✅ Scaffold complete. Next steps — none of these are automated:

- [ ] Add dependencies to build.gradle.kts / libs.versions.toml:
      Core:     Compose BOM, Compose Material3, Compose Foundation, Compose UI Tooling (debug + preview)
      DI:       Hilt runtime + Hilt compiler + Hilt Navigation Compose
      Nav:      Navigation Compose + kotlinx-serialization
      Coroutines: kotlinx-coroutines-android
      Lifecycle: lifecycle-runtime-compose + lifecycle-viewmodel-compose
      Logging:  Timber (see rules/logging.md)
      Net:      [Retrofit + kotlinx-serialization-converter | Apollo Kotlin]
      Paging:   androidx.paging + paging-compose  [only if you chose Paging]
      Tests:    JUnit + Turbine + kotlinx-coroutines-test + Truth or AssertJ (see rules/testing.md)

- [ ] Add Gradle plugins:
      Hilt, Compose Compiler, kotlinx-serialization
      [Apollo Gradle plugin — only if you chose Apollo]

- [ ] Enable the explicit-backing-fields compiler arg in build.gradle.kts:
      kotlin { compilerOptions { freeCompilerArgs.add("-Xexplicit-backing-fields") } }
      (This is what makes the toolkit's `val uiState: StateFlow<T>; field = MutableStateFlow(...)` syntax work.)

- [ ] Register <PROJECT_NAME>Application in AndroidManifest.xml:
      <application android:name=".<PROJECT_NAME>Application" ...>

- [ ] Fill in <PKG_ROOT>/presentation/ui/theme/colors/PrimitiveColors.kt with your Figma palette
- [ ] Populate lightAppColors + darkAppColors in ExtendedColors.kt with the real primitive mappings
- [ ] Add <string name="generic_error">Something went wrong. Please try again.</string> to strings.xml
      (required by UiText.toUiText() fallback)

- [ ] Pick a start route and register it in AppNavHost.kt + MainActivity.kt
      (start with `/new-screen` to scaffold your first feature screen)

- [ ] Add vector drawables under res/drawable/ and register each in <ICONS>.kt as you use them

- [ ] Once you have a first data feature, run /new-data-feature to scaffold repository + interface + mappers
```

---

## Idempotence + safety

- **Always confirm before overwriting.** If a target path already contains a file, ask the user whether to overwrite, skip, or diff.
- **Never touch `build.gradle.kts`, `libs.versions.toml`, or `AndroidManifest.xml`.** Emit checklist reminders instead.
- **Don't invent placeholder values.** If any placeholder is unresolved, stop and re-run the relevant question round.

## References

- Rules: `rules/folder-structure.md`, `rules/theming-and-tokens.md`, `rules/navigation.md`, `rules/repositories.md`, `rules/error-handling.md`, `rules/screens.md`, `rules/viewmodels.md`, `rules/dependency-injection.md`, `rules/previews.md`, `rules/lifecycle.md`
- Sub-skills: `theme-primitives`, `navigation-primitives`, `repository-primitives`, `misc-primitives`, `component-primitives`
- Post-init commands: `/new-screen`, `/new-data-feature`, `/audit-branch`
