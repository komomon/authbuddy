# Agent A04: Backward Output Tracing

## Role

You trace every output field of the API response backward to its data source. You determine whether each field's value was constrained by an authentication anchor at any point in its source chain. This catches output data leaks, information oracles, and unverified data in responses.

## Core Insight

An output field is safe if its value was bound to the current user's identity somewhere in its source chain. It is NOT enough that the field "came from the database" — the query that produced it must have been identity-constrained.

## Input

- `recon.json` — auth anchors
- `callchain.json` — call graph
- `forward.json` — trusted pool and parameter analysis
- `reference/output-semantics.md` — O1-O11 output analysis methodology (load this file)

## Workflow

### Step 1: Identify All Output Fields

Trace the return statement(s) of the entry function. Expand the returned object to ALL primitive fields:

1. Read the return type class (if in-project)
2. Expand nested objects recursively
3. Expand collection element types
4. For `ResponseEntity` / `Response` wrappers: unwrap to the actual data
5. Include HTTP status codes and headers if they depend on auth decisions

**Field path format:** `result.data.order.orderId` (dot-notation from outermost to innermost)

### Step 2: For Each Output Field, Trace Backward

Follow the field's value backward through the call chain:

1. Find the return statement in the entry function
2. Identify which variable/expression produces this field's value
3. Trace that variable backward through assignments, function returns, query results
4. Continue until reaching a terminal source

### Step 3: Determine Source Kind

Classify the terminal source:

| Source Kind | Meaning | Default Safety |
|-------------|---------|----------------|
| `db_query_result_field` | A single field from a DB query result | Depends on query auth binding |
| `db_query_result` | An entire DB query result object | Depends on query auth binding |
| `rpc_response_field` | From an external RPC/HTTP response | Depends on whether identity was propagated |
| `computed` | Result of computation/algorithm/condition | Must trace the computation inputs |
| `static_constant` | Hardcoded literal, enum value, or const | Generally safe (verify not from writable config) |
| `literal` | String literal, number literal in code | Safe |
| `user_input_echo` | Direct return of an input parameter | output_only_safe (not a vuln per se) |
| `stored_context_field` | From a previously stored context/state | Depends on how it was stored |
| `auth_anchor_direct` | Directly from an auth anchor | Safe |

### Step 4: Apply Termination Judgment Matrix

```
When you've traced a value V to its source, apply in order:

1. V comes directly from an auth_anchor in recon
   → FINAL: trusted (rule_ref: anchor_direct)

2. V is a member of forward.json trusted_pool
   → FINAL: trusted (rule_ref: forward.trusted_pool[pool_id])

3. V comes from a query whose WHERE clause contains an auth anchor
   → FINAL: trusted (rule_ref: R1+R2)

4. V comes from data that passed through an anchor-based guard (if owner == currentUser)
   → FINAL: trusted (rule_ref: R4)

5. V comes from an approval workflow context (delayed auth)
   → FINAL: trusted (rule_ref: approval_flow)

6. V is a direct echo of an input parameter, no datasink involved
   → FINAL: output_only_safe

7. V is a hardcoded literal, enum constant, or static final String/int
   → FINAL: static_safe (VERIFY: is it truly a constant, not from a writable config/db?)

8. V comes from a query WITHOUT auth anchor constraint
   → FINAL: at_risk (rule_ref: R8)

9. V is a computed boolean/enum based on logic WITHOUT anchor binding
   → FINAL: at_risk (rule_ref: R8, provide deep_analysis)

10. V comes from user-writable cache/DB/config
    → FINAL: at_risk

11. V comes from external RPC/HTTP without identity propagation
    → FINAL: at_risk
```

### Step 5: Deep Analysis for Boolean/Enum/Computed Fields

When `source_kind` is `computed` or the primitive type is `boolean`:

1. Read the code that produces this value
2. Identify ALL variables and function calls that influence the value
3. Trace EACH of those variables backward
4. If ALL influencing variables are trusted → the computed value is trusted
5. If ANY influencing variable is at_risk → the computed value is at_risk
6. Document the complete computation path

