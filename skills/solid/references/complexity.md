# Managing Complexity

## The Two Types of Complexity

### Essential Complexity
Inherent to the problem domain. Cannot be removed, only managed.
- Business rules
- Domain logic
- User requirements
- Platform realities (lifecycle, process death, offline)

### Accidental Complexity
Introduced by our solutions. CAN and SHOULD be minimized.
- Poor abstractions
- Unnecessary indirection (use cases that only forward calls, interfaces with one implementation and no test need)
- Framework ceremony
- Clever code that hides intent
- Technical debt

**Goal: Minimize accidental complexity while clearly expressing essential complexity.**

---

## Detecting Complexity

### 1. Change Amplification
Small changes require touching many files.

**Symptom:** "To add this field, I need to update 15 files."

**Cause:** Scattered responsibilities, poor abstraction boundaries.

### 2. Cognitive Load
Code is hard to understand, requires holding too much in memory.

**Symptom:** "I need to understand 10 other classes to understand this one."

**Cause:** Tight coupling, hidden dependencies, unclear naming, implicit behavior.

### 3. Unknown Unknowns
Behavior is surprising, side effects are hidden.

**Symptom:** "I changed this, and something completely unrelated broke."

**Cause:** Global state, hidden dependencies, implicit contracts.

---

## The XP Values for Fighting Complexity

From Extreme Programming:

### 1. Communication
Code should communicate clearly. Names, structure, tests all contribute.

### 2. Simplicity
Do the simplest thing that could possibly work.

### 3. Feedback
Fast feedback loops catch complexity early. TDD, CI, code review.

### 4. Courage
Refactor the code you touch without fear - tests make it safe. Don't let complexity accumulate in your path.

### 5. Respect
Respect future readers, human or agent (including yourself). Both read the code with equal weight.

---

## KISS - Keep It Simple, Silly

> "The simplest solution that works is usually the best."

### How to Apply:
1. Start with the obvious solution
2. Only add complexity when REQUIRED
3. Prefer boring, well-understood approaches
4. Question every abstraction

```kotlin
// Over-engineered
abstract class BaseUseCase<in P, out R> {
    abstract suspend fun execute(params: P): R
}
class GetUserUseCaseFactoryProvider { /* ... */ }

// KISS
class GetUserUseCase(private val userRepository: UserRepository) {
    suspend operator fun invoke(id: UserId): User = userRepository.getUser(id)
}
```

The domain layer itself is optional - add use cases only when they add value.

---

## YAGNI - You Aren't Gonna Need It

> "Don't build features until they're actually needed."

### Warning Signs:
- "We might need this later"
- "It would be nice to have"
- "Just in case"
- "For future extensibility"

### The Cost of YAGNI Violations:
1. **Development time** - Building unused features
2. **Maintenance burden** - Code that must be maintained
3. **Cognitive load** - More to understand
4. **Wrong abstraction** - Guessing future needs incorrectly

```kotlin
// YAGNI violation: Building for hypothetical needs
data class User(
    val name: String,
    val email: Email,
    // "We might need these someday"
    val middleName: String? = null,
    val secondaryEmail: Email? = null,
    val faxNumber: String? = null,
    val linkedinProfile: String? = null,
)

// YAGNI: Only what's needed NOW
data class User(
    val name: String,
    val email: Email,
)
```

---

## DRY - Don't Repeat Yourself (with The Rule of Three)

> "Every piece of knowledge should have a single, unambiguous representation."

### BUT: The Rule of Three

**Don't extract duplication until you see it THREE times.**

Why? The wrong abstraction is worse than duplication.

```
Duplication #1 → Leave it
Duplication #2 → Note it, leave it
Duplication #3 → NOW extract it
```

### Example:
```kotlin
// First time - leave it
suspend fun placeUserOrder(order: Order) {
    validate(order)
    applyTax(order)
    orderRepository.saveOrder(order)
}

// Second time - note the similarity, but leave it
suspend fun placeGuestOrder(order: Order) {
    validate(order)
    applyTax(order)
    orderRepository.saveOrder(order)
    sendGuestEmail(order)
}

// Third time - NOW extract
suspend fun placeCorporateOrder(order: Order) {
    validate(order)
    applyTax(order)
    orderRepository.saveOrder(order)
    applyCorporateDiscount(order)
}

// After three, extract the common parts
suspend fun placeOrder(order: Order, afterSaving: suspend (Order) -> Unit = {}) {
    validate(order)
    applyTax(order)
    orderRepository.saveOrder(order)
    afterSaving(order)
}
```

---

## Separation of Concerns

> "Each module should address a single concern."

### Concerns to Separate:
- **Business logic** vs **Infrastructure** (database, network, SDKs)
- **UI** vs **State production** (composables/Views vs ViewModels)
- **What** (policy) vs **How** (mechanism)
- **Input** vs **Processing** vs **Output**

### Example:
```kotlin
// BAD: Mixed concerns in a ViewModel
class CheckoutViewModel(private val context: Context) : ViewModel() {
    fun checkout(order: Order) {
        viewModelScope.launch(Dispatchers.IO) {
            require(order.items.isNotEmpty())                                  // validation
            val total = order.items.sumOf { item -> item.price * item.quantity } // business logic
            AppDatabase.getInstance(context).orderDao().insert(order.toEntity()) // persistence
            NotificationManagerCompat.from(context).notify(/* ... */)            // notification
        }
    }
}

// GOOD: Separated concerns
class PlaceOrderUseCase(
    private val orderValidator: OrderValidator,
    private val orderRepository: OrderRepository,
    private val orderNotifier: OrderNotifier,
) {
    suspend operator fun invoke(order: Order): PlaceOrderResult {
        orderValidator.validate(order)
        val savedOrder = orderRepository.saveOrder(order)
        orderNotifier.notifyConfirmation(savedOrder)
        return PlaceOrderResult.Success(savedOrder)
    }
}

class CheckoutViewModel(private val placeOrder: PlaceOrderUseCase) : ViewModel() {
    // turns the result into UI state
}
```

---

## Managing Technical Debt

### Types of Technical Debt:
1. **Deliberate** - Conscious trade-off for speed
2. **Accidental** - Mistakes, lack of knowledge
3. **Bit rot** - Code degrades over time

### The Boy Scout Rule:
> "Leave the code better than you found it."

Inside the code the task already touches:
- Improve one small thing
- Fix one naming issue
- Extract one function
- Add one missing test

Outside that code, report what you'd improve instead of changing it. Improving code is not the same as replacing its libraries - don't migrate libraries as a "cleanup" unless the user asks.

### When to Pay Down Debt:
- When it's in your path (you're already there)
- When it's blocking new features
- When it's causing bugs
- During dedicated refactoring time

### When NOT to Refactor:
- Code that works and won't change
- Code being replaced soon
- When you don't have tests

---

## The Four Elements of Simple Design

In priority order (from XP):

1. **Runs all the tests**
   - If it doesn't work, nothing else matters

2. **Expresses intent**
   - Clear names, obvious structure
   - Code tells the story

3. **No duplication**
   - DRY (but Rule of Three)
   - Single source of truth

4. **Minimal**
   - Fewest classes and functions possible
   - Remove anything unnecessary

If these four are true, the design is simple enough.
