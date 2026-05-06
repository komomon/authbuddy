# Authorization Vulnerability Scenario Taxonomy (S0-S8)

Nine authorization vulnerability scenarios. Report findings by mapping to these scenarios.

---

## S0: Public Endpoint (公开端点)

**Definition:** An endpoint that is intentionally public — no authentication is required or expected. This is NOT a vulnerability; it's a confirmation that the endpoint is designed for unauthenticated access.

**Trigger conditions:**
- `recon.endpoint.is_public_endpoint` = `true`
- `recon.endpoint.sensitive_operations` = `false`
- Endpoint has annotations like `@PermitAll`, `@Anonymous`, `permitAll()`
- OR endpoint is in an auth exclusion/whitelist
- OR URL pattern indicates public: `/api/public/*`, `/login`, `/health`

**Severity:** `info` — informational only, not a vulnerability.

**When to ESCALATE to S4:**
- Public endpoint accesses sensitive data (PII, internal data)
- Public endpoint performs write operations
- Public endpoint provides admin-level functionality
- In these cases, the DESIGN is the vulnerability → escalate to S4: Unauthenticated Access to sensitive operations.

---

## S1: BOLA (Broken Object Level Authorization)

**Definition:** A resource identifier parameter reaches a data operation without binding to an auth anchor. The attacker can access/modify objects belonging to other users by changing the resource ID.

**Trigger conditions:**
- Parameter with `semantic_role` = `resource_identifier` or `relationship_context`
- `forward.json` → `final_trust_status` = `at_risk`
- Parameter reaches at least one datasink (db_read, db_write, file_read, file_write, rpc_call)

