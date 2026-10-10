# Testing Strategy

> For installing or configuring test infrastructure (frameworks, runners, Robolectric, screenshot tools, coverage), use the `testing-setup` skill. This file covers how to design and write tests.

> Based on the official guides: [Testing fundamentals](https://developer.android.com/training/testing/fundamentals), [What to test](https://developer.android.com/training/testing/fundamentals/what-to-test), [Test doubles](https://developer.android.com/training/testing/fundamentals/test-doubles), [Testing strategies](https://developer.android.com/training/testing/fundamentals/strategies), [Testing coroutines](https://developer.android.com/kotlin/coroutines/test), [Testing flows](https://developer.android.com/kotlin/flow/test).

## The Testing Pyramid

```
       /\
      /  \        UI / E2E Tests (Few)
     /----\       - UI behavior: Robolectric in src/test
    /      \      - E2E: instrumented, high fidelity
   /--------\
  /          \    Integration Tests (Some)
 /------------\   - Local or instrumented
/              \  - Database, network, navigation
----------------
      Unit Tests (Many)
      - Local JVM (src/test)
      - Fast, isolated
```

**Fidelity vs speed:** big tests are closer to production but slower and flakier. Bugs caught by unit tests cost minutes to fix; bugs caught by E2E tests cost days. Choose the **lowest layer that gives the feedback you need**.

## Where Tests Run

| Location | Source set | Runs on | Use for |
|----------|------------|---------|---------|
| Local | `src/test` | JVM on your machine | Unit tests, ViewModels, repositories; Robolectric for UI/framework |
| Instrumented | `src/androidTest` | Device or emulator | Real Android APIs, instrumented UI tests when requested, E2E |

## What to Test

**Essential:**
- ViewModels (including their flows and state)
- Data layer: repositories and data sources
- Domain layer: use cases
- Other platform-independent classes (formatters, validators, mappers)
- UI: screen tests and navigation tests for critical user journeys

**Edge cases:** boundaries (zero, negative, max), every network error, malformed data, full storage, configuration changes in the middle of an operation.

**Don't test:**
- The framework or third-party libraries themselves
- Activities/Fragments with unit tests - they should contain no logic; cover them with UI tests

## Test Types

### Unit Tests

Test ONE class or function in isolation.

```kotlin
class OrderTest {
    @Test
    fun calculateTotal_twoItems_returnsSum() {
        val order = Order(items = listOf(OrderItem(Money.of(100)), OrderItem(Money.of(50))))

        val total = order.calculateTotal()

        assertEquals(Money.of(150), total)
    }
}
```

### Integration Tests

Test real implementations working together at a boundary, e.g. a database data source against an in-memory database, or a remote data source against a mock web server.

```kotlin
@RunWith(AndroidJUnit4::class)
class NewsLocalDataSourceTest {
    private lateinit var database: NewsDatabase
    private lateinit var newsLocalDataSource: NewsLocalDataSource

    @Before
    fun createDatabase() {
        database = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            NewsDatabase::class.java,
        ).build()
        newsLocalDataSource = NewsLocalDataSource(database.articleDao())
    }

    @After
    fun closeDatabase() = database.close()

    @Test
    fun saveNews_thenGetNewsStream_emitsSavedArticles() = runTest {
        newsLocalDataSource.saveNews(listOf(ARTICLE_ENTITY))

        val articles = newsLocalDataSource.getNewsStream().first()

        assertEquals(listOf(ARTICLE_ENTITY), articles)
    }
}
```

*(Room and MockWebServer are examples - use the project's storage and network stack.)*

### UI Tests

UI behavior tests run with Robolectric in `src/test` by default. Put them in `src/androidTest` only when instrumented tests are requested. Use the Compose test rule (Espresso for legacy Views):

```kotlin
class NewsScreenTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun newsScreen_loadingState_showsProgressIndicator() {
        composeTestRule.setContent {
            NewsScreen(uiState = NewsUiState(isFetchingArticles = true), onRefresh = {})
        }

        composeTestRule.onNodeWithContentDescription("Loading news").assertIsDisplayed()
    }
}
```

- Match with semantic matchers first (text, content description, role). Use `testTag` when finding the node would otherwise need more than three matchers
- Verify state restoration on screens that hold user input

### Testing Layers in Large Projects

| Layer | Scope | Network | Runs on | When |
|-------|-------|---------|---------|------|
| Unit | One class | None | Local | Every commit |
| Component | One module | None | Local / Robolectric / emulator | Every commit |
| Feature | Several modules | Faked | Local / emulator | Pre-merge |
| Application | Debug app | Faked or staging | Emulator / devices | Post-merge |
| Release candidate | Minified release build | Production | Many devices | Pre-release |

---

## Arrange-Act-Assert (AAA)

Structure EVERY test this way:

```kotlin
@Test
fun calculateTotal_premiumUser_appliesTwentyPercentDiscount() {
    // Arrange - Set up the test world
    val cart = Cart(user = User(isPremium = true))
    cart.addItem(CartItem(price = Money.of(100)))

    // Act - Execute the behavior under test
    val total = cart.calculateTotal()

    // Assert - Verify the expected outcome
    assertEquals(Money.of(80), total)
}
```

### Writing AAA Backwards

Sometimes easier to write in reverse:

1. **Assert first** - What do you want to verify?
2. **Act** - What action produces that result?
3. **Arrange** - What setup is needed?

---

## Test Naming

Format: `subjectUnderTest_trigger_expectedResult`. No backtick names with spaces (not fully supported on Android).

```kotlin
// BAD: Abstract, technical
@Test fun worksCorrectly()
@Test fun handlesEdgeCase()

// GOOD: Concrete examples, domain language
@Test fun calculateDiscount_premiumUser_returnsTwentyPercent()
@Test fun checkout_emptyCart_showsEmptyCartMessage()
@Test fun isPalindrome_racecar_returnsTrue()
```

Test classes are named `<ClassUnderTest>Test`.

---

## Test Doubles

Official definitions:

| Type | Description |
|------|-------------|
| **Fake** | Working implementation unsuitable for production (e.g. in-memory repository). **Preferred.** |
| **Mock** | Programmed behavior + expectations about interactions; fails if they aren't met |
| **Stub** | Programmed behavior, no interaction expectations |
| **Dummy** | Passed but never used |
| **Spy** | Wraps a real object and records calls (usually avoided) |
| **Shadow** | A Robolectric fake of a framework class |

### Fake (preferred)

```kotlin
class FakeUserRepository : UserRepository {
    private val users = MutableStateFlow<Map<UserId, User>>(emptyMap())

    override suspend fun saveUser(user: User) {
        users.update { current -> current + (user.id to user) }
    }

    override fun getUserStream(id: UserId): Flow<User?> = users.map { current -> current[id] }
}
```

- Named with the `Fake` prefix
- Live in test source sets (or a shared `:core:testing` module), never in production code
- Need no mocking framework and survive refactoring of the subject

### Mock

Use when the interaction itself is the behavior:

```kotlin
// MockK example - only if the project already uses MockK
val analytics = mockk<AnalyticsLogger>(relaxed = true)
checkoutViewModel.confirmPurchase()
verify(exactly = 1) { analytics.logPurchase(any()) }
```

A hand-written fake that records events is an equally valid, library-free alternative. **Follow the project's existing tooling.**

### Dummy

```kotlin
NewsScreen(uiState = NewsUiState(), onRefresh = {}) // onRefresh is a dummy
```

---

## Testing Strategies by Layer

### ViewModels

```kotlin
class NewsViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    private val newsRepository = FakeNewsRepository()
    private val viewModel = NewsViewModel(newsRepository)

    @Test
    fun uiState_repositoryEmitsArticles_showsArticles() = runTest {
        backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) { viewModel.uiState.collect {} }

        newsRepository.emit(listOf(ARTICLE))

        assertEquals(listOf(ARTICLE.toUiState()), viewModel.uiState.value.newsItems)
    }
}
```

- Replace `Dispatchers.Main` with `MainDispatcherRule` (see [coroutines-flow.md](coroutines-flow.md))
- Assert on `uiState.value`; start an empty collector when the state uses `stateIn(WhileSubscribed)`
- Use `SavedStateHandle(mapOf("id" to "42"))` directly in tests

### Use Cases

```kotlin
class GetLatestNewsWithAuthorsUseCaseTest {
    @Test
    fun invoke_articlesWithAuthors_combinesThem() = runTest {
        val useCase = GetLatestNewsWithAuthorsUseCase(
            newsRepository = FakeNewsRepository(articles = listOf(ARTICLE)),
            authorsRepository = FakeAuthorsRepository(authors = listOf(AUTHOR)),
            defaultDispatcher = StandardTestDispatcher(testScheduler),
        )

        val result = useCase()

        assertEquals(listOf(ArticleWithAuthor(ARTICLE, AUTHOR)), result)
    }
}
```

### Repositories

- Unit test with fake data sources (business logic, mapping, conflict resolution, error handling)
- Integration test data sources against real implementations (in-memory database, mock web server)

### UI

- Screen tests on stateless composables with fake state
- Navigation tests for critical journeys
- Screenshot tests (if the project uses them) at different font scales, themes, and locales

---

## Contract Tests

Verify that every implementation honors the same interface:

```kotlin
abstract class UserRepositoryContractTest {
    abstract fun createRepository(): UserRepository

    @Test
    fun saveUser_thenGetUserStream_emitsSavedUser() = runTest {
        val repository = createRepository()

        repository.saveUser(USER)

        assertEquals(USER, repository.getUserStream(USER.id).first())
    }

    @Test
    fun getUserStream_unknownId_emitsNull() = runTest {
        assertNull(createRepository().getUserStream(UserId("unknown")).first())
    }
}

class FakeUserRepositoryTest : UserRepositoryContractTest() {
    override fun createRepository(): UserRepository = FakeUserRepository()
}
```

This guarantees the fake behaves like the real implementation.

---

## Test Data

Kotlin default arguments and `copy()` replace most test builders:

```kotlin
fun anOrder(
    id: OrderId = OrderId("order-1"),
    items: List<OrderItem> = emptyList(),
    status: OrderStatus = OrderStatus.Pending,
): Order = Order(id = id, items = items, status = status)

// Usage - only what matters for this test is visible
val paidOrder = anOrder(items = listOf(OrderItem(Money.of(100)))).copy(status = OrderStatus.Paid)
```

---

## Common Testing Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| Testing implementation | Brittle tests | Test behavior only |
| Too many mocks | Tests prove nothing | Prefer fakes and real objects |
| Shared state | Flaky tests | Isolate each test |
| No assertions | False confidence | Always assert something meaningful |
| Testing trivial code | Wasted effort | Focus on logic and edge cases |
| Hardcoded dispatchers | Non-deterministic tests | Inject dispatchers, use `runTest` |
| Asserting every StateFlow emission | Flaky (conflation) | Assert on `.value` |
| Unit testing Activities/Fragments | Logic in the wrong place | Move logic to ViewModel; UI test the screen |
