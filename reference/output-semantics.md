# Output Semantics — Backward Tracing Methodology

This document defines how to judge whether an output field's value is authorized for the current user. Applied by A04 (Backward Output Tracing).

---

## Termination Judgment Matrix

When tracing a value V backward to its source, apply these rules IN ORDER (first match wins):

### Rule O1: Direct Anchor Source
**When:** V comes directly from an auth anchor (session.userId, token.sub, etc.)
**Judgment:** `trusted`
**Rule ref:** `anchor_direct`

### Rule O2: Forward Trusted Pool Member
**When:** V is listed in `forward.json` → `trusted_pool[].pool_members_added`
**Judgment:** `trusted`
**Rule ref:** `forward.trusted_pool[pool_id]`

### Rule O3: Anchor-Bound Query Result
**When:** V comes from a DB query/read operation whose WHERE clause (or ORM equivalent) contains an auth anchor
**Judgment:** `trusted`
**Rule ref:** `R1+R2`
**Verify:** The anchor in the WHERE clause must be the same as in recon. Check `bound_anchor` in the corresponding `forward.json` → `datasinks_reached` entry.

### Rule O4: Anchor-Guarded Data
**When:** V comes from data that passed through an auth guard (if owner == currentUser → throw/return)
**Judgment:** `trusted` (in the passing branch that reaches the return)
**Rule ref:** `R4`

### Rule O5: Approval Flow Context
**When:** V is related to an approval workflow (ticket status, approval result, pending action) that was initiated with auth anchor binding
**Judgment:** `trusted`
**Rule ref:** `approval_flow`
**Verify:** The approval creation must have used an auth anchor. Trace the approval workflow initiation in forward.json.

### Rule O6: User Input Echo
**When:** V is the direct value of an input parameter, passed through to output without modification or with only R7 transforms (toString, format, etc.), and NOT used in any datasink constraint
**Judgment:** `output_only_safe`
**Note:** This is not a vulnerability — the user is just seeing their own input. But mark it in case downstream consumers depend on it.

### Rule O7: Static Constant / Literal
**When:** V is a hardcoded literal, `static final` constant, enum value, or other compile-time constant that CANNOT be modified by any user
**Judgment:** `static_safe`
**WARNING:** Before applying O7, VERIFY:
- Is it truly a compile-time constant? Or is it loaded from a config file that users can modify?
- Is it an enum value used as-is? Or is the enum choice based on user input?
- Is it a "constant" from a database table that users can write to?
- If uncertain → do NOT apply O7 → continue to O8

### Rule O8: Unbound Query Result
**When:** V comes from a DB query/read WITHOUT auth anchor in the WHERE clause
**Judgment:** `at_risk`
**Rule ref:** `R8`

### Rule O9: Unbound Computed Value
**When:** V is a computed boolean/enum/number based on logic that does NOT involve auth anchor comparison
**Judgment:** `at_risk` (unless deep_analysis proves all inputs are trusted)
**Rule ref:** `R8`
**Requires:** `deep_analysis` block tracing ALL computation inputs

### Rule O10: User-Writable Source
**When:** V comes from a cache, database field, or configuration that ANY user can write to (not auth-scoped)
**Judgment:** `at_risk`

### Rule O11: External Call Without Identity Propagation
**When:** V comes from an RPC/HTTP call to an external service, and the call does NOT pass the current user's identity
**Judgment:** `at_risk`

---

## Boolean / Enum Deep Analysis

When the output field is `boolean` or an enum type, ALWAYS perform deep analysis:

### Step 1: Identify the Computation
Find the code that produces this boolean/enum value. It could be:
- Direct return of a boolean expression
- Ternary: `condition ? value1 : value2`
- Function call returning boolean: `return checkAccess(userId, resourceId)`
- Multiple branches: if/else setting the value

### Step 2: Identify Input Variables
List ALL variables/function-call-results that influence the final value.

### Step 3: Trace Each Input
For each input variable:
- Is it an auth anchor? → trusted
- Is it a trusted pool member (from forward.json)? → trusted
- Is it a constant/literal? → static_safe
- Is it a user input parameter? → check forward.json for its trust status
- Is it a query result? → check if the query binds an anchor
- Is it from an external source? → check if identity was propagated

### Step 4: Synthesize
- If ALL inputs are trusted → the boolean/enum is `trusted`
- If ANY input is at_risk → the boolean/enum is `at_risk`
- Document the full chain in `deep_analysis.computation_path`

### Example: Safe Boolean
```java
// Output: result.canAccess
public boolean canAccessOrder(Long orderId, Long currentUserId) {
    Order order = orderRepo.findByIdAndUserId(orderId, currentUserId);
    // Both orderId AND currentUserId in query → R1
    return order != null;
}
// → deep_analysis: Both inputs trusted (orderId via R1, currentUserId is anchor)
// → final_judgment: trusted
```

### Example: Risky Boolean
```java
// Output: result.isActive
public boolean isActiveOrder(Long orderId) {
    Order order = orderRepo.findById(orderId);  // No anchor binding
    return order.getStatus().equals("ACTIVE");
}
// → deep_analysis: orderId is user-controlled, query has no anchor → R8
// → The boolean reveals whether orderId exists AND is active
// → final_judgment: at_risk (information oracle: attacker can enumerate valid order IDs)
```

---

## Information Oracle Risk

An "information oracle" is when the response (even a 403/404 difference, or a true/false) reveals information the user shouldn't have.

### Detection Patterns

1. **Boolean existence check:** `return orderRepo.existsById(orderId)` → reveals whether resource exists
2. **HTTP status difference:** 403 Forbidden vs 404 Not Found → reveals existence
3. **Different error messages:** "wrong password" vs "user not found" → reveals valid usernames
4. **Boolean "has access" check:** `return hasPermission(userId, resourceId)` → reveals resource existence

### Classification

- If the boolean ONLY uses trusted data → info oracle risk is LOW (user is checking their own data)
- If the boolean uses untrusted parameter in query → info oracle risk is HIGH

Mark information oracle findings with `risk_reason` including "information_oracle: true" description.

---

## Multi-Source Field Handling

When an output field can come from multiple sources (branches):

```java
if (condition) {
    return queryA();  // Source A
} else {
    return queryB();  // Source B
}
```

**Required:** Trace BOTH branches. The field's safety is the WORST of all branches:
- If Source A is trusted but Source B is at_risk → `at_risk`
- Document both branches in `computation_path`

---

## Nested Object Expansion

When the return type is a complex object, expand ALL fields:

```
ResponseEntity<OrderDTO>
  → OrderDTO
    → orderId (Long)
    → userId (Long)
    → items (List<OrderItemDTO>)
      → itemId (Long)
      → productName (String)
    → status (String)
```

Field paths:
- `result.data.orderId`
- `result.data.userId`
- `result.data.items[].itemId`
- `result.data.items[].productName`
- `result.data.status`

Each field gets its own entry in `output_fields` with its own `final_judgment`.
