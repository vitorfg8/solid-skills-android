# Solid Skills for Android

Professional software engineering skills for AI coding agents working on **Android with Kotlin**. Transforms code into senior-engineer quality software through SOLID principles, TDD, clean and human-readable Kotlin, and the official Android architecture guidance.

Skills follow the [Agent Skills](https://github.com/anthropics/skills) format.

## Available Skills

### solid

Transform junior-level code into senior-engineer quality Android software. Designed for **Android + Kotlin** projects, with **Jetpack Compose** as the default UI toolkit and XML Views supported for legacy code.

**Use when** working on Kotlin code in Android app modules:

- Writing features, fixes, and utilities
- Refactoring existing code
- Planning or designing architecture
- Reviewing code quality
- Debugging issues
- Creating tests
- Making design decisions

**Core principles:**

| Principle | Focus |
|-----------|-------|
| Readability First | Code read by humans and agents with equal weight, explicit about what it does |
| TDD | Red-Green-Refactor cycle, tests before code |
| SOLID | Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion |
| Clean Code | Kotlin style guide, meaningful names, small functions, idiomatic Kotlin |
| Android Architecture | UI / domain / data layers, UDF, single source of truth, ViewModels, repositories |
| Coroutines & Flow | Main-safety, injected dispatchers, StateFlow, structured concurrency |
| Compose | API design, state hoisting, readable composables |
| UI Correctness | String resources, localization, touch targets, labels |

**Reference documentation included:**

- `solid-principles.md` - SOLID principles with Kotlin/Android examples
- `tdd.md` - Test-Driven Development practices
- `testing.md` - Testing strategy, test doubles, testing by layer
- `clean-code.md` - Clean, readable and explicit Kotlin
- `code-smells.md` - Code smells (general, Android, Compose, resources) and fixes
- `design-patterns.md` - GoF patterns expressed with Kotlin idioms
- `architecture.md` - Official Android app architecture
- `object-design.md` - Object stereotypes and responsibilities
- `complexity.md` - Managing essential vs accidental complexity
- `ui-layer.md` - UI state, events, state holders, ViewModel, saved state
- `coroutines-flow.md` - Coroutines, Flow, StateFlow and their tests
- `compose.md` - Compose API design and readability

**Key features:**

- Enforces TDD workflow (write failing test first)
- Prioritizes readable, explicit code over clever code - for humans and agents alike
- Follows the official Android and Kotlin guidance (style guide, architecture recommendations, Compose API guidelines)
- Applies SOLID principles to every class and function
- Uses value classes for domain primitives (IDs, emails, money)
- Detects and fixes code smells, including Android- and Compose-specific ones
- **Library neutral:** follows the libraries your project already uses (e.g. Koin or Hilt, SharedPreferences or DataStore) and asks you to choose when starting something new - it never silently adds or migrates a library

## Use with the Android CLI

This skill is **self-contained**: every rule works without any other tool or skill installed. It is complementary to the Android CLI (`android`) and the official [Android skills](https://github.com/android/skills):

- **This skill** covers the quality of the code: SOLID, TDD, readability, naming, and architecture boundaries.
- **The Android CLI and skills** cover platform workflows and current API guidance: test infrastructure, migrations, adaptive layouts, navigation, edge-to-edge, profiling, R8, security, and SDK integrations.

When the `android` CLI is available, the agent uses it to stay current and to find workflow skills:

```bash
android docs search "<keywords>"   # confirm the current official guidance for an API
android skills find "<keyword>"    # find a workflow skill for a platform task
```

When both apply, the Android skill drives the platform-specific steps and this skill drives how the resulting code is written. If an Android skill would add or replace a library, this skill still asks you first.

## Installation

```bash
npx skills add vitorfg8/solid-skills-android
```

## Usage

Skills are automatically available once installed. The agent will use them when relevant tasks are detected.

**Examples:**

- "Implement a login screen"
- "Refactor this Fragment to follow the official architecture"
- "Review this ViewModel for quality issues"
- "Add tests for this repository"
- "Design the architecture for an offline-first notes app"

## Skill Structure

```
skills/
└── solid/
    ├── SKILL.md           # Main skill instructions
    └── references/        # Supporting documentation
        ├── solid-principles.md
        ├── tdd.md
        ├── testing.md
        ├── clean-code.md
        ├── code-smells.md
        ├── design-patterns.md
        ├── architecture.md
        ├── object-design.md
        ├── complexity.md
        ├── ui-layer.md
        ├── coroutines-flow.md
        └── compose.md
```

## Credits

- Based on [solid-skills](https://github.com/ramziddin/solid-skills) by ramziddin.
- Clean code practices in `clean-code.md` include concepts from the **"Clean Code" course summary** by [Academind GmbH / Maximilian Schwarzmuller](https://academind.com) (c) 2020.
- Android and Kotlin guidance based on the official documentation at [developer.android.com](https://developer.android.com) and [kotlinlang.org](https://kotlinlang.org).

## License

MIT
