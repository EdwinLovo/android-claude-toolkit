---
name: misc-primitives
description: First-time setup for the small cross-cutting utility files — ErrorEventBus, UiText, ObserveAsEvents, OnLifecycleResumed, OnLifecycleEvent, Constants, StateFlowExt, the <PREVIEW> annotations + PreviewContainer, and the <ICONS> / <SCAFFOLD> / <DIALOG> component stubs. Invoke when scaffolding these in a new project, or when a rule references any of them and the file doesn't exist yet.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# Misc primitives

The small utility and shared-component files every project needs but that aren't big enough to warrant their own skill. Copy each into the paths below. Placeholder substitution applies throughout.

For the conventions that use these types:
- `ErrorEventBus`, `UiText`, `ObserveAsEvents` → `rules/error-handling.md`
- `OnLifecycleResumed`, `OnLifecycleEvent` → `rules/lifecycle.md`
- `<PREVIEW>` annotations + `<PROJECT_NAME>PreviewContainer` → `rules/previews.md`
- `<SCAFFOLD>` → `rules/screens.md` (single-shared-scaffold rule)
- `<DIALOG>` → `rules/dialogs.md`
- `<ICONS>` → `rules/icons.md`
- `Constants` (`ABSOLUTE_WEIGHT`, `EMPTY_STRING`, …) → `rules/constants-in-viewmodels.md`
- `StateFlowExt.reduce` → `rules/viewmodels.md`

---

## File 1 — `presentation/utils/ErrorEventBus.kt`

Global bus for repository errors. `Channel`-backed so late subscribers don't drop events; `receiveAsFlow()` for lifecycle-aware collection.

```kotlin
package <PKG_ROOT>.presentation.utils

import kotlinx.coroutines.channels.Channel
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.receiveAsFlow

object ErrorEventBus {
    private val _events = Channel<UiText>(Channel.BUFFERED)
    val events: Flow<UiText> = _events.receiveAsFlow()

    suspend fun send(error: UiText) {
        _events.send(error)
    }

    // Drains buffered events left over from a previous test so the channel is clean.
    fun drain() {
        generateSequence { _events.tryReceive().getOrNull() }.count()
    }
}
```

---

## File 2 — `presentation/utils/UiText.kt`

Sealed carrier for user-visible strings. Non-composable code produces `UiText`; the composable renders via `.asString()` — no `Context` leaks.

Requires `<string name="generic_error">Something went wrong. Please try again.</string>` in `strings.xml`.

```kotlin
package <PKG_ROOT>.presentation.utils

import androidx.annotation.StringRes
import androidx.compose.runtime.Composable
import androidx.compose.ui.res.stringResource
import <PKG_ROOT>.R

sealed interface UiText {
    data class StringResource(
        @param:StringRes val id: Int,
    ) : UiText

    data class DynamicString(val value: String) : UiText
}

@Composable
fun UiText.asString(): String =
    when (this) {
        is UiText.StringResource -> stringResource(id)
        is UiText.DynamicString -> value
    }

fun String?.toUiText(): UiText =
    this?.let { UiText.DynamicString(it) } ?: UiText.StringResource(R.string.generic_error)
```

---

## File 3 — `presentation/utils/ObserveAsEvents.kt`

Lifecycle-aware `Flow` collector for events (one-shot signals like error dialogs), not state.

```kotlin
package <PKG_ROOT>.presentation.utils

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.ui.platform.LocalLifecycleOwner
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.repeatOnLifecycle
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.withContext

@Composable
fun <T> ObserveAsEvents(
    flow: Flow<T>,
    key1: Any? = null,
    onEvent: (T) -> Unit,
) {
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(lifecycleOwner.lifecycle, key1, flow) {
        lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
            withContext(Dispatchers.Main.immediate) {
                flow.collect(onEvent)
            }
        }
    }
}
```

---

## File 4 — `presentation/utils/OnLifecycleResumed.kt`

Fires `onResumed` every time the composable's lifecycle transitions to RESUMED. Use for refresh-on-return, resume-time analytics, etc.

```kotlin
package <PKG_ROOT>.presentation.utils

import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.compose.LocalLifecycleOwner
import androidx.lifecycle.repeatOnLifecycle
import kotlinx.coroutines.launch

@Composable
fun OnLifecycleResumed(onResumed: () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    LaunchedEffect(lifecycleOwner) {
        launch {
            lifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.RESUMED) {
                onResumed()
            }
        }
    }
}
```

