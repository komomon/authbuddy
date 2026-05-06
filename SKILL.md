---
name: authbuddy
description: |
  Universal authorization vulnerability audit (越权漏洞审计) for single API endpoints.
  Methodology-driven, language-agnostic, rule-free. Detects BOLA, identity impersonation,
  BFLA, unauthorized access, mass assignment, approval bypass, output data leak,
  and trust anchor compromise through complete trust chain analysis.
  Uses multi-agent pipeline: Recon → Entry Analysis → CallChain → Forward Trust →
  Backward Output → Report. Each stage verified before proceeding.
tools:
  - Read
  - Grep
  - Glob
  - Bash
  - Task
  - Agent
model: sonnet
priority: high
---

# AuthBuddy — Universal Authorization Vulnerability Audit

Analyze a single API endpoint for authorization vulnerabilities by tracing trust relationships between user-controlled parameters and authentication anchors. Works across languages and frameworks through semantic understanding.

## Trigger

```
/authbuddy /api/users/getInfo              # Analyze specific HTTP endpoint
/authbuddy com.example.controller.UserController.getInfo  # Analyze by class.method
/authbuddy /api/orders/{id} --verbose      # With detailed progress output
```

---

## Results Directory Convention

All pipeline artifacts are written to a results directory derived from the endpoint identifier:

```
results/{sanitized_endpoint_name}/
├── recon.json
├── entry.json
├── callchain.json
├── callchain-verify.json
├── forward.json
├── forward-verify.json
├── backward.json
├── backward-verify.json
├── report.md
└── report.json
```

**Sanitization rule:** Replace `/`, `{`, `}`, `\`, `:`, `*`, `?`, `<`, `>`, `|`, spaces with `_`.
Example: `/api/orders/{id}` → `results/_api_orders__id_/`

**The orchestrator MUST create this directory before dispatching A00.**

---

## Core Principles (READ FIRST)

### Principle 1: Trust Chain Analysis

**Authorization vulnerability = user-controlled parameter reaches a data operation without establishing trust relationship with an authentication anchor.**

Trust relationship can form at ANY point in the data flow — before, during, or after the data operation. The analysis traces the complete lifecycle of each parameter.

### Principle 2: Per-Endpoint Independence

**Every endpoint MUST be analyzed as if the attacker calls it directly and independently.**

- Do NOT assume data in the database is "safe" because another endpoint wrote it with auth checks
- Each input parameter must establish its OWN trust chain to an auth anchor WITHIN THIS endpoint's execution scope
- Data from database queries is only trusted if the query itself includes an auth anchor constraint

### Principle 3: Evidence Completeness

**Every judgment field MUST include three elements:**
1. Code location: `file` + `line`
2. Rule reference: `rule_ref` citing the specific methodology rule:
   - R1-R10: `reference/trust-propagation-rules.md` (trust propagation)
   - O1-O11: `reference/output-semantics.md` (output tracing)
   - S0-S8: `reference/scenario-taxonomy.md` (vulnerability scenarios)
3. Code snippet: `code_snippet` (1-3 lines of actual code from the file)

**Forbidden words in any judgment field:** "可能", "也许", "大概", "似乎", "should be", "might be", "probably", "seems like", "待确认", "需进一步分析"

### Principle 4: No Depth Limit

Do NOT truncate analysis at a fixed depth. Continue until a semantic termination condition is met: external code, pure static method, already-visited node (cycle), simple getter/setter, or leaf node with no sub-calls.

### Principle 5: Methodology Over Keywords

Keywords and framework names are weak hints only — they never drive judgments. Authorization conclusions come from semantic analysis of trust relationships, not pattern matching.

### Principle 6: Output File Enforcement

Every sub-agent MUST write its result to a file. Text output in conversation is NOT a substitute. The orchestrator checks file existence and non-emptiness before proceeding.

### Principle 7: Confidence Awareness

Every trust judgment MUST include a `confidence` level. Agents must honestly express certainty:

| confidence | Meaning | When to use |
|-----------|---------|-------------|
| `high` | Complete trace, no gaps | Full data flow traced end-to-end, all calls resolved |
| `medium` | Minor gaps, unlikely to change conclusion | One or two unresolved calls, but surrounding evidence is strong |
| `low` | Significant gaps, judgment tentative | Multiple unresolved calls, reflection, dynamic dispatch, or large code skipped |

`low` confidence on a `trusted` judgment is a red flag — it means "probably safe but can't prove it." The orchestrator must weigh confidence when making final scenario classifications.

---

## Public Endpoint Handling

Some endpoints are LEGITIMATELY public — no authentication required by design (e.g., `/api/public/status`, `/api/health`, login endpoints, public search).

**A00-Recon MUST detect and record this.** If the endpoint has NO auth mechanism at all (no interceptors, no session, no token, no annotations requiring auth), add to `recon.json`:

```json
"endpoint": {
  "is_public_endpoint": true,
  "public_endpoint_rationale": "string (why this appears to be intentionally public — e.g., 'No auth interceptors in chain', 'Annotated @PermitAll', 'Login endpoint')"
}
```

**If `is_public_endpoint: true` AND `sensitive_operations: false`, the orchestrator skips A01-A04 and dispatches A05 for an S0 report:**

- **S0: Public Endpoint** — Confirmed intentionally public. No authorization vulnerability by design.
- Report documents the endpoint structure from recon.json alone.

**If `is_public_endpoint: true` AND `sensitive_operations: true`, the orchestrator continues the full pipeline.** The endpoint is public but accesses sensitive data (reads PII, writes data, admin functions) — the DESIGN is the vulnerability → will be classified as S4.

**Decision flow:**
```
A00 reports is_public_endpoint = true
  │
  ├─ Endpoint performs sensitive operations? (read PII, write data, admin functions)
  │   └─ YES → CONTINUE pipeline (S4 risk: sensitive operations without auth)
  │
  └─ Endpoint is truly public (health check, login, public catalog, static content)
      └─ SKIP A02-A04, A05 reports S0: Public Endpoint
