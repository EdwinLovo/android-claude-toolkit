---
name: data-sources
description: RemoteDataSource is a call factory (no execution, no mapping); split by auth level via named clients
paths:
  - "**/data/remote/**/*.kt"
  - "**/data/local/**/*.kt"
---

# Data sources

A `RemoteDataSource` is a **call factory**. Its methods build and return an executable call object (Apollo `ApolloCall<D>`, Retrofit `Call<T>`, or a suspending function reference). **It does not execute, and does not map to domain types.** Repositories execute the call and pass a mapper.

This split lets repositories reuse `BaseRepository.safeCallSuspend` for consistent error handling and lets tests stub the data source without a network layer.

## Split by auth level, not by feature

If the app has multiple authentication contexts (public, device-authenticated, employee-authenticated), create one data source **per auth level**, each holding a named client:

| Client | Data source | Used for |
|---|---|---|
| `"public"` | `PublicRemoteDataSource` | Unauthenticated (sign-in, device pairing) |
| `"device"` | `DeviceRemoteDataSource` | Device-authenticated (employee sign-in) |
| `"user"` | `UserRemoteDataSource` | User-authenticated (main app operations) |
| `"public"` (reused) | `RefreshRemoteDataSource` | Token-refresh mutations that must not carry an auth header |

Split by feature would force every data source to hold every client — not what you want.

## Interface

Methods are **non-suspend** and return the executable call object:

```kotlin
package <PKG_ROOT>.data.remote.datasource

interface UserRemoteDataSource {
    fun getProfile(id: String): <Call>Type<GetProfileQuery.Data>
    fun updateProfile(id: String, input: ProfileInput): <Call>Type<UpdateProfileMutation.Data>
}
```

`<Call>Type` = `ApolloCall<...>` (Apollo) or a suspending function type (Retrofit — see the "Retrofit variant" note below).

## Implementation

Holds the **named** client. Constructs network input types (Apollo `Optional`/`UuidInput`, Retrofit `@Body` objects). Returns the call — no `execute()`, no mapping.

```kotlin
@Singleton
class UserRemoteDataSourceImpl @Inject constructor(
    @param:Named(APOLLO_CLIENT_USER) private val apolloClient: ApolloClient,
) : UserRemoteDataSource {

    override fun getProfile(id: String): ApolloCall<GetProfileQuery.Data> =
        apolloClient.query(GetProfileQuery(id = Uuid(id)))

    override fun updateProfile(id: String, input: ProfileInput): ApolloCall<UpdateProfileMutation.Data> =
        apolloClient.mutation(UpdateProfileMutation(id = Uuid(id), input = input.toNetworkInput()))
}
```

## Hard rules

- **`RemoteDataSource` is the ONLY class that imports the network client** (`ApolloClient`, `Retrofit`, `HttpClient`). Repositories never see it.
- **Non-suspend method signatures** — return the call, don't `execute()` it
- **Never map to domain types inside the data source** — that's the repository's job (mapper lambda)
- **Never throw** — data source methods build; execution happens later
- **Construct network input types here**, not in the repository — keeps the network vocabulary out of `data/repository/`
- **`@Singleton`** — clients are expensive; hold one instance per data source
- **Feature-agnostic method names** — `getProfile(id)` not `getProfileFromEmployeeApi(id)`; the client choice is the auth split

## Retrofit variant

For a Retrofit-based project, the data source wraps a Retrofit service and returns a suspending function reference or a `NetworkResult<T>` wrapper:

```kotlin
interface UserRemoteDataSource {
    suspend fun getProfile(id: String): Response<GetProfileResponse>
    suspend fun updateProfile(id: String, body: ProfileBody): Response<UpdateProfileResponse>
}

class UserRemoteDataSourceImpl @Inject constructor(
    @param:Named(RETROFIT_USER) private val api: UserApi,
) : UserRemoteDataSource {
    override suspend fun getProfile(id: String) = api.getProfile(id)
    override suspend fun updateProfile(id: String, body: ProfileBody) = api.updateProfile(id, body)
}
```

The `BaseRepository.safeCallSuspend` variant for Retrofit accepts a `suspend () -> Response<T>` lambda and unwraps into `ApiResult<T>`. The layering is identical — only the call type changes.

## Local data sources

Same split idea for local persistence — `LocalDataSource` wraps a DAO (Room) or a `DataStore`. Repositories inject the local data source; the DAO stays hidden.

## Common violations

- `apolloClient.execute()` or `.await()` inside a data source method → return the call, let the repository execute
- Data source method returning a domain model → return the raw network type; mapping is the repo's job
- Data source that imports domain models → remove the imports; data sources speak network vocabulary
- Repository field of type `ApolloClient` / `Retrofit` → inject a data source instead
- One giant `RemoteDataSource` used for every operation → split by auth level
