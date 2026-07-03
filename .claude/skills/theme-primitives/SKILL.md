---
name: theme-primitives
description: First-time setup for the theme layer — reference implementations for the theme composable, accessor object, primitive/extended color layers, Material color schemes, dimension tokens, and typography. Invoke when scaffolding theming in a new project, when a rule references <THEME>.colors / <THEME>.spacing / etc. and the files don't exist, or when adding a new color token or dimension size.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# Theme primitives

Reference implementations for the seven files that back the theming pattern documented in `.claude/rules/theming-and-tokens.md`. Copy them into `<PKG_ROOT>/presentation/ui/theme/` when setting up a new project. Every `<THEME>.colors.<token>`, `<THEME>.spacing.<size>`, `<THEME>.padding.<size>`, `<THEME>.cornerRadius.<size>`, `<THEME>.iconSize.<size>`, and `<THEME>.borderWidth.<size>` reference in the toolkit assumes they exist.

For the conventions that use these types (accessor rules, Material3 ban, one-off literals, common violations), see `.claude/rules/theming-and-tokens.md`.

## Overview — three-layer color architecture

Colors flow through three layers, ordered by distance from the UI:

1. **Primitive** (`colors/PrimitiveColors.kt`, `internal`) — the raw palette (`slate500`, `primary500`, `red500`). Referenced only by `ExtendedColors` implementations and the Material color schemes. UI code never sees these.
2. **Extended** (`colors/ExtendedColors.kt`) — `interface ExtendedColors` with semantic tokens (`backgroundPrimaryDefault`, `textDefault`, `borderMuted`) and its light + dark implementations. This is the app's color surface — everything the UI reads passes through here, via `LocalExtendedColors.current`.
3. **Material** (`colors/{Light,Dark}MaterialColorScheme.kt`, `internal`) — thin mapping from primitives to Material3 roles (`primary`, `onPrimary`, `background`, `error`, `outline`). Consumed only by `MaterialTheme(colorScheme = ...)` inside the `<THEME>` composable.

Dimensions (`Dimens`) and typography (`Type`) sit outside the color layers — no CompositionLocal, no light/dark variants; a single set of tokens applies to the whole app.

