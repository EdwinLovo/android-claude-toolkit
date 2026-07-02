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

Follow the standards in `.claude/skills/data-feature/SKILL.md` throughout.

---

## Step 1 — Gather requirements

Ask with `AskUserQuestion` **before writing any code**:

1. **Resource name** — PascalCase singular noun (e.g., `Clinic`, `Order`, `Location`). Base name for all generated files.
2. **Operations needed** — query, mutation, or both?
3. **Operation names** — for each operation, the network op name (e.g., `GetClinic`, `CreateClinic`, `UpdateLocation`). Must be unique across the project.
4. **Use case needed?** Remind the user of the guiding rule: *only create a use case when the logic is non-trivial or reused across multiple ViewModels.*

---

## Step 2 — Scaffold in order

After collecting answers, create files in this sequence.

### a. Network operation file(s)

Path (Apollo variant): `app/src/main/graphql/<featureLower>/<Resource>Queries.graphql` and/or `<Resource>Mutations.graphql`

```graphql
# Example — ClinicQueries.graphql
query GetClinic($uuid: UUID!) {
  clinic(uuid: $uuid) {
    uuid
    name
    # TODO: add fields from schema
  }
}
```

Leave field selection as a stub with `# TODO: add fields from schema` — the developer fills this in after consulting the schema.

For Retrofit projects, add a `@GET` / `@POST` method to the appropriate service interface instead.

### b. Domain model

Path: `domain/model/response/<feature>/<Resource>.kt`

```kotlin
package <PKG_ROOT>.domain.model.response.<feature>

data class <Resource>(
    val uuid: String,
    // TODO: add fields
)
```

If a mutation requires many parameters, also create:

Path: `domain/model/request/<Action><Resource>Request.kt`

```kotlin
package <PKG_ROOT>.domain.model.request

data class <Action><Resource>Request(
    // TODO: add fields
)
```

### c. Repository interface

Path: `domain/repository/<Resource>Repository.kt`

- Include a `SharedFlow<Unit>` invalidation signal only if mutations are present.
- Parameters and return types are plain Kotlin / domain types — no network-lib imports.

```kotlin
package <PKG_ROOT>.domain.repository

import <PKG_ROOT>.domain.model.response.<feature>.<Resource>
import <PKG_ROOT>.domain.utils.ApiResult
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.SharedFlow

interface <Resource>Repository {
    // Include only if mutations are present:
    val <resource>Invalidated: SharedFlow<Unit>

    // One-shot fetch:
    suspend fun get<Resource>(uuid: String): ApiResult<<Resource>>

    // Reactive stream (if needed):
    fun observe<Resource>(uuid: String): Flow<ApiResult<<Resource>>>

    // Mutation example:
    suspend fun create<Resource>(request: Create<Resource>Request): ApiResult<<Resource>>
}
```

### d. Data source (if a new one is needed)

If this feature needs a new auth-scoped data source, follow `.claude/skills/data-feature/SKILL.md` §3 to create the interface and impl, and add the `@Binds @Singleton` entry to `DataSourceModule`.

If the feature reuses an existing data source, add a new method to that interface + impl.

### e. Repository implementation + mappers

Path: `data/repository/<resource>/<Resource>RepositoryImpl.kt`

Include:
- `BaseRepository()` + `<Resource>Repository` in the class header
- Data sources injected via `@Inject constructor` (never the network client directly)
- `MutableSharedFlow` + exposed `SharedFlow` for invalidation (if mutations present)
- `safeCallSuspend` (or Apollo variant) for one-shot, `safeCall` for reactive streams
- Mapper extension functions in a separate file under `data/mappers/<feature>/`

```kotlin
package <PKG_ROOT>.data.repository.<feature>

class <Resource>RepositoryImpl @Inject constructor(
    private val remoteDataSource: <Resource>RemoteDataSource,
) : BaseRepository(), <Resource>Repository {

    private val _<resource>Invalidated = MutableSharedFlow<Unit>(
        replay = 0, extraBufferCapacity = 1, onBufferOverflow = BufferOverflow.DROP_OLDEST,
    )
    override val <resource>Invalidated: SharedFlow<Unit> = _<resource>Invalidated.asSharedFlow()

    override suspend fun get<Resource>(uuid: String): ApiResult<<Resource>> =
        safeApolloCallSuspend(
            apolloCall = remoteDataSource.get<Resource>(uuid),
            mapper = { it.to<Resource>() },
        )
}
```

Path: `data/mappers/<feature>/<Resource>Mapper.kt`

```kotlin
package <PKG_ROOT>.data.mappers.<feature>

// TODO: fill in after codegen (./gradlew build) generates the network type
fun Get<Resource>Query.Data.to<Resource>(): <Resource> = <Resource>(
    uuid = <resource>.uuid,
    // TODO: map fields
)
```

### f. Use case (only if requested)

Path: `domain/usecase/<featureLower>/<Verb><Resource>UseCase.kt`

```kotlin
package <PKG_ROOT>.domain.usecase.<featureLower>

import <PKG_ROOT>.domain.model.response.<feature>.<Resource>
import <PKG_ROOT>.domain.repository.<Resource>Repository
import <PKG_ROOT>.domain.utils.ApiResult
import javax.inject.Inject

class <Verb><Resource>UseCase @Inject constructor(
    private val <resource>Repository: <Resource>Repository,
) {
    suspend operator fun invoke(/* params */): ApiResult<<Resource>> =
        <resource>Repository.<verb><Resource>(/* params */)
}
```

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
