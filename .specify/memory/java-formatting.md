# Java Code Formatting & Style

## Indentation & Spacing
- 4 spaces per indent (no tabs)
- Max line length: 120 characters
- One blank line between methods; two blank lines between classes

## Braces
- Opening brace on same line (K&R style)
- Always use braces even for single-line `if`/`for`/`while`

```java
// ✅ Correct
if (condition) {
    doSomething();
}

// ❌ Wrong
if (condition) doSomething();
```

## Imports
- No wildcard imports (`import java.util.*` is forbidden)
- Order: static imports → java.* → javax.* → third-party → internal
- Remove unused imports

## Class Structure Order
1. Static constants
2. Instance fields
3. Constructors
4. Public methods
5. Private/protected methods
6. Inner classes

## Annotations
- One annotation per line
- `@Override` always required when overriding

## Formatting Tool
- Use **Google Java Format** or **Checkstyle** with project config
- Run formatter before every commit
