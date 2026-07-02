---
description: Scaffold all boilerplate files for a new feature screen (Route, Contract, ViewModel, Screen, delegates) and register in AppNavHost
argument-hint: <ScreenName>
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - AskUserQuestion
---

# /new-screen

Scaffold all boilerplate files for a new feature screen.

**Before doing anything else**, read the screen anatomy skill:
`.claude/skills/screen-anatomy/SKILL.md`

All file templates, package names, import rules, and AppNavHost update instructions are defined there. Do not guess or invent templates — always use the ones from that file.

---

## Important guidelines

- **Always use `AskUserQuestion`** when asking the user anything
- **Never assume** screen name, route type, or events — always ask
- **Generate all files in one pass** after collecting all answers
- **Imports must match the templates exactly** — refer to `SKILL.md`

---

## Step 1 — Screen name

Use `AskUserQuestion`:

> What is the screen name? (PascalCase, e.g. PatientProfile, AppointmentDetails)

Derive:
- `<Screen>` — exact PascalCase name provided
- `<feature>` — lowercase, no separators (e.g. `patientprofile`)

---

## Step 2 — Route type

Use `AskUserQuestion`:

> Route type?
>
> 1. **object** — no navigation params (like HomeRoute)
> 2. **data class** — requires params
>
> If data class: list each param as `name: Type` (e.g. `patientId: String`)

---

## Step 2.5 — Delegates

Use `AskUserQuestion`:

> Does this screen have `@ViewModelScoped` delegates? (yes / no)
>
> Use delegates when sub-flows (detail dialogs, multi-step forms) each manage their own loading/error/data state independently.

**If yes**, follow up:

> What delegate groups do you need? List one per line (no "Event" suffix).
> Example:
>   ProductDetail
>   ServiceDetail

For each group: generate a `contracts/<Child>Contract.kt` + a `delegates/<Child>Delegate.kt`, and wire the ViewModel with delegate injection, `init` calls, and top-level state getters. The Screen composable collects each delegate state separately.

**If no**: no `delegates/` subfolder, no child contract files, the ViewModel handles all events directly.

---

## Step 3 — Events

Use `AskUserQuestion` for the screen-level `<Screen>Event` (in `contracts/<Screen>Contract.kt`):

> Events for `<Screen>Event`?
>
> List one per line: `EventName` or `EventName(paramName: ParamType)`
> Example:
>   OnPopBackStack
>   OnTabSelected(tab: HomeTab)
>
> Leave blank if all events belong to delegate child groups.

For each delegate group from Step 2.5:

> Events for `<Child>Event`?
>
> List one per line: `EventName` or `EventName(paramName: ParamType)`
> `OnDismissed` is always included automatically.

---

## Step 4 — UiState fields

Use `AskUserQuestion` for the screen-level `<Screen>UiState` (in `contracts/<Screen>Contract.kt`):

> UiState fields for `<Screen>UiState`?
>
> List one per line: `fieldName: Type = defaultValue`
> Example:
>   isLoading: Boolean = false
>   patientName: String = ""
>
> Leave blank or type "none" to skip (TODO placeholder will be used).

For each delegate group:

> UiState fields for `<Child>UiState`?
>
> `isLoading: Boolean = false` is always included automatically.
> List any additional fields, or leave blank.

---

## Step 5 — Generate files

Using the templates from `.claude/skills/screen-anatomy/SKILL.md`, create:

1. `<feature>/<Screen>Route.kt`
2. `<feature>/contracts/<Screen>Contract.kt` — `<Screen>UiState` + `interface <Screen>Event`
3. `<feature>/contracts/<Child>Contract.kt` — one per delegate group (if any)
4. `<feature>/delegates/<Child>Delegate.kt` — one per delegate group (if any)
5. `<feature>/<Screen>ViewModel.kt`
6. `<feature>/<Screen>Screen.kt` (with `<Screen>ScreenContent` preview at bottom)

Do **not** create standalone `<Screen>Event.kt`, `<Screen>UiState.kt`, or an `events/` subfolder.

---

## Step 6 — Update AppNavHost.kt

Follow the AppNavHost update rules from `.claude/skills/screen-anatomy/SKILL.md`.

---

## Step 7 — Confirm

Report:

```
Created:
- <feature>/<Screen>Route.kt
- <feature>/contracts/<Screen>Contract.kt
- <feature>/contracts/<Child>Contract.kt  (one per delegate group, if any)
- <feature>/delegates/<Child>Delegate.kt  (one per delegate group, if any)
- <feature>/<Screen>ViewModel.kt
- <feature>/<Screen>Screen.kt
- AppNavHost.kt updated

Next: run ./gradlew assembleDebug to verify it compiles.
```
