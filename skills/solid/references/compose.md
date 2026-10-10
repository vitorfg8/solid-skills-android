# Jetpack Compose

> Based on the official guides: [Compose API guidelines](https://developer.android.com/develop/ui/compose/api-guidelines), [Kotlin for Compose](https://developer.android.com/develop/ui/compose/kotlin), [Compose architecture](https://developer.android.com/develop/ui/compose/architecture).

This file covers Compose **API design and readability**. Platform workflows live in the official Android skills: adaptive layouts in `adaptive`, Styles API theming in `styles`, XML migrations in `migrate-xml-views-to-jetpack-compose`, insets in `edge-to-edge`.

Compose is the default UI toolkit. XML Views are acceptable in legacy code - the UDF, state hoisting, and naming principles below still apply.

## Readable Composables

A screen composable should read like the screen it draws. Extract well-named pieces so the structure is visible at a glance:

```kotlin
@Composable
fun ProfileScreen(
    uiState: ProfileUiState,
    onEditClick: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(modifier = modifier.fillMaxSize()) {
        ProfileHeader(user = uiState.user, onEditClick = onEditClick)
        ProfileStatistics(statistics = uiState.statistics)
        RecentActivityList(activities = uiState.recentActivities)
    }
}
```

## Naming

| Kind | Convention | Example |
|------|------------|---------|
| Composable returning `Unit` (emits UI) | PascalCase **noun** (exception to "functions start with a verb") | `FancyButton`, `ProfileHeader` |
| Composable returning a value | camelCase, like any function | `defaultTextStyle()` |
| Remembered factory | `remember` prefix | `rememberSearchBarState()` |
| CompositionLocal | `Local` prefix | `LocalAppSpacing` |
| Hoisted state class | `<Component>State` | `SearchBarState` |
| Defaults holder | `<Component>Defaults` | `SearchBarDefaults` |

**A composable either emits UI or returns a value, never both.**

## Parameters

Order:
1. Required parameters (data first)
2. `modifier: Modifier = Modifier`
3. Optional parameters (with meaningful defaults)
4. Trailing `@Composable` `content` slot (if any)

```kotlin
@Composable
fun ArticleCard(
    article: ArticleUiState,
    onBookmarkClick: () -> Unit,
    modifier: Modifier = Modifier,
    isBookmarked: Boolean = false,
    actions: @Composable RowScope.() -> Unit = {},
) {
    Card(modifier = modifier) { /* ... */ }
}
```

**`modifier` rules:**
- Every UI-emitting composable has exactly one `modifier` parameter with default `Modifier`
- Apply it **once**, **first**, to the **root** element
- Never `Modifier.padding(8.dp).then(modifier)`, never a non-empty default, never extra `rowModifier`/`iconModifier` parameters (use slots)

**Many defaulted, named parameters are fine for composables** - this is the official style and an explicit exception to "minimize parameters". Call them with named arguments.

**Avoid:**
- `MutableState<T>` parameters (splits ownership) - pass the value and an `onValueChange` lambda
- `State<T>` parameters - pass `T`, or `() -> T` to defer a frequently changing read
- `null` meaning "use the default" - nullable means "absent"
- Parameters for behavior a modifier already provides (`onClick`, `clipToCircle`)

## State Hoisting and Slots

- Prefer **stateless** composables: `value` + `onValueChange`, events as `on<Event>` lambdas
- Hoist state to the lowest common owner that reads and writes it
- Screen state comes from the ViewModel; only the route composable knows the ViewModel (see [ui-layer.md](ui-layer.md))
- UI element state that needs logic goes into a plain `@Stable` state class with a `rememberX()` factory
- Use `rememberSaveable` for small UI state that must survive recreation
- Prefer **slots** (`@Composable` lambdas) over `title: String?`/`icon: ImageVector?` parameters for flexible content
- Each component solves one problem; prefer separate components (`PrimaryButton`) over configuration parameter bags. Theming with the Compose Styles API belongs to the `styles` skill.

**CompositionLocal:** only for app/screen-wide values like theming. Never to pass data or dependencies implicitly. Read it in default parameter expressions, not deep inside the body.

**Previews:** the initial state must render on the first frame; stateless composables preview and screenshot-test easily.

## Write State Only in Events and Effects

```kotlin
// BAD: writes state in the composition body
count++

// GOOD: write in the event that caused the change
Button(onClick = { count++ }) { Text("Increment") }
```

- Write state in event callbacks (`onClick`), effects (`LaunchedEffect`, `SideEffect`), or the ViewModel
- Keep composable bodies light; move calculations to the ViewModel or `remember`
- Measure on release builds before optimizing. For profiling, jank, and memory, use the `android-profiler` skill; for keep rules and app size, use `r8-analyzer`

## Kotlin Idioms in Compose

- Default arguments + named arguments instead of overloads
- Trailing lambdas for `content` slots; scoped receivers (`RowScope.() -> Unit`)
- `var isDialogVisible by remember { mutableStateOf(false) }` (delegated properties)
- `rememberCoroutineScope()` to launch from event handlers; `LaunchedEffect(key)` for work tied to composition

## Legacy XML Views

- Keep Views dumb: render `UiState`, forward events
- Collect with `repeatOnLifecycle` (see [ui-layer.md](ui-layer.md))
- Use ViewBinding (if the project uses it) and clear it in `onDestroyView`
- Migrating a screen to Compose is the `migrate-xml-views-to-jetpack-compose` skill, and only when the user asks
