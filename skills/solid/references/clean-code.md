# Clean Code Practices

> Parts of this guide are based on concepts from the **"Clean Code" course summary** by [Academind GmbH / Maximilian Schwarzmuller](https://academind.com) (c) 2020.
>
> Kotlin-specific rules follow the [Android Kotlin style guide](https://developer.android.com/kotlin/style-guide), [Kotlin coding conventions](https://kotlinlang.org/docs/coding-conventions.html), and [Common Kotlin patterns with Android](https://developer.android.com/kotlin/common-patterns). Where they disagree, the Android style guide wins.

## What is Clean Code?

Code that is:
- **Easy to understand** - reveals intent clearly, on first read
- **Easy to change** - modifications are localized
- **Easy to test** - dependencies are injectable
- **Simple** - no unnecessary complexity

## The Human-Centered Approach

Code has THREE consumers:
1. **Users** - get their needs met
2. **Customers** - make or save money
3. **Developers** - must maintain it

Design for all three, but remember: **developers read code 10x more than they write it.**

---

## Readability & Explicitness

**Code must be easily readable and understandable by humans, and it must say explicitly what it does.** If a reader has to run, debug, or jump across files to understand a function, the function is not done.

When readability conflicts with brevity, cleverness, or a size guideline, **readability wins**.

### Names reveal every effect

```kotlin
// BAD: hides that it also saves and sends a notification
fun process(draft: Draft) { /* validates, saves, notifies */ }

// GOOD: the name tells the whole story
fun saveDraftAndNotifyCollaborators(draft: Draft) { /* ... */ }
```

### Make arguments explicit

```kotlin
// BAD: what do true and false mean?
uploadPhoto(photo, true, false)

// GOOD: named arguments
uploadPhoto(photo, shouldCompress = true, isPublic = false)
```

Use named arguments for every `Boolean` argument and when several parameters share a type.

### No magic numbers

```kotlin
// BAD
if (password.length < 8) return PasswordStrength.Weak

// GOOD
if (password.length < MIN_PASSWORD_LENGTH) return PasswordStrength.Weak

private const val MIN_PASSWORD_LENGTH = 8
```

### Don't chain scope functions

```kotlin
// BAD: the reader must track what `it` and `this` are at each step
user?.let { it.profile }?.also { analytics.log(it.id) }?.run { avatarUrl ?: defaultAvatarUrl }

// GOOD: plain statements, named values
val profile = user?.profile ?: return defaultAvatarUrl
analytics.logProfileViewed(profile.id)
return profile.avatarUrl ?: defaultAvatarUrl
```

Scope functions are fine when they make code clearer (`apply` for configuring an object, a single `?.let` for a nullable value) - never as a puzzle.

### Name lambda parameters when nesting

```kotlin
// BAD
orders.filter { it.isPaid }.forEach { it.items.forEach { println(it.name) } }

// GOOD
for (order in orders.filter { order -> order.isPaid }) {
    order.items.forEach { item -> println(item.name) }
}
```

Use `it` only in short, non-nested lambdas.

### Prefer an early return over `?.let { } ?: run { }`

```kotlin
// BAD
user?.let { showProfile(it) } ?: run { showLoginPrompt() }

// GOOD
if (user == null) {
    showLoginPrompt()
    return
}
showProfile(user)
```

### Avoid surprising Kotlin features

- `infix` only for two values playing similar roles, never for functions with side effects
- Operator overloading only when the meaning is obvious (`Money + Money`)
- Extension functions for natural, discoverable operations - not to hide unrelated behavior on common types (`String.saveToDatabase()`)

### The one-read test

Before finishing, ask: **"Could a developer new to this codebase explain what this code does after reading it once?"** If not, rename, extract, or simplify. A comment is the last resort and only explains *why*.

---

## Naming Principles

### 1. Consistency & Uniqueness (HIGHEST PRIORITY)
Same concept = same name everywhere. One name per concept.

```kotlin
// BAD: Inconsistent names for same concept
getUserById(id)
fetchCustomerById(id)
retrieveClientById(id)

// GOOD: Consistent
getUser(id)
getOrder(id)
getProduct(id)
```

### 2. Understandability
Use domain language, not technical jargon.

```kotlin
// BAD: Technical
val arr = users.filter { u -> u.isActive }

// GOOD: Domain language
val activeCustomers = users.filter { user -> user.isActive }
```

### 3. Specificity
Avoid vague names: `data`, `info`, `manager`, `handler`, `processor`, `Util`, `Helper`, `Wrapper`.

```kotlin
// BAD: Vague
class DataManager
fun processInfo(data: Any) {}

// GOOD: Specific
class OrderRepository
fun validatePayment(payment: Payment) {}
```

### 4. Brevity (but not at cost of clarity)

```kotlin
// BAD: Too cryptic
val usrLst = getUsrs()

// BAD: Unnecessarily long
val listOfAllActiveUsersInTheSystem = getActiveUsers()

// GOOD: Brief but clear
val activeUsers = getActiveUsers()
```

### 5. Searchability

```kotlin
// BAD: Common word, hard to search
val data = fetch()

// GOOD: Unique, searchable
val orderSummary = fetchOrderSummary()
```

### 6. Pronounceability

```kotlin
// BAD
val genymdhms = generateYearMonthDayHourMinuteSecond()

// GOOD
val timestamp = generateTimestamp()
```

### 7. Austerity
Avoid unnecessary filler words.

```kotlin
// BAD: Redundant
val userData = user // 'Data' adds nothing
class UserClass     // 'Class' adds nothing

// GOOD
class User
```

### Kotlin Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Classes, interfaces, objects | PascalCase nouns | `OrderRepository` |
| Functions | camelCase verbs | `calculateTotal()` |
| `@Composable` returning `Unit` | PascalCase nouns | `OrderSummaryCard` |
| Properties, local variables | camelCase nouns | `activeOrders` |
| Booleans | questions | `isLoading`, `hasError`, `canSubmit` |
| Constants (`const val`, deeply immutable top-level/`object` `val`) | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT` |
| Backing properties | `_` prefix | `private val _uiState` |
| Packages | lowercase, no underscores | `com.example.orders` |
| Files | PascalCase, named after the main class | `OrderRepository.kt` |
| Test functions | `subject_trigger_result` (no backticks) | `checkout_emptyCart_showsError` |

- **No Hungarian/`m` prefixes** (`mName`, `sInstance`)
- **Acronyms as words**: `XmlHttpRequest`, `userId`, `HtmlParser`
- Values inside a `class` use camelCase even if they never change; use `const val` in a `companion object` for true constants
- Mutating vs copying: `sort()` changes in place, `sorted()` returns a copy

---

## Functions & Methods

Functions are the building blocks of any codebase. Clean functions are **small, focused, and readable**.

### Parameters: defaults and named arguments

Kotlin's idiom is **default parameter values + named arguments**, not overloads or "options" objects.

```kotlin
// BAD: Positional arguments of the same type
createRectangle(10, 9, 30, 12)

// GOOD: Named arguments
createRectangle(x = 10, y = 9, width = 30, height = 12)

// GOOD: Defaults instead of overloads
fun createUser(
    name: String,
    email: Email,
    newsletterOptIn: Boolean = false,
): User
```

Group parameters into a `data class` only when the group is a **real domain concept** (`Address`, `DateRange`), not just to reduce the count.

Composables are an explicit exception: many defaulted, named parameters is the official Compose style (see [compose.md](compose.md)).

### Keep Functions Small

Small functions are easier to read and force you to extract well-named helper functions.

```kotlin
// BAD: Long function with mixed concerns
fun login(email: String, password: String) {
    if (!email.contains("@") || password.length < 7) {
        throw IllegalArgumentException("Invalid input!")
    }
    val existingUser = database.query("users", "email", email)
        ?: throw UserNotFoundException(email)
    if (existingUser.password != password) {
        throw InvalidCredentialsException()
    }
    // create a session
}

// GOOD: Small, composed functions
fun login(email: Email, password: Password) {
    val existingUser = findUserByEmail(email)
    existingUser.verifyPassword(password)
    startSession(existingUser)
}
```

Expression bodies are welcome for single expressions: `fun isAdult(age: Int): Boolean = age >= ADULT_AGE`.

### Do One Thing (Levels of Abstraction)

All operations in the function body should be on the **same level** of abstraction, exactly **one level below** the function name.

```kotlin
// BAD: Mixed levels of abstraction
fun printDocument(documentPath: String) {
    val file = File(documentPath)
    val document = file.bufferedReader().use { reader -> reader.readText() }
    val printer = Printer(PrinterFormat.Pdf)
    printer.print(document)
}

// GOOD: Consistent abstraction level
fun printDocument(documentPath: String) {
    val document = readDocument(documentPath)
    val printer = Printer(PrinterFormat.Pdf)
    printer.print(document)
}
```

### Split Functions Reasonably

Don't extract just for the sake of extraction. **Three warning signs** of bad splits:
1. You're just **renaming the operation** (e.g. `buildUser()` that just calls `User(...)`)
2. You need to **scroll more** to follow a simple function
3. You **can't find a good name** because the original name is already taken

```kotlin
// OVER-SPLIT: Too many trivial extractions
fun createUser(email: Email, password: Password) {
    validateInput(email, password)
    saveUser(email, password)
}
fun saveUser(email: Email, password: Password) {
    val user = buildUser(email, password)
    userRepository.save(user)
}
fun buildUser(email: Email, password: Password) = User(email, password) // just renaming!

// BETTER: Meaningful splits only
fun createUser(email: Email, password: Password) {
    validateInput(email, password)
    userRepository.save(User(email, password))
}
```

**Two rules of thumb for when to split:**
1. Extract code that works on the **same functionality** / is closely related
2. Extract code that requires **more interpretation** than the surrounding code

### Avoid Unexpected Side Effects

A side effect is unexpected when the function name doesn't imply it.

```kotlin
// BAD: name says "validate", but it starts a session
fun validateUserInput(email: String, password: String) {
    require(isEmail(email) && isValidPassword(password)) { "Invalid input!" }
    startSession() // unexpected!
}

// GOOD: separate, or rename to imply it
fun validateInputAndStartSession(email: String, password: String) {
    validateUserInput(email, password)
    startSession()
}
```

Properties must never have side effects. Use a property only when it's cheap, can't throw, and returns the same value while state doesn't change; otherwise use a function.

---

## Control Structures

### Prefer Positive Checks

```kotlin
// GOOD: Positive check - zero thinking required
if (blogContent.isEmpty()) { /* ... */ }

// LESS CLEAR: Negated positive - extra interpretation
if (!blogContent.hasContent()) { /* ... */ }
```

Exception: a negative check is cleaner when it avoids listing many valid states (`if (!transaction.isOpen)`). Kotlin provides positive pairs: `isNotEmpty()`, `isNullOrBlank()`, `none { }`.

### Avoid Deep Nesting

#### 1. Use Guards & Fail Fast

```kotlin
// BAD: Deep nesting
fun messageUser(user: User?, message: String?) {
    if (user != null) {
        if (message != null) {
            if (user.acceptsMessages) {
                user.sendMessage(message)
            }
        }
    }
}

// GOOD: Guard clauses
fun messageUser(user: User?, message: String?) {
    if (user == null || message == null) return
    if (!user.acceptsMessages) return
    user.sendMessage(message)
}

// GOOD: Elvis guard
fun showProfile(userId: UserId) {
    val user = userRepository.findUser(userId) ?: return
    render(user)
}
```

#### 2. Use `if`/`when` as expressions

Kotlin prefers expressions over statements when choosing a value. Use `if` for two branches, `when` for three or more.

```kotlin
val discountPercent = if (user.isPremium) PREMIUM_DISCOUNT else NO_DISCOUNT

val shippingCost = when (shippingMethod) {
    ShippingMethod.Standard -> Money.of(5)
    ShippingMethod.Express -> Money.of(15)
    ShippingMethod.SameDay -> Money.of(25)
}
```

An exhaustive `when` over a `sealed`/`enum` type (no `else` branch) makes the compiler flag every place to update when a new case is added.

#### 3. Extract Control Structures into Functions

```kotlin
// BAD: Nested control structure
fun connectDatabase(uri: String?): ConnectionDetails {
    if (uri == null) throw IllegalArgumentException("A URI is required!")
    val database = Database(uri)
    if (!database.connect()) {
        if (database.hasFallbackConnection) {
            return database.fallbackConnectionDetails
        } else {
            throw ConnectionException("Could not connect!")
        }
    }
    return database.connectionDetails
}

// GOOD: Extract nested logic
fun connectDatabase(uri: String?): ConnectionDetails {
    requireNotNull(uri) { "A URI is required!" }
    val database = Database(uri)
    if (database.connect()) return database.connectionDetails
    return connectFallbackDatabase(database)
}
```

#### 4. Use Polymorphism

Replace repeated type checks with `sealed` types or interfaces.

```kotlin
// BAD: Repeated type checks
fun processTransaction(transaction: Transaction) {
    if (transaction.type == "payment") {
        if (transaction.method == "card") processCardPayment(transaction)
        if (transaction.method == "paypal") processPaypalPayment(transaction)
    } else {
        if (transaction.method == "card") processCardRefund(transaction)
        if (transaction.method == "paypal") processPaypalRefund(transaction)
    }
}

// GOOD: Polymorphic processor
interface PaymentProcessor {
    fun processPayment(transaction: Transaction)
    fun processRefund(transaction: Transaction)
}

fun processTransaction(transaction: Transaction, processor: PaymentProcessor) {
    when (transaction.kind) {
        TransactionKind.Payment -> processor.processPayment(transaction)
        TransactionKind.Refund -> processor.processRefund(transaction)
    }
}
```

#### 5. Embrace Errors (not error codes)

```kotlin
// BAD: Synthetic error codes
fun createUser(email: String, password: String): Int {
    if (!email.contains("@")) return 1
    if (password.length < MIN_PASSWORD_LENGTH) return 2
    // ...
    return 0
}

// GOOD: Real errors
fun createUser(email: String, password: String): User {
    require(email.contains("@")) { "Invalid email: $email" }
    require(password.length >= MIN_PASSWORD_LENGTH) { "Password too short" }
    if (userRepository.exists(email)) throw EmailAlreadyTakenException(email)
    // ...
}
```

- `require(...)` for invalid arguments (`IllegalArgumentException`), `check(...)` for invalid state (`IllegalStateException`), `requireNotNull`/`checkNotNull` for nulls
- Domain-specific exceptions for expected failures, or a `sealed` result type / `Result` when the caller must handle them
- Catch **specific** exceptions; only catch to recover or add context
- **Never swallow `CancellationException`** in coroutines

### Nullability

- Prefer non-null types; make nullability meaningful
- `?.` and `?:` instead of `!!` - `!!` reintroduces `NullPointerException`
- `?: return` / `?: throw` for guards
- Use `requireContext()`/`requireActivity()` instead of `context!!`
- Treat Java platform types as nullable and declare explicit Kotlin types for them
- Check nullable `Boolean`s with `== true` / `== false`

---

## Classes & Objects

### Objects vs Data Holders

Distinguish between **objects** (hide data, expose behavior) and **data holders** (expose data, no behavior).

```kotlin
// Data holder: exposes data
data class UserDetails(val name: String, val age: Int)

// Object: hides data, exposes behavior
class Account(private var balance: Money) {
    fun withdraw(amount: Money): WithdrawResult { /* ... */ }
}
```

Both are valid - use the right kind for the right job. **Don't mix types**: don't reach into the internals of objects, and don't add business behavior to data holders (derived read-only properties are fine).

### Class Cohesion

**Cohesion** = how much methods use class properties. High cohesion means every method uses most properties.

- **High cohesion** → well-designed, focused class
- **Low cohesion** → should probably be a data holder or split into smaller classes

### Visibility and Immutability

- Use the narrowest visibility (`private`, `internal`)
- Prefer `val` over `var`, and read-only `List`/`Set`/`Map` over mutable ones
- Never expose mutable state; use a private backing property

```kotlin
class ShoppingCart {
    private val _items = mutableListOf<CartItem>()
    val items: List<CartItem> get() = _items
}
```

- `lateinit` only for values guaranteed to be set before use (e.g. injected fields, test setup)
- `by lazy` for expensive values computed on first access

### Class Layout

1. Properties and `init` blocks
2. Secondary constructors
3. Functions (public first, helpers below in order of use)
4. `companion object` last

Keep related members together; don't sort by visibility.

---

## Object Calisthenics (adapted to Kotlin)

Exercises to improve OO design. Each rule keeps its **intent**, expressed in idiomatic Kotlin. Treat them as strong guidelines, not as reasons to write unidiomatic code.

### 1. Shallow Nesting

```kotlin
// BAD: Multiple levels
fun process(orders: List<Order>) {
    for (order in orders) {
        if (order.isValid) {
            for (item in order.items) {
                if (item.isInStock) {
                    ship(item)
                }
            }
        }
    }
}

// GOOD: Collection operations and extraction
fun process(orders: List<Order>) {
    orders.filter { order -> order.isValid }.forEach(::processOrder)
}

private fun processOrder(order: Order) {
    order.items.filter { item -> item.isInStock }.forEach(::ship)
}
```

### 2. Prefer Expressions and Early Returns over `else` Blocks

The original rule "don't use `else`" exists to avoid nested branching. In Kotlin, keep that intent:

```kotlin
// BAD: Statement-style branching with mutation
fun getDiscount(user: User): Int {
    var discount: Int
    if (user.isPremium) {
        discount = PREMIUM_DISCOUNT
    } else {
        discount = 0
    }
    return discount
}

// GOOD: Expression
fun getDiscount(user: User): Int = if (user.isPremium) PREMIUM_DISCOUNT else 0
```

- Guard clauses and `?: return` for preconditions
- `if`/`else` and `when` **as expressions** are idiomatic and encouraged
- Avoid `else` blocks that hide long logic - extract or return early

### 3. Wrap Meaningful Primitives

Wrap primitives in domain types when they carry **meaning or invariants** - not every primitive.

```kotlin
// BAD: Primitive obsession
fun createUser(email: String, age: Int) {}

// GOOD: Value classes (zero runtime overhead)
@JvmInline
value class Email(val value: String) {
    init { require(EMAIL_PATTERN.matches(value)) { "Invalid email" } }
}

@JvmInline
value class Age(val years: Int) {
    init { require(years in VALID_AGE_RANGE) { "Invalid age: $years" } }
}

fun createUser(email: Email, age: Age) {}
```

Always wrap: IDs, emails, money, quantities with units. Use a `data class` for multi-field values (`Money(amount, currency)`).

### 4. First-Class Collections

When a collection has its own rules, give it its own class:

```kotlin
class OrderItems(private val items: List<OrderItem> = emptyList()) {
    val total: Money get() = items.fold(Money.ZERO) { sum, item -> sum + item.price }
    val isEmpty: Boolean get() = items.isEmpty()
    fun add(item: OrderItem): OrderItems = OrderItems(items + item)
}

data class Order(val id: OrderId, val customerId: CustomerId, val items: OrderItems)
```

### 5. Law of Demeter

Don't reach through object graphs:

```kotlin
// BAD: Train wreck through domain objects
val city = order.customer.address.city

// GOOD: Ask the immediate friend
val city = order.shippingCity
```

Chained calls on collections, Flows, and builders are idiomatic and fine:

```kotlin
val names = users.filter { user -> user.isActive }.map { user -> user.name }.sorted()
```

### 6. Don't Abbreviate

If a name is too long to type, the class is doing too much.

```kotlin
// BAD
val custRepo = CustRepo()

// GOOD
val customerRepository = CustomerRepository()
```

### 7. Keep All Entities Small

Guidelines, not hard limits:
- Functions: ~10-15 lines
- Classes: small and focused on one responsibility
- Files: one main class (plus closely related small types)

If larger, it's probably doing too much. Split it - unless splitting makes it harder to read.

### 8. High Cohesion over Instance-Variable Limits

The original "max two instance variables" rule pushes toward small, cohesive classes. In Kotlin, `data class`es and primary-constructor properties are normal (and required by tools like `@Parcelize`). Keep the intent:
- If a class has many properties, check whether they form smaller concepts (`Address`, `DateRange`)
- If methods only use a subset of properties, split the class

### 9. Behavior over Exposed Mutable State

Kotlin properties replace getters/setters - use them. The intent is: **tell objects what to do instead of manipulating their state from outside.**

```kotlin
// BAD: Caller manipulates state
class Account {
    var balance: Money = Money.ZERO
}

if (account.balance >= amount) {
    account.balance = account.balance - amount
}

// GOOD: Behavior-rich object, read-only exposure
class Account(initialBalance: Money) {
    var balance: Money = initialBalance
        private set

    fun withdraw(amount: Money): WithdrawResult {
        if (balance < amount) return WithdrawResult.InsufficientFunds
        balance -= amount
        return WithdrawResult.Success(balance)
    }
}
```

- No public `var` for state with invariants; use `private set` or immutable data + `copy()`

---

## Comments

### When to Write Comments

**Only write comments to explain WHY, not WHAT or HOW.**

```kotlin
// BAD: Explains what (redundant)
// Add 1 to counter
counter++

// GOOD: Explains why
// Legacy API uses 1-based page numbers
val page = pageIndex + 1
```

### Prefer Self-Documenting Code

```kotlin
// BAD: Comment needed
// Check if user can access premium features
if (user.subscriptionLevel >= 2 && !user.isBanned) {}

// GOOD: Self-documenting
if (user.canAccessPremiumFeatures) {}
```

### KDoc

Write KDoc for public APIs of shared modules and libraries; skip it for obvious members.

---

## Formatting

- **4 spaces** indentation, no tabs
- **100 characters** max line length
- K&R braces; braces for multi-line `if`/`for`/`when` branches
- **Trailing commas** in multi-line declarations and calls
- No wildcard imports, no semicolons
- Break lines **before** `.`, `?.`, `::` and **after** binary operators
- Long signatures: one parameter per line

```kotlin
fun createOrder(
    customerId: CustomerId,
    items: List<OrderItem>,
    deliveryAddress: Address,
): Order {
    // ...
}
```

### Vertical Spacing
- Related code together
- Blank lines between concepts
- Most important/public at top

### Storytelling
Code should read top-to-bottom like a story. High-level at top, details below.

```kotlin
class OrderProcessor(/* ... */) {
    // Public API first
    fun process(order: Order): ProcessResult {
        validate(order)
        val total = calculateTotal(order)
        return save(order, total)
    }

    // Supporting functions below, in order of appearance
    private fun validate(order: Order) { /* ... */ }
    private fun calculateTotal(order: Order): Money { /* ... */ }
    private fun save(order: Order, total: Money): ProcessResult { /* ... */ }
}
```

---

## Clean Code Checklist

### Readability
- [ ] Understandable after **one read** by someone new to the codebase
- [ ] Names reveal **every effect** of a function
- [ ] No magic numbers; named arguments for `Boolean`s and same-typed parameters
- [ ] No scope-function chains, nested `it`, or clever one-liners

### Naming
- [ ] Use **descriptive** and meaningful names
  - Variables & Properties: Nouns or short phrases with adjectives
  - Functions: Verbs (composables returning `Unit`: nouns)
  - Classes: Nouns
- [ ] Be as **specific** as necessary and possible
- [ ] Use **yes/no "questions"** for booleans (e.g. `isValid`)
- [ ] Avoid **misleading** names
- [ ] Be **consistent** with names (e.g. stick to `get...` instead of `fetch...`)

### Comments & Formatting
- [ ] **Most comments are bad** - avoid them!
- [ ] Acceptable comments: legal info, warnings, regex explanations, *why* explanations
- [ ] Use **vertical formatting**: keep related concepts close, separate unrelated ones
- [ ] Write code **top to bottom**: called functions below calling functions
- [ ] Follow the Kotlin style guide (4 spaces, 100 columns, trailing commas)

### Functions
- [ ] **Few positional parameters** - defaults and named arguments
- [ ] Functions should be **small and do one thing**
  - Abstraction levels in the body should be **one level below** the function name
  - **Don't mix** levels of abstraction
  - But **avoid redundant splitting**!
- [ ] Stay **DRY** (Rule of Three)
- [ ] **Avoid unexpected side effects**

### Control Structures & Errors
- [ ] Prefer **positive checks**
- [ ] Avoid **deep nesting**
  - Use **guard** clauses and `?: return`
  - Use `if`/`when` **expressions**, exhaustive `when` on `sealed` types
  - **Extract** control structures into separate functions
- [ ] Use **real errors** (`require`, `check`, exceptions, `Result`) instead of error codes
- [ ] No `!!`; never swallow `CancellationException`

### Objects & Classes
- [ ] Build either **objects** or **data holders** - don't mix
- [ ] Build **small classes** focused on a **single responsibility** (not "single method"!)
- [ ] Build classes with **high cohesion**
- [ ] Prefer `val`, read-only types, narrow visibility; never expose mutable state
- [ ] Follow the **Law of Demeter** for object graphs
- [ ] Follow the **SOLID principles**
