# Security, Permissions, and Overrides

## Table of Contents
- [Baseline Controls](#baseline-controls)
- [Macro/UDF Syntax and Auto-Detection](#macroud-syntax-and-auto-detection)
- [Permission Model (Hierarchy)](#permission-model-hierarchy)
  - [Policy Storage and Consistency](#policy-storage-and-consistency)
- [Roles and Defaults (Editor vs Viewer)](#roles-and-defaults-editor-vs-viewer)
  - [Default Macro Permissions (Aligned to Edit vs View)](#default-macro-permissions-aligned-to-edit-vs-view)
  - [Operation-Level Enforcement](#operation-level-enforcement)
  - [Policy Hierarchy Interaction](#policy-hierarchy-interaction)
- [Admin-Granted Overrides](#admin-granted-overrides)
- [Policy Configuration Examples](#policy-configuration-examples)
- [Example Errors](#example-errors) 

## Baseline Controls
- Sandbox macro code (no filesystem/process access).
- Optional network access (configurable).
- Enforce time/step limits per macro execution.
- Validate macro names, argument counts, input sizes.
- **AST Validation**: Parse code into Abstract Syntax Tree (AST) for safe analysis—enforce single-function structure, validate imports against whitelist if global imports allowed. Code is rejected on commit if it violates AST rules or policy. Runtime sandbox enforces policies to prevent bypass (e.g., blocks non-whitelisted imports even if AST misses edge cases)it if it violates AST rules or policy. Runtime sandbox enforces policies to prevent bypass (e.g., blocks non-whitelisted imports even if AST misses edge cases).

## Macro/UDF Syntax and Auto-Detection
- **File-Based Differentiation**: Macros in `.macro.js` files; UDFs in `.func.js` files. Any JS function defined in the respective file is treated as such.
- **Code Structure**: Single named JS function (e.g., `function myMacro(args) { ... }`). Optional top-level imports if policy allows (e.g., `import { util } from 'allowedModule';`).
- **AST Validation**: Parse code into Abstract Syntax Tree (AST) for safe analysis—enforce single-function structure, validate imports against whitelist if global imports allowed. Code is rejected on commit if it violates AST rules or policy.
- **Global Imports**: If enabled by policy, validate imports against a whitelist (e.g., no arbitrary file system access). Imports must be at top-level; main logic in the function.

## Permission Model (Hierarchy)
1. **Global policy** (server default)
2. **Per-room policy** (admin-controlled)
3. **Per-macro flags** (can only reduce permissions; applies to both UDFs and action macros)
4. **Per-user overrides** (optional; only if server enables)

Effective rule: **most restrictive wins**, unless a trusted override is explicitly granted.

Note: UDFs (user-defined functions) and action macros are differentiated by file extension: `.func.js` for UDFs (used in formulas); `.macro.js` for macros (executed via run). Type is inferred from file extension. Global imports are allowed if sandbox policy permits, validated for whitelisted modules only.

### Policy Storage and Consistency
- **Ghost Policy Prevention**: Policies must be stored atomically with macro/UDF code to ensure consistency. On macro edit commit or repo sync, scan for orphaned policies (e.g., policies referencing non-existent macros). If found, interrupt the operation with a warning and require user to fix/remove the policy before proceeding.
- **Workflow Interruption**:
  - **Macro Editing**: Commit blocked if policies would be orphaned.
  - **Repo Sync**: Sync blocked if fetched changes would orphan policies; user must resolve conflicts.
- **Atomic Updates**: Code and policies are bundled together, allowing atomic rejection of updates that would create inconsistencies.

## Roles and Defaults (Editor vs Viewer)
EtherCalc already has a view/edit model that acts as the base trust boundary:
- **Editor**: a client with valid edit auth (`auth=<hmac(room)>`). Editors can send execute/command messages and change sheet state.
- **Viewer**: a client in view mode (`auth=0` or `view=1`). Viewers cannot send execute/command messages; they only receive updates.

### Default Macro Permissions (Aligned to Edit vs View)
- **Create macros/UDFs**: Editor only.
- **Edit macros/UDFs**: Editor only.
- **Run macros/UDFs**: Editor only.
- **View-only users**: cannot create/edit/run macros; they only see results produced by editors or server-side recalc.

### Operation-Level Enforcement
- **Create/Edit**: requires Editor auth, must pass policy checks (global → room → macro), and must acquire/hold the macro edit lock.
- **Run**: requires Editor auth and must pass policy checks (global → room → macro).
- **Locks**: lock acquisition and modification are restricted to Editors; locks are scoped per room + macro.
- **Policy Edits**: Modifying room/macro policies requires acquiring the same write lock used for editing the associated macro/UDF code. This shared lock prevents concurrent changes to both code and policies. Locks are per macro resource and must be held during edit/save.

### Policy Hierarchy Interaction
- The role check (Editor vs Viewer) is the baseline gate.
- Global/room/macro policies can further restrict Editor capabilities.
- Per-user overrides (if enabled) are the only way to grant more than global/room defaults.

## Admin-Granted Overrides
- Allow trusted users to exceed room/global limits only if server policy permits.
- Overrides should be scoped (per room, per macro, or user-wide).
- All override usage must be audited.

## Policy Configuration Examples
To differentiate from the global sandbox configuration file (see [11-sandboxing.md](11-sandboxing.md#6-configuration-file)), room and macro policies are **override objects** that can only tighten restrictions (most restrictive wins). They contain only the settings being overridden, not the full config.

#### Room Policy Example
A room policy applies to all macros/UDFs in a specific room, tightening the global defaults. Only editors with admin rights can set room policies.

```json
{
  "resourceLimits": {
    "maxRuntimeMs": 200,
    "maxMemoryMb": 32
  },
  "network": {
    "enabled": false
  },
  "api": {
    "allowClipboardAccess": false
  }
}
```

#### Macro Policy Example
A macro policy applies to a specific macro/UDF, further restricting room/global policies. Authors can set macro policies when creating/editing their macros.

```json
{
  "execution": {
    "allowEval": false
  },
  "resourceLimits": {
    "maxRuntimeMs": 100
  },
  "network": {
    "enabled": false
  },
  "api": {
    "allowSheetWrite": false,
    "allowWorkbookWrite": false
  }
}
```

These overrides are merged with higher-level policies, and any conflicts (attempts to loosen restrictions) are rejected.

## Example Errors
- Permission denied: "Macro execution blocked: network access not allowed."
- Policy conflict: "Macro requires network but room policy denies it."
- Timeout: "Macro exceeded time limit (500ms)."
- Validation: "Macro name invalid."
- AST Violation: "Code rejected: multiple functions or non-whitelisted import detected."
- Import Bypass: "Runtime blocked: import not allowed by policy."

Excel analogy:
- Equivalent to Excel’s macro security levels (Disable, Warn, Allow) and Trusted Locations/Publishers.

## Dependencies
- Backend enforcement in macro execution (server worker) and policy storage (TBD: backend spec file).
- API endpoints to read/write policies and macro metadata (MACRO-PLAN/05-backend-api.md).
- UI controls to configure policies, overrides, and prompts (TBD: UI spec file).

## Remaining todos for planning
- Security/sandboxing: allowing network access safely requires a clear policy and strict runtime isolation.
- Lock down permission checks for all endpoints and websocket events (create/edit/run/lock).
- Implement policy lifecycle: atomic storage with code, ghost policy detection/prevention, workflow interruption for fixes, and lock requirements for edits. Ensure AST validation and runtime enforcement prevent policy bypass.
