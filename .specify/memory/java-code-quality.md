# Java Code Quality & SonarQube Standards

## SonarQube — Quality Gate is MANDATORY

**No code is complete until it passes the SonarQube Quality Gate.**
Every file written or edited must be verified before the PR is raised.

---

## VS Code — SonarLint Connected Mode

When SonarQube is connected to VS Code via SonarLint Connected Mode:

1. **After writing or editing any file** — check the SonarLint panel for issues immediately
2. **Fix all issues before moving on** — do not defer Sonar issues to later
3. **Connected mode syncs your project's Quality Gate rules** — local feedback matches CI results
4. **Never suppress a Sonar issue without team approval** — no `@SuppressWarnings("squid:...")` without a documented reason

### Workflow
```
Write / Edit file
      ↓
Check SonarLint panel in VS Code (Problems tab + SonarLint tab)
      ↓
Fix all BLOCKER and CRITICAL issues
      ↓
Fix all MAJOR issues
      ↓
Commit only when panel is clean
```

---

## Quality Gate Thresholds (Non-Negotiable)

| Metric | Minimum Required |
|--------|-----------------|
| Line Coverage | **80%** (new code) |
| Duplicated Lines | < 3% |
| Maintainability Rating | **A** |
| Reliability Rating | **A** (zero bugs) |
| Security Rating | **A** (zero vulnerabilities) |
| Code Smells | Zero new smells on changed code |

---

## Common Sonar Issues to Prevent

### Cognitive Complexity
- Keep method complexity below **15**
- Extract complex conditionals into well-named private methods

```java
// ❌ High complexity — Sonar will flag this
public void process(Order order) {
    if (order != null) {
        if (order.getStatus() == OrderStatus.PENDING) {
            if (order.getUser() != null && order.getUser().isActive()) {
                // nested logic...
            }
        }
    }
}

// ✅ Extracted — lower complexity, readable
public void process(Order order) {
    if (!isEligibleForProcessing(order)) return;
    processEligibleOrder(order);
}

private boolean isEligibleForProcessing(Order order) {
    return order != null
        && order.getStatus() == OrderStatus.PENDING
        && order.getUser() != null
        && order.getUser().isActive();
}
```

### Duplication
- Sonar flags blocks of 10+ duplicated lines
- Extract to a `util` class or shared private method immediately
- See `java-reusability.md` for full rules

### Security Hotspots
- Always review and mark hotspots as `Safe` or fix them — never leave them unreviewed
- Common: hardcoded credentials, weak random, SQL injection risk, insecure deserialization

---

## Sonar Issue Severity Reference

| Severity | Action Required |
|----------|----------------|
| BLOCKER | Fix immediately — blocks merge |
| CRITICAL | Fix before PR |
| MAJOR | Fix before PR |
| MINOR | Fix if in scope, log if not |
| INFO | Fix when convenient |
