# TypeAPI Conventions for `targets/`

Single source of truth for the `sdk-fabric` skill and the `architect`, `type-builder` and `operation-builder` agents. All examples are taken from `targets/airtable.json`, which validates and follows every rule below.

---

## File Layout

```json
{
  "baseUrl": "https://api.airtable.com",
  "security": { "type": "httpBearer" },
  "operations": {},
  "definitions": {}
}
```

* `baseUrl`: SaaS/cloud API → concrete URL including any version prefix that is shared by all paths (e.g. `https://discord.com/api/v10`). Self-hosted/open-source API → `""`.
* `security`: usually `{"type": "httpBearer"}`. API key in a header: `{"type": "apiKey", "in": "header", "name": "Authorization"}`. Omit the key entirely for public APIs without auth.
* **Targets vs. file keys:** the `sdkgen` target `types.<Name>` maps to `definitions.<Name>` in the file. The target `operations.<name>` maps to `operations.<name>`. There is no `types` key in the file.

---

## Naming

| Entity | Convention | Examples |
|---|---|---|
| Operation | `resource.action`, camelCase segments | `records.getAll`, `records.get`, `records.create`, `records.update`, `records.replace`, `records.delete`, `meta.getWhoami` |
| Type | `Pascal_Snake` (words joined by `_`), nested types prefixed with their parent | `Record`, `Record_Collection`, `Comment_Author`, `Create_Record_Request`, `Error_Details` |

Standard action names: `getAll` (list), `get` (single), `create` (POST), `update` (PATCH), `replace` (PUT), `delete` (DELETE). Bulk variants append `All` (`updateAll`, `replaceAll`).

---

## Types (`definitions.*`)

Allowed top-level `type` values: **`struct`** never `object`.

### Struct
```json
{
  "description": "Paginated list of table records.",
  "type": "struct",
  "properties": {
    "offset": {
      "description": "Pagination offset string for the next page.",
      "type": "string"
    },
    "records": {
      "description": "List of record objects.",
      "type": "array",
      "schema": { "type": "reference", "target": "Record" }
    }
  }
}
```

### Property types
* Scalars: `string`, `integer`, `number`, `boolean`, `any`.
* `"format": "date-time"` or `"format": "date"` on strings for timestamps/dates.
* `"nullable": true` when the API may return `null`.
* Arrays and maps carry their item type in `schema`: `{"type": "array", "schema": {...}}`, `{"type": "map", "schema": {...}}`.
* References: `{"type": "reference", "target": "Type_Name"}`. The target must exist in `definitions`.
* Never inline nested objects. Create a separate named type and reference it.
* Every type and property gets a `description` taken from the vendor docs.

### Inheritance / polymorphism (see `targets/deepseek.json`)
Base type:
```json
{
  "type": "struct",
  "base": true,
  "properties": { "role": { "type": "string" } },
  "discriminator": "role",
  "mapping": { "Completion_Message_User": "user", "Completion_Message_Tool": "tool" }
}
```
Child type:
```json
{
  "type": "struct",
  "parent": { "type": "reference", "target": "Completion_Message" },
  "properties": { "content": { "type": "string" } }
}
```

---

## Operations (`operations.*`)

Every operation contains **all** of these keys:

```json
{
  "path": "/v0/:baseId/:tableIdOrName",
  "method": "POST",
  "return": {
    "code": 200,
    "schema": { "type": "reference", "target": "Record_Collection" }
  },
  "arguments": {
    "baseId": { "in": "path", "schema": { "type": "string" } },
    "tableIdOrName": { "in": "path", "schema": { "type": "string" } },
    "payload": { "in": "body", "schema": { "type": "reference", "target": "Create_Record_Request" } }
  },
  "throws": [
    { "code": 999, "schema": { "type": "reference", "target": "Error" } }
  ],
  "description": "Creates multiple records.",
  "stability": 1,
  "security": [],
  "authorization": true
}
```

Rules:
* **Path parameters use `:name`, never `{name}`.** Every `:name` in the path needs a matching argument with `"in": "path"`, and the other way round.
* `arguments` is always present, even when empty (`{}`).
* `in` is one of `path`, `query`, `header`, `body`.
* The request body is a single argument named **`payload`** with `"in": "body"` that references a type. No body on `GET`/`DELETE`.
* `return` is always `{"code": <status>, "schema": {...}}`, never a bare reference.
* `throws` uses code `999` (catch-all) pointing to the service's error type, unless the API documents distinct error bodies per status code. Public APIs without an error type may use `[]`.
* `stability: 1`, `security: []`, `authorization: true` by default. Use `authorization: false` for endpoints that need no auth.
* `description` comes from the vendor docs.

---

## sdkgen Behavior (v0.5, verified)

| Behavior | Consequence |
|---|---|
| Absolute paths fail (the working dir gets prepended, e.g. `specs\C:\...`) | Always run from the repo root with **relative** paths (`targets/x.json`, `tmp_x.json`) |
| `patch` accepts invalid payloads without complaint | Always run `validate` afterwards |
| `patch` fails if the spec file does not exist | Create the skeleton file with the Write tool first |
| `patch` rewrites the **whole** file with keys sorted alphabetically (the canonical format, same as the web editor) | Never patch the same file from two agents at once (lost updates). Never hand-edit key order |
| `--file` with an object → replaces/creates the entire entity | Use for new entities or deliberate full rewrites |
| `--file` with an array → applied as RFC 6902 ops relative to the entity | Use for small edits to existing entities so other fields are kept |
| `--dry-run` prints the entire spec | Avoid on large files. Check results with `sdkgen inspect <file> --target <entity>` |
| `validate` runs three stages: JSON schema, references, conventions (naming, required operation keys, `:param` paths and path arguments, body arguments) | It stops at the first failing stage, so re-run after each fix |

RFC 6902 example (adds a property to an existing type):
```json
[
  { "op": "add", "path": "/properties/email", "value": { "description": "User's primary email address.", "type": "string" } }
]
```

---

## Payload Files

* Write payloads to the repo root as `tmp_<service>_<Entity>.json` (e.g. `tmp_airtable_Record_Collection.json`, `tmp_airtable_records.getAll.json`). The pattern `tmp_*.json` is gitignored.
* Never pass JSON inline via `--data`.
* Delete the payload file after a successful patch.
