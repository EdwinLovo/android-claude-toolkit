# android-claude-toolkit

A drop-in `.claude/` folder plus a `CLAUDE.md` template that teaches Claude Code the conventions of an opinionated Android/Kotlin project — Jetpack Compose + MVVM + Clean Architecture + Hilt.

Extracted from a real production Android POS app, then abstracted with placeholders so it grafts onto any project.

## What's inside

```
android-claude-toolkit/
├── CLAUDE.md                              # Always-loaded universal standards
└── .claude/
    ├── rules/                             # Path-scoped conventions (auto-loaded via glob)
    │   ├── folder-structure.md
    │   ├── theming-and-tokens.md
    │   ├── viewmodels.md
    │   ├── contracts.md
    │   ├── delegates.md
    │   ├── screens.md
    │   ├── composables.md
    │   ├── previews.md
    │   ├── dialogs.md
    │   ├── navigation.md
    │   ├── lifecycle.md
    │   ├── repositories.md
    │   ├── data-sources.md
    │   ├── use-cases.md
    │   ├── mappers.md
    │   ├── models.md
    │   ├── dependency-injection.md
    │   ├── error-handling.md
    │   ├── string-resources.md
    │   ├── icons.md
    │   ├── constants-in-viewmodels.md
    │   ├── extensions.md
    │   ├── utils.md
    │   ├── logging.md
    │   └── testing.md
    ├── skills/                            # On-demand task workflows
    │   ├── screen-anatomy/SKILL.md
    │   ├── data-feature/SKILL.md
    │   ├── navigation-primitives/SKILL.md # First-time setup: NavigationRoute, NavigationAction, ViewModelNav
    │   ├── repository-primitives/SKILL.md # First-time setup: BaseRepository, ApiResult, BasePagingSource
    │   ├── theme-primitives/SKILL.md      # First-time setup: Theme composable + accessor, Dimens, PrimitiveColors, ExtendedColors, Material schemes
    │   └── audit-branch/SKILL.md          # context: fork — runs the audit in an isolated sub-agent
    └── commands/                          # Slash commands
        ├── new-screen.md
        ├── new-data-feature.md
        ├── audit-branch.md
        ├── pr-template.md
        ├── add-tests.md
        └── update-docs.md
```

Layout follows the Anthropic Claude Code guidelines:

- **CLAUDE.md** — always loaded, holds universal standards
- **`.claude/rules/*.md`** — auto-loaded when editing files matching each rule's `paths:` glob (preferred over subdirectory CLAUDE.md files for conventions that span the codebase)
- **`.claude/skills/<name>/SKILL.md`** — on-demand, invoked by name or when Claude decides they're relevant
- **`.claude/commands/*.md`** — user-typed slash commands (e.g. `/new-screen`)

## Install into an Android project

```bash
# From your target project root
cp -R /path/to/android-claude-toolkit/.claude .
cp /path/to/android-claude-toolkit/CLAUDE.md .

# Then substitute placeholders (see table below)
```

You can automate the substitution with a one-liner:

```bash
# macOS / BSD sed
grep -RlE '<(PROJECT_NAME|PKG_ROOT|APP_ID|THEME|ICONS|SCAFFOLD|DIALOG|PREVIEW|TICKET_PREFIX|NET_LIB)>' .claude CLAUDE.md \
  | xargs sed -i '' \
    -e 's|<PROJECT_NAME>|MyApp|g' \
    -e 's|<PKG_ROOT>|com.example.myapp|g' \
    -e 's|<APP_ID>|com.example.myapp|g' \
    -e 's|<THEME>|AppTheme|g' \
    -e 's|<ICONS>|AppIcons|g' \
    -e 's|<SCAFFOLD>|AppScaffold|g' \
    -e 's|<DIALOG>|AppDialogContainer|g' \
    -e 's|<PREVIEW>|@AppPreview|g' \
    -e 's|<TICKET_PREFIX>|APP-XXX|g' \
    -e 's|<NET_LIB>|Retrofit|g'
```

