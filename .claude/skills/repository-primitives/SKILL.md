---
name: repository-primitives
description: First-time setup for the repository layer — reference implementations for ApiResult, BaseRepository (Apollo and Retrofit variants), HttpConstants, and BasePagingSource. Invoke when scaffolding data-layer helpers in a new project, when a rule references safeApolloCall*/safeCallSuspend/ApiResult and they don't exist yet, or when adding a paging source.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
---

# Repository primitives

Reference implementations for the files that back the repository pattern documented in `.claude/rules/repositories.md`. Copy them verbatim into the paths below when setting up a new project. Every repository interface, every impl class, and every `safeApolloCall*` / `safeCallSuspend` call assumes they exist.

For the conventions that use these types (interface shape, impl variants, error routing, common violations), see `.claude/rules/repositories.md`.

---

## File 1 — `ApiResult.kt`

Sealed result type used across the data → presentation boundary. All repository methods return `ApiResult<T>` (one-shot) or `Flow<ApiResult<T>>` (streaming).

Path: `<PKG_ROOT>/domain/utils/ApiResult.kt`

```kotlin
package <PKG_ROOT>.domain.utils

sealed interface ApiResult<T : Any>

class ApiSuccess<T : Any>(val data: T) : ApiResult<T>

class ApiError<T : Any>(val code: Int? = null, val message: String? = null) : ApiResult<T>

class ApiLoading<T : Any>(val data: T? = null) : ApiResult<T>

suspend fun <T : Any> ApiResult<T>.onSuccess(block: suspend (T) -> Unit): ApiResult<T> {
    if (this is ApiSuccess) block(data)
    return this
}

suspend fun <T : Any> ApiResult<T>.onError(block: suspend (code: Int?, message: String?) -> Unit): ApiResult<T> {
    if (this is ApiError) block(code, message)
    return this
}

suspend fun <T : Any> ApiResult<T>.onLoading(block: suspend () -> Unit): ApiResult<T> {
    if (this is ApiLoading) block()
    return this
}
```

Note the three concrete types (`ApiSuccess`, `ApiError`, `ApiLoading`) are **top-level classes**, not nested inside `ApiResult`. `is ApiSuccess` — not `is ApiResult.Success`.

---

## File 2 — `HttpConstants.kt` (Apollo variant only)

Three `const val` GraphQL error-code strings that `BaseRepository.processResponse` maps to HTTP-style codes. Skip this file entirely if you're using the Retrofit variant.

Path: `<PKG_ROOT>/data/remote/HttpConstants.kt`

```kotlin
package <PKG_ROOT>.data.remote

const val GRAPHQL_ERROR_CODE_KEY = "code"
const val GRAPHQL_ERROR_CODE_UNAUTHENTICATED = "UnauthorizedException"
const val GRAPHQL_ERROR_CODE_INTERNAL_SERVER_ERROR = "INTERNAL_SERVER_ERROR"
```

Adjust the string values to whatever your GraphQL server actually emits under `extensions.code`.

---

## File 3 — `BaseRepository.kt` (Apollo variant — default)

Every repository impl extends this class and uses its `safeApolloCall` / `safeApolloCallSuspend` helpers to execute Apollo calls, translate errors, and map to domain types.

Path: `<PKG_ROOT>/data/repository/BaseRepository.kt`

```kotlin
package <PKG_ROOT>.data.repository

import <PKG_ROOT>.data.remote.GRAPHQL_ERROR_CODE_INTERNAL_SERVER_ERROR
import <PKG_ROOT>.data.remote.GRAPHQL_ERROR_CODE_KEY
import <PKG_ROOT>.data.remote.GRAPHQL_ERROR_CODE_UNAUTHENTICATED
import <PKG_ROOT>.domain.utils.ApiError
import <PKG_ROOT>.domain.utils.ApiLoading
import <PKG_ROOT>.domain.utils.ApiResult
import <PKG_ROOT>.domain.utils.ApiSuccess
import com.apollographql.apollo.ApolloCall
import com.apollographql.apollo.api.ApolloResponse
import com.apollographql.apollo.api.Operation
import com.apollographql.apollo.exception.ApolloException
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.onStart
import timber.log.Timber

open class BaseRepository {
    fun <D : Operation.Data, T : Any> safeApolloCall(
        apolloCall: ApolloCall<D>,
        mapper: (D) -> T,
    ): Flow<ApiResult<T>> =
        flow {
            emit(processResponse(apolloCall.execute(), mapper))
        }.onStart {
            emit(ApiLoading())
        }.catch { throwable ->
            Timber.e(throwable)
            emit(ApiError(message = throwable.message))
        }

    suspend fun <D : Operation.Data, T : Any> safeApolloCallSuspend(
        apolloCall: ApolloCall<D>,
        mapper: (D) -> T,
    ): ApiResult<T> =
        try {
            processResponse(apolloCall.execute(), mapper)
        } catch (e: ApolloException) {
            Timber.e(e)
            ApiError(message = e.message)
        }

    private fun <D : Operation.Data, T : Any> processResponse(
        response: ApolloResponse<D>,
        mapper: (D) -> T,
    ): ApiResult<T> {
        val exception = response.exception
        return when {
            exception != null -> {
                Timber.e(exception)
                ApiError(message = exception.message)
            }
            response.hasErrors() -> {
                val error = response.errors?.firstOrNull()
                val code = error?.extensions?.get(GRAPHQL_ERROR_CODE_KEY) as? String
                when (code) {
                    GRAPHQL_ERROR_CODE_UNAUTHENTICATED -> {
                        Timber.w("Unauthenticated response")
                        ApiError(code = 401, message = "Unauthenticated")
                    }
                    GRAPHQL_ERROR_CODE_INTERNAL_SERVER_ERROR -> {
                        Timber.e("Internal server error: ${error?.message}")
                        ApiError(code = 500)
                    }
                    else -> {
                        Timber.e("GraphQL error: ${error?.message}")
                        ApiError(message = error?.message)
                    }
                }
            }
            else -> ApiSuccess(mapper(response.dataAssertNoErrors))
        }
    }
}
```

