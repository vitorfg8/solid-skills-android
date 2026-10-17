---
name: solid-android
description: Use when writing, reviewing, refactoring, testing, or designing Kotlin code in Android apps (ViewModels, repositories, use cases, Compose or Views UI, coroutines and Flow). Applies SOLID, TDD, readable Kotlin, and the official Android architecture.
---

# Solid Skills: Professional Android Engineering

This skill defines how Android Kotlin code should be written: readable for humans and agents alike, test-driven, SOLID, and aligned with the official Android guidance.

## When This Skill Applies

Use it for Kotlin code in Android app modules: writing features and fixes, refactoring, designing architecture, reviewing, debugging, and writing tests.

**Self-contained:** every rule here applies without any other tool or skill installed.

**With the Android CLI (`android`), when available:**
- `android docs search <keywords>` - confirm the current official guidance for an API before using it
- `android skills find <keyword>` - find a workflow skill (test setup, migrations, navigation, insets, profiling, R8, AGP upgrades, SDK integrations). If one is installed, follow its steps; the code it produces still follows this skill

**Project rules:** if a project rule (`AGENTS.md`, `.cursor/rules`, `CLAUDE.md`, an internal style guide) conflicts with this skill, ask the user which one to follow (AskQuestion when available), showing both options. Never resolve the conflict silently.

**Library policy - no library is mandatory; principles are:**
1. **Existing project:** follow what `build.gradle(.kts)`, `libs.versions.toml`, and existing code already use. **Never introduce or migrate to a competing library** (Koin → Hilt, SharedPreferences → DataStore, LiveData → StateFlow, MockK → fakes only) unless the user asks.
2. **No established choice:** ask the user (AskQuestion when available), marking the official recommendation. Never pick silently. This also applies when another skill would add or replace a library.
3. **Always:** constructor injection, repositories as the single entry point to data, main-safety, lifecycle-aware observation.

When this skill names a library, read it as "e.g.".

## Core Philosophy

> "Code is to create products for users & customers. Testable, flexible, and maintainable code that serves the needs of the users is GOOD because it can be cost-effectively maintained by developers."

**Code is read by humans and by agents with equal weight.** Both must understand what it does, and why, from the code alone - without running it, debugging it, or opening other files. When readability conflicts with brevity, cleverness, or a size rule, **readability wins**.

## The Non-Negotiable Process

### 1. Start with Tests for Behavior (TDD)

```
1. RED      - Write a failing test that describes the behavior
2. GREEN    - Write the simplest code that makes it pass
3. REFACTOR - Clean up, remove duplication (Rule of Three)
```

- **In scope:** new or changed behavior in use cases, repositories, data sources, ViewModels, and other JVM-testable logic. For UI, test screen state and behavior
- **Out of scope:** layout, styling, copy, or a one-line fix with no behavior change
- **Debugging:** write the failing test first when the bug is expressible as behavior; don't block a fix on an unreproducible test
- **Verify before finishing:** run the affected module's unit tests, Android lint, and any configured Kotlin linter (detekt, ktlint, spotless) - for the variant the change affects. Fix failures you introduced. See [references/tdd.md](references/tdd.md#verify-before-finishing) for the commands

Open [references/tdd.md](references/tdd.md) for the TDD loop, the three laws, and test naming.

### 2. Apply SOLID Principles

| Principle | Question to Ask |
|-----------|-----------------|
| **S**RP - Single Responsibility | "Does this have ONE reason to change?" |
| **O**CP - Open/Closed | "Can I extend without modifying?" |
| **L**SP - Liskov Substitution | "Can subtypes replace base types safely?" |
| **I**SP - Interface Segregation | "Are clients forced to depend on unused methods?" |
| **D**IP - Dependency Inversion | "Do high-level modules depend on abstractions?" |

Receive dependencies through the constructor. Add an interface where implementations vary or tests need a fake - not for every class.

Open [references/solid-principles.md](references/solid-principles.md) when a design decision depends on one of the principles.

### 3. Write Clean, Readable Kotlin

**Make intent explicit:**
- Names say exactly what a function does, including side effects (`saveDraftAndNotifyCollaborators()`, not `process()`)
- Names are specific and greppable: same concept = same name everywhere; avoid `data`, `info`, `manager`, `Util`, `Helper`
- Booleans read as questions (`isLoading`, `hasUnsavedChanges`, `canSubmit`)
- Explicit types on public APIs; domain types (`value class`, `sealed interface`) in signatures
- Named arguments for every `Boolean` argument and for parameters of the same type
- Named constants instead of magic numbers
- No hidden inputs: no global state, no `CompositionLocal` for data, no surprising `infix`, operators, or extensions
- One way to do each thing in a codebase

**Avoid clever Kotlin:**
- No scope-function chains (`let`/`run`/`also`/`apply`) when a plain `val` or `if` reads better
- No `it` in nested lambdas - name the parameter
- No `?.let { } ?: run { }` when an early return reads better
- No one-liners that hide branching or side effects