---

## File 5 — `presentation/utils/OnLifecycleEvent.kt`

General-purpose single-event observer. Prefer `OnLifecycleResumed` for the RESUMED-only case; use this when you need a different lifecycle event (STARTED, STOPPED, DESTROYED, …).

```kotlin
package <PKG_ROOT>.presentation.utils

import androidx.compose.runtime.Composable
import androidx.compose.runtime.DisposableEffect
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.LifecycleEventObserver
import androidx.lifecycle.compose.LocalLifecycleOwner

@Composable
fun OnLifecycleEvent(event: Lifecycle.Event, onEvent: () -> Unit) {
    val lifecycleOwner = LocalLifecycleOwner.current
    DisposableEffect(lifecycleOwner) {
        val observer = LifecycleEventObserver { _, e ->
            if (e == event) onEvent()
        }
        lifecycleOwner.lifecycle.addObserver(observer)
        onDispose { lifecycleOwner.lifecycle.removeObserver(observer) }
    }
}
```

---

## File 6 — `presentation/utils/Constants.kt`

Cross-feature named constants. Replaces magic literals (`1f`, `""`, `500L`, `401`, …) with intent-revealing names. Extend as the project grows.

```kotlin
package <PKG_ROOT>.presentation.utils

const val EMPTY_STRING = ""
const val SPACE_STRING = " "
const val DOT_STRING = "."

const val ABSOLUTE_WEIGHT = 1F
const val DEBOUNCE_TIME = 500L
const val FLOW_SHARING_STARTED = 5_000L

// HTTP
const val UNAUTHORIZED_CODE = 401
const val NOT_FOUND_CODE = 404
const val INTERNAL_SERVER_ERROR_CODE = 500

// Max lines
const val MAX_LINES_SINGLE = 1

// Opacity
const val DEFAULT_OPACITY = 1f
const val DISABLED_OPACITY = 0.5f
const val NO_OPACITY = 0f
```

---

## File 7 — `presentation/utils/ext/StateFlowExt.kt`

The `reduce` extension referenced by `rules/viewmodels.md`. Lets `uiState.reduce { copy(...) }` sit natively next to a `val uiState: StateFlow<T>` with an explicit backing field.

```kotlin
package <PKG_ROOT>.presentation.utils.ext

import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.update

fun <T> MutableStateFlow<T>.reduce(block: T.() -> T) = update { it.block() }
```

---

## File 8 — `presentation/utils/<PREVIEW>.kt`

The two annotations (component + screen) plus `<PROJECT_NAME>PreviewContainer`. Every `@Composable` preview in the app uses these — see `rules/previews.md`.

Adjust `widthDp` / `heightDp` on `@<PREVIEW>Screen` for your target device (tablet `1280 × 800` shown here; phone e.g. `411 × 892`).

```kotlin
package <PKG_ROOT>.presentation.utils

import android.content.res.Configuration
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.padding
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import <PKG_ROOT>.presentation.ui.theme.<THEME>

@Preview(name = "Light", showBackground = true, uiMode = Configuration.UI_MODE_NIGHT_NO, backgroundColor = 0xffffffff)
@Preview(name = "Dark", showBackground = true, uiMode = Configuration.UI_MODE_NIGHT_YES, backgroundColor = 0xff000000)
annotation class <PREVIEW>

@Preview(name = "Light — Tablet", showBackground = true, uiMode = Configuration.UI_MODE_NIGHT_NO, backgroundColor = 0xffffffff, widthDp = 1280, heightDp = 800)
@Preview(name = "Dark — Tablet", showBackground = true, uiMode = Configuration.UI_MODE_NIGHT_YES, backgroundColor = 0xff000000, widthDp = 1280, heightDp = 800)
annotation class <PREVIEW>Screen

@Composable
fun <PROJECT_NAME>PreviewContainer(content: @Composable () -> Unit) {
    <THEME> {
        Column(modifier = Modifier.padding(<THEME>.padding.xs)) {
            content()
        }
    }
}
```

---

## File 9 — `presentation/utils/<ICONS>.kt`

Registry stub. Every icon in the app is looked up through this object — never `Icons.*` (Material Icons). See `rules/icons.md`.

Workflow: (1) drop a vector drawable into `res/drawable/` (e.g. `ic_settings.xml`); (2) register it here; (3) reference from UI as `Icon(painterResource(<ICONS>.Settings), contentDescription = ...)`.

