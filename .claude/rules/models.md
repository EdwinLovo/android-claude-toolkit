---
name: models
description: Domain models are plain data classes with no framework deps; grouped by feature under domain/model/
paths:
  - "**/domain/model/**/*.kt"
---

# Domain models

Plain `data class` types living under `domain/model/`. No Android, no Compose, no networking library imports — `domain/` compiles standalone.

## Organization

```
domain/model/
├── response/<feature>/     ← types returned by repositories on read paths
│   ├── auth/DeviceAuth.kt
│   ├── auth/EmployeeAuth.kt
│   └── catalog/Product.kt
├── request/                ← types passed INTO repositories on write paths
│   ├── CreateOrderRequest.kt
│   └── PackageRedemptionRequest.kt
├── uimodel/                ← display-oriented models built by use cases
│   └── CatalogListItem.kt
└── <feature>/              ← feature-shared value types (enums, IDs, etc.)
    └── CatalogTab.kt
```

- **Response models** — one file per aggregate resource, grouped under `response/<feature>/`
- **Request models** — created when a mutation takes more than ~3 params, or when the shape is reused. Small mutations can take plain params on the repository interface.
- **UI models** — output of use cases that flatten multiple response models for a screen (e.g. a list item that pulls fields from Product + Inventory)
- **Feature enums / value types** — go in `domain/model/<feature>/`

## Shape

```kotlin
package <PKG_ROOT>.domain.model.response.catalog

data class Product(
    val uuid: String,
    val name: String,
    val priceCents: Int,
    val inventory: Int?,
) {
    val isInStock: Boolean get() = (inventory ?: 0) > 0
    val displayPrice: String get() = "$${priceCents / 100.0}"
}
```

- **`data class`** — never `class` alone
- **No `"Response"` / `"Dto"` suffix** — call it what the domain calls it (`Product`, not `ProductResponse`)
- **Value-typed fields** — `String`, `Int`, `Boolean`, `enum class`, nested `data class`, `List<T>` of value types
- **Computed properties allowed** — they're pure Kotlin, they belong on the model
- **No `@Serializable` / `@Parcelable` in domain** — apply serialization/parcel annotations to network DTOs or presentation-layer wrappers, not domain models

## Hard rules

- **No framework imports** — grep the file for `androidx.`, `android.`, `com.apollographql.`, `retrofit2.`, `com.squareup.moshi.`, `com.google.gson.` — none should appear
- **No mutable state** — `val` only; if you need to change a field, `.copy(field = new)` returns a new instance
- **`equals()` / `hashCode()` come free** from `data class` — never override them
- **No `Context` field** — a domain model that needs `Context` is a UI concern, move it to `presentation/`

## Nested vs flat

If a response has a natural nested shape (an order with line items), reflect it in the domain:

```kotlin
data class Order(
    val uuid: String,
    val total: Int,
    val items: List<OrderItem>,
)

data class OrderItem(
    val uuid: String,
    val productId: String,
    val quantity: Int,
    val priceCents: Int,
)
```

Group these in the same file if they only ever appear together (`Order.kt` holding both). Split into separate files when a nested type is used independently.

## Common violations

- `class Product(...)` (missing `data`) → make it `data class`
- Field marked `var` → make it `val`, use `.copy()` to change
- `ProductResponse` name → rename to `Product` (or a more specific noun)
- Model in `presentation/` reading domain-level fields → move to `domain/model/`
- Domain model importing Compose / Retrofit / Apollo types → violation, refactor
- File under `domain/` containing both a domain model and its mapper from a network type → mapper belongs in `data/mappers/`