**Kotlin style (Android Kotlin style guide):**
- Classes PascalCase nouns; functions camelCase verbs; no `m`/`s` prefixes; acronyms as words (`XmlParser`, `userId`)
- Backing properties `_name`; constants `UPPER_SNAKE_CASE` as `const val`
- `@Composable` functions returning `Unit` are PascalCase nouns (`ProfileHeader`)

**Functions and control flow:**
- Small (~10-15 lines), doing one thing, all statements one level of abstraction below the name
- Default parameters + named arguments over overloads; group parameters into a `data class` only when the group is a real domain concept
- Guard clauses (`?: return`, early `return`) and shallow control flow: at most one level of `if`/`when`/`for`/lambda nesting per function (layout nesting in composables doesn't count)
- `if`/`when` as expressions; exhaustive `when` over `sealed` types
- Real errors (`require`, `check`, exceptions, `Result`) instead of error codes; never swallow `CancellationException`
- `val` and read-only types; never expose mutable state; no `!!`

**Value objects for domain concepts:**
```kotlin
@JvmInline value class UserId(val value: String)

@JvmInline value class Email(val value: String) {
    init { require(EMAIL_PATTERN.matches(value)) { "Invalid email: $value" } }
}

// BAD:  fun createOrder(userId: String, email: String)
// GOOD: fun createOrder(userId: UserId, email: Email)
```

**Self-check:** "Could a developer or agent new to this codebase explain this code after reading it once?" If not, rename, extract, or simplify. A comment is the last resort and only explains *why*.

Open [references/clean-code.md](references/clean-code.md) only to check a specific naming, formatting, or calisthenics rule.

### 4. Design with Responsibility in Mind

For every class ask: "What role is this?" and "Is it doing too much?"

- **Information Holder** - `data class`, `UiState`, domain models
- **Service Provider** - use cases, data sources
- **Coordinator** - repositories orchestrating data sources
- **Controller** - ViewModels turning events into state
- **Interfacer** - mappers, network/database adapters

Distinguish **objects** (hide data, expose behavior) from **data holders** (`data class`). Don't mix them.

Open [references/object-design.md](references/object-design.md) when assigning responsibilities to new classes.

### 5. Manage Complexity

- **Essential** complexity comes from the domain; **accidental** complexity comes from our solutions
- Detect it through change amplification, cognitive load, and unknown unknowns
- Fight it with YAGNI, KISS, and DRY only after the Rule of Three

Open [references/complexity.md](references/complexity.md) when a design feels heavier than the problem.

### 6. Architect for Change (Official Android Architecture)

- **Layers:** UI (Compose or Views + `ViewModel`) → domain (optional use cases) → data (repositories + data sources)
- Activities/Fragments only host UI; no app data or business logic in them
- **Single source of truth** - only the owner mutates data and exposes it immutably
- **Unidirectional data flow** - state flows down, events flow up
- ViewModels and use cases reach data only through repositories
- Constructor injection with the project's DI approach (Hilt is the official recommendation; Koin or manual DI are equally valid)
- Feature-first modules with a minimal public surface

Open [references/architecture.md](references/architecture.md) when adding a repository, use case, data source, or module.

### 7. UI Layer

- One immutable `UiState` per screen, exposed as `StateFlow` (or `LiveData` if the project uses it) through a private backing property
- ViewModels are screen-level and never hold `Context`, `Resources`, `View`, `Activity`, or `Fragment`
- ViewModel events become UI state - no one-off event channels for things the UI must not miss
- Collect state lifecycle-aware (`collectAsStateWithLifecycle()`; `repeatOnLifecycle` in Views); pass state and lambdas down, never the ViewModel
- State survives configuration changes and process death (`ViewModel`, `SavedStateHandle`, `rememberSaveable`)

Open [references/ui-layer.md](references/ui-layer.md) when creating or changing a ViewModel, `UiState`, or screen.

### 8. Coroutines & Flow

- Inject dispatchers through the constructor (a `Dispatchers.IO` default value is fine); never hardcode `withContext(Dispatchers.IO)` inside a function body
- `suspend` functions are main-safe; data/domain layers expose `suspend` for one-shot calls and `Flow` for streams
- The ViewModel launches coroutines (`viewModelScope`); no `GlobalScope` - inject a scope for work that must outlive the screen
- Catch specific exceptions; always rethrow `CancellationException`

Open [references/coroutines-flow.md](references/coroutines-flow.md) when writing coroutines, flows, or their tests.

### 9. Compose (default UI toolkit)

- Exactly one `modifier: Modifier = Modifier`, the first optional parameter, applied once and first to the root
- Parameter order: required → `modifier` → optional → trailing `content` slot
- Stateless composables with hoisted state and `on<Event>` lambdas; never write state during composition
- One component solves one problem; behavior for any element is a `Modifier`; explicit parameters over `CompositionLocal`

XML Views are fine in legacy code with the same UDF principles; migrate a screen to Compose only when the user asks.

Open [references/compose.md](references/compose.md) when writing or reviewing composables.

### 10. UI Correctness

- No hardcoded or concatenated user-facing strings; use resources with format arguments
- Format numbers, dates, and currency with the current locale
- Interactive elements have 48dp touch targets and a label from resources; decorative images use `contentDescription = null`
- Text uses `sp` and survives large font and display sizes

## The Four Elements of Simple Design (XP)

In priority order:
1. **Runs all the tests** - Must work correctly
2. **Expresses intent** - Readable, reveals purpose
3. **No duplication** - DRY (but Rule of Three)
4. **Minimal** - Fewest classes, methods possible

## Code Smell Detection

**Refactor smells only in code you touch for the task; report other smells as suggestions instead of changing them.**

| Smell | Solution |
|-------|----------|
| Long Function | Extract functions |
| Large Class / God Activity | Move logic to ViewModel, use case, or repository |
| Long Parameter List | Default + named arguments, or a domain `data class` |
| Divergent Change | Split into focused classes |
| Shotgun Surgery | Move related code together |
| Feature Envy | Move function to the envied class |
| Primitive Obsession | Wrap in `value class` |
| Repeated `when` on type | Replace with polymorphism |
| Speculative Generality | YAGNI - remove unused abstractions |
| Clever Code / Hidden Intent | Rename, unroll, make explicit |

Open [references/code-smells.md](references/code-smells.md) for Android, Compose, and resource smells.

## Design Patterns Awareness

**Creational:** Singleton (`object`, DI scope), Factory, Builder (named/default args), Prototype (`copy()`)
**Structural:** Adapter, Decorator, Composite, Proxy (`by lazy`)
**Behavioral:** Strategy, Observer (`Flow`), Template Method, Command (`sealed` actions)

Don't force patterns; let them emerge from refactoring. Open [references/design-patterns.md](references/design-patterns.md) for the Kotlin idiom of each.

## Testing Strategy

- **Unit tests** (many) - local JVM (`src/test`): ViewModels, use cases, repositories, data sources
- **Integration tests** (some) - real implementations at a boundary (in-memory database, mock web server)
- **UI behavior tests** - Robolectric in `src/test` by default; instrumented `src/androidTest` when requested
- **E2E tests** (few) - critical user journeys

Rules:
- Prefer **fakes** over mocks; keep the project's mocking library if it already uses one
- Don't unit test Activities/Fragments
- Coroutines: `runTest`, injected `TestDispatcher`s, `MainDispatcherRule`; assert on `uiState.value`
- Arrange-Act-Assert; names `subjectUnderTest_trigger_expectedResult`, no backtick names

Open [references/testing.md](references/testing.md) when choosing test types, test doubles, or testing a specific layer.

## Behavioral Principles

- **Tell, Don't Ask** - Command objects, don't query and decide
- **Design by Contract** - Preconditions (`require`), invariants (`check`), postconditions
- **Hollywood Principle** - "Don't call us, we'll call you" (IoC, DI, observable state)
- **Law of Demeter** - Only talk to immediate friends

## Pre-Code Checklist

1. [ ] Do I understand the requirement? (Write acceptance criteria first)
2. [ ] Which libraries and project rules apply? Do I need to ask the user about a choice or a conflict?
3. [ ] What test will I write first?
4. [ ] What is the simplest solution? Am I solving a real problem?

## During-Code Checklist

1. [ ] Does this class have a single responsibility?
2. [ ] Does every name reveal every effect?
3. [ ] Am I staying inside the scope of the task?

## Post-Code Checklist

1. [ ] Do the unit tests, lint, and configured linters pass for the affected variant?
2. [ ] Is there dead code to remove or a condition to simplify?
3. [ ] Would a developer or agent new to the codebase understand this after one read?
4. [ ] Are strings in resources and interactive elements accessible?

## Red Flags - Stop and Rethink

- Changing behavior without a test that describes it
- Code you need to explain verbally to be understood
- Clever one-liners, nested scope functions, nested `it`
- Function name that hides a side effect; magic numbers; `Boolean` positional arguments
- Function longer than ~10-15 lines or control flow nested deeper than one level
- Abstractions before the third duplication; features "just in case"
- Instantiating dependencies inside a class instead of constructor injection
- Logic in Activities or Fragments; ViewModel holding `Context` or using `AndroidViewModel`
- Publicly exposed `MutableStateFlow`/`MutableLiveData`; lifecycle-unaware collection
- One-off event `Channel`s for events the UI must not miss
- `withContext(Dispatchers.IO)` hardcoded in a function body, `GlobalScope`, swallowed `CancellationException`
- `!!` instead of handling nullability; state written during composition
- Hardcoded or concatenated user-facing strings; interactive element without a 48dp target or label
- Refactoring code outside the task
- Silently resolving a conflict with a project rule; silently adding, replacing, or migrating a library

## Remember

> "A little bit of duplication is 10x better than the wrong abstraction."

> "Code is read far more often than it is written. Optimize for the reader - human or agent."
