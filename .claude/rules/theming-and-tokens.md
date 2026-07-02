---
name: theming-and-tokens
description: Design tokens come from the project theme object, not MaterialTheme or hardcoded literals
paths:
  - "**/presentation/**/*.kt"
---

# Theming & design tokens

Source every design value from `<THEME>`. `MaterialTheme` is banned in UI code with a single exception.

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
