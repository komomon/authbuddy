# Trust Propagation Rules (R1-R10)

These 10 rules are the formal judgment framework for trust chain analysis. Apply them in order of specificity when evaluating each parameter at its usage point.

---

## R1 — Direct Association

**When:** Parameter and trust anchor appear in the same operation's constraint/condition (e.g., same WHERE clause, same filter predicate).

**Result:** Parameter is **trusted**.

```java
// a1 is user-controlled, sessionUid is trust anchor
SELECT * FROM orders WHERE order_id = a1 AND user_id = sessionUid
// → a1 is TRUSTED (Direct Association within same WHERE clause)
```

```python
Order.objects.filter(id=order_id, user_id=request.user.id)
# → order_id is TRUSTED
```

```go
db.Where("id = ? AND user_id = ?", orderId, currentUserId).Find(&order)
// → orderId is TRUSTED
```

```javascript
db.query('SELECT * FROM orders WHERE id = $1 AND user_id = $2', [orderId, req.user.id])
// → orderId is TRUSTED
```

---

## R2 — Derived Trust

**When:** Result comes from an operation that was constrained by a trust anchor (R1 applied to the query).

**Result:** All fields of the result are **trusted**.

```java
CResult c = db.query("SELECT c1,c2,c3 FROM t WHERE c1=? AND c3=?", a1, sessionUid);
// c is from an anchor-constrained query → c.c1, c.c2, c.c3 are ALL trusted
```

**Crucial:** This only applies when the result variable stays within the same execution context and is not tampered with by untrusted data before use.

---

## R3 — Transitive Trust

**When:** Operation uses already-trusted data (from R1/R2/R3) as its constraint.

**Result:** Output is **trusted**.

```java
// c.c3 is trusted (from R2)
EResult e = db.query("SELECT e1,e2 FROM t2 WHERE e3=?", c.getC3());
// → e.e1, e.e2 are trusted (R3)
```

**Trust chain:** `sessionUid → c (R1+R2) → e (R3)`

**Critical R3 boundary:** A trusted value being used as a CONSTRAINT extends trust. A trusted value being used only as DISPLAY/RETURN data does NOT extend trust. Example:
```java
// c.c3 is trusted. It's returned as display data.
return new Response(c.getC3());  
// Later query using this response field? → NOT automatically trusted
// The trust must be re-established at the new usage point.
```

---

## R4 — Conditional Guard

**When:** Code compares the parameter (or its derivative) with a trust anchor AND throws/returns/aborts on mismatch. Parameter is trusted ONLY after the guard passes (in the branch where execution continues).

**Result:** **Trusted** in the passing branch.

```java
if (!order.getUserId().equals(sessionUid)) {
    throw new AccessDeniedException("Not your order");
}
// After this: order and its fields are trusted (R4 — guard validated ownership)
```

```python
if obj.owner_id != request.user.id:
    raise PermissionDenied()
# After this: obj is trusted
```

**Distinguish from business logic:**
```java
if (order.getStatus().equals("ACTIVE")) { ... }  // NOT auth → no trust change
```

**Permission service queries also qualify:**
```java
if (!permissionService.hasAccess(sessionUid, resourceId)) {
    throw new ForbiddenException();
}
// sessionUid and resourceId are trusted in this context → R4
// The permission table acts as the trust bridge
```

**Important:** A guard that passes but the code then continues to use ANOTHER untrusted parameter does NOT protect that other parameter. Each parameter needs its own guard or constraint.

---

## R5 — Post-Auth Read (Retroactive Trust)

**When:** A READ operation uses untrusted parameters, but the result is LATER associated with a trust anchor (through query, filter, or guard) BEFORE reaching the user.

**Result:** The original parameter's read usage is **trusted** retroactively.

```java
// Step 1: Untrusted read
DResult d = db.query("SELECT d1,d2 FROM t WHERE d3=?", a3);  // a3 has no anchor

// Step 2: Result associated with anchor before return
FilteredResult f = db.query("SELECT * FROM t2 WHERE key=? AND uid=?", d.d1, sessionUid);
return f;  // Only f is returned, not raw d

// → R5: a3's read is trusted (result was filtered by anchor before reaching user)
```

