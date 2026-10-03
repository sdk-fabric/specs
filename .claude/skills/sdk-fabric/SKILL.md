---
name: sdk-fabric
description: Multi-agent orchestrator to inspect, build, modify, and validate TypeAPI specifications in targets/ using the sdkgen toolchain. Use when the user wants to create a new API spec, or add/change/fix operations or types in an existing spec under targets/.
---

# TypeAPI Specification Builder Skill

This skill runs a multi-agent workflow (**Architect → Type Builder → Operation Builder → Validation**) to build and modify TypeAPI specifications in `targets/` using `sdkgen`.

**Before starting, read [conventions.md](conventions.md).** It is the single source of truth for file layout, naming, type/operation shapes and verified `sdkgen` quirks. Every sub-agent reads it too.

---

## Ground Rules

* Run all commands from the repo root with **relative** paths (`sdkgen` breaks on absolute paths).
* Each spec is `targets/<service>.json` (lowercase service name, e.g. `targets/airtable.json`).
* **Never run two builders against the same spec file at the same time.** `sdkgen patch` rewrites the whole file, so concurrent patches lose changes. Builders for *different* files may run in parallel.
* Sub-agents start without this conversation's context. Every task you hand off must be self-contained: target file, full entity details, and the instruction to read `conventions.md`.

---

## Tooling Quick Reference

| Command | Purpose |
|---|---|
| `sdkgen inspect targets/<s>.json` | List all operations and types |
| `sdkgen inspect targets/<s>.json --search <term>` | Filter by substring |
| `sdkgen inspect targets/<s>.json --target types.<Name>` | Show one entity as JSON |
| `sdkgen patch targets/<s>.json --target <entity> --file tmp_<s>_<Entity>.json` | Create/replace (object payload) or edit (RFC 6902 array payload) |
| `sdkgen validate targets/<s>.json` | Schema, reference and convention validation (`:param` paths, path args, required keys, naming) |

---

## Execution Protocol

### Step 1: Planning (Architect Agent)
1. Launch the `architect` agent with: the user's request verbatim, the target file path, and any vendor doc URLs the user gave.
2. The architect researches the vendor API docs, inspects the existing spec, and returns a build plan with complete type and operation specs.
3. Review the plan. If it lists **Open Questions** that would change the outcome, ask the user before continuing. For large requests (e.g. "add the whole API"), confirm the scope with the user.

### Step 2: Initialize (new specs only)
If `targets/<service>.json` does not exist, create it with the Write tool using the metadata from the plan (`patch` cannot create files):
```json
{
  "baseUrl": "<from plan, or \"\" for self-hosted>",
  "security": { "type": "httpBearer" },
  "operations": {},
  "definitions": {}
}
```
Omit `security` if the plan says the API is unauthenticated.

### Step 3: Types (Type Builder Agent)
Launch **one** `type-builder` agent with **all** type tasks from the plan (copy the full task blocks, not summaries). It applies them in order, leaves before referrers.

### Step 4: Operations (Operation Builder Agent)
After the type builder has finished, launch **one** `operation-builder` agent with **all** operation tasks from the plan (full task blocks).

> For very large plans (more than about 30 entities), split each phase into sequential batches, waiting for each batch to finish before launching the next.

### Step 5: Validation & Feedback Loop
1. Run:
   ```bash
   sdkgen validate targets/<service>.json
   ```
2. If it fails, decide which entity is at fault from the error path (`/definitions/X` → `types.X`, `/operations/x.y` → `operations.x.y`). Then launch the matching builder with the exact error text plus the current entity JSON from `sdkgen inspect --target`.
3. Re-run `sdkgen validate`. Stop after **3** fix rounds and report the remaining errors to the user instead of looping.
4. Make sure no `tmp_*.json` files are left in the repo root.

### Step 6: Report
Summarize for the user: types and operations added or changed, validation results, any assumptions or open questions.
