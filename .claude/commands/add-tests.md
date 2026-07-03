---
description: Add or update unit tests for ViewModels, repositories, mappers, and use cases changed in the current branch
argument-hint: (no args)
allowed-tools:
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
---

# /add-tests

Add or update unit tests covering work done in the current branch. Focus on `app/src/test/` (unit tests only — not instrumented Compose UI tests).

Detailed patterns are in `.claude/rules/testing.md`.

---

## Steps

### 1 — Identify changed files

Run:
```
git diff main...HEAD --name-only
```

Filter to testable layers:

| Source file pattern | Test type |
|---|---|
| `*ViewModel.kt` | State transition tests — `MainDispatcherRule` + Turbine |
| `*RepositoryImpl.kt` | Repository tests — fake data source, assert `ApiResult` |
| `*UseCase.kt` | Use case tests — stub repository, verify delegation and logic |
| `*Mappers.kt` or files with mapper extension functions | Mapper correctness tests |
| `*Delegate.kt` | Same as VM tests, applied to the delegate's `handleEvent` |

Skip: `*Screen.kt`, `*Route.kt`, `*Contract.kt`, `*Module.kt` — no unit-testable logic.

---

### 2 — Check existing tests

For each changed file, check whether a corresponding test already exists at:
```
app/src/test/java/<PKG_ROOT_PATH>/<same package as source>/
```

If a test file exists, read it before adding cases — avoid duplicating existing coverage.

---

### 3 — Write or update tests

Follow the patterns in `.claude/rules/testing.md` throughout.

**Test function naming**: `subjectOrMethod_condition_expectedBehavior`

**ViewModel test template:**

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
    fun handleEventOnLoadSetsLoadingThenPopulatesState() = runTest {
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

**Mapper test:**
```kotlin
@Test
fun toDeviceAuthMapsAllFields() {
    val data = ValidateDeviceCodeMutation.Data(/* ... */)
    val result = data.toDeviceAuth()
    assertEquals("expected-token", result.token)
}
```

**Repository test:**
```kotlin
@Test
fun signInEmployeeReturnsEmployeeAuthOnSuccess() = runTest {
    val fakeDataSource = FakeDeviceRemoteDataSource(response = successResponse)
    val repository = AuthRepositoryImpl(fakeDataSource)

    val result = repository.signInEmployee("1234")

    assertIs<ApiSuccess<EmployeeAuth>>(result)
}
```

Use **fakes**, not mocking libraries. Fakes live in `app/src/test/java/<PKG_ROOT_PATH>/<layer>/fakes/`.

---

### 4 — Confirm

After writing tests, print:

```
Tests added/updated:
- <TestClassName> (<path>) — N new test(s)
  - testName_condition_expectedBehavior
  - ...

Next: run ./gradlew test to verify all pass.
```
