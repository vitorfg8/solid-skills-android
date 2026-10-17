# Test-Driven Development

## When TDD Applies

Red-Green-Refactor is for **new or changed behavior**, not for every file touch.

- **In scope:** use cases, repositories, data sources, ViewModels, and other JVM-testable logic. The three laws below apply here.
- **UI:** test screen behavior and state (a stateless composable, or the ViewModel state). Do not run a fake Red-Green-Refactor cycle for layout, styling, copy, or a one-line fix with no behavior change.
- **Debugging:** write a failing test when the bug is a behavior you can express. Do not block the fix on a test when the failure cannot be reproduced in a unit or screen test.

## Verify Before Finishing

Run the unit tests, Android lint, and configured Kotlin linters of the module you changed - not only the new test.

**1. Detect flavors:** look for `productFlavors` in the module's `build.gradle(.kts)` (or its convention plugin), or list the tasks with `./gradlew :<module>:tasks --all`.

**2. Choose the variant:**

| Change | Unit tests | Android lint |
|--------|------------|--------------|
| Module without flavors | `./gradlew :<module>:testDebugUnitTest` | `./gradlew :<module>:lintDebug` |
| Flavor source set (e.g. `src/prod/`) | `./gradlew :<module>:test<Flavor>DebugUnitTest` | `./gradlew :<module>:lint<Flavor>Debug` |
| Common code (`src/main/`) in a module with flavors | `./gradlew :<module>:test` (all variants) | `./gradlew :<module>:lint` |

**3. Run the Kotlin linters the project already configures** (plugin in `build.gradle(.kts)`, `libs.versions.toml`, or a convention plugin):

| Linter | Task |
|--------|------|
| detekt | `./gradlew :<module>:detekt` |
| ktlint | `./gradlew :<module>:ktlintCheck` |
| Spotless | `./gradlew :<module>:spotlessCheck` |

Skip linters that aren't configured; never install one to verify a change.

Fix failures you introduced; report pre-existing failures instead of hiding them.

## The Core Loop

```
RED → GREEN → REFACTOR → RED → ...
```

### RED Phase
Write a failing test that describes the behavior you want. The test should:
- Use domain language, not technical jargon
- Describe WHAT, not HOW
- Be a concrete example, not an abstract statement

```kotlin
// BAD: Abstract
@Test fun canAddNumbers() { /* ... */ }

// GOOD: Concrete example
@Test fun add_twoAndThree_returnsFive() { /* ... */ }
```

### GREEN Phase
Write the **simplest possible code** to make the test pass. Two strategies:

1. **Fake It** - Return a hardcoded value
   ```kotlin
   fun add(first: Int, second: Int): Int = 5 // Simplest thing!
   ```

2. **Obvious Implementation** - If you know the solution
   ```kotlin
   fun add(first: Int, second: Int): Int = first + second
   ```

**Prefer Obvious Implementation** when the solution is clear. Use Fake It only when unsure, and let the next tests drive the real implementation. **Never leave a faked value in finished code.**

### REFACTOR Phase
This is where **design happens**. Look for:
- Duplication (but wait for Rule of Three)
- Long functions to extract
- Poor names to improve
- Complex conditions to simplify
- Anything a new reader would struggle to understand

## The Three Laws of TDD

1. **No production code** without a failing test
2. **No more test code** than sufficient to fail (compilation failures count)
3. **No more production code** than sufficient to pass the one failing test

## The Rule of Three

**Only extract duplication when you see it THREE times.**

Why? Wrong abstractions are worse than duplication. Wait for the pattern to emerge.

```kotlin
// Duplication #1 - Leave it
// Duplication #2 - Note it, leave it
// Duplication #3 - NOW extract it
```

## Triangulation

Each new test "sculpts" the solution toward a general, robust implementation.

Think of **degrees of freedom** - like a car that needs forward/back, left/right, and rotation. Each test carves out one degree of freedom until the implementation handles all cases.

## Transformation Priority Premise

When going from RED to GREEN, prefer simpler transformations:

| Priority | Transformation |
|----------|----------------|
| 1 | {} → null |
| 2 | null → constant |
| 3 | constant → variable |
| 4 | unconditional → conditional |
| 5 | scalar → collection |
| 6 | statement → recursion |
| 7 | value → mutated value |

Higher priority = simpler. Avoid jumping to complex transformations too early.

## Arrange-Act-Assert

Structure every test:

```kotlin
@Test
fun calculateTotal_tenPercentDiscount_returnsNinety() {
    // Arrange - Set up the world
    val order = Order(items = listOf(OrderItem(price = Money.of(100))))
    val discount = PercentDiscount(percent = 10)

    // Act - Execute the behavior
    val total = order.calculateTotal(discount)

    // Assert - Verify the outcome
    assertEquals(Money.of(90), total)
}
```

`// Given / When / Then` comments are an equivalent convention used in the official Android samples.

## Writing Tests Backwards

Sometimes it helps to write AAA in reverse:
1. Write the ASSERT first - what do you want to verify?
2. Write the ACT - what action produces that result?
3. Write the ARRANGE - what setup is needed?

## Test Naming Principles

- Use **behavior-driven names** with domain language
- Provide **concrete examples**, not abstract statements
- **One example per test** for easy debugging
- Avoid leaking implementation details
- Format: `subjectUnderTest_trigger_expectedResult` (underscores allowed in test names)
- **Don't use backtick names with spaces** - the Android Kotlin style guide forbids them because Android doesn't fully support them

```kotlin
// BAD: Technical, implementation-focused
@Test fun setsDataPropertyToOne() { /* ... */ }

// GOOD: Behavior-focused, domain language
@Test fun isPalindrome_mom_returnsTrue() { /* ... */ }
@Test fun login_wrongPassword_showsInvalidCredentialsMessage() { /* ... */ }
```

## Classic vs Mockist TDD

**Classic (Detroit/Chicago) TDD:**
- Test with real collaborators or **fakes**
- Higher confidence, tests survive refactoring
- Best for: domain logic, use cases, ViewModels with fake repositories

**Mockist (London) TDD:**
- Mock collaborators and verify interactions
- More isolated, but coupled to implementation details
- Best for: verifying an interaction that *is* the behavior (e.g. "analytics event was sent")

**Official Android guidance prefers fakes over mocks.** Start with classic TDD and fakes. If the project already uses a mocking library (MockK, Mockito), keep using it where it fits - don't migrate test suites unless asked.

## TDD on Android

- Drive **ViewModels, use cases, repositories, and data sources** with local JVM tests (`src/test`) - they run in milliseconds
- Keep Android framework types out of these classes so they stay testable without a device
- Use `runTest` and a `MainDispatcherRule` for coroutines (see [coroutines-flow.md](coroutines-flow.md))
- For UI behavior, test the stateless screen (state in, events out) or the ViewModel state. Skip TDD for layout, styling, and copy
- Don't TDD Activities/Fragments - they should contain no logic

## Common Mistakes

1. **Writing in-scope behavior before a test** - The three laws apply to use cases, repositories, data sources, ViewModels, and expressed bugs. They do not apply to layout, styling, copy, or a fix that cannot be reproduced in a test
2. **Writing too much test** - Just enough to fail
3. **Writing too much code** - Just enough to pass
4. **Skipping refactor** - This is where design lives
5. **Testing implementation** - Test behavior, not how it's done
6. **Abstract test names** - Use concrete examples
7. **Extracting too early** - Wait for Rule of Three
8. **Hardcoded dispatchers** - Make coroutine code untestable; inject them
