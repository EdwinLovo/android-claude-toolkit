---
description: Interactive first-run scaffolder — asks about networking, paging, and naming, then writes the folder tree and every primitive the rules assume exists
---

# /init-project

**Run once at project start.** Interactive first-run scaffolder: asks a handful of questions (networking library, paging, project + package names), then produces the presentation/data/domain folder tree, the utility files, and every primitive the rules assume exists.

Idempotent-friendly caveat: this command **overwrites** files with the same paths. If you're re-running mid-project, review the diff before accepting the writes.

## What this creates

- **Folder tree** under `<PKG_ROOT>/` — `domain/`, `data/` (with `di/`, `mappers/`, `paging/`, `remote/datasource/`, `repository/`), `presentation/` (with `ui/components/*`, `ui/navigation/`, `ui/theme/colors/`, `utils/ext/`, `ux/main/`)
- **Theme layer** — via the `theme-primitives` skill (7 files under `presentation/ui/theme/` + `colors/`)
- **Navigation layer** — via the `navigation-primitives` skill (3 files under `presentation/ui/navigation/`)
- **Repository layer** — via the `repository-primitives` skill (`ApiResult.kt`, `BaseRepository.kt` for the chosen networking library, `HttpConstants.kt` if Apollo, `BasePagingSource.kt` if paging)
- **Utility files + shared components** — via the `misc-primitives` skill: `ErrorEventBus.kt`, `UiText.kt`, `ObserveAsEvents.kt`, `OnLifecycleResumed.kt`, `OnLifecycleEvent.kt`, `Constants.kt`, `StateFlowExt.kt`, the `<PREVIEW>` annotations + `<PROJECT_NAME>PreviewContainer`, plus `<SCAFFOLD>.kt`, `<DIALOG>.kt`, `<PROJECT_NAME>AlertDialog.kt`, `<PROJECT_NAME>ErrorAlertDialog.kt`, and `<ICONS>.kt` stubs
- **App-shell stubs** — `<PROJECT_NAME>Application.kt` (with `@HiltAndroidApp`), `MainActivity.kt` (with `@AndroidEntryPoint` and `setContent { <THEME> { ... } }`), `AppNavHost.kt`, `MainScreen.kt`, `MainViewModel.kt`
- **DI module stubs** — `ApiModule.kt`, `RepositoryModule.kt`, plus `DataSourceModule.kt` if Apollo

## What this does NOT touch

- **`build.gradle.kts` / `libs.versions.toml`** — dependency versions vary too much per project. A checklist prints at the end with the exact deps to add.
- **`AndroidManifest.xml`** — register the `<PROJECT_NAME>Application` yourself (checklist reminds you).
- **Signing config, ProGuard, CI/CD** — out of scope.
- **Feature code** — no `HomeScreen`, no `LoginScreen`. Run `/new-screen` after init for the first feature.
- **Real color / dimension values** — `PrimitiveColors.kt` and `ExtendedColors.kt` ship as skeletons; fill from your design system.

## How it runs

Invoke the `project-scaffold` skill. It handles the question flow, the folder tree, sub-skill orchestration (theme/navigation/repository primitives), and the utility + app-shell + DI-stub writes. Skill file: `.claude/skills/project-scaffold/SKILL.md`.
