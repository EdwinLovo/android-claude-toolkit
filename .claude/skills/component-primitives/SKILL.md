---
name: component-primitives
description: First-time setup for the shared UI component stubs — the single <SCAFFOLD>, the <DIALOG> container, and the <PROJECT_NAME>AlertDialog / <PROJECT_NAME>ErrorAlertDialog variants. Invoke when scaffolding these in a new project, or when a rule references any of them and the file doesn't exist yet.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# Component primitives

The shared UI components every screen and dialog composes through. Copy each into the paths below. Placeholder substitution applies throughout.

For the conventions that use these components:
- `<SCAFFOLD>` → `rules/screens.md` (single-shared-scaffold rule)
- `<DIALOG>`, `<PROJECT_NAME>AlertDialog` → `rules/dialogs.md`
- `<PROJECT_NAME>ErrorAlertDialog` → `rules/error-handling.md` (rendered by `MainScreen` when the `ErrorEventBus` emits)

These depend on primitives from other skills: `<THEME>` tokens (`theme-primitives`) and `UiText` / `asString()` (`misc-primitives`). Run those first in a fresh project.

---

## File 1 — `presentation/ui/components/scaffold/<SCAFFOLD>.kt`

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

## File 2 — `presentation/ui/components/dialogs/<DIALOG>.kt`

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

## File 3 — `presentation/ui/components/dialogs/<PROJECT_NAME>AlertDialog.kt`

The standard confirmation dialog — a title, a message, a primary button, and an optional ghost button, stacked at the compact width. Confirmation and error dialogs use this instead of building a custom body. See `rules/dialogs.md`.

The `Button` / `TextButton` pair is a starting point — swap in your project's styled button component once one exists.

```kotlin
package <PKG_ROOT>.presentation.ui.components.dialogs

import androidx.compose.foundation.layout.Arrangement
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Button
import androidx.compose.material3.ButtonDefaults
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.material3.TextButton
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import <PKG_ROOT>.presentation.ui.theme.<THEME>

@Composable
fun <PROJECT_NAME>AlertDialog(
    title: String,
    message: String,
    primaryButtonText: String,
    onPrimaryClick: () -> Unit,
    onDismiss: () -> Unit,
    ghostButtonText: String? = null,
    onGhostClick: (() -> Unit)? = null,
) {
    <DIALOG>(onDismissRequest = onDismiss) {
        Column(
            modifier = Modifier.padding(<THEME>.padding.xl),
            verticalArrangement = Arrangement.spacedBy(<THEME>.spacing.lg),
        ) {
            Text(
                text = title,
                style = MaterialTheme.typography.titleLarge,
                color = <THEME>.colors.textDefault,
            )

            Text(
                text = message,
                style = MaterialTheme.typography.bodyMedium,
                color = <THEME>.colors.textMuted,
            )

            Column(verticalArrangement = Arrangement.spacedBy(<THEME>.spacing.sm)) {
                Button(
                    modifier = Modifier.fillMaxWidth(),
                    colors = ButtonDefaults.buttonColors(containerColor = <THEME>.colors.backgroundPrimaryDefault),
                    onClick = onPrimaryClick,
                ) {
                    Text(text = primaryButtonText)
                }

                if (ghostButtonText != null) {
                    TextButton(
                        modifier = Modifier.fillMaxWidth(),
                        onClick = onGhostClick ?: onDismiss,
                    ) {
                        Text(text = ghostButtonText, color = <THEME>.colors.textPrimary)
                    }
                }
            }
        }
    }
}
```

---

## File 4 — `presentation/ui/components/dialogs/<PROJECT_NAME>ErrorAlertDialog.kt`

The error variant rendered by `MainScreen` when the `ErrorEventBus` emits — takes a `UiText` directly. See `rules/error-handling.md`.

Requires two entries in `strings.xml`: `<string name="error_alert_title">Something went wrong</string>` and `<string name="error_alert_dismiss">OK</string>`.

```kotlin
package <PKG_ROOT>.presentation.ui.components.dialogs

import androidx.compose.runtime.Composable
import androidx.compose.ui.res.stringResource
import <PKG_ROOT>.R
import <PKG_ROOT>.presentation.utils.UiText
import <PKG_ROOT>.presentation.utils.asString

@Composable
fun <PROJECT_NAME>ErrorAlertDialog(
    message: UiText,
    onDismiss: () -> Unit,
) {
    <PROJECT_NAME>AlertDialog(
        title = stringResource(R.string.error_alert_title),
        message = message.asString(),
        primaryButtonText = stringResource(R.string.error_alert_dismiss),
        onPrimaryClick = onDismiss,
        onDismiss = onDismiss,
    )
}
```

---

## Verification after copying

Once these files exist:

- `presentation/ui/components/scaffold/<SCAFFOLD>.kt`
- `presentation/ui/components/dialogs/<DIALOG>.kt`
- `presentation/ui/components/dialogs/<PROJECT_NAME>AlertDialog.kt`
- `presentation/ui/components/dialogs/<PROJECT_NAME>ErrorAlertDialog.kt`

Every rule that references `<SCAFFOLD> { }`, `<DIALOG>`, `<PROJECT_NAME>AlertDialog`, or `<PROJECT_NAME>ErrorAlertDialog` will now resolve. See the corresponding rules linked at the top of this skill for usage patterns.
