---
name: testing
description: ViewModel/repository/mapper/use-case unit tests — Turbine + MainDispatcherRule + fakes; subject_condition_expected naming
paths:
  - "**/src/test/**/*.kt"
  - "**/src/androidTest/**/*.kt"
---

# Testing

Unit tests live in `app/src/test/`. Instrumented (Compose UI, Room-on-device) tests live in `app/src/androidTest/`. The vast majority of tests are unit tests — Compose semantic tests are heavier and reserved for tricky UI states.

## Naming

`<subjectOrMethod>_<condition>_<expectedBehavior>`

Examples:
- `handleEvent_onRefresh_setsLoadingThenPopulatesItems`
- `signInEmployee_success_returnsEmployeeAuth`
- `toDeviceAuth_mapsAllFields`
- `getProducts_emptyQuery_debouncesZero`

## What to test in each layer

| Layer | What to test | How |
|---|---|---|
| ViewModel | State transitions in `handleEvent` | `MainDispatcherRule` + Turbine on `uiState.test { }` |
| Repository | Behavior around `ApiResult` — success, error, invalidation emit | Fake data source; assert `ApiResult` variant + emitted `SharedFlow` |
| Mapper | Field-by-field correctness | Instantiate the network type, call `.toDomain()`, assert equality |
| Use case | Composition + business logic | Stub the repositories, verify delegation and combined output |
| Delegate | Same as ViewModel but for the delegate's `handleEvent` | `MainDispatcherRule` + Turbine on delegate's `uiState` |

**Skip:** `*Screen.kt`, `*Route.kt`, `*Contract.kt`, `*Module.kt` — no unit-testable logic there.

## ViewModel test template

```kotlin
class ExampleViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private val fakeRepository = FakeExampleRepository()
    private lateinit var viewModel: ExampleViewModel

    @Before
    fun setUp() {
        viewModel = ExampleViewModel(fakeRepository)
    }

    @Test
    fun handleEvent_onLoad_setsLoadingThenPopulatesState() = runTest {
        viewModel.uiState.test {
            val initial = awaitItem()
            assertFalse(initial.isLoading)

            viewModel.handleEvent(ExampleEvent.OnLoad)

            assertTrue(awaitItem().isLoading)
            val loaded = awaitItem()
            assertFalse(loaded.isLoading)
            assertEquals(fakeRepository.items, loaded.items)

            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

- **`MainDispatcherRule`** replaces `Dispatchers.Main` with `UnconfinedTestDispatcher` — put it in `app/src/test/java/<PKG_ROOT>/testutil/MainDispatcherRule.kt`
- **`uiState.test { }`** is Turbine — sequential `awaitItem()` calls make state transitions explicit
- **Cancel remaining events** at the end — Turbine is strict about unhandled emissions

## Fakes over mocks

Prefer hand-written fakes over Mockito/MockK. Fakes:
- Encode realistic behavior once, reused across many tests
- Don't couple tests to implementation details (no "verify called once with X")
- Compile-check when the interface changes

```kotlin
class FakeExampleRepository(
    var items: List<Item> = emptyList(),
    var errorOnLoad: Boolean = false,
) : ExampleRepository {
    override suspend fun getItems(): ApiResult<List<Item>> =
        if (errorOnLoad) ApiResult.Error(0, "boom") else ApiResult.Success(items)
}
```

Fakes live next to the interface they fake, under `app/src/test/java/<PKG_ROOT>/data/repository/fakes/`.

## Repository test

```kotlin
@Test
fun signInEmployee_success_returnsEmployeeAuth() = runTest {
    val fakeDataSource = FakeDeviceRemoteDataSource(response = successResponse)
    val repository = AuthRepositoryImpl(fakeDataSource)

    val result = repository.signInEmployee("1234")

    assertIs<ApiResult.Success<EmployeeAuth>>(result)
}
```

## Mapper test

```kotlin
@Test
fun toDeviceAuth_mapsAllFields() {
    val data = ValidateDeviceCodeMutation.Data(
        validateDeviceCode = ValidateDeviceCodeMutation.ValidateDeviceCode(
            accessToken = "abc",
            expiresAt = "2030-01-01",
        ),
    )

    val result = data.toDeviceAuth()

    assertEquals("abc", result.accessToken)
    assertEquals("2030-01-01", result.expiresAt)
}
```

## Hard rules

- **No mocking libraries** unless there's genuinely no way to fake — fakes first, mocks last
- **`runTest { }` for coroutine tests** — not `runBlocking`
- **`assertIs<ApiResult.Success<T>>(result)`** is preferred over `result is ApiResult.Success<T> && ...` — the assertion smart-casts and the failure message shows the wrong variant
- **One assertion cluster per test** — testing multiple transitions in one `uiState.test { }` block is fine; asserting five unrelated things is not
- **Fakes go in `src/test/`**, never in `src/main/` — don't ship test doubles in production
- **Compose UI tests live in `src/androidTest/`** — use `createComposeRule()` and `SemanticsNodeInteraction` matchers

## Common violations

- Test named `test1`, `foo`, `worksCorrectly` → rename `<subject>_<condition>_<expected>`
- `Mockito.when(repo.get()).thenReturn(...)` → write a fake instead
- Test that assumes execution order across ViewModels (`shared state`) → each test gets a fresh `viewModel` in `@Before`
- Missing `MainDispatcherRule` → `Dispatchers.Main.immediate` isn't set in unit tests, coroutines from `viewModelScope` never run
- Fake in `src/main/java/.../fakes/` → move to `src/test/java/.../fakes/`
