---
name: repositories
description: Interface in domain, impl extends BaseRepository, safeCall + mapper pattern, ApiResult return type
paths:
  - "**/repository/**/*.kt"
---

# Repositories

**Interface lives in `domain/repository/`, implementation lives in `data/repository/<feature>/`.** The presentation layer only ever depends on the interface.

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

    // Invalidation signal (only when the repo has mutations that other collectors care about)
    val <resource>Invalidated: SharedFlow<Unit>
}
```

**Hard rules on the interface:**
- Parameters and return types are **plain Kotlin / domain types only** — never network-layer types (Apollo generated classes, Retrofit response wrappers, DTOs). Impose the boundary at the interface.
- `suspend fun` returns `ApiResult<T>` — one-shot operations
- `fun` returns `Flow<ApiResult<T>>` — reactive/streaming reads
- Include a `SharedFlow<Unit>` invalidation signal only when mutations exist and other consumers need to react

## Implementation

`data/repository/<feature>/<Resource>RepositoryImpl.kt`:

```kotlin
package <PKG_ROOT>.data.repository.<feature>

class <Resource>RepositoryImpl @Inject constructor(
    private val remoteDataSource: <Resource>RemoteDataSource,
) : BaseRepository(), <Resource>Repository {

    private val _<resource>Invalidated = MutableSharedFlow<Unit>(
        replay = 0, extraBufferCapacity = 1, onBufferOverflow = BufferOverflow.DROP_OLDEST,
    )
    override val <resource>Invalidated: SharedFlow<Unit> = _<resource>Invalidated.asSharedFlow()

    override suspend fun getById(id: String): ApiResult<<Resource>> =
        safeCallSuspend(
            call = remoteDataSource.getById(id),
            mapper = { it.to<Resource>() },
        )

    override fun observe(id: String): Flow<ApiResult<<Resource>>> =
        safeCall(
            call = remoteDataSource.observe(id),
            mapper = { it.to<Resource>() },
        )

    override suspend fun create(request: Create<Resource>Request): ApiResult<<Resource>> {
        val result = safeCallSuspend(
            call = remoteDataSource.create(request.toNetworkInput()),
            mapper = { it.to<Resource>() },
        )
        if (result is ApiResult.Success) _<resource>Invalidated.tryEmit(Unit)
        return result
    }
}
```

**Hard rules on the impl:**
- **Extends `BaseRepository()`** — provides `safeCallSuspend` / `safeCall` that wrap network calls and return `ApiResult<T>`, catching exceptions and translating error bodies
- **Injects data sources, never the network client directly** — no `ApolloClient` / `Retrofit` field in a repository
- **Mapping happens inside the `mapper` lambda** — never mid-function or inside the data source
- **Emit `Invalidated` only on success** — a failed mutation shouldn't invalidate caches
- **Errors bubble through `ApiResult.Error`** — no exceptions escape a repository method

## Errors

Repositories never throw. `BaseRepository.safeCallSuspend` catches network / parse / server errors and returns `ApiResult.Error(code, message)`. The ViewModel or delegate calling the repository handles the error:

```kotlin
repository.getById(id)
    .onSuccess { resource -> uiState.reduce { copy(resource = resource) } }
    .onError { _, message -> ErrorEventBus.send(message.toUiText()) }
```

See `rules/error-handling.md` for `ErrorEventBus` and `UiText` details.

## Common violations

- Repository interface returning a network type (e.g. `Response<T>`, `ApolloResponse<D>`) → return `ApiResult<DomainType>`
- Repository impl injecting `ApolloClient` / `Retrofit` directly → inject a `RemoteDataSource`
- `try/catch` inside a repository method → use `safeCallSuspend` / `safeCall`
- Mapper logic inline in the impl body → move to the `mapper` lambda or an extension function in `data/mappers/`
- `_invalidated.tryEmit(Unit)` on every call, including failures → emit only on `ApiResult.Success`
- Two implementations of the same interface (e.g. `Real` + `Fake`) both in `data/repository/` → fakes belong in `app/src/test/` or `app/src/androidTest/`
