---
name: dependency-injection
description: Hilt modules split by concern — ApiModule, DataSourceModule, RepositoryModule; @Binds > @Provides where possible
paths:
  - "**/di/**/*.kt"
---

# Dependency injection (Hilt)

DI modules live in `data/di/` and are split by concern. Three modules cover 95% of the app.

## Module split

| Module | Provides | Scope |
|---|---|---|
| `ApiModule` | Network clients (`ApolloClient`, `Retrofit`, `HttpClient`), authenticators, interceptors, DataStore | `@Singleton` |
| `DataSourceModule` | Bindings from `RemoteDataSource` / `LocalDataSource` interfaces to their impls | `@Singleton` |
| `RepositoryModule` | Bindings from `<Resource>Repository` interfaces to their impls | `@Singleton` |

Additional modules only when a subsystem is large enough to warrant one — e.g. `AuthModule` for authenticator / token storage, `PagingModule` for `Pager` builders.

## Prefer `@Binds` over `@Provides`

When binding an interface to an impl, `@Binds` (abstract module) is faster and generates less code:

```kotlin
// ✅ Preferred — @Binds in an abstract module
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds @Singleton
    abstract fun bind<Resource>Repository(impl: <Resource>RepositoryImpl): <Resource>Repository
}
```

Use `@Provides` only when construction requires calling a function (e.g. a builder pattern):

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ApiModule {

    @Provides @Singleton @Named(APOLLO_CLIENT_USER)
    fun provideUserApolloClient(
        authenticator: TokenAuthenticator,
        okHttpClient: OkHttpClient,
    ): ApolloClient = ApolloClient.Builder()
        .serverUrl(BuildConfig.SERVER_URL)
        .okHttpClient(okHttpClient)
        .addInterceptor(authenticator)
        .build()
}
```

## Named qualifiers

Use `@Named(CONST)` with a string constant when the same type is provided in multiple flavors (multiple `ApolloClient` instances, multiple `OkHttpClient` variants, multiple `CoroutineDispatcher` types).

```kotlin
// In a top-level constants file — data/di/DiConstants.kt
const val APOLLO_CLIENT_PUBLIC = "apollo.public"
const val APOLLO_CLIENT_DEVICE = "apollo.device"
const val APOLLO_CLIENT_USER   = "apollo.user"
```

At the field declaration, use `@param:Named(...)` on the constructor parameter (not `@Named(...)` on the property):

```kotlin
class UserRemoteDataSourceImpl @Inject constructor(
    @param:Named(APOLLO_CLIENT_USER) private val apolloClient: ApolloClient,
) : UserRemoteDataSource
```

`@param:` targets the constructor parameter, which is what Hilt's field-injection reads.

## Scoping

| Type | Scope | Why |
|---|---|---|
| `ApolloClient`, `Retrofit`, `OkHttpClient`, `HttpClient` | `@Singleton` | Expensive, thread-safe, hold connection pools |
| `RemoteDataSource` impls | `@Singleton` | Wrap the singleton client; no per-request state |
| `Repository` impls | `@Singleton` | Hold `SharedFlow` invalidation signals; state must survive VM recreation |
| `<Child>Delegate` | `@ViewModelScoped` | One instance per host VM; dies with the VM |
| Use cases | Unscoped | Stateless, cheap to construct |
| `ViewModel` | `@HiltViewModel` + `@Inject constructor` | Hilt manages VM scope |

## Hard rules

- **DI modules live in `data/di/`** — never in `presentation/`
- **`@Binds` for interface→impl bindings**, `@Provides` only when construction needs code
- **`@Named` constants live in a single top-level file** — grep-friendly, no duplicated strings
- **`@param:Named(...)` on constructor params** — the `@field:` / `@property:` variants target the wrong element
- **`@Singleton` on expensive shared state** — clients, data sources, repos with invalidation signals
- **`@ViewModelScoped` on delegates** — never plain `@Inject` (would create a fresh delegate per VM), never `@Singleton` (would outlive the VM)
- **No `@Provides` returning `Context`** — Hilt supplies `@ApplicationContext Context` automatically

## Common violations

- `@Provides` for a `Repository` interface → change to `@Binds` in an `abstract class` module
- Two modules providing the same type without `@Named` → resolve with named qualifiers
- `@Named(...)` on the property annotation → change to `@param:Named(...)` on the constructor param
- Missing `@Singleton` on a client / data source → adds retained per-request cost
- Injecting the concrete impl (`<Resource>RepositoryImpl`) into a ViewModel → inject the interface (`<Resource>Repository`)
- DI module in `presentation/` → move to `data/di/`