**Key requirement:** The trust association MUST happen BEFORE the data reaches the user (return/response). If raw `d` is returned without filtering, R5 does NOT apply.

---

## R6 — Post-Auth Write (Risk Pending)

**When:** A WRITE operation (INSERT/UPDATE/DELETE, file write, network send) executes using untrusted parameters, and auth association happens only AFTERWARD.

**Result:** **At Risk** — the write is already executed and cannot be undone.

```java
// WRITE with untrusted param — IRREVERSIBLE
db.execute("DELETE FROM orders WHERE id=?", orderId);  // orderId has no anchor

// Auth check comes too late
Order o = db.query("SELECT * FROM orders WHERE id=? AND uid=?", orderId, sessionUid);
if (o == null) { throw new NotFoundException(); }  // DELETE already happened!

// → R6: AT RISK — mark as HIGH severity
```

**R5 vs R6 distinction:**
- R5 (read): data can still be filtered before reaching user → retroactive trust valid
- R6 (write): operation has side effects that cannot be rolled back → risk remains

---

## R7 — Transform Neutral

**When:** A parameter undergoes transformation (toString, split, parseInt, format, trim, toUpperCase, substring, etc.).

**Result:** Trust status **unchanged**. Transformation does not create or destroy trust.

```java
String orderIdStr = String.valueOf(orderId);  // untrusted → still untrusted
Long parsed = Long.parseLong(input);          // untrusted → still untrusted
String[] parts = input.split(",");            // untrusted → each part still untrusted
```

Also applies to: `Integer.parseInt()`, `substring()`, `replace()`, `trim()`, `concat()`, string interpolation, `String.format()`, type casting between compatible types.

---

## R8 — No Association

**When:** A parameter (or its derivatives) reaches a datasink without ANY trust anchor association throughout its entire data flow lifecycle. No R1-R6 applies.

**Result:** **At Risk** — this is a potential authorization vulnerability.

```java
DResult d = db.query("SELECT d1,d2 FROM t WHERE d3=?", a3);
return d;  // d1, d2 returned based solely on user-controlled a3
// → R8: AT RISK — classic BOLA vulnerability
```

**Severity by semantic role:**
- `resource_identifier` → HIGH
- `identity` → CRITICAL (identity impersonation)
- `relationship_context` → HIGH
- `data_payload` → MEDIUM (may be mass assignment)
- `filter` → LOW (may only affect view, not access)
- `pagination` / `sorting` → INFO

---

## R9 — Stored Identity Re-validation

**When:** Data retrieved from database/cache contains an **identity field** (operatorUserId, createdBy, ownerId, etc.) that represents WHO performed a previous action. This identity may differ from the current requesting user.

**Result:** The stored identity field is **NOT automatically trusted** even if the query that retrieved it has some auth binding. It MUST be **explicitly compared** with the current user's trust anchor.

**Two MANDATORY checks:**

**Check 1: Identity comparison**
```java
ValidateMO mo = repo.findByOrderNo(orderNo);  // even if query is auth-scoped

// R9 REQUIRES this check:
if (!currentUserId.equals(mo.getOperatorUserId())) {
    throw new AccessDeniedException("Not the original operator");
}
// Without this → CRITICAL: Identity Impersonation Risk
```

**Check 2: Stored value usage**
```java
// ✅ Use stored values for business operations:
order.setEnterpriseId(mo.getEnterpriseId());

// ❌ Do NOT use user input when stored value exists:
order.setEnterpriseId(request.getEnterpriseId());
// This allows parameter substitution — attacker can alter fields that should be fixed
```

**Detection pattern — multi-stage operations:**
- Phase 1: creates record, stores operator identity and business data
- Phase 2: reads record, uses stored identity/data for business operation
- **Check 1:** Does Phase 2 verify `currentUserId == storedOperatorUserId`?
- **Check 2:** Does Phase 2 use stored values (not user input) for business fields?