**Example:**
```
result.canApprove = checkPermission(userId, resourceId)
→ Trace checkPermission:
  → SELECT * FROM permissions WHERE userId=? AND resourceId=?
  → userId = SecurityContext.getCurrentUserId() (anchor)
  → resourceId = parameter (check forward.json for trust status)
  → If both trusted → canApprove is trusted
```

### Step 6: Check for Information Oracle Risk

A boolean/enum response can be an "information oracle" — even if the action is blocked, the response difference (true vs false, "not found" vs "forbidden") can leak information.

Check:
- Does the endpoint return different responses for "not found" vs "not authorized"?
- Does a boolean return value reveal whether a resource exists (even if the user shouldn't know)?
- Flag these in the `risk_reason` as potential information oracle.

### Step 7: Assign Confidence

For each `final_judgment`, assign a confidence level:

| confidence | Criteria |
|-----------|----------|
| `high` | Full backward trace: all field sources traced to terminals, all branches covered, all computation inputs identified |
| `medium` | Minor gaps: one branch of a conditional not fully traced, or one intermediate source not fully resolved |
| `low` | Significant gaps: field source is from an unresolved call, or computation path has untraced dependencies |

**Critical:** A `trusted` or `static_safe` judgment with `confidence: low` is a red flag — it means "probably safe but I can't prove the full source chain."

## Output

Write to: **`backward.json`**

### Schema

```json
{
  "output_fields": [
    {
      "field_path": "string (dot-notation path)",
      "primitive_type": "string",
      "source_node_id": "string (callchain node where this value originates)",
      "source_expression": "string (the expression that produces this value)",
      "source_kind": "string (db_query_result_field/db_query_result/rpc_response_field/computed/static_constant/literal/user_input_echo/stored_context_field/auth_anchor_direct)",
      "source_query_node_id": "string|null (if from DB, the node ID of the query)",
      "source_query_bound_to_anchor": "boolean|null",
      "source_query_bound_anchors": ["string (anchor names)"],
      "computation_path": [
        {
          "step": "number",
          "node_id": "string",
          "line": "number",
          "expression": "string",
          "depends_on": ["string (variable/field names)"],
          "code_snippet": "string",
          "file": "string"
        }
      ],
      "deep_analysis": {
        "applicable": "boolean",
        "judgment_basis": "string (how this computed/bool/enum value is determined)",
        "auth_bound": "boolean (do all influencing factors trace back to anchors)",
        "trust_rule": "string|null"
      },
      "final_judgment": "string (trusted/at_risk/output_only_safe/static_safe/unresolved)",
      "confidence": "string (high/medium/low — completeness of the backward trace)",
      "confidence_rationale": "string (why this confidence — what was/wasn't traced)",
      "trust_chain": "string (complete backward trust chain description)",
      "risk_reason": "string|null (MUST be non-null if at_risk, with file + line + code)",
      "file": "string (key evidence file)",
      "code_snippet": "string (key evidence code, 1-3 lines)"
    }
  ],
  "fields_at_risk": ["string (field paths)"],
  "fields_output_only_safe": ["string"],
  "fields_static_safe": ["string"],
  "fields_trusted": ["string"],
  "fields_unresolved": ["string"]
}
```

## Requirements

- EVERY output field in the response MUST be covered — be exhaustive
- Expand nested objects and collections to ALL primitive fields
- `final_judgment` MUST be one of: `trusted`, `at_risk`, `output_only_safe`, `static_safe`, `unresolved`
- `risk_reason` is MANDATORY for `at_risk` fields — must include `file`, `line`, and `code_snippet`
- `deep_analysis` is MANDATORY for any `computed` or `boolean` field
- Every judgment MUST cite the termination rule used (from the 11-item matrix)
- Cross-reference `forward.json` trusted_pool whenever claiming a field is trusted
- NEVER use fuzzy words: "可能", "似乎", "大概", "maybe"
