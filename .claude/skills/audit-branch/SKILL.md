---
name: audit-branch
description: Audit Kotlin source changes in the current branch against project standards from .claude/rules/. Runs in an isolated sub-agent so findings are not biased by the current conversation. Use when the user asks to audit a branch, review changes for convention violations, or check work before opening a PR.
allowed-tools:
  - Read
  - Bash
  - Grep
  - Glob
context: fork
---

# audit-branch

Audit all Kotlin source changes in the current branch against the project's coding standards. **Enter plan mode immediately** — the audit investigation and findings ARE the plan. Do not make any edits; only read and report.

The authoritative standards live in:
- `CLAUDE.md` (universal rules)
- `.claude/rules/*.md` (path-scoped conventions — auto-loaded when you read matching files)

You are running in a **forked context** (`context: fork` in this skill's frontmatter). You have no memory of the parent conversation. Read the code cold — treat every finding as a first-time observation. Do not assume any violation is "already known" or "already decided".

---

## Steps

### 1 — Enter plan mode, then identify changed source files

Enter plan mode first. Then run:
```
git diff main...HEAD --name-only
```

Keep only files matching `app/src/main/**/*.kt`.

Skip: `*Contract.kt`, `*Route.kt`, `*Module.kt`, resource / build / GraphQL files — audited elsewhere or no logic to audit.

---

### 2 — Read and audit each file

Read every file in the list. As you open each file, the path-scoped rule for that file type auto-loads (see `.claude/rules/`). Cross-check the file against the loaded rule set.

Record every violation with its file path and one-line description.

#### Rule checks (grouped by area — each maps to a rule file)

**Theme & colors** (`rules/theming-and-tokens.md`)
- `MaterialTheme.colorScheme.*` → must be `<THEME>.colors.*`
- `MaterialTheme.shapes.*` → must be `<THEME>.cornerRadius.*`
- Hardcoded `Color(...)` or `Color.Red` → must be `<THEME>.colors.*`
- Inline `.dp` / Int / Float with no private constant → declare as `private val NAME = X.dp` at file level

Exception: `MaterialTheme.typography.*` is correct.

**String resources** (`rules/string-resources.md`)
- Literal strings in composables (`Text("Submit")`) → must use `stringResource(R.string.xxx)`
- Enum with a `String` label → change to `@param:StringRes val labelRes: Int`

**Icons** (`rules/icons.md`)
- `Icons.*` from Material → must use `<ICONS>.*` + `painterResource()`

**Scaffold & dialogs** (`rules/dialogs.md`)
- `Scaffold { }` used directly → must use `<SCAFFOLD>`
- Two scaffolds nested → never nest
- Dialog `@Composable` not wrapped in `<DIALOG>` as the outermost wrapper

**Component file placement** (`rules/composables.md`)
- More than one non-preview `@Composable` in a single file → one component per file
- `private @Composable` that is not a `*Preview()` or `*ScreenContent` → extract to its own file, mark `internal`, add its own preview
- Feature-specific component in `presentation/ui/components/` → move to `ux/<feature>/components/`
- Reusable component in `ux/<feature>/` → move to `ui/components/<category>/`

**ViewModel & UiState** (`rules/viewmodels.md`)
- `_uiState` underscore prefix → use explicit backing field
- `MutableStateFlow` exposed publicly → expose `StateFlow<T>` only
- `collectAsState()` → must be `collectAsStateWithLifecycle()`
- Class extends `BaseViewModel` → ViewModels extend `ViewModel()` directly
- `uiState.value = ...` → use `uiState.reduce { copy(...) }`

**Contracts structure** (`rules/contracts.md`)
- Standalone `<Screen>Event.kt` / `<Screen>UiState.kt` → move to `contracts/<Screen>Contract.kt`
- `events/` subfolder → use `contracts/` only

**Data layer** (`rules/repositories.md`, `rules/data-sources.md`, `rules/mappers.md`, `rules/models.md`)
- Network-lib type imported in `domain/` or `presentation/` → violation
- Repository interface method with network type in signature → return domain types
- Repository impl calls network client directly (without `safeCallSuspend` / `safeApolloCallSuspend`) → must use `BaseRepository` helpers
- Mapper logic inline in `RepositoryImpl` → move to `data/mappers/<feature>/` as extension function
- Domain-only mapper placed under `data/mappers/` → move to `domain/model/<area>/<Name>Mappers.kt`

**Logging** (`rules/logging.md`)
- `android.util.Log.*` → must use `timber.log.Timber`
- `TAG` constant for logging → delete, Timber doesn't need it

**Null safety**
- `!!` operator → replace with smart cast, `?: return`, `let`, or `checkNotNull` with a message

**Coroutines**
- `GlobalScope` → use `viewModelScope` or a delegate-injected `CoroutineScope`

**Previews** (`rules/previews.md`)
- `@Composable` function with no `<PREVIEW>` / `<PREVIEW>Screen` → add a preview at the bottom of the file

**VM constants** (`rules/constants-in-viewmodels.md`)
- File-level `private const val` in a ViewModel file → move into a `companion object` at the bottom
- Inline magic literals inside the class body → declare in the companion with a name

#### Bad practices & improvements (advisory)

After the rule checks, scan for general code-quality smells. Not strict violations — flag as suggestions with a one-line rationale.

- **Duplicate logic** — repeated formatting, predicates, `when` mappings → shared extension or constant
- **Dead code** — unused imports, unused strings, unused private helpers, unnecessary `else` branches
- **Magic literals** — same `.dp`/`Int`/`String` appearing more than once → file-level `private val` / `const val`
- **Compose smells** — inline lambdas in hot paths, `remember { }` of a stable value, `LazyColumn`/`LazyRow` missing `key = ...`
- **State/flow smells** — missing `distinctUntilChanged`, unnecessary `flatMapLatest`, mutable shared state passed by reference
- **Modifier hygiene** — repeated `Modifier` chains, `Modifier` parameter not passed through
- **Readability** — deeply nested layouts, long parameter lists, cryptic identifiers

Default severity: **Low**. Promote to **Medium** if the smell measurably hurts the file (visible recomposition cost, real maintenance trap, duplicate already inconsistent).

---

### 3 — Compile findings

Group by severity:

**High** — breaks architecture contracts or risks bugs:
- Network types crossing domain boundary
- `!!` operator
- `GlobalScope`
- Repository impl bypassing `BaseRepository`

**Medium** — convention violations causing inconsistency:
- Hardcoded colors / `MaterialTheme.colorScheme`
- Hardcoded UI strings
- Wrong UiState pattern (`_uiState`, wrong exposure, direct assignment)
- Wrong contracts structure
- `Log.*` instead of `Timber`
- Dialog missing `<DIALOG>`

**Low** — style / polish:
- Inline magic numbers without file-level constants
- Wrong component file location
- Missing preview
- `private @Composable` helpers that should be their own file
- Code smells from the bad-practices scan

---

### 4 — Write findings as the plan

Structure the plan file:

```
## Findings

### High
- `path/to/File.kt` — <rule violated> → <one-line fix>

### Medium
- ...

### Low
- ...

## Fixes (grouped by file)

### `path/to/File.kt`
- [ ] <fix 1>
- [ ] <fix 2>
```

If **no violations found**:
```
✓ No violations found — branch follows project standards.
```

Call `ExitPlanMode` when done. The user reviews and approves before any fixes are applied.
