# Evidence Requirements — The Evidence Triad

Every judgment field in every agent output MUST include three pieces of evidence. This is enforced at every gate by the orchestrator.

---

## Rule Reference Abbreviations

All rule references (`rule_ref`, `trust_rule`, `bound_trust_rule`, `judgment_basis`) use these abbreviations. Each maps to a specific reference file:

| Abbreviation | Source File | Description |
|-------------|-------------|-------------|
| `R1`-`R10` | `reference/trust-propagation-rules.md` | Trust propagation rules (e.g., R1=Direct Association, R4=Conditional Guard) |
| `O1`-`O11` | `reference/output-semantics.md` | Output termination judgment rules (e.g., O1=Direct Anchor Source, O8=Unbound Query Result) |
| `S0`-`S8` | `reference/scenario-taxonomy.md` | Authorization vulnerability scenarios (e.g., S0=Public Endpoint, S1=BOLA, S8=Anchor Compromise) |

**Always load the corresponding reference file before citing these rules.** The rule numbers alone are meaningless without the methodology context.

---

## The Evidence Triad

| Element | Field Name | Description | Usage |
|---------|-----------|-------------|-------|
| Code Location | `file` + `line` | Where in the source code the evidence is found | Absolute file path + integer line number |
| Rule Reference | `rule_ref` / `trust_rule` / `bound_trust_rule` | Which methodology rule justifies the judgment | `R1`-`R10`, `O1`-`O11`, `S0`-`S8` (see table above) |
| Code Snippet | `code_snippet` | Actual code from the file that supports the judgment | 1-3 lines, extracted from the source, with line numbers |

---

## Where Evidence is Required

### In A00 (recon.json):
| Field | Required Evidence |
|-------|------------------|
| `endpoint.code_snippet` | Entry function signature |
| `auth_anchors[].code_snippet` | Anchor acquisition code |
| `auth_anchors[].credibility_checklist.credibility_notes` | file_ref for each risk noted |
| `global_filters[].code_snippet` | Filter code |
| `interceptor_chain[].code_snippet` | Interceptor method signature + super call |

### In A01 (entry.json):
| Field | Required Evidence |
|-------|------------------|
| `parameters[].code_snippet` | Line(s) where parameter is declared/obtained |
| `entry_signature_code_snippet` | Full function signature |

### In A02 (callchain.json):
| Field | Required Evidence |
|-------|------------------|
| `nodes[].code_snippet` | Function signature |
| `edges[].code_snippet` | Call line |
| `unresolved_calls[].code_snippet` | Unresolved call line |

### In A02b (callchain-verify.json):
| Field | Required Evidence |
|-------|------------------|
| `missing_nodes[].code_snippet` | The missed call |
| `incorrect_edges[].code_snippet` | The problematic edge |
| `premature_terminations[].code_snippet` | The prematurely terminated node |

### In A03 (forward.json):
| Field | Required Evidence |
|-------|------------------|
| `trusted_pool[].propagation_chain[].code_snippet` + `file` | Code showing anchor usage |
| `trusted_pool[].propagation_chain[].trust_rule_applied` | Rule reference (R1-R10) |
| `parameter_analysis[].propagation_path[].code_snippet` + `file` | Code showing param usage |
| `parameter_analysis[].propagation_path[].trust_rule` | Rule reference for status change |
| `parameter_analysis[].datasinks_reached[].code_snippet` + `file` | Datasink code |
| `parameter_analysis[].trust_chain` | Complete trust chain description |
| `parameter_analysis[].unresolved_reason` | (if unresolved) file_ref for what's missing |

### In A03b (forward-verify.json):
| Field | Required Evidence |
|-------|------------------|
| All `*[].code_snippet` | Problematic code |
| All `*[].issue` / `*[].reason_current` / `*[].note` | file_ref in text |

### In A04 (backward.json):
| Field | Required Evidence |
|-------|------------------|
| `output_fields[].code_snippet` | Return statement or field assignment |
| `output_fields[].computation_path[].code_snippet` + `file` | Each computation step |
| `output_fields[].deep_analysis.judgment_basis` | Rule reference (O1-O11) |
| `output_fields[].trust_chain` | Complete backward trust chain |
| `output_fields[].risk_reason` | MUST be non-null for at_risk, with file + line + code ref |

### In A04b (backward-verify.json):
| Field | Required Evidence |
|-------|------------------|
| All `*[].code_snippet` | Problematic code |
| All `*[].issue` | file_ref in text |

### In A05 (report.json):
| Field | Required Evidence |
|-------|------------------|
| `scenarios_found[].evidence_chain` | References to specific entries in pipeline products |
| `scenarios_found[].fix_code_snippet` | Concrete fix code |
| `inconsistencies[].description` | Conflicting product refs |

---

## Forbidden Words in Judgment Fields

The following words and their equivalents MUST NOT appear in any `reason`, `trust_chain`, `risk_reason`, `final_trust_status`, `final_judgment`, `trust_effect`, `judgment_basis`, `issue`, or `note` field:

| Forbidden | Reason |
|-----------|--------|
| "可能" | Ambiguous — either it is or it isn't |
| "也许", "或许" | Ambiguous |
| "大概" | Ambiguous |
| "似乎", "好像" | Ambiguous |
| "待确认" | Incomplete — state WHAT needs confirmation |
| "需进一步分析" | Incomplete — state WHY and WHAT's missing |
| "should be" | Not a conclusion |
| "might be", "may be" | Not a conclusion |
| "probably", "likely" | Not a conclusion |
| "seems like", "appears to" | Not a conclusion |
| "一般", "通常" | Over-generalization |
| "大部分情况" | Over-generalization |

**Replacement:**
- If uncertain about a judgment → use `"unresolved"` and fill `unresolved_reason` with:
  - What specific information is missing
  - Why the current evidence is insufficient
  - What would be needed to reach a conclusion

---

## Code Snippet Format

```json
"code_snippet": "Line 45:     Order order = orderRepo.findById(orderId);\nLine 46:     if (!order.getUserId().equals(currentUserId)) {\nLine 47:         throw new AccessDeniedException();"
```

Requirements:
- Include line numbers in the snippet (format: `Line {N}: {code}`)
- Include enough context to understand the judgment (1-3 lines)
- Do NOT truncate in the middle of a key expression
- Separate multiple lines with `\n`
- If the relevant code spans >3 lines, extract the 3 most critical lines and note the full range in `line_start`/`line_end`

---

## Sample Violations

### VIOLATION: Missing code_snippet
```json
// WRONG:
{ "final_trust_status": "trusted", "trust_chain": "R1: param is in same WHERE clause as anchor" }
// Missing: code_snippet, file, line
```

### VIOLATION: Fuzzy word
```json
// WRONG:
{ "risk_reason": "orderId 似乎没有绑定到 auth anchor" }
// Problem: "似乎" (seems) is a forbidden fuzzy word
```

### VIOLATION: No rule reference
```json
// WRONG:
{ "trust_effect": "查询结果可信" }
// Missing: which rule? R1? R2? R3?
```

### CORRECT:
```json
{
  "final_trust_status": "trusted",
  "trust_chain": "currentUserId → WHERE user_id = currentUserId (R1: orderId in same constraint) → trusted",
  "trust_rule": "R1",
  "file": "src/main/java/com/example/OrderService.java",
  "line": 82,
  "code_snippet": "Line 82:     List<Order> orders = orderRepo.findByOrderIdAndUserId(orderId, currentUserId);"
}
```
