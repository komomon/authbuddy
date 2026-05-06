# Auth Anchor Recognition Methodology

An auth anchor is a value that identifies the current user and CANNOT be controlled by the user. It is the foundation of all trust chain analysis. If an anchor is compromised, ALL conclusions based on it are invalid (R10).

---

## Anchor Source Types

### 1. Server-Side Session

**How it works:** User identity stored server-side, keyed by a session cookie. The client only has the opaque session ID, not the identity value itself.

**Recognition patterns:**
```java
request.getSession().getAttribute("userId")
request.getSession().getAttribute("user")
```
```python
request.session.get("user_id")
request.session["user"]
```
```go
session.Get(ctx, "userId")
```
```javascript
req.session.userId
req.session.user
```

**Credibility:** HIGH. Session data is stored server-side, user cannot directly modify.

### 2. Verified Token/JWT

**How it works:** Identity extracted from a cryptographically verified token.

**Recognition patterns:**
```java
Jwts.parser().setSigningKey(key).parseClaimsJws(token).getBody().getSubject()
tokenService.verify(token).getUserId()
jwtDecoder.decode(token)
```
```python
jwt.decode(token, key, algorithms=["HS256"])["sub"]
```
```go
token.Claims.(jwt.MapClaims)["sub"]
```
```javascript
jwt.verify(token, secret).sub
```

**Credibility:** HIGH — IF token signature is verified. Must check:
- Is the signing key/secretthe same on every verification call?
- Is the token verified BEFORE extracting claims?

### 3. Framework Auth Injection

**How it works:** Framework extracts authenticated user from request context and injects it.

**Recognition patterns:**
```java
@AuthenticationPrincipal UserDetails user
SecurityContextHolder.getContext().getAuthentication().getPrincipal()
```
```python
request.user  # Django
current_user  # Flask-Login
```
```go
ctx.Value("user")  # after auth middleware
```
```javascript
req.user  # Passport.js
req.currentUser  # after auth middleware
```

**Credibility:** HIGH — IF the framework auth middleware is configured correctly. Must verify the middleware is actually applied to this endpoint.

### 4. Internal SDK / Trusted Internal Source

**How it works:** Identity obtained from internal infrastructure (RPC context, DRM, service mesh, SSO).

**Recognition patterns:**
```java
RpcContext.getUid()
ThreadLocalHolder.getUserId()
SsoUtil.getCurrentUserId()
AuthSdk.getOperatorId()
DrmContext.getUser()
MistContextHolder.getUserId()
```

**Credibility:** MEDIUM-HIGH — Depends on the SDK implementation. Must verify:
- Does the SDK get the value from a trusted source (e.g., request metadata injected by API gateway)?
- Or does it fall back to user-controllable input?
- Is the ThreadLocal/context properly set for ALL code paths?

### 5. Configuration-Based

**How it works:** Identity from a configuration that is not user-modifiable.

```java
@Value("${system.service.account.id}")
```

**Credibility:** HIGH for server-side config. LOW if config is user-overridable (e.g., per-tenant config that tenants can modify).

---

## Anchor Credibility Checklist (R10)

For EVERY anchor discovered, answer these four questions:

### Q1: Does the endpoint allow unauthenticated access?

Look for:
- `@PermitAll`, `permitAll()`, `@Anonymous`, `security=none`
- `noLoginExchangeUid=true`, `noLogin=true`
- `@csrf_exempt` without `@login_required` (Django)
- `authenticate=false` in route config
- Whitelist/exclusion list entries

**If YES:** The anchor may be null/absent. ALL trust conclusions must handle the null case. If code allows execution when anchor is null, all trust is invalid.

### Q2: Is there a fallback to user input?

Look for:
```java
if (uid == null) {
    uid = request.getParameter("userId");  // DANGER
}
uid = Optional.ofNullable(tokenUid).orElse(request.getUserId());  // DANGER
```
```python
user_id = token.get("sub") or request.args.get("userId")  # DANGER
```

**If YES:** CRITICAL — Anchor can be replaced by attacker input. ALL trust conclusions invalid.

### Q3: Is there a gray toggle / feature flag?

Look for:
```java
if (grayToggle.isEnabled("NEW_AUTH")) {
    uid = strictVerify(token);
} else {
    uid = request.getParameter("userId");  // DANGER when toggle off
}
```

**If YES:** Check which path is DEFAULT. If the weaker path is the default → CRITICAL.

### Q4: Are there multiple fallback sources with different credibility?

A chain like:
```
1. RPC context → trusted
2. Token → trusted (if verified)
3. Query parameter → NOT trusted
```

**If YES:** The anchor credibility equals the credibility of the WEAKEST link that can be reached.

---

## Anchor Acquisition Chain Tracing

When documenting how an anchor is obtained, trace the COMPLETE chain:

**Example (Java Spring):**
```
SecurityContextHolder.getContext()          // ThreadLocal
  → Authentication                          // set by filter chain
    → getPrincipal()                        // UserDetails or Principal
      → ((UserPrincipal) principal).getUserId()  // application-specific
```

**Example (Django):**
```
request.user                                // set by auth middleware
  → user.is_authenticated                   // check
    → request.user.id                       // DB-backed ID
```

**Example (Go Gin + JWT):**
```
c.Get("user")                               // set by auth middleware
  → token.Claims.(jwt.MapClaims)            // JWT claims map
    → claims["sub"]                         // subject claim
```

---

## Anchor Naming Conventions (Cross-Language Hints)

Common variable names that often hold auth anchors (weak hints, not rules):

| Pattern | Typical Meaning |
|---------|----------------|
| `*userId`, `*user_id`, `*uid` | Current user identifier |
| `*currentUser*`, `*current_user*` | Current user object |
| `*sessionUid`, `*session_uid` | Session-based user ID |
| `*operatorId`, `*operator_id` | Operator/actor ID |
| `*ownerId`, `*owner_id` | Resource owner (may be stored, not current user — R9) |
| `*creatorId`, `*creator_id` | Creator of resource (stored identity — R9) |
| `*tenantId`, `*tenant_id` | Multi-tenant context |
| `*tokenUid`, `*token_uid` | User ID from verified token |
| `*RpcContext.getUid()` | RPC-provided user ID |
| `*SecurityContext.get*` | Framework security context |
| `*ThreadLocalHolder.get*` | Thread-local stored identity |

**Important:** Variable names are HINTS only. An anchor is defined by WHERE its value comes from, not what it's named. A variable named `currentUserId` that reads from `request.getParameter("userId")` is NOT an anchor.

---

## Anchor Propagation Patterns

Anchors spread through the code in these ways:

1. **Argument passing:** Anchor is passed as a parameter to sub-functions
2. **Field assignment:** Anchor stored in a field, used later by same object
3. **ThreadLocal:** Anchor stored in ThreadLocal, accessible from anywhere in the same thread
4. **Closure capture:** Lambda/anonymous class captures the anchor from enclosing scope
5. **Return value:** Anchor embedded in a returned object (e.g., query result with owner_id)

When building the trusted pool (A03 Phase 1), track which variables/fields carry the anchor's trust at each point.
