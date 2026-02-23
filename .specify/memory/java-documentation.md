# Java Documentation — Javadoc & LLM-Aware Standards

> **For AI/Copilot:** When generating or editing any class or method, apply every rule in this file.
> Good documentation is not optional — it is how the LLM understands intent without reading implementation.
> When in doubt, **over-document rather than under-document**.

---

## Javadoc Must Stay in Sync — NON-NEGOTIABLE

**Whenever a method is edited, its Javadoc MUST be reviewed and updated in the same change.**
Stale Javadoc is worse than no Javadoc — it actively misleads both humans and LLMs.

**This applies to:**
- Changed parameters (added, removed, renamed, type changed)
- Changed return type or return behavior
- Changed exceptions thrown
- Changed business logic or side effects
- Changed preconditions or postconditions

```java
// ❌ Wrong — method was changed but Javadoc still describes old behavior
/**
 * @param userId the user ID
 * @return the user's email   ← STALE: method now returns full User object
 */
public User getUser(Long userId) { ... }

// ✅ Correct — Javadoc reflects current method exactly
/**
 * @param userId the user ID (must be positive)
 * @return the full User object including roles and preferences
 * @throws NotFoundException if no user exists with the given ID
 */
public User getUser(Long userId) { ... }
```

---

## LLM-Aware Javadoc

Write Javadoc so an LLM can **fully reconstruct the intent, contract, and side effects** of a method without reading the implementation. If the LLM has to open the method body to understand what it does, the Javadoc is incomplete.

### What Every Method Javadoc Must Answer

| Question | Tag | Example |
|----------|-----|---------|
| What does it do in business terms? | First sentence | `Creates a new order and reserves inventory.` |
| What does each parameter mean in the domain? | `@param` | `customerId – the authenticated customer placing the order (must be active)` |
| What does the return value mean? | `@return` | `the persisted Order with generated ID and PENDING status` |
| What can go wrong and when? | `@throws` | `DuplicateOrderException – if an identical order was placed within the last 5 minutes` |
| Does it write to the DB, cache, or publish events? | `<p>Side effects:` | See template below |
| Are there constraints a caller must satisfy? | `<p>Preconditions:` | `order.items must not be empty` |

### Method Javadoc Template

```java
/**
 * [One sentence: what it does in business terms — not implementation terms.]
 *
 * <p>[Optional: 1-2 sentences of context, business rules applied, or state transitions.]
 *
 * <p>Side effects:
 * <ul>
 *   <li>Persists [entity] to [table/repository]</li>
 *   <li>Publishes {@code [EventName]} to topic {@code [topic-name]}</li>
 *   <li>Evicts [cache-name] cache entry for [key]</li>
 * </ul>
 *
 * @param [name]  [business meaning + constraints, e.g., "must be a positive, existing order ID"]
 * @return [what the value means in domain terms, not just its Java type]
 * @throws [ExceptionType]  [exact condition that triggers it]
 * @throws [ExceptionType]  [exact condition that triggers it]
 */
```

### Full Example

```java
/**
 * Creates a new order for the given customer and publishes an OrderCreated event.
 *
 * <p>Validates that no identical order (same customer, same items) was placed
 * within the last 5 minutes. Persists the order in PENDING status and
 * reserves inventory for all line items.
 *
 * <p>Side effects:
 * <ul>
 *   <li>Persists Order and OrderItems to the database</li>
 *   <li>Publishes {@code OrderCreatedEvent} to topic {@code food-mela.order.created}</li>
 * </ul>
 *
 * @param request  the order details including customer ID and line items (must not be empty)
 * @return the persisted {@link Order} with generated ID and {@link OrderStatus#PENDING} status
 * @throws DuplicateOrderException  if an identical order was placed within the last 5 minutes
 * @throws CustomerNotFoundException  if no active customer exists for the given customer ID
 * @throws InsufficientInventoryException  if any line item cannot be fulfilled
 */
public Order createOrder(OrderRequestDto request) { ... }
```

---

## Inline Comment Conventions — LLM Signals

Use these **structured inline comment prefixes** to signal intent to both humans and LLMs. These are not general comments — they mark decision points that the LLM must not silently change.

### Required Patterns

