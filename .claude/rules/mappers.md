---
name: mappers
description: Network↔domain mappers in data/mappers; domain↔domain mappers in domain/model/<area>/<Name>Mappers.kt
paths:
  - "**/data/mappers/**/*.kt"
  - "**/domain/model/**/*Mappers.kt"
---

# Mappers

Two flavors, two locations. The layer the mapper's **input and output** types live in decides where the file goes.

| Mapper direction | Location |
|---|---|
| Network → domain (or domain → network input) | `data/mappers/<feature>/<Resource>Mapper.kt` |
| Domain → domain (e.g. building a request model from response models) | `domain/model/<area>/<Name>Mappers.kt`, next to the target model |

**Why the split?** Placing a domain-only mapper under `data/mappers/` would force the presentation layer to `import data.mappers.*` for pure domain logic — a layering violation. `data/mappers/` is reserved for cross-boundary conversion.

## Network → domain mapper (`data/mappers/<feature>/`)

Extension function on the network type, returning a domain type:

```kotlin
package <PKG_ROOT>.data.mappers.auth

fun SignInEmployeeMutation.Data.toEmployeeAuth(): EmployeeAuth =
    EmployeeAuth(
        accessToken = signInEmployee.accessToken,
        expiresAt = signInEmployee.expiresAt,
        employeeId = signInEmployee.employee.uuid,
    )
```

**Rules:**
- **Extension function on the network type** — `Data.toX()`, not `mapDataToX(data: Data)`
- **Return a domain type** — never a raw network type
- **Named `.to<DomainType>()`** — descriptive, not `.toModel()`
- **One file per Apollo operation type** (or per domain model when multiple operations map to the same domain type)
- **Called from the repository's `mapper` lambda** — never inline the body inside `RepositoryImpl.getX()`

## Domain → domain mapper (`domain/model/<area>/<Name>Mappers.kt`)

Extension function on a domain type, returning another domain type. Lives **next to the target model**:

```kotlin
// domain/model/request/PackageRedemptionRequestMappers.kt
package <PKG_ROOT>.domain.model.request

fun PackageResponse.toRedemptionRequest(quantity: Int): PackageRedemptionRequest =
    PackageRedemptionRequest(
        packageId = uuid,
        clientId = clientUuid,
        quantity = quantity,
    )
```

**Rules:**
- **File name is `<TargetName>Mappers.kt`** — plural, one file per target model regardless of how many source types map into it
- **Located next to the target model** — reader finds it by looking where the output lives
- **No Apollo / Retrofit imports** — if either shows up, the file belongs under `data/mappers/`
- **No `@Composable` and no Android imports** — `domain/` is platform-neutral

## Common violations

- Mapping done inline inside `RepositoryImpl.getX() { ... }` → extract to a `.to<DomainType>()` extension in `data/mappers/`
- Mapper named `mapDataToProfile(data)` → rewrite as `fun ...Data.toProfile()`
- Domain-only mapper placed under `data/mappers/` → move to `domain/model/<area>/<Name>Mappers.kt`
- Mapper returning a network type (`ApolloResponse<X>`) → return the domain type; that's the whole point
- One giant `Mappers.kt` file with 20 mixed mappers → split by target model / feature
- Mapper marked `internal` in `data/` but called from `presentation/` → it's likely a domain mapper miscategorized; move it
