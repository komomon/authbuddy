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
2. Rule reference: `rule_ref` (R1-R10, S1-S8, termination judgment 1-11)
3. Code snippet: `code_snippet` (1-3 lines of actual code from the file)

**Forbidden words in any judgment field:** "可能", "也许", "大概", "似乎", "should be", "might be", "probably", "seems like", "待确认", "需进一步分析"

### Principle 4: No Depth Limit

Do NOT truncate analysis at a fixed depth. Continue until a semantic termination condition is met: external code, pure static method, already-visited node (cycle), simple getter/setter, or leaf node with no sub-calls.

### Principle 5: Methodology Over Keywords

Keywords and framework names are weak hints only — they never drive judgments. Authorization conclusions come from semantic analysis of trust relationships, not pattern matching.

### Principle 6: Output File Enforcement

Every sub-agent MUST write its result to a file. Text output in conversation is NOT a substitute. The orchestrator checks file existence and non-emptiness before proceeding.

---

## Execution State Machine

```
START
  │
  ▼
[A00: Recon] ──── READ agents/agent-00-recon.md
  │
  ▼ Gate 1
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

Check `recon.json`:
- [ ] `endpoint.entry_file` non-empty AND file exists
- [ ] `endpoint.entry_function` non-empty
- [ ] `auth_context.auth_anchors` at least one entry
- [ ] `interceptor_chain` array exists (empty is OK)
- [ ] `framework.language` identified

Fail → re-run A00 with specific missing items listed

### Gate 2: CallChain Completeness

Check `callchain-verify.json`:
- [ ] `overall_pass` is true OR no `missing_nodes` with `severity: high`
- [ ] `callchain.json` has at least 1 datasink or leaf node reached
- [ ] `callchain.json` edges array non-empty

Fail → dispatch A02 回补 agent with `missing_nodes` list

### Gate 3: Forward Trust Completeness

Check `forward-verify.json`:
- [ ] `overall_pass` is true OR no high severity issues
- [ ] `trusted_pool` covers ALL `auth_anchors` from recon
- [ ] All `resource_identifier` + `identity` role params have `final_trust_status`

Fail → dispatch A03 回补 agent with issue list

### Gate 4: Backward Output Completeness

Check `backward-verify.json`:
- [ ] `overall_pass` is true OR no high severity issues
- [ ] All `at_risk` fields have non-empty `risk_reason`

Fail → dispatch A04 回补 agent with issue list

---

## Retry Decision Logic

| Verify Finding Severity | Action |
|--------------------------|--------|
| `high` | MUST 回补 — dispatch new sub-agent with missing item list |
| `medium` + involves `resource_identifier` / `identity` param | 回补 |
| `medium` + other role | Defer to report limitations (judgment call by orchestrator) |
| `low` | Record in `limitations`, do NOT 回补 |

---

## Task Dispatch Format

When launching a sub-agent, use this exact format:

```
[AGENT_TASK]
Agent: {A00|A01|A02|A02b|A03|A03b|A04|A04b|A05}
Definition: agents/agent-XX-name.md
Inputs:
  - {input_file_1}
  - {input_file_2}
Output: {output_file_path}
Schema: schemas/{schema_name}.json
Special Instructions: {any extra context}
```

When launching a 回补 agent, add:
```
回补 Context: {missing_items from verify JSON}
回补 Target: {specific nodes/params/fields to supplement}
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
→ PASS: proceed to next step
→ FAIL: re-dispatch with reason
```

---

## Final Report Assembly (A05)

After Gate 4 passes, dispatch A05 with all artifacts. A05 produces:
- `report.md` — human-readable audit report
- `report.json` — structured machine-consumable report

A05 classifies findings into scenarios S1-S8 defined in `reference/scenario-taxonomy.md`.

---

## Self-Learning Protocol

If an agent discovers a pattern not covered in the reference files:
1. Complete the current analysis using semantic understanding
2. In the report, list all newly discovered items with full details
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
- `reference/scenario-taxonomy.md` — Authorization vulnerability scenario S1-S8
- `reference/evidence-requirements.md` — Evidence triad specification

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

---

## Version

- **Current:** 1.0
- **Updated:** 2026-05-06

### v1.0 (Initial Release)
- 9-agent pipeline architecture
- R1-R10 trust propagation rules
- S1-S8 authorization scenario taxonomy
- Evidence triad enforcement
- Semantic-driven termination (no fixed depth)
- Language/framework agnostic methodology