**Naming note** — the `<THEME>` placeholder maps to a single Kotlin identifier used both as the theme composable and as the accessor object (same pattern as Material3's `MaterialTheme { }` composable + `MaterialTheme` object). If your project prefers separate names (e.g. `AppTheme` composable + `AppTokens` accessor), rename after substitution.

---

## File 1 — `Theme.kt`

The theme composable + accessor object. `MainActivity` wraps its content in `<THEME> { AppNavHost(...) }`; every UI file reads tokens through the accessor.

Path: `<PKG_ROOT>/presentation/ui/theme/Theme.kt`

```kotlin
package <PKG_ROOT>.presentation.ui.theme

import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.MaterialTheme
import androidx.compose.runtime.Composable
import androidx.compose.runtime.CompositionLocalProvider
import <PKG_ROOT>.presentation.ui.theme.colors.DarkMaterialColorScheme
import <PKG_ROOT>.presentation.ui.theme.colors.ExtendedColors
import <PKG_ROOT>.presentation.ui.theme.colors.LightMaterialColorScheme
import <PKG_ROOT>.presentation.ui.theme.colors.LocalExtendedColors
import <PKG_ROOT>.presentation.ui.theme.colors.darkAppColors
import <PKG_ROOT>.presentation.ui.theme.colors.lightAppColors

@Composable
fun <THEME>(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit,
) {
    val extendedColors = if (darkTheme) darkAppColors else lightAppColors
    val materialColorScheme = if (darkTheme) DarkMaterialColorScheme else LightMaterialColorScheme

    CompositionLocalProvider(LocalExtendedColors provides extendedColors) {
        MaterialTheme(
            colorScheme = materialColorScheme,
            typography = Typography,
            content = content,
        )
    }
}

object <THEME> {
    val colors: ExtendedColors
        @Composable get() = LocalExtendedColors.current

    val spacing get() = Dimens.Spacing
    val padding get() = Dimens.Padding
    val cornerRadius get() = Dimens.CornerRadius
    val iconSize get() = Dimens.IconSize
    val borderWidth get() = Dimens.BorderWidth
}
```

The composable and object intentionally share the name — Kotlin resolves `<THEME> { ... }` as the function call and `<THEME>.colors` as the object member. Rename `lightAppColors` / `darkAppColors` to match your project (e.g. `lightAcmeColors`) before or after substitution.

---

## File 2 — `Type.kt`

Typography scaffold. Consumed by `MaterialTheme(typography = Typography, ...)` inside `<THEME>`. UI code reads typography through `MaterialTheme.typography.<style>` — this is the **only** valid `MaterialTheme.*` accessor per `rules/theming-and-tokens.md`.

Path: `<PKG_ROOT>/presentation/ui/theme/Type.kt`

```kotlin
package <PKG_ROOT>.presentation.ui.theme

import androidx.compose.material3.Typography
import androidx.compose.ui.text.TextStyle
import androidx.compose.ui.text.font.FontFamily
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.unit.sp

val Typography =
    Typography(
        bodyLarge = TextStyle(
            fontFamily = FontFamily.Default,
            fontWeight = FontWeight.Normal,
            fontSize = 16.sp,
            lineHeight = 24.sp,
            letterSpacing = 0.5.sp,
        ),
        // Fill in the other Material3 styles your app uses:
        // displayLarge, displayMedium, displaySmall,
        // headlineLarge, headlineMedium, headlineSmall,
        // titleLarge, titleMedium, titleSmall,
        // bodyMedium, bodySmall,
        // labelLarge, labelMedium, labelSmall
    )
```

---

## File 3 — `Dimens.kt`

Dimension tokens grouped by category. `<THEME>` exposes each nested object through a get-only property (`<THEME>.spacing`, `<THEME>.padding`, etc.), so UI code reads `<THEME>.spacing.md` — plain `object` access, no CompositionLocal.

Path: `<PKG_ROOT>/presentation/ui/theme/Dimens.kt`

The Figma-token comments (`// Spacing/spacing-xs`) pin each value to its source-of-truth entry in the design system — makes it obvious in review when a value has drifted from Figma. Drop them if your project doesn't publish a token catalog.

```kotlin
package <PKG_ROOT>.presentation.ui.theme

import androidx.compose.ui.unit.dp

object Dimens {
    object Spacing {
        val none = 0.dp   // Spacing/spacing-none
        val xxs = 2.dp    // Spacing/spacing-xxs
        val xs = 4.dp     // Spacing/spacing-xs
        val sm = 8.dp     // Spacing/spacing-sm
        val md = 12.dp    // Spacing/spacing-md
        val lg = 16.dp    // Spacing/spacing-lg
        val xl = 24.dp    // Spacing/spacing-xl
        val xxl = 32.dp   // Spacing/spacing-xxl
        val xxxl = 40.dp  // Spacing/spacing-3xl
        val xxxxl = 64.dp // Spacing/spacing-4xl
    }

    object Padding {
        val none = 0.dp   // Padding/padding-none
        val xxs = 8.dp    // Padding/padding-xxs
        val xs = 12.dp    // Padding/padding-xs
        val sm = 16.dp    // Padding/padding-sm
        val md = 20.dp    // Padding/padding-md
        val lg = 24.dp    // Padding/padding-lg
        val xl = 32.dp    // Padding/padding-xl
        val xxl = 40.dp   // Padding/padding-xxl
        val xxxl = 48.dp  // Padding/padding-3xl
        val xxxxl = 64.dp // Padding/padding-4xl
    }

    object CornerRadius {
        val none = 0.dp   // Radius/radius-none
        val xs = 2.dp     // Radius/radius-xs
        val sm = 4.dp     // Radius/radius-sm
        val md = 8.dp     // Radius/radius-md
        val lg = 12.dp    // Radius/radius-lg
        val xl = 16.dp    // Radius/radius-xl
        val xxl = 24.dp   // Radius/radius-xxl
        val full = 400.dp // Radius/radius-full — pill shape
    }

    object IconSize {
        val xxs = 12.dp
        val xs = 16.dp
        val sm = 20.dp
        val md = 24.dp
        val lg = 32.dp
        val xl = 40.dp
        val xxl = 48.dp
        val xxxl = 54.dp
    }

    object BorderWidth {
        val xs = 1.dp
    }
}
```

**Spacing vs Padding** — spacing is between sibling elements (gap between two cards); padding is inside a container (space between the card's edge and its content). Same unit, different semantic — keep them separate so review can catch "wait, why 24dp padding on a small chip?" without a table lookup.

---

## File 4 — `colors/PrimitiveColors.kt`

The raw palette. **`internal` — never referenced from UI code.** UI reads `<THEME>.colors.<semanticName>` via the extended layer. This file is consumed only by `ExtendedColors` implementations (File 5) and the Material color schemes (Files 6–7).

Path: `<PKG_ROOT>/presentation/ui/theme/colors/PrimitiveColors.kt`

Populate from your project's design system. The skeleton below shows the **family-with-shades pattern** — one family per color role (slate, gray, primary, red, …), 5–11 shades each on a `50 / 100 / 200 / … / 900 / 950` scale. Fill in the actual hex values from your Figma color styles.

```kotlin
package <PKG_ROOT>.presentation.ui.theme.colors

import androidx.compose.ui.graphics.Color

internal object PrimitiveColors {
    // Slate — neutral cool grays
    val slate50 = Color(0xFFF8FAFC)
    val slate100 = Color(0xFFF1F5F9)
    val slate500 = Color(0xFF64748B)
    val slate900 = Color(0xFF0F172A)
    val slate950 = Color(0xFF020617)
    // ... slate200, slate300, slate400, slate600, slate700, slate800 as needed

    // Gray — neutral warm grays (only if distinct from slate in your palette)
    val gray50 = Color(0xFFF9FAFB)
    val gray500 = Color(0xFF6B7280)
    val gray900 = Color(0xFF111827)
    // ...

    // Primary — brand color
    val primary50 = Color(0xFFEFF6FF)
    val primary500 = Color(0xFF3B82F6)
    val primary700 = Color(0xFF1D4ED8)
    // ...

    // Secondary, Tertiary — additional brand roles if your design system has them
    val secondary500 = Color(0xFF14B8A6)
    val tertiary500 = Color(0xFFF59E0B)

    // Semantic — error, success, warning
    val red500 = Color(0xFFEF4444)
    val green500 = Color(0xFF22C55E)
    val amber500 = Color(0xFFF59E0B)

    // Pure
    val white = Color(0xFFFFFFFF)
    val black = Color(0xFF000000)
    val transparent = Color(0x00000000)
}
```

**Naming rule** — primitives are named by *appearance* (`slate500`, `primary500`), not by *use* (`buttonBackground`, `dangerBorder`). Semantics belong in `ExtendedColors`.

---

## File 5 — `colors/ExtendedColors.kt`

The semantic layer. Three parts in one file: (a) the `LocalExtendedColors` CompositionLocal, (b) the `ExtendedColors` interface, (c) the light + dark implementations.

Path: `<PKG_ROOT>/presentation/ui/theme/colors/ExtendedColors.kt`

Semantic tokens describe *what a color means*, not what shade it is — the same token has different primitive values in light vs dark. Group by section (`// Background`, `// Text`, `// Border`, …) and use state-variant suffixes (`Default`, `DefaultHover`, `Light`, `LightHover`) so the token catalog stays predictable.

```kotlin
package <PKG_ROOT>.presentation.ui.theme.colors

import androidx.compose.runtime.compositionLocalOf
import androidx.compose.ui.graphics.Color

val LocalExtendedColors = compositionLocalOf<ExtendedColors> {
    error("No ExtendedColors provided — is your content wrapped in the <THEME> composable?")
}

interface ExtendedColors {
    // Background — surfaces the UI paints on
    val backgroundDefault: Color
    val backgroundMuted: Color
    val backgroundCard: Color

    // Background — Primary (brand)
    val backgroundPrimaryDefault: Color
    val backgroundPrimaryDefaultHover: Color
    val backgroundPrimaryLight: Color
    val backgroundPrimaryLightHover: Color

    // Background — Destructive (errors, delete confirmations)
    val backgroundDestructiveDefault: Color
    val backgroundDestructiveDefaultHover: Color

    // Text
    val textDefault: Color
    val textMuted: Color
    val textPrimary: Color
    val textDestructive: Color

    // Border
    val borderDefault: Color
    val borderMuted: Color
    val borderPrimary: Color

    // Icon
    val iconDefault: Color
    val iconMuted: Color
    val iconPrimary: Color

    // Transparent — for explicit "no color" cases
    val transparent: Color
}

// Light implementation
internal val lightAppColors: ExtendedColors = object : ExtendedColors {
    override val backgroundDefault = PrimitiveColors.white
    override val backgroundMuted = PrimitiveColors.slate50
    override val backgroundCard = PrimitiveColors.white

    override val backgroundPrimaryDefault = PrimitiveColors.primary500
    override val backgroundPrimaryDefaultHover = PrimitiveColors.primary700
    override val backgroundPrimaryLight = PrimitiveColors.primary50
    override val backgroundPrimaryLightHover = PrimitiveColors.primary50 // adjust to a slightly darker step

    override val backgroundDestructiveDefault = PrimitiveColors.red500
    override val backgroundDestructiveDefaultHover = PrimitiveColors.red500 // adjust to a red700

    override val textDefault = PrimitiveColors.slate900
    override val textMuted = PrimitiveColors.slate500
    override val textPrimary = PrimitiveColors.primary700
    override val textDestructive = PrimitiveColors.red500

    override val borderDefault = PrimitiveColors.slate100
    override val borderMuted = PrimitiveColors.slate50
    override val borderPrimary = PrimitiveColors.primary500

    override val iconDefault = PrimitiveColors.slate900
    override val iconMuted = PrimitiveColors.slate500
    override val iconPrimary = PrimitiveColors.primary500

    override val transparent = PrimitiveColors.transparent
}

// Dark implementation — mirror with inverted primitives
internal val darkAppColors: ExtendedColors = object : ExtendedColors {
    override val backgroundDefault = PrimitiveColors.slate950
    override val backgroundMuted = PrimitiveColors.slate900
    override val backgroundCard = PrimitiveColors.slate900

    override val backgroundPrimaryDefault = PrimitiveColors.primary500
    override val backgroundPrimaryDefaultHover = PrimitiveColors.primary700
    override val backgroundPrimaryLight = PrimitiveColors.slate900
    override val backgroundPrimaryLightHover = PrimitiveColors.slate900

    override val backgroundDestructiveDefault = PrimitiveColors.red500
    override val backgroundDestructiveDefaultHover = PrimitiveColors.red500

    override val textDefault = PrimitiveColors.white
    override val textMuted = PrimitiveColors.slate500
    override val textPrimary = PrimitiveColors.primary500
    override val textDestructive = PrimitiveColors.red500

    override val borderDefault = PrimitiveColors.slate900
    override val borderMuted = PrimitiveColors.slate900
    override val borderPrimary = PrimitiveColors.primary500

    override val iconDefault = PrimitiveColors.white
    override val iconMuted = PrimitiveColors.slate500
    override val iconPrimary = PrimitiveColors.primary500

    override val transparent = PrimitiveColors.transparent
}
```

**Adding a new token** — declare it in the `interface`, populate both `lightAppColors` and `darkAppColors`, then reference it from UI as `<THEME>.colors.<newToken>`. If a semantic token needs a shade that doesn't exist in the primitive palette yet, add the primitive first.

---

## File 6 — `colors/LightMaterialColorScheme.kt`

Thin mapping from primitives to Material3 roles. Consumed only by the `MaterialTheme(colorScheme = ...)` call inside `<THEME>`. UI code doesn't read from here — the extended layer covers everything except when Material3 components (e.g. `Button`, `Card`) need their built-in `colorScheme`-based defaults.

Path: `<PKG_ROOT>/presentation/ui/theme/colors/LightMaterialColorScheme.kt`

```kotlin
package <PKG_ROOT>.presentation.ui.theme.colors

import androidx.compose.material3.lightColorScheme

internal val LightMaterialColorScheme = lightColorScheme(
    primary = PrimitiveColors.primary500,
    onPrimary = PrimitiveColors.white,
    secondary = PrimitiveColors.secondary500,
    onSecondary = PrimitiveColors.white,
    tertiary = PrimitiveColors.tertiary500,
    onTertiary = PrimitiveColors.white,
    background = PrimitiveColors.white,
    onBackground = PrimitiveColors.slate950,
    surface = PrimitiveColors.white,
    onSurface = PrimitiveColors.slate950,
    error = PrimitiveColors.red500,
    onError = PrimitiveColors.white,
    outline = PrimitiveColors.slate100,
)
```

---

## File 7 — `colors/DarkMaterialColorScheme.kt`

Dark mirror. Same roles, inverted primitives.

Path: `<PKG_ROOT>/presentation/ui/theme/colors/DarkMaterialColorScheme.kt`

```kotlin
package <PKG_ROOT>.presentation.ui.theme.colors

import androidx.compose.material3.darkColorScheme

internal val DarkMaterialColorScheme = darkColorScheme(
    primary = PrimitiveColors.primary500,
    onPrimary = PrimitiveColors.white,
    secondary = PrimitiveColors.secondary500,
    onSecondary = PrimitiveColors.white,
    tertiary = PrimitiveColors.tertiary500,
    onTertiary = PrimitiveColors.white,
    background = PrimitiveColors.slate950,
    onBackground = PrimitiveColors.white,
    surface = PrimitiveColors.slate950,
    onSurface = PrimitiveColors.white,
    error = PrimitiveColors.red500,
    onError = PrimitiveColors.white,
    outline = PrimitiveColors.slate900,
)
```

---

## Verification after copying

Once these files exist in a project:

- `presentation/ui/theme/Theme.kt` — `<THEME>` composable + accessor object
- `presentation/ui/theme/Type.kt` — `val Typography = Typography(...)`
- `presentation/ui/theme/Dimens.kt` — `object Dimens { Spacing, Padding, CornerRadius, IconSize, BorderWidth }`
- `presentation/ui/theme/colors/PrimitiveColors.kt` — `internal object PrimitiveColors` — raw palette
- `presentation/ui/theme/colors/ExtendedColors.kt` — `interface ExtendedColors` + `LocalExtendedColors` + `lightAppColors` + `darkAppColors`
- `presentation/ui/theme/colors/LightMaterialColorScheme.kt` — `internal val LightMaterialColorScheme = lightColorScheme(...)`
- `presentation/ui/theme/colors/DarkMaterialColorScheme.kt` — `internal val DarkMaterialColorScheme = darkColorScheme(...)`

`MainActivity` wraps its Compose content in `<THEME> { AppNavHost(navController, startDestination) }`. UI code reads tokens through `<THEME>.colors.<x>` / `<THEME>.spacing.<x>` / `<THEME>.padding.<x>` / `<THEME>.cornerRadius.<x>` / `<THEME>.iconSize.<x>` / `<THEME>.borderWidth.<x>`, and typography through `MaterialTheme.typography.<style>`. See `rules/theming-and-tokens.md` for the full usage rules and violation list.
