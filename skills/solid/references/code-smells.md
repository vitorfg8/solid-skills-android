# Code Smells & Anti-Patterns

## What Are Code Smells?

Indicators that something MAY be wrong. Not bugs, but design problems that make code hard to understand, change, or test.

**Scope:** refactor smells only in code you touch for the task. Report smells elsewhere as suggestions instead of changing them - unrequested refactors make the change harder to review and riskier to ship.

## The Six Categories

### 1. Bloaters
Code that has grown too large.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Long Function** | More than ~10-15 lines, several concerns | Extract Function |
| **Large Class** | Multiple responsibilities (God Activity, God ViewModel) | Extract Class, move logic to the right layer |
| **Long Parameter List** | Many positional parameters of the same type | Named/default arguments, domain `data class` |
| **Data Clumps** | Same group of variables appear together | Extract `data class` |
| **Primitive Obsession** | Primitives instead of small domain types | Wrap in `value class` |

### 2. Object-Orientation Abusers
Misuse of OO principles.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Repeated Type Switches** | Same `when`/`if` on a type string or flag in several places | `sealed` type or interface (polymorphism) |
| **Parallel Inheritance** | Adding a subclass requires adding another | Merge hierarchies |
| **Refused Bequest** | Subclass doesn't use parent behavior | Replace inheritance with delegation |
| **Base Class Dumping Ground** | `BaseActivity`/`BaseViewModel` full of shared helpers | Composition, extension functions |

### 3. Change Preventers
Code that makes changes difficult.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Divergent Change** | One class changed for many reasons | Extract Class (SRP) |
| **Shotgun Surgery** | One change touches many classes | Move related code together |
| **Parallel Inheritance** | (see above) | Merge hierarchies |

### 4. Dispensables
Code that can be removed.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Comments** | Explaining bad code | Rename, Extract Function |
| **Duplicate Code** | Copy-paste | Extract Function (Rule of Three) |
| **Dead Code** | Unreachable or unused code | Delete |
| **Speculative Generality** | "Just in case" code | Delete (YAGNI) |
| **Lazy Class** | Class that does almost nothing (e.g. a use case that only forwards one call without adding value) | Inline Class |

### 5. Couplers
Excessive coupling between classes.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Feature Envy** | Function uses another class's data extensively | Move Function |
| **Inappropriate Intimacy** | Classes know too much about each other | Move Function, Extract Class |
| **Message Chains** | `order.customer.address.city` | Hide Delegate |
| **Middle Man** | Class only delegates | Inline Class |

### 6. Obscurers
Code that hides what it does. These directly violate the Readability First principle.

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Clever Code** | Chained scope functions, nested `it`, dense one-liners, surprising operators | Unroll into named values and plain statements |
| **Hidden Intent** | Name hides side effects (`validate()` that saves), `Boolean` arguments, magic numbers | Rename to reveal effects, named arguments, constants |
| **Implicit Dependencies** | Singletons/global state accessed from inside functions | Constructor injection |

---

## Android-Specific Smells

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **God Activity/Fragment** | Business logic, data access, or state in UI components | Move to ViewModel, use case, repository |
| **Context in ViewModel** | ViewModel holds `Context`, `Resources`, `View`, `Activity`, `Fragment`; uses `AndroidViewModel` | Move framework work to the UI or a data-layer abstraction |
| **ViewModel passed down** | ViewModel given to child composables, adapters, or other classes | Pass state and lambdas |
| **Layer skipping** | ViewModel or use case calls a database/API/preferences directly | Go through a repository |
| **Exposed mutable state** | Public `MutableStateFlow`/`MutableLiveData`/`MutableList` | Private backing property, read-only exposure |
| **Lifecycle-unaware collection** | `lifecycleScope.launch { flow.collect {} }` without `repeatOnLifecycle` | `repeatOnLifecycle` / `collectAsStateWithLifecycle` |
| **One-off event channels** | `Channel`/`SharedFlow` for navigation or messages the UI must not miss | Model as UI state + "consumed" event |
| **Hardcoded dispatchers** | `withContext(Dispatchers.IO)` hardcoded in a function body | Inject a `CoroutineDispatcher` through the constructor (a `Dispatchers.IO` default value is fine) |
| **GlobalScope** | Work launched in `GlobalScope` | Injected scope |
| **Swallowed cancellation** | `catch (e: Exception)` around suspend calls without rethrowing `CancellationException` | Catch specific exceptions |
| **Main-thread I/O** | Disk or network work on the main thread | Main-safe data layer |
| **`!!` everywhere** | Forced non-null assertions | `?.`, `?:`, `requireNotNull`, better types |
| **Opting out of recreation** | `android:configChanges` or locked orientation to "fix" state loss | `ViewModel`, `SavedStateHandle`, `rememberSaveable` |
| **Silent library swap** | Introducing or migrating a library (DI, storage, image loading) without asking | Follow existing choices; ask the user |

