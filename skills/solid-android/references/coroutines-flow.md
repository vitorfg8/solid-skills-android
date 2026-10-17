# Coroutines & Flow

> Based on the official guides: [Coroutines best practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices), [Coroutines](https://developer.android.com/kotlin/coroutines), [Advanced coroutines](https://developer.android.com/kotlin/coroutines/coroutines-adv), [Lifecycle-aware coroutines](https://developer.android.com/topic/libraries/architecture/coroutines), [Flow](https://developer.android.com/kotlin/flow), [StateFlow and SharedFlow](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow), [Testing coroutines](https://developer.android.com/kotlin/coroutines/test), [Testing flows](https://developer.android.com/kotlin/flow/test).

## Best Practices

### 1. Inject dispatchers

```kotlin
// BAD: hardcoded in the function body, untestable
class NewsRemoteDataSource(private val blockingNewsApi: BlockingNewsApi) {
    suspend fun fetchLatestNews(): List<NetworkArticle> =
        withContext(Dispatchers.IO) { blockingNewsApi.fetchLatestNews() }
}

// GOOD: injected through the constructor (a default value is fine)
class NewsRemoteDataSource(
    private val blockingNewsApi: BlockingNewsApi,
    private val ioDispatcher: CoroutineDispatcher = Dispatchers.IO,
) {
    suspend fun fetchLatestNews(): List<NetworkArticle> =
        withContext(ioDispatcher) { blockingNewsApi.fetchLatestNews() }
}
```

Tests can then pass a `TestDispatcher`.

### 2. Suspend functions are main-safe

The class that does blocking work moves it off the main thread. Callers never choose a dispatcher. Libraries that are already main-safe (e.g. Room, Retrofit and Ktor `suspend` APIs) don't need `withContext`.

| Dispatcher | Use for |
|------------|---------|
| `Main` | UI and quick, non-blocking work |
| `IO` | Disk and network I/O |
| `Default` | CPU-intensive work (sorting, parsing large JSON, diffing) |

### 3. The ViewModel creates coroutines

```kotlin
// BAD: the UI must call this from a coroutine and manage its lifecycle
class LatestNewsViewModel(private val getLatestNews: GetLatestNewsUseCase) : ViewModel() {
    suspend fun loadNews(): List<Article> = getLatestNews()
}

// GOOD: the ViewModel launches and exposes state
class LatestNewsViewModel(private val getLatestNews: GetLatestNewsUseCase) : ViewModel() {
    private val _uiState = MutableStateFlow<LatestNewsUiState>(LatestNewsUiState.Loading)
    val uiState: StateFlow<LatestNewsUiState> = _uiState.asStateFlow()

    fun loadNews() {
        viewModelScope.launch {
            _uiState.value = LatestNewsUiState.Success(getLatestNews())
        }
    }
}
```

Views/composables don't launch coroutines for business logic (they may for UI work, e.g. animations or scrolling).

### 4. Don't expose mutable types

Keep `MutableStateFlow`/`MutableSharedFlow` private; expose `StateFlow`/`SharedFlow`.

### 5. Data and domain layers expose `suspend` and `Flow`

```kotlin
interface NewsRepository {
    fun getNewsStream(): Flow<List<Article>>    // changes over time
    suspend fun getLatestNews(): List<Article>  // one-shot
    suspend fun refreshNews()                   // one-shot
}
```

The caller controls lifecycle and cancellation.

### 6. Creating coroutines in the data/domain layer

- **Work tied to the caller** (e.g. parallel requests): `coroutineScope { }` / `supervisorScope { }` with `async`/`await`
- **Work that must outlive the screen** (e.g. save that must complete): an injected external scope owned by the `Application` (or a navigation-graph-scoped holder)

```kotlin
class ArticlesRepository(
    private val articlesDataSource: ArticlesDataSource,
    private val externalScope: CoroutineScope,
) {
    suspend fun bookmarkArticle(article: Article) {
        externalScope.launch { articlesDataSource.bookmarkArticle(article) }.join()
    }
}
```

Work that must survive process death belongs in persistent work (e.g. WorkManager), not a coroutine scope.

### 7. Avoid `GlobalScope`

It hardcodes the scope, can't be controlled in tests, and ignores structured concurrency. Inject a `CoroutineScope` instead.

### 8. Make coroutines cancellable

Cancellation is cooperative. In long loops or blocking work call `ensureActive()` (or `yield()`). `withContext` and `delay` already check for cancellation.

### 9. Watch out for exceptions

```kotlin
viewModelScope.launch {
    try {
        loginRepository.login(username, token)
    } catch (exception: IOException) {
        _uiState.update { it.copy(errorMessage = UserMessage.NetworkError) }
    }
}
```

- Uncaught exceptions crash the app
- Catch **specific** exceptions, not `Exception`/`Throwable`
- **Never swallow `CancellationException`** - if you must catch broadly, rethrow it
- `async` defers exceptions until `await()`; don't start `async` from regular functions

## Structured Concurrency

- A scope tracks its coroutines; cancelling the scope cancels its children
- `coroutineScope` waits for all children and propagates their failures
- `supervisorScope` lets children fail independently
- A cancelled scope can't start new coroutines
- `CoroutineContext` = `Job` + `CoroutineDispatcher` + `CoroutineName` + `CoroutineExceptionHandler`

## Lifecycle-Aware Scopes

| Scope | Cancelled when | Use for |
|-------|----------------|---------|
| `viewModelScope` | ViewModel is cleared | Screen business logic |
| `lifecycleScope` / `viewLifecycleOwner.lifecycleScope` | Lifecycle is destroyed | UI work in Views |
| `rememberCoroutineScope()` | Composable leaves composition | Launching from Compose event handlers |
| Injected application scope | Never (app process) | Work that must outlive the screen |

Collecting in Views: always inside `repeatOnLifecycle(Lifecycle.State.STARTED)` (see [ui-layer.md](ui-layer.md)).

## Flow

- **Producer** emits → optional **intermediaries** transform (`map`, `filter`, `onEach`) → **consumer** collects
- Flows are **cold**: each collector re-runs the producer. Share with `shareIn`/`stateIn`
- Don't `emit` from a different context inside `flow { }` (no `withContext`/`launch` around `emit`)
- `flowOn(dispatcher)` changes the context of everything **above** it; put it in data sources
- `catch { }` handles upstream exceptions and may `emit` a fallback

```kotlin
class NewsRemoteDataSource(
    private val newsApi: NewsApi,
    private val ioDispatcher: CoroutineDispatcher,
) {
    fun getLatestNewsStream(): Flow<List<ArticleHeadline>> = flow {
        while (true) {
            emit(newsApi.fetchLatestNews())
            delay(REFRESH_INTERVAL_MILLIS)
        }
    }.flowOn(ioDispatcher)

    private companion object {
        const val REFRESH_INTERVAL_MILLIS = 5_000L
    }
}
```

**Wrapping callback APIs:**

```kotlin
fun locationUpdatesStream(): Flow<Location> = callbackFlow {
    val callback = LocationCallback { location -> trySend(location) }
    locationClient.requestUpdates(callback)
    awaitClose { locationClient.removeUpdates(callback) }
}
```

Always end `callbackFlow` with `awaitClose` to unregister.

## StateFlow and SharedFlow

| | `StateFlow` | `SharedFlow` |
|--|--|--|
| Initial value | Required | No |
| Replay | Latest value only (conflated) | Configurable `replay` |
| Typical use | UI state | Broadcasts to multiple collectors |

- Both are **hot**
- `StateFlow` doesn't stop collecting on `STOPPED` like `LiveData` - collect with `repeatOnLifecycle` / `collectAsStateWithLifecycle`
- `stateIn(scope, SharingStarted.WhileSubscribed(5_000), initialValue)` converts a cold flow into UI state
- `shareIn(scope, started, replay)`: the scope must outlive all consumers
- `SharingStarted.Eagerly` starts immediately; `Lazily` starts on first subscriber and never stops; `WhileSubscribed` stops when nobody collects

## Testing Coroutines

```kotlin
class MainDispatcherRule(
    val testDispatcher: TestDispatcher = UnconfinedTestDispatcher(),
) : TestWatcher() {
    override fun starting(description: Description) = Dispatchers.setMain(testDispatcher)
    override fun finished(description: Description) = Dispatchers.resetMain()
}

class NewsViewModelTest {
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()

    @Test
    fun loadNews_repositoryReturnsArticles_showsArticles() = runTest {
        // Arrange
        val viewModel = LatestNewsViewModel(GetLatestNewsUseCase(FakeNewsRepository(articles = FAKE_ARTICLES)))
        // Act
        viewModel.loadNews()
        // Assert
        assertEquals(LatestNewsUiState.Success(FAKE_ARTICLES), viewModel.uiState.value)
    }
}
```

- `runTest` skips `delay`s with virtual time and waits for child coroutines
- `StandardTestDispatcher` queues coroutines (run with `advanceUntilIdle()`/`runCurrent()`); `UnconfinedTestDispatcher` runs them eagerly
- **One scheduler per test**: create extra dispatchers with `StandardTestDispatcher(testScheduler)`
- Replace `Dispatchers.Main` in local tests with `MainDispatcherRule`; don't do it in instrumented tests
- Inject `TestScope`/`backgroundScope` into classes that take a `CoroutineScope`
- Needing `advanceUntilIdle()` after something like `initialize()` is a design smell - make it `suspend` or lazy

## Testing Flows

- **Subject consumes a flow:** inject a fake repository whose flow the test controls

```kotlin
class FakeNewsRepository(articles: List<Article> = emptyList()) : NewsRepository {
    private val newsFlow = MutableStateFlow(articles)

    fun emit(articles: List<Article>) {
        newsFlow.value = articles
    }

    override fun getNewsStream(): Flow<List<Article>> = newsFlow

    override suspend fun getLatestNews(): List<Article> = newsFlow.value

    override suspend fun refreshNews() = Unit
}
```

- **Subject produces a flow:** `first()`, `take(n).toList()`, `toList()` (finite flows only)
- **Continuous collection:** `backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) { flow.toList(values) }`
- **StateFlow:** assert on `.value` (it's conflated, intermediate values may be skipped)
- **`stateIn` with `WhileSubscribed`:** start an empty collector first, otherwise the state never updates:

```kotlin
backgroundScope.launch(UnconfinedTestDispatcher(testScheduler)) { viewModel.uiState.collect {} }
```

- Third-party helpers (e.g. Turbine) are optional; follow what the project already uses
