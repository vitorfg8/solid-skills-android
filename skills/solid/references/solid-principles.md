# SOLID Principles

## Overview

SOLID helps structure software to be flexible, maintainable, and testable. These principles reduce coupling and increase cohesion. On Android they map directly onto the official architecture: ViewModels, use cases, repositories, and data sources each with one job, connected through abstractions.

## S - Single Responsibility Principle (SRP)

> "A class should have one, and only one, reason to change."

### Problem It Solves
God objects that do everything - hard to test, hard to change, hard to understand. On Android the classic example is the **God Activity/Fragment** or a ViewModel that talks to the database, the network, and formats text.

### How to Apply
Each class handles ONE responsibility. If you find yourself saying "and" when describing what a class does, split it.

```kotlin
// BAD: Multiple responsibilities
class OrderViewModel(private val database: AppDatabase, private val api: OrderApi) : ViewModel() {
    fun calculateTotal(items: List<OrderItem>): Money { /* business rule */ }
    fun saveOrder(order: Order) { /* persistence + network sync */ }
    fun formatInvoice(order: Order): String { /* presentation */ }
}

// GOOD: Single responsibility each
data class Order(val id: OrderId, val items: List<OrderItem>) {
    val total: Money get() = items.fold(Money.ZERO) { sum, item -> sum + item.price }
}

interface OrderRepository {
    suspend fun saveOrder(order: Order)
}

class PlaceOrderUseCase(private val orderRepository: OrderRepository) {
    suspend operator fun invoke(order: Order) = orderRepository.saveOrder(order)
}

class OrderViewModel(private val placeOrder: PlaceOrderUseCase) : ViewModel() {
    // turns user events into UI state - nothing else
}
```

### Detection Questions
- Does this class have multiple reasons to change?
- Can I describe it without using "and"?
- Would different stakeholders request changes to different parts?
- Does this Activity/Fragment contain logic beyond rendering and forwarding events?

---

## O - Open/Closed Principle (OCP)

> "Software entities should be open for extension but closed for modification."

### Problem It Solves
Having to modify existing, tested code every time requirements change. Risk of breaking working features.

### How to Apply
Design abstractions that allow new behavior through new types, not edits to existing ones.

```kotlin
// BAD: Must modify to add new shipping
class ShippingCalculator {
    fun calculate(type: String, orderValue: Money): Money = when (type) {
        "standard" -> if (orderValue < FREE_SHIPPING_THRESHOLD) Money.of(5) else Money.ZERO
        "express" -> Money.of(15)
        else -> throw IllegalArgumentException("Unknown type: $type") // must add more branches!
    }
}

// GOOD: Open for extension
fun interface ShippingMethod {
    fun calculateCost(orderValue: Money): Money
}

class StandardShipping : ShippingMethod {
    override fun calculateCost(orderValue: Money): Money =
        if (orderValue < FREE_SHIPPING_THRESHOLD) Money.of(5) else Money.ZERO
}

class ExpressShipping : ShippingMethod {
    override fun calculateCost(orderValue: Money): Money = Money.of(15)
}

// Add new shipping by creating a new class, not modifying existing ones
class SameDayShipping : ShippingMethod {
    override fun calculateCost(orderValue: Money): Money = Money.of(25)
}
```

**`sealed` vs `interface`:** a `sealed` hierarchy with an exhaustive `when` is the right choice when the set of cases is closed and owned by you (UI states, results) - the compiler then lists every place to update. Use an `interface` when new implementations should be addable without touching existing code.

### Architectural Insight
OCP at architecture level means: **design your codebase so new features are added by adding code (a new feature module, a new data source), not changing existing code.**

---

## L - Liskov Substitution Principle (LSP)

> "Subtypes must be substitutable for their base types without altering program correctness."

### Problem It Solves
Implementations that break expectations, requiring type checks and special cases.

### How to Apply
Implementations must honor the contract of the abstraction. If the contract says "returns a non-negative discount", no implementation may return a negative one.

```kotlin
// BAD: Violates the contract
open class DiscountPolicy {
    open fun getDiscount(orderValue: Money): Money = Money.ZERO // non-negative expected
}

class WeirdDiscount : DiscountPolicy() {
    override fun getDiscount(orderValue: Money): Money = Money.of(-5) // increases cost!
}

// GOOD: The contract is enforced
class DiscountPolicy(private val discount: Money) {
    init { require(discount >= Money.ZERO) { "Discount must be non-negative" } }

    fun getDiscount(): Money = discount
}
```

