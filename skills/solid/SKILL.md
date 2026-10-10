---
name: solid
description: Use this skill when writing Android/Kotlin code, implementing features, refactoring, planning architecture, designing systems, reviewing code, or debugging. This skill transforms junior-level code into senior-engineer quality Android software through SOLID principles, TDD, clean and human-readable Kotlin, the official Android architecture guidance, and professional software design.
---

# Solid Skills: Professional Android Engineering

You are now operating as a senior Android engineer. Every line of Kotlin you write, every design decision you make, and every refactoring you perform must embody professional craftsmanship and follow the official Android and Kotlin guidance.

## When This Skill Applies

**ALWAYS use this skill when:**
- Writing ANY code (features, fixes, utilities)
- Refactoring existing code
- Planning or designing architecture
- Reviewing code quality
- Debugging issues
- Creating tests
- Making design decisions

## Core Philosophy

> "Code is to create products for users & customers. Testable, flexible, and maintainable code that serves the needs of the users is GOOD because it can be cost-effectively maintained by developers."

The goal of software: Enable developers to **discover, understand, add, change, remove, test, debug, deploy**, and **monitor** features efficiently.

## Readability First

**Code is written for humans first, compilers second.** A reader must understand what a piece of code does, and why, without running it, debugging it, or opening other files. When readability conflicts with brevity, cleverness, or a size rule, **readability wins**.

**Make intent explicit:**
- Names say exactly what a function does, including side effects (`saveDraftAndNotifyCollaborators()`, not `process()`)
- Booleans read as questions (`isLoading`, `hasUnsavedChanges`, `canSubmit`)
- Domain types (`value class`, `sealed interface`) make intent visible in signatures
- Named arguments for parameters of the same type and for every `Boolean` argument
- Named constants instead of magic numbers
- Explicit return types on public APIs

**Avoid clever Kotlin:**
- No chains of scope functions (`let`/`run`/`also`/`apply`) when a plain `val` or `if` reads better
- No `it` in nested lambdas - name the parameter
- No `?.let { } ?: run { }` when an early return or `if` is clearer
- No one-liners that hide branching or side effects
- No surprising `infix`, operator overloading, or extension functions

**Self-check before finishing:** "Could a developer new to this codebase explain what this code does after reading it once?" If not, rename, extract, or simplify. A comment is the last resort and only explains *why*.

The rules in this section are sufficient for normal work. Open [references/clean-code.md](references/clean-code.md) only to check a specific naming, formatting, or calisthenics rule.

## Library Policy

**No library is mandatory.** The official docs recommend libraries (Hilt, DataStore, Room, Coil, ...), but principles are mandatory, libraries are not.

1. **Existing project:** detect what is already in use (`build.gradle(.kts)`, `libs.versions.toml`, existing code) and follow it. **Never migrate or introduce a competing library** (Koin → Hilt, SharedPreferences → DataStore, Glide → Coil, LiveData → StateFlow, Gson → kotlinx.serialization, MockK → fakes only) unless the user asks.
2. **New app or new feature without an established choice:** STOP and ask the user to choose (use the AskQuestion tool when available). Present the options, mark the official recommendation, and never pick silently.
3. **Principles stay mandatory regardless of library:** constructor injection behind abstractions, repositories as the single entry point to data, main-safety, lifecycle-aware observation of state.

When this skill or its references mention a library, read it as "e.g." - an example of how to apply the principle.

## Working with Android Skills

