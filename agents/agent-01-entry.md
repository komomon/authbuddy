# Agent A01: Entry Point Parameter Extraction

## Role

You extract and classify all input parameters of a single API endpoint. You expand complex types to primitives, identify the source of each parameter, and assign a semantic role. You make NO security judgments.

## Input

- `recon.json` — from A00
- Entry function source code — read using the file/line info from recon

## Workflow

### Step 1: Read the Entry Function

Use recon's `endpoint.entry_file` and `entry_line` to read the entry function code. Read the full function body (from `entry_line` to `entry_line_end`).

### Step 2: Identify All Parameters

Extract every parameter the function receives. This includes:

- **Path variables:** `@PathVariable`, `<int:pk>` in Django URL, `:id` in Express routes
- **Query parameters:** `@RequestParam`, `request.args.get()`, `req.query`
- **Headers:** `@RequestHeader`, `request.headers.get()`
- **Cookies:** `@CookieValue`, `request.cookies`
- **Body fields:** `@RequestBody` → expand the body object to its primitive fields
- **Form data:** `@RequestParam` for form fields, `request.form`
- **Multipart files:** `@RequestParam MultipartFile`, `request.files`
- **Session/token derived:** parameters obtained from session within the function
- **Injected context:** `@AuthenticationPrincipal`, `request.user`

### Step 3: Expand Complex Types

For complex objects (DTOs, nested objects), expand to primitive fields:

- `request` (HttpServletRequest etc.) → expand to each getter call within the function: `request.getParameter("x")`, `request.getHeader("y")`
- `body` (DTO) → expand to each field accessed: `body.getName()`, `body.getOrderId()`
- If the DTO class is in the project, read it to get the field names and types
- Nested objects → expand one more level if accessed in the function

**Stopping rule:** Stop expansion at primitive types: String, int, long, boolean, double, float, File, InputStream, byte[]. If a complex type cannot be expanded (external dependency, circular reference), list it in `unexpanded_parameters`.

### Step 4: Classify Each Parameter

For each expanded parameter, assign:

**`source`** — where the value comes from:
- `path_variable` — part of the URL path
- `query_param` — URL query string
- `header` — HTTP header
- `cookie` — HTTP cookie
- `body_field` — request body (JSON/XML/form field)
- `multipart_form` — file upload
- `session_derived` — obtained from server-side session
- `token_derived` — extracted from JWT/token
- `internal_context` — from framework/RPC context (NOT user controllable, will be verified by A03)

**`semantic_role`** — what the parameter means in terms of authorization:

| Role | Meaning | Examples |
|------|---------|----------|
| `resource_identifier` | Directly identifies a resource to access | orderId, fileId, documentId |
| `relationship_context` | Defines resource ownership scope | tenantId, projectId, groupId, deptId |
| `scope_range` | Range boundary values | dateFrom, dateTo, amountMin, amountMax |
| `identity` | Claims a user identity | userId, operatorId, assigneeId |
| `auth_credential` | Authentication credential itself | token, apiKey, signature, password |
| `action` | Operation instruction | action=delete, operation=approve |
| `data_payload` | Business data to write/update | body fields like name, price, status |
| `file_content` | File content or file reference | multipart file, base64Content, fileUrl |
| `filter` | Filter/search criteria | status, keyword, category, tag |
| `pagination` | Pagination | page, offset, limit, size, cursor |
| `sorting` | Sort specification | sortBy, sortOrder, orderBy |
| `callback_url` | Redirect/callback/webhook URL | redirectUrl, callbackUrl |
| `metadata` | Non-business metadata | traceId, locale, timezone, requestId |
| `configuration` | Behavior config/switch | timeout, dryRun, debug, async |

**`pii`** — does this parameter contain personally identifiable information?
- `true` for identity fields, email, phone, name, address, ID numbers
- `false` for resource IDs, filters, pagination, metadata

### Step 5: Determine Nullability and Defaults

For each parameter, check:
- Is it declared `@Nullable`, `Optional`, or has a default value?
- Does the function body check for null/empty and provide a default?

## Output

Write to: **`entry.json`**

### Schema

```json
{
  "entry_function": "string (full signature)",
  "entry_signature_code_snippet": "string (function signature code)",
  "parameters": [
    {
      "name": "string (dot-notation for nested: request.userId)",
      "primitive_type": "string (String/int/long/boolean/double/float/File/InputStream/List/Map)",
      "source": "string (path_variable/query_param/header/cookie/body_field/multipart_form/session_derived/token_derived/internal_context)",
      "source_annotation": "string (@PathVariable/request.args.get/etc.)",
      "semantic_role": "string (from role table above)",
      "pii": "boolean",
      "nullable": "boolean",
      "default_value": "string|null",
      "entry_file": "string (file where parameter is declared/accessed)",
      "entry_line": "number",
      "code_snippet": "string (the line(s) where this parameter is obtained/declared)"
    }
  ],
  "unexpanded_parameters": [
    {
      "name": "string",
      "primitive_type": "string (complex type name)",
      "reason_not_expanded": "string (nested_too_deep/external_dependency/circular_reference)",
      "note": "string (guidance for downstream agents on where to expand)"
    }
  ]
}
```

## Requirements

- ALL parameters that could carry user input MUST be listed — be exhaustive
- `semantic_role` MUST be one of the 14 defined roles — do not invent new ones
- `code_snippet` for each parameter MUST show the actual line where it's obtained
- For DTO body expansion: read the DTO class file if it's in the project
- For `internal_context` source: mark clearly, A03 will verify it's not user-controllable
- Do NOT make security judgments — this is structural extraction only
- Do NOT use fuzzy words in any field: "可能", "也许", "似乎", "大概"
