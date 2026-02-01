# Security, Permissions, and Overrides

## Baseline Controls
- Sandbox macro code (no filesystem/process access).
- Optional network access (configurable).
- Enforce time/step limits per macro execution.
- Validate macro names, argument counts, input sizes.

## Permission Model (Hierarchy)
1. **Global policy** (server default)
2. **Per-room policy** (admin-controlled)
3. **Per-macro flags** (can only reduce permissions)
4. **Per-user overrides** (optional; only if server enables)

Effective rule: **most restrictive wins**, unless a trusted override is explicitly granted.

## Admin-Granted Overrides
- Allow trusted users to exceed room/global limits only if server policy permits.
- Overrides should be scoped (per room, per macro, or user-wide).
- All override usage must be audited.

## Example Errors
- Permission denied: “Macro execution blocked: network access not allowed.”
- Policy conflict: “Macro requires network but room policy denies it.”
- Timeout: “Macro exceeded time limit (500ms).”
- Validation: “Macro name invalid.”

Excel analogy:
- Equivalent to Excel’s macro security levels (Disable, Warn, Allow) and Trusted Locations/Publishers.

## Dependencies
- Backend enforcement in macro execution (server worker) and policy storage (TBD: backend spec file).
- API endpoints to read/write policies and macro metadata (MACRO-PLAN/05-backend-api.md).
- UI controls to configure policies, overrides, and prompts (TBD: UI spec file).