---

## R10 — Trust Anchor Credibility

**When:** The trust anchor itself may be derived from user-controllable sources.

**Result:** If the trust anchor can be user-controlled, it is **NOT a valid anchor**. ALL trust conclusions based on it are **invalidated**. Mark as **CRITICAL: Trust Anchor Compromised**.

```java
// DANGEROUS: trust anchor has fallback to user input
@AuthAnnotation(noLoginExchangeUid = true)
public Result operation(Request request) {
    String currentUserId = null;
    if (RpcHolder.getUserUid() != null) {
        currentUserId = RpcHolder.getUserUid();      // Source 1: RPC → trusted
    } else if (request.getToken() != null) {
        currentUserId = tokenService.verify(request.getToken());  // Source 2: token → trusted
    }
    if (currentUserId == null) {
        currentUserId = request.getUserUid();  // Source 3: USER INPUT → NOT TRUSTED!
    }
    // ALL auth checks using currentUserId are now MEANINGLESS if Source 3 is reached
}
```

**Detection checklist:**
1. Does the endpoint have annotations allowing unauthenticated access? (`noLoginExchangeUid = true`, `permitAll`, `@Anonymous`)
2. Does the trust anchor getter have fallback logic to user input? (`defaultIfEmpty(tokenUid, request.getUserUid())`)
3. Do gray toggles / feature flags control whether strict auth is enforced?
4. Is there a conditional path where the anchor comes from the request instead of the session/token?

**Gray toggle risk pattern:**
```java
if (grayToggleManager.isHit("FORCE_CHECK_TOKEN")) {
    uid = tokenService.verifyStrict(token);  // strict → trusted
} else {
    uid = request.getUserUid();  // toggle off → user-controlled!
}
```

---

## Decision Flowchart

```
Start: Parameter P at usage point
  │
  ├─ Is P itself a trust anchor? → YES → TRUST ANCHOR (R10 check needed)
  │
  ├─ Anchor in same constraint? → YES → R1: TRUSTED
  │
  ├─ Derived from trusted result? → YES → R3: TRUSTED (check boundary)
  │
  ├─ Guard validates P against anchor? → YES → R4: TRUSTED (in passing branch)
  │
  ├─ READ op with result later anchor-associated? → YES → R5: TRUSTED
  │
  ├─ WRITE op with anchor after? → YES → R6: AT RISK (irreversible)
  │
  ├─ P reaches any datasink? → YES, no anchor → R8: AT RISK
  │
  └─ P unused in any datasink? → output_only_safe (check in backward analysis)
```

## Common Patterns

### Ownership Check Before Access
```java
Order order = orderRepo.findById(orderId);           // untrusted read
if (!order.getUserId().equals(currentUserId)) {       // R4 guard
    throw new AccessDeniedException();
}
return order;  // trusted after guard
```

### Scoped Query
```java
List<Order> orders = orderRepo.findByUserIdAndStatus(currentUserId, status);
// → R1 for currentUserId, status is also trusted by association
```

### Permission Table Lookup
```java
if (!aclService.canAccess(currentUserId, resourceId, "READ")) {  // R4
    throw new ForbiddenException();
}
Resource res = resourceRepo.findById(resourceId);  // trusted after guard
```

### Batch Operation with Partial Auth
```java
for (Long id : requestIds) {              // untrusted list
    Item item = itemRepo.findById(id);     // untrusted read
    if (item.getOwnerId().equals(uid)) {   // per-item R4 guard
        results.add(item);                 // only guarded items in result
    }
}
return results;  // R4 per-item → trusted
```

### Late-Binding Auth (Post-Query Filter)
```java
List<Record> all = recordRepo.findByCategory(categoryId);  // untrusted bulk read
List<Record> mine = all.stream()
    .filter(r -> r.getUserId().equals(currentUserId))       // R5 post-read filter
    .collect(Collectors.toList());
return mine;  // R5: trusted
```