**Severity:**
- `critical`: resource_identifier reaches db_write without auth binding (modify/delete others' data)
- `high`: resource_identifier reaches db_read without auth binding (read others' data)
- `medium`: relationship_context reaches datasink without auth binding
- `low`: resource_identifier but resource is semi-public (e.g., product catalog, public profile)

**NOT BOLA (false positive patterns):**
- Parameter is bound to anchor in the same WHERE clause (R1) → trusted
- Parameter is in a query result that is later filtered by anchor (R5) → trusted
- Parameter is overridden by interceptor to session value → it's now an anchor-carrier
- The operation goes through an approval flow → delayed auth (S6)

**Example:**
```java
// BOLA: orderId reaches datasink without auth binding
GET /api/orders/{orderId}
→ SELECT * FROM orders WHERE id = orderId  // No user_id constraint!
```

---

## S2: Identity Impersonation (身份冒用)

**Definition:** An `identity`-role parameter that claims "who the user is" is not overridden by an auth anchor. The attacker can impersonate other users.

**Trigger conditions:**
- Parameter with `semantic_role` = `identity`
- `recon.interceptor_chain` does NOT show this parameter being overridden by any interceptor
- `forward.json` → `final_trust_status` = `at_risk`
- The parameter is used in a datasink (as query constraint, in auth_decision, etc.)

**Severity:**
- `critical`: identity used in db_write (acting as another user) or across multi-tenant boundary
- `high`: identity used in db_read (seeing another user's perspective)
- `medium`: identity used in non-data-changing operation

**NOT Identity Impersonation:**
- Parameter is `request.getUserId()` but an interceptor already called `request.setUserId(sessionUserId)` → the value is TRUSTED
- Parameter named `userId` but it's actually a `resource_identifier` (it identifies the resource owner, not the acting user)
- Parameter is `targetUserId` and it's used for admin functionality with role check → vertical privilege, not impersonation

**Example:**
```java
// Identity Impersonation: operatorId from request, not from session
POST /api/confirm
Body: { "operatorId": 123, "orderId": 456 }
→ UPDATE orders SET confirmed_by = operatorId WHERE id = orderId
// Attacker sets operatorId to victim's ID → action attributed to victim
```

---

## S3: BFLA (Broken Function Level Authorization)

**Definition:** A lower-privilege user can access higher-privilege functionality. The endpoint lacks role/permission checks appropriate for its functionality.

**Trigger conditions:**
- `recon.endpoint.annotations` shows no or weak role/permission requirements
- OR `recon.interceptor_chain` roles are insufficient for the endpoint's functionality
- The endpoint performs functionality that semantically requires elevated privileges (admin operations, data modification, user management)

**Severity:**
- `critical`: admin-only operation accessible to any authenticated user
- `high`: write operation with read-level auth
- `medium`: sensitive read with minimal auth

**Example:**
```java
// BFLA: No admin role check on admin functionality
@GetMapping("/api/admin/users")
public List<User> getAllUsers() {  // Only @LoginRequired but no @AdminRequired
    return userRepo.findAll();
}
```

---

## S4: Unauthenticated Access (未授权访问)

**Definition:** An endpoint that requires authentication can be accessed without it. The auth anchor is absent or can be bypassed.

**Trigger conditions:**
- R10 triggered: `credibility_checklist` shows a path where anchor is null or user-controlled
- OR `recon` shows endpoint is in a whitelist/exclusion list despite needing auth
- OR `noLoginExchangeUid = true` or similar annotation allows unauthenticated access

**Severity:**
- `critical`: endpoint modifies data AND allows unauthenticated access
- `high`: endpoint reads sensitive data AND allows unauthenticated access
- `medium`: any authenticated endpoint with unauthenticated bypass path

---

## S5: Mass Assignment

**Definition:** User can supply sensitive fields in request body that get bound and written to persistent storage without filtering.

**Trigger conditions:**
- Parameter(s) with `semantic_role` = `data_payload`
- Reaches datasink with `type` = `mass_assignment` or `db_write`
- No field-level filtering observed (no whitelist of allowed fields)
- Sensitive fields are writable (e.g., `role`, `isAdmin`, `balance`, `permissions`)

**Severity:**
- `critical`: privilege-related fields writable (role, isAdmin, permissions)
- `high`: identity-related fields writable (userId, ownerId) enabling cross-user modification
- `medium`: business-sensitive fields writable (price, status, balance)
- `low`: non-sensitive fields writable (description, tags)

**Example:**
```java
// Mass Assignment: request body directly bound to entity
@PostMapping("/api/users/{id}")
public User updateUser(@PathVariable Long id, @RequestBody User user) {
    // Attacker can send: { "role": "admin", "isPremium": true }
    return userRepo.save(user);  // All fields from request written to DB!
}
```

---

## S6: Approval Bypass (延迟授权绕过)

**Definition:** An operation supposed to be protected by an approval workflow can be executed without the workflow completing or the workflow itself lacks auth binding.

**Trigger conditions:**
- Direct operation on data that SHOULD go through approval
- AND the direct operation lacks auth anchor binding (R8)
- AND the approval flow is either absent, incomplete, or can be skipped

**Severity:**
- `critical`: critical action (payment, deletion, permission change) bypasses approval
- `high`: important action bypasses approval
- `medium`: workflow step skipped but other protections exist

**NOT Approval Bypass:**
- If an operation goes THROUGH approval → the approval flow IS the auth → S1 does NOT apply here
- If code creates an approval ticket with auth binding → the operation is authorized (delayed)

**Example:**
```java
// Approval Bypass: Direct update without going through approval
@PostMapping("/api/orders/{id}/confirm")
public void confirmOrder(@PathVariable Long id) {
    // This should create an approval ticket, but instead directly confirms
    order.setStatus("CONFIRMED");
    orderRepo.save(order);
}
```

---

## S7: Output Data Leak (出参信息泄露)

**Definition:** The response includes data that belongs to other users or reveals information the current user shouldn't have.

**Trigger conditions:**
- `backward.json` → output field with `final_judgment` = `at_risk`
- OR output field acts as information oracle (boolean/enum reveals existence of others' data)
- Response includes PII/identity data without auth binding

**Severity:**
- `critical`: output contains PII of other users (identity theft risk)
- `high`: output contains sensitive business data of other users
- `medium`: output reveals existence of resources the user shouldn't know about (oracle)
- `low`: output contains non-sensitive metadata about other users

**Example:**
```java
// Output Data Leak: Returns orders for ALL users
@GetMapping("/api/orders")
public List<Order> getOrders(@RequestParam Long userId) {
    return orderRepo.findByUserId(userId);  // No check that userId == current user!
}
```

---

## S8: Trust Anchor Compromise (信任锚点失效)

**Definition:** The auth anchor itself can be controlled or influenced by the user, invalidating ALL authorization decisions based on it.

**Trigger conditions:**
- R10 triggered: `credibility_checklist` has any `true` field
- `has_user_input_fallback` = true → anchor can be replaced with attacker value
- `has_gray_toggle` = true → anchor credibility depends on toggle state
- `has_noLogin_fallback` = true → anchor can be null, code may execute anyway

**Severity:**
- `critical`: anchor has direct user input fallback OR anchor is null and code executes
- `high`: gray toggle can disable strict auth
- `medium`: anchor has multiple sources with unclear which is active

**Effect:** When S8 is found, ALL other scenarios based on this anchor are automatically valid. S8 is the ROOT CAUSE — fix it first.

---

## Scenario Dependency

```
S8 (Anchor Compromise)
 ├─→ S4 (Unauthenticated Access) — if anchor is null
 ├─→ S2 (Identity Impersonation) — if anchor is replaced
 └─→ ALL other scenarios become valid (trust foundation destroyed)

S4 (Unauthenticated Access)
 └─→ S3 (BFLA) — if endpoint has sensitive functionality

S1 (BOLA) + S2 (Identity Impersonation) can co-exist:
 └─→ Combined severity = max(S1, S2) + 1 level
```

## Severity Assignment Rules

| Severity | Criteria |
|----------|----------|
| `critical` | S8 anchor compromise OR S2 identity impersonation confirmed + direct data access/write OR S1 resource_identifier reaches db_write |
| `high` | S1 resource_identifier reaches db_read OR S3 sensitive function without role check OR S5 privilege fields writable OR S6 critical action bypasses approval |
| `medium` | S1 relationship_context reaches datasink OR S7 output leak of sensitive data OR S5 business fields writable |
| `low` | S1 semi-public resource OR S7 info oracle OR S6 approval step skipped but other protections |
| `info` | Defense-in-depth noted OR best practice deviation without clear exploit path |
