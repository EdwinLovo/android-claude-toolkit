---
name: data-sources
description: RemoteDataSource is a call factory (no execution, no mapping); split by auth level via named clients. Applies to Apollo/GraphQL-style projects where a data-source layer exists between repo and client.
paths:
  - "**/data/remote/datasource/**/*.kt"
  - "**/data/local/datasource/**/*.kt"
---

# Data sources

> **When does this rule apply?**
> Only when the project uses a **data-source layer** between repositories and the network client — typical for Apollo / GraphQL, where multiple named `ApolloClient` instances need to be split by auth level.
>
> **Retrofit projects don't have a data-source layer.** A Retrofit `interface UserApi { @GET ... }` already provides a clean per-endpoint contract, so repositories inject the API service directly — no wrapper. See `rules/repositories.md` for that pattern.

A `RemoteDataSource` is a **call factory**. Its methods build and return an executable call object (e.g. Apollo `ApolloCall<D>`). **It does not execute, and does not map to domain types.** Repositories execute the call and pass a mapper.

This split lets repositories reuse `BaseRepository.safeApolloCallSuspend` for consistent error handling, and lets tests stub the data source without a network layer.

## Split by auth level, not by feature

The primary reason a data-source layer exists is to bind each auth context to a dedicated `ApolloClient`. Create one data source **per auth level**, each holding a named client:

| Named client | Data source | Used for |
|---|---|---|
| `"public"` | `PublicRemoteDataSource` | Unauthenticated operations (sign-in, device pairing) |
| `"device"` | `DeviceRemoteDataSource` | Device-authenticated (employee sign-in) |
| `"user"` / `"employee"` | `<User>RemoteDataSource` | User-authenticated (main app operations) |
| `"public"` (reused) | `RefreshRemoteDataSource` | Token-refresh mutations that must not carry an auth header |

Split by feature would force every data source to hold every client — not what you want.

## Interface

Methods are **non-suspend** and return the executable call object:

```kotlin
package <PKG_ROOT>.data.remote.datasource

interface EmployeeRemoteDataSource {
    fun getProfile(id: String): ApolloCall<GetProfileQuery.Data>
    fun updateProfile(id: String, input: ProfileInput): ApolloCall<UpdateProfileMutation.Data>
}
```

## Implementation

Holds the **named** client. Constructs Apollo input types (`Optional`, `UuidInput`) here. Returns the call — no `execute()`, no mapping.

```kotlin
package <PKG_ROOT>.data.remote.datasource

@Singleton
class EmployeeRemoteDataSourceImpl @Inject constructor(
    @param:Named(APOLLO_CLIENT_EMPLOYEE) private val apolloClient: ApolloClient,
) : EmployeeRemoteDataSource {

    override fun getProfile(id: String): ApolloCall<GetProfileQuery.Data> =
        apolloClient.query(GetProfileQuery(id = Uuid(id)))

    override fun updateProfile(id: String, input: ProfileInput): ApolloCall<UpdateProfileMutation.Data> =
        apolloClient.mutation(UpdateProfileMutation(id = Uuid(id), input = input.toNetworkInput()))
}
```

## Hard rules

- **`RemoteDataSource` is the ONLY class that imports `ApolloClient`.** Repositories never see it.
- **Non-suspend method signatures** — return the call, don't `execute()` it
- **Never map to domain types inside the data source** — that's the repository's job via the `mapper` lambda in `safeApolloCallSuspend`
- **Never throw** — data source methods build; execution happens later
- **Construct Apollo input types here**, not in the repository — keeps GraphQL vocabulary out of `data/repository/`
- **`@Singleton`** — clients are expensive; hold one instance per data source
- **Feature-agnostic method names** — `getProfile(id)`, not `getProfileFromEmployeeApi(id)`; the auth split is already encoded in *which* data source you inject
- **Location**: `data/remote/datasource/` — both interface and impl live in this flat folder

## Local data sources — usually not needed

Room DAOs already provide a clean per-table interface, same reasoning as Retrofit's `interface UserApi { }`. Repositories typically inject the DAO directly — no `LocalDataSource` wrapper.

Introduce a `LocalDataSource` only when you have a real reason: combining several DAOs behind one façade, wrapping a mix of Room + DataStore + SharedPreferences, or when tests need a swap point the DAO doesn't provide.

## Common violations

- `apolloClient.execute()` / `.await()` inside a data source method → return the call, let the repository execute
- Data source method returning a domain model → return the raw Apollo type; mapping is the repo's job
- Data source importing domain models → remove the imports; data sources speak GraphQL vocabulary
- Repository field of type `ApolloClient` → inject a `RemoteDataSource` instead
- One giant `RemoteDataSource` covering every operation → split by auth level
- File placed under `data/remote/graphql/` or `data/remote/` root instead of `data/remote/datasource/` → move it
- Retrofit `interface UserApi { @GET ... }` wrapped in a `UserRemoteDataSource` that just forwards each method → drop the wrapper, inject `UserApi` directly into the repository (see `rules/repositories.md`)
