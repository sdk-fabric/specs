---
name: sdk-fabric
description: Multi-agent orchestrator to inspect, build, modify, and validate TypeAPI specifications in targets/ using the sdkgen toolchain.
---

# TypeAPI Specification Builder Skill

This skill orchestrates a multi-agent workflow (**Architect -> Type Builder / Operation Builder -> Validator**) to safely build and modify TypeAPI specifications in the `targets/` directory using `sdkgen`.

---

## Repository Structure & Target Conventions

* All TypeAPI specifications reside inside the **`targets/`** directory (e.g., `targets/airtable.json`, `targets/deepseek.json`, `targets/discord.json`).
* **New specifications:** Created as `targets/<service>.json`.
* **Existing specifications:** Inspected and patched in-place.

---

## Tooling Quick Reference

1. **`sdkgen inspect <file>`**: Inspects operations/types or specific entities using `--target <domain.Name>` or `--search <term>`.
2. **`sdkgen patch <file> --target <target> --file <patch.json>`**: Mutates a targeted entity cleanly using a JSON payload file.
3. **`sdkgen validate <file>`**: Validates structural correctness, types, and cross-references.

---

### Multi-Agent Orchestration Workflow

1. **Phase 1: Planning & Analysis (Architect Agent)**
    * **Inspect:** Runs `sdkgen inspect targets/<service>.json` to check existing types, operations, and metadata.
    * **Identify Metadata:** Determines if the API is a SaaS service (requires concrete `baseUrl` like `https://api.airtable.com`) or self-hosted (requires empty `"baseUrl": ""`).
    * **Build Execution Plan:** Creates a structured build plan breaking down the request into discrete tasks for data models (`types.*`) and endpoints (`operations.*`).

2. **Phase 2: Data Model Execution (Type Builder Agent)**
    * **Process Type Tasks:** Receives `types.*` tasks sequentially or in parallel from the plan.
    * **Write Payload:** Writes each type schema into a temporary JSON file (`tmp_type_<Name>.json`).
    * **Patch Spec:** Executes `sdkgen patch targets/<service>.json --target types.<TypeName> --file tmp_type_<Name>.json`.
    * **Cleanup:** Removes temporary payload files.

3. **Phase 3: Endpoint Execution (Operation Builder Agent)**
    * **Process Operation Tasks:** Receives `operations.*` tasks once dependent types are in place.
    * **Write Payload:** Writes each operation schema into a temporary JSON file (`tmp_op_<Name>.json`).
    * **Patch Spec:** Executes `sdkgen patch targets/<service>.json --target operations.<OpName> --file tmp_op_<Name>.json`.
    * **Cleanup:** Removes temporary payload files.

4. **Phase 4: Validation & Feedback Loop (Main Orchestrator)**
    * **Validate:** Runs `sdkgen validate targets/<service>.json`.
    * **Check Results:**
        * If **valid**: Task is complete.
        * If **errors found** (e.g., missing type reference or invalid path argument): Routes targeted error logs back to `type-builder` or `operation-builder` to re-patch and re-validate.

---

## Execution Protocol

### Step 1: Planning Phase (Architect Agent)
When a user requests a creation or modification task:
1. Initialize `targets/<service>.json` with metadata (`baseUrl`, `security`) if it does not exist:
  * SaaS/Cloud API: set concrete `baseUrl` (e.g., `"https://api.airtable.com"`).
  * Self-hosted/Open-source API: set empty `baseUrl` (`""`).
2. Delegate to the `architect` agent to run `sdkgen inspect` and output a structured execution plan.

### Step 2: Execution Phase (Sub-Agents)
Execute the Architect's plan by dispatching tasks to dedicated agents:
1. **Types First:** Dispatch all type definition tasks to the `type-builder` agent.
2. **Operations Second:** Dispatch all endpoint/operation tasks to the `operation-builder` agent.
3. **Shell Safety:** Ensure sub-agents write payload definitions to temporary JSON files and pass `--file <patch.json>` to `sdkgen patch`.

### Step 3: Validation Phase (Validator)
1. Run `sdkgen validate targets/<service>.json`.
2. If validation fails (e.g., missing type references or syntax error):
  * Inspect error details.
  * Dispatch a targeted fix task to `type-builder` or `operation-builder`.
  * Re-run `sdkgen validate targets/<service>.json`.

