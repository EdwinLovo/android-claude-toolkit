---
name: constants-in-viewmodels
description: ViewModel constants live in a companion object at the bottom of the class; no file-level private const val
paths:
  - "**/*ViewModel.kt"
---

# ViewModel constants

All `const val` values specific to a ViewModel live in a `companion object` **at the bottom** of the class body — after all methods, before the closing brace.

## Shape

```kotlin
@HiltViewModel
class ExampleViewModel @Inject constructor(...) : ViewModel() {

    val uiState: StateFlow<ExampleUiState>
        field = MutableStateFlow(ExampleUiState())

    fun handleEvent(event: ExampleEvent) { ... }

    private fun loadItems() {
        // uses DEBOUNCE_MS and MAX_RETRIES
    }

    companion object {
        private const val DEBOUNCE_MS = 300L
        private const val MAX_RETRIES = 3
        private const val EMPTY_QUERY = ""
    }
}
```

## Hard rules

- **`companion object` at the very bottom** — after all methods
- **`private const val`** — never `public`; if two VMs need the same value, it's not VM-specific and belongs elsewhere
- **No file-level `private const val`** — banned in ViewModel files. Any file-level constant is a signal the value is shared and should live in `presentation/utils/`
- **No inline magic literals** in the class body — a `300L` or `"init"` appearing inline is a violation; declare it in the `companion object` with a name

## What belongs elsewhere

| Value | Where |
|---|---|
| Used by multiple ViewModels | `presentation/utils/Constants.kt` |
| Used across data + presentation | `<PKG_ROOT>/utils/Constants.kt` (top-level) |
| Design token (spacing, corner radius, color) | `<THEME>` — see `rules/theming-and-tokens.md` |
| Route argument key | `<Screen>Route.kt` file top |
| `SavedStateHandle` key | The class that owns the handle (usually the VM's companion) |
| DI-scoped constants (`@Named` string) | `data/di/DiConstants.kt` |

## Before declaring a new constant

Check `presentation/utils/Constants.kt` first. If a timeout, debounce, retry count, or other value with the same meaning already lives there, reuse it — do not redeclare with a different name in your companion object. Promote a companion constant to `presentation/utils/Constants.kt` the moment a second ViewModel needs the same value.

## Common violations

- `private const val TIMEOUT = 5_000L` at the top of `FooViewModel.kt` (file-level) → move into `companion object`
- Magic `300L` inline: `debounce(300L)` → declare `DEBOUNCE_MS = 300L` in the companion, reference by name
- `companion object` placed above the methods → move to the bottom
- `const val PUBLIC_KEY = ...` (public) → make it `private`, or if it's shared, move it out of the VM
- File-level `private val PLACEHOLDER_ITEMS = listOf(...)` in a VM file → declare inside the companion object (mind that `const val` only works for compile-time constants; non-const shared data goes in an `object` in `utils/`)
