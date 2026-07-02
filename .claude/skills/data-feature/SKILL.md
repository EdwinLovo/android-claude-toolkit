---
name: data-feature
description: Templates for scaffolding a networking-backed data feature — network op file, domain model, mapper, data source method, repository interface + impl, DI wiring, and optional use case. Use when adding a new resource to the data layer.
allowed-tools:
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

# Data feature

Templates for scaffolding a networking-backed feature end-to-end (no UI). `/new-data-feature` references this skill.

> Primary examples use **<NET_LIB> (Apollo/GraphQL)**. For a Retrofit codebase, the "Retrofit variant" section at the bottom shows the equivalent shape — the layering is identical.

---

## Guiding rule

> **Only create a use case when the logic is non-trivial or reused across multiple ViewModels.**
> A ViewModel calling a single repository method does not warrant a use case.
> ([Google — Domain Layer](https://developer.android.com/topic/architecture/domain-layer))

---

## Section 1 — Domain models (`domain/model/`)

Plain `data class` types, no framework deps. Response entities have no `"Response"` suffix.

```kotlin
// domain/model/response/auth/DeviceAuth.kt
package <PKG_ROOT>.domain.model.response.auth

data class DeviceAuth(
    val accessToken: String,
    val expiresAt: String,
    val location: BasicLocation,
)
```

- **Network codegen types never cross the domain boundary** — neither `domain/` nor `presentation/` may import Apollo/Retrofit generated classes
- **Repository interfaces use plain Kotlin types only** — the data source impl constructs any network input types internally

---

## Section 2 — GraphQL files (Apollo variant)

`.graphql` files live in `data/remote/graphql/<feature>/`. Configure the Apollo Gradle plugin `srcDir` to point there.

```
data/remote/graphql/
├── schema.graphqls
├── auth/
│   ├── SignInEmployee.graphql
│   └── SignOutEmployee.graphql
└── catalog/
    └── PublicPackages.graphql
```

One operation per file. Operation names must be unique across the project.

```graphql
# auth/SignOutEmployee.graphql
mutation SignOutEmployee {
  signOutEmployee {
    success
  }
}
```

Apollo generates `SignOutEmployeeMutation`, `SignOutEmployeeMutation.Data`, etc. After adding/editing a `.graphql` file, run `./gradlew build` to trigger codegen.

---

## Section 3 — Data source layer

Split by auth level, not by feature. Named client → matching data source. Data sources are the only classes that touch `ApolloClient` (or `Retrofit`).

| Named client | Data source interface | Used for |
|---|---|---|
| `"public"` | `PublicRemoteDataSource` | Unauthenticated (device pairing) |
| `"device"` | `DeviceRemoteDataSource` | Device-authenticated (employee sign-in) |
| `"user"` / `"employee"` | `<User>RemoteDataSource` | User-authenticated (main app operations) |
| `"public"` (reused) | `RefreshRemoteDataSource` | Token-refresh mutations (must not carry auth header) |

### Interface — non-suspend, returns call factory

```kotlin
// data/remote/datasource/EmployeeRemoteDataSource.kt
package <PKG_ROOT>.data.remote.datasource

interface EmployeeRemoteDataSource {
    fun signOutEmployee(): ApolloCall<SignOutEmployeeMutation.Data>
}
```

### Implementation — holds named client, constructs inputs

```kotlin
// data/remote/datasource/EmployeeRemoteDataSourceImpl.kt
@Singleton
class EmployeeRemoteDataSourceImpl @Inject constructor(
    @param:Named(APOLLO_CLIENT_EMPLOYEE) private val apolloClient: ApolloClient,
) : EmployeeRemoteDataSource {
    override fun signOutEmployee(): ApolloCall<SignOutEmployeeMutation.Data> =
        apolloClient.mutation(SignOutEmployeeMutation())
}
```

### DI — `data/di/DataSourceModule.kt`

```kotlin
@Binds @Singleton
abstract fun bindEmployeeRemoteDataSource(
    impl: EmployeeRemoteDataSourceImpl,
): EmployeeRemoteDataSource
```

---

## Section 4 — Repository interface (`domain/repository/<Resource>Repository.kt`)

One interface per aggregate resource. Plain Kotlin / domain types only — never network types.

```kotlin
package <PKG_ROOT>.domain.repository

interface AuthRepository {
    suspend fun signInEmployee(code: String): ApiResult<EmployeeAuth>
    suspend fun signOutEmployee(): ApiResult<Boolean>
    suspend fun disconnectDevice(): ApiResult<Unit>
}
```

- `suspend fun` + `ApiResult<T>` → one-shot operations (mutations, single fetches)
- `fun` returning `Flow<ApiResult<T>>` → reactive/streaming reads
- `val <resource>Invalidated: SharedFlow<Unit>` → include only when mutations exist and consumers need to react

---

## Section 5 — Repository implementation (`data/repository/<feature>/<Resource>RepositoryImpl.kt`)

Extends `BaseRepository()`. Injects data sources (not `ApolloClient`). Executes calls with `safeApolloCallSuspend` / `safeApolloCall`.

```kotlin
package <PKG_ROOT>.data.repository.auth

class AuthRepositoryImpl @Inject constructor(
    private val publicRemoteDataSource: PublicRemoteDataSource,
    private val deviceRemoteDataSource: DeviceRemoteDataSource,
    private val employeeRemoteDataSource: EmployeeRemoteDataSource,
) : BaseRepository(), AuthRepository {

    override suspend fun signInEmployee(code: String): ApiResult<EmployeeAuth> =
        safeApolloCallSuspend(
            apolloCall = deviceRemoteDataSource.signInEmployee(code),
            mapper = { it.toEmployeeAuth() },
        )

    override suspend fun signOutEmployee(): ApiResult<Boolean> =
        safeApolloCallSuspend(
            apolloCall = employeeRemoteDataSource.signOutEmployee(),
            mapper = { it.signOutEmployee.success },
        )
}
```

- `safeApolloCallSuspend(apolloCall, mapper)` → executes, maps on success, returns `ApiResult.Error` on exception or GraphQL error
- `safeApolloCall(apolloCall, mapper)` → reactive `Flow<ApiResult<T>>`

---

## Section 6 — Mappers (`data/mappers/<feature>/`)

Extension functions on network codegen types, returning domain models. **Never return a raw codegen type from a mapper.**

```kotlin
// data/mappers/auth/EmployeeAuthMapper.kt
package <PKG_ROOT>.data.mappers.auth

fun SignInEmployeeMutation.Data.toEmployeeAuth(): EmployeeAuth =
    EmployeeAuth(
        accessToken = signInEmployee.accessToken,
        expiresAt = signInEmployee.expiresAt,
    )
```

- Naming: `.to<DomainType>()` — descriptive, not `.toModel()`
- One file per network operation type (or per domain model when multiple ops map to the same target)
- Called from the `mapper` lambda in `safeApolloCallSuspend`

**Domain-only mappers.** When a mapper transforms only between domain models (no network types on either side), it lives in `domain/model/<area>/<Name>Mappers.kt` next to the target model. `data/mappers/` is reserved for network↔domain bridging.

---

## Section 7 — DI (`data/di/RepositoryModule.kt`)

One `@Binds @Singleton` per new repository:

```kotlin
@Binds @Singleton
abstract fun bindAuthRepository(impl: AuthRepositoryImpl): AuthRepository
```

DI is split across three modules:
- `ApiModule` — provides the named `ApolloClient` (or `Retrofit`) instances
- `DataSourceModule` — binds data source interfaces to impls
- `RepositoryModule` — binds repository interfaces to impls

---

## Section 8 — Use cases (`domain/usecase/<feature>/`)

Only when the logic is non-trivial or reused.

```kotlin
class GetCatalogProductsUseCase @Inject constructor(
    private val productRepository: ProductRepository,
) {
    operator fun invoke(locationUuid: String, search: String?): Flow<PagingData<CatalogListItem>> =
        productRepository.getProducts(locationUuid, search)
}
```

- Single public `operator fun invoke()`
- Stateless — no `@Singleton`

---

## Section 9 — End-to-end checklist

```
[ ] Network op file added
    Apollo:    data/remote/graphql/<feature>/<Operation>.graphql
    Retrofit:  new method on the Retrofit service interface
[ ] Run codegen if applicable (`./gradlew build` for Apollo)
[ ] domain/model/response/<feature>/<Resource>.kt created (if new)
[ ] domain/repository/<Resource>Repository.kt — method declared (plain Kotlin types only)
[ ] Decide which data source (public / device / user) owns the operation
[ ]   If new operation: add method to data source interface returning ApolloCall<D> (or suspend Response<T>)
[ ]   Add impl; construct network input types there
[ ] data/repository/<feature>/<Resource>RepositoryImpl.kt — method implemented
      safeApolloCallSuspend / safeCallSuspend → one-shot
      safeApolloCall / safeCall               → reactive / streaming
[ ] data/mappers/<feature>/<Resource>Mapper.kt — mapper extension function written
[ ] @Binds @Singleton added to DataSourceModule (new data source only)
[ ] @Binds @Singleton added to RepositoryModule (new repository only)
[ ] Use case added ONLY if logic warrants it
[ ] ViewModel injects repository (or use case), collects ApiResult<T> correctly
[ ] ./gradlew ktlintCheck / test / lintDebug pass
```

---

## Retrofit variant

Everything above holds; only two things change.

**Data source signature** — suspending, returns `Response<T>`:

```kotlin
interface UserRemoteDataSource {
    suspend fun getProfile(id: String): Response<GetProfileResponse>
}

class UserRemoteDataSourceImpl @Inject constructor(
    @param:Named(RETROFIT_USER) private val api: UserApi,
) : UserRemoteDataSource {
    override suspend fun getProfile(id: String) = api.getProfile(id)
}
```

**Repository execution** — use the Retrofit variant of `safeCallSuspend`:

```kotlin
override suspend fun getProfile(id: String): ApiResult<Profile> =
    safeCallSuspend(
        call = { remoteDataSource.getProfile(id) },
        mapper = { it.toProfile() },
    )
```

`BaseRepository.safeCallSuspend` for Retrofit takes a `suspend () -> Response<T>` lambda, checks `.isSuccessful`, and translates HTTP errors into `ApiResult.Error(code, message)`.

Mappers, DI, use cases, and the folder layout are identical to the Apollo variant.