## Placeholders

Every file uses these tokens instead of hardcoded names. Replace them with your project's values.

| Placeholder | Meaning | Example |
|---|---|---|
| `<PROJECT_NAME>` | Human-readable project name | `MyApp` |
| `<PKG_ROOT>` | Kotlin package root | `com.example.myapp` |
| `<APP_ID>` | Android application ID | `com.example.myapp` |
| `<THEME>` | Compose theme object name | `AppTheme` |
| `<ICONS>` | Icons registry object | `AppIcons` |
| `<SCAFFOLD>` | Scaffold wrapper composable | `AppScaffold` |
| `<DIALOG>` | Dialog container composable | `AppDialogContainer` |
| `<PREVIEW>` | Preview annotation | `@AppPreview` (component) / `@AppScreenPreview` (screen) |
| `<TICKET_PREFIX>` | VCS ticket prefix pattern | `APP-XXX`, `PROJ-XXX`, `JIRA-XXX` |
| `<NET_LIB>` | Primary networking library | `Retrofit`, `Apollo`, `Ktor` |

The angle-bracket form is intentional so `sed 's|<THEME>|AppTheme|g'` is safe — no other file content matches.

## Assumptions this toolkit encodes

These are the opinions baked into the rules. If your project disagrees with any of them, edit the affected rule file — that's the point of copying the folder in.

- **Architecture**: single-module clean architecture (`domain / data / presentation`) with Hilt
- **UI**: Jetpack Compose + Material 3 + MVVM
- **State**: `StateFlow<UiState>` with Kotlin's explicit backing field feature (`-Xexplicit-backing-fields`); no `_uiState` prefix
- **Screens**: two-composable split — entry `<Screen>Screen()` + stateless `<Screen>ScreenContent()`; state and events dispatched via `handleEvent`
- **Contracts pattern**: `contracts/<Screen>Contract.kt` holds `<Screen>UiState` + `interface <Screen>Event`
- **Delegates pattern**: `@ViewModelScoped` delegates own their own `StateFlow` for isolated sub-flows
- **Theming**: hardcoded `Color(...)` / `.dp` / `MaterialTheme.colorScheme` are violations; everything comes from a project theme object (`<THEME>.colors`, `<THEME>.spacing`, `<THEME>.cornerRadius`, ...)
- **Errors**: a global `ErrorEventBus` receives typed `UiText` messages from repositories, rendered by the top-level scaffold
- **Data layer**: data sources are **factories** (return call objects, no execution); repositories execute + map; mappers live in `data/mappers/` (Apollo↔domain) or `domain/model/` (domain-only)
- **Use cases**: only when logic is non-trivial or reused across ViewModels (Google's Domain Layer guidance)

## Syncing updates

Recommended: add the toolkit as a git subtree in each downstream project so you can pull upstream fixes.

```bash
# In your downstream project
git subtree add --prefix=.claude-toolkit https://github.com/<you>/android-claude-toolkit main --squash
# then hardlink or copy .claude-toolkit/.claude → .claude, .claude-toolkit/CLAUDE.md → CLAUDE.md

# To sync later
git subtree pull --prefix=.claude-toolkit https://github.com/<you>/android-claude-toolkit main --squash
```

## Contributing back

When a new convention emerges in a downstream project that would benefit every project, port it back here:

1. Edit the relevant `.claude/rules/*.md` (or add a new one)
2. Sanity-check placeholders — the rule must not contain any project-specific literals
3. Update this README's "Assumptions" section if the change is a new opinion

## Not included

- **`tech-android`** — a generic Android/Kotlin/Compose skill (Clean Arch, Compose state, coroutines, testing, DI, accessibility, permissions, networking, Room). It lives separately and is treated as an external dependency.
- **Language-agnostic patterns** — this toolkit is Android/Kotlin/Compose-specific by design. Team decisions (specific manager classes, deployment infra, etc.) belong in each project's own `CLAUDE.md`.