```java
// BUSINESS RULE: Orders placed within 5 minutes with the same customerId + items are duplicates
if (isDuplicate(request)) throw new DuplicateOrderException(request.customerId());

// INVARIANT: Order status must be PENDING before it can be confirmed
if (order.getStatus() != OrderStatus.PENDING) throw new InvalidStateException(...);

// INTENTIONAL: Empty list is valid here — downstream handles it gracefully
if (items.isEmpty()) return Collections.emptyList();

// TODO(ticket): SCRUM-42 — replace with async call once NotificationService is available
notificationService.sendSync(customerId, message);

// SECURITY: Never log the full payload here — it may contain PII
log.info("Order created: orderId={}", order.getId());
```

| Prefix | When to use |
|--------|-------------|
| `// BUSINESS RULE:` | Non-obvious domain logic that must not be changed without BA sign-off |
| `// INVARIANT:` | State constraint that must always hold — a violation is a bug |
| `// INTENTIONAL:` | Code that looks wrong but is deliberate — prevents future "fix" regressions |
| `// TODO(ticket):` | Deferred work — must reference a ticket, never a bare `// TODO` |
| `// SECURITY:` | Security-sensitive line — changes need security review |

---

## Class-Level Javadoc Template

```java
/**
 * [One sentence: the bounded-context role of this class.]
 *
 * <p>[What it is responsible for. What it is NOT responsible for (boundaries).]
 * <p>[Thread-safety note if relevant. Transactional boundary if applicable.]
 *
 * @see [RelatedClass or Interface]
 * @since [version when introduced]
 */
```

### Full Example

```java
/**
 * Manages the full lifecycle of food orders from creation to delivery.
 *
 * <p>Responsible for: creating orders, updating status, cancellation, and duplicate detection.
 * NOT responsible for: payment processing (see PaymentService) or delivery tracking.
 *
 * <p>All public methods are transactional. Events are published after commit.
 *
 * @see OrderRepository
 * @see OrderCreatedEvent
 * @since 1.0.0
 */
@Service
public class OrderServiceImpl implements OrderService { ... }
```

---

## package-info.java — Required for Every Package

Every package must have a `package-info.java` so LLMs understand the package's role without navigating all its files.

```java
/**
 * Order domain services.
 *
 * <p>Contains all business logic for the order bounded context.
 * Classes in this package depend on {@code repository} and publish
 * events to the {@code events} package. No web/controller dependencies.
 */
package com.foodmela.orderservice.service;
```

---

## Documenting Kafka Events

Any method that publishes a Kafka event **must** document it explicitly in both the `<p>Side effects:` block and the `@see` tag.

```java
/**
 * Cancels a PENDING or CONFIRMED order.
 *
 * <p>Side effects:
 * <ul>
 *   <li>Updates Order status to {@link OrderStatus#CANCELLED}</li>
 *   <li>Publishes {@code OrderCancelledEvent} to topic {@code food-mela.order.cancelled}</li>
 * </ul>
 *
 * @param orderId  the ID of the order to cancel
 * @throws OrderNotFoundException  if no order exists with that ID
 * @throws InvalidStateException   if the order is already DELIVERED or CANCELLED
 * @see OrderCancelledEvent
 */
public void cancelOrder(UUID orderId) { ... }
```

---

## When to Write Javadoc

| Target | Required? |
|--------|-----------|
| `public` class / interface / enum | Yes — always |
| `public` / `protected` method | Yes — always |
| `@RestController` endpoint method | Yes — `@Operation` (Swagger) + Javadoc |
| Constants and non-obvious fields | Yes |
| `private` helper methods > 5 lines | Recommended |
| Trivial getters/setters with no logic | No |

---

## Tags Reference

| Tag | Usage |
|-----|-------|
| `@param` | Each parameter with **business meaning and constraints**, not just its type |
| `@return` | What the value represents in domain terms (skip for `void`) |
| `@throws` | Each exception and the **exact condition** that triggers it |
| `@since` | Version when introduced |
| `@deprecated` | Include: since which version, why deprecated, and exact replacement |
| `@see` | Related class/method/event links |
| `{@link}` | Reference to a Java type or enum constant inline |
| `{@code}` | Inline code snippet, topic names, field names |

---

## Anti-Patterns

- No Javadoc that just restates the method name (`getUserById` → "Gets the user by ID")
- No stale `@param` or `@return` that no longer match the actual signature
- No bare `// TODO` without a ticket reference
- No `@author` per method — class-level only
- No commented-out dead code — delete it, it's in git
- No `// BUSINESS RULE:` on obvious Java mechanics — reserve it for domain decisions only
- Use `{@code topic-name}` for Kafka topics, not plain backtick strings in Javadoc
