# UI Layer

> Based on the official guides: [UI layer](https://developer.android.com/topic/architecture/ui-layer), [UI events](https://developer.android.com/topic/architecture/ui-layer/events), [State holders](https://developer.android.com/topic/architecture/ui-layer/stateholders), [State production](https://developer.android.com/topic/architecture/ui-layer/state-production), [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel), [Saving UI states](https://developer.android.com/topic/libraries/architecture/saving-states), [Runtime changes](https://developer.android.com/guide/topics/resources/runtime-changes).

The UI layer has two parts: **UI elements** (composables, or Views in legacy code) that render state, and **state holders** (`ViewModel` or plain classes) that produce it.

## Unidirectional Data Flow (UDF)

```
        state (UiState)
ViewModel ───────────────► UI
    ▲                        │
    └────────────────────────┘
        events (function calls)
```

- State flows **down**, events flow **up**
- Only the owner of the state changes it - the UI never mutates `UiState`
- The ViewModel exposes state through an observable holder and receives actions as function calls

## UI State

Model screen state as an **immutable** type named `<Feature>UiState`.

```kotlin
data class NewsUiState(
    val isSignedIn: Boolean = false,
    val isPremium: Boolean = false,
    val isFetchingArticles: Boolean = false,
    val newsItems: List<NewsItemUiState> = emptyList(),
    val userMessages: List<UserMessage> = emptyList(),
)

// Derived values as extension properties, so they never go out of sync
val NewsUiState.canBookmarkNews: Boolean get() = isSignedIn && isPremium
```

Use a `sealed interface` when states are mutually exclusive:

```kotlin
sealed interface ProfileUiState {
    data object Loading : ProfileUiState
    data class Success(val profile: Profile) : ProfileUiState
    data class Error(val message: UserMessage) : ProfileUiState
}
```

**Rules:**
- One state object for related data (keeps it consistent); split streams only for unrelated data or very different update frequencies
- Never put mutable types inside `UiState` (e.g. `PagingData` gets its own stream)
- Keep names UI-agnostic in the ViewModel (`isFetchingArticles`, not `isProgressBarVisible`)

## Exposing State

Private mutable backing property, public read-only type:

```kotlin
class NewsViewModel(
    private val newsRepository: NewsRepository,
) : ViewModel() {

    private val _uiState = MutableStateFlow(NewsUiState())
    val uiState: StateFlow<NewsUiState> = _uiState.asStateFlow()

    fun refreshNews() {
        _uiState.update { it.copy(isFetchingArticles = true) }
        viewModelScope.launch {
            val newsItems = newsRepository.getLatestNews()
            _uiState.update { it.copy(isFetchingArticles = false, newsItems = newsItems.toUiState()) }
        }
    }
}
```

- `_uiState.update { }` is atomic and thread-safe - prefer it over `_uiState.value = ...`
- If the project uses `LiveData`, keep `LiveData` with the same backing-property pattern (`private val _uiState = MutableLiveData<...>()`). `StateFlow` is the documented default for new code; don't migrate unless asked.

## State Production

| Inputs | API |
|--------|-----|
| One-shot calls (user actions, `suspend` functions) | `MutableStateFlow` + `update {}` inside `viewModelScope.launch` |
| Streams (`Flow` from repositories) | `combine(...)` + `stateIn(...)` |
| Both | Pipe one-shot results into a private `MutableStateFlow` and `combine` it with the streams |

```kotlin
val uiState: StateFlow<TopicUiState> = combine(
    topicsRepository.getTopicStream(topicId),
    userDataRepository.getFollowedTopicIdsStream(),
) { topic, followedTopicIds ->
    TopicUiState.Success(topic = topic, isFollowed = topic.id in followedTopicIds)
}.stateIn(
    scope = viewModelScope,
    started = SharingStarted.WhileSubscribed(STOP_TIMEOUT_MILLIS),
    initialValue = TopicUiState.Loading,
)

private companion object {
    const val STOP_TIMEOUT_MILLIS = 5_000L
}
```

- `WhileSubscribed(5_000)` keeps the pipeline alive through configuration changes but stops it in the background
- **Initialize lazily**: prefer `stateIn` (work starts when the UI subscribes) over launching work in `init`. If you need explicit initialization, make it idempotent
- Long-running work (≥ a few seconds) that must finish regardless of the screen does not belong in `viewModelScope` - use the data layer with an external scope or persistent work (e.g. WorkManager)
- Catch specific exceptions in `viewModelScope` coroutines and map them to state; always rethrow `CancellationException`

## UI Events

**UI events** (user actions) are either handled directly by the UI (pure UI logic: expand, scroll, navigate on tap) or forwarded to the ViewModel when they involve business logic.

**ViewModel events always become UI state.** Don't model "show a snackbar" or "navigate" as one-off events in a `Channel`/`SharedFlow`: delivery isn't guaranteed when the producer outlives the consumer, so events get lost. Think about how the action affects the state, not about what the UI should do.

```kotlin
data class LoginUiState(
    val isLoggingIn: Boolean = false,
    val isUserLoggedIn: Boolean = false,
    val errorMessage: UserMessage? = null,
)

class LoginViewModel(/* ... */) : ViewModel() {
    // ...
    fun errorMessageShown() {
        _uiState.update { it.copy(errorMessage = null) }
    }
}
```

The UI shows the message, then **reports consumption** (`errorMessageShown()`), which is just another event. Navigation after business validation works the same way: the UI observes `isUserLoggedIn` and navigates. Navigation itself is UI logic.

**Event handlers in lists:** never pass the ViewModel to items or adapters - pass lambdas named `on<Verb><Target>`:

```kotlin
NewsItem(item = item, onBookmarkClick = { viewModel.bookmark(item.id) })
```

## State Holders

| | Business logic state holder | UI logic state holder |
|--|--|--|
| Type | `ViewModel` | Plain class (e.g. `SearchBarState`) |
| Scope | Screen / navigation destination | Same as the UI element |
| Survives configuration change | Yes | No (use `rememberSaveable`/saved state for small pieces) |
| May reference `Context`, `Resources`, `NavController` | **No** | Yes |
| Reusable | No - one per screen | Yes |

**Rules:**
- **Business logic** (product requirements) lives in the domain/data layer and is applied before **UI logic** (how to display: strings, navigation, scrolling)
- Hold state as low as possible while keeping proper ownership
- A state holder only depends on holders with an equal or shorter lifetime
- Never pass a ViewModel into another state holder, composable, or class - pass only the `StateFlow`s and lambdas needed

## ViewModel Rules

- **Screen-level only** - never for reusable components
- **Never** hold `Context`, `Resources`, `View`, `Activity`, `Fragment`, or `Lifecycle` - avoid `AndroidViewModel`
- Expose data or resource IDs, not localized text resolved with `Resources`. *(Inference from the docs: resolved strings go stale after a language change because the ViewModel survives recreation.)*
- Launch coroutines in `viewModelScope`, or inject a `CoroutineScope` for tests
- Release resources in `onCleared()` or via `Closeable`s passed to the constructor
- Create ViewModels with constructor dependencies through the project's DI (e.g. `@HiltViewModel`, Koin `viewModel {}`) or a `ViewModelProvider.Factory` (`viewModelFactory { initializer { ... } }`) kept next to the ViewModel

```kotlin
class DetailViewModel(
    private val savedStateHandle: SavedStateHandle,
    private val detailRepository: DetailRepository,
) : ViewModel() {

    val query: StateFlow<String> = savedStateHandle.getStateFlow(QUERY_KEY, "")

    fun updateQuery(newQuery: String) {
        savedStateHandle[QUERY_KEY] = newQuery
    }

    private companion object {
        const val QUERY_KEY = "query"
    }
}
```

## Surviving Configuration Changes and Process Death

| | ViewModel | Saved state (`SavedStateHandle`, `rememberSaveable`) | Persistent storage |
|--|--|--|--|
| Survives configuration change | Yes | Yes | Yes |
| Survives system-initiated process death | No | Yes | Yes |
| Survives user dismissal / `finish()` | No | No | Yes |
| Size limit | Memory | Small (Bundle) | Disk |

- Persist app data in the data layer; keep screen state in the ViewModel; keep only small transient inputs (IDs, queries, in-progress text) in saved state
- No bitmaps or large lists in saved state; use `@Parcelize` for small custom types
- **Don't opt out of recreation** with `android:configChanges` or by locking orientation as a quick fix - the Activity can still be recreated for other reasons
- Test rotation, resizing, dark theme, font scale, and language changes

## Consuming State: Compose (default)

```kotlin
@Composable
fun NewsRoute(
    viewModel: NewsViewModel,
    modifier: Modifier = Modifier,
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    NewsScreen(
        uiState = uiState,
        onRefresh = viewModel::refreshNews,
        modifier = modifier,
    )
}

@Composable
fun NewsScreen(
    uiState: NewsUiState,
    onRefresh: () -> Unit,
    modifier: Modifier = Modifier,
) { /* stateless, previewable, testable */ }
```

- The route composable is the only one that knows the ViewModel; everything below receives state and lambdas
- Use `collectAsStateWithLifecycle()` - don't start/stop collection manually

## Consuming State: XML Views (legacy)

```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    viewLifecycleOwner.lifecycleScope.launch {
        viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.uiState.collect { uiState -> render(uiState) }
        }
    }
}
```

- Use `viewLifecycleOwner` in Fragments, never the Fragment itself, for view-related collection
- **Never** collect UI flows with plain `launch`/`launchIn` - they keep running while the view is not visible
- Don't override `onResume`/`onPause` for logic; use `repeatOnLifecycle` or a `DefaultLifecycleObserver`
- With ViewBinding, clear the binding in `onDestroyView`
- Obtain ViewModels with `by viewModels()`, `by activityViewModels()`, or `by navGraphViewModels(R.id.graph)` (or the DI library's equivalent). For Navigation 3 scoping, check the current guidance (`android docs search navigation 3 viewmodel`) or a matching Android skill (`android skills find navigation`).

## Quick Reference

| Do | Don't |
|----|-------|
| Immutable `UiState` + backing `StateFlow` | Expose `MutableStateFlow` |
| Turn ViewModel events into state | One-off event channels for must-not-miss events |
| Lifecycle-aware collection | `launch { flow.collect {} }` in the UI |
| Pass state + lambdas down | Pass the ViewModel down |
| `SavedStateHandle` for small inputs | Bitmaps/lists in saved state |
| Screen-level ViewModels | ViewModels for reusable widgets |
