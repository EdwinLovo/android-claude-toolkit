---
description: Scaffold data-layer files for a new networking-backed feature — network op, domain model, mapper, data source, repository, DI, optional use case (no UI)
argument-hint: <ResourceName>
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - AskUserQuestion
---

# /new-data-feature

Scaffolds the data layer for a new networking-backed feature: network operation file, domain model, mapper extension functions, repository interface + implementation, and optionally a use case. **No UI scaffolding** — presentation layer is out of scope.

**Before doing anything else**, read the data-feature skill:
`.claude/skills/data-feature/SKILL.md`

All file paths, package names, imports, and template bodies are defined there. Do not guess or invent templates — always use the ones from that skill.

---

## Important guidelines

- **Always use `AskUserQuestion`** when asking the user anything
- **Never assume** resource name, operations, or use-case necessity — always ask
- **Generate all files in one pass** after collecting all answers
- **Templates come from the skill, not this file** — this command owns only the interactive workflow

---

## Step 1 — Gather requirements

Ask with `AskUserQuestion` **before writing any code**:

1. **Resource name** — PascalCase singular noun (e.g., `Clinic`, `Order`, `Location`). Base name for all generated files.
2. **Operations needed** — query, mutation, or both?
3. **Operation names** — for each operation, the network op name (e.g., `GetClinic`, `CreateClinic`, `UpdateLocation`). Must be unique across the project.
4. **Use case needed?** Remind the user of the guiding rule: *only create a use case when the logic is non-trivial or reused across multiple ViewModels.*
5. **Networking library confirmation** — Apollo vs. Retrofit vs. Ktor. The skill covers Apollo primarily with a Retrofit variant; pick the matching template block.

---

## Step 2 — Generate files

Using the templates from `.claude/skills/data-feature/SKILL.md`, create the files in this order. Each template block in the skill is one section (§1–§8) — read the section, then apply it verbatim substituting `<Resource>`, `<feature>`, `<PKG_ROOT>`, and the user's answers.

1. **§2 — Network operation file(s)** — `.graphql` file(s) under `data/remote/graphql/<feature>/` (Apollo) or new methods on the service interface (Retrofit)
2. **§1 — Domain model(s)** — `domain/model/response/<feature>/<Resource>.kt`; also `domain/model/request/<Action><Resource>Request.kt` if the mutation has many parameters
3. **§4 — Repository interface** — `domain/repository/<Resource>Repository.kt`
4. **§3 — Data source** — either add methods to the existing auth-scoped data source, or create a new one following the interface + impl + DI binding template
5. **§5 — Repository implementation** — `data/repository/<feature>/<Resource>RepositoryImpl.kt`
6. **§6 — Mapper(s)** — `data/mappers/<feature>/<Resource>Mapper.kt` (network↔domain) or `domain/model/<area>/<Name>Mappers.kt` (domain-only)
7. **§7 — DI binding** — `@Binds @Singleton` entry in `RepositoryModule` (and `DataSourceModule` if a new data source was created)
8. **§8 — Use case** — `domain/usecase/<featureLower>/<Verb><Resource>UseCase.kt`, **only if** the answer to step 1.4 was yes

Do **not** inline template bodies here. If a template is unclear, re-read the skill section — never guess.

---

## Step 3 — Print reminders

After scaffolding, always print:

```
Next steps:
1. Add to RepositoryModule.kt:
   @Binds @Singleton
   abstract fun bind<Resource>Repository(impl: <Resource>RepositoryImpl): <Resource>Repository

2. If new data source: add to DataSourceModule.kt:
   @Binds @Singleton
   abstract fun bind<Resource>RemoteDataSource(
       impl: <Resource>RemoteDataSourceImpl,
   ): <Resource>RemoteDataSource

3. Apollo: run ./gradlew build to trigger codegen, then fill in the mapper
   extension functions with the generated types (Get<Resource>Query.<Resource>, etc.)
   Retrofit: no codegen needed — the mapper reads the DTO fields directly.
```
