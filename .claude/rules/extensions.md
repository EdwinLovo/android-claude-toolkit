---
name: extensions
description: Extension function conventions — file naming <Type>Ext.kt, one receiver per file, placement across shared/feature/domain scopes
paths:
  - "**/presentation/utils/ext/*.kt"
  - "**/presentation/ux/**/utils/*Ext.kt"
  - "**/domain/utils/*.kt"
---

# Extension functions

Extension functions get their own file per receiver type, named `<Type>Ext.kt`. Placement depends on where the extension is used.

## Naming

- **`<Type>Ext.kt`** — file name states the receiver: `StringExt.kt` for `String.xxx`, `DoubleExt.kt` for `Double.xxx`, `ModifierExt.kt` for `Modifier.xxx`, `ApiResultExt.kt` for `ApiResult<T>.xxx`
- **Never `<Type>Utils.kt`** — the `Ext` suffix is what marks the file as extension functions; `Utils` is ambiguous and banned (see `rules/utils.md`)
- **One receiver type per file** — mixing receivers is a violation. If both `String` and `Int` need extensions, that's `StringExt.kt` + `IntExt.kt`, not `HelperExt.kt`

## Placement

The receiver's scope decides which folder the file lives in:

| Extension use | Location |
|---|---|
| Used across multiple features | `presentation/utils/ext/<Type>Ext.kt` |
| Used inside one feature only | `presentation/ux/<feature>/utils/<Type>Ext.kt` |
| On a domain-utility sealed type (e.g. `ApiResult`) | Beside the type in `domain/utils/` (same file or `<Type>Ext.kt`) |
| On a domain **model**, and touches only domain types | `domain/model/<area>/<Name>Mappers.kt` (see `rules/mappers.md`) |
| On a network codegen type, produces a domain type | `data/mappers/<feature>/<Name>Mapper.kt` (see `rules/mappers.md`) |

**No `data/utils/` folder.** Data-layer helpers either live under `data/mappers/` or beside the class that uses them.

## Mixing `@Composable` and non-composable extensions

Allowed on the same receiver when they are short and clearly related. Example: `ModifierExt.kt` holds both `Modifier.thenIf()` (non-composable) and `Modifier.<projectCard>()` (composable) — they're both on `Modifier` and both about visual chaining.

Split the file when:
- Either the composable or the non-composable side grows past ~5 functions
- The two flavors serve different purposes (e.g., testable pure functions vs. Compose-only helpers)
- A reader has to jump between mental models to understand the file

## Visibility

- **Public by default** (implicit) — the project treats extensions as internal-to-module API
- **`internal`** only when the extension genuinely shouldn't leak out of its subsystem
- **Never `private`** at file level for extensions — if it's private-to-file, it belongs inline where it's used, not as an extension

## Documentation

KDoc is optional but encouraged when:
- The receiver + name doesn't fully convey behavior (`fun String.parseIsoDate(): LocalDate` — obvious; `fun String.toUiText(): UiText` — worth a one-liner)
- There's a non-obvious edge case (nullability, truncation, error handling)

## Common violations

- `object DoubleUtils { fun format(v: Double): String }` → rewrite as `fun Double.toDisplayPrice(): String` in `utils/ext/DoubleExt.kt`
- Two receivers in one file (`fun String.parseX()` next to `fun Int.formatY()`) → split into `StringExt.kt` + `IntExt.kt`
- File named `<Type>Utils.kt` → rename to `<Type>Ext.kt`
- Extension used in only one feature placed under global `presentation/utils/ext/` → move to `presentation/ux/<feature>/utils/<Type>Ext.kt`
- Extension used across features placed under `<feature>/utils/` → move to `presentation/utils/ext/`
- Extension on a domain model in `data/mappers/` when it doesn't cross the network boundary → move to `domain/model/<area>/<Name>Mappers.kt`
- `private fun` at file level in `utils/ext/*.kt` → either delete (unused) or inline at the call site (single-file scope defeats the point of `utils/ext/`)
