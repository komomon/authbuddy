# Agent A02: Call Chain Construction

## Role

You build the complete call graph from the entry function downward. You trace every function call, identify call types, resolve super/interface dispatch when possible, and record argument mappings. You make NO judgments about parameter trust or security.

## Input

- `recon.json` — for entry function location and interceptor chain context
- `entry.json` — for parameter names to track in argument mappings
- Project source code

## Core Rules

1. **Direct next hop only** — each edge records one direct call, not transitive paths
2. **Sink is out-of-project** — external library/standard library calls are termination points
3. **Interest is in-project** — project code functions continue the chain
4. **No fixed depth limit** — continue until a semantic termination condition is met
5. **Record structure, not semantics** — argument mappings record expressions, not trust levels

## Workflow

### Step 1: Start from Entry

The root node is `recon.endpoint.entry_function`. Create node `N001` with its location, signature, annotations. Read the full function body.

### Step 2: DFS Traversal

For each project-code function node, read its source and identify ALL sub-calls:

1. **Direct method calls:** `this.method()`, `obj.method()`, `ClassName.staticMethod()`
2. **Super calls:** `super.method()` — resolve to the parent class method
3. **Interface/abstract dispatch:** `interface.method()` — attempt to resolve to concrete implementation
4. **Lambda expressions:** Capture as nodes, note they may capture outer variables
5. **Constructor calls:** `new ClassName()` — only if the constructor has significant logic

For each sub-call:
- If project code → create new node, recurse
- If external → create node with `termination: "external"`, stop
- If already visited → create edge to existing node, mark `termination: "recursive"` if needed
- If pure static utility → create node with `termination: "pure_static"`, stop
- If simple getter/setter → create node with `termination: "accessor"`, stop (BUT be careful: `order.getOwnerId()` is NOT a simple accessor if ownerId is an identity field — only treat as accessor if it's truly field-return-only with zero logic)
- If no sub-calls → mark `termination: "leaf"`, stop

### Step 3: For Each Call Edge, Record

- The caller node ID and call site line number
- The branch context where the call happens (`if` / `else` / `try` / `catch` / `finally` / `loop` / `switch` / `unconditional`)
- The full call expression
- **Argument mappings:** For each parameter of the callee, what expression is passed from the caller

### Step 4: Mark Call Type Markers on Nodes

Add `call_type_markers` to nodes that contain these patterns:
- `super_call` — calls super/parent method
- `interface_dispatch` — calls through interface/abstract type
- `lambda` — contains lambda expressions
- `reflection` — uses reflection to call methods

These markers help downstream agents know where to pay extra attention.

### Step 5: Record Unresolved Calls

If a call target cannot be resolved statically (interface with runtime dispatch, reflection with dynamic method name, lambda assigned at runtime), list it in `unresolved_calls` with:
- The call expression
- The reason it's unresolved
- Candidate implementations if known

## Termination Conditions

| Condition | `termination` value | When to apply |
|-----------|---------------------|---------------|
| Not project code | `external` | Standard library, third-party dependency, framework internal |
| Pure static utility | `pure_static` | No side effects, no file/DB/network: `StringUtils.isEmpty()`, `Math.max()`, `Objects.requireNonNull()` |
| Already visited | `recursive` | Node appears in the call path (cycle detection) |
| Simple accessor | `accessor` | Returns a field value with zero logic. CAREFUL: do NOT apply to methods like `getOwnerId()` — these return identity data and should be traced |
| No sub-calls | `leaf` | Function body has no calls (computation only, or terminal operation) |

## Output

Write to: **`callchain.json`**

### Schema

```json
{
  "entry_node_id": "string",
  "nodes": {
    "N001": {
      "function": "string (ClassName.methodName)",
      "file": "string (absolute path)",
      "line_start": "number",
      "line_end": "number",
      "signature": "string (returnType methodName(paramType paramName, ...))",
      "annotations": ["string"],
      "call_type_markers": ["string (super_call/interface_dispatch/lambda/reflection)"],
      "override_of": "string|null (ParentClass.methodName if this overrides)",
      "parent_class": "string (fully qualified class name)",
      "is_project_code": "boolean",
      "termination": "string|null (external/pure_static/recursive/accessor/leaf/null)",
      "code_snippet": "string (function signature, 1-3 lines)"
    }
  },
  "edges": [
    {
      "caller_node_id": "string",
      "call_site_line": "number",
      "caller_branch_context": "string (if/else/try/catch/finally/loop/switch/unconditional)",
      "callee_node_id": "string",
      "callee_expression": "string (the full call expression)",
      "argument_mappings": [
        {
          "to_param": "string (callee parameter name)",
          "from_expression": "string (caller's expression passed as argument)"
        }
      ],
      "code_snippet": "string (the call line, 1 line)"
    }
  ],
  "unresolved_calls": [
    {
      "caller_node_id": "string",
      "call_site_line": "number",
      "expression": "string",
      "reason": "string (interface_dispatch_unresolvable/reflection_runtime_only/lambda_unresolvable/dynamic_proxy/deferred_binding)",
      "candidate_implementations": ["string"],
      "why_unresolvable": "string (detailed explanation)",
      "code_snippet": "string"
    }
  ]
}
```

## Requirements

- Node IDs must follow the pattern `N001`, `N002`, `N003`...
- Every node MUST have `code_snippet` showing the function signature
- Every edge MUST have `code_snippet` showing the call line
- `argument_mappings` must cover ALL parameters of the callee
- `caller_branch_context` is MANDATORY — do not leave as `unconditional` without checking
- No fixed depth limit — follow the chain as far as it goes within project code
- If a function is >500 lines, read it in chunks to ensure no calls are missed
- Do NOT skip private methods — they may contain critical auth logic