```

---

## Execution State Machine

```
START
  │
  ▼
Create results/{endpoint}/ directory
  │
  ▼
[A00: Recon] ──── READ agents/agent-00-recon.md
  │
  ▼ Gate 1 (includes public endpoint check)
  │
  ├─ is_public_endpoint + NO sensitive ops → SKIP to S0 report → DONE
  │
  ▼
[A01: Entry] ──── READ agents/agent-01-entry.md
  │
  ▼
[A02: CallChain] ── READ agents/agent-02-callchain.md
  │
  ▼
[A02b: Verify] ──── READ agents/agent-02b-callchain-verify.md
  │
  ▼ Gate 2
[A03: Forward] ──── READ agents/agent-03-forward.md
  │
  ▼
[A03b: Verify] ──── READ agents/agent-03b-forward-verify.md
  │
  ▼ Gate 3
[A04: Backward] ─── READ agents/agent-04-backward.md
  │
  ▼
[A04b: Verify] ──── READ agents/agent-04b-backward-verify.md
  │
  ▼ Gate 4
[A05: Report] ──── READ agents/agent-05-report.md
  │
  ▼
DONE
```

---

## Gating Conditions

### Gate 1: Recon Completeness

Check `results/{endpoint}/recon.json`:
- [ ] `endpoint.entry_file` non-empty AND file exists
- [ ] `endpoint.entry_function` non-empty
- [ ] `framework.language` identified
- [ ] `endpoint.is_public_endpoint` field present

If `is_public_endpoint = false`:
- [ ] `auth_context.auth_anchors` at least one entry
- [ ] `interceptor_chain` array exists (empty is OK)

If `is_public_endpoint = true`:
- [ ] `endpoint.public_endpoint_rationale` non-empty
- [ ] Orchestrator decides: does endpoint access sensitive data? → YES: continue pipeline. NO: skip to S0 report.

Fail → re-run A00 with specific missing items listed

### Gate 2: CallChain Completeness

Check `results/{endpoint}/callchain-verify.json`:
- [ ] `overall_pass` is true OR no `missing_nodes` with `severity: high`
- [ ] `callchain.json` has at least 1 datasink or leaf node reached
- [ ] `callchain.json` edges array non-empty

Fail → dispatch A02 回补 agent with `missing_nodes` list

### Gate 3: Forward Trust Completeness

Check `results/{endpoint}/forward-verify.json`:
- [ ] `overall_pass` is true OR no high severity issues
- [ ] `trusted_pool` covers ALL `auth_anchors` from recon
- [ ] All `resource_identifier` + `identity` role params have `final_trust_status` AND `confidence`
- [ ] No `low` confidence `trusted` judgments without explicit review note

Fail → dispatch A03 回补 agent with issue list

### Gate 4: Backward Output Completeness

Check `results/{endpoint}/backward-verify.json`:
- [ ] `overall_pass` is true OR no high severity issues
- [ ] All `at_risk` fields have non-empty `risk_reason` AND `confidence`
- [ ] No `low` confidence `trusted` judgments without explicit review note

Fail → dispatch A04 回补 agent with issue list

---

## Retry Decision Logic

| Verify Finding Severity | Action |
|--------------------------|--------|
| `high` | MUST 回补 — dispatch new sub-agent with missing item list |
| `medium` + involves `resource_identifier` / `identity` param | 回补 |
| `medium` + other role | Defer to report limitations (judgment call by orchestrator) |
| `low` | Record in `limitations`, do NOT 回补 |

Confidence-weighted 回补: if an agent's `trusted` judgment has `confidence: low`, treat as equivalent to `severity: medium` — it needs re-examination.

---

## Agent Tool Invocation

The orchestrator uses the `Agent` tool to spawn sub-agents. Each invocation MUST include:
- The agent definition file content (read it first, then include key instructions in the prompt)
- The exact output file path
- All input file paths
- A clear instruction that the agent MUST write to the output file

### Dispatch Template

When dispatching each agent, use this pattern:

```
Prompt:
"You are executing the {Agent ID} task from the AuthBuddy authorization audit pipeline.

