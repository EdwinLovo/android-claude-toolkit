---
name: icons
description: All icons come from the central <ICONS> registry; never Material Icons; add drawable + register
paths:
  - "**/presentation/**/*.kt"
---

# Icons

All icons come from `<ICONS>` (`presentation/utils/<ICONS>.kt`). **`androidx.compose.material.icons.Icons.*` is banned** in UI code — the goal is one shared visual vocabulary controlled by design, not the Material default set.

## Adding a new icon

1. Get the SVG or Vector Drawable XML from design.
2. Save it as `app/src/main/res/drawable/<icon_name>.xml`.
3. Register it in `<ICONS>`:

```kotlin
// presentation/utils/<ICONS>.kt
object <ICONS> {
    val Add = R.drawable.ic_add
    val Delete = R.drawable.ic_delete
    val CheckCircle = R.drawable.ic_check_circle
    val Search = R.drawable.ic_search
    // ...
}
```

Names in `<ICONS>` are PascalCase and describe the icon, not the use case (`Trash` not `RemoveItemIcon`).

## Using an icon

**Tinted vector icon** — use `Icon()` with `painterResource(<ICONS>.X)`:

```kotlin
Icon(
    painter = painterResource(<ICONS>.Delete),
    contentDescription = stringResource(R.string.delete),
    tint = <THEME>.colors.onSurface,
)
```

`Icon` applies the tint, so the vector should typically have `android:tint="?attr/colorControlNormal"` or a solid black fill so the tint takes effect.

**Full-color raster / PNG** — use `Image()` (no tint):

```kotlin
Image(
    painter = painterResource(<ICONS>.LogoFull),
    contentDescription = stringResource(R.string.app_logo),
)
```

## Content descriptions

Icons that convey meaning have a non-null `contentDescription` sourced from `stringResource`. Purely decorative icons (icon next to a labeled text button) pass `contentDescription = null`:

```kotlin
// Meaningful — needed by TalkBack
Icon(painter = painterResource(<ICONS>.Delete), contentDescription = stringResource(R.string.delete))

// Decorative — a text label already exists next to it
Row {
    Icon(painter = painterResource(<ICONS>.Add), contentDescription = null)
    Text(text = stringResource(R.string.add))
}
```

## Size

Prefer `<THEME>.iconSize.<size>` over inline `.dp`:

```kotlin
Icon(
    painter = painterResource(<ICONS>.Search),
    contentDescription = null,
    modifier = Modifier.size(<THEME>.iconSize.medium),
)
```

## Hard rules

- **No `Icons.Default.*`, `Icons.Filled.*`, `Icons.Outlined.*`, `Icons.Rounded.*`** — grep for `androidx.compose.material.icons.` and remove
- **No `androidx.compose.material.icons.Icons` import** — banned in `presentation/`
- **Every new drawable is registered in `<ICONS>`** — inline `painterResource(R.drawable.ic_x)` scattered around the codebase is a violation; use `<ICONS>.X`
- **Content description from `stringResource` or explicitly `null`** — never a literal string
- **Size via `<THEME>.iconSize.<size>`** — inline `24.dp` is a violation (see `rules/theming-and-tokens.md`)

## Common violations

- `Icon(imageVector = Icons.Default.Delete, ...)` → replace with `Icon(painter = painterResource(<ICONS>.Delete), ...)`
- `painterResource(R.drawable.ic_delete)` at a call site with no `<ICONS>` entry → add `Delete = R.drawable.ic_delete` to `<ICONS>` and reference `<ICONS>.Delete`
- `contentDescription = "Delete"` → `contentDescription = stringResource(R.string.delete)`
- Meaningful icon with `contentDescription = null` → add a resource; TalkBack users can't perceive it otherwise
- `Icon` used for a colored raster (loses colors to tint) → use `Image` instead