### Compose Smells

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Backwards write** | State written in the composition body | Write in event callbacks or effects |
| **Misused `modifier`** | Missing, applied to a child, `Modifier.x().then(modifier)`, or non-empty default | One `modifier: Modifier = Modifier` applied first to the root |
| **State as a parameter** | `MutableState<T>` or `State<T>` passed into a composable | Value + `onValueChange`, or `() -> T` |
| **Verb-named UI composables** | `RenderHeader()`, `showProfile()` emitting UI | PascalCase nouns (`ProfileHeader`) |

### Resources & Accessibility Smells

| Smell | Symptom | Refactoring |
|-------|---------|-------------|
| **Hardcoded strings** | User-facing text in Kotlin/XML layouts | String resources |
| **String concatenation** | `getString(a) + name + getString(b)` | One resource with format arguments |
| **Plurals as logic** | Plurals used for "only if count == 1" decisions | Regular `if`; plurals only for grammar |
| **Resolved text in ViewModel** | ViewModel formats localized strings with `Resources` | Expose data/resource IDs; resolve in UI |
| **Unlabeled interactive element** | Icon button without `contentDescription`; touch target < 48dp | Label from resources; `sizeIn(min = 48.dp)` |
| **Fixed-size text containers** | `height(40.dp)` around text | `heightIn(min = ...)`, `sp` text |

---

## The Most Common Code Smells

### 1. Long Function

**Symptom:** Function doing multiple things.

```kotlin
// SMELL
suspend fun processOrder(order: Order) {
    // Validate
    require(order.items.isNotEmpty()) { "Empty order" }
    requireNotNull(order.customer) { "No customer" }

    // Calculate
    var total = Money.ZERO
    for (item in order.items) {
        total += item.price * item.quantity
        item.discount?.let { discount -> total -= discount }
    }

    // Apply tax
    val taxRate = taxRates.getValue(order.customer.state)
    total *= (BigDecimal.ONE + taxRate)

    // Save
    orderDao.insert(order.toEntity(total))

    // Notify
    notificationSender.send(order.customer.email, "Order confirmed")
}

// REFACTORED
suspend fun processOrder(order: Order) {
    validateOrder(order)
    val total = calculateTotal(order)
    orderRepository.saveOrder(order, total)
    notifyCustomer(order)
}
```

### 2. Large Class / God Activity

**Symptom:** Class with many responsibilities.

```kotlin
// SMELL: God Activity
class ProfileActivity : AppCompatActivity() {
    private val database by lazy { AppDatabase.getInstance(this) }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // loads user from database, calls the API, validates input,
        // formats dates, saves preferences, renders UI...
    }
}

// REFACTORED: each responsibility in its layer
class ProfileActivity : ComponentActivity()                    // hosts UI only
class ProfileViewModel(/* use cases */) : ViewModel()          // UI state
class UpdateProfileUseCase(/* repositories */)                 // business rule
class DefaultUserRepository(/* data sources */) : UserRepository // data
```

### 3. Feature Envy

**Symptom:** Function uses another class's data more than its own.

```kotlin
// SMELL: Order envies Customer
class Order(val customer: Customer) {
    fun calculateShipping(): Money {
        if (customer.country == Country.US) {
            return if (customer.state == State.CA) Money.of(10) else Money.of(15)
        }
        return Money.of(25)
    }
}

// REFACTORED: Move to Customer
class Customer(val country: Country, val state: State?) {
    val shippingCost: Money
        get() = when {
            country != Country.US -> Money.of(25)
            state == State.CA -> Money.of(10)
            else -> Money.of(15)
        }
}

class Order(val customer: Customer) {
    val shippingCost: Money get() = customer.shippingCost
}
```

### 4. Primitive Obsession

**Symptom:** Using primitives for domain concepts.

```kotlin
// SMELL
fun createUser(email: String, age: Int, zipCode: String) {
    // No validation, easy to swap arguments
}

// REFACTORED: Value classes
@JvmInline value class Email(val value: String) {
    init { require(EMAIL_PATTERN.matches(value)) { "Invalid email" } }
}

@JvmInline value class Age(val years: Int) {
    init { require(years in VALID_AGE_RANGE) { "Invalid age" } }
}

fun createUser(email: Email, age: Age, address: Address) {
    // The type system prevents invalid or swapped data
}
```