Load your full agent definition from: agents/agent-XX-name.md
Read and follow ALL instructions in that file.

## Input Files
- {input_file_1}: {path}  (read this file)
- {input_file_2}: {path}  (read this file)

## Output Requirement
Write your results to: results/{endpoint}/{output_file}
Schema: schemas/{schema_name}.json

## Critical Rules
- You MUST write your output to the specified file path
- Do NOT skip the file write — text in conversation is not accepted
- Every judgment MUST include: file + line + code_snippet + rule_ref
- No fuzzy words allowed in judgment fields

## Agent Definition Summary
{Key instructions from the agent definition file, pasted here after reading it}"
```

### Example: Dispatching A00-Recon

```
Agent tool call:
- description: "A00-Recon for /api/orders/{id}"
- prompt: "You are executing the A00-Recon task...

Load your full agent definition from: agents/agent-00-recon.md

## Target
- Endpoint: /api/orders/{id}
- Project root: /path/to/project

## Output
Write results to: results/_api_orders__id_/recon.json
Schema: schemas/recon.schema.json

## Critical
- Identify is_public_endpoint: does this endpoint require authentication?
- Use Glob/Grep/Read to locate the entry function
- Trace ALL interceptors, including super/parent class methods
- Every auth_anchor must have credibility_checklist complete
- Write JSON to the output file — text in conversation is NOT accepted"
```

### Example: Dispatching a 回补 Agent

```
Agent tool call:
- description: "A02 回补 for callchain.json"
- prompt: "You are executing a 回补 task for A02-CallChain.

Load agent definition: agents/agent-02-callchain.md

## 回补 Context
The following nodes were MISSING from the original call chain:
- caller_node_id: N001, call_site_line: 55
  expression: super.formatParam(eventContext, arguments)
  target: BaseController.formatParam (src/.../BaseController.java:120)
  reason: Parent class method may contain identity parameter override

## Input
- Original callchain.json: results/_api_orders__id_/callchain.json (read it)
- recon.json: results/_api_orders__id_/recon.json

