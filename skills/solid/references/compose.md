# Jetpack Compose

> Based on the official guides: [Style guidelines for Compose APIs](https://developer.android.com/develop/ui/compose/api-guidelines), [API Guidelines for Jetpack Compose](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-api-guidelines.md), [API Guidelines for @Composable components](https://android.googlesource.com/platform/frameworks/support/+/androidx-main/compose/docs/compose-component-api-guidelines.md), [Kotlin for Compose](https://developer.android.com/develop/ui/compose/kotlin), [Compose architecture](https://developer.android.com/develop/ui/compose/architecture).

This file covers Compose **API design and readability** for app code. The official guidelines are strictest for the Compose framework and libraries; the rules below are the ones that make app code easier to read, change, and test. Shared UI modules (e.g. a design system) should follow them more strictly.

Platform workflows (adaptive layouts, Styles API theming, XML migrations, insets, profiling) are out of scope: use a matching Android skill if one is installed (`android skills find compose`).

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

Layout nesting (`Column { Row { ... } }`) describes the UI tree and is not the control-flow nesting that the "one level" rule limits. Extract a composable when a block deserves its own name.

## Before Creating a Component

**One component solves one problem.** If a component does two jobs, split it:

```kotlin
// BAD: a button that is also a toggle
@Composable
fun ActionButton(
    onClick: () -> Unit = {},
    isChecked: Boolean = false,
    onCheckedChange: (Boolean) -> Unit = {},
)

// GOOD: one problem each
@Composable
fun ActionButton(onClick: () -> Unit, modifier: Modifier = Modifier)

@Composable
fun ActionToggleButton(isChecked: Boolean, onCheckedChange: (Boolean) -> Unit, modifier: Modifier = Modifier)
```

**Do you need a component at all?** Write the UI with existing building blocks first. If the result is short, readable, and needs no hidden knowledge, keep it inline (YAGNI). Extract a component when it is reused or when its name makes the screen easier to read.

**Layer components:** small single-purpose building blocks first, then opinionated combinations (`CheckboxRow` = `Row` + `Checkbox` + `Text`). Higher-level components have more opinionated defaults and fewer options.

**Component or `Modifier`?**
- Behavior that applies to **any single** element (padding, click, semantics, drawing) → `Modifier` extension
- Behavior that **adds or removes** elements in the hierarchy (`AnimatedVisibility`) → component

```kotlin
// BAD: is the padding around both children or each one?
@Composable
fun Padding(all: Dp, content: @Composable () -> Unit)

// GOOD
UserCard(modifier = Modifier.padding(12.dp))
```

## Naming

| Kind | Convention | Example |
|------|------------|---------|
| Composable returning `Unit` (visual or not) | PascalCase **noun**, optionally with adjectives - never a verb, adverb, or adjective alone | `FancyButton`, `ProfileHeader`, `BackPressHandler` |
| Composable returning a value | camelCase, like any function - not the PascalCase name of the returned type | `defaultTextStyle()`, not `TextStyle()` |
| Composable that remembers and returns a mutable object | `remember` prefix | `rememberSearchBarState()` |
| CompositionLocal | `Local` as an adjective **prefix**, never a suffix | `LocalAppSpacing`, not `AppSpacingLocal` |
| Hoisted state class | `<Component>State` | `SearchBarState` |
| Defaults holder | `<Component>Defaults` | `SearchBarDefaults` |
| State-dependent colors | `<Component>Colors` | `SearchBarColors` |

**A composable either emits UI or returns a value, never both.** Control surfaces are passed in as parameters (a state object or lambdas), not returned.

In shared UI modules:
- Variants from the design spec use spec prefixes (`OutlinedButton`, `TextButton`); the most common variant has no prefix (`Button`)
- `Basic*` marks undecorated building blocks meant to be wrapped (`BasicTextField`)
- Avoid company or module prefixes (`AcmeButton`); prefer use-case names

**Constants:** keep `UPPER_SNAKE_CASE` (Android Kotlin style guide). The Compose guidelines allow app code to use PascalCase constants (`DefaultKeyName`); follow it only if the project already does. Members of `<Component>Defaults` objects use PascalCase, matching Material (`ButtonDefaults.MinHeight`).

## Parameters

Order:
1. Required parameters
2. `modifier: Modifier = Modifier`
3. Optional parameters
4. Trailing `@Composable` `content` slot (if any)

Inside the required and optional groups, put **data** ("what") before **metadata and customization** ("how"), and keep related parameters together (`backgroundColor` next to `contentColor`).

```kotlin
@Composable
fun ArticleCard(
    article: ArticleUiState,            // required data
    onBookmarkClick: () -> Unit,        // required event
    modifier: Modifier = Modifier,
    isBookmarked: Boolean = false,      // optional data
    colors: ArticleCardColors = ArticleCardDefaults.colors(),
    actions: @Composable RowScope.() -> Unit = {},
) {
    Card(modifier = modifier) { /* ... */ }
}
```

**`modifier` rules:**
- Every UI-emitting composable has exactly one parameter named `modifier` of type `Modifier`, the first optional parameter, with the no-op default `Modifier`
- Only components without an intrinsic size (e.g. a drawing canvas) may make it required
- Apply it **once**, **first**, to the **root** element; internal modifiers are chained **after** it (`modifier.padding(16.dp)`)
- Never `Modifier.padding(8.dp).then(modifier)`, never a non-empty default, never extra `rowModifier`/`iconModifier` parameters (use slots)

**Many defaulted, named parameters are fine for composables** - this is the official style and an explicit exception to "minimize parameters". Call them with named arguments.

**Don't add parameters for behavior a modifier already provides** (`onClick` on an image, `clipToCircle`). Parameters customize what the component does internally.

### Nullable parameters and defaults

- `null` means **absent** (`contentDescription: String?` - null adds no accessibility label). Never use `null` to mean "use the default"
- For "present but empty", use a meaningful empty default instead of `null`
- Defaults are **public and meaningful**: callers that wrap the component can reuse them. No private default constants
- Several defaults live in a public `<Component>Defaults` object; a few short ones can stay inline (`elevation: Dp = 0.dp`)

```kotlin
// BAD: null as "use default", private default
@Composable
fun IconCard(bitmap: ImageBitmap, elevation: Dp? = null) {
    val resolvedElevation = elevation ?: DefaultElevation
}
private val DefaultElevation = 8.dp

// GOOD
@Composable
fun IconCard(
    bitmap: ImageBitmap,
    modifier: Modifier = Modifier,
    elevation: Dp = IconCardDefaults.Elevation,
)

object IconCardDefaults {
    val Elevation = 8.dp
}
```

### Explicit dependencies

Prefer **explicit parameters** over implicit inputs. A reader of the call site must be able to tell where a value comes from.

- Don't read component-specific customization from a `CompositionLocal` (`LocalButtonBorder`)
- `CompositionLocal` only for app- or screen-wide values like theming - and read it in **default expressions**, not in the body, so callers can override it

```kotlin
// BAD: hidden input, impossible to override
@Composable
fun SaveButton(onClick: () -> Unit) {
    val backgroundColor = LocalAppTheme.current.primaryColor
}

// GOOD: visible and overridable
@Composable
fun SaveButton(
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    backgroundColor: Color = LocalAppTheme.current.primaryColor,
)
```

- **No "style" grab-bag classes** (`ButtonStyle(background, border, shape, padding)`): create semantically named components instead (`PrimaryButton`, or a `LoginButton` that wraps `SecondaryButton`)
- For values that depend on component state (enabled/disabled), use a simple `if` in the default expression, or a `<Component>Colors` class created by `<Component>Defaults.colors()` when the logic grows

## State Hoisting

- Prefer **stateless and controlled** composables: `value` + `onValueChange`, events as `on<Event>` lambdas. The caller owns the state and can apply its own policy (validation, limits)
- Compose consumes **state, not events**. State is conflated and observers must be idempotent - another reason ViewModel events become UI state (see [ui-layer.md](ui-layer.md))
- Hoist state to the lowest common owner that reads and writes it
- Screen state comes from the ViewModel; only the route composable knows the ViewModel
- Use `rememberSaveable` for small UI state that must survive recreation

```kotlin
// BAD: the component owns the state; the caller can't apply a policy
@Composable
fun TermsCheckbox(initialValue: Boolean, onCheckedChange: (Boolean) -> Unit)

// GOOD: controlled by the caller
@Composable
fun TermsCheckbox(isChecked: Boolean, onCheckedChange: (Boolean) -> Unit, modifier: Modifier = Modifier)
```

**Hoisted state classes:** when related state and callbacks grow unwieldy, group them in a `<Component>State` class with a `remember<Component>State()` factory.
- In app code, use a **concrete `@Stable` class** backed by `mutableStateOf`. Extract an interface only when a second implementation is needed (Rule of Three)
- The class keeps its properties consistent (e.g. clamps a position into its range)
- Provide it as a remembered default when the component works without caller configuration: `state: SearchBarState = rememberSearchBarState()`
- Never `state: SearchBarState? = null` with an internal `remember` fallback

**Avoid:**
- `MutableState<T>` parameters (split ownership) - pass the value and an `onValueChange` lambda, or a `<Component>State` class
- `State<T>` parameters - pass `T`, or `() -> T` to defer a frequently changing read

## Slots

- Prefer **slots** (`@Composable` lambdas) over `title: String?`/`icon: ImageVector?` parameters for flexible content
- Name a single slot `content` and put the main slot last for trailing-lambda syntax
- Give a slot the layout scope it is placed in (`content: @Composable RowScope.() -> Unit`) so readers know how multiple children are arranged
- Prefer plain slots over DSL-based slots; DSLs are for lazy content (`LazyColumn { items(...) }`)
- Slot content must not be disposed and recreated when the parent only changes its layout; use one layout or `movableContentOf` instead of calling `content()` in two branches

```kotlin
// BAD: switching between Row and Column recreates content() and loses its state
if (isExpanded) Row { content() } else Column { content() }

// GOOD
val movableContent = remember(content) { movableContentOf(content) }
if (isExpanded) Row { movableContent() } else Column { movableContent() }
```

## Write State Only in Events and Effects

```kotlin
// BAD: writes state in the composition body
count++

// GOOD: write in the event that caused the change
Button(onClick = { count++ }) { Text(stringResource(R.string.increment)) }
```

- Write state in event callbacks (`onClick`), effects (`LaunchedEffect`, `SideEffect`), or the ViewModel
- Keep composable bodies light; move calculations to the ViewModel or `remember`
- Measure on release builds before optimizing

## Accessibility

- Build on foundation blocks (`Modifier.clickable`, `Modifier.toggleable`, `Image`, `Icon`) - they provide semantics by default. Low-level `Layout` and `pointerInput` need semantics by hand
- Merge a group that reads as one item with `Modifier.semantics(mergeDescendants = true)`
- Require a `contentDescription: String?` on image-like leaf components; `null` only for decorative content
- Don't add a parameter for every semantics property - callers can pass `Modifier.semantics { }` through `modifier`
- For a custom component, copy the semantics of the closest standard component

## Previews and Tests

- The initial state renders on the **first frame**: no `LaunchedEffect` or async work to set up initial state
- Use `LocalInspectionMode.current` only for the minimal change a preview needs; never replace the component with a placeholder
- Stateless components preview and screenshot-test in every state by passing state as parameters

## Kotlin Idioms in Compose

- Default arguments + named arguments instead of overloads
- Trailing lambdas for `content` slots; scoped receivers (`RowScope.() -> Unit`)
- `var isDialogVisible by remember { mutableStateOf(false) }` (delegated properties)
- `rememberCoroutineScope()` to launch from event handlers; `LaunchedEffect(key)` for work tied to composition

## KDoc for Shared Components

For components in shared UI modules: a one-line summary, a paragraph on behavior, and `@param` for each parameter (the trailing `content` may be omitted). Skip KDoc for screen-private composables with self-explanatory names.

## Legacy XML Views

- Keep Views dumb: render `UiState`, forward events
- Collect with `repeatOnLifecycle` (see [ui-layer.md](ui-layer.md))
- Use ViewBinding (if the project uses it) and clear it in `onDestroyView`
- Migrate a screen to Compose only when the user asks
