---
name: theming-and-tokens
description: Design tokens come from the project theme object, not MaterialTheme or hardcoded literals
paths:
  - "**/presentation/**/*.kt"
---

# Theming & design tokens

Source every design value from `<THEME>`. `MaterialTheme` is banned in UI code with a single exception.

**Setting up the theme in a new project:** invoke the `theme-primitives` skill (`.claude/skills/theme-primitives/SKILL.md`). It ships the reference files this rule assumes exist — `Theme.kt`, `Type.kt`, `Dimens.kt`, and the three-file color layer under `colors/`.

| Need | Use | Never |
|---|---|---|
| Colors | `<THEME>.colors.<token>` | `MaterialTheme.colorScheme.*`, `Color(...)`, `Color.Red` |
| Spacing (between siblings) | `<THEME>.spacing.<size>` | Inline `.dp` |
| Padding (inside a container) | `<THEME>.padding.<size>` | Inline `.dp` |
| Corner radius | `<THEME>.cornerRadius.<size>` | `RoundedCornerShape(8.dp)`, `MaterialTheme.shapes.*` |
| Icon size | `<THEME>.iconSize.<size>` | Inline `.dp` |
| Border width | `<THEME>.borderWidth.<size>` | Inline `.dp` |
| Typography | `MaterialTheme.typography.<style>` | Inline `TextStyle(...)` |

**The only valid `MaterialTheme` use is `MaterialTheme.typography.*`.** Anything else is a violation.

## When a value has no token

If a `.dp`, `Int`, or `Float` is genuinely one-off (used exactly once, nothing similar exists in `<THEME>`), declare it as a private file-level constant at the top of the file:

```kotlin
private val FAB_ELEVATION = 6.dp

@Composable
fun MyButton() {
    Box(modifier = Modifier.shadow(FAB_ELEVATION)) { ... }
}
```

Inlining `6.dp` inside the composable is a violation. Naming the constant lets it show up in review as "should this be a token?" instead of hiding as an unnamed magic number.

## Extending the theme

If the same literal appears twice in the codebase, promote it to a token:

1. Add the field to the relevant token dataclass (e.g. `AppSpacing`, `AppCornerRadius`)
2. Populate light and dark values in `<THEME>` initializers
3. Update both usages to consume the new token

## Structure

The theme has **three color layers**, ordered by distance from the UI:

1. **Primitive** (`colors/PrimitiveColors.kt`, `internal`) — the raw palette (`slate500`, `primary500`, `red500`, …). Never referenced from UI code.
2. **Extended** (`colors/ExtendedColors.kt`) — `interface ExtendedColors` with semantic tokens (`backgroundPrimaryDefault`, `textDefault`, `borderMuted`, …) and its light + dark implementations. This is the app's color surface — everything the UI reads passes through here.
3. **Material** (`colors/{Light,Dark}MaterialColorScheme.kt`, `internal`) — thin mapping from primitives to Material3 roles (`primary`, `onPrimary`, `background`, `error`, `outline`). Consumed only by `MaterialTheme(colorScheme = ...)` inside the `<THEME>` composable; UI code never reads from here.

**Dimensions and typography sit outside the color layers.** `Dimens` is a plain `object` with nested `Spacing`/`Padding`/`CornerRadius`/`IconSize`/`BorderWidth` sub-objects — no CompositionLocal needed because dimensions don't change with dark mode. Typography flows through `MaterialTheme.typography.*` (the only permitted `MaterialTheme.*` accessor).

**Accessor pattern** — `<THEME>` is both a `@Composable fun` (the theme wrapper) and an `object` (the token accessor), same shape as Material3's `MaterialTheme`. Colors go through a `@Composable get()` that reads `LocalExtendedColors.current`; dimensions are plain `object` references. See the `theme-primitives` skill for the reference file layout.

**Adding a new color token** — add the field to `ExtendedColors`, populate both light and dark implementations, and update usages. Adding a new primitive is only necessary when the new semantic token needs a shade that doesn't already exist in the palette.

## Common violations

```kotlin
// ❌ MaterialTheme.colorScheme
Text("Hi", color = MaterialTheme.colorScheme.primary)

// ❌ Hardcoded color
Text("Hi", color = Color(0xFFFF0000))

// ❌ Named Color constant
Text("Hi", color = Color.Red)

// ❌ Inline .dp with no constant
Spacer(modifier = Modifier.height(24.dp))

// ❌ MaterialTheme.shapes
Card(shape = MaterialTheme.shapes.medium) { }

// ✅ Correct
Text("Hi", color = <THEME>.colors.primary)
Spacer(modifier = Modifier.height(<THEME>.spacing.large))
Card(shape = RoundedCornerShape(<THEME>.cornerRadius.medium)) { }
```

- `PrimitiveColors.slate500` referenced directly from a `@Composable` → route through an `ExtendedColors` semantic token; UI code never reads primitives directly. If the semantic token you need doesn't exist yet, add it to `ExtendedColors` (and both light/dark implementations).