### 5. Repeated Type Switches

**Symptom:** Switching on a type, repeated across the codebase.

```kotlin
// SMELL
fun area(shape: ShapeData): Double = when (shape.type) {
    "circle" -> PI * shape.radius * shape.radius
    "rectangle" -> shape.width * shape.height
    else -> throw IllegalArgumentException()
}

fun perimeter(shape: ShapeData): Double = when (shape.type) { // same switch again!
    "circle" -> 2 * PI * shape.radius
    else -> TODO()
}

// REFACTORED: Polymorphism
sealed interface Shape {
    val area: Double
    val perimeter: Double

    data class Circle(val radius: Double) : Shape {
        override val area: Double get() = PI * radius * radius
        override val perimeter: Double get() = 2 * PI * radius
    }

    data class Rectangle(val width: Double, val height: Double) : Shape {
        override val area: Double get() = width * height
        override val perimeter: Double get() = 2 * (width + height)
    }
}
```

### 6. Inappropriate Intimacy

**Symptom:** Classes know too much about each other's internals.

```kotlin
// SMELL
class Order(private val items: List<OrderItem>) {
    fun process(inventory: Inventory) {
        for (item in items) {
            val stock = inventory.stockLevels.getValue(item.sku) // reaching into internals
            check(stock.quantity >= item.quantity) { "Out of stock" }
            stock.quantity -= item.quantity
        }
    }
}

// REFACTORED: Tell, don't ask
class Inventory(private val stockLevels: MutableMap<Sku, Stock>) {
    fun reserve(items: List<OrderItem>): ReserveResult {
        val unavailableItem = items.firstOrNull { item -> !canReserve(item) }
        if (unavailableItem != null) return ReserveResult.OutOfStock(unavailableItem)
        deductStock(items)
        return ReserveResult.Success
    }
    // ...
}

class Order(private val items: List<OrderItem>) {
    fun process(inventory: Inventory) {
        val result = inventory.reserve(items)
        if (result is ReserveResult.OutOfStock) throw OutOfStockException(result.item)
    }
}
```

### 7. Speculative Generality

**Symptom:** "Just in case" abstractions that aren't used.

```kotlin
// SMELL: Over-engineered for hypothetical needs
interface PaymentProcessor {
    fun process()
    fun rollback()
    fun audit()
    fun generateReport()
    fun scheduleRecurring()
}

class StripeProcessor : PaymentProcessor {
    override fun process() { /* actual code */ }
    override fun rollback() = TODO()
    override fun audit() = TODO()
    override fun generateReport() = TODO()
    override fun scheduleRecurring() = TODO()
}

// REFACTORED: YAGNI
interface PaymentProcessor {
    fun process()
}
// Add other functions when actually needed
```

### 8. Clever Code

**Symptom:** The reader has to decode the code to understand it.

```kotlin
// SMELL
fun badge(user: User?) = user?.takeIf { it.isActive }?.let { it.orders.count { it.isPaid } }
    ?.takeIf { it > 0 }?.let { "$it" } ?: ""

// REFACTORED: explicit steps with names
fun paidOrdersBadge(user: User?): String {
    if (user == null || !user.isActive) return ""
    val paidOrderCount = user.orders.count { order -> order.isPaid }
    return if (paidOrderCount > 0) paidOrderCount.toString() else ""
}
```

### 9. Hidden Intent

**Symptom:** You must read the implementation to know what a call does.

```kotlin
// SMELL
settings.update(true, false, 3)

// REFACTORED
settings.update(
    areNotificationsEnabled = true,
    isDarkModeForced = false,
    syncIntervalHours = DEFAULT_SYNC_INTERVAL_HOURS,
)
```

---

## Prevention Strategies

1. **Follow the Readability First principle** - code that's explicit hides fewer smells
2. **Practice TDD** - Tests reveal design problems early
3. **Review in pairs** - Fresh eyes catch smells
4. **Refactor continuously** - Don't let smells accumulate
5. **Apply SOLID** - Prevents structural smells
6. **Use static analysis** - Android Lint, detekt/ktlint (if the project uses them), Compose compiler reports

---

## When You Find a Smell

1. **Confirm it's a problem** - Not all smells need fixing
2. **Ensure test coverage** - Before refactoring
3. **Refactor in small steps** - Keep tests passing
4. **Commit frequently** - Easy to revert if needed