## Output
- Supplement the original callchain.json with ONLY the missing nodes/edges
- Write the COMPLETE updated callchain.json to: results/_api_orders__id_/callchain.json
- Schema: schemas/callchain.schema.json"
```

---

## Task Completion Check

After each sub-agent completes:

```
[TASK_CHECK: {Agent ID}]
1. {output_file} EXISTS? → YES/NO
2. {output_file} NON-EMPTY? → YES/NO
3. Evidence scan: any forbidden words ("可能", "似乎", etc.) in judgment fields? → NONE/FOUND
4. Evidence scan: code_snippet empty in any judgment entry? → NONE/FOUND
5. Confidence scan: any `trusted` or `at_risk` judgment without `confidence` field? → NONE/FOUND
6. Confidence scan: any `confidence: low` on `trusted` judgment? → FLAGGED (needs review)
→ PASS: proceed to next step
→ FAIL: re-dispatch with reason
```

---

## Final Report Assembly (A05)

After Gate 4 passes, dispatch A05 with all artifacts. A05 produces:
- `results/{endpoint}/report.md` — human-readable audit report
- `results/{endpoint}/report.json` — structured machine-consumable report

A05 classifies findings into scenarios S0-S8 defined in `reference/scenario-taxonomy.md`.

If the endpoint was determined to be public (S0), A05 produces the simplified S0 report directly from recon.json.

---

## Cross-Endpoint Awareness (Future)

While this version focuses on single-endpoint auditing, the orchestrator SHOULD note patterns that span endpoints:
- Multiple endpoints sharing the same vulnerable helper function
- Inconsistent auth checks between similar endpoints (e.g., GET has auth but POST doesn't)
- These observations go into `review_notes` for manual follow-up

This is NOT a full cross-endpoint analysis — just flagging for awareness.

---

## Self-Learning Protocol

If an agent discovers a pattern not covered in the reference files:
1. Complete the current analysis using semantic understanding
2. In the report, list all newly discovered items with full details in `extended_knowledge_candidates`
3. Present the list to the user for confirmation — do NOT write to any file automatically
4. After user approval, write confirmed entries to `reference/extended-knowledge.md`
5. Never modify existing reference files — extended-knowledge.md is append-only

---

## Key References

**Agent definitions (loaded when dispatching each agent):**
- `agents/agent-00-recon.md` — Pre-audit reconnaissance
- `agents/agent-01-entry.md` — Entry point parameter extraction
- `agents/agent-02-callchain.md` — Call chain construction
- `agents/agent-02b-callchain-verify.md` — Call chain verification
- `agents/agent-03-forward.md` — Forward trust propagation
- `agents/agent-03b-forward-verify.md` — Forward trust verification
- `agents/agent-04-backward.md` — Backward output tracing
- `agents/agent-04b-backward-verify.md` — Backward output verification
- `agents/agent-05-report.md` — Report generation

**Methodology (loaded by agents as needed):**
- `reference/trust-propagation-rules.md` — R1-R10 trust propagation rules
- `reference/datasink-patterns.md` — Data operation patterns requiring authorization
- `reference/auth-anchor-recognition.md` — Trust anchor identification methodology
- `reference/output-semantics.md` — Output field semantic analysis
- `reference/scenario-taxonomy.md` — Authorization vulnerability scenario S0-S8
- `reference/evidence-requirements.md` — Evidence triad specification
- `reference/extended-knowledge.md` — User-confirmed patterns from previous analyses

**Schemas:**
- `schemas/` — JSON Schema for each stage's output validation

---

## Important Scenarios (NEVER MISS THESE)

1. **Post-auth read (R5):** Data queried without auth, then filtered by anchor before returning → trusted. Do NOT flag as vulnerability.

2. **Super/parent class calls:** If an interceptor calls `super.xxx()`, the auth logic (like param override) may be in the parent class. Always trace to parent.

3. **Approval flow = authorization:** User A operating on User B's data through an approval workflow is delayed authorization — NOT a vulnerability.

4. **Redundant defense:** If Interceptor-1 already covers auth completely, Interceptor-2 being incomplete does NOT mean there's a vulnerability. This is defense-in-depth.

5. **Param override in interceptors:** Before calling any param user-controllable, check if an interceptor already replaced it with a session-derived value. `command.getXxx()` may already return trusted data.

6. **Beyond SQL:** Authorization vulnerabilities exist in file reads/writes, SSRF, RPC calls, network connections — not just SQL queries.

7. **Boolean/enum outputs:** A `true/false` response requires deep analysis — trace what logic produces that boolean and whether it involves auth anchors.

8. **Public endpoints:** Not every endpoint without auth is vulnerable. Login pages, health checks, public APIs are intentionally public. But if a public endpoint accesses sensitive data — flag it.

9. **Confidence matters:** A `trusted` judgment with `confidence: low` is not a clean pass — it means "probably safe but I can't trace the full path." Treat with suspicion.

---

## Version

- **Current:** 1.1
- **Updated:** 2026-05-06

### v1.1
- Added results directory convention
- Added public endpoint (S0) handling
- Added confidence level to all judgments
- Added Agent tool invocation examples
- Added extended-knowledge.md
- Added cross-endpoint awareness notes

### v1.0 (Initial Release)
- 9-agent pipeline architecture
- R1-R10 trust propagation rules
- S0-S8 authorization scenario taxonomy
- Evidence triad enforcement
- Semantic-driven termination (no fixed depth)
- Language/framework agnostic methodology