---

## File 3 (alternative) — `BaseRepository.kt` (Retrofit variant)

**For projects using Retrofit instead of Apollo, replace File 3 with this body.** Template — not extracted verbatim from a production codebase. Adjust exception types to match your networking stack's actual failure modes, and add auth-refresh handling in the `HttpException` branch if your project needs it.

Path: `<PKG_ROOT>/data/repository/BaseRepository.kt`

```kotlin
package <PKG_ROOT>.data.repository

import <PKG_ROOT>.domain.utils.ApiError
import <PKG_ROOT>.domain.utils.ApiLoading
import <PKG_ROOT>.domain.utils.ApiResult
import <PKG_ROOT>.domain.utils.ApiSuccess
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.onStart
import retrofit2.HttpException
import retrofit2.Response
import timber.log.Timber
import java.io.IOException

open class BaseRepository {
    suspend fun <T : Any, R : Any> safeCallSuspend(
        call: suspend () -> Response<T>,
        mapper: (T) -> R,
    ): ApiResult<R> =
        try {
            val response = call()
            when {
                response.isSuccessful -> response.body()?.let { ApiSuccess(mapper(it)) }
                    ?: ApiError(code = response.code(), message = "Empty body")
                else -> ApiError(code = response.code(), message = response.errorBody()?.string())
            }
        } catch (e: IOException) {
            Timber.e(e)
            ApiError(message = e.message)
        } catch (e: HttpException) {
            Timber.e(e)
            ApiError(code = e.code(), message = e.message)
        }

    fun <T : Any, R : Any> safeCall(
        call: suspend () -> Response<T>,
        mapper: (T) -> R,
    ): Flow<ApiResult<R>> =
        flow { emit(safeCallSuspend(call, mapper)) }
            .onStart { emit(ApiLoading()) }
            .catch { throwable ->
                Timber.e(throwable)
                emit(ApiError(message = throwable.message))
            }
}
```

---

## File 4 — `BasePagingSource.kt` (optional)

Only ship this file if the project uses Paging 3 with Apollo. Provides an `ApolloCall<D>.toPage(block)` extension that Paging sources use inside `load()` to execute a call and translate errors into `LoadResult.Error`.

Path: `<PKG_ROOT>/data/paging/BasePagingSource.kt`

```kotlin
package <PKG_ROOT>.data.paging

import androidx.paging.PagingSource
import androidx.paging.PagingSource.LoadResult
import com.apollographql.apollo.ApolloCall
import com.apollographql.apollo.api.Operation
import com.apollographql.apollo.exception.ApolloException
import kotlin.coroutines.cancellation.CancellationException
import timber.log.Timber

abstract class BasePagingSource<K : Any, V : Any> : PagingSource<K, V>() {
    @Suppress("TooGenericExceptionCaught")
    protected suspend fun <D : Operation.Data> ApolloCall<D>.toPage(block: (D) -> LoadResult<K, V>): LoadResult<K, V> =
        try {
            val response = execute()
            response.exception?.let { throw it }
            block(response.dataAssertNoErrors)
        } catch (e: CancellationException) {
            throw e
        } catch (e: ApolloException) {
            Timber.e(e)
            LoadResult.Error(e)
        } catch (e: Exception) {
            Timber.e(e)
            LoadResult.Error(e)
        }
}
```

Concrete `PagingSource` subclasses call it like:
```kotlin
override suspend fun load(params: LoadParams<String>): LoadResult<String, Item> =
    remoteDataSource.itemsPage(cursor = params.key).toPage { data ->
        LoadResult.Page(
            data = data.items.map { it.toDomain() },
            prevKey = null,
            nextKey = data.pageInfo.endCursor,
        )
    }
```

---

## Verification after copying

Once these files exist in a project:

- `domain/utils/ApiResult.kt` — sealed interface + three top-level classes (`ApiSuccess`, `ApiError`, `ApiLoading`) + three suspend extensions (`onSuccess`, `onError`, `onLoading`)
- `data/repository/BaseRepository.kt` — Apollo variant (default) **or** Retrofit variant, never both
- `data/remote/HttpConstants.kt` — three GraphQL error code constants (Apollo variant only)
- `data/paging/BasePagingSource.kt` — only if the project uses Paging 3 with Apollo

Repositories can then declare `class <Resource>RepositoryImpl @Inject constructor(...) : BaseRepository(), <Resource>Repository` and call `safeApolloCallSuspend(apolloCall, mapper)` / `safeCallSuspend(call, mapper)`. See `rules/repositories.md` for the full pattern (interface shape, mutation handling, error routing to `ErrorEventBus`).
