---
name: repositories
description: Interface in domain, impl extends BaseRepository, safeCall + mapper pattern, ApiResult return type
paths:
  - "**/repository/**/*.kt"
---

# Repositories

**Interface lives in `domain/repository/`, implementation lives in `data/repository/<feature>/`.** The presentation layer only ever depends on the interface.

**Setting up repositories in a new project:** invoke the `repository-primitives` skill (`.claude/skills/repository-primitives/SKILL.md`). It ships the reference files this rule assumes exist — `ApiResult.kt`, `BaseRepository.kt` (Apollo or Retrofit variant), plus `HttpConstants.kt` for Apollo and `BasePagingSource.kt` for Paging 3.

## Interface

`domain/repository/<Resource>Repository.kt`:

```kotlin
package <PKG_ROOT>.domain.repository

interface <Resource>Repository {
    // One-shot: suspend + ApiResult<T>
    suspend fun getById(id: String): ApiResult<<Resource>>
    suspend fun create(request: Create<Resource>Request): ApiResult<<Resource>>

    // Reactive / streaming: Flow<ApiResult<T>>
    fun observe(id: String): Flow<ApiResult<<Resource>>>
}
```

**Hard rules on the interface:**
- Parameters and return types are **plain Kotlin / domain types only** — never network-layer types (Apollo generated classes, Retrofit response wrappers, DTOs). Impose the boundary at the interface.
- `suspend fun` returns `ApiResult<T>` — one-shot operations
- `fun` returns `Flow<ApiResult<T>>` — reactive/streaming reads

## Implementation

Two variants — pick based on the networking library. Both extend `BaseRepository()` and both return `ApiResult<T>`. The difference is what gets injected:

- **Apollo / GraphQL** → inject a `<Resource>RemoteDataSource` (see `rules/data-sources.md`)
- **Retrofit** → inject the Retrofit API service directly (no data-source wrapper — Retrofit's interface IS the per-endpoint contract)

### Apollo variant — with a data source

`data/repository/<feature>/<Resource>RepositoryImpl.kt`:

```kotlin
package <PKG_ROOT>.data.repository.<feature>

class <Resource>RepositoryImpl @Inject constructor(
    private val remoteDataSource: <Resource>RemoteDataSource,
) : BaseRepository(), <Resource>Repository {

    override suspend fun getById(id: String): ApiResult<<Resource>> =
        safeApolloCallSuspend(
            apolloCall = remoteDataSource.getById(id),
            mapper = { it.to<Resource>() },
        )

    override fun observe(id: String): Flow<ApiResult<<Resource>>> =
        safeApolloCall(
            apolloCall = remoteDataSource.observe(id),
            mapper = { it.to<Resource>() },
        )

    override suspend fun create(request: Create<Resource>Request): ApiResult<<Resource>> =
        safeApolloCallSuspend(
            apolloCall = remoteDataSource.create(request.toNetworkInput()),
            mapper = { it.to<Resource>() },
        )
}
```

### Retrofit variant — no data source, direct API-service injection

Retrofit's `interface UserApi { @GET ... }` already provides the per-endpoint contract that Apollo needs a data source to synthesize. Wrapping it in a `UserRemoteDataSource` that just forwards each call is pure ceremony. Inject the API service directly:

```kotlin
package <PKG_ROOT>.data.repository.<feature>

class <Resource>RepositoryImpl @Inject constructor(
    private val api: <Resource>Api,
) : BaseRepository(), <Resource>Repository {

    override suspend fun getById(id: String): ApiResult<<Resource>> =
        safeCallSuspend(
            call = { api.getById(id) },
            mapper = { it.to<Resource>() },
        )

    override suspend fun create(request: Create<Resource>Request): ApiResult<<Resource>> =
        safeCallSuspend(
            call = { api.create(request.toDto()) },
            mapper = { it.to<Resource>() },
        )
}
```

The `BaseRepository.safeCallSuspend` Retrofit variant takes a `suspend () -> Response<T>` lambda, checks `.isSuccessful`, translates HTTP errors into `ApiError(code, message)`, and applies the mapper on success.

Multiple auth contexts in a Retrofit project are handled at the Retrofit builder level — one `@Named("public")` Retrofit instance, one `@Named("user")` Retrofit instance, each producing its own service interface — no data-source layer needed.

**Hard rules on the impl (both variants):**
- **Extends `BaseRepository()`** — provides `safeCall*` / `safeApolloCall*` helpers that wrap network calls and return `ApiResult<T>`
- **Injects the data source (Apollo) or the API service (Retrofit)**, never `ApolloClient` / `Retrofit` itself
- **Mapping happens inside the `mapper` lambda** — never mid-function
- **Errors bubble through `ApiError`** — no exceptions escape a repository method

## Errors

Repositories never throw. `BaseRepository.safeApolloCallSuspend` / `safeCallSuspend` catch network / parse / server errors and return `ApiError(code, message)`. The ViewModel or delegate calling the repository handles the error:

```kotlin
repository.getById(id)
    .onSuccess { resource -> uiState.reduce { copy(resource = resource) } }
    .onError { _, message -> ErrorEventBus.send(message.toUiText()) }
```

See `rules/error-handling.md` for `ErrorEventBus` and `UiText` details.

## Common violations

- Repository interface returning a network type (e.g. `Response<T>`, `ApolloResponse<D>`) → return `ApiResult<DomainType>`
- Repository impl injecting `ApolloClient` directly → inject a `RemoteDataSource` instead (Apollo variant)
- Repository impl injecting `Retrofit` directly → inject the API service interface (e.g. `<Resource>Api`) instead
- Retrofit project with a `<Resource>RemoteDataSource` that just forwards each method to the API service → drop the wrapper, inject the API service directly into the repository
- `try/catch` inside a repository method → use `safeApolloCallSuspend` / `safeCallSuspend`
- Mapper logic inline in the impl body → move to the `mapper` lambda or an extension function in `data/mappers/`
- Two implementations of the same interface (e.g. `Real` + `Fake`) both in `data/repository/` → fakes belong in `app/src/test/` or `app/src/androidTest/`
- `BaseRepository`, `ApiResult`, or `safeApolloCall*` / `safeCallSuspend` referenced but the files don't exist in `<PKG_ROOT>/data/repository/` and `<PKG_ROOT>/domain/utils/` → invoke the `repository-primitives` skill
