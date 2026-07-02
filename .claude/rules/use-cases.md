---
name: use-cases
description: Only create a use case when logic is non-trivial or reused across multiple ViewModels
paths:
  - "**/domain/usecase/**/*.kt"
---

# Use cases

> **Only create a use case when the logic is non-trivial or reused across multiple ViewModels.**
> A ViewModel that simply calls a single repository method does not need a use case in between.
>
> Reference: [Google — Domain Layer](https://developer.android.com/topic/architecture/domain-layer)

## When to create one

Create a use case when at least **one** applies:
- The logic combines two or more repositories (e.g. fetch user, then fetch that user's orders, then merge)
- The logic contains real business rules (calculations, validation, business-workflow state transitions)
- Two or more ViewModels need the exact same repository invocation
- The logic is hard to test in isolation as part of a VM

## When NOT to create one

Skip the use case if the ViewModel would just forward args and results:

```kotlin
// ❌ No — pointless indirection
class GetProfileUseCase @Inject constructor(private val repository: ProfileRepository) {
    suspend operator fun invoke(id: String): ApiResult<Profile> = repository.getById(id)
}

// ✅ Yes — VM injects the repository directly
class ProfileViewModel @Inject constructor(private val repository: ProfileRepository) : ViewModel()
```

## Shape

- Single public `operator fun invoke(...)` — call site reads like `useCase(arg1, arg2)`
- Stateless — **no `@Singleton`**, no fields other than injected dependencies
- Returns `ApiResult<T>` for one-shot, `Flow<ApiResult<T>>` for streaming, or `Flow<PagingData<T>>` for paging
- Located at `domain/usecase/<feature>/<Verb><Resource>UseCase.kt`

```kotlin
package <PKG_ROOT>.domain.usecase.<feature>

class GetCatalogProductsUseCase @Inject constructor(
    private val productRepository: ProductRepository,
    private val locationRepository: LocationRepository,
) {
    operator fun invoke(search: String?): Flow<PagingData<CatalogListItem>> {
        val locationUuid = locationRepository.currentLocationUuid()
        return productRepository.getProducts(locationUuid, search)
    }
}
```

## Hard rules

- **Single `operator fun invoke`** — the whole point is a callable object
- **No `@Singleton`** — use cases are stateless and cheap to construct
- **No Compose, no Android, no framework imports** — `domain/` is platform-neutral
- **No error handling that hides `ApiResult.Error`** — pass results through; the VM/UI decides how to display errors
- **Named `<Verb><Resource>UseCase`** — e.g. `GetProductsUseCase`, `SubmitOrderUseCase`, `RefreshCatalogUseCase`. `<Verb>` is a real verb, not a noun.

## Common violations

- Use case that only calls a single repository method with no added logic → delete it, VM injects the repo directly
- Use case with multiple public methods → split into multiple use cases (one `invoke` each)
- Use case marked `@Singleton` → drop it, use cases are stateless
- Use case that throws exceptions → wrap in `ApiResult` and let the caller handle
- Use case named `<Noun>UseCase` (e.g. `ProductUseCase`) → rename with a verb (`GetProductUseCase`, `UpdateProductUseCase`)
- Use case importing Android or Compose → move framework work back to `presentation/`
