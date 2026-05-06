# Extended Knowledge

User-confirmed discoveries from previous analyses. This file supplements — does NOT replace — built-in references.

## Purpose

When an agent discovers an auth pattern, datasink pattern, or analysis insight NOT covered in the other reference files, it is listed in the report's `extended_knowledge_candidates`. After user review and approval, confirmed patterns are appended here.

## Rules

- **Append only** — never modify existing reference files
- **Include evidence** — each entry must reference the analysis where it was discovered
- **Include date** — patterns may become stale as frameworks evolve
- **User confirmed** — only write entries the user has explicitly approved

## Format

Each entry follows this format:

```markdown
### {pattern_name} — {date_confirmed}

**Category:** {trust_propagation | datasink | anchor_recognition | output_analysis | scenario}

**Discovery context:**
- Project: {project_name}
- Framework: {framework/version}
- Endpoint: {endpoint}
- Date: {YYYY-MM-DD}

**Description:**
{What was discovered, how to recognize it, why existing references didn't cover it}

**Detection guidance:**
{For agents: how to spot this pattern in future analyses}

**Trust implication:**
{How this affects authorization judgments}
```

## Confirmed Patterns

<!-- New patterns will be added below after user approval -->
<!-- Example entry format is shown above; actual entries appear here after confirmation -->

_No confirmed patterns yet. Patterns discovered during audits will be added here after user review and approval._