```kotlin
package <PKG_ROOT>.presentation.utils

import <PKG_ROOT>.R

/**
 * Every icon in the app is registered here — do not use Material Icons or import `Icons.*`.
 *
 * To add a new icon:
 * 1. Drop the vector drawable XML into `res/drawable/`, e.g. `ic_settings.xml`.
 * 2. Register it here: `val Settings = R.drawable.ic_settings`.
 * 3. Use it: `Icon(painterResource(<ICONS>.Settings), contentDescription = ...)` for tinted vectors,
 *    or `Image(painterResource(<ICONS>.SomeArt))` for full-color raster images.
 */
object <ICONS> {
    // val Settings = R.drawable.ic_settings
    // val Search = R.drawable.ic_search
}
```

---

## File 10 — `presentation/ui/components/scaffold/<SCAFFOLD>.kt`

The single shared scaffold. Every feature screen renders through this — no per-feature `<Feature>Scaffold` variants. See `rules/screens.md`.

Screens pass their own top/bottom bars as slot arguments. Adjust the `containerColor` once `ExtendedColors` is populated (typically `backgroundMuted` or `backgroundDefault`).

```kotlin
package <PKG_ROOT>.presentation.ui.components.scaffold

import androidx.compose.foundation.layout.PaddingValues
import androidx.compose.material3.Scaffold
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import <PKG_ROOT>.presentation.ui.theme.<THEME>

@Composable
fun <SCAFFOLD>(
    modifier: Modifier = Modifier,
    topBar: @Composable (() -> Unit)? = null,
    bottomBar: @Composable (() -> Unit)? = null,
    content: @Composable (PaddingValues) -> Unit,
) {
    Scaffold(
        modifier = modifier,
        containerColor = <THEME>.colors.backgroundDefault,
        topBar = topBar ?: {},
        bottomBar = bottomBar ?: {},
        content = content,
    )
}
```

---

## File 11 — `presentation/ui/components/dialogs/<DIALOG>.kt`

The shared dialog wrapper. Every dialog composes through this — provides adaptive sizing, theme background, corner radius. See `rules/dialogs.md`.

The width fraction is a starting point (~62% at expanded window widths). Add a `WindowSizeClass` branch if your app supports both compact and expanded windows.

```kotlin
package <PKG_ROOT>.presentation.ui.components.dialogs

import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Surface
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.window.Dialog
import androidx.compose.ui.window.DialogProperties
import <PKG_ROOT>.presentation.ui.theme.<THEME>

@Composable
fun <DIALOG>(
    onDismissRequest: () -> Unit,
    modifier: Modifier = Modifier,
    dismissOnBackPress: Boolean = true,
    dismissOnClickOutside: Boolean = true,
    content: @Composable () -> Unit,
) {
    Dialog(
        onDismissRequest = onDismissRequest,
        properties = DialogProperties(
            dismissOnBackPress = dismissOnBackPress,
            dismissOnClickOutside = dismissOnClickOutside,
            usePlatformDefaultWidth = false,
        ),
    ) {
        Surface(
            modifier = modifier.fillMaxWidth(fraction = 0.62f),
            shape = RoundedCornerShape(<THEME>.cornerRadius.lg),
            color = <THEME>.colors.backgroundDefault,
            content = { content() },
        )
    }
}
```

---

## Verification after copying

Once these files exist:

- `presentation/utils/ErrorEventBus.kt`
- `presentation/utils/UiText.kt`
- `presentation/utils/ObserveAsEvents.kt`
- `presentation/utils/OnLifecycleResumed.kt`
- `presentation/utils/OnLifecycleEvent.kt`
- `presentation/utils/Constants.kt`
- `presentation/utils/ext/StateFlowExt.kt`
- `presentation/utils/<PREVIEW>.kt`
- `presentation/utils/<ICONS>.kt`
- `presentation/ui/components/scaffold/<SCAFFOLD>.kt`
- `presentation/ui/components/dialogs/<DIALOG>.kt`

Every rule that references `ErrorEventBus.send(...)`, `uiText.asString()`, `OnLifecycleResumed { }`, `<PREVIEW>Screen`, `<SCAFFOLD> { }`, `<DIALOG>`, `<ICONS>.<Name>`, `ABSOLUTE_WEIGHT` / `EMPTY_STRING`, or `uiState.reduce { copy(...) }` will now resolve. See the corresponding rules linked at the top of this skill for usage patterns.
