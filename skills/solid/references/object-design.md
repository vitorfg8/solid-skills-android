# Object-Oriented Design

## Responsibility-Driven Design (RDD)

The key insight: **Objects are defined by their responsibilities, not their data.**

### Finding Objects

Start with:
1. **Nouns** in requirements → candidate objects
2. **Verbs** → candidate functions/behaviors
3. **Domain concepts** → value objects (`value class`, `data class`)

### Finding Responsibilities

Each object should answer:
- What does this object **know**?
- What does this object **do**?
- What does this object **decide**?

### Object Stereotypes

Every class fits one (or maybe two) stereotypes. On Android they map onto familiar roles:

| Stereotype | Purpose | Android examples |
|------------|---------|------------------|
| **Information Holder** | Knows things, holds data | `User`, `Article` (`data class`), `NewsUiState` |
| **Structurer** | Maintains relationships | `OrderItems`, `UserGroup` |
| **Service Provider** | Performs work | `FormatDateUseCase`, `NewsRemoteDataSource` |
| **Coordinator** | Orchestrates workflow | `OfflineFirstNewsRepository` (local + remote sources) |
| **Controller** | Makes decisions, delegates | `NewsViewModel` (events → state) |
| **Interfacer** | Transforms between systems | Mappers (`asExternalModel()`), API/DAO wrappers |

### The Two Questions

For every class, ask:
1. **"What role is this?"** - Which stereotype? Which architectural role?
2. **"Is it doing too much?"** - Check SRP and cohesion

If you can't answer clearly, the class needs refactoring.

---

## Tell, Don't Ask

**Command objects to do work. Don't interrogate them and do the work yourself.**

```kotlin
// BAD: Asking, then doing
if (account.balance >= amount) {
    account.balance = account.balance - amount
    // more logic here...
}

// GOOD: Telling
when (val result = account.withdraw(amount)) {
    is WithdrawResult.Success -> showNewBalance(result.newBalance)
    WithdrawResult.InsufficientFunds -> showInsufficientFundsMessage()
}
```

The object that has the data should have the behavior.

---

## Design by Contract (DbC)

Every function has:
- **Preconditions** - What must be true BEFORE calling (`require`)
- **Postconditions** - What will be true AFTER calling
- **Invariants** - What is ALWAYS true about the object (`init { require(...) }`, `check`)

```kotlin
sealed interface WithdrawResult {
    data class Success(val newBalance: Money) : WithdrawResult
    data object InsufficientFunds : WithdrawResult
}

class BankAccount(initialBalance: Money) {
    // INVARIANT: balance is never negative
    var balance: Money = initialBalance
        private set

    init { require(initialBalance >= Money.ZERO) { "Balance cannot be negative" } }

    // PRECONDITION: amount > 0
    // POSTCONDITION: balance decreased by amount OR InsufficientFunds returned
    fun withdraw(amount: Money): WithdrawResult {
        require(amount > Money.ZERO) { "Amount must be positive" }
        if (balance < amount) return WithdrawResult.InsufficientFunds
        balance -= amount
        return WithdrawResult.Success(balance)
    }
}
```

Programming errors (violated preconditions) throw; expected business outcomes are modeled as results.

---

## Composition Over Inheritance

**Prefer composing objects over extending classes.** Kotlin classes are `final` by default - keep it that way unless you design for inheritance.

### Why Inheritance is Problematic:
- Tight coupling between parent and child
- Fragile base class problem
- Difficult to change parent without breaking children
- Forces "is-a" relationship that may not fit

### When to Use Inheritance:
- True "is-a" relationship (rare)
- Framework requirements (`ViewModel`, `Activity`, `CoroutineWorker`)
- Template Method pattern (intentional)
- `sealed` hierarchies modeling a closed set of cases

Avoid "base" classes (`BaseActivity`, `BaseViewModel`) that accumulate shared behavior - use composition, delegation, or extension functions instead.

### Prefer Composition:
```kotlin
// BAD: Inheritance
open class User { open val discountPercent: Int = 0 }
class PremiumUser : User() { override val discountPercent: Int = 20 }

// GOOD: Composition
fun interface DiscountPolicy {
    fun calculateDiscount(orderValue: Money): Money
}

class User(private val discountPolicy: DiscountPolicy) {
    fun discountFor(orderValue: Money): Money = discountPolicy.calculateDiscount(orderValue)
}

// Now discount behavior is pluggable
User(PremiumDiscount())
User(StandardDiscount())
User(NoDiscount)
```

Kotlin's class delegation (`class LoggingRepository(private val delegate: NewsRepository) : NewsRepository by delegate`) makes composition cheap.