### Key Insight
This is why you can swap `OfflineFirstNewsRepository` for `FakeNewsRepository` in tests - both honor the `NewsRepository` contract. Contract tests (see [testing.md](testing.md)) prove it.

---

## I - Interface Segregation Principle (ISP)

> "Clients should not be forced to depend on methods they do not use."

### Problem It Solves
Fat interfaces that force partial implementations, empty methods, or `TODO()`s.

### How to Apply
Split large interfaces into smaller, cohesive ones. Clients depend only on what they need.

```kotlin
// BAD: Fat interface
interface MediaDevice {
    fun playAudio(track: Track)
    fun recordVideo(): Video
    fun castToTv(media: Media)
}

class BasicSpeaker : MediaDevice {
    override fun playAudio(track: Track) { /* works */ }
    override fun recordVideo(): Video = throw UnsupportedOperationException() // forced!
    override fun castToTv(media: Media) = throw UnsupportedOperationException()
}

// GOOD: Segregated interfaces
interface AudioPlayer { fun playAudio(track: Track) }
interface VideoRecorder { fun recordVideo(): Video }
interface TvCaster { fun castToTv(media: Media) }

class BasicSpeaker : AudioPlayer {
    override fun playAudio(track: Track) { /* only what it does */ }
}
```

On Android this also applies to repositories: a screen that only reads user settings shouldn't depend on a repository interface with 20 write methods.

### Detection
If you see `TODO()`, `throw UnsupportedOperationException()`, or empty overrides, the interface is too fat.

---

## D - Dependency Inversion Principle (DIP)

> "High-level modules should not depend on low-level modules. Both should depend on abstractions."

### Problem It Solves
Tight coupling to specific implementations (database, HTTP client, SDKs, Android framework). Hard to test, hard to swap.

### How to Apply
Receive dependencies through the **constructor**. Add an interface where implementations vary (offline-first vs in-memory, real gateway vs fake) or tests need a fake - not for every class. A class with one implementation and no test double can be injected as a concrete class; extract the interface when the second implementation appears.

```kotlin
// BAD: Creates its own concrete dependency
class CheckoutViewModel : ViewModel() {
    private val paymentGateway = StripePaymentGateway() // locked in, untestable

    fun pay(amount: Money) {
        viewModelScope.launch { paymentGateway.charge(amount) }
    }
}

// GOOD: Depends on an abstraction, injected through the constructor
interface PaymentGateway {
    suspend fun charge(amount: Money): ChargeResult
}

class CheckoutViewModel(
    private val paymentGateway: PaymentGateway,
) : ViewModel() {
    fun pay(amount: Money) {
        viewModelScope.launch { paymentGateway.charge(amount) }
    }
}
```

The class doesn't know **how** it's built. The same class works with any DI approach - use the one the project already has (Hilt, Koin, manual DI); for a new project, ask the user:

```kotlin
// Manual DI
CheckoutViewModel(paymentGateway = StripePaymentGateway(httpClient))

// Tests
CheckoutViewModel(paymentGateway = FakePaymentGateway())
```

Inject dispatchers and coroutine scopes the same way (see [coroutines-flow.md](coroutines-flow.md)).

### The Dependency Rule
Source code dependencies point from the UI toward the data layer, and from details toward abstractions:

```
UI layer  →  Domain layer (optional)  →  Data layer
(Compose/Views, ViewModels)   (use cases)    (repositories → data sources)

The data layer never depends on the UI layer.
ViewModels and use cases never depend on data sources directly.
```

---

## Applying SOLID at Architecture Level

| Principle | Android Architecture Application |
|-----------|----------------------------------|
| SRP | Each module/feature has one responsibility; each layer has one job |
| OCP | New features = new feature modules, not edits to existing ones |
| LSP | Repository implementations (offline-first, in-memory, fake) are interchangeable |
| ISP | Small repository/data source interfaces tailored to their clients |
| DIP | ViewModels and use cases don't know about databases, HTTP clients, or SDKs |

---

## Quick Reference

| Principle | One-Liner | Red Flag |
|-----------|-----------|----------|
| SRP | One reason to change | "This class handles X and Y and Z"; logic in Activities |
| OCP | Add, don't modify | `when`/`if` chains on type strings |
| LSP | Subtypes are substitutable | Type checks (`is`) in calling code |
| ISP | Small, focused interfaces | `TODO()` / empty overrides |
| DIP | Constructor injection; interfaces where implementations vary | Dependencies instantiated inside ViewModels/use cases |
