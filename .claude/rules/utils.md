---
name: utils
description: Non-extension utility files — where singletons, sealed types, composable helpers, enums, and constants live across presentation/domain scopes
paths:
  - "**/presentation/utils/*.kt"
  - "**/presentation/ux/**/utils/*.kt"
  - "**/domain/utils/*.kt"
---

# Utility files

Three utility locations, each with a distinct purpose. `Utils` never appears in a file name — the file is named for what it holds.

## `presentation/utils/` — global presentation utilities

Content types you'll find here:

| Type | Example | Notes |
|---|---|---|
| Singleton event bus | `ErrorEventBus.kt`, `OrderEventBus.kt` | `object` or `@Singleton class`; exposes a `SharedFlow<T>` and a `send()` |
| Sealed type shared across the UI layer | `UiText.kt` (with `StringResource` / `DynamicString`) | Sealed type + its helper extensions can live in the same file |
| Annotation definitions | `<PREVIEW>.kt` — the `@<PREVIEW>` and `@<PREVIEW>Screen` annotations + `<PROJECT_NAME>PreviewContainer` | Compose preview annotations |
| Composable helpers | `ObserveAsEvents.kt`, `OnLifecycleResumed.kt` | Small `@Composable` helpers that don't belong in `components/` |
| Icons registry | `<ICONS>.kt` | Central mapping of `<ICONS>.X = R.drawable.ic_x` (see `rules/icons.md`) |
| Global constants | `Constants.kt` | `const val` values used app-wide (HTTP codes, opacity values, shared strings) |

## `presentation/ux/<feature>/utils/` — feature-scoped utilities

Content types:

| Type | Example | Notes |
|---|---|---|
| Feature enum | `CatalogTab.kt`, `OrdersTab.kt`, `SettingsSection.kt` | Enum used only inside one feature; if it becomes truly shared, promote to `domain/model/<area>/` |
| Extension on feature enum | `CatalogTabExt.kt`, `OrdersTabExt.kt` | Follows `rules/extensions.md`; one receiver per file |
| Receiver-less helper | `DetailComputations.kt`, `MembershipChipLabel.kt` | Rare — prefer an extension when a natural receiver exists |
| Preview parameter providers | `<Screen>PreviewParams.kt` (e.g. `OrdersPreviewParams.kt`) | Complex `PreviewParameterProvider` classes for screen-level `UiState` (see `rules/previews.md`) |
| Feature constants | `<feature>/utils/Constants.kt` | Only when several files in the feature need shared `const val`s |

## `domain/utils/` — domain-layer cross-cutting types

Content types:

| Type | Example | Notes |
|---|---|---|
| Sealed type bridging layer vocabulary | `ApiResult<T>` and its `.onSuccess` / `.onError` / `.onLoading` extensions | Used by both `data/repository/` (produces) and `presentation/` (consumes) |

Keep `domain/utils/` small — it's for genuinely cross-layer types, not a catch-all.

## Naming rules

- **No `*Utils.kt` suffix, ever.** The file's name is the thing it holds:
  - ✅ `ErrorEventBus.kt`, `UiText.kt`, `Constants.kt`, `ApiResult.kt`, `CatalogTab.kt`
  - ❌ `EventBusUtils.kt`, `StringUtils.kt`, `AppHelpers.kt`
- **Singletons / sealed types / annotations** → `<Type>.kt` (no suffix)
- **Extensions on any type** → `<Type>Ext.kt` (see `rules/extensions.md`)
- **`Constants.kt`** — one per scope; feature-scoped constants go in `<feature>/utils/Constants.kt`, global constants in `presentation/utils/Constants.kt`

## Do not

- Create a `data/utils/` folder — data-layer helpers live in `data/mappers/` or beside the class they support
- Put a feature-only enum under global `presentation/utils/` — it belongs in `presentation/ux/<feature>/utils/`
- Bury a domain-level cross-layer sealed type (`ApiResult`, etc.) inside `presentation/utils/` — it goes in `domain/utils/`
- Create an `object AppHelpers { fun a(); fun b(); fun c() }` catch-all — split by responsibility; each item becomes an extension, a singleton, or a focused `<Purpose>.kt`

## Common violations

- `presentation/utils/CatalogTab.kt` (feature-only enum in global utils) → move to `presentation/ux/catalog/utils/CatalogTab.kt`
- `presentation/utils/StringHelpers.kt` (contains extensions on `String`) → rename to `StringExt.kt`, move to `presentation/utils/ext/`
- `object AppHelpers { ... }` — a catch-all singleton → split. If items have a receiver, they're extensions; if not, each gets its own `<Purpose>.kt` file
- `data/utils/NetworkHelpers.kt` — no such folder should exist → move contents to `data/mappers/<feature>/`, `BaseRepository`, or wherever the code that uses them lives
- `presentation/utils/ApiResult.kt` — wrong layer for a domain-vocabulary type → move to `domain/utils/ApiResult.kt`
- Two unrelated singletons in one file (`ErrorEventBus` + `OrderEventBus` in `EventBuses.kt`) → one file per singleton (`ErrorEventBus.kt`, `OrderEventBus.kt`)