---

## The Law of Demeter (Principle of Least Knowledge)

**Only talk to your immediate friends.**

A function should only call:
1. Functions on `this`
2. Functions on parameters
3. Functions on objects it creates
4. Functions on its direct components

```kotlin
// BAD: Reaching through objects
order.customer.address.city

// GOOD: Ask the immediate friend
order.shippingCity
```

This reduces coupling - changes to `Address` don't ripple through all callers. Chains of collection or Flow operators are not violations.

---

## Encapsulation

**Hide internal details, expose behavior.**

### Levels of Encapsulation:
1. **Data** - `private` properties, read-only exposure
2. **Implementation** - how things work internally
3. **Type** - concrete class hidden behind an interface
4. **Design** - architectural decisions hidden from clients (`internal` in modules)

```kotlin
// BAD: Exposed internals
class Order {
    val items = mutableListOf<OrderItem>()
    var total: Money = Money.ZERO
}

// Client can corrupt state
order.items.add(item) // total is now wrong
order.total = Money.of(-999)

// GOOD: Encapsulated
class Order {
    private val _items = mutableListOf<OrderItem>()
    val items: List<OrderItem> get() = _items

    var total: Money = Money.ZERO
        private set

    fun addItem(item: OrderItem) {
        _items += item
        total += item.price
    }
}
```

The same rule applies to ViewModels: `private val _uiState = MutableStateFlow(...)` and `val uiState: StateFlow<...>`.

---

## Polymorphism

**Replace conditionals with types.**

```kotlin
// BAD: Type checking with strings
fun calculateShipping(method: String, orderValue: Money): Money = when (method) {
    "standard" -> if (orderValue < FREE_SHIPPING_THRESHOLD) Money.of(5) else Money.ZERO
    "express" -> Money.of(15)
    "overnight" -> Money.of(25)
    else -> throw IllegalArgumentException("Unknown method")
}

// GOOD (open set): interface - new methods without touching existing code
fun interface ShippingMethod {
    fun calculateCost(orderValue: Money): Money
}

// GOOD (closed set): sealed type - the compiler enforces exhaustive handling
sealed interface Shipping {
    data object Standard : Shipping
    data object Express : Shipping
    data object Overnight : Shipping
}

fun Shipping.cost(orderValue: Money): Money = when (this) {
    Shipping.Standard -> if (orderValue < FREE_SHIPPING_THRESHOLD) Money.of(5) else Money.ZERO
    Shipping.Express -> Money.of(15)
    Shipping.Overnight -> Money.of(25)
}
```

---

## Value Objects vs Entities

### Value Objects
- Defined by their attributes (no identity)
- Immutable
- Compared by value (`data class` / `value class` give you `equals` for free)
- Examples: `Money`, `Email`, `Address`, `DateRange`

```kotlin
data class Money(val amount: BigDecimal, val currency: Currency) {
    operator fun plus(other: Money): Money {
        require(currency == other.currency) { "Currency mismatch: $currency vs ${other.currency}" }
        return copy(amount = amount + other.amount)
    }
}

@JvmInline
value class Email(val value: String) {
    init { require(EMAIL_PATTERN.matches(value)) { "Invalid email" } }
}
```

### Entities
- Have identity (survives attribute changes)
- Compared by identity
- Examples: `User`, `Order`, `Product`
- In Kotlin, entities are often immutable `data class`es updated with `copy()`, keyed by their ID

```kotlin
data class User(
    val id: UserId,
    val email: Email,
    val name: String,
) {
    fun changeEmail(newEmail: Email): User = copy(email = newEmail) // still the same user (same id)
}
```

---

## Aggregates

A cluster of objects treated as a single unit for data changes.

- One object is the **aggregate root** (entry point)
- External code only references the root
- Root enforces invariants for the entire cluster

```kotlin
// Order is the aggregate root
class Order(val id: OrderId) {
    private val _items = mutableListOf<OrderItem>()
    val items: List<OrderItem> get() = _items

    // All changes go through the root
    fun addItem(product: Product, quantity: Int) {
        val newItem = OrderItem(product, quantity)
        require(calculateTotal() + newItem.subtotal <= MAX_ORDER_VALUE) { "Order total exceeded" }
        _items += newItem
    }

    fun removeItem(itemId: ItemId) {
        _items.removeAll { item -> item.id == itemId }
    }

    fun calculateTotal(): Money = _items.fold(Money.ZERO) { sum, item -> sum + item.subtotal }
}

// Callers can't bypass validation: `items` is read-only
order.addItem(product, quantity = 2)
```
