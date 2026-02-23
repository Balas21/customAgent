# Javadoc Verification Process

This file defines how the Documentation Agent verifies Javadoc accuracy before using it in METHOD_SUMMARY.md.

---

## Verification Steps

For EVERY method in EVERY class, perform this verification process:

### Step 1: Extract Javadoc
Read the Javadoc comment block (`/** ... */`) above the method.

### Step 2: Extract Actual Signature
Parse the real method signature: return type, name, parameters, annotations, thrown exceptions.

### Step 3: Cross-Verify
Compare Javadoc against actual code:

| Check | What to Verify | Status |
|-------|---------------|--------|
| **@param tags** | Every parameter in signature has a matching @param. No extra @param tags for removed params. | ✅/❌ |
| **@return tag** | Return type in Javadoc matches actual return type. Void methods have no @return. | ✅/❌ |
| **@throws tags** | Every exception the method can throw (directly or via called methods) has a @throws tag. No stale @throws for exceptions no longer thrown. | ✅/❌ |
| **Description accuracy** | Javadoc description matches what the method actually does (check method body logic). | ✅/❌ |
| **Business rules** | Documented business rules in Javadoc match actual validation/logic in method body. | ✅/❌ |
| **Annotations match** | If Javadoc mentions HTTP method/path, it matches actual @GetMapping/@PostMapping etc. | ✅/❌ |

### Step 4: Flag Issues
For each mismatch, generate a finding:

```
⚠️ JAVADOC STALE: UserService.registerUser()
  - @throws DuplicatePhoneException — EXISTS in code ✅
  - @param request — matches signature ✅  
  - @return UserResponse — matches signature ✅
  - Business Rule "Password should be encrypted" — NOT implemented (TODO in code) ⚠️
  - Missing @throws: None ✅
```

### Step 5: Generate Verified Summary
Only AFTER verification, generate the method summary:
- **Javadoc is accurate** → use Javadoc text as summary source
- **Javadoc is stale/missing** → generate summary from actual code and mark with `[GENERATED FROM CODE - Javadoc outdated]`
- **Javadoc is missing entirely** → generate from code and mark with `[NO JAVADOC - Generated from code analysis]`

### Step 6: Produce Javadoc Health Report
Create a verification summary at the top of METHOD_SUMMARY.md:

```
📋 Javadoc Verification Report
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total Methods Scanned: {count}
✅ Javadoc Up-to-Date: {count} ({percent}%)
⚠️ Javadoc Stale/Partial: {count} ({percent}%)
❌ Javadoc Missing: {count} ({percent}%)
```

---

## Summary Source Priority

| Priority | Condition | Summary Source | Marker |
|----------|-----------|---------------|--------|
| 1 | Javadoc exists AND all checks pass | Javadoc text | ✅ Verified |
| 2 | Javadoc exists BUT some checks fail | Code analysis + Javadoc context | ⚠️ `[GENERATED FROM CODE - Javadoc outdated]` |
| 3 | No Javadoc present | Code analysis only | ❌ `[NO JAVADOC - Generated from code analysis]` |

---

## What Counts as "Stale"

A Javadoc is marked **stale** if ANY of these are true:
- A `@param` tag references a parameter that no longer exists
- A `@param` tag is missing for a parameter that exists
- `@return` describes a different type than the actual return
- `@throws` lists an exception the method can no longer throw
- A thrown exception has no corresponding `@throws`
- The description describes behavior that doesn't match the method body
- Documented business rules don't match actual validation logic
- HTTP method/path in Javadoc doesn't match actual annotation
