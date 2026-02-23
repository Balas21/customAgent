# Java Constants & Magic Values

## Rule: No Magic Numbers or Magic Strings — EVER

Magic values are hardcoded literals (`42`, `"ACTIVE"`, `"admin"`) scattered in code.
They break readability, cause bugs on change, and are forbidden in this project.

```java
// ❌ Wrong — magic number and magic string
if (user.getAge() > 18 && user.getRole().equals("ADMIN")) { ... }
Thread.sleep(5000);
int[] scores = new int[100];

// ✅ Correct — named constants
if (user.getAge() > UserConstants.MIN_ADULT_AGE && user.getRole() == Role.ADMIN) { ... }
Thread.sleep(RetryConstants.DEFAULT_RETRY_DELAY_MS);
int[] scores = new int[AppConstants.MAX_SCORE_COUNT];
```

## Exception: Log Message Strings

**Do NOT extract log message text into constants.** Log statements are narrative descriptions — they are exempt from the magic string rule.

```java
// ❌ Wrong — over-engineering log messages into constants
public static final String ERR_USER_NOT_FOUND = "User not found for id={}";
log.error(UserConstants.ERR_USER_NOT_FOUND, userId);

// ✅ Correct — log messages stay inline, always
log.error("User not found for id={}", userId);
log.warn("Retry attempt {} of {} for orderId={}", attempt, RetryConstants.MAX_RETRIES, orderId);
```

**The rule applies to:** values used in logic, comparisons, conditions, config, and sizing.
**The rule does NOT apply to:** log message text, exception message text, comment strings.

---

## Where Constants Live

### Project-wide constants → `constants/` package
Create one constants class per domain area. Never dump everything into one `Constants.java`.

```
com.company.project.constants/
├── AppConstants.java       ← app-level limits, timeouts, defaults
├── UserConstants.java      ← user domain constants
├── OrderConstants.java     ← order domain constants
└── ValidationConstants.java← regex patterns, field lengths
```

### Class-scoped constants → top of the class
If a constant is only used in one class, declare it `private static final` at the top of that class — don't move it to a shared constants file unnecessarily.

```java
public class PasswordEncoder {
    private static final int BCRYPT_STRENGTH = 12;
    private static final int MIN_PASSWORD_LENGTH = 8;
}
```

---

## Constants Class Pattern

```java
// ✅ Correct — final class, private constructor, grouped logically
public final class UserConstants {
    private UserConstants() {} // prevent instantiation

    public static final int MIN_ADULT_AGE = 18;
    public static final int MAX_USERNAME_LENGTH = 50;
    public static final int SESSION_TIMEOUT_MINUTES = 30;
    public static final String DEFAULT_LOCALE = "en-US";
}
```

---

## When to Use ENUM Instead of Constants

If a constant represents **one of a fixed set of values**, use an **Enum**, not a String or int constant.

```java
// ❌ Wrong — string constants for states
public static final String STATUS_ACTIVE = "ACTIVE";
public static final String STATUS_INACTIVE = "INACTIVE";
public static final String STATUS_PENDING = "PENDING";

// ✅ Correct — Enum
public enum UserStatus {
    ACTIVE, INACTIVE, PENDING
}
```

**Use Enum when:**
- The value is one of N known options (status, type, role, category)
- You switch/branch on the value
- The value has associated behavior or metadata

**Use constant when:**
- The value is a scalar limit, threshold, or configuration (`MAX_RETRIES = 3`)
- The value is a fixed string key (HTTP header name, config property key)

---

## Reuse Existing Enums

**Before creating a new Enum or constant, check if one already exists in the codebase.**

```java
// ❌ Wrong — duplicating an existing enum
public static final String HTTP_GET = "GET";

// ✅ Correct — use existing
import org.springframework.http.HttpMethod;
HttpMethod.GET

// ✅ Correct — reuse project enum
OrderStatus.PENDING  // not "PENDING" string
```

**Always search for:** existing enums in `enums/` or `model/` packages before declaring new ones.
If an existing enum is missing a value you need, **add to it** rather than creating a parallel constant.
