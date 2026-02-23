# Java Exception Handling

## Core Rules
- Never swallow exceptions silently — always log or rethrow
- Never catch `Exception` or `Throwable` unless at the top-level boundary
- Use specific exception types

```java
// ❌ Wrong
try { ... } catch (Exception e) { }

// ✅ Correct
try { ... } catch (IOException e) {
    log.error("Failed to read file: {}", path, e);
    throw new FileProcessingException("Read failed", e);
}
```

## Try-Catch Scope — CRITICAL
**One try-catch block per logical operation. Do NOT wrap each line individually.**

```java
// ❌ Wrong — granular try-catch per statement
try { user = repo.findById(id); } catch (Exception e) { ... }
try { order = orderRepo.findByUser(user); } catch (Exception e) { ... }
try { invoice = invoiceService.generate(order); } catch (Exception e) { ... }

// ✅ Correct — one block covering the logical unit of work
try {
    User user = repo.findById(id);
    Order order = orderRepo.findByUser(user);
    Invoice invoice = invoiceService.generate(order);
} catch (UserNotFoundException e) {
    log.warn("User not found: id={}", id, e);
    throw e;
} catch (InvoiceGenerationException e) {
    log.error("Invoice generation failed: userId={}", id, e);
    throw e;
}
```

**Rules:**
- Wrap the entire logical operation in one try block
- Use multiple `catch` clauses for different exception types
- Use multi-catch (`|`) when handling multiple exceptions the same way: `catch (IOException | SQLException e)`

## Custom Exceptions
- Extend `RuntimeException` for unchecked (preferred)
- Extend `Exception` for checked only when caller MUST handle it
- Include context in message: `"User not found: id=" + userId`

## Exception Hierarchy
```
AppException (base)
├── NotFoundException
├── ValidationException
└── ServiceException
```

## Try-With-Resources
- Always use for `Closeable` resources (streams, connections)

```java
try (InputStream is = new FileInputStream(file)) {
    // use is
}
```

## Anti-Patterns to Avoid
- No `printStackTrace()` — use logger
- No `return null` in catch — throw or return `Optional`
- No exception for flow control (e.g., checking existence)