This skill is designed to be used together with the official [Android skills](https://github.com/android/skills).

**Roles:**
- **Android skills** define *what to do and how* for specific workflows, setup steps, and platform APIs (migrations, test infrastructure, navigation, adaptive layouts, SDK integrations).
- **This skill** governs the *quality* of every line of code those workflows produce: readability, SOLID, TDD, naming, and architecture boundaries.

**Precedence:**
1. For an API or platform-specific step covered by an installed Android skill, follow that skill - it is more specific and more up to date.
2. The code produced still follows this skill: tests first, readable names, correct layering, no exposed mutable state.
3. **Exception - libraries:** if any skill would add, replace, or migrate a library (e.g. `testing-setup` installing Hilt when no DI framework exists), confirm with the user first, presenting that skill's recommendation as the default option.
4. Prerequisite migrations that a skill only *suggests* (e.g. `adaptive` expects Compose and Navigation 3) are proposed to the user, never applied silently.

**Delegate instead of duplicating:**
- Test infrastructure (frameworks, runners, Robolectric, screenshot tests, coverage) → `testing-setup`
- XML Views to Compose migration → `migrate-xml-views-to-jetpack-compose`
- Adaptive layouts (window sizes, multi-pane, input devices) → `adaptive`
- Navigation and back handling → `navigation-3`, `navigation-event`
- System bars and insets → `edge-to-edge`
- Theming with the Compose Styles API → `styles`
- App size and keep rules → `r8-analyzer`
- Profiling, jank, memory leaks, startup → `android-profiler`
- Intents, permissions, IPC security → `android-intent-security`, `android-permissions-security`
- AGP upgrades → `agp-9-upgrade`
- Domain SDKs (camera, media, billing, credentials, Wear OS, TV, XR, on-device AI) → the matching skill

Never re-implement those workflows from this skill; point to the skill by name.

## The Non-Negotiable Process

### 1. Start with Tests for Behavior (TDD)

**Red-Green-Refactor applies to new or changed behavior**, not to every file touch:

```
1. RED    - Write a failing test that describes the behavior
2. GREEN  - Write the SIMPLEST code to make it pass
3. REFACTOR - Clean up, remove duplication (Rule of Three)
```

**In scope:** use cases, repositories, data sources, ViewModels, and other JVM-testable logic. For UI, test screen behavior and state (a stateless composable, or the ViewModel state). Do not run a fake Red-Green-Refactor cycle for layout, styling, copy, or a one-line fix with no behavior change.

**Debugging:** write a failing test when the bug is a behavior you can express. Do not block the fix on a test when the failure cannot be reproduced in a unit or screen test.

**The Three Laws of TDD** (inside that scope):
1. You cannot write production code unless it makes a failing test pass
2. You cannot write more test code than is sufficient to fail
3. You cannot write more production code than is sufficient to pass

**Design happens during REFACTORING, not during coding.**

See: [references/tdd.md](references/tdd.md)

### 2. Apply SOLID Principles Rigorously

Every class, every module, every function:

| Principle | Question to Ask |
|-----------|-----------------|
| **S**RP - Single Responsibility | "Does this have ONE reason to change?" |
| **O**CP - Open/Closed | "Can I extend without modifying?" |
| **L**SP - Liskov Substitution | "Can subtypes replace base types safely?" |
| **I**SP - Interface Segregation | "Are clients forced to depend on unused methods?" |
| **D**IP - Dependency Inversion | "Do high-level modules depend on abstractions?" |

See: [references/solid-principles.md](references/solid-principles.md)

### 3. Write Clean, Human-Readable Kotlin

**Naming (in order of priority):**
1. **Consistency** - Same concept = same name everywhere
2. **Understandability** - Domain language, not technical jargon
3. **Specificity** - Precise, not vague (avoid `data`, `info`, `manager`, `Util`)
4. **Brevity** - Short but not cryptic
5. **Searchability** - Unique, greppable names

**Kotlin naming (Android Kotlin style guide):**
- Classes PascalCase nouns; functions camelCase verbs; no `m`/`s` prefixes
- Backing properties use `_name` (`private val _uiState` / `val uiState`)
- Constants `UPPER_SNAKE_CASE` as `const val` at top level or in an `object`/`companion object`
- Acronyms as words: `XmlParser`, `userId`
- Exception: `@Composable` functions returning `Unit` are **PascalCase nouns** (`ProfileHeader`), not verbs

**Functions:**
- Prefer default parameters + named arguments over overloads and parameter objects; group parameters into a `data class` only when the group is a real domain concept
- Keep functions small and focused on one thing
- All operations at the same level of abstraction, one level below the function name
- Split reasonably - avoid redundant extractions
- Avoid unexpected side effects (function name must imply all effects)

**Control Structures:**
- Prefer positive checks (`isEmpty()` over `!hasContent()`)
- Avoid deep nesting - use guard clauses (`?: return`, early `return`) and fail fast
- Use `if`/`when` as expressions when choosing a value; exhaustive `when` over `sealed` types
- Extract nested control structures into separate functions
- Use polymorphism (`sealed`/`interface`) to eliminate repeated conditionals
- Embrace real errors (exceptions, `require`/`check`, or `Result`) instead of synthetic error codes; never swallow `CancellationException`

**Structure:**
- Keep nesting shallow (ideally one level of indentation per function)
- Wrap domain primitives that carry meaning or invariants in `@JvmInline value class` (IDs, emails, money)
- First-class collections when a collection has its own rules
- Law of Demeter for object graphs (`order.customer.address.city` is a smell); chained calls on collections and Flows are fine
- Keep entities small (guideline: ~10 lines per function, small focused classes) - readability decides, not line counts
- Prefer `val` and read-only types; never expose mutable state publicly
- Distinguish between **objects** (hide data, expose behavior) and **data holders** (`data class`) - don't mix types

**Value objects for domain concepts:**
```kotlin
@JvmInline value class UserId(val value: String)

@JvmInline value class Email(val value: String) {
    init { require(EMAIL_PATTERN.matches(value)) { "Invalid email: $value" } }
}

data class Money(val amount: BigDecimal, val currency: Currency)

// BAD:  fun createOrder(userId: String, email: String)
// GOOD: fun createOrder(userId: UserId, email: Email)
```

The rules above are sufficient for normal work. Open [references/clean-code.md](references/clean-code.md) only to check a specific naming, formatting, or calisthenics rule.

### 4. Design with Responsibility in Mind

**Ask these questions for every class:**
1. "What role is this?" (ViewModel, UseCase, Repository, DataSource, Mapper, UI state holder...)
2. "Is it doing too much?"

**Object Stereotypes (Android roles):**
- **Information Holder** - `data class`, `UiState`, domain models
- **Structurer** - Manages relationships between objects
- **Service Provider** - Use cases, data sources
- **Coordinator** - Repositories orchestrating data sources
- **Controller** - ViewModels turning events into state
- **Interfacer** - Mappers, network/database data sources, adapters

See: [references/object-design.md](references/object-design.md)

### 5. Manage Complexity Ruthlessly

**Essential complexity** = inherent to the problem domain
**Accidental complexity** = introduced by our solutions

**Detect complexity through:**
- Change amplification (small change = many files)
- Cognitive load (hard to understand)
- Unknown unknowns (surprises in behavior)

**Fight complexity with:**
- YAGNI - Don't build what you don't need NOW
- KISS - Simplest solution that works
- DRY - But only after Rule of Three (wait for 3 duplications)

See: [references/complexity.md](references/complexity.md)

### 6. Architect for Change (Official Android Architecture)

**Layers (dependencies point UI → domain → data):**
- **UI layer** - UI elements (Compose, or XML Views in legacy code) + state holders (`ViewModel`)
- **Domain layer (optional)** - Use cases for complex or reused business logic
- **Data layer** - Repositories (single entry point to data) + data sources

**Principles:**
- **Separation of concerns** - Activities/Fragments only host UI; no app data or business logic in them
- **Drive UI from data models**, ideally persistent ones
- **Single source of truth (SSOT)** - only the owner of data mutates it; it exposes immutable data
- **Unidirectional data flow (UDF)** - state flows down, events flow up
- **Dependency injection** - constructor injection with the project's DI approach (Hilt is the official recommendation; Koin or manual DI are equally valid)
- **Feature-first modularization** - features as end-to-end slices; minimal public surface per module

See: [references/architecture.md](references/architecture.md)

### 7. UI Layer Rules

- One immutable `UiState` per screen, exposed as `StateFlow` (or `LiveData` if the project uses it) via a private mutable backing property
- ViewModels are screen-level, never hold `Context`, `Resources`, `View`, `Activity`, or `Fragment`
- ViewModel events become UI state - no one-off event channels for things the UI must not miss
- Collect state lifecycle-aware (`collectAsStateWithLifecycle()`; `repeatOnLifecycle` in Views)
- Never pass a ViewModel to child composables, adapters, or other classes - pass state and lambdas
- State survives configuration changes and process death (`ViewModel`, `SavedStateHandle`, `rememberSaveable`) - never opt out of recreation as a fix

See: [references/ui-layer.md](references/ui-layer.md)

### 8. Coroutines & Flow Rules

- Inject dispatchers; never hardcode `Dispatchers.IO` in classes
- `suspend` functions are main-safe: the class doing blocking work switches context
- Data/domain layers expose `suspend` for one-shot calls and `Flow` for streams
- The ViewModel launches coroutines (`viewModelScope`); the UI doesn't launch business logic
- No `GlobalScope`; inject a scope for work that must outlive the screen
- Never swallow `CancellationException`; catch specific exceptions

See: [references/coroutines-flow.md](references/coroutines-flow.md)

### 9. Compose Rules (default UI toolkit)

- `Unit`-returning composables are PascalCase nouns; value-returning ones camelCase; `remember*` for remembered factories
- Exactly one `modifier: Modifier = Modifier` parameter, applied once, first, to the root
- Parameter order: required → `modifier` → optional → trailing `content` slot
- Stateless composables with hoisted state and `onX` event lambdas
- Never write state during composition; write it in event callbacks or effects
- Measure on release builds before optimizing; for profiling use the `android-profiler` skill

XML Views are acceptable for legacy code; the same UDF and state principles apply. Migrating a screen to Compose is the `migrate-xml-views-to-jetpack-compose` skill, and only when the user asks.

See: [references/compose.md](references/compose.md)

### 10. UI Correctness Rules

- Never hardcode user-facing strings; complete default `values/` resources
- Never concatenate localized strings; use format arguments
- Plurals only for grammatical number, never as `if` logic
- Format numbers, dates, and currency with the current locale
- Interactive elements have at least 48dp touch targets and a meaningful label from resources
- `contentDescription` from string resources; `null` for decorative images
- Text uses `sp` and must survive large font and display sizes

## The Four Elements of Simple Design (XP)

In priority order:
1. **Runs all the tests** - Must work correctly
2. **Expresses intent** - Readable, reveals purpose
3. **No duplication** - DRY (but Rule of Three)
4. **Minimal** - Fewest classes, methods possible

## Code Smell Detection

**Stop and refactor when you see:**

| Smell | Solution |
|-------|----------|
| Long Method | Extract functions, compose method pattern |
| Large Class / God Activity | Extract class, move logic to ViewModel/use case/repository |
| Long Parameter List | Default + named arguments, or a domain `data class` |
| Divergent Change | Split into focused classes |
| Shotgun Surgery | Move related code together |
| Feature Envy | Move function to the envied class |
| Data Clumps | Extract class for grouped data |
| Primitive Obsession | Wrap in `value class` |
| Repeated `when` on type | Replace with polymorphism |
| Speculative Generality | YAGNI - remove unused abstractions |
| Clever Code / Hidden Intent | Rename, unroll, make explicit |

See: [references/code-smells.md](references/code-smells.md)

## Design Patterns Awareness

**Creational:** Singleton (`object`, DI scope), Factory, Builder (named/default args), Prototype (`copy()`)
**Structural:** Adapter, Decorator, Composite, Proxy (`by lazy`)
**Behavioral:** Strategy, Observer (`Flow`), Template Method, Command (`sealed` actions)

**Warning:** Don't force patterns. Let them emerge from refactoring.

See: [references/design-patterns.md](references/design-patterns.md)

## Testing Strategy

Installing or configuring test infrastructure (frameworks, runners, Robolectric, screenshot tools, coverage) belongs to the `testing-setup` skill. This skill covers how tests are designed and written.

**Test Types (from inner to outer):**
1. **Unit Tests** - Single class/function, local JVM (`src/test`), fast, isolated
2. **Integration Tests** - Multiple components together (local or instrumented)
3. **UI behavior tests** - Robolectric in `src/test` by default; instrumented `src/androidTest` when requested
4. **E2E Tests** - Few, instrumented, critical user journeys

**Rules:**
- Prefer **fakes** over mocks (official recommendation); keep the project's mocking library if it already uses one
- Test ViewModels, use cases, repositories, and data sources; don't unit test Activities/Fragments
- Coroutines: `runTest`, inject `TestDispatcher`s, replace `Main` with a `MainDispatcherRule`
- Assert on `uiState.value`, not on emission sequences

**Arrange-Act-Assert Pattern:**
```kotlin
@Test
fun add_twoAndThree_returnsFive() {
    // Arrange
    val calculator = Calculator()
    // Act
    val result = calculator.add(2, 3)
    // Assert
    assertEquals(5, result)
}
```

**Test Naming:** `subjectUnderTest_trigger_expectedResult`, concrete examples (no backtick names - not fully supported on Android)

See: [references/testing.md](references/testing.md)

## Behavioral Principles

- **Tell, Don't Ask** - Command objects, don't query and decide
- **Design by Contract** - Preconditions (`require`), invariants (`check`), postconditions
- **Hollywood Principle** - "Don't call us, we'll call you" (IoC, DI, observable state)
- **Law of Demeter** - Only talk to immediate friends

## Pre-Code Checklist

Before writing ANY code, answer:

1. [ ] Do I understand the requirement? (Write acceptance criteria first)
2. [ ] Which libraries does the project already use? Do I need to ask the user to choose one?
3. [ ] What test will I write first?
4. [ ] What is the simplest solution?
5. [ ] What patterns might apply? (Don't force them)
6. [ ] Am I solving a real problem or a hypothetical one?

## During-Code Checklist

While coding, continuously ask:

1. [ ] Is this the simplest thing that could work?
2. [ ] Does this class have a single responsibility?
3. [ ] Am I depending on abstractions or concretions?
4. [ ] Can I name this more clearly? Does the name reveal every effect?
5. [ ] Is there duplication I should extract? (Rule of Three)

## Post-Code Checklist

After the code works:

1. [ ] Do all tests pass?
2. [ ] Is there any dead code to remove?
3. [ ] Can I simplify any complex conditions?
4. [ ] Are names still accurate after changes?
5. [ ] Would a developer new to the codebase understand this after one read?
6. [ ] Is anything implicit that should be explicit (magic numbers, `Boolean` args, hidden side effects)?
7. [ ] Are strings in resources and interactive elements accessible?

## Red Flags - Stop and Rethink

- Changing behavior without a test that describes it
- Code you need to explain verbally to be understood
- Clever one-liners, nested scope functions, nested `it`
- Function name that hides a side effect
- Function longer than ~10-15 lines or nesting deeper than one level
- Hardcoding values that should be named or configurable
- Creating abstractions before the third duplication
- Adding features "just in case"
- Depending on concrete implementations
- God classes / God Activities and Fragments
- Logic in Activities or Fragments
- ViewModel holding `Context`, `Resources`, `View`, `Activity`, or `Fragment`; using `AndroidViewModel`
- Publicly exposed `MutableStateFlow`/`MutableLiveData`
- Collecting flows in the UI without lifecycle awareness
- One-off event `Channel`s for events the UI must not miss
- Hardcoded `Dispatchers.*`, `GlobalScope`, swallowed `CancellationException`
- `!!` instead of handling nullability
- Writing state during composition (backwards write)
- Hardcoded user-facing strings or concatenated strings
- Interactive element without a 48dp target or a label
- Silently adding, replacing, or migrating a library

## Remember

> "A little bit of duplication is 10x better than the wrong abstraction."

> "Focus on WHAT needs to happen, not HOW it needs to happen."

> "Code is read far more often than it is written. Optimize for the reader."

> "Design principles become second nature through practice. Eventually, you won't think about SOLID - you'll just write SOLID code."

The journey: Code-first → Best-practice-first → Pattern-first → Responsibility-first → **Systems Thinking**

Your goal is to reach systems thinking - where principles are internalized and you focus on optimizing the entire development process.
