# Android App Architecture

> Based on the official guides: [Guide to app architecture](https://developer.android.com/topic/architecture), [Recommendations](https://developer.android.com/topic/architecture/recommendations), [App fundamentals](https://developer.android.com/guide/components/fundamentals), [Domain layer](https://developer.android.com/topic/architecture/domain-layer), [Data layer](https://developer.android.com/topic/architecture/data-layer), [Offline-first](https://developer.android.com/topic/architecture/data-layer/offline-first), [DataStore](https://developer.android.com/topic/libraries/architecture/datastore), [Persistent work](https://developer.android.com/develop/background-work/background-tasks/persistent), [Views recommendations](https://developer.android.com/topic/architecture/views/recommendations-views).

## The Goal of Architecture

Enable the development team to:
1. **Add** features with minimal friction
2. **Change** existing features safely
3. **Remove** features cleanly
4. **Test** features in isolation
5. **Survive** the Android lifecycle (configuration changes, process death) without losing user data

## Why Android Needs Architecture

App components (`Activity`, `Fragment`, `Service`, `BroadcastReceiver`, `ContentProvider`) are **entry points** that the OS creates and destroys independently, out of order, at any time. Therefore:

- **Never store app data or state in app components**
- Components must not depend on each other
- Keep `Context`, `Toast`, and other framework APIs at the edges (components, UI)
- An `Activity` only hosts UI - a "God Activity" is the most common architecture mistake

## Core Principles

| Principle | Meaning |
|-----------|---------|
| **Separation of concerns** | Each class has one well-defined responsibility; UI classes only handle UI and OS interactions |
| **Drive UI from data models** | Ideally persistent models, independent of UI and lifecycle |
| **Single source of truth (SSOT)** | One owner per data type; only it mutates the data, exposing it as immutable |
| **Unidirectional data flow (UDF)** | State flows down (data → UI), events flow up (UI → SSOT) |

## Layers

```
┌──────────────────────────────────────────────┐
│ UI layer                                     │  UI elements (Compose / Views)
│                                              │  State holders (ViewModel, plain classes)
├──────────────────────────────────────────────┤
│ Domain layer (optional)                      │  Use cases
├──────────────────────────────────────────────┤
│ Data layer                                   │  Repositories → data sources
└──────────────────────────────────────────────┘
          Dependencies: UI → domain → data
```

- **UI layer** - renders state and forwards events. See [ui-layer.md](ui-layer.md)
- **Domain layer** - optional; add it only when logic is complex or reused by several ViewModels
- **Data layer** - business logic and data; repositories are the **only** entry point to data

### The Dependency Rule

- The UI depends on the domain/data layers, never the reverse
- ViewModels and use cases never touch data sources (database, network, preferences, sensors) directly - always through a repository
- Depend on abstractions (interfaces) where you need to swap implementations or fakes in tests

```kotlin
// The contract the rest of the app depends on
interface NewsRepository {
    fun getNewsStream(): Flow<List<Article>>
    suspend fun refreshNews()
}

// One implementation, named by what makes it different
class OfflineFirstNewsRepository(
    private val newsLocalDataSource: NewsLocalDataSource,
    private val newsRemoteDataSource: NewsRemoteDataSource,
) : NewsRepository {
    override fun getNewsStream(): Flow<List<Article>> =
        newsLocalDataSource.getNewsStream().map { entities -> entities.map(ArticleEntity::asExternalModel) }

    override suspend fun refreshNews() {
        val networkArticles = newsRemoteDataSource.fetchLatestNews()
        newsLocalDataSource.saveNews(networkArticles.map(NetworkArticle::asEntity))
    }
}
```

## Domain Layer

- **One use case = one action**, named **verb in present tense + noun + `UseCase`**: `FormatDateUseCase`, `GetLatestNewsWithAuthorsUseCase`, `LogOutUserUseCase`
- Callable via `operator fun invoke`
- **No mutable state**; no lifecycle of its own (new instance per injection)
- Depends on repositories and other use cases
- **Main-safe**: move blocking work with `withContext(injectedDispatcher)`
- Prefer use cases over `Util` classes for reusable business logic
- Add a use case only when it adds value - don't wrap every repository call (YAGNI)

```kotlin
class GetLatestNewsWithAuthorsUseCase(
    private val newsRepository: NewsRepository,
    private val authorsRepository: AuthorsRepository,
    private val defaultDispatcher: CoroutineDispatcher = Dispatchers.Default,
) {
    suspend operator fun invoke(): List<ArticleWithAuthor> = withContext(defaultDispatcher) {
        val articles = newsRepository.fetchLatestNews()
        articles.map { article -> ArticleWithAuthor(article, authorsRepository.getAuthor(article.authorId)) }
    }
}
```

## Data Layer

**Repositories:**
- Expose data, centralize changes, resolve conflicts between sources, abstract sources, contain business logic
- Expose **immutable** data: `suspend` functions for one-shot operations, `Flow` for data that changes over time
- Create a repository even with a single data source
- One SSOT per repository (usually the local database in offline-first apps)
- Repositories may depend on other repositories

**Data sources:**
- Each handles exactly **one** source (network API, database, file, preferences, sensor)
- Injected into repositories via the constructor

**Threading:** repositories and data sources are **main-safe**, using an injected dispatcher when the underlying library isn't.

**Scoping:**
- App-wide data → a single instance for the app (singleton scope in the DI container)
- Flow-specific data (e.g. registration) → scoped to that flow

**Errors:** use Kotlin's mechanisms - `try/catch` around `suspend` calls, the `catch` operator on flows, or a `Result` return type. Custom exceptions (`UserNotAuthenticatedException`) are fine.

**Caching:** in-memory cache guarded by a `Mutex` when concurrent access is possible.

### Operation Types

| Operation | Lifetime | How |
|-----------|----------|-----|
| UI-oriented | While the screen is visible | Caller's scope (`viewModelScope`) |
| App-oriented | Must finish even if the user leaves the screen | Injected external `CoroutineScope` |
| Business-oriented | Must survive process death | Persistent work (e.g. WorkManager) |

### Models per Layer

Separate models when a source's shape differs from what the app needs:

```
NetworkArticle (network)  ──asEntity()──►  ArticleEntity (database)  ──asExternalModel()──►  Article (app)
```

- Network and database models stay internal to the data layer
- Map with small extension functions next to the models
- `UiState` may contain app models or UI-specific models

## Offline-First

- At minimum, the app must be able to **read** offline
- Every network-backed repository has a **local and a network data source**; the **local source is the SSOT** and the only one read by higher layers
- **Reads:** expose `Flow`s from the local source; refresh by writing network data into it
- **Writes:**
  - *Online-only* - critical transactions (e.g. payments)
  - *Queued* - non-critical data synced later (e.g. analytics)
  - *Lazy* - write locally first, sync to the network later (best user experience)
- **Sync:** pull (on demand), push (server notifies), or hybrid
- **Conflicts:** versioning/timestamps; "last write wins" is the common default
- Retry network failures with exponential backoff (but not for errors like 401)

## Storage Choices

| Data shape | Typical option |
|------------|----------------|
| Large, structured, queryable, relational | A database (e.g. Room - official recommendation) |
| Small key-value or typed settings | DataStore (official recommendation) or SharedPreferences where already used |
| Blobs (JSON, images) | Files |

**Follow the project's existing choice.** If nothing is established, ask the user. If DataStore is used:
- Never create more than one instance per file in the same process
- Its type must be immutable
- Don't mix single-process and multi-process instances for the same file
- Access it only through a data source/repository

Whatever storage is used: access it only inside the data layer and never do disk I/O on the main thread.

## Persistent Work

Work that must survive leaving the app, process death, or reboot (sync, uploads, periodic refresh) uses a persistent work scheduler (e.g. WorkManager - official recommendation):

- Immediate (expedited), long-running (foreground), or deferrable (periodic)
- Constraints (network, charging), unique work, exponential backoff
- The worker calls a repository; it doesn't contain business logic

In-process work that can die with the app uses coroutines instead.

## Dependency Injection

- **Constructor injection** for every class with dependencies
- Depend on interfaces where implementations vary or tests need fakes
- Scope instances that hold shared mutable data or are expensive to create
- **Use the project's DI approach.** Hilt is the official recommendation; Koin and manual DI (an `AppContainer`) are equally valid. For a new project, ask the user.

```kotlin
// Works with any DI approach - the class doesn't know how it's built
class ProfileViewModel(
    private val getUserProfile: GetUserProfileUseCase,
    private val savedStateHandle: SavedStateHandle,
) : ViewModel()
```

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Repository | type of data + `Repository` | `NewsRepository` |
| Data source | type of data + type of source + `DataSource` | `NewsRemoteDataSource`, `NewsLocalDataSource` |
| Use case | verb + noun + `UseCase` | `GetLatestNewsUseCase` |
| UI state | functionality + `UiState` | `NewsUiState` |
| Stream function | `get<Model>Stream()` | `getAuthorsStream(): Flow<List<Author>>` |
| Implementation | what makes it different | `OfflineFirstNewsRepository`, `InMemoryNewsRepository` |
| Default implementation | `Default` prefix | `DefaultNewsRepository` |
| Test double | `Fake` prefix | `FakeNewsRepository` |
| Methods | verb phrases | `makePayment()` |
| Properties | noun phrases | `inProgressTopicSelection` |

Never name data sources after the implementation (`UserSharedPreferencesDataSource`) - the name should survive a storage change.

## Project Structure

Organize by **feature**, then by layer. Small apps can use packages; larger apps use Gradle modules:

```
app/                          # Application, single Activity, navigation graph
feature/
  news/                       # :feature:news
    NewsRoute.kt
    NewsScreen.kt
    NewsViewModel.kt
    NewsUiState.kt
  profile/                    # :feature:profile
core/
  domain/                     # :core:domain   use cases
  data/                       # :core:data
    model/                    # app models
    repository/
    local/                    # entities, DAOs, local data sources
    network/                  # network models, API, remote data sources
  ui/                         # :core:ui       shared composables
  designsystem/               # :core:designsystem  theme, components
  testing/                    # :core:testing  fakes, test rules
```

- Each module exposes as little as possible (`internal` by default)
- Features don't depend on other features
- Avoid ever-growing `common`/`util` modules

## Navigation

- Prefer a single Activity
- Navigation is UI logic: the UI navigates in response to user events or UI state, not the ViewModel calling a navigator directly
- Implementation and migrations belong to the `navigation-3` and `navigation-event` skills

## Cross-Cutting Concerns

| Concern | Android approach |
|---------|------------------|
| Network auth, logging, retries | HTTP client interceptors (e.g. OkHttp interceptors) |
| Object creation and lifetimes | DI container / scopes |
| Reacting to lifecycle | `DefaultLifecycleObserver` / lifecycle-aware components |
| App startup initialization | App Startup initializers or DI, not heavy `Application.onCreate` |
| Error reporting | A data-layer abstraction (e.g. `CrashReporter` interface) |

## Conway's Law

> "Organizations design systems that mirror their communication structure."

**Implication:** Team structure affects module structure. Align both intentionally.

## The Walking Skeleton

Start with a minimal end-to-end slice:

1. **Thinnest possible feature** that touches all layers (screen → ViewModel → repository → data source)
2. **Runnable** from day one
3. **Proves the architecture** works

Example: one screen showing ONE hardcoded item from a fake repository, then replace the fake with a real data source.

## Testing Architecture

```
┌────────────────────────────────────────────┐
│  UI / E2E tests (instrumented)             │  Few, slow, high fidelity
├────────────────────────────────────────────┤
│  Integration tests                         │  Some (DB, network, navigation)
├────────────────────────────────────────────┤
│  Unit tests (local JVM)                    │  Many, fast, isolated
└────────────────────────────────────────────┘
```

**Test by layer:**
- **ViewModels, use cases, repositories, data sources:** local unit tests with fakes
- **Database/network integration:** real implementations in tests (e.g. in-memory database, mock web server)
- **UI:** screen tests and navigation tests for critical journeys

See [testing.md](testing.md).

## Architecture Decision Records (ADRs)

Document significant decisions - especially library choices made with the user:

```markdown
# ADR 001: Key-value storage for user settings

## Status
Accepted

## Context
The app needs to persist a few user settings. No storage library is established yet.
Options presented to the user: DataStore (official recommendation), SharedPreferences.

## Decision
DataStore (Preferences), chosen by the team for:
- Asynchronous, Flow-based API (no main-thread I/O)
- Transactional updates

## Consequences
- Settings are read through `UserSettingsRepository` as a Flow
- One DataStore instance per file, provided as a singleton
```

## Red Flags in Architecture

- **Logic or data in Activities/Fragments**
- **ViewModel or use case accessing a data source directly**
- **Data layer depending on the UI layer** (or on `Context` where an abstraction would do)
- **Mutable data exposed from repositories**
- **Network as source of truth** in an app that should work offline
- **Circular dependencies** between modules; features depending on features
- **Shared mutable state** across modules
- **"Util" or "Common" modules** that grow forever
- **Database or API schema leaking** into the UI
- **Library chosen or replaced without asking the user**
